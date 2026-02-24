---
title: hearthstone database項目總結
date: 2024-08-27 14:19:50
tags:
  - Spring Boot
  - Java
  - MyBatis
  - MySQL
categories:
  - 實戰項目
cover: https://pics.findfuns.org/hearthstone.jpg
---



## 冩在前麵

這是一個前後端分離的全棧開髮項目。後端採用的技術棧是SpringBoot + MyBatis + MySQL。前端採用的是vue3框架。在項目完成後部署在了AWS雲服務上，OS採用的是ubuntu並使用nginx進行了反向代理。

整個項目從收集數據集，清洗數據，整合數據庫開始，一直到最終部署在服務器上曆時大約10天。其實前後端開髮佔用時間並不是很長，也沒有用到很複雜的技術，主要的時間消耗在於數據庫的清洗和整合工作。元數據來自一個第三方的hearthstone網站提供的API(在此非常感謝)，但在獲取元數據之後髮現並不能立刻投入使用，因爲存在很多髒數據和一些冗餘數據，並且一些數據還存在格式上的不匹配。把這些問題處理完畢耗費了大約3-4天的時間。

本文章用於總結和回顧整個項目開髮過程中遇到的問題和主要的工作量。

先貼一張IDEA的項目結構圖

<img src="https://pics.findfuns.org/backend-overview.png" alt="back" style="zoom:33%;" />

整個後端項目採用的是經典的SpringMVC框架,即Controller-Service-Mapper三層。同時也設計了Pojo類以及自定義`Typehandler`，自定義`TypeHandler`用於處理數據庫查詢結果和Java實例之間的映射（當映射關繫沒有默認處理時需要設計自定義`TypeHandler`）。

### POJO

先來講一講pojo類。爲了契合卡牌的不同類型和各種屬性，pojo類進行了詳細的設計，包括繼承關繫，枚舉類，數據類型等。

<img src="https://pics.findfuns.org/pojo-hierarchical-diagram.png" alt="z" style="zoom:33%;" />

`Card`類是所有卡牌的基類，所有類型的卡牌都是`Card`類的子類。`Card`類也包含了所有卡牌都有的、最基本的幾個屬性。

```java
package com.zzb.hearthstoneDB.pojo;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Card {

    private String id; // 每張卡牌的唯一id

    private String name; // 卡牌名稱

    private Integer cost; // 卡牌費用

    private CardClass cardClass; // MAGE, DRUID, PRIEST ...

    private Integer cardSet; // 一個整數，對應着一個版本

    private String rule; // 卡牌描述
}
```

Card共有6個子類： `Spell`， `Hero`， `HeroPower`， `Location`， `Weapon`， `Minion`。分別對應6種不同的卡牌類型： 法術， 英雄牌， 英雄技能， 地標， 武器， 隨從。

例如`Minion`類的設計

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

    private Integer attack; // 攻擊力

    private Integer health; // 生命值

    private Rarity rarity; // 稀有度

    private Race race; // 種族

    private String flavor; // 簡介
}
```

對於幾個屬性值，`Rarity`， `Race`，`CardClass`，`SpellSchool`，他們都有一些固定的值的集合，非常適合採用枚舉類來表示。

例如`SpellSchool`類的設計

```java
package com.zzb.hearthstoneDB.pojo;

public enum SpellSchool {

    ARCANE, // 奧術
    FIRE, // 火焰
    FROST, // 冰霜
    NATURE, // 自然
    SHADOW, // 暗影
    HOLY, // 神聖
    FEL, // 邪能
}
```

### Controller

Controller層主要的邏輯是路由規劃並從URL中獲取查詢參數並將參數傳遞給Service層。

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

`@RestController`是一個複合注解，包括了`@Controller`和`@ResponseBody`這兩個注解。`@Controller`注解將類標記爲SpringMVC的Controller，`@ResponseBody`注解指示方法的返回值直接冩入響應體而不是視圖，並以JSON的格式返回。

同時使用`@Autowired`注解自動注入Service組件。

使用`@RequestMapping`注解規定根路由。注意，**根路由的開頭是不帶’/‘的**，這與每個方法上的子路由是不同的。

Controller中的方法的設計大緻相同，如下

```java
@GetMapping("/minion") // 字路由的開頭帶‘/’
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

`@GetMapping`規定這個方法用於處理一個get請求並説明了對應的路由。

`@CrossOrigin`允許客戶端進行跨域請求，可以使用origins參數規定允許訪問的源，methods參數可以規定允許請求的類型（get，post，put）

`@RequestParam`用於捕獲get請求的查詢字符串並解析其中的參數。如

```html
localhost:8080/minion?cost=9&rarity=RARE
```

required參數默認爲true，即客戶端必須提供對應的參數否則會導緻請求失敗（HTTP 400），同時Spring會拋出`MissingServletRequestParameterException`異常。設置爲false之後對應參數可以爲空。

**Tips**:

`@RequestParam`注解可以配合required和defaultValue來使用，defaultValue可以提供一個默認值，當來自客戶端的請求中未包含參數時，對應參數會被設置爲默認值。

### Service

service層的主要業務邏輯是將controller解析的請求參數封裝到對應的Pojo類中，並將Pojo類傳到Mapper的查詢方法中進行查詢。

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

`@Service`注解將類標記爲SpringMVC的service組件。

service的方法大緻相同，如下

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

在Model層採用的是ORM框架中的MyBatis。在MyBatis中的Mapper是映射SQL查詢的接口。在查詢接口中，可以選擇直接用注解的形式（如`@Select("SELECT * FROM STUDENT")`等）來自定義查詢的sql語句，但這一般僅適合簡單的查詢語句，當涉及多表查詢，自定義結果集映射等時還是需要編輯XML文件的。

在項目中採用的是XML文件配置MyBatis，本文也重點講述這種配置方法。

XML配置文件一般放置在resources文件夾中，注意，對於MyBatis的XML配置文件，**在resources中的結構必須與main文件夾中完全相同。**比如，mapper文件的結構是com.my.project.mapper.CardMapper，那麼XML配置的結構也必須是com.my.project.mapper.CardMapper.xml，否則配置無法生效！

`@Param`注解給查詢參數設置了別名，這個別名可以用與xml中的動態sql。

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

下麵講解一下XML配置文件的冩法和結構

首先是xml的標準配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
```

可以算是xml文件的頭文件，必須攜帶。

接下來是mapper的配置部分。

最大的標籤是`<mapper>`,如

```xml
<mapper namespace="com.zzb.hearthstoneDB.mapper.CardMapper"></mapper>
```

有若幹個二級標籤, 如resultMap，select等。

resultMap用來定義數據庫entity和Java中Pojo的映射關繫。

id是每一個resultMap的唯一標識符，在後麵的實際sql語句編冩時如果需要用到resultMap來處理映射，需要用resultMap的id來指明使用的是哪一個resultMap。

type是映射到的Pojo類的全類名。

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

在resultMap中每一個result標籤對應着一個屬性的映射。property是pojo中的成員變量名，column是數據庫表的列名。

如果需要使用typeHandler，需要使用typeHandler屬性並指明使用的typeHandler的全類名。

`<select>`是xml中的動態sql查詢標籤，類似的還有`<update>`，`<delete>`等。

id對應Mapper接口的方法名。

resultMap表示使用的映射集。

在動態查詢中，可以使用`<where>`和`if`標籤，非常的智能，可以自動處理AND的連接。

爲了防止SQL注入攻擊，建議使用`#{}`而不是`${}`。前者會經過綁定和預處理，後者會直接拼接進入sql。

```xml
<select id="selectMinion" resultMap="minionResultMap">
    SELECT minion.id, name, cost, attack, health, rarity, race, card_class, card_set_id.id as card_set, rule, flavor
    from minion join card_set_id on minion.card_set = card_set_id.card_set
    <where>
            AND collectible = 1
        <if test="minion.name != null">
            AND name LIKE CONCAT(''%'', #{minion.name}, ''%'')
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
            AND rule LIKE CONCAT(''%'', #{minion.rule}, ''%'')  <!--模糊查詢-->
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

由於在xml中''<''和''>''有特殊意義，所以在sql中如果需要使用小於號和大於號等，需要使用轉義字符，如`&lt;`（less than）。

### 數據庫設計

<img src="https://pics.findfuns.org/hearthstone-database-diagram.png" style="zoom:33%;" />

上麵是數據庫的表設計。共有6張表，分別對應6種不同的卡牌類型。所有表的主鍵都是`id`。