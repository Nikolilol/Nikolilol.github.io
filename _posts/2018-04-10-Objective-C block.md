---
title: Objective-C block
teaser: 关于 block 的使用和底层原理
category: Objective-C
tags: [markdown]
---
# Objective-C block 使用和底层原理
  block 也叫做闭包，是指向函数的指针（同时也是一个函数）。再加上该函数执行的外部的上下文变量（有时候也称作自由变量）

## block 使用
#### 声明定义和调用：
    void (^blockName)(int arg1, int arg2) = ^void(int arg1, int arg2) 
    {
    	NSLog(@"arg1+arg2 = %d", arg1 + arg2);
    }
    blockName(1,2);
#### block 没有参数，又返回值，作为方法的参数：
	- (void)viewDidLoad {
		//1. 没有参数
		void (^blockName2)() = ^() {
		NSLog(@"block2");
		};
		blockName2();
		    
		//2. block有返回值
		int (^blockName3)(int) = ^(int n) {
		return n * 2;
		};
		
		//3. block作为方法的参数
		[self testBlock2:blockName3];
	}
	
	- (void)testBlock2:(int(^)(int))myBlock {
		myBlock(10);
	}
block 只有在调用的时候才会执行里面的函数内容。

## block 调用外部变量
#### 全局变量，block 可以进行读取和修改

	@interface ViewController () {
		NSInteger num;
	}

	@implementation ViewController
	
	- (void)viewDidLoad {
	    //1、block修改成员变量
	    void (^block1)() = ^(){
	        ++num;
	        NSLog(@"调用成员变量： %ld", num);
	    };
	    
	    block1();
	}
#### 局部变量，block 只能读取，不能修改局部变量。这个时候是值传递。
如果想修改局部变量，要用__block来修饰。这个时候是引用传递。下面会聊下block的实现原理。
	
	//2、调用局部变量，不用__block
	NSInteger testNum2 = 10;
	void (^block2)() = ^() {
	    //testNum = 1000; 这样是编译不通过的
	    NSLog(@"修改局部变量： %ld", testNum2); //打印：10
	};
	testNum2 = 20;
	block2();
	NSLog(@"testNum最后的值： %ld", 20);//打印：30
	
	//3、修改局部变量，要用__block
	__block NSInteger testNum3 = 10;
	void (^block3)() = ^() {
	    NSLog(@"读取局部变量： %ld", testNum3); //打印：20
	    testNum3 = 1000;
	    NSLog(@"修改局部变量： %ld", testNum3); //打印：1000
	};
	testNum3 = 20;
	block3();
	testNum3 = 30;
	NSLog(@"testNum最后的值： %ld", testNum3);//打印：30
## block 代码分析

### 1. block 不包含任何变量
	#import <Foundation/Foundation.h>
	
	int main()
	{
		void (^blockName)() = ^{
			NSLog(@"打印 block 函数");
		};
		blockName();
		return 0;
	}
命令行 cd 到文件目录，执行 clang 命令：
	
    clang -rewrite-objc test.m

打开生成的 .cpp 文件。其核心代码主要在文件的头部和底部。
	
	// __block_impl 是 block 的结构体
	struct __block_impl {
		void *isa;		// isa 指针，OC 中任何对象都有 isa 指针。类似协议。
		int Flags;		// 按 bit 位表示一些 block 的附加信息。在 block copy 的实现时就会使用。
		int Reserved;	// 保留变量
		void *FuncPtr;	// 函数指针，指向 block 声明的方法。
	};
	// __main_block_func_0 是 block 中定义要实行的函数
	static void __main_block_func_0(struct 	__main_block_impl_0 *__cself) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_38__5s24cwj7qv8qcnzkwjpvd2r0000gn_T_test_42ed96_mi_0);
	};
	// __main_block_desc_0 是 block 描述信息 的结构体
	static struct __main_block_desc_0 {
		size_t reserved;	// 结构体保留字段
		size_t Block_size;	// 结构体大小
	} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
	// __main_block_impl_0 是该 block 的实现，也是 block 实现的入口
	struct __main_block_impl_0 {
		struct __block_impl impl; // imp 为 block 类型的变量，函数里对它的 impl 的 isa, Flags, FunPtr 来进行赋值。
		struct __main_block_desc_0* Desc;
		__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
		impl.isa = &_NSConcreteStackBlock; // block 的三种类型：_NSConcreteStackBlock, _NSConcreteGlobalBlock, _NSConcreteMallocBlock.
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
		}
	};
		
	int main()
	{
		// 定义并初始化了 block 类型的变量
		void (*blockName)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
		// 调用 block
		((void (*)(__block_impl *))((__block_impl *)blockName)->FuncPtr)((__block_impl *)blockName);
		return 0;
	}
	
#### 1.1 block 结构体：
    struct __block_impl{
    	void *isa;
    	int Flags;
    	int Reserved;
    	void *FuncPtr;
    };
    
isa：isa指针，在 OC 中，任何对象都有 isa 指针。block 有三种类型：

_NSConcreteGlobalBlock：全局的静态 block，类似函数。如果block里不获取任何外部变量。或者的变量是全局作用域时，如成员变量、属性等； 这个时候就是 Global 类型

_NSConcreteStackBlock：保存在栈中的 block，栈都是由系统管理内存，当函数返回时会被销毁。\_\_block 类型的变量也同样被销毁。为了不被销毁，block 会将 block 和 \_\_block变量从栈拷贝到堆。

_NSConcreteMallocBlock：保存在堆中的 block，堆内存可以由开发人员来控制。当引用计数为 0 时会被销毁。

代码执行的时候，block 的 isa 有上面3种值。后面还会进行详细的说明。

#### 1.2 \_\_main\_block\_func\_0 是 block 要执行的函数：
    static void __main_block_func_0(struct 	__main_block_impl_0 *__cself) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_38__5s24cwj7qv8qcnzkwjpvd2r0000gn_T_test_42ed96_mi_0);
	};
	
#### 1.3 \_\_main\_block\_desc\_0 是 block 描述信息的结构体

#### 1.4 block 的类型。
从上面的代码可以看到：

	impl.isa = &_NSConcreteStackBlock;

这里 impl.isa 的类型为 \_NSConcreteStackBlock，由于 clang 改写的具体实现方式和 LLVM 不太一样，所以这里 isa 指向的还是_NSConcreteStackBlock。但在 LLVM 的实现中，开启 ARC 时，block 应该是 _NSConcreteGlobalBlock 类型。

所以 block 是什么类型 在 clang 代码里是看不出来的。

如果要查看block的类型还是要通过Xcode进行打印：
	
	- (void)clangCode{
		void (^clangBlk)() = ^{
			NSlog(@"打印block函数");
		};
		NSLog(@"clangBlk = %@", clangBlk);
		clangBlk();
	}
	
打印结果:
	
	clangBlk = <__NSGlobalBlock__: 0x100054240>

上面block代码，没有获取任何外部变量，应该是 _NSConcreteGlobalBlock类型的。该类型的block和函数一样 存放在 代码段 内存段。内存管理简单。

### 2. block 访问 局部变量
新建 test2.m 文件，代码如下：
	
	int main(){
   		int num = 1;
    	void (^blockName)() = ^(){
       	printf("num = %d", num);
    	};
    	blockName();
    	return 0;
	}
	
通过 clang 命令生成 的核心代码如下，和刚才 clang 的代码 不同的地方 已经加了注释：
	
	struct __block_impl {
		void *isa;
		int Flags;
		int Reserved;
		void *FuncPtr;
	};
	struct __main_block_impl_0 {
		struct __block_impl impl;
		struct __main_block_desc_0* Desc;
		int num;	// 1. 多了一个变量 num, block 访问局部变量，赋值给了这个 int 变量。
		__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _num, int flags=0) : num(_num) {
			impl.isa = &_NSConcreteStackBlock;	// 2. 注意这里的 stackBlock 是不正确的，在 Xcode 上打印 block 类型为 __NSMallocBlock。
			impl.Flags = flags;
			impl.FuncPtr = fp;
			Desc = desc;
		}
	};
	
	static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	int num = __cself->num; // bound by copy 3. 在实现方法中，将 __main_block_impl_0 的 num 赋值给 __main_block_func_0 里的变量。
	    printf("num = %d", num);
	}
	
	static struct __main_block_desc_0 {
		size_t reserved;
		size_t Block_size;
	} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
	
	int main(){
		int num = 1;
		void (*blockName)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, num));
		((void (*)(__block_impl *))((__block_impl *)blockName)->FuncPtr)((__block_impl *)blockName);
		return 0;
	}
	
#### 2.1 block 持有局部变量
可以看到 \_\_main\_block\_impl\_0 中添加了 一个 int num 的变量。在 \_\_main\_block\_func\_0 中使用了该变量。从这里可以看出来 这里是 值拷贝，不能修改，只能访问。

#### 2.2 block 访问局部变量时的类型
用Xcode打印上面block代码，得到的类型为：\_\_NSMallocBlock。

在说 \_NSConcreteMallocBlock 类型前，我们先说下\_NSConcreteStackBlock 类型。

\_NSConcreteStackBlock 类型的 block 存在栈区，当变量作用域结束的时候，这个 block 和 block 上的 \_\_block 变量就会被销毁。

这样当 block 获取了局部变量，在其他地方访问的时候就会崩溃。 block 通过 copy 来解决了这个问题，可以将 block 从栈拷贝到堆。这样当栈上的作用域结束后，仍然可以访问 block 和 block 中的外部变量。

#### 2.3 什么情况下block会进行copy操作。
用代码显示的调用copy操作：

	[captureBlk2 copy];
在 MRC 下 block 定义的属性都要加上 copy，ARC 的时候 block 定义 copy 或 strong 都是可以的，因为 ARC 下 strong 类型的 block 会自动完成 copy 的操作。

	@property (nonatomic, strong) captureObjBlk2 captureBlk21;
	
当 block 作为函数返回值返回时。

当 block 被赋值给 __strong id 类型的对象或 block 的成员变量时。

当 block 作为参数被传入方法名带有 usingBlock 的 Cocoa Framework 方法或 GCD 的 API 时。

### 3. __block 在 block 中的作用.
新建 test3.m，代码如下：

	int main(){
		__block int num = 1;
		void (^blockName)() = ^(){
		    num = 10;
		    printf("num = %d", num);
		};
		blockName();
		return 0;
	}

用 clang 生成的代码如下，进行了详细的注释：

	struct __block_impl {
		void *isa;
		int Flags;
		int Reserved;
		void *FuncPtr;
	};
	
	// 用于封装 __block 修饰的外部变量
	struct __Block_byref_num_0 {
		void *__isa;	// 对象指针
		__Block_byref_num_0 *__forwarding;	// 指向 拷贝到堆上的 指针
		int __flags;	// 标志位置量
		int __size;		// 结构体大小
		int num;		// 外部变量
	};
	struct __main_block_impl_0 {
		struct __block_impl impl;
		struct __main_block_desc_0* Desc;
		__Block_byref_num_0 *num; // by ref __block int num 变成了 __Block_byref_num_0 指针变量，也就是 __block 的变量通过指针传递给 block
		__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_num_0 *_num, int flags=0) : num(_num->__forwarding) {
			impl.isa = &_NSConcreteStackBlock;
			impl.Flags = flags;
			impl.FuncPtr = fp;
			Desc = desc;
		}
	};
	
	static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
		__Block_byref_num_0 *num = __cself->num; // bound by ref
	
		(num->__forwarding->num) = 10;
		printf("num = %d", (num->__forwarding->num));
	}
    
	static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
		// _Block_object_assign 函数：当 block 从栈拷贝到堆时，调用此函数。
		_Block_object_assign((void*)&dst->num, (void*)src->num, 8/*BLOCK_FIELD_IS_BYREF*/);
	}
	
	// 当 block 从堆内存中释放时，调用此函数：__main_block_dispose_0
	static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->num, 8/*BLOCK_FIELD_IS_BYREF*/);}

	static struct __main_block_desc_0 {
	  size_t reserved;
	  size_t Block_size;
	  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
	  void (*dispose)(struct __main_block_impl_0*);
	} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

	int main(){
	    __attribute__((__blocks__(byref))) __Block_byref_num_0 num = {(void*)0,(__Block_byref_num_0 *)&num, 0, sizeof(__Block_byref_num_0), 1};
	    void (*blockName)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_num_0 *)&num, 570425344));
	    ((void (*)(__block_impl *))((__block_impl *)blockName)->FuncPtr)((__block_impl *)blockName);
	    return 0;
	}

block 访问的外部变量，\_\_block 修饰的变量在 block 中就是一个结构体：__Block_byref_num_0：
	
	// 一、用于封装 __block 修饰的外部变量
	struct __Block_byref_num_0 {
	    void *__isa;    // 对象指针
	    __Block_byref_num_0 *__forwarding;  // 指向 拷贝到堆上的 指针
	    int __flags;    // 标志位变量
	    int __size;     // 结构体大小
	    int num;        // 外部变量
	};

其中 int num 为外部变量名。

\_\_Block\_byref\_num\_0 *\_\_forwarding; 这个是指向自己堆上的指针，这个后面会详细说明。

为了对 \_\_Block\_byref\_num\_0 结构体进行内存管理。新加了 copy 和 dispose 函数：

	//四、对__Block_byref_num_0结构体进行内存管理。新加了copy和dispose函数。
	static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
	    // _Block_object_assign 函数：当 block 从栈拷贝到堆时，调用此函数。
	    _Block_object_assign((void*)&dst->num, (void*)src->num, 8/*BLOCK_FIELD_IS_BYREF*/);
	}
	
	// 当 block 从堆内存释放时，调用此函数:__main_block_dispose_0
	static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->num, 8/*BLOCK_FIELD_IS_BYREF*/);}
	
\_\_main\_block\_impl\_0 中增加了 \_\_Block\_byref\_num\_0 类型的指针变量。所以 \_\_block 的变量之所以可以修改 是因为 指针传递。所以 block 内部修改了值，外部也会改变：

	struct __main_block_impl_0 {
		struct __block_impl impl;
		struct __main_block_desc_0* Desc;
		__Block_byref_num_0 *num; // 二、__block int num  变成了 __Block_byref_num_0指针变量。也就是 __block的变量通过指针传递给block
		__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_num_0 *_num, int flags=0) : num(_num->__forwarding) {
		    impl.isa = &_NSConcreteStackBlock;
		    impl.Flags = flags;
		    impl.FuncPtr = fp;
		    Desc = desc;
		}
	};
	
在 block 要执行的函数 \_\_main\_block\_func\_0 中，我们通过 \_\_Block\_byref\_num\_0 的 \_\_forwarding 指针来修改的 外部变量，即：(num->\_\_forwarding->num) = 10;

	static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_num_0 *num = __cself->num; // bound by ref
    
    (num->__forwarding->num) = 10;  //三、这里修改的是__forwarding 指向的内存的值
    printf("num = %d", (num->__forwarding->num));
	}
	
使用 \_\_block 修饰变量，在 block 内外的值时互相影响的。


---
