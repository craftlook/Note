# JAVA 拆箱与装箱介绍

## 拆箱与装箱概念

​	Java中的基础数据类型都有对应的包装类型。

| 基础类型                                                     | 包装器类型 |
| ------------------------------------------------------------ | ---------- |
| byte(1字节)                                                  | Byte       |
| short(2字节)                                                 | Short      |
| int(4字节)                                                   | Integer    |
| long(8字节)                                                  | Long       |
| float(4字节)                                                 | Float      |
| double(8字节)                                                | Double     |
| char(2字节)                                                  | Charater   |
| boolean(JVM规范解释：4字节，boolean数组每个元素1字节。具体看jvm规范) | Boolean    |

​	什么是拆箱与装箱？**装箱**是自动将**基础类型转换为包装器类型**；**拆箱**是自动将**包装器类型转换为基础类型**；

```
Integer i = 10; //装箱
int j = i; //拆箱
```

​	**拆箱与装箱调用的方法**

* 拆箱过程通过调用包装器类的xxxValue（xxx根据包装器对应哪个基础类型）；

* 装箱过程通过调用包装器类的ValueOf；

其中装箱ValueOf方法：Integer、Short、Byte、Character、Long方法实现类型；Double、Float的方法的实现类似。<font color='red'>此处参考JDK版本1.8</font>

   **Integer** （范围内，整型数值的个数确定，所以可以使用cache）

```
public static Integer valueOf(int i) {
	if (i >= IntegerCache.low && i <= IntegerCache.high)
		return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

   **Double** （范围内，浮点数的个数不确定，所以是new）

```
public static Double valueOf(double d) {
    return new Double(d);
}
```

## 相关问题

1. 如下代码输出结果是什么

```
public static void main(String[] args) {
    // Integer
    Integer i1 = 100;
    Integer i2 = 100;
    Integer i3 = 200;
    Integer i4 = 200;
    System.out.println("------Integer比较结果-----");
    System.out.println(i1==i2);
    System.out.println(i3==i4);
}
```

实际输出结果：

```
------Integer比较结果-----
true
false
```

结果分析：

​	我们通过源码发现 Integer中的ValueOf方法用到了IntegerCache.cache。如果数值在[-128,127]之间，cache中已经存在的对象的引用，否则进行创建新的对象

* i1 == i2 ：true。数值小于127，cache存在对象引用，i1 与 i2 装箱后的对象引用相同；
* i3 == i4 ：false。数值大于127，cache不存在对象引用，i1 与 i2 装箱后的对象引用不相同；

2. 如下代码输出结果是什么

```
public static void main(String[] args) {
    // Double
    Double d1 = 100.0;
    Double d2 = 100.0;
    Double d3 = 200.0;
    Double d4 = 200.0;
    System.out.println("------Double比较结果-----");
    System.out.println(d1==d2);
    System.out.println(d3==d4);
}
```

实际输出结果：

```
------Double比较结果-----
false
false
```

结果分析：

​	我们通过Double中的ValueOf方法源码发现，每一次都进行new对象

* d1==d2：false
* d3==d4：false

3. 如下代码输出结果是什么

```
public static void main(String[] args) {
    // 运算的拆装箱
    Integer a = 100000000;
    Integer b = 200000000;
    Integer c = 300000000;
    Long g = 300000000L;
    Long h = 200000000L;
    System.out.println("------运算比较结果-----");
    System.out.println(c==(a+b));
    System.out.println(c.equals(a+b));
    System.out.println(g==(a+b));
    System.out.println(g.equals(a+b));
    System.out.println(g.equals(a+h));
}
```

实际输出结果：

```
------运算比较结果-----
true
true
true
false
true
```

结果分析：

​	下面是进行运算的比较。

* c==(a+b) ：true。其中 （a+b）会进行拆箱（intValue调用），拆箱后进行数值计算；
* c.equals(a+b)：true。先会（a+b）拆箱，因为用的equals比较，拆箱后再触发自动装箱。 先Integer.intValue,后Integer.valueOf，然后进行equals比较；
* g==(a+b)：true。拆箱后，进行数值比较；
* g.equals(a+b)：false。g 与（a+b）类型不同， a+b 数值int类型 ，装箱时Integer.valueOf；
* g.equals(a+h)：true。（a+h）数值是long类型(int类型和long类型相加，这个会触发类型晋升，结果是long类型的)，装箱时Long.valueOf;
