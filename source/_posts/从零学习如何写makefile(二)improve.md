---
title: 从零学习写makefile(二)基础学习
date: 2025-10-04 21:38:07
tags: [学习记录]
index_img: /img/avatar.png

---

# 从零学习写makefile(二)improve

## 一.Makefile基础学习

#### 基本语法：

```shell
目标：依赖
[TAB]命令
```

目标：一般是指要编译的目标，也可以是一个动作

依赖：指执行当前目标所要依赖的先项，包括其它目标，某个具体文件或库，一个目标可以有多个依赖

命令：该目标下要执行的具体命令，可以没有，也可以有多条（多条时每个命令一行）

## 二.MAKE环境搭建

linux系统下装make环境很简单，这里主要介绍一下windows系统下的make环境搭建。

如果没有mingw installation manager的话先装，然后打开mingw installation manager，在Basic Setup下找到mingw32-base，mingw32-gcc-g++，msys-base并右键mark for install，然后apply changes即可。（这里我踩了一个坑，就是先装了msys下的make然后发现版本太低了很多不兼容）

安装完后需要将路径加到系统环境变量中，先找到新装的make放置的路径并复制（这是我的路径C:\MinGW\bin），然后复制mingw32-make.exe并在这个路径粘贴，将新的粘贴文件重命名为make.exe（这样命令行中就能直接输入make而不是mingw32-make）

打开环境变量窗口，在用户变量中找到Path并双击，然后新建并把刚刚复制的路径粘贴，然后所有的窗口都要点击确认，此时make环境就搭建好了。

如果要在vscode中写makefile，需要装makefile tool拓展

## 三.Makefile进阶学习

### 基础命令

首先介绍一下最重要的，命令行输入make -h可以打印帮助列表

然后介绍一下常用的几个命令

-f 指定除上述文件名之外的文件作为输入文件

-n 输出命令，但不执行

-s 执行命令，但不显示具体命令，也可以加@符号抑制命令输出

-w 显示执行前执行后的路径

-C dir 指定makefile所在的目录

```
gcc add.cpp sub.cpp multi.cpp calc.cpp -o calc
```

上面这种写法可以实现编译并输出一个calc.o的目标文件，但是这种写法显然是有弊端的，弊端在于当只修改了其中一个文件时仍需对所有文件编译，然后就会浪费编译时间，这对于开发者开发大型项目时是不可接受的，所以一般会用目标：依赖    [tab]命令这种方式分开写每个文件，每个.cpp文件都有自己的.o文件

```
#这样分开来写，保证只编译有改动的代码，耗时短
calc:add.o sub.o multi.o
	gcc add.o sub.o multi.o calc.cpp -o calc
	
add.o:add.cpp
	gcc -c add.cpp -o add.o

sub.o:sub.cpp
	gcc -c sub.cpp -o sub.o
	
multi.o:multi.cpp
	gcc -c multi.cpp -o multi.o	
```

gcc命令可以拆解为几个“子”命令

```
gcc -lstdc++ main.cpp

#把过程拆分
#预处理 gcc -E main.cpp >main.ii
#编译 gcc -S main.ii		 得到名为main.s的汇编文件
#汇编 gcc -c main.s		 得到名为main.o(.obj)的二进制文件
#链接 gcc -lstdc++ main.o  得到名为a.out的可执行文件
```

### makefile中的变量

#### 自定义变量

定义：变量名=变量值

使用：$(变量名)或${变量名}

```
OBJ = add.o sub.o multi.o calc.o
TARGET = calc

$(TARGET):$(OBJ)
	gcc $(OBJ) -o $(TARGET)
	
add.o:add.cpp
	gcc -c add.cpp -o add.o

sub.o:sub.cpp
	gcc -c sub.cpp -o sub.o
	
multi.o:multi.cpp
	gcc -c multi.cpp -o multi.o
	
calc.o:calc.cpp
	gcc -c calc.cpp -o calc.o
	
clean:
	rm -f *.o $(TARGET)

```

#### 系统变量

```
$* #不包括扩展名的目标文件名称
$+ #所有的依赖文件，以空格分隔
$< #表示规则中的第一个条件
$? #所有时间戳比目标文件晚的依赖文件，以空格分隔
$@ #目标文件的完整名称
$^ #所有不重复的依赖文件，以空格分隔
$% #如果目标是归档成员，则该变量表示目标的归档成员名称
```

```
calc:add.o sub.o multi.o calc.o
	gcc $^ -o $@
	
add.o:add.cpp
	gcc -c $^ -o $@

sub.o:sub.cpp
	gcc -c $^ -o $@
	
multi.o:multi.cpp
	gcc -c $^ -o $@
	
calc.o:calc.cpp
	gcc -c $^ -o $@
	
clean:
	rm -f *.o calc

```

#### 系统常量

```
AS	#汇编程序的名称，默认为as
CC	#C编译器名称，默认为cc
CPP	#C预编译器名称，默认为cc -E
CXX	#C++编译器名称，默认为g++
RM	#文件删除程序别名，默认rm -f
```

```
#$(CC)可替换为$(CXX)，因为$(CXX)可实现跨平台
calc:add.o sub.o multi.o calc.o
	$(CC) add.o sub.o multi.o calc.o -o calc
	
add.o:add.cpp
	$(CC) -c add.cpp -o add.o

sub.o:sub.cpp
	$(CC) -c sub.cpp -o sub.o
	
multi.o:multi.cpp
	$(CC) -c multi.cpp -o multi.o
	
calc.o:calc.cpp
	$(CC) -c calc.cpp -o calc.o
	
clean:
	$(RM) *.o calc

```

### 伪目标与模式匹配

#### 伪目标

.PHONY:目标	
声明目标为伪目标之后，makefile将不会判断目标是否存在或该目标是否需要更新

- 如果项目路径中有与clean同名的文件，`make clean`命令无法执行，需要在makefile文件中添加`.PHONY:clean`，才能使命令正常执行

```
.PHONY:clean
OBJ = add.o sub.o multi.o calc.o
TARGET = calc

$(TARGET):$(OBJ)
	gcc $(OBJ) -o $(TARGET)
	
add.o:add.cpp
	gcc -c add.cpp -o add.o

sub.o:sub.cpp
	gcc -c sub.cpp -o sub.o
	
multi.o:multi.cpp
	gcc -c multi.cpp -o multi.o
	
calc.o:calc.cpp
	gcc -c calc.cpp -o calc.o
	
clean:
	rm -f *.o $(TARGET)
```

#### 模式匹配

```makefile
%目标:%依赖	#目标和依赖相同部分，可以用%来通配
```

- 依赖中的`add.o sub.o multi.o calc.o`都可以通过使用`%.o:%.cpp`产生

```
OBJ = add.o sub.o multi.o calc.o
TARGET = calc

$(TARGET):$(OBJ)
	gcc $(OBJ) -o $(TARGET)
	
%.o:%.cpp
	gcc -c $^ -o $@
	
clean:
	rm -f *.o $(TARGET)
```

#### 内置函数

```
$(wildcard 文件列表)	#获取对应文件路径下的对应模式的文件名
$(patsubst 源模式, 目标模式, 文件列表)	#将文件列表中想要改变的源模式替换成想要的目标模式
```

- **`$(wildcard ./\*.cpp)`\获取当前目录下所有的\.cpp**文件名
- **`$(patsubst %.cpp, %.o, $(wildcard ./\*.cpp))`\**将对应的\**.cpp**文件替换为**.o**文件

```
#OBJ = add.o sub.o multi.o calc.o
OBJ = $(patsubst %.cpp, %.o, $(wildcard ./*.cpp))
TARGET = calc

$(TARGET):$(OBJ)
	gcc $(OBJ) -o $(TARGET)
	
%.o:%.cpp
	gcc -c $^ -o $@
	
clean:
	rm -f *.o $(TARGET)
```

#### makefile中编译动态链接库

**动态链接库**：不会把代码编译到二进制文件中，而是在**运行时**才去加载，所以需要维护一个地址

- **动态**：动态加载，运行时才加载
- **链接**：指库文件和二进制程序分离，用某种手段维护两者之间的关系
- **库**：库文件
  - **Windows** 中后缀为 **.dll**
  - **Linux** 中后缀为 **.so**

**优点：**程序可以和库文件分离，可以分别发版，然后库文件可以被多处共享

这里举个例子来看看正常是怎么得到动态链接库的

**soTest.h**

```
class soTest{
public:
    void func1();
    virtual void func2();
    virtual void func3() = 0;
};

```

**soTest.cpp**

```
#include <iostream>
#include "soTest.h"

void sotest::func1(){
    printf("func1\n");
}

void sotest::func2(){
    printf("func2\n");
}

```

```
g++ -shared -fPIC soTest.cpp -o libSoTest.so
```

- **soTest.cpp**编译后会产生**libsoTest.so**，不用担心文件名不同会出现问题，在后续引用中，**lib**自动会被丢弃

**项目发布：**

**soTest.h**

```
class soTest{
public:
    void func1();
    virtual void func2();
    virtual void func3() = 0;
};
```

**main.cpp**

```
#include <iostream>
#include "soTest.h"

class MainTest:public soTest{
public:
    void func2(){
        printf("MainTest-func2\n");
    }
    void func3(){
        printf("MainTest-func3\n");
    }
};

int main(){
    MainTest t1;
    t1.func1();
    t1.func2();
    t1.func3();  
    return 0;
}
```

```
g++ -lsoTest -L./001 main.cpp -o main
```

编译时指定了要依赖的动态库，但运行时，会无法找到**.so**文件
**解决方法：**

- 将动态库文件移动到main.cpp文件同级目录下
- 运行时手动指定动态库文件所在目录


**Linux环境下的命令**

```
LD_LIBRARY_PATH = ./001
```

```
export LD_LIBRARY_PATH
```

**001/Makefile**

```
Test:libsoTest.so
	$(CXX) -lsoTest -L./ Test.cpp -o Test
	cp libsoTest.so /usr/lib  #路径最好为main.cpp同级目录
libsoTest.so:
	$(CXX) -shared -fPIC soTest.cpp -o libsoTest.so
	
clean:
	$(RM) *.so Test
```


cp libsoTest.so /usr/lib将动态库文件复制到动态链接库默认路径下 不推荐复制，污染原库环境

**Linux 默认动态库路径配置文件**

- /etc/ld.so.conf

- /etc/ld.so.conf.d/*.conf



















