---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Python
---

# 一. setup.py
  我们从例子开始。假设你要分发一个叫foo的模块，文件名foo.py，那么setup.py内容如下
  ```
  from distutils.core import setup
  setup(name='foo',
        version='1.0',
        py_modules=['foo'],
  )
  ```
  
  然后，运行`python setup.py sdist`为模块创建一个源码包
  ```
  root@network:/kong/setup# python setup.py sdist
  running sdist
  running check
  warning: check: missing required meta-data: url
  warning: check: missing meta-data: either (author and author_email) or (maintainer and maintainer_email) must be supplied
  warning: sdist: manifest template 'MANIFEST.in' does not exist (using default file list)
  warning: sdist: standard file not found: should have one of README, README.txt
  writing manifest file 'MANIFEST'
  creating foo-1.0
  making hard links in foo-1.0...
  hard linking foo.py -> foo-1.0
  hard linking setup.py -> foo-1.0
  creating dist
  Creating tar archive
  removing 'foo-1.0' (and everything under it)
  ```

  在当前目录下，会创建dist目录，里面有个文件名为foo-1.0.tar.gz，这个就是可以分发的包。使用者拿到这个包后，解压，到foo-1.0目录下执行：python setup.py install，那么，foo.py就会被拷贝到python类路径下，可以被导入使用。
  ```
  root@network:/kong/setup/dist/foo-1.0# python setup.py install
  running install
  running build
  running build_py
  creating build
  creating build/lib.linux-x86_64-2.7
  copying foo.py -> build/lib.linux-x86_64-2.7
  running install_lib
  copying build/lib.linux-x86_64-2.7/foo.py -> /usr/local/lib/python2.7/dist-packages
  byte-compiling /usr/local/lib/python2.7/dist-packages/foo.py to foo.pyc
  running install_egg_info
  Removing /usr/local/lib/python2.7/dist-packages/foo-1.0.egg-info
  Writing /usr/local/lib/python2.7/dist-packages/foo-1.0.egg-info
  root@network:/kong/setup/dist/foo-1.0# ll /usr/local/lib/python2.7/dist-packages/foo
  foo-1.0.egg-info foo.py foo.pyc
  ```
  
  若要生成RPM包，执行python setup.py bdist_rpm，但系统必须有rpm命令的支持。可以运行下面的命令查看所有格式的支持
  ```
  root@network:/kong/setup# python setup.py bdist --help-formats
  List of available distribution formats:
     --formats=rpm RPM distribution
     --formats=gztar gzip'ed tar file
     --formats=bztar bzip2'ed tar file
     --formats=ztar compressed tar file
     --formats=tar tar file
     --formats=wininst Windows executable installer
     --formats=zip ZIP file
     --formats=msi Microsoft Installer
  ```

# 二. setup函数参数
1. packages
   告诉Distutils需要处理那些包（包含__init__.py的文件夹）
2. package_dir
   告诉Distutils哪些目录下的文件被映射到哪个源码包。一个例子：package_dir = {'': 'lib'}，表示“root package”中的模块都在lib目录中。
3. ext_modules
   是一个包含Extension实例的列表，Extension的定义也有一些参数。
4. ext_package
   定义extension的相对路径
5. requires
   定义依赖哪些模块
6. provides
   定义可以为哪些模块提供依赖
7. scripts
   指定python源码文件，可以从命令行执行。在安装时指定--install-script
8. package_data
   通常包含与包实现相关的一些数据文件或类似于readme的文件。如果没有提供模板，会被添加到MANIFEST文件中。
9. data_files
   指定其他的一些文件（如配置文件）

```
setup(...,
data_files=[('bitmaps', ['bm/b1.gif', 'bm/b2.gif']),
('config', ['cfg/data.cfg']),
('/etc/init.d', ['init-script'])]
)
```

规定了哪些文件被安装到哪些目录中。如果目录名是相对路径，则是相对于sys.prefix或sys.exec_prefix的路径。如果没有提供模板，会被添加到MANIFEST文件中。

执行sdist命令时，默认会打包哪些东西呢？
1. 所有由py_modules或packages指定的源码文件
2. 所有由ext_modules或libraries指定的C源码文件
3. 由scripts指定的脚本文件
4. 类似于test/test*.py的文件
5. README.txt或README，setup.py，setup.cfg
6. 所有package_data或data_files指定的文件

# 三. Setuptools
  上面的setup.py和setup.cfg都是遵循python标准库中的Distutils，而setuptools工具针对Python官方的distutils做了很多针对性的功能增强，比如依赖检查，动态扩展等。很多高级功能我就不详述了，自己也没有用过，等用的时候再作补充。

  一个典型的遵循setuptools的脚本：
```
from setuptools import setup, find_packages
setup(
name = "HelloWorld",
version = "0.1",
packages = find_packages(),
scripts = ['say_hello.py'],

# Project uses reStructuredText, so ensure that the docutils get
# installed or upgraded on the target machine
install_requires = ['docutils>=0.3'],

package_data = {
# If any package contains *.txt or *.rst files, include them:
'': ['*.txt', '*.rst'],
# And include any *.msg files found in the 'hello' package, too:
'hello': ['*.msg'],
},

# metadata for upload to PyPI
author = "Me",
author_email = "me@example.com",
description = "This is an Example Package",
license = "PSF",
keywords = "hello world example examples",
url = "http://example.com/HelloWorld/", # project home page, if any

# could also include long_description, download_url, classifiers, etc.
)
```

  如何让一个egg可被执行？
```
setup(
# other arguments here...
entry_points = {
'setuptools.installation': [
'eggsecutable = my_package.some_module:main_func',
]})
```

# 四. setup.py和pip
  表面上，python setup.py install和pip install都是用来安装python包的，实际上，pip提供了更多的特性，更易于使用。

1. pip会自动下载依赖，而如果使用setup.py，则需要手动搜索和下载；
2. pip会自动管理包的信息，使卸载/更新更加方便和容易，使用pip uninstall即可。而使用setup.py，必须手动删除，有时容易出错。
3. pip提供了对virtualenv更好的整合。

