+++
title = "import"
draft = false
date = "2016-11-25T17:11:17+08:00"

thumbnail = "images/python/python.jpg"
categories = ["python"]

+++

# module
模块是一个文件。文件名=模块名+文件后缀，后缀有`.py`、`.pyc`、`.pyo`、`.so`等。模块中可以使用`__name__`获取模块名。

# 模块搜索路径
当模块被导入时，解释器首先搜索`built-in`模块，如果未找到，再搜索`sys.path`列表。

# 示例1
    # file: demo.py

    import sys

    foo = 1
    _bar = 1

### 通过import导入
    >>> import demo
    >>> dir()
    ['__builtins__', '__doc__', '__name__', '__package__', 'demo']
    >>> dir(demo)
    ['__builtins__', '__doc__', '__file__', '__name__', '__package__', '_bar', 'foo', 'sys']
    >>> 

### 通过from...import...导入
    >>> from demo import foo
    >>> from demo import _bar
    >>> dir()
    ['__builtins__', '__doc__', '__name__', '__package__', '_bar', 'foo']
    >>> 

### 通过from...import * 导入
    >>> from demo import *
    >>> dir()
    ['__builtins__', '__doc__', '__name__', '__package__', 'foo', 'sys']
    >>> 

# package
包是一个目录。包目录下为首的一个文件便是`__init__.py`，然后是一些模块文件和子目录，假如子目录中也有`__init__.py`那么它就是这个包的子包了。

# 示例2
    sound/                          Top-level package
          __init__.py               Initialize the sound package
          formats/                  Subpackage for file format conversions
                  __init__.py
                  wavread.py
                  wavwrite.py
                  aiffread.py
                  aiffwrite.py
                  auread.py
                  auwrite.py
                  ...
          effects/                  Subpackage for sound effects
                  __init__.py
                  echo.py
                  surround.py
                  reverse.py
                  ...
          filters/                  Subpackage for filters
                  __init__.py
                  equalizer.py
                  vocoder.py
                  karaoke.py
                  ...

### 通过import导入
    import sound.effects.echo
    sound.effects.echo.echofilter(input, output, delay=0.7, atten=4)

### 通过from...import...导入
    from sound.effects import echo
    echo.echofilter(input, output, delay=0.7, atten=4)

or

    from sound.effects.echo import echofilter
    echofilter(input, output, delay=0.7, atten=4)

### 通过from...import * 导入
包作者可以通过在包的`__init__.py`里定义`__all__`变量来显式指定可以导出的名字

    # file: __init__.py

    __all__ = ["echo", "surround", "reverse"]

如果没有定义`__all__`，将会导入所有子模块和`__init__.py`中定义的名字

# 包内相互导入
## subpackages内
可以直接导入，也就是说import语句先搜索当前包，再搜索`sys.path`列表。如surround模块可以如下导入：

    import echo

or

    from echo import echofilter

## subpackages间
如sound.filter.vocoder导入sound.effects.echo

### 绝对导入

    from sound.effects import echo

### 相对导入

    from ..effects import echo


# 参考
[Python2官网](https://docs.python.org/2/tutorial/modules.html)
