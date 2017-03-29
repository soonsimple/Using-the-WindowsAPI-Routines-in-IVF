# 在Intel Visual Fortran（IVF）程序中调用Windows API例程


本文译自Intel Fortran编译器的参考文档。

Intel Visual Fortran提供了绝大部分的Windows API的接口定义模块，这使得在Fortran程序中使用Windows API系统调用变得很容易。

## 第一步，在Fortran程序中包含IVF提供的Windows API接口定义

IVF提供的接口定义文件包含了Fortran标准例程库和大多数的Windows API例程，存放在标准include目录里。关于Windows API的文档可以在msdn.microsoft.com上找到。

可以按照下面两种方法中的一种，将Windows API接口定义包含到你的Fortran程序中：

1、使用语句USE IFWIN，这将会包含全部的Windows API例程定义。
USE IFWIN语句使得大多数的Windows例程的所有参数和接口对你的Fortran程序都可用。任何需要使用Windows功能的程序或子程序都可以包含这条USE IFWIN语句，而且出现了Windows API调用的每个（子）程序都需要添加这条语句。应将USE IFWIN语句放在所有其他声明语句（比如IMPLICIT NONE 或 INTEGER）之前。

2、你可以根据需要，限定包含进来的参数和接口的类型，这样可以缩短编译时间。方法是使用几条USE语句，只包含Windows API的子集（参见文件...\include\IFWIN.F90）。

想定位你所调用的Windows API例程是由哪个库文件提供的，可以查看msdn.microsoft.com上Windows API文档中相应的例程所在页面，那里列出了导入库的文件名。例如，如果只调用GetSystemTime的话，你只需要kernel32.mod（二进制格式）提供的接口定义，使用下面的USE语句就可以了：

    USE KERNEL32
如果想进一步最小化编译时间，可以给USE语句添加ONLY关键字。例如：

    USE KERNEL32, only: GetSystemTime, GetLastError

## 第二步，调用Windows API例程

使用USE语句在你的Fortran程序里包含了IVF提供的接口定义后，就可以调用Windows API例程了。请遵循以下详细指南：

1、仔细查阅你要调用的Windows API例程（如GetSystemTime）的文档，获取下列信息：

- 参数的个数和顺序。你需要使用正确的数据类型声明和初始化每一个参数。比如GetSystemTime例程需要引用传递一个结构类型（派生类型）的参数。

    如果出现字符型参数，在调用Windows API例程之前，需要给字符型变量的值后面添加一个空字符作为结尾。

- API例程是否有返回值（决定了该例程是函数还是子例程）。例如，由于GetSystemTime例程的调用格式是以VOID开头的，因此应把它作为子例程调用，使用CALL语句。


2、如果不能确定参数的数据类型或函数返回值的数据类型，可以查看...\include\目录下相应.F90文件里的接口定义。以GetSystemTime为例，查看接口定义的方法如下：

- 用文本编辑器（如notepad）从...\include\目录打开KERNEL32.F90文件；
- 找到例程名GetSystemTime；
- 查看接口定义，并将它与Windows API文档里的描述作比较，以确定参数在Fortran中如何表示。应着重注意参数是如何传递的；有的情况下比如某个参数是按地址传递的，这时你必须调用LOC内置函数，传递变量的地址，而不是直接传递变量。


3、如果其中有个参数是结构，还要查阅...\include\目录里的IFWINTY.F90文件。例如，要查看GetSystemTime用到的T_SYSTEMTIME类型的定义：

- 用文本编辑器（如notepad）从...\include\目录打开IFWINTY.F90文件；
- 找到数据类型的名字（这里为T_SYSTEMTIME）；
- 查看数据类型定义，应注意结构各字段的的字段名，有时会和Windows API文档里列出的略有出入；
- 在你的Fortran程序中使用派生类型定义一个变量，像这样：


        TYPE(T_SYSTEMTIME) mytime

4、许多Windows API例程都有一个描述为“handle（句柄）”的参数，或是返回一个描述为“handle（句柄）”的值。它通常是一个地址大小的整型数，必须使用相应的KIND值来声明它。典型地，在32位或64位平台上，KIND值取为HANDLE，系统会自动提供正确的值。例如：

        INTEGER(HANDLE) :: hwnd

需要注意的就这么多。

上述调用Win32API例程GetSystemTime的完整示例代码如下：

    ! Getsystemtime.f90程序演示如何调用Windows API例程。
    ! 由于调用的只有一个GetSystemTime例程，因此只需从kernel32.mod模块
    ! 将接口定义包含进来，而不是从ifwin.f90包含所有的模块。
    ! 类型定义在IFWINTY模块里，KERNEL32已包含了该模块。
    PROGRAM Getsystemtime
    USE KERNEL32
    TYPE(T_SYSTEMTIME) mytime
    CALL GetSystemTime(mytime)
    WRITE (*,*) "Current UTC time : ", mytime.wHour, mytime.wMinute
    END PROGRAM
你可以新建一个Fortran控制台应用程序项目，将上面的代码添加进去，然后编译、查看结果。

## 进一步了解数据类型的区别

IFWINTY模块（IFWIN模块和其他Win32 API模块都包含了该模块）定义了一整套INTEGER类型和REAL类型的KIND常量，这些常量与Windows的WINDOWS.H头文件提供的类型定义相对应。应在INTEGER类型声明和REAL类型声明中使用这些KIND值。下表给出了部分常用Windows类型的对应关系：

Windows数据类型     | 与之等同的Fortran数据类型
-------------------------------------------------
BOOL, BOOLEAN      | INTEGER(BOOL)
BYTE               | INTEGER(BYTE)
CHAR, CCHAR, UCHAR | CHARACTER or INTEGER(UCHAR)
DWORD              | INTEGER(DWORD)
ULONG              | INTEGER(ULONG)
SHORT              | INTEGER(SHORT)
LPHANDLE           | INTEGER(LPHANDLE)
PLONG              | INTEGER(PLONG)
DOUBLE             | REAL(DOUBLE)


应使用这些定义好的KIND常量，不要显式地指定KIND数字或设定默认KIND值。

注意Windows的BOOL类型不等同于Fortran的LOGICAL类型，因而不能使用Fortran的LOGICAL运算符和文字常量。应使用IFWINTY模块中定义的常量TRUE和FALSE，而不是Fortran的文字量.TRUE.和.FALSE.，也不能用LOGICAL表达式测试BOOL值。

关于参数的对等数据类型的其他注意事项：

- 如果参数在Windows API文档中描述为一个指针，那么，该参数相应的Fortran接口定义必须具有REFERENCE属性。老的接口定义使用POINTER - Integer构建的对象指针对儿来传递参数的地址，或者使用LOC内置函数。
- 指针参数在IA-32架构的系统上长度为32位（4个字节），在Intel 64架构的系统上长度为64位（8个字节）.
- 应注意，需要将Fortran字符变量变换成以空字符结尾。可以使用C字符串扩展来完成：

        forstring = "This is a null-terminated string."C

    还可以使用内置模块ISO_C_BINDING中定义的C_NULL_CHAR或CHAR(0)来连接一个空字符结尾：

        use, intrinsic :: ISO_C_BINDING
        …
        forstring = 'This is a null-terminated string'//C_NULL_CHAR
        forstring2 = 'This is another null-terminated string'//CHAR(0)

在IFWINTY模块中，已经将windows.h中的结构转换成相应的派生类型，结构中的联合Union已经转换成派生类型内的union或map。

组件命名基本没变。C位域不能直接转换到Fortran；位域集声明为Fortran的INTEGER类型，单个位域在源码文件IFWINTY.F90里标记为注释了。想查看如何将某个Windows声明转换到Fortran，请阅读Include文件夹里相应的.F90文件中的对应声明。


## IVF提供的Windows API模块

Intel Visual Fortran提供了下列Windows API模块，与Windows同名导入库相对应：

    ADVAPI32
    COMCTL32
    COMDLG32
    GDI32
    KERNEL32
    LZ32
    OLE32
    OLEAUT32
    PSAPI
    SCRNSAVE
    SHELL32
    USER32
    VERSION
    WINMM
    WINSPOOL
    WS2_32
    WSOCK32

另外，IFOPNGL模块声明了用于Windows系统的OPENGL例程，组件命名为Fortran特有。

### 另附IFWIN.F90文件的主要内容：

    MODULE IFWIN
        use advapi32
        use comdlg32
        use ifwbase
        use gdi32
        use kernel32
        use lz32
        use mpr
        use shell32
        use user32
        use version
        use winmm
        use winspool
        use wsock32
        use comctl32
        use ole32
        use oleaut32
        use psapi
    END MODULE IFWIN
