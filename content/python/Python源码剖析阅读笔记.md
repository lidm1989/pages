+++
date = "2016-10-08T22:28:53+08:00"
draft = false
title = "Python源码剖析阅读笔记"
image = "python.jpg"

categories = ["python"]

+++

# 00: 准备工作
## Python源码目录
* Include：该目录包含了Python提供的所有头文件，如果用户需要自己用C或C++来编写自定义模块扩展Python，那么就需要乃至这里提供的头文件。
* Lib：该目录包含了Python自带的所有标准库，Lib中的库都是用Python语言编写的。
* Modules: 该目录中包含了所有用C语言编写的模块，比如random, cStringIO等。Modules中的模块是那些对速度要求非常严格的模块，而有一些雄起速度没有太严格要求的模块，比如os，就是用Python编写，并且放在Lib目录下的。
* Parser: 该目录中包含了Python解释器中的Scanner和Parser部分，即对Python源代码进行词法分析和语法分析的部分。除了之些，Parser目录下还包含了一些有用的工具，这些工具能够根据Python语言的语法自动生成Python语言的词法和语法分析器，与YACC非常类似。
* Objects：该目录中包含了所有Python的内建对象，包括整数、list、dict等。同时，该目录还包括了Python在运行时需要的所有的内部使用对象的实现。
* Python：该目录下包含了Python解释器中的Compiler和执行引擎部分，是Python运行的核心所在。
## 编译源码
* ./configure
* make
* make install

# 01: Python对象初探
## PyObject_HEAD
    #define PyObject_HEAD                   \
        Py_ssize_t ob_refcnt;               \
        struct _typeobject *ob_type;

## PyObject
    typedef struct _object {      
        PyObject_HEAD
    } PyObject;

## PyObject_VAR_HEAD
    #define PyObject_VAR_HEAD               \
        PyObject_HEAD                       \
        Py_ssize_t ob_size; /* Number of items in variable part */

## PyVarObject
    typedef struct {              
        PyObject_VAR_HEAD
    } PyVarObject;

## 
