---
title: 你真的理解了hashCode与equals了吗
date: 2020-06-08 14:54:59
tags:
- Java
categories: Java
keywords: hashCode,equals,hashcode
cover: https://i.loli.net/2020/06/08/XZN8VMlGI9KWYgh.png
---
## 前言
前天听隔壁的同事电话面试，问别人hashCode与equals相关的问题，我自己也想了一下，虽然能想出来，但是好像还是感觉缺点什么东西，于是乎，也查找了相关资料，觉得必要整理一下，做个记录。

在研究这个问题之前，先说明一下JDK对equesl和hashCode这两个方法的定义和规范：在Java中任何一个对象都具备这两个方法，因为他们都是在Object类中定义的。equals方法用来判断两个对象是否“相同”。hashCode方法返回一个int数值，在Object类中的默认实现是：将该对象的内部地址转换成一个整形的数值返回。

下面是官方文档给出的一些说明：
> hashCode的常规协定：
> 在Java程序执行期间，在同一个对象上多次调用hashCode方法时，必须一致地返回相同的整数，前提是对象上的equals比较汇中所用的信息没有被修改。从某一应用程序的一次执行到同一应用程序的另一次执行，该整数无需保持一致。
> 如果根据equals方法，两个对象是相等的，那么在两个对象中的任一对象上调用hashCode方法都必须生成相同的整数结果。
> 以下情况不是必须的：如果根据equals方法两个对象不相等，那么在两个对象中任一对象上调用hashCode方法必定会生成不同的整数结果。
> 但是程序猿应该知道，为不相等的对象生成不同整数结果可以提高哈希表的性能。实际上，由Object类定义的hashCode方法确实会针对不同的对象返回不同的整数。（这一般是通过将该对象的内部地址转换成一个整数来实现的，但是JavaTM编程语言不需要这种实现技巧。）

## 分析
以下是对上面的官方说明做的归纳总结：
* 如果重写了equals方法，则有必要重写hashCode方法。
* 如果两个对象equals返回true，则hashCode有必要返回相同的整数。
* 如果两个对象equals返回false，则hashCode不一定返回不同的int数值。
* 如果hashCode返回相同的int数值，则equals不一定返回true。
* 如果hashCode返回不同的int数值，则equals一定返回false。
* 同一个对象在执行期间如果已经存在集合中，则不能修改影响hashCode值相关的字段信息，否则会导致内存泄漏问题。

想要弄清楚以上六点，就先要知道什么时候需要重写equals和hashCode方法。

一般来说，涉及到对象之间的比较大小就需要重写equals方法，但是为什么第一点说到重写了equals方法就要重写hashCode呢？实际上这知识一条规范，如果不这样做，程序也可以运行，只不过会可能有隐藏彩蛋。一般一个类的对象如果会存储在HashTable、HashSet、HashMap等散列存储结构中，那么重写了equals后最好也重写hashCode，否则就会无法保证存储数据的唯一性（存储了两个equals相等的对象）。当然，如果确定不会存储在这些散列结构中，则不必重写hashCode。个人觉得还是重写比较好一点，谁能保证后期不会将数据存储到这些结构中呢，况且重写了hashCode也不会降低性能，像在线性结构（例如ArrayList）中是不会用到hashCode的。

下面来看一张对象放入散列集合的流程图：
![](https://i.loli.net/2020/06/08/uN9TGHs6agSKfpF.png)
从上面的图中可以清晰地看出，在存储一个对象时，先进行hashCode值的比较，然后比较equals。想要保证散列结构中元素的唯一性，必须同时覆盖hashCode与equals才行。
（注意HashSet中插入他认为的已存在的元素时，会被舍弃，而在HashMap的Key中会覆盖掉之前的元素的Value）

## 内存泄漏问题
你以为结束了？不，重点才刚刚开始。重写hashCode和equals之后，在散列集合中，如果使用不当会引发内存泄漏，我们先看一段示例。
```java
public class AAA {
    private int x ;
    private int y ;

    public int getX() {
        return x;
    }

    public AAA setX(int x) {
        this.x = x;
        return this;
    }

    public int getY() {
        return y;
    }

    public AAA setY(int y) {
        this.y = y;
        return this;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        AAA aaaa = (AAA) o;

        if (x != aaaa.x) return false;
        return y == aaaa.y;

    }

    @Override
    public int hashCode() {
        int result = x;
        result = 31 * result + y;
        return result;
    }

    @Override
    public String toString() {
        return "AAA{" +
                "x=" + x +
                ", y=" + y +
                '}';
    }
}
```
```java
public class Demo {
    public static void main(String[] args) {
        Set<AAA> set = new HashSet<>();
        AAA a = new AAA();
        a.setX(1);
        a.setY(1);
        AAA b = new AAA();
        b.setX(2);
        b.setY(2);
        set.add(a);
        set.add(b);
        System.out.println(set);
        a.setY(3);
        set.remove(a);//删除元素a
        System.out.println(set);
    }
}
```
这段代码中main方法运行之后得到以下结果：
```shell
[AAA{x=1, y=1}, AAA{x=2, y=2}]
[AAA{x=1, y=3}, AAA{x=2, y=2}]

Process finished with exit code 0
```

## 分析
假设a元素的hasCode是1，b元素的hashCode是2，在存储时他们分别存在HashSet中不同的位置。这时候修改了元素a中与计算hashCode有关的字段信息x，假设修改后hashCode时3，当调用remove时，首先会查找该hashCode值（此时为3）的元素是否在集合中，这是查找肯定没有，JDK则认为当前集合不存在这个对象，所以不会删除。然后用户却以为删除了这个元素，导致这个对象长期存在与集合中，造成了内存泄漏。

解决该问题的办法是不要在执行期间修改与hashCode值有关的对象信息，如果非要修改，则必须先从集合中删除，更新信息后再加入集合中。

## 总结
1. hashCode是为了提高在散列结构存储中查找的效率，在线性表中没有作用。
2. equals和hashCode需要同时覆盖。
3. 若两个对象equals返回true，则hashCode有必要也返回相同的int数。
4. 若两个对象equals返回false，则hashCode不一定返回不同的int数,但为不相等的对象生成不同hashCode值可以提高 哈希表的性能。
5. 若两个对象hashCode返回相同int数，则equals不一定返回true。
6. 若两个对象hashCode返回不同int数，则equals一定返回false。
7. 同一对象在执行期间若已经存储在集合中，则不能修改影响hashCode值的相关信息，否则会导致内存泄露问题。