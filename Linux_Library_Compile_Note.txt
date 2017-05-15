3. 静态库

有时候需要把一组代码编译成一个库，这个库在很多项目中都要用到，例如libc就是这样一个库，我们在不同的程序中都会用到libc中的库函数（例如printf），也会用到libc中的变量（例如以后要讲到的environ变量）。本节介绍怎么创建这样一个库。

我们继续用stack.c的例子。为了便于理解，我们把stack.c拆成四个程序文件（虽然实际上没太大必要），把main.c改得简单一些，头文件stack.h不变，本节用到的代码如下所示：

/* stack.c */
char stack[512];
int top = -1;
/* push.c */
extern char stack[512];
extern int top;

void push(char c)
{
	stack[++top] = c;
}
/* pop.c */
extern char stack[512];
extern int top;

char pop(void)
{
	return stack[top--];
}
/* is_empty.c */
extern int top;

int is_empty(void)
{
	return top == -1;
}
/* stack.h */
#ifndef STACK_H
#define STACK_H
extern void push(char);
extern char pop(void);
extern int is_empty(void);
#endif
/* main.c */
#include <stdio.h>
#include "stack.h"

int main(void)
{
	push('a');
	return 0;
}
这些文件的目录结构是：

$ tree
.
|-- main.c
`-- stack
    |-- is_empty.c
    |-- pop.c
    |-- push.c
    |-- stack.c
    `-- stack.h

1 directory, 6 files
我们把stack.c、push.c、pop.c、is_empty.c编译成目标文件：

$ gcc -c stack/stack.c stack/push.c stack/pop.c stack/is_empty.c
然后打包成一个静态库libstack.a：

$ ar rs libstack.a stack.o push.o pop.o is_empty.o
ar: creating libstack.a
库文件名都是以lib开头的，静态库以.a作为后缀，表示Archive。ar命令类似于tar命令，起一个打包的作用，但是把目标文件打包成静态库只能用ar命令而不能用tar命令。选项r表示将后面的文件列表添加到文件包，如果文件包不存在就创建它，如果文件包中已有同名文件就替换成新的。s是专用于生成静态库的，表示为静态库创建索引，这个索引被链接器使用。ranlib命令也可以为静态库创建索引，以上命令等价于：

$ ar r libstack.a stack.o push.o pop.o is_empty.o
$ ranlib libstack.a
然后我们把libstack.a和main.c编译链接在一起：

$ gcc main.c -L. -lstack -Istack -o main
-L选项告诉编译器去哪里找需要的库文件，-L.表示在当前目录找。-lstack告诉编译器要链接libstack库，-I选项告诉编译器去哪里找头文件。注意，即使库文件就在当前目录，编译器默认也不会去找的，所以-L.选项不能少。编译器默认会找的目录可以用-print-search-dirs选项查看：

$ gcc -print-search-dirs
install: /usr/lib/gcc/i486-linux-gnu/4.3.2/
programs: =/usr/lib/gcc/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/:/usr/lib/gcc/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/:/usr/libexec/gcc/i486-linux-gnu/4.3.2/:/usr/libexec/gcc/i486-linux-gnu/:/usr/lib/gcc/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../../i486-linux-gnu/bin/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../../i486-linux-gnu/bin/
libraries: =/usr/lib/gcc/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../../i486-linux-gnu/lib/i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../../i486-linux-gnu/lib/../lib/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../i486-linux-gnu/4.3.2/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../../lib/:/lib/i486-linux-gnu/4.3.2/:/lib/../lib/:/usr/lib/i486-linux-gnu/4.3.2/:/usr/lib/../lib/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../../i486-linux-gnu/lib/:/usr/lib/gcc/i486-linux-gnu/4.3.2/../../../:/lib/:/usr/lib/
其中的libraries就是库文件的搜索路径列表，各路径之间用:号隔开。编译器会在这些搜索路径以及-L选项指定的路径中查找用-l选项指定的库，比如-lstack，编译器会首先找有没有共享库libstack.so，如果有就链接它，如果没有就找有没有静态库libstack.a，如果有就链接它。所以编译器是优先考虑共享库的，如果希望编译器只链接静态库，可以指定-static选项。

那么链接共享库和链接静态库有什么区别呢？在第 2 节 “main函数和启动例程”讲过，在链接libc共享库时只是指定了动态链接器和该程序所需要的库文件，并没有真的做链接，可执行文件main中调用的libc库函数仍然是未定义符号，要在运行时做动态链接。而在链接静态库时，链接器会把静态库中的目标文件取出来和可执行文件真正链接在一起。我们通过反汇编看上一步生成的可执行文件main：

$ objdump -d main
...
08048394 <main>:
 8048394:       8d 4c 24 04             lea    0x4(%esp),%ecx
 8048398:       83 e4 f0                and    $0xfffffff0,%esp
 804839b:       ff 71 fc                pushl  -0x4(%ecx)
...
080483c0 <push>:
 80483c0:       55                      push   %ebp
 80483c1:       89 e5                   mov    %esp,%ebp
 80483c3:       83 ec 04                sub    $0x4,%esp
有意思的是，main.c只调用了push这一个函数，所以链接生成的可执行文件中也只有push而没有pop和is_empty。这是使用静态库的一个好处，链接器可以从静态库中只取出需要的部分来做链接。如果是直接把那些目标文件和main.c编译链接在一起：

$ gcc main.c stack.o push.o pop.o is_empty.o -Istack -o main
则没有用到的函数也会链接进来。当然另一个好处就是使用静态库只需写一个库文件名，而不需要写一长串目标文件名。
