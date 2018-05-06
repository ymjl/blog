---
layout: "post"
title: "spring data jpa-Specification 多条件模糊查询"
category: springboot
date: "2018-04-18 11:39"
---

## 问题

  app端 可以通过姓名，手机号或身份证号对客户进行模糊搜索。spring data jpa 通过继承CurdRepository预先生成的方法及自定义的接口不能满足此需求。

## 解决

### 通过继承JpaRepository<CustomerEntity,Integer> 以及 JpaSpecificationExecutor

  通过继承JpaRepository 使用其提供的分页方法，通过JpaSpecificationExecutor 实现多条件分页功能。

### 代码

  持久层代码

  ``` java
  public interface CustomerRepository extends JpaRepository<CustomerEntity,Integer> ,JpaSpecificationExecutor {


      Page<CustomerEntity> findAll(Specification spec, Pageable pageable);
  }


  ```


  Server 层代码

  ``` Java

  public Page<CustomerEntity> searchCustomer(String input ,Integer index,Integer size){

      Sort sort = new Sort(Sort.Direction.DESC,"id");
      Pageable pageable = new PageRequest(index,size,sort);


      return customerRepository.findAll((r,c,b)->{
          Path<String> name = r.get("name");
          Path<String> idCard = r.get("idCard");
          Path<String> phone = r.get("phone");

          c.where(b.or(b.like(name,"%"+ input + "%"),b.like(idCard,"%"+ input + "%"),b.like(phone,"%"+ input + "%")));

        return null;
      },pageable);



  }
  ```



  原始代码：

  ``` java
  customerRepository.findAll(new Specification() {
      @Override
      public Predicate toPredicate(Root root, CriteriaQuery criteriaQuery, CriteriaBuilder criteriaBuilder) {
          return null;
      }
  });

  ```
## 分析

### Root

-  Root 定义查询的From子句中能出现的类型
-  Criteria查询的查询根定义了实体类型，能为将来导航获得想要的结果，它与SQL查询中的FROM子句类似。
-  Root实例也是类型化的，且定义了查询的FROM子句中能够出现的类型。
-  查询根实例能通过传入一个实体类型给 AbstractQuery.from方法获得。
-  Criteria查询，可以有多个查询根。
-  CustomerEntity实体的查询根对象可以用以下的语法获得

  ``` java
    Root<CustomerEntity> customer = criteriaQuery.from(CustomerEntity.class);
  ```


### Predicate 过滤条件 CriteriaQuery 接口

代表一个specific的顶层查询对象，它包含着查询的各个部分，比如：select 、from、where、group by、order by等注意：CriteriaQuery对象只对实体类型或嵌入式类型的Criteria查询起作用
提供的方法
``` java
 CriteriaQuery<T> select(Selection<? extends T> var1);

 CriteriaQuery<T> multiselect(Selection... var1);

 CriteriaQuery<T> multiselect(List<Selection<?>> var1);

 CriteriaQuery<T> where(Expression<Boolean> var1);

 CriteriaQuery<T> where(Predicate... var1);

 CriteriaQuery<T> groupBy(Expression... var1);

 CriteriaQuery<T> groupBy(List<Expression<?>> var1);

 CriteriaQuery<T> having(Expression<Boolean> var1);

 CriteriaQuery<T> having(Predicate... var1);

 CriteriaQuery<T> orderBy(Order... var1);

 CriteriaQuery<T> orderBy(List<Order> var1);

 CriteriaQuery<T> distinct(boolean var1);
```

### CriteriaBuilder

  安全查询创建工厂,创建CriteriaQuery，创建查询具体具体条件Predicate 等
