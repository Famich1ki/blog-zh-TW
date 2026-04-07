---
title: Java Enum Design & Generic Utility
date: 2026-04-07 23:41:50
tags:
  - Java
  - Enumeration
categories:
  - [Java, Enumeration]
cover: https://pics.findfuns.org/java-enumeration.png
---

# Java 枚舉設計與泛型工具類

## 1. 問題背景

在很多業務場景中，枚舉需要從 `String` 值進行轉換：

```
​```java
StudentStatus.from("enrolled");
```

如果沒有抽象，每個枚舉都會重複相同的邏輯：

```
for (StudentStatus s : StudentStatus.values()) {
    if (s.getValue().equals(value)) {
        return s;
    }
}
```

這會導致**代碼重複**以及較差的可維護性。我們需要讓代碼保持 **DRY（Don't Repeat Yourself，不重複自己）**。

## 2. 目標

- 消除重複的 `from()` 邏輯
- 保持類型安全
- 保持 API 簡潔清晰
- 讓方案可以在所有枚舉中複用

## 3. 最終設計（最佳實踐）

### 3.1 基礎接口

```
public interface BaseEnum {
    String getValue();
}
```

作用：

- 定義一個**規範**
- 確保所有枚舉都提供 `getValue()` 方法

## 3.2 泛型工具類

```
public class EnumUtil {

    public static <T extends Enum<T> & BaseEnum> T fromValue(Class<T> enumClass, String value) {
        for (T constant : enumClass.getEnumConstants()) {
            if (constant.getValue().equals(value)) {
                return constant;
            }
        }
        throw new RuntimeException("Invalid value: " + value);
    }
}
```

## 3.3 枚舉實現

```
@Getter
public enum StudentStatus implements BaseEnum {

    ENROLLED("enrolled"),
    ABSENT("absent"),
    COMPLETED("completed");

    private final String value;

    StudentStatus(String value) {
        this.value = value;
    }

    public static StudentStatus from(String value) {
        return EnumUtil.fromValue(StudentStatus.class, value);
    }
}
```

# 4. 關鍵概念解析

### 4.1 爲什麼要使用接口？

如果沒有 `BaseEnum`：

```
<T extends Enum<T>>
```

編譯器只知道它是一個枚舉，
但並不知道 `getValue()` 方法存在。

使用 `BaseEnum` 後：

```
<T extends Enum<T> & BaseEnum>
```

編譯器現在可以保證：

- 一定存在 `getValue()` 方法

### 4.2 Lombok vs 接口

| 功能                 | Lombok | 接口 |
| -------------------- | ------ | ---- |
| 自動生成 getter      | ✅      | ❌    |
| 強制方法存在（約束） | ❌      | ✅    |

Lombok 用於減少代碼量，
接口用於保證**類型安全**。

### 4.3 理解泛型

**<T>**

- 一個**具名類型參數**
- 可以在整個方法中使用

```
?
```

- 一個**匿名通配符**
- 不能用於具體操作

### 4.4 區別：`T` vs `?`

| 特性           | `T`  | `?`  |
| -------------- | ---- | ---- |
| 有名字         | ✅    | ❌    |
| 可作爲返回類型 | ✅    | ❌    |
| 可以修改       | ✅    | ❌    |
| 只讀使用       | ✅    | ✅    |

### 4.5 `Class<T>` vs `Class<?>`

| 類型       | 含義         |
| ---------- | ------------ |
| `Class<T>` | 已知具體類型 |
| `Class<?>` | 未知類型     |

示例：

```
Class<StudentStatus> clazz = StudentStatus.class; // 具體類型
Class<?> clazz = StudentStatus.class;             // 泛型（未知類型）
```

# 5. 爲什麼不省略接口？

## 方案 1：只使用 Lombok

問題：

- 編譯器無法確認 `getValue()` 是否存在
- 泛型方法無法正常工作

## 方案 2：使用 `name()`

```
constant.name().equals(value)
```

問題：

- 不夠靈活
- 與枚舉名稱強耦合
- 不適用於真實業務場景

## 方案 3：使用反射

```
constant.getClass().getMethod("getValue")
```

問題：

- 性能較差
- 不安全
- 難以維護

# 6. 爲什麼把 `from()` 放在枚舉內部？

### 可讀性更好：

```
StudentStatus.from("enrolled");
```

對比：

```
EnumUtil.fromValue(StudentStatus.class, "enrolled");
```

枚舉中的寫法更具表達力，更符合領域語義（domain-oriented）。

# 7. 設計思想

### 核心理念：

> 接口不是用來寫代碼的，而是用來約束規範的。

### 好的設計 =

- 使用簡單
- 內部約束嚴格
- 組件可複用

# 8. 最終總結

- 使用 **接口 + 泛型工具類 + 枚舉封裝**
- Lombok 是爲了方便，不是爲了類型安全
- 泛型（`T`）讓設計既可複用又安全
- 避免爲了省事而犧牲可維護性的寫法
