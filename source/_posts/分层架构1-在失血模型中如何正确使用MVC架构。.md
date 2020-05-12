---
title: '分层架构1:在失血模型中如何正确使用MVC架构。'
categories:
  - 编程思想
tags:
  - 重构
  - 分层架构
abbrlink: 67dcf0ad
date: 2020-05-05 21:18:35
---

###  什么是贫血模型？

也许你是第一次听说贫血模型，但我想你对它一定不陌生。我们日常开发过程中使用最多的就是贫血模型。

所谓贫血模型是指，将数据与操作分离，它破坏了面向对象的封装特性，是一种典型的面向过程的编程风格。

```java

////////// Controller+VO(View Object) //////////
public class UserController {
  private UserService userService; //通过构造函数或者IOC框架注入
  
  public UserVo getUserById(Long userId) {
    UserBo userBo = userService.getUserById(userId);
    UserVo userVo = [...convert userBo to userVo...];
    return userVo;
  }
}

public class UserVo {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
}

////////// Service+BO(Business Object) //////////
public class UserService {
  private UserRepository userRepository; //通过构造函数或者IOC框架注入
  
  public UserBo getUserById(Long userId) {
    UserEntity userEntity = userRepository.getUserById(userId);
    UserBo userBo = [...convert userEntity to userBo...];
    return userBo;
  }
}

public class UserBo {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
}

////////// Repository+Entity //////////
public class UserRepository {
  public UserEntity getUserById(Long userId) { //... }
}

public class UserEntity {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
}
```



上面的代码示例中，UserEntity 和 UserRepository 组成了数据访问层，UserBo 和 UserService 组成了业务逻辑层，UserVo 和 UserController 在这里属于接口层。

从代码中，我们可以发现，UserBo 是一个纯粹的数据结构，只包含数据，不包含任何业务逻辑。业务逻辑集中在 UserService 中。我们通过 UserService 来操作 UserBo。换句话说，Service 层的数据和业务逻辑，被分割为 BO 和 Service 两个类中。像 UserBo 这样，只包含数据，不包含业务逻辑的类，就叫作贫血模型（Anemic Domain Model）。



### 什么是MVC架构？

MVC指的Model、View、Controller即模型、视图、控制器，MVC 三层开发架构是一个比较笼统的分层方式，落实到具体的开发层面，很多项目也并不会 100% 遵从 MVC 固定的分层方式，而是会根据具体的项目需求，做适当的调整。比如，现在很多 Web 或者 App 项目都是将后端项目分为 Repository 层、Service 层、Controller 层。其中，Repository 层负责数据访问，Service 层负责业务逻辑，Controller 层负责暴露接口。



### 如何维护失血模型中MVC架构下的代码质量

在讨论如何维护代码质量之前我们先明确一下什么是代码质量。关于代码质量，通常情况下包含但不限于代码的可维护性、可读性、可复用性、可测试性。实际上代码质量的高低是主观印象，业内还有很多词语用来评价一段代码质量的好坏，不同的人对同一段代码的评价也有可能是不一样的。所以下面的讨论仅是本人对MVC架构下代码质量的一些思考。

#### Repository 层

repository层也称为持久层，顾名思义该层主要是为了数据的持久化。实际开发过程中主要是与关系型数据库打交道。它不应该包含实际的业务逻辑，代码简单，所以这层我们应该最大限度的保证代码的可复用性。下面是本人总结的关于repository层的编码规范。

##### 1、查询请求应尽量返回整张表的所有字段。

在实际开发过程中，对同一张表，不用的页面需求可能要展示的字段是不一样的。业务需求总是多变的，如果我们只返回前端需要的字段会导致Repository层方法的复用性很低，增加不必要的工作量。

我们再看看下面这段代码

```java


public class UserEntity {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
  private String email;
}

////////// Repository+Entity //////////
public class UserRepository {
  //实际的sql是：select id，name，cellphone from user where id = ？
  public UserEntity getUserById(Long userId); 
}

public class UserService {
  private UserRepository userRepository; //通过构造函数或者IOC框架注入
  
  private EmailSender emailSender;//通过构造函数或者IOC框架注入
  
  public void sendEmail(Long userId,String content) {
    UserEntity userEntity = userRepository.getUserById(userId);
    emailSender.send(userEntity.getEmail(),content);
  }
}
```



如上代码会因为userEntity.getEmail() 为null而导致邮件发送失败。我们看下getUserById()方法有什么问题：

+ 没有查询整表却用UserEntity封装。
+ 方法名上也没有体现没有查询某个字段。

如果Repository层都这么写意味着我们无法通过方法名+入参+出参直接判断这方法是不是我们想要的，我们每次使用前都得去查看该方法的实现逻辑。这无疑会降低我们的工作效率。

事实上目前最流行的mysql的innodb引擎的数据存储结构是B+树。表中所有字段的数据都会存在叶子结点，这意味着当我们查寻的字段里包含非索引字段时，查出所有字段和查某几个字段的查询效率是一致的。无非是查所有字段的IO流更大些，但这几乎可以忽略不计。

所以在查询过程中如果包含非索引字段请返回整表数据以提高代码的复用性。

##### 2、可以适当的开放查询条件以提高代码的复用度。

除了返回数据查询参数也会影响我们代码的复用性，比如下面这段代码

```java
public class UserEntity {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
  private String email;
}

////////// Repository+Entity //////////
public class UserRepository {
 
  public UserEntity getUserById(Long userId); 
  
  public UserEntity getUserByEmial(String email);
  
  public UserEntity getUserByEmailAndCellphone(String email,String cellphone);
}
```

如上所示，UserRepository的3个方法只是查询条件有些许差别，事实上我们可以通过封装参数的方式把它组合为一个方法。

```java
public class UserRepository {
  public UserEntity getUserByCondition(UserCondition condition);
}

public class UserCondition {
  private Long id;
  private String email;
  private String cellphone;
  private String name;
}
```

```xml
<!-- mybaits 示例，用其他框架也是同样的原理-->
<select id="getUserByCondition"
		parameterType="com.cujia.repository.user.UserCondition"
		resultType="com.cujia.entity.UserCondition">
		SELECT <include refid="table_column_list"></include>
		<where>
			<if test="id != null">
				and id=#{condition.id}
			</if>
			<if test="email != null">
				and email=#{condition.email}
			</if>
			<if test="name != null">
				and name = #{condition.name}
			</if>
			<if test="cellphone != null">
				and cellphone = #{condition.cellphone}
			</if>
		</where>
</select>
```

这样在实际开发过程中可以解决大部分查询需求，能有效的降低我们的工作量，事实上现有的一些mybaits代码生成框架已支持生成上面这种格式的代码。

#### Service 层

在repository-service-controller 这种分层架构中，repository和controller层会很薄，所有的业务逻辑都集中在service层中，代码的腐败往往也是从这层开始的。下面是本人总结的关于service层的编码规范。

##### 1、减少不必要的service类

比如在电商系统中有Order（订单表） 和 OrderItem （订单详情）表结构如下：

```sql
CREATE TABLE `tb_order` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键',
  `total_amount` int NOT NULL COMMENT '金额（分）',
  `buyer_id` bigint NOT NULL COMMENT '买家id',
  `seller_id` bigint NOT NULL COMMENT '卖家ID',
  `receiving_address_id` bigint NOT NULL COMMENT '收获地址ID',
  `status` int DEFAULT NULL COMMENT '订单状态',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='订单';

CREATE TABLE `tb_order_item` (
  `id` bigint DEFAULT NULL COMMENT '主键',
  `order_id` bigint DEFAULT NULL COMMENT '订单ID',
  `product_id` bigint DEFAULT NULL COMMENT '商品ID',
  `product_name` varchar(255) COLLATE utf8_bin DEFAULT NULL COMMENT '商品名称',
  `price` bigint DEFAULT NULL COMMENT '购买单价',
  `quantity` int DEFAULT NULL COMMENT '购买数量'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='订单详情';
```

OrderItem的增删改查所有操作都依赖于对Order的操作，我们不会单独操作OrderItem表，像这种情况我们就没有必要创建一个OrderItemService类，只需要在OrderService中引入OrderItemRepository类对表进行管理就可以了。

##### 2、不要直接依赖不相关的Repository

假设我们现在页面要展示上面的订单信息同时还要包含卖家的名称和logo信息，这些信息保存在一张Shop表中，因为只是一个简单的查询，这时我们直接引入ShopRepository和引入ShopService似乎没什么区别。下面是引入ShopRepository的示例代码。

```java
public class OrderServiceImpl implements OrderService {
  
  @Autowried
  private OrderRepository orderRepository;
  
  @Autowried
  private OrderItemRepository orderItemRepository;
  
  @Autowried
  private ShopRepository shopReopsitory;
  
  public OrderDTO buyerListOrder(Long buyerId){
    List<Order> orders = orderRepository.listByBuyerId(buyerId);
    List<Long> orderIds = getOrderIds(orders);
    List<OrderItem> items = orderItemRepository.listByOrderIds(orderIds);
    //获取卖家信息
    List<Long> shopIds = getShopIds(orderIds);
    List<Shop> shops = shopReopsitory.listByShopIds(shopIds);
    return convert(orders,items,shops);
  }
  
  private List<Long> getOrderIds(List<Order> orders){
    //省略实现过程……
  }
  
  private List<Long> getShopIds(List<Order> orders){
    //省略实现过程……
  }
  
  private List<OrderDTO> convert(List<Order> orders,List<OrderItem> items,List<Shop> shops){
    //省略实现过程……
  }
} 
```

但是如果Shop表中Logo信息的存储方式由原来的绝对路径改为了相对路径，这时数据库里的数据既有在修改之前保存的绝对路径，又有修改之后保存的相对路径。但要求返还给前端的统一为绝对路径。这时如果我们之前引入的是ShopRepository会导致在OrderService中需要处理logo路径相关的业务逻辑。

事实上，对于相关业务模块的依赖，引入Service而非Repository 能够有效的帮我们屏蔽其业务内部的变动而对当前模块造成的影响。下面是引入ShopService的示例代码。

```java
public class OrderServiceImpl implements OrderService {
  
  @Autowried
  private OrderRepository orderRepository;
  
  @Autowried
  private OrderItemRepository orderItemRepository;
  
  @Autowried
  private ShopService shopService;
  
  public OrderDTO buyerListOrder(Long buyerId){
    List<Order> orders = orderRepository.listByBuyerId(buyerId);
    List<Long> orderIds = getOrderIds(orders);
    List<OrderItem> items = orderItemRepository.listByOrderIds(orderIds);
    //获取卖家信息
    List<Long> shopIds = getShopIds(orderIds);
    List<ShopDTO> shops = shopService.listByShopIds(shopIds);
    return convert(orders,items,shops);
  }
  
  private List<Long> getOrderIds(List<Order> orders){
    //省略实现过程……
  }
  
  private List<Long> getShopIds(List<Order> orders){
    //省略实现过程……
  }
  
  private List<OrderDTO> convert(List<Order> orders,List<OrderItem> items,List<ShopDTO> shops){
    //省略实现过程……
  }
} 

public class ShopServiceImpl implements ShopService{
  
  @Autowried
  private ShopRepository shopRepository;
  
  public List<ShopDTO> getByShopIds(List<Long> shopIds){
    List<Shop> shops = shopRepository.getByShopIds(shopIds);
    return convert(shops);
  }
  
  private List<ShopDTO> convert(List<Shop> shops){
    List<ShopDTO> dtos = new ArrayList<>();
    for(Shop shop:shops){
      //省略部分代码……
      String url = getURL(shop.getUri());
      dto.setUrl(url);
      dtos.add(dto);
    }
    return dtos;
  }
  
  private String getURL(String uri){
    //省略实现过程……
  }
}
```

##### 3、service类要与实际业务语言对应起来

一个好的service命名能体现该service的职责边界，让我们明白什么该写在这个service内，什么不能。另一方面service类与实际业务语言对应起来可以使整体团队对同一个业务术语有统一的认识，避免理解的偏差，并将这些“术语”映射到代码中，随着系统的演进变迁。从而达到减少技术人员与非技术人员沟通的成本的目的。下面是service类要与实际业务语言对应起来的示例代码。

```java
//service与实际业务语言对应前
public interface UserService{
  void getByEmail(String email);
  
  void createUser(UserDTO userDto);
  
  void updatePassword(LoginUser user,String oldPassword,String newPassword);
  
  void forgetPassword(String sellphone);
  
  Token login(LoginContext context);
  
  void sendLoginCode(String sellphone);
  
  Long register(RegisterContext context);
  
  void sendRegisterCode(String sellphone);
  //省略其他方法代码……
}

//Service与实际业务语言对应后
//用户服务
public interface UserService{
  void getByEmail(String email);
  
  void createUser(UserDTO userDto);
  
}

public interface LoginUserService{
  void updatePassword(LoginUser user,String oldPassword,String newPassword);
}

//登陆服务
public interface LoginService{
  Token login(LoginContext context);
  
  void forgetPassword(String sellphone);

  void sendCode(String sellphone);
}

//注册服务
public interface RegisterService{
  Long register(RegisterContext context); 
  
  void sendCode(String sellphone);
}
```

在上面的代码中，service未与实际业务语言对应起来前我们通常会把它与表结构对应起来。在简单的业务中我们的需求可能也就是对表的增删改查，这么写并没有什么问题。但是当UserService既有对用户的增删改查功能，也有登陆注册相关功能时，你很难定义这个类究竟是这做什么？似乎与User表相关的它都管。在Service与实际业务语言对应后我们对不同等应用场景和不同的使用者把UserService拆分为UserService、LoginUserService、LoginService、RegisterService，使其职责更单一，代码结构更清晰，更利于维护。

#### Controller层

Controller层作为控制器用来调度View层和Model层，将用户界面和业务逻辑合理的组织在一起，起粘合剂的效果。所以Controller中的内容能少则少，这样才能提供最大的灵活性。

比方说，有一个View会提交数据给Model进行处理以实现具体的行为，View通常不会直接提交数据给Model，它会先把数据提交给Controller，然后Controller再将数据转发给Model。假如此时程序业务逻辑的处理方式有变化，那么只需要在Controller中将原来的Model换成新实现的Model就可以了，**控制器的作用就是这么简单， 用来将不同的View和不同的Model组织在一起，顺便替双方传递消息，仅此而已。**

所以在Controller我们一般只是对入参和用户权限等与业务逻辑无关的事物进行校验（好的权限和参数校验框架也可以在切面层校验，减少controller的代码量）。



### 总结

本文主要是描述了失血模型中MVC架构中的应用，以及如何在失血模型中MVC架构中维护代码的质量，由于每个人对代码质量的评断不尽相同，所以上面的讨论仅是本人对MVC架构下代码质量的一些思考，个人水平有限，如果你有不同意见还请留言探讨。

### 参考

[《深入理解MVC》](https://zhuanlan.zhihu.com/p/35680070)