---
layout: post
title: "IOS内存管理简析"
date: 2014-09-01 21:43:41 +0800
comments: true
categories: IOS
---

##内存管理与引用计数

###引用计数式内存管理的思考方式

引用计数式的内存管理方式主要涉及四个操作：生成对象，持有对象，释放对象，废弃对象。

* 内存管理的思考方式
	
	+ 自己生成的对象，自己持有
	+ 非自己生成的对象，自己也能持有
	+ 不再需要自己持有的对象时释放
	+ 非自己持有的对象无法释放

与oc中方法对应如下：

	+对象操作						+Oc方法
	生成并持有对象				alloc/new/copy/mutablecopy等方法
	持有对象						retain方法
	释放对象						release方法
	废弃对象						dealloc方法
	
* 自己生成的对象，自己持有

使用alloc, new, copy, mutablecopy开头的方法意味着自己生成的对象自己持有

注：命名要符合驼峰命名法。

* 非自己生成的对象，自己也能持有

通过retain方法，非自己生成的对象也可以持有对象了。

* 不再需要自己持有的对象时释放

自己持有的对象，一旦不再需要，持有者有义务释放该对象。释放使用release方法。

使用autorelease方法可以使取得的对象存在，但是自己不持有对象。

* 无法释放非自己持有的对象

释放非自己持有的对象会造成程序的崩溃。

###GNUSTEP和APPLE中alloc/retain/release/dealloc的实现

在申请内存的时候，会在内存头部之前申请一块内存用来保存引用计数。然后分别在不同的操作的时候对这一引用计数值进行判断并作相应调整。总结如下：

	1.调用alloc或是retain方法后，引用计数值加1。
	2.调用release之后，引用计数值减1
	3.引用计数值为0时，调用delloc方法废弃对象
	
苹果在实现上述方式时没有使用内存头部管理引用计数，而是使用采用了散列表管理引用计数

##autorelease

###自动释放--autorelease

aurorelease就是自动释放，类似于c语言中的自动变量。

autorelease的具体使用方法：

	1，生成并持有NSAutoreleasePool对象
	2，调用已分配对象的autorelease实例方法
	3，废弃NSAutoreleasePool对象

在Cocoa框架中，相当于程序主循环的NSRunloop或者在其他程序可运行的地方，对NSAutoreleasePool对象进行生成，持有和废弃处理。

##ARC规则

###所有权修饰符：
* _strong修饰符
* _weak修饰符
* _unsafe_unretained修饰符
* _autoreleasing修饰符

####_strong修饰符
_strong修饰符是id类型和对象类型默认的所有权修饰符。附有_strong修饰符的变量在超出其变量作用域时，释放其被赋予的对象。_strong修饰符表示对对象的“强引用”，持有强引用的变量在超出其作用域时被放弃，随着强引用的失效，引用的对象会随之释放。通过_strong修饰符，不必再次键入retain或者release，就完美的满足了“引用计数式内存管理的思考方式”。

注：id类型和对象类型的所有权修饰符默认为_strong修饰符。

####_weak修饰符
如果仅使用_strong修饰符会造成“循环引用”的问题，这个时候就用到了_weak。带有_weak修饰符的变量不持有对象，所以在超出其变量作用域时，对象即被释放。_weak的另一个优点是，在持有某对象的弱引用时，若该对象被释放，则此弱引用将自动失效且处于nil被赋值状态。

####_unsafe_unretained修饰符
在ios5以下版本中无法使用_weak，所以_unsafe_unretained成为替代品。但是_unsafe_unretained不保证最后的对象被释放，且处于nil被赋值状态。所以赋值给附有_unsafe_unretained修饰符变量的对象在使用时，要确保它存在。

####_autoreleasing修饰符
在ARC有效时，用“@autoreleasepool块”来代替“NSAutoreleasePool”。

编译器会检查方法名是否以alloc/new/copy/mutableCopy开始，如果不是则自动将返回值的对象注册到autoreleasepool(注：init返回值的对象不注册到autoreleasepool)。在使用附有_weak修饰符的变量时必须访问注册到autoreleasepool的对象，因为_weak修饰符持有对象的弱引用，在访问引用对象的过程中，该对象可能被丢弃，所以把要访问的对象注册到autoreleasepool中，那么在@autoreleasepool结束之前多能确保该对象存在。

###属性
		  属性声明的属性						所有权修饰符
			assign							_unsafe_unretained
			copy							_strong
			retain							_strong
			strong							_strong
			unsafe_unretained				_unsafe_unretained
			weak							_weak
其中：
assign：一般基本变量用该属性声明，eg:int BOOL

copy：声明的变量是拷贝赋值源所生成的对象

retain和strong表示意思相同

strong：属性所声明的变量将成为对象的持有者

unsafe_unretained：ios5之前的系统用该属性代替weak

[关于ARC认识很好的博文](http://www.yifeiyang.net/development-of-the-iphone-simply-1/),一个系列，值得一读。
	
	
