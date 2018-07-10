---
title: VS 下 GeoDa 开发环境配置
date: 2018-03-09 18:58:45
tags:
---
最近帮师兄配置 GeoDa 的环境，顺便记录下。
<!-- more -->

目录：

- [依赖库下载与安装](#%E4%BE%9D%E8%B5%96%E5%BA%93%E4%B8%8B%E8%BD%BD%E4%B8%8E%E5%AE%89%E8%A3%85)
    - [下载依赖库](#%E4%B8%8B%E8%BD%BD%E4%BE%9D%E8%B5%96%E5%BA%93)
    - [安装依赖库](#%E5%AE%89%E8%A3%85%E4%BE%9D%E8%B5%96%E5%BA%93)
        - [GDAL 编译安装](#gdal-%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85)
        - [wxWidgets 编译](#wxwidgets-%E7%BC%96%E8%AF%91)
        - [Eigen](#eigen)
        - [Boost 编译](#boost-%E7%BC%96%E8%AF%91)
        - [BLAS 和 CLAPACK 库编译](#blas-%E5%92%8C-clapack-%E5%BA%93%E7%BC%96%E8%AF%91)
        - [SQLite 编译、cURL 编译、 json_spirit 编译](#sqlite-%E7%BC%96%E8%AF%91%E3%80%81curl-%E7%BC%96%E8%AF%91%E3%80%81-jsonspirit-%E7%BC%96%E8%AF%91)
- [GeoDa 的编译](#geoda-%E7%9A%84%E7%BC%96%E8%AF%91)

主要会用到一些工具：

- Visual Studio 的命令提示符，主要用到 nmake 命令。
- Visual Studio，我使用的是 Visual Studio 2017。
- Internet Download Manager(IDM)，用于下载依赖库。
- [CMake][CMake-website]，用于编译一些库。
- [MinGW][MinGW-home]，用于编译 Fortan 源码和一些库。

下面分别说明编译过程

# 依赖库下载与安装

## 下载依赖库

> GeoDa 要下载依赖库，吧？要的吧？官网没说啊。但是感觉肯定要吧。都没说哪些怎么下载啊！！！

具体这个工程用到什么依赖库，可以在项目设置的“附加依赖项”中进行查看。

根据官网给出的 [README][geoda-build-readme] 文件，需要运行一个 `Build.bat` 的批处理文件。
但是运行这个文件后，会开始下载一些依赖库。

由于依赖库放在亚马逊云上，国内往往是访问不到的。
因此需要提取出 `Build.bat` 中的下载链接，手动下载这些包。
还有一种方法，就是把这些第三方库分别从其官网下载对应的版本。

如果一个一个找下载链接，那就太慢了。这时可以使用 IDM 提取剪贴板内下载链接的功能，批量下载。
具体过程就不详述了。需要注意的是，MySQL 按照文件中给出的地址是无法下载的，需要自己下载。

## 安装依赖库

这里就要大量使用到 VS 的命令提示符了，大量使用 nmake 以命令行的方式安装。

### GDAL 编译安装

> 致谢：在土哥大神的指导下，我完成了 GDAL 的编译和安装。编译详细信息可参见 `nmake.opt` 文件。

编译 GDAL 之前要编译 GEOS 、 proj4 两个库。编译方法见后面。

1. 在 GDAL 的**安装目录**（下文使用 `%GDAL_HOME%` 以模拟环境变量的方式表示）下，运行以下命令

    ``` bash
    nmake -f makefile.vc MSVC_VER=1910 DEBUG=1
    ```

    需要注意的是，`MSVC_VER` 变量代表了编译器的版本。

    使用对应的版本号替换即可。另外， `DEBUG` 参数为 1 表示以 DEBUG 方式编译，为 0 表示以 RELEASE 方式编译。

1. 编译完成后，需要安装 GDAL，才能把库文件放置在硬盘上。在 `nmake.opt` 文件中，找到下面行并修改：

    ``` bash
    - GDAL_HOME = "C:\warmerda\bld"
    + GDAL_HOME = "D:\lib\gdal"
    ```

    然后运行命令

    ``` bash
    nmake -f makefile.vc MSVC_VER=1910 DEBUG=1 DEVINSTALL   # DEBUG 环境下
    nmake -f makefile.vc MSVC_VER=1910 DEBUG=1 INSTALL      # RELEASE 环境下
    ```

    就可以把库文件放置在 `GDAL_HOME` 设定的目录中。

1. 编译完成后，将 `%GDAL_HOME%\include` 路径添加到 GeoDa 工程的包含目录中；
   将 `%GDAL_HOME%\lib` 路径添加到 GeoDa 工程的库目录中。

| `MSVC_VER` 值 | 版本号                                                             |
| ------------- | ------------------------------------------------------------------ |
| 1910          | 15.0(2017)                                                         |
| 1900          | 14.0(2015)                                                         |
| 1800          | 12.0(2013)                                                         |
| 1700          | 11.0(2012)                                                         |
| 1600          | 10.0(2010)                                                         |
| 1500          | 9.0 (2008)                                                         |
| 1400          | 8.0 (2005) - specific compilation flags, different from older VC++ |
| 1310          | 7.1 (2003) # is it still supported ?                               |
| 1300          | 7.0 (2002) # is it still supported ?                               |

### wxWidgets 编译

wxWidgets 是开源跨平台的 GUI 库，GeoDa 的界面基于 wxWidgets。
但是为了多语言，wxWidgets 使用 Unicode 编译，因此 GeoDa 也要用 Unicode 编译。在咨询了开发人员之后
（[Issus #1598: Why use a wxWidgets built in unicode?](https://github.com/GeoDaCenter/geoda/issues/1598)），
他们告诉了我正确的编译方式。

设 wxWidgets 的源文件目录为 `%WX_HOME%`，
则它的 `makefile.vc` 文件位于 `%WX_HOME%\build\msw`，其编译指令如下：

``` bash
nmake -f makefile.vc UNICODE=1 SHARED=1 RUNTIME_LIBS=dynamic MONOLITHIC=1 USE_OPENGL=1 USE_POSTSCRIPT=1 TARGET_CPU=AMD64
```

这里的参数：

- `BUILD=debug` 表示 DEBUG 方式编译。如果要 RELEASE 方式编译，使用 `BUILD=release`。
- `MONOLIHIC=1` 表示将所有的库打包到一个文件中，参见 [WxWidgets Build Configurations][wx-wiki]
    这种方式编译时，必须编译动态库。
- `SHARED=1` 表示编译 DLL 动态链接库。
- `RUNTIME_LIBS=dynamic` 表示编译动态运行时库。

<!-- 此外还需要加一个参数： `wxUSE_WCHAR_T=1`，表示定义 `wxUSE_WCHAR_T` 宏，才能顺利编译通过。 -->

编译完成后，将 `%WX_HOME%\include`、`%WX_HOME%\include\msvc` 路径添加到 GeoDa 工程的包含目录中；
GeoDa 需要 `%WX_HOME%\lib\vc_lib\mswud\msvc\setup.h` 头文件，请确保其存在

### Eigen

Eigen 是矩阵运算的库，无需编译。设 Eigen 的源文件目录为 `%EIGEN_HOME%`，
则将 `%EIGEN_HOME%` 路径添加到 GeoDa 工程的包含目录中即可。

SQLite 相同。

### Boost 编译

> 参考 [boost 1.56.0 编译及使用][boost-build-blog]，版本不同但方法相同。

下载 Boost 库后，设目录为 `%BOOST_SRC_HOME%`，使用 VS 命令提示符运行 `%BOOST_SRC_HOME%\bootstrap.bat`批处理文件。
可以得到 `b2.exe` 和 `bjam.exe`，两个文件作用相同。

我用的编译命令是：

``` bash
b2 install --toolset=msvc-9.0 --without-python --prefix="E:\SDK\boost\bin\vc9" link=static runtime-link=shared runtime-link=static threading=multi debug release
```

参数含义是：

| 参数              | 含义                                                                                                                                      |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `stage/install`   | stage表示只生成库（dll和lib），install还会生成包含头文件的include目录。                                                                   |
| `toolset`         | 指定编译器，可选的如borland、gcc、msvc（VC6）、msvc-9.0（VS2008）等。                                                                     |
| `without/with`    | 选择不编译/编译哪些库。                                                                                                                   |
| `stagedir/prefix` | stage时使用stagedir，install时使用prefix，表示编译生成文件的路径。推荐给不同的IDE指定不同的目录。                                         |
| `build-dir`       | 编译生成的中间文件的路径。这里记为 `%BOOST_HOME%`。                                                                                       |
| `link`            | 生成动态链接库/静态链接库。生成动态链接库需使用shared方式，生成静态链接库需使用static方式。                                               |
| `runtime-link`    | 动态/静态链接C/C++运行时库。同样有shared和static两种方式，这样runtime-link和link一共可以产生4种组合方式，各人可以根据自己的需要选择编译。 |
| `threading`       | 单/多线程编译。                                                                                                                           |
| `debug/release`   | 编译debug/release版本。                                                                                                                   |

> `link` 和 `runtime-link` 的缺省配置是 `link=static`、`runtime-link=shared`。

编译完成后，将 `%BOOST_HOME%\include` 路径添加到 GeoDa 工程的包含目录中；
将 `%BOOST_HOME%\lib` 路径添加到 GeoDa 工程的库目录中，
在“附加依赖项”中修改引用的 Boost 库 lib 的版本。

> 由于 GeoDa 只引用了 thread 这一个模块，因此可以只编译这一个模块，减少编译时间。

### BLAS 和 CLAPACK 库编译

BALS 和 CLAPACK 都是 GeoDa 的依赖库，但是 BALS 已经包括在 CLAPACK 中。只需要编译 CLAPACK 即可。

下载 [CLAPACK 的 VS 解决方案][CLAPACK-vs-sln]，完成后打开解决方案文件，运行 VS 编译即可。

> 官网给出的是使用 CMake 编译的方法，但是我在用 CMake 的时候总是报错，所以直接找了解决方案。

### SQLite 编译、cURL 编译、 json_spirit 编译

有人在 [GitHub][sqlite-cmake-build] 上开源了 SQLite3 的 CMakeLists 文件，可以直接拿来编译。

1. 克隆其 GitHub 仓库：
    ``` bash
    git clone https://github.com/snikulov/sqlite.cmake.build.git
    ```
2. 使用 CMake 创建 VS 工程。
3. 打开工程后，直接编译即可。

其他两个库的方法相同。

# GeoDa 的编译

把上面这些依赖库编译好了之后，打开 GeoDa 的 VS 工程，要进行如下修改：

1. 将项目设置为使用 Unicode 编译。
2. 包含目录和库目录加入之前编译的依赖库。
3. 附加依赖项中，
    - Boost 依赖项改为编译出来的依赖项；
    - 所有 `wx` 开头的依赖项，结尾如果是 `d` 不是 `ud` 的，改为 `ud`；
    - 其他依赖项如果名称有误，改为编译出来的名称。

然后可以执行编译。

> GeoDa 编译的工作实在是太繁重了，因此本文未完，将来会更新编译的最新进展。

[CMake-website]:https://cmake.org/
[MinGW-home]:http://mingw.org/
[geoda-build-readme]:https://github.com/GeoDaCenter/geoda/blob/master/BuildTools/windows/readme.md
[boost-build_blog]:http://www.cnblogs.com/zhcncn/p/3950477.html
[BLAS-website]:BLAS-website
[wx-wiki]:https://wiki.wxwidgets.org/WxWidgets_Build_Configurations#configure
[CLAPACK-vs-sln]:http://www.netlib.org/clapack/CLAPACK-3.1.1-VisualStudio.zip
[sqlite-cmake-build]:https://github.com/snikulov/sqlite.cmake.build