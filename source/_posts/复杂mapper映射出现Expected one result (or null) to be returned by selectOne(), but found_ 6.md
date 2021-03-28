---
title: Mybatis复杂mapper映射出现Expected one result (or null) to be returned by selectOne(), but found:6
date: 2021-2-1
tags:
	- Mybatis
categories:
	- Spring
---

> 今天在项目使用mybatis写映射文件时，做测试发现报错`Expected one result (or null) to be returned by selectOne(), but found: 6`，下面记录一下解决的过程：

![image.png](1612149739610-d371499a-4d5a-4ba7-9b0d-8f9b8bc133ec.png)
### <font color="orange">1. 问题介绍：</font>
<!--more-->
<font size=3>**对应的表关联关系如下：**</font>

#### <img src="1612149616517-3d3f353b-1319-4315-a12a-3592c873810c.png" alt="image.png" style="zoom:50%;" />

其中tb_product与tb_product_img是一对多的关系，外键关联为product_id字段，其他表之间是一一对应关系，关联外键如表中灰色字段所示。

<font size=3>**dao层接口：**</font>

通过给定的productId查询出该product的信息，包括商品的所在分类，商品所属店铺信息，以及商品的详情图（一对多）
```java
Product getProductById(long productId);
```
<font size=3>**实体类对象：**</font>

```java
public class Product {
    private Long productId;
    private String productName;
    private String productDesc;
    private String imgAddr; //简略图
    private String normalPrice; //日常价格
    private String promotionPrice; //促销价格
    private List<ProductImg> productImgList; //商品详情图列表
    private ProductCategory productCategory;
    private Shop shop;
    // 管理信息
    private Integer priority;
    private Date createTime;
    private Date lastEditTime;
    private Integer enableStatus; //商品状态，0表示下架不可用，1表示上架
}
public class ProductImg {
    private Long productImgId;
    private String imgAddr;
    private String imgDesc;
    private Integer priority;
    private Date createTime;
    private Long productId;
}
public class ProductCategory {
    private Long productCategoryId;
    private Long shopId; // 查询商品类别时，不太需要Shop实体类对，所以这里使用ShopId
    private String productCategoryName;
    private Integer priority;
    private Date createTime;
}
public class Shop {
    private Long shopId;
    // 店铺基本信息
    private String shopName;
    private String shopDesc;
    private String shopAddr;
    private String phone;
    private String shopImg;
    // 管理信息
    private Integer priority;
    private Date createTime;
    private Date lastEditTime;
    private Integer enableStatus; //-1：不可用，0：审核中 1：可用
    private String advice; //超级管理员给店家的提醒
    // 外键信息
    private Area area; //外键区域id
    private ShopCategory shopCategory; //外键类别id
    private PersonInfo owner; //店家
}
public class PersonInfo {
    private Long userId; //用户id，用Long表示
    private String name;
    private String profileImg;
    private String email;
    private String gender;
    private Integer enableStatus; //用户状态，标识是否黑名单
    private Integer userType; //1顾客 2店家 3超级管理员
    private Date createTime;
    private Date lastEditTime;
}
```
<font size=3>**对应的mapper映射文件文件：**</font>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org/DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.imooc.o2o.dao.ProductDao">
  	<!--对于冲突名称使用SQL中指定的别名，对于owner_id指定为owner.userId，意思为shop中名为owner的属性，对owner中的userId进行设置字段owner_id的值-->
    <resultMap id="productMap" type="com.imooc.o2o.entity.Product">
        <id property="productId" column="product_id"/>
        <result property="productName" column="product_name"/>
        <result property="productDesc" column="product_desc"/>
        <result property="imgAddr" column="img_addr"/>
        <result property="normalPrice" column="normal_price"/>
        <result property="promotionPrice" column="promotion_price"/>
        <result property="priority" column="product_priority"/>
        <result property="createTime" column="product_create_time"/>
        <result property="lastEditTime" column="product_last_edit_time"/>
        <result property="enableStatus" column="product_enable_status"/>
        <association property="productCategory" javaType="com.imooc.o2o.entity.ProductCategory">
            <id property="productCategoryId" column="product_category_id"/>
            <result property="productCategoryName" column="product_category_name"/>
            <result property="priority" column="product_category_priority"/>
            <result property="createTime" column="product_category_create_time"/>
            <result property="shopId" column="product_category_shop_id"/>
        </association>
        <association property="shop" javaType="com.imooc.o2o.entity.Shop">
            <id property="shopId" column="shop_id"/>
            <result property="owner.userId" column="owner_id"/>
            <result column="shop_name" property="shopName" />
            <result column="shop_desc" property="shopDesc" />
            <result column="shop_addr" property="shopAddr" />
            <result column="phone" property="phone" />
            <result column="shop_img" property="shopImg" />
            <result column="shop_priority" property="priority" />
            <result column="shop_create_time" property="createTime" />
            <result column="last_edit_time" property="lastEditTime" />
            <result column="shop_enable_status" property="enableStatus" />
            <result column="advice" property="advice" />
        </association>
        <collection property="productImgList" ofType="com.imooc.o2o.entity.ProductImg">
            <id property="product_img_id" column="productImgId"/>
            <result property="imgAddr" column="img_addr"/>
            <result property="priority" column="priority"/>
            <result property="createTime" column="product_img_create_time"/>
        </collection>
    </resultMap>

  	<!--SQL中使用as对冲突名进行重命名-->
    <select id="getProductById" resultMap="productMap">
        select p.product_name,p.product_desc,p.img_addr,p.normal_price,p.priority,p.create_time as product_create_time,
               p.last_edit_time as product_last_edit_time,p.enable_status as product_enable_status,
               pc.product_category_id,pc.product_category_name,pc.priority as product_category_id,pc.create_time as product_category_create_time,
               pc.shop_id as product_category_shop_id,
               s.shop_id,s.shop_name,s.shop_desc,s.phone,s.create_time as shop_create_time,s.owner_Id,pi.product_img_id,pi.priority,pi.img_addr,pi.img_desc,pi.create_time as product_img_create_time
               from tb_product p,tb_product_category as pc,tb_shop as s,tb_product_img as pi where
               p.product_category_id = pc.product_category_id and p.shop_id = s.shop_id and p.product_id = pi.product_img_id
               and p.product_id = #{productId};
    </select>
</mapper>
```

<font size=3>**对应SQL在数据库中查询结果**：</font>

![image.png](1612150436476-e7e658e8-81a4-4292-9b5b-9c00ab6b823f.png)

上述可以看见product中包含了三个实体类对象：ProductCategory，Shop，List\<ProductImg\>，而Shop中又有owner实体对象，如果根据一个product_id找到相应的product，要求有实体类对象ProductCategory，Shop，List\<ProductImg\>，同时Shop属性中还需要有Owner实体类，应该怎么做？



### <font color="orange">2. 错误分析：</font>

**1. 首先查询的SQL语句有很多重名字段，例如priority在好几个类里面都有，lastEditTime也同样如此，如何区分字段并把查询出来的字段这些分配到正确的属性中去呢？**

解决办法：在SQL语句中使用AS关键词来给有冲突的列名进行重新命名，同时在ResultMap中写入的Column就是重新命名过的名字，不这样做会造成mybatis遇到重名不知道怎么放，可能会造成查询的lastEditTime是一致的

**2. 查询Shop表中有owner_id字段，查询的时候如何给复杂属性导航，给product的成员变量shop对象中的Owner成员变量设置userId呢？**

通过查阅官方文档的方法：映射到复杂对象的嵌套属性时，可以使用点式分隔方法进行复杂属性的导航，这里就可以使用owner.userId来进行复杂导航
![image.png](1612109243469-4d89a31b-0d9d-487e-bfed-faf3497e774e.png)

**3. <font color="red"><u>查询Product对象中有productImgList属性，是一个list，如何把SQL查出来多条结果的属于同一个ProductId的ProductImg放入到List中呢？</u></font>**

- 如果`collection`中没有指定`column`字段，那么会默认通过resultMap的`id`标签来判断（这里的id就是product_id）查询的多条结果，如果查询多条结果的product_id是相同的，那么应该放入到List\<ProductImg\>中，Product不会增加；如果查询的product_id是不同的，那么就会返回多条结果，这里应该就返回List\<Product\>
- 如果`collection`标签通过指定`column`为product_id或者shop_id（只要是数据库多条结果中有一个字段是相同可以归类为相同就行），mybatis会知道根据column给定的product_id检索SQL查询出来的多条结果，如果多条结果的product_id是一样的，那么查询出来的多条结果中应该放入到List\<ProductImg\>中，Product不会增加），如果不指定column字段并且查询结果中也没有指定的id标签字段（这里为product_id），那么查询出来的多条结果中不会放入到List\<ProductImg\>中，而是返回多条语句的查询结果List\<Product\>，而dao层接口接收的返回对象只有Product，这就是出现了`Expected one result (or null) to be returned by selectOne(), but found: 6`的原因。

> 根据作者本人的解释, MyBatis为了降低内存开销,采用ResultHandler逐行读取的JDBC ResultSet结果集的,这就会造成MyBatis在结果行返回的时候无法判断以后的是否还会有这个id的行返回,所以它采用了一个方法来判断当前id的结果行是否已经读取完成,从而将其加入结果集List,这个方法是:
>
> 1.读取当前行记录A,将A加入自定义Cache类,同时读取下一行记录B
>
> 2.使用下一行记录B的id列和值为key(这个key由resultMap的\<id\>标签列定义)去Cache类里获取记录
>
> 3.假如使用B的key不能够获取到记录,则说明B的id与A不同,那么B将被加入到List（所以这里应该会返回List\<Product>\，出现报错）
>
> 4.假如使用B的key可以获取到记录,说明A与B的id相同,则会将A与B合并(相当于将两个productImg合并到一个List中,而product本身并不会增加)
>
> 5.将B定为当前行,同时读取下一行C,重复1-5,直到没有下一行记录
>
> 6.当没有下一行记录的时候,将最后一个合并的resultMap对应的java对象加入到List(最后一个被合并goodsImg的Goods)
> 所以
>   a. 当结果行是乱序的,例如BBAB这样的顺序,在记录行A遇到一个id不同的曾经出现过的记录行B时, A将不会被加入到List里(因为Cache里已经存在B的id为key的cahce了)
>   b. 当结果是顺序时,则结果集不会有任何问题,因为 记录行 A 不可能 遇到一个曾经出现过的 记录行B, 所以记录行A不会被忽略,每次遇到新行B时,都不可能使用B的key去Cache里取到值,所以A必然可以被加入到List

在Mybatis中代码如下：

```java
@Override
  protected void handleRowValues(ResultSet rs, ResultMap resultMap, ResultHandler resultHandler, RowBounds rowBounds, ResultColumnCache resultColumnCache) throws SQLException {
    final DefaultResultContext resultContext = new DefaultResultContext();
    skipRows(rs, rowBounds);
    Object rowValue = null;
    while (shouldProcessMoreRows(rs, resultContext, rowBounds)) {
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rs, resultMap, null);
      // 下一记录行的id构成的cache key
      final CacheKey rowKey = createRowKey(discriminatedResultMap, rs, null, resultColumnCache);
      Object partialObject = objectCache.get(rowKey);
      // 判断下一记录行是否被记录与cache中,如果不在cache中则将该记录行的对象插入List
      if (partialObject == null && rowValue != null) { // issue #542 delay calling ResultHandler until object 
        if (mappedStatement.isResultOrdered()) objectCache.clear(); // issue #577 clear memory if ordered
        callResultHandler(resultHandler, resultContext, rowValue);
      } 
      // 当前记录行的值
      rowValue = getRowValue(rs, discriminatedResultMap, rowKey, rowKey, null, resultColumnCache, partialObject);
    }
    // 插入最后一记录行的对象到List
    if (rowValue != null) callResultHandler(resultHandler, resultContext, rowValue);
  }
```

### <font color="orange">3. 解决</font>

通过上述分析，知道了问题所在，select标签中SQL语句既没有查询对应的product的id标签字段product_id也没有在collection中通过显式地指定column字段表明应该根据什么字段将多条查询结果应该归为到一个List\<ProductImg\>中，而不是List\<Product\>中

提供一下两种解决办法（二者选其一）：

a. 在select标签中加入id标签的字段product_id

```SQl
select p.product_id,p.product_name,p.product_desc,.......
```

b. 在collection中指定column字段（采用可以表示为多条结果相同归类即可，这里可以采用shop_id或者product_id，不能采用produt_img_id）

```java
 <collection property="productImgList" column='product_id'ofType="com.imooc.o2o.entity.ProductImg">
```



### <font color="orange">4.总结</font>

利用collection进行一对多关系查询时，要么SQL中select出resultMap中指定id字段（这时不用指定collection的column字段，mybatis会more按照id字段判断查询结果是否归类到一个list中）；要么显式地在colleciton标签指定column字段，mybatis会根据指定的column字段，查询的多条结果如果指定的column字段相同，那么mybatis会把它们都放入到collection指定的property中，如果不指定或者指定的字段在查询的多条结果中不是相同的或者SQL中没有查出该字段，那么mybatis会认为是多条结果，分别封装成一个对象，那么查询结果就不是1个，而是多个，就会出现报错Expected one result (or null) to be returned by selectOne(), but found: 6










