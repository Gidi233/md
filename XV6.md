# XV6

# lab 3

## kvminit

在proc_mapstacks给每个进程申请一页，地址相差两页（这样就有guard page了）procinit里把kstack分配（赋值）给进程

![image-20231027113742406](/home/lu/.config/Typora/typora-user-images/image-20231027113742406.png)

## 页表

kalloc后拿到的页表要`memset(p, 0, PGSIZE);`，在要用的时候自己设置PTE_V



## exec

进程页表在新进程初始化的时候将旧页表释放

## 问题

**usertest:**不知道为什么把usyscall放到proc里就不会lost some free pages（丢失了一些可用页面 内存泄漏？）（例子：https://www.cnblogs.com/duile/p/16087757.html），我都放到proc_pagetable里就不行，按理说fork什么的时候都复制了的。

# lab4

## trampolice

ecall后切换为supervisor mode，所有的进程的trampolice都映射到了trampolice.S的指令执行trap再跳转到usertrap

通过vm作为下标索引PTE（可转化为下一级页表的pa），三级索引（即索引三次）得到pa

页表、PTE是进程独立的，可以实现在相同pa在不同进程，权限不同

#### 为什么要static inline？

一个原因可能是因为里面基本都只有一句汇编，经过函数调用包一层让编译器来选择使用哪个寄存器，但会导致代码膨胀，所以要再内联，另一个是为了避免重复定义，但避免的是相同定义的重定义，如果是不同定义的函数，会产生未定义行为，选择占用空间最大的一个？？（这块有点乱，网上的中英文博客都查了，我都不确定inline是在编译期就替换，还是链接期，统一了弱符号后再替换，迷）补：应该是没展开的保留了弱符号， 可能链接期再挑一个展开

**以下是我找的各处的分析：**

因为根据编译过程来看，在头文件定义的static函数，当被源文件引用时，其static关键字的作用范围在引用该头文件的源文件，根据编译过程，首先是在编译最初时候头文件被加载到源文件，然后每一个源文件是独立的编译单元，也就是这些编译单元里面不能出现相同的函数定义，如果是inline函数被编译器inlined了，那么就类似于宏那样展开而不会存在函数的定义，但是如果inline函数没有被编译器inlined，那么源文件（也就是单独的编译单元）中就会存在inline函数的函数定义，如果存在两个以上的源文件在编译时编译器没有将他们inlined，那么也就是说这两个以上的源文件会定义同一个函数，也就是会重复定义。
但是当使用static关键字之后，就不会报重复定义的错误，因为其作用域被static修饰了，不是全局的。inline之所以不加修饰就可以被不同的源文件包含编译通过，是因为在每个编译单元中，其都成功被编译器inlined了，因此就在预处理的时候直接展开在代码里而没有编译生成对应的函数（就跟宏定义的效果一样），所以使用static能提高程序的健壮性，因为这样即使是没有被编译器成功inlined也是不会出错的

---

inline 关键字用于函数，有两个作用，第一个作用(相对老版本编译器来说)，就是前面说的(指令或者代码替换)；而第二个，使得在多个翻译单元(Translation Unit, 在此可以理解为.cc/.cpp等源文件)定义同名同参函数成为了可能。

------

Linux 内核作者 Linus 作者关于对 static inline 的解释：

> “static inline” means “we have to have this function, if you use it, but don’t inline it, then      make a static version of it in this compilation unit”. “extern inline” means “I actually have an extern for this function, but if you want to inline it, here’s the inline-version”.

我的理解是这样的：[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)函数为什么要定义在头文件中呢？因为它是一个[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)函数，可以像宏一样使用，任何想使用这个[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)函数的源文件，不必亲自再去定义一遍，直接包含这个头文件，即可像宏一样使用。那为什么还要用 static 修饰呢？因为我们使用 inline 定义的[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)函数，编译器不一定会[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)展开，那么当多个文件都包含这个[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)函数的定义时，编译时就有可能报重定义错误。而使用 static 修饰，可以将这个函数的作用域局限在各自本地文件内，避免了重定义错误。理解了这两点，就能够看懂 Linux 内核头文件中定义的大部分[内联](https://www.zhaixue.cc/c-arm/c-arm-inline.html)函数了。

------

 > If the user compiles with optimization off (`-O0` usually), then `inline` gets ignored and functions defined in headers get multiply instantiated (read: link fails), *unless* they are also marked `static`. This is not an unusual case: Sometimes, when hunting a difficult bug with a source-level debugger, you'll want inlining (and other optimizations) turned off to keep from breaking the single-step debug function.

  > 如果用户在优化关闭的情况下进行编译（通常为 `-O0` ），则 `inline` 将被忽略，并且标头中定义的函数将被多次实例化（读取：链接失败），除非它们也被标记为 `static` 。这并不是一个不寻常的情况：有时，当使用源代码级调试器寻找困难的错误时，您会希望关闭内联（和其他优化）以防止破坏单步调试功能。

# 问题

- cpuid是从哪来的，哪里写到tp或者proc->trapframe->kernel_hartid 的？（哪里初始化的trapframe）
- 在usertrapret函数中，为什么要设置trapframe中的数据，应该没有改变啊



![image-20240205130256910](/home/lu/.config/Typora/typora-user-images/image-20240205130256910.png)

![image-20240205130154013](/home/lu/.config/Typora/typora-user-images/image-20240205130154013.png)

![image-20240205130232830](/home/lu/.config/Typora/typora-user-images/image-20240205130232830.png)

![img](https://pic2.zhimg.com/v2-c53caaced467099a45c2c68d1b8b14bd_r.jpg)
