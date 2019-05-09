---
layout: post
title: "Java8流操作(Stream)"
subtitle: 'Some tips on leraning Swagger'
author: "FengJiaWen"
header-style: text
tags:
  - Java8新特性
---

Update: Java8新特性学习总结

---

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;最近开始学习java8的新特性(平时虽然一直用的jdk8,但是它的lumbda表达式,函数式接口,流操作等等一直没有学习和使用过),之前的lumbda表达式和函数式接口感觉似懂非懂的,故这两个特性的感悟后面再写,先写下感悟比较深的流操作(顺便把刚做的项目中间一些foreach改成流操作).</p>
<p>流的定义:从支持数据处理操作的源生成的元素序列</p>
<p>(1)元素序列:集合与流的区别
   <p>1.集合:主要目的是以特定的时间/空间复杂度存储和访问元素(如ArrayList/LinkedList).集合元素先全部算出来.
   <p>2.流:主要目的在于表达计算(如filter()/sorted()/map())等.流元素按需计算.流只能遍历一次.
<p>(2)源:流会使用一个提供数据的源:如集合、数组、或输入输出流等.
   <p>1.从有序集合生成的流会保留原有顺序.
   <p>2.由列表生成的流,其元素顺序与列表一致.</p>
<p>(3)数据处理操作:
   <p>1.流的数据处理功能类似数据库的操作以及函数式编程中的常用操作,如filter、map、reduce、find、match、sort等等.
   <p>2.流的操作可以顺序执行,也可以并行执行.</p>
<p>(4)流水线:
   <p>1.大部分流操作之后返回的还是一个流(中间操作),这样我们可以将多个操作串联起来来对集合进行处理(终端操作),形成一条巨大的流水线.
   <p>2.流的操作可以看作是对数据源进行数据库式查询(类似).</p>
<p>(5)内部迭代:
   <p>1.与使用显示迭代器的集合不同,流的迭代操作实在背后进行的.</p>
<p>流的扁平化:
    <p>1.flatmap方法让你把一个流中的每个值都换成另外一个流,然后把所有的流连接起来成为一个流.</p>

</p>一些项目中用到jdk8特性的地方</p>

        Map<Long, List<devAtmsDTO>> collect = devAtmDTO.stream().collect(Collectors.groupingBy(devAtmsDTO::getFloorNo, Collectors.toList()));
        return getSuccessResult(collect);

 
