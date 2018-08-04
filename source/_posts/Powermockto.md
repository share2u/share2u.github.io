---
title: Powermockto
date: 2018-08-04 23:30:36
categories:
 - 工具
tags:
 - test
---

mock是模拟对象，用于模拟真实对象的行为  
powermock拓展了esaymock和mockito框架，增加了对static 和final方法mock支持等功能，

本文阐述powermockto简单用法，未包含 answer,spy,captor等

<!-- more-->

## jar包
```
<dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-module-junit4</artifactId>
        <version>1.6.3</version>
        <scope>test</scope>
    </dependency>    
    <dependency>    
        <groupId>org.powermock</groupId>    
        <artifactId>powermock-api-mockito</artifactId>    
        <version>1.6.3</version>    
        <scope>test</scope>    
    </dependency>   
    
```
## mock流程
1. 创建一个mock对象
2. 期望：记录希望mock对象有的行为，通常是伪造的部分
3. 回放（replay）：调用我们的测试的代码
4. 验证（Verfications）:验证结果
## 注解概述
### @RunWith(PowerMockRunner.class)
相当于
```java
@Before  
public void initMocks() {
    //注解初始化
    MockitoAnnotations.initMocks(this);  
}
```
### @PrepareForTest({Student.class})
如果mock的对象的方法是静态、final、私有方法，将类加到注解数组中
### @Mock
mock的对象
### @InjectMocks
mock对象自动注入到该测试类中,尝试类型注入，如果有多个类型的mock对象，那么会根据名称进行注入，当注入失败的时候，不会抛出异常

## mock
### 普通方法

```java
public class T1 {
    public boolean isExist(File file){
        return file.exists();
    }
}

@Test
    public void test1(){
        T1 t1 = PowerMockito.mock(T1.class);
        when(t1.isExist(any())).thenReturn(true);
        assertTrue(t1.isExist(new File("xx.xml")));

    }
//不需要写注解
```

### 模拟静态方法

```java
public class T1 {
    public static boolean isExist(String path){
        File file = new File(path);
        return file.exists();
    }
}

@RunWith(PowerMockRunner.class)
@PrepareForTest(T1.class)
public class testMock {
    @Test
    public void test1() throws Exception {
        PowerMockito.mockStatic(T1.class);
        when(T1.isExist(any())).thenReturn(true);
        assertTrue(T1.isExist("xx"));

    }
}
```



### 模拟final类或方法

```java
public class T1 {
    public final boolean isExist(String path){
        File file = new File(path);
        return file.exists();
    }
}

@RunWith(PowerMockRunner.class)
@PrepareForTest(T1.class)
//加注解表明final的类
public class testMock {
    @Test
    public void test1() throws Exception {
        T1 t1 = PowerMockito.mock(T1.class);
        when(t1.isExist(any())).thenReturn(true);
        assertTrue(t1.isExist("xx"));

    }
}
```



### 构造函数

```java
public class T1 {
    public boolean isExist(String path){
        File file = new File(path);
        return file.exists();
    }
}


@RunWith(PowerMockRunner.class)
@PrepareForTest(T1.class)
public class testMock {

    @Test
    public void test1() throws Exception {
        T1 t1 = new T1();
        File file = PowerMockito.mock(File.class);
        //使用whenNew 构造file返回值，需要添加两个注解
        whenNew(File.class).withAnyArguments().thenReturn(file);
        when(file.exists()).thenReturn(true);
        assertTrue(t1.isExist("xx.xml"));

    }
}
```



### 模拟私有方法

```java
public class T1 {
    private boolean isExist(String path){
        File file = new File(path);
        return file.exists();
    }
    public  boolean Exist(String path){
        return isExist(path);
    }
}

@RunWith(PowerMockRunner.class)
@PrepareForTest(T1.class)
public class testMock {
    @Test
    public void test1() throws Exception {
        T1 t1 = PowerMockito.mock(T1.class);
        when(t1.Exist("xx")).thenCallRealMethod();
        when(t1,"isExist","xx").thenReturn(true);
        assertTrue(t1.Exist("xx"));
    }
}
```



## verfications
### 验证方法调用

Mockito.verify(mock).create()验证调用了create方法。 

Mockito.verify(mock, Mockito.never()).update();验证没有调用update方法。 

### 验证调用次数

`Mockito.times(int n)` : 准确的验证方法调用的次数:n

 `Mockito.atLeastOnce()` : 验证方法至少调用1次 

 `Mockito.atLeast(int n)` : 验证方法最少调用n次 

 `Mockito.atMost(int n)`: 验证方法最多调用n次  

`Mockito.inOrder`:验证方法调用的顺序 

## 原理简述

- 当某个方法被注解@PrepareForTest注解后，启动该测试用例，会创建一个新的MockclassLoader实例，然后加载测试用例用到的类（系统类除外）
- 加载过程中，会根据mock请求修改mock的class文件，eg,去掉final标识，修改方法体
- 对于系统类，会修改调用系统类的class文件，不会直接修改系统类class

## 参考文献
1. https://www.cnblogs.com/IamThat/p/5072499.html
2. https://blog.csdn.net/qisibajie/article/details/79068086#mockito%E5%92%8Cpowermock%E7%9A%84%E7%94%A8%E6%B3%95
3. http://hotdog.iteye.com/blog/937862