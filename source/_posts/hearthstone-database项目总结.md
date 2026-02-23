---
title: hearthstone database项目总结
date: 2024-08-27 14:19:50
tags:
  - Spring Boot
  - Java
  - MyBatis
  - MySQL
categories:
  - 实战项目
cover: https://pics.findfuns.org/hearthstone.jpg
---



## 写在前面

这是一个前后端分离的全栈开发项目。后端采用的技术栈是SpringBoot + MyBatis + MySQL。前端采用的是vue3框架。在项目完成后部署在了AWS云服务上，OS采用的是ubuntu并使用nginx进行了反向代理。

整个项目从收集数据集，清洗数据，整合数据库开始，一直到最终部署在服务器上历时大约10天。其实前后端开发占用时间并不是很长，也没有用到很复杂的技术，主要的时间消耗在于数据库的清洗和整合工作。元数据来自一个第三方的hearthstone网站提供的API(在此非常感谢)，但在获取元数据之后发现并不能立刻投入使用，因为存在很多脏数据和一些冗余数据，并且一些数据还存在格式上的不匹配。把这些问题处理完毕耗费了大约3-4天的时间。

本文章用于总结和回顾整个项目开发过程中遇到的问题和主要的工作量。

先贴一张IDEA的项目结构图

<img src="https://pics.findfuns.org/backend-overview.png" alt="back" style="zoom:33%;" />

整个后端项目采用的是经典的SpringMVC框架,即Controller-Service-Mapper三层。同时也设计了Pojo类以及自定义`Typehandler`，自定义`TypeHandler`用于处理数据库查询结果和Java实例之间的映射（当映射关系没有默认处理时需要设计自定义`TypeHandler`）。

### POJO

先来讲一讲pojo类。为了契合卡牌的不同类型和各种属性，pojo类进行了详细的设计，包括继承关系，枚举类，数据类型等。

<img src="https://pics.findfuns.org/pojo-hierarchical-diagram.png" alt="z" style="zoom:33%;" />

`Card`类是所有卡牌的基类，所有类型的卡牌都是`Card`类的子类。`Card`类也包含了所有卡牌都有的、最基本的几个属性。

```java
package com.zzb.hearthstoneDB.pojo;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Card {

    private String id; // 每张卡牌的唯一id

    private String name; // 卡牌名称

    private Integer cost; // 卡牌费用

    private CardClass cardClass; // MAGE, DRUID, PRIEST ...

    private Integer cardSet; // 一个整数，对应着一个版本

    private String rule; // 卡牌描述
}
```

Card共有6个子类： `Spell`， `Hero`， `HeroPower`， `Location`， `Weapon`， `Minion`。分别对应6种不同的卡牌类型： 法术， 英雄牌， 英雄技能， 地标， 武器， 随从。

例如`Minion`类的设计

```java
package com.zzb.hearthstoneDB.pojo;

@Data
@NoArgsConstructor
public class Minion extends Card{
    public Minion(String id, String name, Integer cost, CardClass cardClass, Integer cardSet,
                  String rule, Integer attack, Integer health, Rarity rarity, Race race, String flavor) {
        super(id, name, cost, cardClass, cardSet, rule);
        this.attack = attack;
        this.health = health;
        this.rarity = rarity;
        this.race = race;
        this.flavor = flavor;
    }

    private Integer attack; // 攻击力

    private Integer health; // 生命值

    private Rarity rarity; // 稀有度

    private Race race; // 种族

    private String flavor; // 简介
}
```

对于几个属性值，`Rarity`， `Race`，`CardClass`，`SpellSchool`，他们都有一些固定的值的集合，非常适合采用枚举类来表示。

例如`SpellSchool`类的设计

```java
package com.zzb.hearthstoneDB.pojo;

public enum SpellSchool {

    ARCANE, // 奥术
    FIRE, // 火焰
    FROST, // 冰霜
    NATURE, // 自然
    SHADOW, // 暗影
    HOLY, // 神圣
    FEL, // 邪能
}
```

### Controller

Controller层主要的逻辑是路由规划并从URL中获取查询参数并将参数传递给Service层。

```java
package com.zzb.hearthstoneDB.controller;

@RestController
@RequestMapping("cards/api")
@CrossOrigin
public class CardController {

    private final CardService cardService;

    @Autowired
    public CardController(CardService cardService) {
        this.cardService = cardService;
    }
```

`@RestController`是一个复合注解，包括了`@Controller`和`@ResponseBody`这两个注解。`@Controller`注解将类标记为SpringMVC的Controller，`@ResponseBody`注解指示方法的返回值直接写入响应体而不是视图，并以JSON的格式返回。

同时使用`@Autowired`注解自动注入Service组件。

使用`@RequestMapping`注解规定根路由。注意，**根路由的开头是不带’/‘的**，这与每个方法上的子路由是不同的。

Controller中的方法的设计大致相同，如下

```java
@GetMapping("/minion") // 字路由的开头带‘/’
public List<Card> selectMinions(
  @RequestParam(value = "name", required = false) String name, 
  @RequestParam(value = "cost", required = false) Integer cost,
  @RequestParam(value = "attack", required = false) Integer attack, 
  @RequestParam(value = "health", required = false) Integer health,
  @RequestParam(value = "rarity", required = false) String rarity, 
  @RequestParam(value = "race", required = false) String race,
  @RequestParam(value = "cardClass", required = false) String cardClass,         		       	 @RequestParam(value = "cardSet", required = false) Integer cardSet,
  @RequestParam(value = "rule", required = false) String rule) {

    return cardService
      .selectMinions(name, cost, attack, health, rarity, race, cardClass, cardSet, rule);
}
```

`@GetMapping`规定这个方法用于处理一个get请求并说明了对应的路由。

`@CrossOrigin`允许客户端进行跨域请求，可以使用origins参数规定允许访问的源，methods参数可以规定允许请求的类型（get，post，put）

`@RequestParam`用于捕获get请求的查询字符串并解析其中的参数。如

```html
localhost:8080/minion?cost=9&rarity=RARE
```

required参数默认为true，即客户端必须提供对应的参数否则会导致请求失败（HTTP 400），同时Spring会抛出`MissingServletRequestParameterException`异常。设置为false之后对应参数可以为空。

**Tips**: 

`@RequestParam`注解可以配合required和defaultValue来使用，defaultValue可以提供一个默认值，当来自客户端的请求中未包含参数时，对应参数会被设置为默认值。

### Service

service层的主要业务逻辑是将controller解析的请求参数封装到对应的Pojo类中，并将Pojo类传到Mapper的查询方法中进行查询。

```java
package com.zzb.hearthstoneDB.service;

@Service
public class CardService {

    private final CardMapper cardMapper;

    @Autowired
    public CardService(CardMapper cardMapper) {
        this.cardMapper = cardMapper;
    }
```

`@Service`注解将类标记为SpringMVC的service组件。

service的方法大致相同，如下

```java
public List<Card> selectMinions(String name, Integer cost, Integer attack, 
                                Integer health, String rarity, String race, 
                                String cardClass,Integer cardSet, String rule) {

    Minion minion = new Minion(null, name, cost, 
                               cardClass == null ? null : CardClass.valueOf(cardClass), 																 cardSet, rule, attack, health, 
                               rarity == null ? null : Rarity.valueOf(rarity), 
                               race == null ? null : Race.valueOf(race), null);

    return cardMapper.selectMinion(minion);
}
```

### Mapper

在Model层采用的是ORM框架中的MyBatis。在MyBatis中的Mapper是映射SQL查询的接口。在查询接口中，可以选择直接用注解的形式（如`@Select("SELECT * FROM STUDENT")`等）来自定义查询的sql语句，但这一般仅适合简单的查询语句，当涉及多表查询，自定义结果集映射等时还是需要编辑XML文件的。

在项目中采用的是XML文件配置MyBatis，本文也重点讲述这种配置方法。

XML配置文件一般放置在resources文件夹中，注意，对于MyBatis的XML配置文件，**在resources中的结构必须与main文件夹中完全相同。**比如，mapper文件的结构是com.my.project.mapper.CardMapper，那么XML配置的结构也必须是com.my.project.mapper.CardMapper.xml，否则配置无法生效！

`@Param`注解给查询参数设置了别名，这个别名可以用与xml中的动态sql。

```java
package com.zzb.hearthstoneDB.mapper;

@Mapper
public interface CardMapper {

    // int addCard(@Param("card") Card card);

    List<Card> selectCards(@Param("card_query") Card card);

    String selectCardById(String id);

    List<Card> selectMinion(@Param("minion") Minion minion);

    List<Card> selectSpell(@Param("spell") Spell spell);

    List<Card> selectWeapon(@Param("weapon") Weapon weapon);

    List<Card> selectHero(@Param("hero") Hero hero);

    List<Card> selectHeroPower(@Param("heroPower") HeroPower heroPower);

    List<Card> selectLocation(@Param("location") Location location);
}
```

下面讲解一下XML配置文件的写法和结构

首先是xml的标准配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
```

可以算是xml文件的头文件，必须携带。

接下来是mapper的配置部分。

最大的标签是`<mapper>`,如

```xml
<mapper namespace="com.zzb.hearthstoneDB.mapper.CardMapper"></mapper>
```

有若干个二级标签, 如resultMap，select等。

resultMap用来定义数据库entity和Java中Pojo的映射关系。

id是每一个resultMap的唯一标识符，在后面的实际sql语句编写时如果需要用到resultMap来处理映射，需要用resultMap的id来指明使用的是哪一个resultMap。

type是映射到的Pojo类的全类名。

```xml
<resultMap id="cardResultMap" type="com.zzb.hearthstoneDB.pojo.Card">
    <result property="id" column="id"/>
    <result property="name" column="name"/>
    <result property="cost" column="cost"/>
    <result property="cardClass" column="card_class" 	typeHandler="com.zzb.hearthstoneDB.typeHandler.CardClassTypeHandler"/>
    <result property="cardSet" column="card_set"/>
    <result property="rule" column="rule"/>
</resultMap>
```

在resultMap中每一个result标签对应着一个属性的映射。property是pojo中的成员变量名，column是数据库表的列名。

如果需要使用typeHandler，需要使用typeHandler属性并指明使用的typeHandler的全类名。

`<select>`是xml中的动态sql查询标签，类似的还有`<update>`，`<delete>`等。

id对应Mapper接口的方法名。

resultMap表示使用的映射集。

在动态查询中，可以使用`<where>`和`if`标签，非常的智能，可以自动处理AND的连接。

为了防止SQL注入攻击，建议使用`#{}`而不是`${}`。前者会经过绑定和预处理，后者会直接拼接进入sql。

```xml
<select id="selectMinion" resultMap="minionResultMap">
    SELECT minion.id, name, cost, attack, health, rarity, race, card_class, card_set_id.id as card_set, rule, flavor
    from minion join card_set_id on minion.card_set = card_set_id.card_set
    <where>
            AND collectible = 1
        <if test="minion.name != null">
            AND name LIKE CONCAT('%', #{minion.name}, '%')
        </if>
        <if test="minion.cost != null and minion.cost &lt; 10">
            AND cost = #{minion.cost}
        </if>
        <if test="minion.cost != null and minion.cost == 10">
            AND cost &gt;= #{minion.cost}
        </if>
        <if test="minion.cardClass != null">
            AND card_class = #{minion.cardClass}
        </if>
        <if test="minion.cardSet != null">
            AND card_set_id.id = #{minion.cardSet}
        </if>
        <if test="minion.rule != null">
            AND rule LIKE CONCAT('%', #{minion.rule}, '%')  <!--模糊查询-->
        </if>
        <if test="minion.attack != null">
            AND attack = #{minion.attack}
        </if>
        <if test="minion.health != null">
            AND health = #{minion.health}
        </if>
        <if test="minion.rarity != null">
            AND rarity = #{minion.rarity}
        </if>
        <if test="minion.race != null">
            AND race = #{minion.race}
        </if>
    </where>
</select>
```

由于在xml中'<'和'>'有特殊意义，所以在sql中如果需要使用小于号和大于号等，需要使用转义字符，如`&lt;`（less than）。

### 数据库设计

<img src="https://pics.findfuns.org/hearthstone-database-diagram.png" style="zoom:33%;" />

上面是数据库的表设计。共有6张表，分别对应6种不同的卡牌类型。所有表的主键都是`id`。

