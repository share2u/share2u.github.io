---
title: java8学习笔记
date: 2018-08-01 21:47:39
categories:
 - java
tags:
 - java8
---

- **Lambda表达式**
- 函数式接口
- 方法引用和构造器引用
- **Stream APi**
- 接口中的默认方法与静态方法
- 新时间日期API
- 其他新特性

<!--more-->

## 新特性简介

- 速度更快
  - hashmap
    - 1.7数组链表
    - 1.8数组链表-红黑树
  - ConcurrentHashMap
    - 1.7 分段锁
    - 1.8cas无锁算法
  - 内存结构变化
    - 1.7方法区是永久区的一部分，存放加载的类信息等
    - 1.8把永久区去掉了，并将方法区叫做元空间MetaSpace,并使用物理内存
- 代码更少（Lambda表达式）
- 强大的Stream API（像sql一样简单）
- 便于并行
- 最大化减少空指针异常Optional

## Lambda

### 为什么使用Lambda

是一个匿名函数=====一段可以传递的代码

example1:

```java
//匿名内部类
Comparator<Integer> com = new Comparator<Integer>(){
    public int compare(Integer o1,Integer o2){
        return Integer.compare(o1,o2);
    }
};
//lambda
Comparator<Integer> com = (x,y) -> Integer.compare(x,y);
TreeSet<Integer> ts = new TreeSet<>(com);
```

example 2:

```java
public class TestLambda {

    /**
     * 一堆员工中工资大于900的
     * 一堆员工中工资大于900的，但是年龄小于18岁的
     */

    /**
     * 方案1：
     *  遍历，各种遍历
     * 方案2：
     *  策略模式以及优化的匿名内部类
     * 方案3：
     *  java8函数式编程
     */
    List<Persion> persions ;
    @Before
    public void testBefore(){
        persions = Arrays.asList(
                new Persion("zs",16,1299),
                new Persion("ls",19,799),
                new Persion("ww",17,1999),
                new Persion("zl",20,1999),
                new Persion("tq",18,699)
        );
    }
    //方案1
    @Test
    public void test1(){
        ArrayList<Persion> result = new ArrayList<Persion>();
        for (Persion p : this.persions) {
            if(p.getSalary()>800){
                result.add(p);
            }
        }
        System.out.println(JSONObject.toJSONString(result));

        //如果再加条件，那么就再一个方法遍历多个条件，缺点：代码重复，核心代码没几行
    }
    //方案2
    @Test
    public void test2(){

        MyPredicate<Persion> myPredicate = new MyPredicate<Persion>() {
            public boolean test(Persion persion) {
                return persion.getSalary()>900;
            }
        };
        ArrayList<Persion> result = new ArrayList<>();
        for (Persion p : this.persions) {
            if(myPredicate.test(p)){
                result.add(p);
            }
        }
        System.out.println(JSONObject.toJSONString(result));
        //主逻辑不需要变，但是还需要写接口之类的
    }
    @Test
    public void test3(){

        MyPredicate<Persion> myPredicate = persion -> persion.getSalary()>900;
        ArrayList<Persion> result = new ArrayList<>();
        for (Persion p : this.persions) {
            if(myPredicate.test(p)){
                result.add(p);
            }
        }
        result.forEach( System.out::println);
    }
}
```

### Lambda 基本语法

####  语法格式：

- 无参数，无返回值

  runable example

  ```java
  //匿名内部类的final问题
  
  ```

  

- 有一个参数，无返回值

  consume example

- 有多个参数，有返回值，多条语句

  Comparator example

#### 函数式接口

接口中只有一个抽象方法的接口，称为函数式接口

@FuncationInterface

对一个数进行运算 example

```java
@FuncationInterface
interface MyFun{
    Integer getValue()
}

//接口 lambda 实现
```



## 四大内置核心函数式接口

| 函数式接口               | 参数类型 | 返回类型 | 用途                                                         |
| ------------------------ | -------- | -------- | ------------------------------------------------------------ |
| Consumer<T> 消费型接口   | T        | void     | 对类型为T的对象应用操作,包含方法 void accept(T t)            |
| Supplier<T>供给型接口    | 无       | T        | 返回类型为T的对象，包含方法：T get()                         |
| Function<T,R> 函数型接口 | T        | R        | 对类型为T的对象应用操作，并返回结果为类型R的对象，包含方法：R apply(T t) |
| Predicate<T> 断言型接口  | T        | boolean  | 确定类型为T的对象是否满足某约束，并返回boolean值，包含方法 boolean test(T t) |



## 方法引用，构造器引用，数组引用

### 方法引用

​	若Lambda体中的内容有方法已经实现了，我们可以使用“方法引用”（可以理解为方法引用是Lambda表达式的另外一种表现形式）

#### 语法格式

可以使用方法引用的前提是 实现的接口的参数和返回值 与 调用方法的参数与返回值一样

- 对象：：实例方法名
- 类：：静态方法名
- 类：：实例方法名
  - 第一个参数是方法的调用者，第二个是方法的参数值

example

```java
 	@Test
    public void test5(){
        Consumer<String> con = (s) -> System.out.println(s);

        PrintStream out = System.out;
        Consumer<String> con1 = out :: println;
    }

    @Test
    public void test6(){
        Comparator<Integer> com = (x, y) -> Integer.compare(x,y);
        Comparator<Integer> com1 = Integer::compare;
    }
    @Test
    public void test7(){
        BiPredicate<String,String> bp = (x,y) -> x.equals(y);
        BiPredicate<String,String> bp1 = String::equals;
    }
```



### 构造器引用

#### 语法格式

CLassName:: new

需要调用的构造器的参数列表要与函数式接口中方法的参数列表保持一致

```java
@Test
    public void test8(){
        Supplier<Persion> sup = ()->new Persion();
        Supplier<Persion> sup1 = Persion::new;
    }
```



### 数组引用

#### 语法格式

Type:new

```java
@Test
    public void test10(){
        Function<Integer,String[]> fun = (x) -> {new String[x]};
        Function<Integer,String[]> fun1 = String[]::new;
    }
```

## Stream

是java8中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤、和映射数据等操作。

1. 数据源：集合、数组等
2. 管道：流水线式的中间操作，筛选、切片等
3. 产生一个新流（终端操作）：跟原数据源没有关系

注意：

- 不会存储元素
- 不会改变原对象
- 延迟执行，需要执行结果才执行

### 创建Stream

#### 方法

1. 通过Collection系列集合提供的stream（）、parallelStream()(串行流和并行流)
2. 通过Arrays中的静态方法stream()获取数组流
3. 通过Stream中的静态方法of(T t)
4. 创建无限流
   1. 迭代，Stream.iterator(seed,unaryOPerator) 
   2. 生成，Stream.generator...

#### 并行流与串行流

并行流就是把一个内容分成一个多个数据库，并用不同的线程分别处理每个数据块的流，可以声明式的通过parallel()与sequential()在并行流与顺序流之间进行切换。联系Fork/Join框架。

### 中间操作

#### 筛选与切片

| 方法                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| filter(Predicate p) | 接收Lambda,从流中排除某些元素                                |
| distinct()          | 筛选，通过流所生成元素的hashcode()和equals去除重复元素       |
| limit(long maxSize) | 截断，使元素不超过给定数量                                   |
| skip(long n)        | 跳过元素，返回一个扔掉了前n个元素的流，若流中元素不足n个，则返回一个空流，与limit(n)互补 |

#### 映射

将流中的元素通过函数映射为其他流

| 方法                            | 描述                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| map(Function f)                 | 接收一个函数作为参数，该函数会被应用到每一个元素上，并将其映射一个新的元素 |
| mapToDouble(ToDoubleFunction f) | 接收一个函数作为参数，该函数会被应用到每个元素上，产生一个新的DoubleStream |
| mapToint(ToIntFunction f)       | 接收一个函数作为参数，该函数会被应用到每个元素上，产生一个新的IntStremam |
| mapToLong(ToLongFunction f)     | 接收一个函数作为参数，该函数会被应用到每个元素上，产生一个新的LongStream |
| flatMap(Function f)             | 接收一个函数作为参数，将流中的每个值换成另外一个流，然后把所有的流连接成一个流 |

#### 排序

| 方法                    | 描述                               |
| ----------------------- | ---------------------------------- |
| sorted()                | 产生一个新流，其中按自然顺序排序   |
| sorted(Comparator comp) | 产生一个新流，其中按比较器顺序排序 |

### 终端操作

终端操作会从流的流水线生成结果，其结果可以是任何不是流的值，例如，list,Integer,void等

#### 查找与匹配

| 方法                   | 描述                     |
| ---------------------- | ------------------------ |
| addMatch(Predicate p)  | 检查是否匹配所有元素     |
| anyMatch(Preducate p)  | 检查是否至少匹配一个元素 |
| noneMatch(Predicate p) | 检查是否没有匹配所有元素 |
| findFirst()            | 返回第一个元素           |
| findany()              | 返回当前流中的任意元素   |

#### 归约与收集

| 方法                                                       | 描述                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| reduce(T identity,BinaryOperatore)、reduce(BinaryOperator) | 将流中元素反复结合起来，得到一个值                           |
| collect(Collector c)                                       | 将流转换为其他形式，接收一个Collector接口是实现，用于给Stream中元素做汇总 |

## optional容器类

代表一个值存在或不存在，原来用null表示一个值不存在，现在Optional 可以更好的表达这个概念，并且可以避免空指针异常。

### 常用方法

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Optional.of(T)                                               | 创建指定引用的Optional实例，若引用为null则快速失败           |
| Optional.absent()                                            | 创建一个空的Optional实例                                     |
| Optional.ofNullable()                                        | 若t不为null,创建Optinal实例，否则创建空实例                  |
| boolean isPresent()                                          | 判断是否包含值                                               |
| T orElse(T other)                                            | 如果调用对象包含值，返回值，否则返回other                    |
| T orElseGet(Supplier<? extends T> other)                     | 如果调用对象包含值，返回该值，否则返回other获取到的值        |
| Optional<U> map(Function<? super T, ? extends U> mapper)     | 如果有值对其处理，并返回处理后的optinal,否则返回Optional.empty() |
| Optional<U> flatMap(Function<? super T, Optional<U>> mapper) | 与map类似                                                    |

有点鸡肋，麻烦的狠

## 接口优化

### 默认方法

- 实现多接口，如果都有默认方法实现，必须在当前的类中对方法进行覆写

###  静态方法

接口中可以有静态方法

## 日期时间API

代办

## 重复注解与类型注解

重复注解：在一个类上重复定义一个注解，该注解的注解需要标注为@Repeatables