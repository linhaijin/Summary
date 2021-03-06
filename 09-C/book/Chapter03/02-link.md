静态链接与动态链接
===

### Linux 目标文件种类

Linux 系统的目标文件 (Object File) 主要有如下三种类型：

1) 可重定位文件 (relocatable file)，可与其它目标文件一起创建可执行文件或共享目标文件。

```
$ gcc -g -c hello.c
$ file hello.o
hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
```
2) 可执行文件 (executable file)。操作系统可创建一个进程执行该文件。

```
$ gcc -g hello.c -o hello
$ file hello
hello: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared
libs), for GNU/Linux 2.6.15, not stripped
```

3) 共享目标文件 (shared object file)，通常是 "函数库"，可静态链接到其他 ELF（Executable and Linking Format，为目标文件格式） 文件中，或动态链接共同创建进程映像 (类似 DLL)。

```
$ gcc -shared -fpic stack.c -o hello.so
$ file hello.so
hello.so: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, not
stripped
```

汇编程序生成的是可重定位目标文件。后两种目标文件还要进行链接处理才可得到。每个源文件都会首先被编译成可重定位目标文件，
每个目标文件都提供一些别的目标文件需要的函数或者数据，同时又从别的目标文件中获得一些函数或者数据。因此，链接的过程就是目标文件间互通有无的过程。

### 链接

链接处理可分为两种：静态链接和动态链接。如下面的例子：

```c
#include <stdio.h>

int main() {
    printf("Hello World!\n");
    return 0;
}
```
main 函数中引用了标准库提供的 printf 函数。链接所要解决的问题就是让程序能正确找到 printf 函数。解决这个问题有两个办法：

1. 在生成可执行文件时，把 printf 函数相关的二进制指令和数据包含在最终的可执行文件中，即静态链接。
2. 在程序运行时，再去加载 printf 函数相关的二进制指令和数据，即动态链接。

### 静态链接

静态链接就是在生成可重定位目标文件时，把所有需要的函数的二进制代码都包含到该文件中。
因此，链接器需要知道参与链接的目标文件需要哪些函数，同时也要知道每个目标文件都能提供什么函数，
这样链接器才能知道是不是每个目标文件所需要的函数都能正确地链接。如果某个目标文件需要的函数在参与链接的目标文件中找不到，链接器就会报错。
目标文件中有两个重要的接口来提供这些信息：一个是符号表，另外一个是重定位表。利用 Linux 中的 readelf 工具可查看这些信息。

用命令 `gcc -c -o main.o main.c` 来编译上文的 main.c 文件来生成目标文件 main.o。
然后用命令 `readelf -s main.o` 查看 main.o 中的符号表：

```
Symbol table '.symtab' contains 11 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 00000000     0 SECTION LOCAL  DEFAULT    1
     3: 00000000     0 SECTION LOCAL  DEFAULT    3
     4: 00000000     0 SECTION LOCAL  DEFAULT    4
     5: 00000000     0 SECTION LOCAL  DEFAULT    5
     6: 00000000     0 SECTION LOCAL  DEFAULT    7
     7: 00000000     0 SECTION LOCAL  DEFAULT    8
     8: 00000000     0 SECTION LOCAL  DEFAULT    6
     9: 00000000    36 FUNC    GLOBAL DEFAULT    1 main
    10: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
```

从最后两行中可以看到 main.o 中提供 main 函数（Type 列为 FUNC，Ndx 为 1 表示它是在本目标文件中第 1 个 Section 中），
main.o 也同时依赖于 printf 函数（Ndx 列为 UND）。

因为在编译 main.c 时，编译器还不知道 printf 函数的地址，所以在编译阶段只是将一个临时地址放到目标文件中，
在链接阶段，该临时地址将被修正为正确的地址，这个过程叫重定位。
所以链接器还要知道该目标文件中哪些符号需要重定位，这些信息是放在了重定位表中。
显然在 main.o 这个可重定位目标文件中，printf 的地址需要重定位，用命令 `readelf -r main.o` 验证一下，这些信息保存在 `.rel.textSection` 中：

```
Relocation section '.rel.text' at offset 0x400 contains 2 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0000000a  00000501 R_386_32          00000000   .rodata
00000019  00000a02 R_386_PC32        00000000   printf
```

既然 main.o 依赖于 printf 函数，那么 printf 在哪个目标文件里？

printf 函数是标准库的一部分，在 Linux 下静态的标准库 libc.a 位于 `/usr/lib/i386-linux-gnu/` 中。
可认为标准库就是把一些常用的函数的目标文件打包在一起，用命令 `ar -t libc.a` 可查看 `libc.a` 中的内容，其中便有 printf.o 文件。
链接时，需要告诉链接器需要链接的目标文件和库文件（默认 gcc 会把标准库作为链接器输入的一部分）。链接器会根据输入的目标文件从库文件中提取需要的目标文件。
如，链接器发现 main.o 会需要 printf 这个函数，在处理标准库文件的时候，链接器就会把 printf.o 从库文件中提取处理。
当然 printf.o 依赖的目标文件也很被一起提取出来。库中其他目标文件就被舍弃，以减小最终生成的可执行文件的大小。

知道了这些信息后，链接器即可开始工作，分为如下两个步骤：

1. 合并相似段，把所有需要链接的目标文件的相似段放在可执行文件的对应段中。
2. 重定位符号使得目标文件能正确调用到其他目标文件提供的函数。

用命令 `gcc -static -o helloworld.static main.c` 来编译并做静态链接，生成可执行文件 helloworld.static。
因为可执行文件 helloworld.static 已经是链接之后的，因此不含重定位表。
命令 `readelf -S helloworld.static | grep .rel.text`将不会有任何输出（注：-S 表示打印出 ELF 文件中的 Sections）。

或者通过 `gcc -c main.c`，然后通过 `ar -rcs main.a main.o` 生成 main.a 静态库。
经过静态链接生成的可执行文件，只要载入内存即可运行。

### 使用静态链接库

假设 main.c 中 引用了 静态库 libhello.a 中的函数。

- 使用方式一：

```
gcc -o test main.c libhello.a
```

生成 可执行文件 test。

- 使用方式二：

```
gcc -o test main.c -L./ -lhello
```

其中 `-L./` 必不可少，否则提示 `cannot find -lhello`，因为不加 `-L./`，仅会在系统默认的路径下查找 hello 函数库。需要显式指定当前目录。

- 头文件默认搜索路径如下：

```
/usr/local/include
/usr/lib/gcc/x86_64-linux-gnu/4.7.3/include
/usr/include
```

- 库文件默认搜索路径如下：
```
/usr/lib/gcc/x86_64-linux-gnu/4.7.3/
/lib/
/usr/lib/
```

### 动态链接

静态链接缺点之一就是对磁盘空间和内存空间的浪费，其所用的标准库函数会被放到每一个静态链接的可执行文件中。
运行时，这些重复的内容也会被不同的可执行文件载入内存。如果静态库有更改，所有可执行文件都需要重新链接。

动态链接便是为了解决该问题。所谓动态链接就是在运行的时候再去链接。理解动态链接需要从两个角度来看：

1. 从动态库的角度。
2. 从使用动态库的可执行文件的角度。

从动态库的角度来看，动态库与可执行文件一样，有其代码段和数据段。
若要动态库在内存中只有一份，需要做到无论动态库装载到什么位置，都不需要修改动态库中代码段的内容，从而实现动态库中代码段共享。
而数据段中的内容需要做到进程间的隔离，因此必须是私有的，也就是每个进程都有一份。
因此，动态库的做法是把代码段中变化的部分（主要包括对外部函数和变量的引用）放到数据段中去，这样代码段中剩下的就是不变的内容，即可载入到虚拟内存的任意位置。

假设把下面的代码做成一个动态库：

```c
#include <stdio.h>

extern int shared;
extern void bar();

void foo(int i) {
    printf("Printing from Lib.so %d\n", i);
    printf("Printing from Lib.so, shared %d\n", shared);
    bar();
    sleep(-1);
}
```

通过 `gcc -shared -fPIC -o Lib.so Lib.c` 生成一个动态库 Lib.so（ -shared 表示生成共享对象，-fPIC 表示生成地址无关的代码）。
该动态库提供（导出）一个函数 foo，依赖（导入）一个函数 bar，和一个变量 shared。

问题在于如何让 foo 函数正确地引用到外部函数 bar 和 shared 变量？

我们知道，程序装载有个特性，代码段和数据段的相对位置是固定的，
因此把这些外部函数和外部变量的地址放到数据段的某个位置，这样代码即可根据其当前的地址从数据段中找到对应外部函数的地址。

动态库中外部变量的地址是放在 `.got（global offset table)`）中，外部函数的地址是放在了.got.plt段中。
用命令 `readelf -S Lib.so | grep got` 会看到 Lib.so 中有这样两个 Section。这便是存放外部变量和函数地址的地方。


```
[20] .got              PROGBITS        00001fe4 000fe4 000010 04  WA  0   0  4
[21] .got.plt          PROGBITS        00001ff4 000ff4 000020 04  WA  0   0  4
```

可见动态库是把地址相关的内容放入数据段中，从而使得动态库能被多个进程共享。那么谁来帮助动态库修正 .got 和 .got.plt 中的地址？

从动态链接器的角度来看，静态链接的可执行文件之所以载入内存即可运行，是由于所有的外部函数都已经包含在可执行文件中。
而动态链接的可执行文件中对外部函数的引用地址在生成可执行文件的时候是未知的，所以在这些地址被修正前是动态链接生成的可执行文件是不能运行的。
因此，动态链接生成的可执行文件运行前，系统会首先将动态链接库加载到内存中，动态链接器所在的路径在可执行文件中可以查到。

还是以上文的 helloworld 为例：
通过 `gcc -o helloworld.dyn main.c` 以动态链接的方式生成可执行文件。

通过 `readelf -l helloworld.dyn | grep interpreter` 可看到动态链接器在系统中的路径。

```
[Requesting program interpreter: /lib/ld-linux.so.2]
```

当动态链接器被载入后，其首先找到该可执行文件所依赖的动态库，这部分信息也存放在可执行文件中。通过 `readelf -d helloworld.dyn` 可看到如下输出：

```
Dynamic section at offset 0xf28 contains 20 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
```

或用命令 `ldd helloworld.dyn` 可看到如下输出：

```
linux-gate.so.1 =>  (0x008cd000)
libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0x00a7a000)
/lib/ld-linux.so.2 (0x0035d000)
```

两种方式都可看出，可执行文件依赖于 libc.so.6 这个动态库，也就是 C 语言标准库的动态链接版本。
如果某个库依赖于别的动态库，它们也会被加载进来，直到所有依赖的库都被载入。

当所有的库均被载入后，动态链接器从各个动态库中可以知道每个库都提供什么函数（符号表）和哪些函数引用需要重定位（重定位表），
然后修正 `.got` 和 `.got.plt` 中的符号到正确的地址，完成之后就可以将控制权交给可执行文件的入口地址，从而开始执行代码逻辑。

可见，动态链接器在程序运行前需要做大量的工作（修正符号地址），为了提高效率，一般采用的是延迟绑定，也就是只有用到某个函数才去修正 `.got.plt` 中地址，详见《程序员的自我修养》一书。

### 动态库的使用

动态链接库的名称有别名（soname）、真名（realname）和链接名（linker name）:

1. 别名：libxxx.so，这种形式的库名正是执行编译命令时编译器要搜索的名字。
2. 真名：动态链接库的真实名称，一般总是在别名的基础上加上一个小版本号、发布版本等构成。
3. 链接名：程序链接时使用的库的名字。

- 动态链接库和静态链接库是一致的，使用 `-lxxx` 方式。

- 如果系统的搜索路径下同时存在静态链接库和动态链接库，默认情况下会链接动态链接库。如果需要强制链接静态链接库，需要加上 `-satic` 选项。

- 如果使用的是标准库中的函数，则只需将头文件包含进来。如果使用的函数所在的库不是标准库（例如我们自己编写的库或标准以外扩展的库），
则在编译时必须加入 `-lxxx` 选项，而头文件则可以包含也可以不包含，包含进来显得规范而已。

### 常用编译命令选项

```
- 无选项编译链接
用法：#gcc test.c
作用：将 test.c 预处理、汇编、编译并链接形成可执行文件。这里未指定输出文件，默认输出为 a.out。
      编译成功后可以看到生成了一个 a.out 的文件。在命令行输入 ./a.out 执行程序。

- 选项 -o
用法：#gcc test.c -o test
作用：将 test.c 预处理、汇编、编译并链接形成可执行文件 test。
      -o 选项用来指定输出文件的文件名。输入 ./test 执行程序。

- 选项 -g
作用：在编译的时候，产生调试信息。

- 选项 -E
用法：#gcc -E test.c -o test.i
作用：将 test.c 预处理输出 test.i 文件。

- 选项 -S
用法：#gcc -S test.i
作用：将预处理输出文件 test.i 汇编成 test.s 文件。

- 选项 -c
用法：#gcc -c test.s
作用：将汇编输出文件 test.s 编译输出 test.o 文件。

用法：gcc -c test.c
作用：仅做预处理、编译、汇编，即生成 test.o 的 obj 文件。

- 无选项链接
用法：#gcc test.o -o test
作用：将编译输出文件 test.o 链接成最终可执行文件 test。输入 ./test 执行程序。

- 选项 -O
用法：#gcc -O1 test.c -o test
作用：使用编译优化级别 1 编译程序。级别为 1~3 ，级别越大优化效果越好，但编译时间越长。输入 ./test 执行程序。

- 编译使用 C++ std 库的程序
用法：#gcc test.cpp -o test -lstdc++
作用：将 test.cpp 编译链接成 test 可执行文件。-lstdc++ 指定链接 std c++ 库。
```

### 多源文件的编译

- 多个文件一起编译。

```
将 test1.c 和 test2.c 分别编译后链接成 test 可执行文件
#gcc test1.c test2.c -o test
```

- 分别编译各个源文件，然后链接相应的目标文件。

```
#gcc -c test1.c                   // 将 test1.c 编译成 test1.o
#gcc -c test2.c                   // 将 test2.c 编译成 test2.o
#gcc -o test1.o test2.o -o test   // 将 test1.o 和 test2.o 链接成 test
```
第一种方法编译时需要所有文件重新编译，而第二种方法可以只重新编译修改的文件，未修改的文件不用重新编译。

### 程序库的链接

对于一些常用的函数的实现，gcc 编译器会自动去连接一些常用库。有些时候在编译程序时还要指定库的路径，这时要用到编译器的 `-L` 选项指定路径。
如一个库在 `/home/hoyt/mylib` 下，编译时要加上 `-L/home/hoyt/mylib`。对于一些标准库来说，不需要指出路径。只要它们在起缺省库的路径下即可。
系统的缺省库的路径 `/lib /usr/lib /usr/local/lib`，在这三个路径下面的库，可不指定路径。
