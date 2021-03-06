---
layout: "post"
title: "Spring Data JPA 分页之Page接口 List到Page转换"
category: "springboot"
date: "2018-04-27 17:57"
---



  前端需要的数据需要后端从几个不同的表整合到一个dto中,有两种解决方案:

    1. 多表联合查询,spring data jpa要进行复杂的查询需要在持久层继承JpaSpecificationExecutor接口.关于这个接口下面放一个例子,是我在项目中提供app多条件模糊查询使用的代码.参考清单1

    2. 在server层进行关联,这样可以充分利用缓存,性能更高,提高可扩展性.但利用jpa分页查询会遇到类型转换的问题,jpa从数据库分页查出的是Page<Object> 格式的，需要封装一个dto 并 把分页信息填充进去 转换成Page<Dto> 。



清单 1
 ```java
    //server层代码
    public Page<CustomerEntity> searchCustomer(String input, Integer index, Integer size) {


        Sort sort = new Sort(Sort.Direction.DESC, "id");
        Pageable pageable = new PageRequest(index, size, sort);
        //Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb); Root 可以实现 多表联合查询
        Page<CustomerEntity> result = customerRepository.findAll((r, c, b) -> {
            Path<String> name = r.get("name");
            Path<String> idCard = r.get("idCard");
            Path<String> phone = r.get("phone");
            c.where(b.or(b.like(name, "%" + input + "%"), b.like(idCard, "%" + input + "%"), b.like(phone, "%" + input + "%")));

            return null;
        }, pageable);
        for (CustomerEntity entity :
                result) {
            entity.setIdCard(StringUtil.idCardFilter(entity.getIdCard()));
        }

        return result;


    }

    //持久层代码
    public interface CustomerRepository extends JpaRepository<CustomerEntity,Integer> ,JpaSpecificationExecutor {


    Page<CustomerEntity> findAll(Specification spec, Pageable pageable);
}

 ```


## Page 源代码

 Page 是个接口，通过查看源代码

1.1 Page.java
 ```Java

 public interface Page<T> extends Slice<T> {
     int getTotalPages();  //得到总页数

     long getTotalElements();//得到总个数

     <S> Page<S> map(Converter<? super T, ? extends S> var1);  //通过提供数据转换方法，可以对数据进行转换  比如 Page<T1Dto> 到 Page<T2Dto>
 }

 ```
这个接口比较简单，看实现Page接口的实现类

PageImpl.java
```java
public class PageImpl<T> extends Chunk<T> implements Page<T> {
    private static final long serialVersionUID = 867755909294344406L;
    private final long total;
    private final Pageable pageable;

    public PageImpl(List<T> content, Pageable pageable, long total) {
        super(content, pageable);
        this.pageable = pageable;
        this.total = !content.isEmpty() && pageable != null && (long)(pageable.getOffset() + pageable.getPageSize()) > total ? (long)(pageable.getOffset() + content.size()) : total;
    }

    public PageImpl(List<T> content) {
        this(content, (Pageable)null, null == content ? 0L : (long)content.size());
    }

    public int getTotalPages() {
        return this.getSize() == 0 ? 1 : (int)Math.ceil((double)this.total / (double)this.getSize());
    }

    public long getTotalElements() {
        return this.total;
    }

    public boolean hasNext() {
        return this.getNumber() + 1 < this.getTotalPages();
    }

    public boolean isLast() {
        return !this.hasNext();
    }

    public <S> Page<S> map(Converter<? super T, ? extends S> converter) {
        return new PageImpl(this.getConvertedContent(converter), this.pageable, this.total);
    }

}

```

这个实现类 继承了 Chunk<T>类 ，实现了 Page接口。

通过看实现类的代码，发现他提供 public PageImpl(List<T> content)的构造方法。
所以很简单的 把 自己的 List<Dto> 放进去就可以得到一个 实习Page接口对象。包含了分页常用的数据。

再看他实现了 Chunk 类 再看其源代码

```java  
abstract class Chunk<T> implements Slice<T>, Serializable {
    private static final long serialVersionUID = 867755909294344406L;
    private final List<T> content = new ArrayList();
    private final Pageable pageable;

    public Chunk(List<T> content, Pageable pageable) {
        Assert.notNull(content, "Content must not be null!");
        this.content.addAll(content);
        this.pageable = pageable;
    }

    public int getNumber() {
        return this.pageable == null ? 0 : this.pageable.getPageNumber();
    }

    public int getSize() {
        return this.pageable == null ? 0 : this.pageable.getPageSize();
    }

    public int getNumberOfElements() {
        return this.content.size();
    }

    public boolean hasPrevious() {
        return this.getNumber() > 0;
    }

    public boolean isFirst() {
        return !this.hasPrevious();
    }

    public boolean isLast() {
        return !this.hasNext();
    }

    public Pageable nextPageable() {
        return this.hasNext() ? this.pageable.next() : null;
    }

    public Pageable previousPageable() {
        return this.hasPrevious() ? this.pageable.previousOrFirst() : null;
    }

    public boolean hasContent() {
        return !this.content.isEmpty();
    }

    public List<T> getContent() {
        return Collections.unmodifiableList(this.content);
    }

    public Sort getSort() {
        return this.pageable == null ? null : this.pageable.getSort();
    }

    public Iterator<T> iterator() {
        return this.content.iterator();
    }

    protected <S> List<S> getConvertedContent(Converter<? super T, ? extends S> converter) {
        Assert.notNull(converter, "Converter must not be null!");
        List<S> result = new ArrayList(this.content.size());
        Iterator var3 = this.iterator();

        while(var3.hasNext()) {
            T element = var3.next();
            result.add(converter.convert(element));
        }

        return result;
    }

}
```

可以看到他是一个抽象类，PageImpl的构造方法调用了这个抽象类的构造方法,把内容放到属性中。
他实现了 Slice<T> 接口的 一些分页的常用方法。


所以自己写的工具类可以很简单的实现

```java  
/**
 * @program: water
 * @description: 用于List<Object> 转换成 Page<Object>
 * @author: SongWanqi
 * @create: 2018-04-27 17:36
 **/
public class PageUtil {

    public static <T> Page<T> ListToPage(List<T> list){

        Page<T> page = new PageImpl<T>(list);

        return page;

    }
}


```

**如上转换 会造成有些分页数据出问题,真正的解决方案如下**

比如：
"last": true,
"totalPages": 1,
"totalElements": 7,
"number": 0,
"size": 10
这些分页数据只是当前查出的的数据进行分页，会总是最后一页第一页,几个属性会以查出的数据为基础，所以这种方式不可取。



```java  
public <S> Page<S> map(Converter<? super T, ? extends S> converter) {
    return new PageImpl(this.getConvertedContent(converter), this.pageable, this.total);
}
```  
Chunk 类中这个方法为了解决用框架查出的Page<Entity> 转换 Page<Dto> 问题。
传进去一个实现 Converter接口能实现两个类之间的转换。

示例：
 ```java
 public Page<OrderDto> getOrderItemsByCustId(Integer custId, Integer page,Integer rows){

     Sort sort = new Sort(Sort.Direction.DESC, "created");
     Pageable pageable = new PageRequest(page-1, rows, sort);
     Page<OrdersEntity> orders = ordersRepository.findAllByCustId(custId,pageable);

     Page<OrderDto> result = orders.map((x)-> initOrderDto(x)); //放入一个Converter的实现类

     return result;

 }

 public OrderDto initOrderDto(OrdersEntity entity){

         OrderDto temp = new OrderDto();
         CustomerEntity customer = customerService.selectOneById(entity.getCustId());
         DepositEntity deposit = entity.getType() == 1?null:depositRepository.findOne(entity.getId());
         WaterKeyEntity newKey = waterKeyRepository.findOne(entity.getKeyId());
         WaterKeyEntity oldKey = entity.getType() == 1?waterKeyRepository.findOne(entity.getOldKeyId()):null;
         temp.setCustomer(customer);
         temp.setDeposit(deposit);
         temp.setOldKey(oldKey);
         temp.setNewKey(newKey);
         temp.setOrder(entity);
         temp.setNewKeyType(keyTypeRepository.findOne(newKey.getTypeId()));
         //temp.setWatermeter();
     return temp;
 }
 ```
