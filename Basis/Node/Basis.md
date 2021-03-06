# Node的特点

## 异步I/O

在Node中绝大部分操作都是异步操作，比如，`读写文件`就是一个异步I/O

## 模块机制

Node模块分为两类，内部的模块被称为核心模块，用户自定义的模块被称为文件模块。Node中引入模块要经历以下三步：

    1. 路径分析
    2. 文件定位
    3. 编译执行

核心模块在Node编译的过程中被加载进内存，所以这部分核心模块引入时，省去了文件定位和编译执行的步骤，而文件模块是在运行时动态加载的。不管是哪种模块类型，它们都遵循优先从缓存加载的机制

### 核心模块

Node的核心模块在编译成可执行的文件过程中被编译成了二进制文件。核心模块其实分为`C/C++`编写的和`JS`编写的两个部分，其中`C/C++`保存在`src`的目录下，`JS`则保存在`lib`目录下

#### JS核心模块的编译过程

需要将所有的JS模块文件编译成C/C++，具体步骤如下：

1. 转存为C/C++代码

将所有内置的JS代码转换成C++数组，生成`node_natives.h头文件`，在这个过程中，JS代码以`字符串的形式`存储在node命名空间中，是不可直接执行的。当Node在启动的时候，JS代码会直接被加载进内存中。在这个加载的过程中，JS核心模块经历标识符分析后直接定位到内存中，比普通的文件模块在磁盘中一处处查找要快很多

2. 编译JS核心模块

lib目录下的所有模块都没有`require module exports`等变量。在引入JS核心模块的过程中，也经历了包装，然后才执行和导出了exports这些对象。与文件模块有区别的地方在于：获取源代码的方式(核心模块是从内存中加载的)以及缓存执行结果的位置

源文件通过`process.binding('natives')`取出，编译成功的模块缓存到`NativeModule._cache`对象上，文件模块则缓存到`Module._cache`对象上

3. C/C++核心模块编译过程

在核心模块中，有些模块全由C/C++编写(这种模块我们称之为，内建模块，优势在于直接被编译成了二进制文件，从而加载到内存中，不会经历`标识符定位`、`文件定位`、`编译`等过程)，而有些模块是C/C++完成核心部分，其他部分由JS完成，以满足性能需求。通过` NODE_MODULE`定义到node命名空间中，被挂载为`register_func`成员

`node_extensions.h`文件将这些散列的内建模块统一放进了一个叫`node_module_list`数组中，可以通过`get_builtin_module()`方法取出

### 导出内建模块

先看看它们的包含关系：内建模块 => 核心模块 => 文件模块，一般只需要调用核心模块就好，那么内建模块是如何让核心模块调用的呢？答案是Node启动时会创建一个全局变量`process`，并提供一个`Binding`方法来协助加载内建模块

## C++扩展模块

JS语言的一个弱点就是位运算，因为JS的位运算是参考了基于Int类型的Java的位运算，而JS只有double类型，所以每次都需要转成Int类型