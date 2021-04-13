###  MongoDB

https://www.runoob.com/mongodb/mongodb-tutorial.html

https://github.com/Vacricticy/mongodb_practice

## 三大基本概念：

- 数据库 database
- 集合(数组) collection
  - 类似与SQL中的数据表，本质上是一个**数组**，里面包含看多个文档对象，[{},{},{}]
- 文档对象 document
  - 类似与SQL中的记录，一个文档**对象**{}就是一条记录
- 一个数据库由多个集合构成，一个集合包含多个文档对象。

| SQL术语/概念 | MongoDB术语/概念 | 解释/说明                           |
| :----------- | :--------------- | :---------------------------------- |
| database     | database         | 数据库                              |
| table        | collection       | 数据库表/集合                       |
| row          | document         | 数据记录行/文档                     |
| column       | field            | 数据字段/域                         |
| index        | index            | 索引                                |
| table joins  |                  | 表连接,MongoDB不支持                |
| primary key  | primary key      | 主键,MongoDB自动将_id字段设置为主键 |

## 基本使用：

- 创建数据库 `use dataBasename` 会去检测数据库是否存在，不存在创建+定位 ，存在直接定位；
  显示数据库 `show dbs`
  删除数据 `db.dropDtabase()`
- 创建集合 `db.createCollection("name",ooptions)`options可不填
  删除集合 `db.collectionName.drop()`
  显示集合 `show collections`

## 数据库的CRUD操作:

### 插入数据

- 插入一条数据（3.2版本之后）
  - db.collectionName.insertOne( {name:'zhangsan'} )
    - db表示的是当前操作的数据库
    - collectionName表示操作的集合，若没有，则会自动创建
    - 插入的文档如果没有手动提供_id属性，则会自动创建一个
- 插入多条数据（3.2版本之后）
  - db.collectionName.insertMany( [ {name:'zhangsan'} , {name:'zhangsan2'} ] )
    - 需要用数组包起来
- 万能API：db.collectionName.insert(document)，可多条也可单条

```javascript
#添加两万条数据
for(var i=0;i<20000;i++){
	db.users.insert({username:'liu'+i}) #需执行20000次数据库的添加操作
}
db.users.find().count()//20000


#优化：
var arr=[];
for(var i=0;i<20000;i++){
	arr.push({username:'liu'+i})
}
db.user.insert(arr) #只需执行1次数据库的添加操作，可以节约很多时间
```

### 查询数据

```shell
db.collection.find(query, projection)
# query：可选，使用查询操作符指定查询条件
# projection ：可选，使用投影操作符指定返回的键。查询时返回文档中所有键值， 只需省略该参数即可（默认省略）。

# 只输出name和title字段，第一个参数为查询条件，空代表查询所有
db.collection.find( {}, { name: 1, title: 1 } )
# 如果需要输出的字段比较多，不想要某个字段，可以用排除字段的方法
# 不输出内容字段，其它字段都输出
db.collection.find( {}, {content: 0 } )
```

- db.collectionName.find() 或db.collectionName.find({})
  - 查询集合所有的文档，即所有的数据。
  - 查询到的是整个**数组**对象。在最外层是有一个对象包裹起来的。
  - db.collectionName.count()或db.collectionName.length() 统计文档个数
- db.collectionName.find({_id:ObjectId('5ed4d1844a9ba463530bc3bf')})
  - 条件查询。注意：结果返回的是一个**数组**
  - _id 查询需要用 `ObjectId()`
- db.collectionName.findOne() 返回的是查询到的对象数组中的第一个对象

| 操作       | 格式                     | 范例                                        | SQL中的类似语句          |
| :--------- | :----------------------- | :------------------------------------------ | :----------------------- |
| 等于       | `{<key>:<value>`}        | `db.col.find({"name":"菜鸟教程"}).pretty()` | `where name= '菜鸟教程'` |
| 小于       | `{<key>:{$lt:<value>}}`  | `db.col.find({"age":{$lt:50}}).pretty()`    | `where age< 50`          |
| 小于或等于 | `{<key>:{$lte:<value>}}` | `db.col.find({"age":{$lte:50}}).pretty()`   | `where age<= 50`         |
| 大于       | `{<key>:{$gt:<value>}}`  | `db.col.find({"age":{$gt:50}}).pretty()`    | `where age> 50`          |
| 大于或等于 | `{<key>:{$gte:<value>}}` | `db.col.find({"age":{$gte:50}}).pretty()`   | `where age>= 50`         |
| 不等于     | `{<key>:{$ne:<value>}}`  | `db.col.find({"age":{$ne:50}}).pretty()`    | `where age!= 50`         |

```shell
# 分页查询
db.users.find().skip((页码-1) * 每页显示的条数).limit(每页显示的条数)

db.users.find().limit(10) #前10条数据
db.users.find().skip(50).limit(10) #跳过前50条数据，即查询的是第61-70条数据，即第6页的数据


#排序
db.emp.find().sort({sal:1}) #1表示升序排列，-1表示降序排列
db.emp.find().sort({sal:1,empno:-1}) #先按照sal升序排列，如果遇到相同的sal，则按empno降序排列

#注意：skip,limit,sort可以以任意的顺序调用，最终的结果都是先调sort，再调skip，最后调limit
```



### 修改数据

#### update() 方法

update() 方法用于更新已存在的文档。语法格式如下：

```
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   }
)
```

**参数说明：**

- **query** : update的查询条件，类似sql update查询内where后面的。
- **update** : update的对象和一些更新的操作符（如$,$inc...）等，也可以理解为sql update查询内set后面的
- **upsert** : 可选，这个参数的意思是，如果不存在update的记录，是否插入objNew,true为插入，默认是false，不插入。
- **multi** : 可选，mongodb 默认是false,只更新找到的第一条记录，如果这个参数为true,就把按条件查出来多条记录全部更新。
- **writeConcern** :可选，抛出异常的级别。

```
# 1.替换整个文档
# db.collectionName.update(condiction,newDocument)

> db.students.update({name:'zhang'},{name:'kang'})


# 2.修改对应的属性，需要用到修改操作符，比如$set,$unset,$push,$addToSet
db.collectionName.update(
	# 查询条件
	{name:'zhang'},
	{
		#修改对应的属性
		$set:{ 
			name:'kang2',
			age:21
		}
		#删除对应的属性
		$unset:{
			gender:1 //这里的1可以随便改为其他的值，无影响
		}
		
	}
)

# 3.update默认与updateOne()等效，即对于匹配到的文档只更改其中的第一个
# updateMany()可以用来更改匹配到的所有文档
db.students.updateMany(
	{name:'liu'},
	{
		$set:{
			age:21,
			gender:222
		}
	}
)


# 4.向数组中添加数据
db.users.update({username:'liu'},{$push:{"hobby.movies":'movie4'}})

#如果数据已经存在，则不会添加
db.users.update({username:'liu'},{$addToSet:{"hobby.movies":'movie4'}})


# 5.自增自减操作符$inc
{$inc:{num:100}} #让num自增100
{$inc:{num:-100}} #让num自减100
db.emp.updateMany({sal:{$lt:1000}},{$inc:{sal:400}}) #给工资低于1000的员工增加400的工资
```

### 删除数据

```shell
# 1. db.collectionName.remove() 
# remove默认会删除所有匹配的文档。相当于deleteMany()
# remove可以加第二个参数，表示只删除匹配到的第一个文档。此时相当于deleteOne()
db.students.remove({name:'liu',true})

# 2. db.collectionName.deleteOne()
# 3. db.collectionName.deleteMany()
db.students.deleteOne({name:'liu'})

# 4. 删除所有数据：db.students.remove({})----性格较差，内部是在一条一条的删除文档。
# 可直接通过db.students.drop()删除整个集合来提高效率。

# 5.删除集合
db.collection.drop()

# 6.删除数据库
db.dropDatabase()

# 7.注意：删除某一个文档的属性，应该用update。   remove以及delete系列删除的是整个文档

# 8.当删除的条件为内嵌的属性时：
db.users.remove({"hobby.movies":'movie3'})
```

