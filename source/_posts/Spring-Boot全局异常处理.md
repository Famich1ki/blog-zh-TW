---
title: Spring Boot全局异常处理
date: 2024-09-16 18:01:40
tags: 
  - Spring Boot
  - 异常处理
categories:
  - [Spring Boot, 异常处理]
cover: https://pics.findfuns.org/exception-handling.png   
---
# 前言

当我们用Spring Boot构建工程的时候，在处理各种复杂的业务需求时往往会涉及到各种各样的异常，比如在进行文件的IO操作时会遇到`IOException`，`FileNotFoundException`，在编写SQL语句或使用JDBC时会遇到`SQLException`，在编写涉及反射相关的代码时会遇到`ClassCastException`。除此之外还有许多常见的异常如空指针异常`NullPointerException`，数组下标越界异常`ArrayIndexOutOfBoundsException`，在使用迭代器遍历集合时修改元素产生的异常`ConcurrentModificationException`，算数异常（如除0）`ArithmeticException`等等。

在处理这些异常时无非就是两种选择，最直接最省事的选择是直接使用`throws`关键字抛出异常，让上层的方法处理异常。另外一种方法就是用try-catch代码块捕捉异常。这两种方法的缺点都很明显，当工程量变大需要处理异常的地方逐渐变多时，如果一个一个的处理异常会显得非常低效，而且也不方便进行统一管理。

那么是否存在一种可以全局管理异常的方法呢。

### `RestControllerAdvice`和`ExceptionHandler`

被`RestControllerAdvice`标记的类可以用于处理全局异常。同时将`ExceptionHandler`标记在方法上可以处理对应的异常。

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

ExceptionHandler接受Class[]类型的参数，代表能处理的异常的类型。

定义基本的异常接口和枚举类

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

定义一个Response类用于统一返回数据的格式

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

测试一下，在controller中故意制造一个算数异常

```java
@PutMapping("/add")
public String addPerson(@RequestBody Person person) {
  int i = 1 / 0;
  return myService.addPerson(person);
}
```

在postman中发送请求

<img src="https://pics.findfuns.org/globalExceptionHandler.png" style="zoom:33%;" />

可以看到返回的msg是"/ by zero"正对应着算数异常，同时code是先前设置好的500。

这样一来我们就能通过设置全局异常处理的方式来统筹管理整个项目中的所有异常，无需再一个一个关注具体的异常处理，可以将注意力全身心放在业务逻辑上，非常方便。