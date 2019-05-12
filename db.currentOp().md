#db.currentOp()

##定义

`db.currentOp()`
返回一个含有数据库实例上执行中操作信息的文档。


`db.currentOp()` 方法有以下几种形式:

>db.currentOp(\<operations\>)

`db.currentOp()` 可以使用以下可选参数:

参数 | 描述
------------ | -------------
Options | 可选，指定返回报告的选项，可传入boolean值或文档。<br>指定`true`,返回包含空闲的操作和系统操作，指定一个查询条件文档，则只返回符合条件的操作，可*参考行为（Behavior）*

##行为（Behavior）

如果在`db.currentOp()`中传入`true`，方法返回所有的操作信息，包括空闲连接的操作和系统操作。

>db.currentOp(true)

传入`true`等价于传入这样一个查询文档`{ '$all': true }`。

如果是传入查询文档至`db.currentOp()`，则输出的结果将只返回符合查询条件的操作，你也可以指定返回的字段。见示例。


##访问控制
对于需要授权访问的系统，使用的用户必须拥有inprog action的访问权限，请参考创建管理当前操作的角色（Role to Manage Current Operations)。

##示例
以下示例演示了`db.currentOp()`使用不同的参数来过滤输出结果。

###__等待锁的写操作（Write Operations Waiting for a Lock）__
这个例子会返回所有等待锁的写操作：

```javascript
db.currentOp(
   {
     "waitingForLock" : true,
     $or: [
        { "op" : { "$in" : [ "insert", "update", "remove" ] } },
        { "query.findandmodify": { $exists: true } }
    ]
   }
)
```

###__活动的未就绪操作（Active Operations with no Yields）__
以下返回所有从未处于就绪态的活动操作：

```javascript
db.currentOp(
   {
     "active" : true,
     "numYields" : 0,
     "waitingForLock" : false
   }
)
```

###__指定数据库上的活动操作（Active Operations on a Specific Database）__
以下返回数据库`db1`上所有运行超过3秒的活动操作：

```javascript
db.currentOp(
   {
     "active" : true,
     "secs_running" : { "$gt" : 3 },
     "ns" : /^db1\./
   }
)
```

###__活动的索引操作（Active Indexing Operations）__

以下返回创建索引的操作：

```javascript
db.currentOp(
    {
      $or: [
        { op: "query", "query.createIndexes": { $exists: true } },
        { op: "insert", ns: /\.system\.indexes\b/ }
      ]
    }
)
```

输出

以下是`db.currentOp()`输出的原型。

```javascript
{
  "inprog": [
       {
         "desc" : <string>,
         "threadId" : <string>,
         "connectionId" : <number>,
         "opid" : <number>,
         "active" : <boolean>,
         "secs_running" : <NumberLong()>,
         "microsecs_running" : <number>,
         "op" : <string>,
         "ns" : <string>,
         "query" : <document>,
         "insert" : <document>,
         "planSummary": <string>,
         "client" : <string>,
         "msg": <string>,
         "progress" : {
             "done" : <number>,
             "total" : <number>
         },
         "killPending" : <boolean>,
         "numYields" : <number>,
         "locks" : {
             "Global" : <string>,
             "MMAPV1Journal" : <string>,
             "Database" : <string>,
             "Collection" : <string>,
             "Metadata" : <string>,
             "oplog" : <string>
         },
         "waitingForLock" : <boolean>,
         "lockStats" : {
             "Global": {
                "acquireCount": {
                   "r": <NumberLong>,
                   "w": <NumberLong>,
                   "R": <NumberLong>,
                   "W": <NumberLong>
                },
                "acquireWaitCount": {
                   "r": <NumberLong>,
                   "w": <NumberLong>,
                   "R": <NumberLong>,
                   "W": <NumberLong>
                },
                "timeAcquiringMicros" : {
                   "r" : NumberLong(0),
                   "w" : NumberLong(0),
                   "R" : NumberLong(0),
                   "W" : NumberLong(0)
                },
                "deadlockCount" : {
                   "r" : NumberLong(0),
                   "w" : NumberLong(0),
                   "R" : NumberLong(0),
                   "W" : NumberLong(0)
                }
             },
             "MMAPV1Journal": {
                ...
             },
             "Database" : {
                ...
             },
             ...
         }
       },
       ...
   ],
   "fsyncLock": <boolean>,
   "info": <string>
}
```

##输出字段

__currentOp.desc__

客户端描述. 这个字符串包括`connectionId`。

__currentOp.threadId__

处理所执行操作和连接的进程标志ID。

__currentOp.connectionId__

发起操作的连接ID。

__currentOp.opid__
操作ID,你可以在mongo shell中将这个ID传给`db.killOp()`来终止操作。

>警告 <br>终止运行的操作要非常谨慎，只使用`db.killOp()`来结束客户端发起的操作，不要用它来结束数据库内部操作

__currentOp.active__
布尔值指示该操作是否已经开始，`true`表示已经开始，`false`表示空闲，如空闲连接或者是当前空闲的内部进程，一个操作对于另外一个操作即使已经处于就绪态，也可以是活动的。

v3.0改动: 因为一些非活动的后台进程，例如一个不活动signalProcessingThread，MongoDB会阻止这些各种空字段。

__currentOp.secs_running__
该操作执行的秒数，Mongodb通过使用当前时间减去操作开始计算该值。

只有运行中的操作会出现; 如 `active`为`true`

__currentOp.microsecs_running__
v2.6 新增

执行的微秒数.


__currentOp.op__
字符串表示操作的类型. 可以是以下这些值:

- `none`
- `update`
- `insert`
- `query`
- `getmore`
- `remove`
- `killcursors`
- `query`

`query` 操作不仅包括读操作，还包括一些命令例如创建索引、`findAndModify `命令。

v3.0 改动： 使用`insert``update``delete`的写操作分别显示`insert``update``delete`，前版本都在`query`里。

__currentOp.ns__
目标操作的命名空间，构成为 <数据库名>.<集合命>

__currentOp.insert__
含有要插入的文档，仅当op的值为`insert`时出现。
使用`db.collection.insert()`插入命令的操作，op的值为query.

__currentOp.query__
op的值不为`insert`的操作的信息文档，当op类型为"getmore"这类操作时，查询文档也可能为空。

在"query"分类下的"update" "remove" 或读操作，查询文档包含这类操作的预查询信息。

另外一些"query"下的查询命令，查询文档包含命令的详细细节。

v3.0 改动： 前版本写命令被分类在"query"里，并在查询文档中返回写命令信息（例如，预查询，更新）；

__currentOp.planSummary__
包含查询计划的字符串可以帮助debug执行慢的查询。

__currentOp.client__
发起操作客户端连接IP地址或者hostname.

__currentOp.locks__
v3.0改动

locks文档告诉你当前操作所使用的锁的模式和类型，锁类型如下：

`Global` 表示全局锁。
`MMAPV1Journal` 表示MMAPv1存储引擎使用该锁同步日志写入； 非MMAPv1存储引擎的MMAPV1Journal的模式为空。
`Database` 表示数据库锁。
`Collection` 表示集合锁。
`Metadata` 表示元数据锁。
`oplog` 表示oplog上的锁。

锁的模式：

`R` 共享锁。
`W` 排他锁。
`r` 意图共享锁。
`w` 意图排他锁。

__currentOp.waitingForLock__
返回一个布尔值，true表示正在等待锁，false表示已经获得需要的锁。

__currentOp.msg__
操作的状态和进度信息。如创建索引和做mapreduce时，返回完成进度的百分比。

__currentOp.progress__
mapreduce或索引操作的进度。progress字段对应msg字段的完成百分比。

currentOp.progress.done
完成数

currentOp.progress.total
总数

__currentOp.killPending__
当操作被标记为需要终止时返回true,当操作进入它的下一个安全终止点时，操作将会被终止。

__currentOp.numYields__
numYields 返回该操作为了让其他操作完成被置空闲的次数。

当操作需要访问的数据Mongodb还未完全读入内存会被置为空闲，当它在等待Mongodb它将允许其他已经有数据在内存中的操作先完成.

__currentOp.fsyncLock__
指明数据库当前是否因为fsync操作被锁。

__currentOp.info__
关于如何从`db.fsyncLock()`中解锁的信息。仅当fsyncLock为true时显示该字段。

__currentOp.lockStats__
为每一种锁类型和模式提供下列信息：

`currentOp.lockStats.acquireCount`
指定模式下，操作获取锁的次数。

`currentOp.lockStats.acquireWaitCount`
操作等待`acquireCount`获取锁的次数，`acquireWaitCount`小于或等于`acquireCount`。

`currentOp.lockStats.timeAcquiringMicros`
等待获取锁累计微秒数。

`timeAcquiringMicros`除以`acquireWaitCount`可以得到特定的锁模式下平均等待时间。

`currentOp.lockStats.deadlockCount`
操作在等待获取锁时遭遇死锁的次数。