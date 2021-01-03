0.背景

复杂应用通常有很多层，一个 api 构建在其他 api 上，但是基本思想很简单：每一层通过干净的数据模型来隐藏其底层的复杂性。

数据模型层级结构：

类/对象 -》存储结构 -》序列化字节流 -》 存储介质

本章仅讨论不同的数据表达模式下使用的差别。

# 1.关系型 vs 非关系型

## 1.1 数据的表示

以Bill Gates的个人简介为例：

![BillGate-Profile](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/BillGates-Profile.png)

<center>图 1 - Bill Gates个人简介</center>

* 关系型

  ![BillGates-Relational](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/BillGates-Relational.png)

  在关系型模型设计规范中，每个表存储一个对象相关的属性，表之间通过外键关联。

* 非关系型

  文档模型的表达

  ```
  {
  	"user_id": 251,
  	"first_name": "Bill",
  	"last_name": "Gates",
  	"summary": "Co-chair of .... Active blogger"
  	"positions": [{
  		"job_title": "Co-chair",
  		"organization": "Bill & Melinda Gates F..."
  	},{
  		"job_title": "Co-founder, Chairman",
  		"organization": "Microsoft"
  	}],
  	"industry_id": 131
  	....
  }
  ```

关系型需要将对象转换成数据库模型的表、行和字段，并催生出了ORM框架；而文档型更贴近原生对象的数据模型，其实就是对象的JSON模型（但是带来了其他问题，与第四章讨论）。

数据模型相近使得非关系型数据库可以方便地修改对象属性，甚至是拆分属性；对于一对多关系中，非关系型模型表示极大减少了复杂度。

## 1.2 多对一和多对多的关系

在上一节的例子中，使用外键时，对于变动不大的表而言（比如公司、学校、地点、职位），我们可以称为字典表。

而通过外键方式关联这些对象时，即使这些对象的信息被更新，完全不需要改动主体对象属性字段。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/many-to-many.png" alt="many-to-many" style="zoom:80%;" />

当我们需要查询曾经就读于某一个学校或公司时：

关系型数据模型中，因为天然支持数据聚合，可以方便地查出结果；

大多非关系型数据库不支持聚合，则可能需要在业务代码里面产生多次查询来模拟数据聚合。

## 1.3 Schema的灵活性

多数文档型数据库，因为是schema-on-read，意味着可以写入任意的key-value到文档中；文档数据库并非没有schema，只是schema内置于代码中，所以client在读取数据时，需要保证文档中存在这个key。

关系型数据库则是schema-on-write，在写入数据前就需要利用对象与表格模型映射关系进行转换。



# 2. 查询语言

<font color='red'>有点复杂，后续待补充</font>

