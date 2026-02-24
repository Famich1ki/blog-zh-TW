---
title: Spring Boot全局異常處理
date: 2024-09-16 18:01:40
tags: 
  - Spring Boot
  - 異常處理
categories:
  - [Spring Boot, 異常處理]
cover: https://pics.findfuns.org/exception-handling.png   
---
# 前言

當我們用Spring Boot構建工程的時候，在處理各種複雜的業務需求時往往會涉及到各種各樣的異常，比如在進行文件的IO操作時會遇到`IOException`，`FileNotFoundException`，在編冩SQL語句或使用JDBC時會遇到`SQLException`，在編冩涉及反射相關的代碼時會遇到`ClassCastException`。除此之外還有許多常見的異常如空指針異常`NullPointerException`，數組下標越界異常`ArrayIndexOutOfBoundsException`，在使用迭代器遍曆集合時修改元素産生的異常`ConcurrentModificationException`，算數異常（如除0）`ArithmeticException`等等。

在處理這些異常時無非就是兩種選擇，最直接最省事的選擇是直接使用`throws`關鍵字拋出異常，讓上層的方法處理異常。另外一種方法就是用try-catch代碼塊捕捉異常。這兩種方法的缺點都很明顯，當工程量變大需要處理異常的地方逐漸變多時，如果一個一個的處理異常會顯得非常低效，而且也不方便進行統一管理。

那麼是否存在一種可以全局管理異常的方法呢。

### `RestControllerAdvice`和`ExceptionHandler`

被`RestControllerAdvice`標記的類可以用於處理全局異常。同時將`ExceptionHandler`標記在方法上可以處理對應的異常。

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public Response exceptionHandler(RuntimeException e) {
        log.error("Internal Server Error " + e);
        return Response.error(e.getMessage(), ExceptionEnum.INTERNAL_SERVER_ERROR.getCode());
    }
}
```

ExceptionHandler接受Class[]類型的參數，代表能處理的異常的類型。

定義基本的異常接口和枚舉類

```java
public interface BaseException {

    String getCode();
    String getMsg();
}
```

```java
public enum ExceptionEnum implements BaseException{
    SUCCESS("200", "Success"),
    BAD_REQUEST("400", "Bad Request"),
    NOT_FOUND("404", "Not Found"),
    METHOD_NOT_ALLOWED("405", "Method Not Allowed"),
    INTERNAL_SERVER_ERROR("500", "Internal Server Error");

    private final String code;
		private final String msg;

    ExceptionEnum(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }
    @Override
    public String getCode() {
        return this.code;
    }

    @Override
    public String getMsg() {
        return this.msg;
    }
}
```

定義一個Response類用於統一返回數據的格式

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Response {

    private String msg;

    private String code;

    public static Response success(ExceptionEnum exceptionEnum) {
        Response response = new Response(exceptionEnum.getMsg(), exceptionEnum.getCode());
        return response;
    }

    public static Response error(ExceptionEnum exceptionEnum) {
        Response response = new Response(exceptionEnum.getMsg(), exceptionEnum.getCode());
        return response;
    }

    public static Response error(String msg, String code) {
        Response response = new Response(msg, code);
        return response;
    }

    public static Response error(String msg) {
        Response response = new Response(msg, "-1");
        return response;
    }
}
```

測試一下，在controller中故意製造一個算數異常

```java
@PutMapping("/add")
public String addPerson(@RequestBody Person person) {
  int i = 1 / 0;
  return myService.addPerson(person);
}
```

在postman中髮送請求

<img src="https://pics.findfuns.org/globalExceptionHandler.png" style="zoom:33%;" />

可以看到返回的msg是"/ by zero"正對應着算數異常，同時code是先前設置好的500。

這樣一來我們就能通過設置全局異常處理的方式來統籌管理整個項目中的所有異常，無需再一個一個關注具體的異常處理，可以將注意力全身心放在業務邏輯上，非常方便。