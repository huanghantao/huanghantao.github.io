---
title: PHP内核剖析(二)-数据类型
date: 2017-12-29 09:53:48
tags:
- PHP
---

额，这篇博客与大家分享的是关于PHP内核的数据类型部分。

何谓数据类型，我们在自己写的PHP代码里面貌似很少去关注一个变量的类型。但是，如果小伙伴们使用过C/C++、Java等静态语言，那么肯定会对变量的数据类型很敏感。例如，我在C语言里面这样写：

```c
int main(int argc, char const *argv[]) {
	int a = 2;

	return 0;
}
```

其中的`int a = 2;`

这句话指的是，程序在运行的时候，在栈上会为变量`a`分配一个`int`类型大小的存储空间，用来存放右值2。而这个存储空间的大小一般是4个字节。那么，这个4字节其实就体现出了变量a的类型。OK，这个时候如果我在程序的某个位置用到了变量a，例如：

```c
int b = a + 2;
```

那么CPU肯定是需要去内存中（不考虑缓存的影响）查找变量a并且取出里面的值对吧。对的，当我们找到了变量a的首地址之后，需要往后再去多少字节的数据呢？因为我们把变量a定义为了整型int，这种类型共占用了4字节的存储空间，所以当知道了变量a的首地址之后，还需要继续往后取3个字节的数据，然后把这四个字节的数据放到总线上面，最后传输给CPU。

再比如说，我把`int a =2; `改成`double a = 2;`那么，就会为变量a分配一个`double`类型大小的存储空间，用来存放右值2。而这个存储空间的大小一般是8个字节。那么这个8字节其实就体现出了变量a的类型。

通过上面的小例子，我们对变量的数据类型有了一个大概的了解。

上面说的是静态语言，但是对于PHP这门动态语言而言，数据类型就显得没有那么明显了，至少我们很少会在定义一个变量的时候去指明该变量的类型。不仅我们不会去指明某个变量的类型，甚至有时候，我们会在代码中这样写：

```php
<?php
$a = 1;
/* some code */
$a = 'I am a string';
```

很神奇吧，当初我学习PHP的时候也被这种行为震惊了（之前先学了一点C语言）。其实不是说PHP没有数据类型，小伙伴们可以在PHP7源码的`include/php/Zend/zend_types.h`里面找到PHP支持的数据类型：

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/1.PNG)

那么为什么PHP这门语言可以支持这样的写法呢？下面的内容就是我想要给大家分享的了。

这个问题要回答起来也很简单，既然PHP可以那么灵活，那么**在底层肯定就为PHP做了很多事情**咯。（我一直觉得PHP在某种意义上来讲是C写的一个框架，主要用于Web）

OK，能够让PHP如此灵活的原因其实就是在底层有一些数据结构支撑着它。我们先介绍一下`zval`这个数据结构。

## zval

同样，我们可以在PHP7源码`include/php/Zend/zend_types.h`中找到这个结构的定义：

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/2.PNG)

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/3.PNG)

这个结构体是用来保存PHP变量的所有信息的，包括变量的值等等信息。我们可以把它理解为一个容器，PHP变量的信息都往这个结构体里面丢。

我大致讲一下这个`zval`结构体里面的东西。

### zval_value  value

这个`zval_value`结构可以用来存放PHP变量的值。

它的定义是这样的：

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/4.PNG)

我们发现这个`zval_value  value`是一个联合体（我喜欢叫它**共用体**）。这个结构可以共用联合体里面变量的首地址。正是因为这个结构里面保存了众多的指向不同数据类型的内存空间的指针，使得PHP的类型切换变得简单了。

### u1

这个u1也是一个联合体，联合了一个结构体v和一个32位无符号整型`type_info`，`ZEND_ENDIAN_LOHI_4`宏是用来解决字节序问题的，我们可以不去管它。

`v`里面的`type`用于标识`zval_value  value`的类型。即：

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/1.PNG)

`v`里面的`type_flags`是类型掩码，用于变量的内存管理。

而u1里面的`type_info`实际上是将v结构的4个成员组合到了一起，v中的成员各占一个字节（共4字节）。所以`type_info`也是4个字节，每个字节对应v的一个成员。

### u2

这个结构体纯粹是用于一些辅助功能，zval结构的value、u1占用的空间分别为8字节、4字节，但是由于系统会进行字节对齐，这个zval结构体最终将占用16个字节。那么就有4个字节被浪费了。所以zval定义了一个u2结构把这4个字节利用了。

嗯，通过这个结构，我们发现，无论PHP的变量类型是什么，都是需要把value结构开到8字节的内存大小的，所以可以看出，PHP因为类型灵活而付出了浪费空间的代价。

## 字符串

PHP中并没有使用char来表示字符串，而是为字符串单独定义了一个结构：`zend_string`。在`zend_value`中通过`str`指向具体的结构。

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/5.PNG)

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/6.PNG)

该结构有4个成员：

- gc：变量的引用计数信息，用于内存管理。
- h：字符串通过 Time33 算法计算得到的Hash Code。
- len：字符串长度。
- val：字符串内容。

我们发现，用来存储字符串内容的结构有些奇怪，并不是直接使用`char*`指针来保存一个字符串内容的地址。而是使用了一个可变数组。正是因为数组是可变的，所以我们把`val[1]`放在了zend_string结构体的最后。val[1]并不代表它只能存储一个字节（可以存储多少与我们为这个结构体分配的内存大小有关），在字符串分配时实际上是类似这样的操作的：

```c
malloc(sizeof(zend_string) + 字符串长度)
```

也就是会多分配一些内存，而多出的这块内存的其实地址就是val（在表达式中数组名可以和指针互换），这样就可以直接将字符串内容存储到val中，通过val进行读取。如果val是一个指针`char*`，则需要额外进行一次分配内存的操作（即我们要让char* 指针指向新malloc出来的一片内存。这样很麻烦，到时候自己还得手动用free去释放这个char* 指针所指向的内存，容易造成内存泄漏），可变长度的结构体不仅可以省一次内存分配，而且有助于内存管理，free时直接释放zend_string即可（因为字符串是存在结构体中的，而没有另开内存去存放字符串）。

另外需要注意的地方是，val多出来的一个字节（结构体中为val[1]而不是val[0]），用于存放字符串的最后一个字符：`'\0'`，比如：

```php
$a = 'abc';
```

对应的zend_string内存结构如图：

![](http://oklbfi1yj.bkt.clouddn.com/PHP%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90%28%E4%BA%8C%29-%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/7.PNG)







