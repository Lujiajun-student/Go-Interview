# 1. MongoDB是什么

MongoDB是文档型的NoSQL数据库。

关系型数据库是Database-Table-Row-Column，MongoDB是Database-Collection-Document-Field。

文档类似JSON，但存储格式是BSON。

BSON是二进制格式，将文本编码成字节流，能够让数据库更快速地处理。

# 2. 基本概念

## 1. Database

数据库是集合的容器。

```js
use shop
```

> 使用shop数据库。如果不存在，就会在写入数据时自动创建。

```js
show shop // 查看数据库
```

```js
db.dropDatabase() // 删除当前数据库
```

## 2. Collection

集合类似表，但没有固定结构。

```js
db.createCollection("users")
```

上面创建了users表，但没有固定的结构，结构根据插入的文档来判断。

```js
db.users.insertOne({
    username: "alice",
    age: 22
})
```

集合应该根据对象的不同来分开存放。如users、orders、products等。

## 3. Documents

文档是基本的数据单位，相当于一行记录。

```js
{
  _id: ObjectId("..."),
  orderNo: "O202606170001",
  userId: ObjectId("..."),
  status: "PAID",
  amount: 299.00,
  items: [
    {
      productId: ObjectId("..."),
      productName: "机械键盘",
      price: 199.00,
      quantity: 1
    },
    {
      productId: ObjectId("..."),
      productName: "鼠标垫",
      price: 100.00,
      quantity: 1
    }
  ],
  receiver: {
    name: "张三",
    phone: "13800000000",
    address: "广州市南沙区..."
  },
  createdAt: ISODate("2026-06-17T10:00:00Z")
}
```

这个文档保存了订单商品和收货地址。将一次业务操作需要的数据放到同一个文档中。

## 4. Field

字段相当于文档的属性。字段可以是简单类型，或者对象、数组。

## 5. id

每个文档都有一个_id字段作为主键。如果不指定，那么就会自动生成ObjectId。

ObjectId包含时间戳、随机值等信息，可以按照生成事件来排序。