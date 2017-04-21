% Python 类库引入机制

Python

---

#::Copyright

+ 作者 = xiaowang (xer345@126.com)
+ 日期 = 2017-04-15
+ 时间 = 2017-04-15 23:33:16 / 2017-04-20 16:58:40
+ 链接 = [Python 类库引入机制](http://onlookee.com/?c=Article&a=view&id=10)
+ 版本 = 1.0
+ 关于 = [ONLOOKEE.COM](http://onlookee.com/) 一直在分享有价值的软件/应用 、Python经验。

#::文章信息

+ 类别 = Python
+ 平台 = Windows,Mac,Linux
+ 官网 = [Welcome to Python.org](https://www.python.org/ "")

#::截图照片

+ ![主界面](images/PythonImport.png)

#::文章内容

-   [python-类库引入机制](#python-类库引入机制)
-   [python 的两种引入机制](#python-的两种引入机制)
    -   [relativeimport](#relativeimport)
    -   [absolute-import](#absolute-import)
-   [一些实践经验](#一些实践经验)
    -   [相对引用还是绝对引用](#相对引用还是绝对引用)
    -   [规范打包发布](#规范打包发布)
    -   [使用 virtualenv 管理包依赖](#使用-virtualenv-管理包依赖)
-   [python import 实现](#python-import-实现)
    -   [查找 module 的过程](#查找-module-的过程)
-   [python import hooks](#python-import-hooks)
    -   [import hook 的重要性](#import-hook-的重要性)
    -   [如何实现 import hooks](#如何实现-import-hooks)
    -   [一些 hook 示例](#一些-hook-示例)

Python
是一门优美简单、功能强大的动态语言。在刚刚接触这门语言时，我们会被其优美的格式、简洁的语法和无穷无尽的类库所震撼。在真正的将
python
应用到实际的项目中，你会遇到一些无法避免的问题。最让人困惑不解的问题有二类，一个
编码问题，另一个则是引用问题。

本文主要讨论关于 Python 中 import 的机制与实现、以及介绍一些有意思的
Python Hooks。

python-类库引入机制
-------------------

首先，看一个简单的例子：

``` {.sourceCode .python}
    """
    目录结构如下：
    ├── __init__.py
    ├── main.py
    └── string.py
    """
    # main.py 内容如下
    import string
    print string.a
    # string.py 内容如下
    a = 2
```

现在，考虑一下：

1.  当我们执行 main.py 的时候，会发生什么事情？
2.  在 main.py 文件执行到`import string`的时候，解释器导入的 string
    类库是当前文件夹下的 string.py 还是系统标准库的 string.py 呢？
3.  如果明确的指明⾃己要引⼊的类库？

为了搞清楚上面的问题，我们需要了解关于 Python 类库引入的机制。

python 的两种引入机制
---------------------

Python 提供了二种引入机制：

1.  relative import
2.  absolute import

### relativeimport

也叫作相对引入，在 Python2.5 及之前是默认的引入方法。它的使用方法如下：

``` {.sourceCode .python}
from .string import a
from ..string import a
from ...string import a
```

这种引入方式使用一个点号来标识引入类库的精确位置。与 linux
的相对路径表示相似，一个点表示当前目录，每多一个点号则代表向上一层目录。

``` {.sourceCode .python}
"""
├── __init__.py
├── foo.py
└── main.py
"""
# foo.py
a = 2
# main.py
print __name__
from .foo import a
print a
```

相对引入，那么我们需要知道相对什么来引入。相对引入使用被引入文件的`__name__`属性来决定该文件在整个包结构的位置。那么如果文件的`__name__`没有包含任何包的信息，例如`__name__`被设置为了`__main__`，则认为其为‘top
level
script’，而不管该文件的位置，这个时候相对引入就没有引入的参考物。如上面的程序所示，当我们执行`python main.py`时，Python
解释器会抛出 ValueError: Attempted relative import in non-package
的异常。

为了解决这个问题，[PEP 0366 – Main module explicit relative
imports](https://www.python.org/dev/peps/pep-0366/)提出了一个解决方案。允许用户使用`python -m ex2.main`的方式,来执行该文件。在这个方案下，引入了一个新的属性`__package__`。

    ╭─liuchang@localhost  ~/Codes/pycon
    ╰─$ cat ex2/main.py
    print __name__
    print __package__
    from .foo import a
    print a
    ╭─liuchang@localhost  ~/Codes/pycon
    ╰─$ python -m ex2.main
    __main__
    ex2
    2

### absolute-import

absolute import 也叫作完全引入，非常类似于 Java 的引入进制，在 Python2.5
被完全实现，但是是需要通过`from __future__ import absolute_import`来打开该引入进制。在
Python2.6 之后以及 Python3，完全引用成为 Python
的默认的引入机制。它的使用方法如下：

``` {.sourceCode .python}
from pkg import foo
from pkg.moduleA import foo
```

要注意的是，需要从包目录最顶层目录依次写下，而不能从中间开始。

在使用该引入方式时，我们碰到比较多的问题就是因为位置原因，Python
找不到相应的库文件，抛出 ImportError 的异常。让我们看一个完全引用的例子:

``` {.sourceCode .python}
"""
ex3
├── __init__.py
├── foo.py
└── main.py
"""
# foo.py
a = 2
```

``` {.sourceCode .python}
# main.py
print __name__
print __package__
from ex2.foo import a
print a
```

我们尝试着去运行 main.py 文件，Python 解释器会抛出
ImportError。那么我们如何解决这个问题呢？

    ╰─$ python ex3/main.py
    __main__
    None
    Traceback (most recent call last):
    File "ex3/main.py", line 3, in <module>
      from ex2.foo import a
    ImportError: No module named ex2.foo

首先，我们也可以使用前文所述的 module 的方式去运行程序，通过-m
参数来告诉解释器`__package__`属性。如下：

    ╭─liuchang@liuchangdeMacBook-Pro  ~/Codes/pycon
    ╰─$ python -m ex3.main                                                                             
    __main__
    ex3
    2

另外，我们还有一个办法可以解决该问题，在描述之前，我们介绍一个关于
Python 的非常有用的小知识：**Python 解释器会自动将当前工作目录添加到
sys.path**。如下所示，可以看到我们打印出的`sys.path`已经包含了当前工作目录。

    ╭─liuchang@liuchangdeMacBook-Pro  ~/Codes/pycon/ex4
    ╰─$ cat main.py
    import sys
    print sys.path
    ╭─liuchang@liuchangdeMacBook-Pro  ~/Codes/pycon/ex4
    ╰─$ python main.py
    ['/Users/liuchang/Codes/pycon/ex4', '/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg', '/Library/Python/2.7/site-packages/mesos-_PACKAGE_VERSION_-py2.7.egg', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python27.zip', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-darwin', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-mac', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-mac/lib-scriptpackages', '/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-tk', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-old', '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload', '/Users/liuchang/Library/Python/2.7/lib/python/site-packages', '/usr/local/lib/python2.7/site-packages', '/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/PyObjC', '/Library/Python/2.7/site-packages']

了解了 Python
解释器的这个特性后，我们就可以解决完全引用的找不到类库的问题：执行的时候，让解释器自动的将类库的目录添加到
PYTHONPATH 中。

我们可以在顶层目录中添加一个 run\_ex3.py
的文件，文件内容和运行结果如下，可以看到 Python 解释器正确的执行了
ex3.main 文件。

    ╭─liuchang@liuchangdeMacBook-Pro  ~/Codes/pycon
    ╰─$ cat run_ex3.py
    from ex3 import main
    ╭─liuchang@liuchangdeMacBook-Pro  ~/Codes/pycon
    ╰─$ python run_ex3.py
    ex3.main
    None
    2

一些实践经验
------------

### 相对引用还是绝对引用

相对引用还是绝对引用？

上面介绍了 Python
的两种引用方式，都可以解决引入歧义的问题。那我们应该使用哪一种呢？

先说明一下 Python 的默认引用方式，在 Python2.4 及之前，Python
只有相对引用这一种方式，在 Python2.5
中实现了绝对引用，但默认没有打开，需要用户自己指定使用该引用方式。在之后的版本和
Python3 版本，绝对引用已经成为默认的引用方式。

其次，二种引用方式各有利弊。绝对引用代码更加清晰明了，可以清楚的看到引入的包名和层次，但是，当包名修改的时候，我们需要手动修改所有的引用代码。相对引用则比较精简，不会被包名修改所影响，但是可读性较差，不如完全引用清晰。

最后，对于两种引用的方式选择，还是有争论的。在 PEP8 中，Python
官方推荐的是绝对引用,详细理由可以参考[这儿](https://www.python.org/dev/peps/pep-0008/#imports)。

> Absolute imports are recommended, as they are usually more readable
> and tend to be better behaved (or at least give better error messages)
> if the import system is incorrectly configured (such as when a
> directory inside a package ends up on sys.path ):

``` {.sourceCode .python}
import mypkg.sibling
from mypkg import sibling
from mypkg.sibling import example
```

> However, explicit relative imports are an acceptable alternative to
> absolute imports, especially when dealing with complex package layouts
> where using absolute imports would be unnecessarily verbose:

``` {.sourceCode .python}
from . import sibling
from .sibling import example
```

> Standard library code should avoid complex package layouts and always
> use absolute imports. Implicit relative imports should never be used
> and have been removed in Python 3.

### 规范打包发布

为了别人使用自己代码的方便，应该尽量使用规范的包分发机制。为自己的
Python 包编写正确的 setup.py 文件，添加相应的 README.md
文件。对于提供一些可执行命令的包，则可以使用 console\_entrypoint
的机制来提供。因为打包和分发不是本文重点，不再详细叙述，大家可以查看官方文档。

### 使用 virtualenv 管理包依赖

使用 virtualenv 管理包依赖

在使用 Python 的时候，尽量使用 virtualenv
来管理项目，所有的项目从编写到运行都在特定的 virtualenv
中。并且为自己的项目生成正确的依赖描述文件。

    pip freeze > requirements.txt

关于 virtualenv 的用法，可以参考我之前的一篇文章[virtualenv
教程](http://lcblog-wordpress.stor.sinaapp.com/uploads/2015/10/virtualenv%E6%95%99%E7%A8%8B.pdf)。

python import 实现
------------------

Python 提供了 import 语句来实现类库的引用，下面我们详细介绍当执行了
import 语句的时候，内部究竟做了些什么事情。

当我们执行一行 `from package import module as mymodule`命令时，Python
解释器会查找 package 这个包的 module 模块，并将该模块作为 mymodule
引入到当前的工作空间。所以 import 语句主要是做了二件事：

1.  查找相应的 module
2.  加载 module 到 local namespace

下面我们详细了解 python 是如何查找模块的。

### 查找 module 的过程

在 import
的第一个阶段，主要是完成了查找要引入模块的功能，这个查找的过程如下：

1.  检查 sys.modules (保存了之前 import 的类库的缓存），如果 module
    被找到，则⾛到第二步。
2.  检查 sys.meta\_path。meta\_path 是一个 list，⾥面保存着一些 finder
    对象，如果找到该 module 的话，就会返回一个 finder 对象。
3.  检查⼀些隐式的 finder 对象，不同的 python 实现有不同的隐式
    finder，但是都会有 sys.path\_hooks, sys.path\_importer\_cache 以及
    sys.path。
4.  抛出 ImportError。

#### sysmodules

对于第一步中 sys.modules，我们可以打开 Python 来实际的查看一下其内容：

    Python 2.7.10 (default, Aug 22 2015, 20:33:39)
    [GCC 4.2.1 Compatible Apple LLVM 7.0.0 (clang-700.0.59.1)] on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import sys
    >>> sys.modules
    {'copy_reg': <module 'copy_reg' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/copy_reg.pyc'>, 'sre_compile': <module 'sre_compile' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/sre_compile.pyc'>, '_sre': <module '_sre' (built-in)>, 'encodings': <module 'encodings' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/encodings/__init__.pyc'>, 'site': <module 'site' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/site.pyc'>, '__builtin__': <module '__builtin__' (built-in)>, 'sysconfig': <module 'sysconfig' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/sysconfig.pyc'>, 'encodings.encodings': None, '__main__': <module '__main__' (built-in)>, 'supervisor': <module 'supervisor' (built-in)>, 'abc': <module 'abc' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/abc.pyc'>, 'posixpath': <module 'posixpath' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/posixpath.pyc'>, '_weakrefset': <module '_weakrefset' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/_weakrefset.pyc'>, 'errno': <module 'errno' (built-in)>, 'encodings.codecs': None, 'sre_constants': <module 'sre_constants' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/sre_constants.pyc'>, 're': <module 're' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/re.pyc'>, '_abcoll': <module '_abcoll' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/_abcoll.pyc'>, 'types': <module 'types' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/types.pyc'>, '_codecs': <module '_codecs' (built-in)>, 'encodings.__builtin__': None, '_warnings': <module '_warnings' (built-in)>, 'genericpath': <module 'genericpath' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/genericpath.pyc'>, 'stat': <module 'stat' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/stat.pyc'>, 'zipimport': <module 'zipimport' (built-in)>, '_sysconfigdata': <module '_sysconfigdata' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/_sysconfigdata.pyc'>, 'mpl_toolkits': <module 'mpl_toolkits' (built-in)>, 'warnings': <module 'warnings' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/warnings.pyc'>, 'UserDict': <module 'UserDict' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/UserDict.pyc'>, 'encodings.utf_8': <module 'encodings.utf_8' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/encodings/utf_8.pyc'>, 'sys': <module 'sys' (built-in)>, '_osx_support': <module '_osx_support' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/_osx_support.pyc'>, 'codecs': <module 'codecs' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/codecs.pyc'>, 'readline': <module 'readline' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/readline.so'>, 'os.path': <module 'posixpath' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/posixpath.pyc'>, '_locale': <module '_locale' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/_locale.so'>, 'signal': <module 'signal' (built-in)>, 'traceback': <module 'traceback' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/traceback.pyc'>, 'linecache': <module 'linecache' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/linecache.pyc'>, 'posix': <module 'posix' (built-in)>, 'encodings.aliases': <module 'encodings.aliases' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/encodings/aliases.pyc'>, 'exceptions': <module 'exceptions' (built-in)>, 'sre_parse': <module 'sre_parse' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/sre_parse.pyc'>, 'os': <module 'os' from '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/os.pyc'>, '_weakref': <module '_weakref' (built-in)>}
    >>> sys.modules['zlib'].__file__
    '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/zlib.so'

可以看到 sys.modules
已经保存了一些包的信息，由这些信息，我们就可以直接知道要查找的包的位置等信息。

#### finderloader 和 importer

在上文中，我们提到了 sys.meta\_path 中保证了一些 finder 对象。在 python
中，不仅定义了 finder 的概念，还定义了 loader 和 importor 的概念。

-   finder 的任务是决定自己是否根据名字找到相应的模块，在 py2 中，finder
    对象必须实现 find\_module()方法，在 py3 中必须要实现
    find\_module()或者 find\_loader（)方法。如果 finder
    可以查找到模块，则会返回一个 loader 对象(在 py3.4 中，修改为返回一个
    module specs)。
-   loader 则是负责加载模块，它必须实现一个 load\_module()的方法。
-   importer 则指一个对象，实现了 finder 和 loader 的方法。因为 Python
    是 duck type，只要实现了方法，就可以认为是该类。

#### sysmeta\_path

在 Python 查找的时候，如果在 sys.modules 没有查找到，就会依次调用
sys.meta\_path 中的 finder 对象。默认的情况下，sys.meta\_path
是一个空列表，并没有任何 finder 对象。

    In [6]: sys.meta_path
    Out[6]: []

我们可以向 sys.meta\_path 中添加一些定义的 finder，来实现对 Python
加载模块的修改。比如下例，我们实现了一个会将每次加载包的信息打印出来的
finder。

``` {.sourceCode .python}
from __future__ import print_function
import sys


class Watcher(object):
    @classmethod
    def find_module(cls, name, path, target=None):
        print("Importing", name, path, target)
        return None

sys.meta_path.insert(0, Watcher)

import socket
```

当我们执行的时候，就可以看到系统加载 socket 包时所发生的事情。

     ╭─liuchang@localhost  ~/Codes/pycon/ex5_meta_path
     ╰─$ python finder1.py
     Importing socket None None
     Importing _socket None None
     Importing functools None None
     Importing _functools None None
     Importing _ssl None None
     Importing cStringIO None None

#### syspath hook

Python import 的 hook 分为二类，一类是上一章节已经描述的 meta
hook，另一类是 path hook。

当处理 sys.path（或者 package.**path**)时，就会调用对应的一部分的 Pack
hook。Path Hook 是通过向 sys.path\_hooks 中添加一个 importer
生成器来注册的。

sys.path\_hooks
是由可被调用的对象组成，它会顺序的检查以决定他们是否可以处理给定的
sys.path 的一项。每个对象会使用 sys.path
项的路径来作为参数被调用。如果它不能处理该路径，就必须抛出
ImportError，如果可以，则会返回一个 importer
对象。之后，不会再尝试其它的 sys.path\_hooks 对象，即使前一个 importer
出错了。

详细可以参考[registering-hooks](https://www.python.org/dev/peps/pep-0302/#specification-part-2-registering-hooks)。

python import hooks
-------------------

在介绍完 Python
的引用机制与一些实现方法后，接下来我们介绍一些关于如何根据自己的需求来扩展
Python 的引用机制。

在开始详细介绍前，给大家展示一个实用性不高，但是很有意思的例子：**让
Python 在执行代码的时候自动安装缺失的类库**。我们会实现一个 autoinstall
的模块，只要 import 了该模块，就可以打开该功能。如下所示，我们尝试引入
tornado 库的时候，iPython 会提示我们没有安装。然后，我们引入了
autoinstall，再尝试引入 tornado，iPython 就会自动的安装 tornado 库。

    In [1]: import tornado
    ---------------------------------------------------------------------------
    ImportError                               Traceback (most recent call last)
    <ipython-input-1-3eac10687b7e> in <module>()
    ----> 1 import tornado

    ImportError: No module named tornado

    In [2]: import autoinstall

    In [3]: import tornado
    Installing tornado

    Collecting tornado
      Downloading tornado-4.2.1.tar.gz (434kB)
    Collecting backports.ssl-match-hostname (from tornado)
      Downloading http://182.92.2.186:7002/packages/backports.ssl_match_hostname-3.4.0.2-py2-none-any.whl
    Collecting certifi (from tornado)
      Downloading certifi-2015.9.6.2-py2.py3-none-any.whl (371kB)
    Installing collected packages: backports.ssl-match-hostname, certifi, tornado
      Running setup.py install for tornado
    Successfully installed backports.ssl-match-hostname-3.4.0.2 certifi-2015.9.6.2 tornado-4.2.1

这个功能的实现其实很简单，利用了 sys.meta\_path。autoinstall
的全部代码如下：

``` {.sourceCode .python}
from __future__ import print_function
import sys
import subprocess

class AutoInstall(object):
    _loaded = set()

    @classmethod
    def find_module(cls, name, path, target=None):
        if path is None and name not in cls._loaded:
            cls._loaded.add(name)
            print("Installing", name)
            try:
                out = subprocess.check_output(['sudo', sys.executable, '-m', 'pip', 'install', name])
                print(out)
            except Exception as e:
                print("Failed" + e.message)
        return None

sys.meta_path.append(AutoInstall)
```

### import hook 的重要性

我们为什么需要 Python import 的 hook 呢？使用 import 的 hook
可以让我们做到很多事情，比如说当我们的 Python
包存储在一个非标准的文件中，或者 Python 程序存储在网络数据库中，或者像
py2exe 一样将 Python
程序打包成了一个文件，我们需要一种方法来正确的解析它们。

其次，我们希望在 Python
加载类库的时候，可以额外的做一些事情，比如上传审计信息，比如延迟加载，比如自动解决上例的依赖未安装的问题。

所以，import 系统的 Hook 技术是值的花时间学习的。

### 如何实现 import hooks

Python 提供了一些方法，让我们可以在代码中动态的调用
import。主要有如下几种：

1.  \_\_import\_\_ : Python 的内置函数
2.  imputil : Python 的 import 工具库，在 py2.6 被声明废弃，py3
    中彻底移除。
3.  imp : Python2 的一个 import 库，py3 中移除
4.  importlib : Python3 中最新添加，backport 到
    py2.7，但只有很小的子集（只有一个函数）。

Python2 所有关于 import 的库的列表参见[Importing
Modules](https://docs.python.org/2/library/modules.html)。Python3
的可以参考[Importing
Modules](https://docs.python.org/3/library/modules.html) [PEP 0302 – New
Import Hooks](https://www.python.org/dev/peps/pep-0302) 提案详细的描述了
importlib 的目的、用法。

### 一些 hook 示例

#### lazy 化库引入

Lazy 化库引入

使用 Import Hook，我们可以达到 Lazy Import 的效果，当我们执行 import
的时候，实际上并没引入该库，只有真正的使用这个库的时候，才会将其引入到当前工作空间。
具体的代码可以参考[github](https://github.com/noahmorrison/limp)。
实现的效果如下：

``` {.sourceCode .python}
#!/usr/bin/python

import limp  # Lazy imports begin now

import json
import sys

print('json' in sys.modules)  # False
print(', '.join(json.loads('["Hello", "World!"]')))
print('json' in sys.modules)  # True
```

它的实现也很简单：

``` {.sourceCode .python}
import sys
import imp

_lazy_modules = {}

class LazyModule():
    def __init__(self, name):
        self.name = name

    def __getattr__(self, attr):
        path = _lazy_modules[self.name]
        f, pathname, desc = imp.find_module(self.name, path)

        lf = sys.meta_path.pop()
        imp.load_module(self.name, f, pathname, desc)
        sys.meta_path.append(lf)


        self.__dict__ = sys.modules[self.name].__dict__
        return self.__dict__[attr]

class LazyFinder(object):

    def find_module(self, name, path):
        _lazy_modules[name] = path
        return self

    def load_module(self, name):
        return LazyModule(name)

sys.meta_path.append(LazyFinder())
```

#### flask-插件库统一入口

使用过 Flask 的同学都知道，Flask
的对于插件提供了统一的入口。比如说我们安装了 Flask\_API
这个库，然后我们可以直接`import flask_api`来使用这个库，同时 Flask
还允许我们采用`import flask.ext.api`的方式来引用该库。

这里 Flask 就是使用了 import 的 hook，当引入 flask.ext
的包时，就自动的引用相应的库。Flask 实现了一个叫 ExtensionImporter
的类，这个类实现了 find\_module 和 load\_module
代码实现如下[github](https://github.com/mitsuhiko/flask/blob/master/flask/exthook.py#L27)：

``` {.sourceCode .python}
class ExtensionImporter(object):
    """This importer redirects imports from this submodule to other locations.
    This makes it possible to transition from the old flaskext.name to the
    newer flask_name without people having a hard time.
    """

    def __init__(self, module_choices, wrapper_module):
        self.module_choices = module_choices
        self.wrapper_module = wrapper_module
        self.prefix = wrapper_module + '.'
        self.prefix_cutoff = wrapper_module.count('.') + 1

    def __eq__(self, other):
        return self.__class__.__module__ == other.__class__.__module__ and \
               self.__class__.__name__ == other.__class__.__name__ and \
               self.wrapper_module == other.wrapper_module and \
               self.module_choices == other.module_choices

    def __ne__(self, other):
        return not self.__eq__(other)

    def install(self):
        sys.meta_path[:] = [x for x in sys.meta_path if self != x] + [self]

    def find_module(self, fullname, path=None):
        if fullname.startswith(self.prefix):
            return self

    def load_module(self, fullname):
        if fullname in sys.modules:
            return sys.modules[fullname]
        modname = fullname.split('.', self.prefix_cutoff)[self.prefix_cutoff]
        for path in self.module_choices:
            realname = path % modname
            try:
                __import__(realname)
            except ImportError:
                exc_type, exc_value, tb = sys.exc_info()
                # since we only establish the entry in sys.modules at the
                # very this seems to be redundant, but if recursive imports
                # happen we will call into the move import a second time.
                # On the second invocation we still don't have an entry for
                # fullname in sys.modules, but we will end up with the same
                # fake module name and that import will succeed since this
                # one already has a temporary entry in the modules dict.
                # Since this one "succeeded" temporarily that second
                # invocation now will have created a fullname entry in
                # sys.modules which we have to kill.
                sys.modules.pop(fullname, None)

                # If it's an important traceback we reraise it, otherwise
                # we swallow it and try the next choice.  The skipped frame
                # is the one from __import__ above which we don't care about
                if self.is_important_traceback(realname, tb):
                    reraise(exc_type, exc_value, tb.tb_next)
                continue
            module = sys.modules[fullname] = sys.modules[realname]
            if '.' not in modname:
                setattr(sys.modules[self.wrapper_module], modname, module)
            return module
        raise ImportError('No module named %s' % fullname)
```

然后在 Flask 的 ext 目录下的\_\_init\_\_.py 文件中，初始化了该
Importer。

``` {.sourceCode .python}
def setup():
    from ..exthook import ExtensionImporter
    importer = ExtensionImporter(['flask_%s', 'flaskext.%s'], __name__)
    importer.install()
```


#::相关下载

[下载地址 = http://onlookee.com/?c=Article&a=download&id=10](http://onlookee.com/?c=Article&a=download&id=10)

#::theEnd