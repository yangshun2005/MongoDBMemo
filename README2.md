MongoDB优化之倒排索引

摘要: 为MongoDB中的数据构建倒排索引(Inverted Index)，然后缓存到内存中，可以大幅提升搜索性能。本文将通过为电影数据构建演员索引，介绍两种构建倒排索引的方法：MapReduce和Aggregation Pipeline。

### 一. 倒排索引
倒排索引(Inverted Index)，也称为反向索引，维基百科的定义是这样的:
是一种索引方法，被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。
这个定义比较学术，忽略...

倒排索引是搜索引擎中的核心数据结构。搜索引擎的爬虫获取的网页数据可以视为键值对，其中，Key是网页地址(url)，而Value是网页内容。网页的内容是由很多关键词(word)组成的，可以视为关键词数组。因此，爬虫获取的网页数据可以这样表示:

```
<url1, [word2, word3]>
<url2, [word2]>
<url3, [word1, word2]>
```
但是，用户是通过关键词进行搜索的，直接使用原始数据进行查询的话则需要遍历所有键值对中的关键词数组，效率是非常低的。

因此，用于搜索的数据结构应该以关键词(word)为Key，以网页地址(url)为Value:
```
<word1, [url3]>
<word2, [ur1, url2, url3]>
<word3, [url1]>
```
这样的话，查询关键词word2，立即能够获取结果: [ur1, url2, url3]。

简单地说，倒排索引就是把Key与Value对调之后的索引，构建倒排索引的目的是提升搜索性能。

### 二. 测试数据
MongoDB是文档型数据库，其数据有三个层级: 数据库(database)，集合(collection)和文档(document)，分别对应关系型数据库中的三个层级的: 数据库(database), 表(table)，行(row)。MongDB中每个的文档是一个JSON文件，例如，本文使用的movie集合中的一个文档如下所示:
```
{
    "_id" : ObjectId("57d02d60b128567fc130287d"),
    "movie" : "Pride & Prejudice",
    "starList" : [
        "Keira Knightley",
        "Matthew Macfadyen"
    ],
    "__v" : 0
}
```
该文档一共有4个属性:

* _id: 文档ID，由MongoDB自动生成。
* __v: 文档版本，由MongoDB的NodeJS接口Mongoose自动生成。
* movie: 电影名称。
* starList: 演员列表。

可知，这个文档表示电影《傲慢与偏见》，由女神凯拉·奈特莉主演。

忽略`_id`与`__v`，movie集合的数据如下:
```
{
    "movie": "Pride & Prejudice",
    "starList": ["Keira Knightley", "Matthew Macfadyen"]
},
{
    "movie": "Begin Again",
    "starList": ["Keira Knightley", "Mark Ruffalo"]
},
{
    "movie": "The Imitation Game",
    "starList": ["Keira Knightley", "Benedict Cumberbatch"]
}
```
其中，Key为电影名称(movie)，而Value为演员列表(starList)。

这时查询Keira Knightley所主演的电影的NodeJS代码如下:
```
Movie.find(
{
    starList: "Keira Knightley"
},
{
    _id: 0,
    movie: 1
}, function(err, results)
{
    if (err)
    {
        console.log(err);
        process.exit(1);
    }
    console.log("search movie success:\n");
    console.log(JSON.stringify(results, null, 4));
    process.exit(0);
});
```
* 注：本文所有代码使用了MongoDB的NodeJS接口Mongoose，它与MongoDB Shell的接口基本一致。

代码并不复杂，但是数据量大时查询性能会很差，因为这个查询需要:

* 遍历整个movie集合的所有文档
* 遍历每个文档的startList数组

构建倒排索引可以有效地提升搜索性能。本文将介绍MongoDB中两种构建倒排索引的方法：MapReduce与Aggregation Pipeline。

### 三 MapReduce
MapReduce是由谷歌提出的编程模型，适用于多种大数据处理场景，在搜索引擎中，MapReduce可以用于构建网页数据的倒排索引，也可以用于编写网页排序算法PageRank(由谷歌创始人佩奇和布林提出)。

MapReduce的输入数据与输出数据均为键值对。MapReduce分为两个函数: Map与Reduce。

Map函数将输入键值<k1, v1>对进行变换，输出中间键值对<k2, v2>。

MapReduce框架会自动对中间键值对<k2, v2>进行分组，Key相同的键值对会被合并为一个键值对<k2, list(v2)>。

Reduce函数对<k2, list(v2)>的Value进行合并，生成结果键值对<k2, v3>。

使用MapReduce构建倒排索引的NodeJS代码如下:
```
var option = {};

option.map = function()
{
    var movie = this.movie;
    this.starList.forEach(function(star)
    {
        emit(star,
        {
            movieList: [movie]
        });
    });
};

option.reduce = function(key, values)
{
    var movieList = [];
    values.forEach(function(value)
    {
        movieList.push(value.movieList[0]);
    });
    return {
        movieList: movieList
    };
};

Movie.mapReduce(option, function(err, results)
{
    if (err)
    {
        console.log(err);
        process.exit(1);
    }
    console.log("create inverted index success:\n");
    console.log(JSON.stringify(results, null, 4));
    process.exit(0);
});
```
代码解释:

* Map函数的输入数据是Movie集合中的各个文档，在代码中用this表示。文档的movie与starList属性构成键值对`<movie, starList>`。Map函数遍历starList，为每个start生成键值对`<star, movieList>`。这时Key与Value进行了对调，且starList被拆分了，movieList仅包含单个movie。

* MongoDB的MapReduce执行框架对成键值对<star, movieList>进行分组，star相同的键值对会被合并为一个键值对`<star, list(movieList)>`。这一步是自动进行的，因此在代码中并没有体现。

* Reduce函数的输入数据是键值对<star, list(movieList)>，在代码中，star即为key，而list(movieList)即为values，两者为Reduce函数的参数。Reduce函数合并list(movieList)，从而得到键值对<star, movieList>，最终，movieList中将包含该star的所有movie。

在代码中，Map函数与Reduce返回的键值对中的Value是一个对象`{ movieList: movieList }`，而不是数组`movieList`，因此代码和结果都显得比较奇怪。MongoDB的MapReduce框架不支持Reduce函数返回数组，因此只能将movieList放在对象里面返回。

输出结果:
```
[
    {
        "_id": "Benedict Cumberbatch",
        "value": {
            "movieList": [
                "The Imitation Game"
            ]
        }
    },
    {
        "_id": "Keira Knightley",
        "value": {
            "movieList": [
                "Pride & Prejudice",
                "Begin Again",
                "The Imitation Game"
            ]
        }
    },
    {
        "_id": "Mark Ruffalo",
        "value": {
            "movieList": [
                "Begin Again"
            ]
        }
    },
    {
        "_id": "Matthew Macfadyen",
        "value": {
            "movieList": [
                "Pride & Prejudice"
            ]
        }
    }
]
```
### 四. Aggregation Pipeline
Aggregation Pipeline，中文称作聚合管道，用于汇总MongoDB中多个文档中的数据，也可以用于构建倒排索引。

Aggregation Pipeline进行各种聚合操作，并且可以将多个聚合操作组合使用，类似于Linux中的管道操作，前一个操作的输出是下一个操作的输入。

使用Aggregation Pipeline构建倒排索引的NodeJS代码如下:
```
Movie.aggregate([
{
    "$unwind": "$starList"
},
{
    "$group":
    {
        "_id": "$starList",
        "movieList":
        {
            "$push": "$movie"
        }
    }
},
{
    "$project":
    {
        "_id": 0,
        "star": "$_id",
        "movieList": 1
    }
}], function(err, results)
{
    if (err)
    {
        console.log(err);
        process.exit(1);
    }
    console.log("create inverted index success:\n");
    console.log(JSON.stringify(results, null, 4));
    process.exit(0);
});
```
代码解释:

$unwind: 将starList拆分，输出结果(忽略`_id`与`__v`)为:
```
[
    {
        "movie": "Pride & Prejudice",
        "starList": "Keira Knightley"
    },
    {
        "movie": "Pride & Prejudice",
        "starList": "Matthew Macfadyen"
    },
    {
        "movie": "Begin Again",
        "starList": "Keira Knightley"
    },
    {
        "movie": "Begin Again",
        "starList": "Mark Ruffalo"
    },
    {
        "movie": "The Imitation Game",
        "starList": "Keira Knightley"
    },
    {
        "movie": "The Imitation Game",
        "starList": "Benedict Cumberbatch"
    }
]
```
* $group: 根据文档的starList属性进行分组，然后将分组文档的movie属性合并为movieList，输出结果为:
```
[
    {
        "_id": "Benedict Cumberbatch",
        "movieList": [
            "The Imitation Game"
        ]
    },
    {
        "_id": "Matthew Macfadyen",
        "movieList": [
            "Pride & Prejudice"
        ]
    },
    {
        "_id": "Mark Ruffalo",
        "movieList": [
            "Begin Again"
        ]
    },
    {
        "_id": "Keira Knightley",
        "movieList": [
            "Pride & Prejudice",
            "Begin Again",
            "The Imitation Game"
        ]
    }
]
```
