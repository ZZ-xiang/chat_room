# 熟悉 Linux 下 C++ 开发常用工具

### 实验介绍

在 Linux 下使用 C++ 开发我们往往会使用 g++ 来编译代码，并使用 gdb 工具进行代码调试。同时在大型的项目当中，我们会编写 makefile 来进行自动化编译。

#### 知识点

- 使用 g++ 编译器编译代码
- 使用 gdb 进行调试
- 编写 makefile

### 使用 g++ 编译代码

在 Linux 开发中，使用 g++ 对代码进行编译是必须掌握的重要技能。

#### C++ 编译过程

首先我们要明确在 C++ 中编译过程分为四个阶段：预处理、编译、汇编、链接：

- 预处理阶段主要负责宏定义的替换、条件编译、将 include 的头文件展开到正文等；
- 编译阶段负责将源代码转为汇编代码；
- 汇编阶段负责将汇编代码转为可重定位的目标二进制文件；
- 链接阶段负责将所有的目标文件（二进制目标文件、库文件等）连接起来，进行符号解析和重定位，最后生成可执行文件。

#### g++ 编译命令

假设我们有如下的 `test.cpp` 文件：

```cpp
#include<iostream>
using namespace std;
int add(int a,int b){
    return a+b;
}
int main(){
    int a=1,b=2;
    cout<<add(a,b)<<endl;
    return 0;
}
```

预处理阶段的结果是直接在终端输出，可由以下命令得到：

```bash
g++ -E test.cpp
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/33674b897a313d0f874cece67cdd6b83-0)

编译阶段可以得到汇编代码（.s 文件），可由以下命令直接得到汇编代码文件：

```bash
g++ -S test.cpp
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/fd62a06aaf8d1afdd1d047a922c5af20-0)

接下来这条必须铭记，可重定位的目标文件（.o 文件）由以下命令直接得到：

```bash
g++ -c test.cpp
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/0872d3fd416dc6c48ea6c9b4390fc850-0)

最后，当我们需要得到最终的可执行文件时可使用下面这条命令，需要自己指定目标文件名，下面这条命令最终得到名为 work 的可执行文件：

```bash
g++ -o work test.cpp
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/fddd59d3f2885521dadb7f86a96e02ea-0)

在实际开发过程中，我们往往需要使用 gdb 进行调试，这就要求我们在编译时加上-g 选项，如下：

```bash
g++ -g -o work test.cpp
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/3c0f5b762111d32760dcc2ac3813d996-0)

可以看到得到的可执行文件的 Size 更大了，因为包含了调试信息。

链接分为动态链接和静态链接，我们在开发过程中往往需要自己指定使用的动态链接库，比如我们需要使用 pthread 线程库，需要加上 -l 选项。

下例中编译器会到 `lib` 目录下找 `libpthread.so` 文件：

```bash
g++ -o work test.cpp -lpthread
```

其它的选项还有 -O、-Wall 等，这里就不详细介绍了。

#### 多文件编译

在实际项目开发中，多个源代码文件一起编译是很常见的事情，因此需要学会使用 g++ 命令完成多文件编译，比如下例我们有三个文件。

`a.cpp`：

```cpp
#include "b.h"
extern int add(int,int);
int main(){
    int a=1,b=2;
    cout<<add(a,b);
    return 0;
}
```

`b.h`：

```cpp
#ifndef BH
#define BH

#include<iostream>
using namespace std;
int add(int,int);

#endif
```

`b.cpp`：

```cpp
#include "b.h"
int add(int a,int b){
    return a+b;
}
```

我们可以采用以下命令进行编译：

```bash
g++ -o test a.cpp b.cpp
```

使用以下命令运行程序：

```bash
./test
```

#### 运行可执行文件

使用 g++ 编译得到的可执行文件，使用如下格式执行：

```bash
./可执行文件名
```

如果程序需要传入参数，则以下述格式执行：

```bash
./可执行文件名 参数1 参数2
```

### 使用 gdb 进行调试

在实际项目开发中，调试的时间往往比写代码还要更长，因此我们需要掌握 gdb 调试工具的使用方法。

#### 安装 gdb

首先我们需要安装 gdb，在命令行操作步骤如下：

步骤 1：

```bash
wget http://ftp.gnu.org/gnu/gdb/gdb-8.0.1.tar.gz
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/6c955e8869481c576ee19943f9c4a3e9-0)

步骤 2：

```bash
tar -zxvf gdb-8.0.1.tar.gz
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/b53ef25feb0a746d4771efc53ab1ff44-0)

步骤 3：

```bash
cd gdb-8.0.1/
./configure
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/821f5efd8fd0724b3893fa13c5c304d5-0)

步骤 4：

```bash
make
```

步骤 5：

```bash
sudo make install
```

最后查看是否安装成功：

```bash
gdb -v
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/af5d7c0666ebc4a977958fbc82d0ecd2-0)

#### 编译时要加入-g 参数

为了能使用 gdb 调试，我们在用 g++ 编译时要将 -g 参数加上，如：

```bash
g++ -g -o test a.cpp b.cpp
```

#### gdb 常用调试命令

以前面“多文件编译”的程序为例，我们可使用如下命令进入 gdb 调试：

```bash
gdb test
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/61a0658ce3a5d7163080ee3c3e12db8d-0)

常用的调试命令有：

- `l`：查看代码
- `b 5`：在程序的第 5 行添加断点
- `info break`：查看断点
- `r`：开始运行
- `s`：进入函数内部
- `n`：进入下一步
- `finish`：跳出函数内部
- `c`：运行到下一个断点

最后退出调试，可以直接输入 `quit` 命令。

#### gdb 调试实例

以 gdb test 为例，我们先用 `b 5` 在第 5 行设置断点，并按下 `r` 开始运行：

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/0ac0968afa7432151c96c097e50cedd8-0)

可以看到运行到了断点处，这时我们按下 `s` 进入 add 函数的内部，并用 `l` 查看详细函数代码：

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/2e9b77c4a9ca729a5a08b99b3dfac3ec-0)

用 `p a` 命令和 `p b` 命令打印 a 和 b 的值，最后点击 c 运行到下一个断点，由于后面没断点了，所以直接运行到程序末尾：

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/682881b39679a063824e9df67bdf20d6-0)

### 编写 makefile 进行自动编译

在大型项目中有大量的源代码文件，我们不可能每次都逐个敲 g++ 命令来进行编译，而是采用编写 makefile 的方式来进行自动编译，提高效率。

#### 创建 makefile 文件

和创建源代码文件一样，我们可以直接用 vi 编辑器来创建 makefile，如下：

```bash
vi makefile
```

在其中写好 makefile 内容后，命令模式输入 `wq` 保存离开即可。

#### makefile 的基本格式

makefile 的一般格式如下：

```makefile
目标名1：依赖文件1，依赖文件2，依赖文件3
    g++ 编译命令
目标名2：依赖文件4，依赖文件5
    g++ 编译命令
```

其中目标名可以由自己定义，也可以是一个文件的名字；依赖文件就是说要达成这个目标所需要的文件。 仍以前面的“多文件编译”代码为例，我们可以写出如下的 makefile 文件（注意：makefile 文件中要使用 tab 键，不能使用空格键，否则会报错）：

```makefile
target:a.cpp b.o
    g++ -o test a.cpp b.o
b.o:b.cpp
    g++ -c b.cpp
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/6f2ca84cb9e5aad89e8f19acdbce7d7e-0)

可以看出，target 依赖于 `a.cpp` 和 `b.o`，而 `b.o` 依赖于 `b.cpp`，因此编译时发现 `b.cpp` 更新了的话就会先执行后面的命令来更新 `b.o`。

保存好 makefile 文件之后，我们用命令行输入 `make` 即可进行自动编译：

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/f67d65abf052353ec2651b5854292ad3-0)

#### 快速清理目标文件

有时候我们想要删掉 makefile 产生出来的所有目标文件，如果逐个去删显得过于麻烦，因此我们可以借助 make clean。

仍然是在前面的 makefile 文件中修改，在后面补上一个 `clean：`，以及相应的清除命令：

```makefile
target:a.cpp b.o
    g++ -o test a.cpp b.o
b.o:b.cpp
    g++ -c b.cpp
.PHONY:clean
clean:
    rm *.o
    rm test
```

这样当我们在命令行执行 `make clean` 就可删掉所有目标文件。

### 实验总结

通过这次实验，我们熟悉了 Linux 下 C++ 开发常用的工具，对 g++ 编译器、gdb 调试、makefile 有了一定了解，这可以帮助我们后续高效地进行开发。

可以通过如下命令下载本次实验的代码：

```bash
wget https://labfile.oss.aliyuncs.com/courses/3573/code1.zip
```