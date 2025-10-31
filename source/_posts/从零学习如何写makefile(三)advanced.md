---
title: 从零学习写makefile(三)advanced
date: 2025-10-15 23:12:17
tags: [学习记录]
index_img: /img/avatar.png

---

# 从零学习写makefile(三)advanced

## makefile编译静态链接库

静态链接库：会把库中的代码编译到二进制文件中，当程序编译完成后，该库文件可以删除

- Windows 中后缀为 .lib

- Linux 中后缀为 .a

**优点：**运行时速度快（不用去加载库文件）

**缺点：**程序体积更大，并且库中的内容如果有更新，则需要重新编译生成程序

对比：

- 动态链接库必须与程序同时部署，还要保证程序能加载得到库文件
- 静态链接库可以不用部署（已经被加载到程序里面了）

例：
**发布前：**

```
//aTest.h
class aTest{
public:
    void func1();
}
```

```
aTest.cpp
#include <iostream>
#include "aTest.h"

void aTest::func1(){
    printf("aTest-func1\n");
}
```

```
g++ -c aTest.cpp -o aTest.o
```

aTest.cpp编译后会产生libaTest.a，不用担心文件名不同会出现问题，在后续引用中，lib自动会被丢弃

```
ar -r libaTest.a aTest.o
```

将aTest.o加入到libaTest.a中。默认的加入方式为append，即加在库的末尾。

**项目发布：**

```
//aTest.h
class aTest{
public:
    void func1();
}
```

```
//main.cpp
#include <iostream>
#include "soTest.h"
#include "aTest.h"

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
    printf("\n\n");
    aTest t2;
    t2.func1();
    return 0;
}
```

```
g++ -lsoTest -L./001 -laTest -L./002 main.cpp -o main
```

指定动态库文件soTest和静态库文件aTest，编译后由main.cpp文件产生main.o文件

```
objdump -DC main>main.txt
```

会生成一个类似于反汇编的文本文档，进入搜索可以找到aTest::func1() 的定义，但只能找到soTest::func1()、soTest::func2()、soTest::func3() 的引用

**002/Makefile**

```
libaTest:
	$(CXX) -c aTest.cpp -o aTest.o
	$(AR) -r libaTest.a aTest.o
	
clean:
	$(RM) *.o *.a
```

**Makefile**

```
MakefileTARGET = main
LDFLAGS = -L./001 -L./002
LIBS = -lsoTest -laTest

$(TARGET):
	$(CXX) $(LIBS) $(LDFLAGS) main.cpp -o $(TARGET)
	
clean:
	$(RM) $(TARGET)

```

