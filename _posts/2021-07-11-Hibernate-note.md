---
layout:     post
title:      "Hibernate基础以及一些使用总结"
subtitle:   " \"Hibernate的一些基础概念，以及目前项目遇到的小问题\""
date:       2021-07-11 21:10:00
author:     "tablesheep"
catalog: true
tags:
    - Java
    - Hibernate
---

>  “Hibernate 的一些基础记录( •̀ ω •́ )✧”



### 基础概念

> Hibernate基于5.4.x

#### Session[^1]

`org.hibernate.Session`以一个逻辑事务（可以包含多个数据库事务）的开始和结束为一次生命周期。

`Session`并不是线程安全的，每个线程或者事务必须通过`org.hibernate.SessionFactory`获取`Session`

```java
Session session = factory.openSession();
Transaction tx;
try {
     tx = session.beginTransaction();
     //do some work
     ...
     tx.commit();
} catch (Exception e) {
     if (tx!=null) {
     	tx.rollback();
     }
     throw e;
} finally {
     session.close();
}
```

#### 持久化对象的几种状态[^2]

transient（瞬态实例）：普通创建的对象，不被Session管理，不存在于数据库中

persistent（持久实例）：被Session所管理，同时在数据库中有具体的记录

detached（分离实例）：脱离Session管理，存在与数据库中的记录，具备主键

removed（要删除的实例）：被Session所管理，但是计划在数据库中删除的对象（这是后期版本出现的状态，可以理解为调用了delete方法的对象，但是实际还未在数据库中删除）

> 关于transient 与 detached 个人感觉，如果不考虑数据库记录，好像就是是否有id（主键）的区别了

#### 状态之间的转换

transient -> persistent：save()，persist()，saveOrUpdate()，merge()

persistent -> detached：close()，clear()，evict()（推荐）

detached -> persistent ：update()，saveOrUpdate()，lock()，replicate()，merge()

persistent、detached -> removed：delete()  

> 在`org.hibernate.Session`的API中提供了各个状态的切换，基本上对象的状态变化展现了Hibernate的生命周期。

> 需要注意使用`OneToOne`，`OneToMany`等级联关系时，部分状态变换会根据 `CascadeType`的设置对级联对象状态的影响



### 实际应用

以下使用`Department`、`Employee`举例

```JAVA
@Entity(name = "Department")
public static class Department {

    @Id
    private Long id;

    //@OneToMany(mappedBy = "department",fetch = FetchType.LAZY, orphanRemoval = true)
    @OneToMany(fetch = FetchType.LAZY, orphanRemoval = true)
    @JoinColumn(name = "DEPARTMENT_ID", foreignKey = @ForeignKey(value = ConstraintMode.NO_CONSTRAINT))
    private List<Employee> employees = new ArrayList<>();

    //Getters and setters omitted for brevity
}

@Entity(name = "Employee")
public static class Employee {

    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name="DEPARTMENT_ID", foreignKey = @ForeignKey(value = ConstraintMode.NO_CONSTRAINT))
    private Department department;

    //Getters and setters omitted for brevity
}
```



#### OneToMany、ManyToOne等级联问题

使用`OneToMany`、`ManyToMany`、`ManyToMany`的好处是插入、更新、查询时对关联表的一些操作Hibernate自动会处理，但同时也会带来一些问题。

##### OneToMany使用

`OneToMany#mappedBy` 不能与`@JoinColumn`  `@JoinTable`一起使用，否则会报

> Associations marked as mappedBy must not define database mappings like @JoinTable or @JoinColumn

对于`OneToMany`，可以选择不与`@JoinColumn` 联用

```java
Department:

@OneToMany(mappedBy = "department",fetch = FetchType.LAZY, orphanRemoval = true)
private List<Employee> employees = new ArrayList<>();

Employee:

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name="DEPARTMENT_ID", foreignKey = @ForeignKey(value = ConstraintMode.NO_CONSTRAINT))
private Department department;

main:

Department department = new Department();
department.setEmployees(employees);
departmentRepository.sava(department); //此时employee中的department_id 并不会自动帮我们插入

//正确的做法
Department department = new Department();
department.setEmployees(employees);
for (Employee employee : employees) {
    employee.setDepartment(department);
}
departmentRepository.save(department);
```

也可以选择与`@JoinColumn` 联用，此时关联表（子表）中的关联字段会自动帮我们插入，只是插入的方式有些感人...（通过日志它是帮我们插入后再去update），建议还是用`OneToMany#mappedBy` 吧。

```java
Department:

@OneToMany(fetch = FetchType.LAZY, orphanRemoval = true, cascade = CascadeType.ALL) //注意没有mappedBy
@JoinColumn(name = "DEPARTMENT_ID", foreignKey = @ForeignKey(value = ConstraintMode.NO_CONSTRAINT))
private List<Employee> employees = new ArrayList<>();

main:

Department department = new Department();
department.setEmployees(employees);
departmentRepository.sava(department);
```

```log
Hibernate: insert into department (id) values (?)
Hibernate: insert into employee (department_id, id) values (?, ?)
Hibernate: insert into employee (department_id, id) values (?, ?)
Hibernate: update employee set department_id=? where id=?
Hibernate: update employee set department_id=? where id=?
```



##### 懒加载性能问题

```java
//这种情况会出现循环查库的问题
List<Department> departments = load();
for(var department : departments) {
    ...
    var employees = department.getEmployees;
    ...   
}

//解决办法可以通过批量查询子项直接应用或者通过setter方法设置以解决
```



##### persistent 实例级联对象setter注意事项

```
//Department的employees有记录
Department department = load(id);

//这种时候重新setEmployees,在session commit时会产生错误（当然在只有查询时并不会有问题）
//A collection with cascade=\"all-delete-orphan\" was no longer referenced by the owning entity instance
department.setEmployees(employees);

//正确的做法应该是
department.getEmployees.clear();
department.getEmployees.addAll(employees);

//对于性能问题，如果我们既需要调用级联对象的setter,而又处于事务当中,应该通过将其变为detached 状态以避免被session提交修改
session.evict(department);

//需要注意的是被evict的对象如果有关联另外的实体，在evict之前不能调用setter方法,
//否则会报 org.hibernate.AssertionFailure: collection owner not associated with session: xxx
department.setEmployees(employees);
session.evict(department);
```



#### Convert

##### enum

针对枚举，Hibernate默认会使用枚举定义的顺序`Enum#ordinal`进行持久化，这种方式风险极大。

```
//可以使用javax.persistence.Enumerated指定Hibernate对于枚举处理
@Enumerated(EnumType.STRING)

//也可以使用@Convert自定义枚举的转换
@Column(name = "gender")
@Convert(converter = GenderConvert.class)
private GenderEnum gender;

@Converter
public static class GenderConvert implements AttributeConverter<GenderEnum, Integer>{

    @Override
    public Integer convertToDatabaseColumn(GenderEnum attribute) {
        ...
    }

    @Override
    public GenderEnum convertToEntityAttribute(Integer dbData) {
        ...
    }
}
```



### To be continued...

[^1]: Session - <https://github.com/hibernate/hibernate-orm/blob/main/hibernate-core/src/main/java/org/hibernate/Session.java>
[^2]: Persistence Context - <https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#pc>