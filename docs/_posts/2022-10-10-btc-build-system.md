---
title: 比特币源码学习01-编译系统
published: true
---

比特币的编译系统使用Automake工具。Automake可以生成和GNU标准兼容的Makefiles文件。

## Automake工具通常的执行流程

当我们下载下来一个使用automake作为编译系统的源代码时，编译和安装的流程通常如下：

```bash
$ tar zxf package-version.tar.gz
$ cd package-version
$ ./configure
$ make
$ make check
$ make install
```

### 下面是常见的make目标的解释

* make all
编译可执行文件、库文件、文档等（等同于make）

* make install
安装文件到指定目录

* make install-strip
与make install相同，同时去除debug信息

* make uninstall
与make install的做用相反

* make clean
删除make命令生成的文件

* make distclean
删除./configure命令生成的文件

* make check
运行测试用例

* make installcheck
测试安装好的程序

* make dist
从源文件中创建 package-version.tar.gz

### 常见的预定义目录

* prefix          /usr/local
* exec_prefix     ${prefix}
* bindir          ${exec_prefix}/bin
* libdir          ${exec_prefix}/lib
* ...
* includedir      ${prefix}/include
* datarootdir     ${prefix}/share
* datadir         ${datarootdir}
* mandir          ${datarootdir}/man
* infodir         ${datarootdir}/info
* docdir          ${datarootdir}/doc/${PACKAGE}

configure时可以改变这些目录的默认值。

### 常见的配置变量
configure时还可以指定额外的编译配置：

* cc
c编译器

* CFLAGS
c编译选项

* CXX
C++ 编译器

* CXXFLAGS
C++编译选项

* LDFLAGS
链接选项

* CPPFLAGS
c/c++预处理选项

## Hello World

下面以一个简单的helloworld程序为例，演示autoconfig编译系统如何配置。

### 在一个空目录下创建如下文件：

* src/main.c

```
$ cat src/main.c

#include <config.h>
#include <stdio.h>

int
main (void)
{
  puts ("Hello World!");
  puts ("This is " PACKAGE_STRING ".");
  return 0;
}
```
这里用到的config.h文件时autoconfig自动生成的，包含了一系列宏定义。代码中使用到了一个：PACKAGE_STRING

* README
```
$ cat README
This is a demonstration package for GNU Automake.
Type 'info Automake' to read the Automake manual.
```

* Makefile.am 和 src/Makefile.am

```
$ cat src/Makefile.am
bin_PROGRAMS = hello
hello_SOURCES = main.c

$ cat Makefile.am
SUBDIRS = src
dist_doc_DATA = README
```
Makefile.am的语法和Makefile的语法是一致的。

以`_PROGRAMS`为后缀的变量是特殊变量，指定了Makefile生成的目标名称。类似的后缀有：
`_SCRIPTS`, `_DATA`, `_LIBRARIES`对应不同类型的文件。

`bin_PROGRAMS`变量里的bin表示该目标将安装到安装目录里的`binddir`目录下。

对于每一个`bin_PROGRAMS`指定的编译目标xxx，automake都会去找对应的源文件来编译，源文件使用xxx_SOURCES来指定。

对于顶层Makefile.am而言，通常使用SUBDIR包含需要递归处理的目录。

`dist_doc_DATA`变量中的doc指定README文件被安装在`docdir`，dist指定安装的时候也需要把README打包在内。

* configure.ac
```
$ cat configure.ac
AC_INIT([amhello], [1.0], [bug-automake@gnu.org])
AM_INIT_AUTOMAKE([-Wall -Werror foreign])
AC_PROG_CC
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([
 Makefile
 src/Makefile
])
AC_OUTPUT
```

以AC开头的指令是Autoconfig的宏。以AM开头的指令是Automake的宏。
第一行初始化Autoconfig，设置了软件包的名称、版本号以及报告bug的邮箱地址；
第二行初始化Automake，foreign参数表示改软件包不遵循GUN标准（没有ChangeLog等文件）。
第三行AC_PROG_CC宏会搜索C编译器并设置CC变量。
第四行AC_CONFIG_HEADERS([config.h])告诉configure命令创建config.h文件，该文件包含configure.ac定义的宏。比如AC_INIT就会定义一些宏变量
AC_CONFIG_FILES宏列出了需要生成的Makefile文件，这些文件会使用Makefile.in来生成。
AC_OUTPUT是结束命令，告诉Aotoconfig启动生成文件。

### 初始化编译系统

```
$ autoreconf --install
```

这一步会生成configure、config.h.in,Makefile.in等文件。这一步之需要执行一次。

### 执行configure和make

```
$ ./configure
$ make
$ ./src/hello
Hello World!
This is amhello 1.0.
```

## bitcoin-core 编译系统解读

### 源码目录结构

```
- build-aux autoconfig 编译辅助文件
- ci        ci脚本
- contrib   第三方脚本
- depends   交叉编译依赖脚本？
- doc       文档
- share     配置文件
- src       源代码目录
  - bench   benchmark 代码
  - common  bloom算法实现
  - compat  字节序列化/反序列化
  - config  autoconfig生成的config头文件目录
  - consensus 共识算法？
  - crc32c  CRC32C算法实现
  - crypto  加密相关算法实现
  - index   区块索引实现
  - init    不同客户端/服务端的初始化代码
  - interface 不同客户端/服务端的界面，目前还不稳定
  - ipc     ipc接口
  - kernel 
  - leveldb
  - logging
  - node
  - policy      交易费用定义？
  - primitives  区块和交易结构定义
  - qt
  - rpc
  - script      交易脚本
  - secp256k1   椭圆曲线加密
  - support
  - test
  - univalue
  - util
  - wallet
  - zmq
  - minisketch
  - net.* .     p2p网络管理
  - init.cpp    节点初始化
  - main.*      区块链管理
  - chain.*     区块链实现
  - coins.*     比特币管理
  - miner.* .   挖矿
  - Makefile.am
- test      测试目录
- configure.ac
- Makefile.am
- README.md
```

### configure.ac

#### 1. 初始化Autoconfig
```
# 设定autoconfig最低版本
AC_PREREQ([2.69])

# 初始化autoconfig
define(_CLIENT_VERSION_MAJOR, 23)
define(_CLIENT_VERSION_MINOR, 99)
define(_CLIENT_VERSION_BUILD, 0)
define(_CLIENT_VERSION_RC, 0)
define(_CLIENT_VERSION_IS_RELEASE, false)
define(_COPYRIGHT_YEAR, 2022)
define(_COPYRIGHT_HOLDERS,[The %s developers])
define(_COPYRIGHT_HOLDERS_SUBSTITUTION,[[Bitcoin Core]])
AC_INIT([Bitcoin Core],m4_join([.], _CLIENT_VERSION_MAJOR, _CLIENT_VERSION_MINOR, _CLIENT_VERSION_BUILD)m4_if(_CLIENT_VERSION_RC, [0], [], [rc]_CLIENT_VERSION_RC),[https://github.com/bitcoin/bitcoin/issues],[bitcoin],[https://bitcoincore.org/])

# 设定源代码目录
AC_CONFIG_SRCDIR([src/validation.cpp])

# 设定生成config.h文件
AC_CONFIG_HEADERS([src/config/bitcoin-config.h])

# 设定帮助文件目录
AC_CONFIG_AUX_DIR([build-aux])
AC_CONFIG_MACRO_DIR([build-aux/m4])
```

#### 2. 编译目标名称设置

```
BITCOIN_DAEMON_NAME=bitcoind
BITCOIN_GUI_NAME=bitcoin-qt
BITCOIN_TEST_NAME=test_bitcoin
BITCOIN_CLI_NAME=bitcoin-cli
BITCOIN_TX_NAME=bitcoin-tx
BITCOIN_UTIL_NAME=bitcoin-util
BITCOIN_CHAINSTATE_NAME=bitcoin-chainstate
BITCOIN_WALLET_TOOL_NAME=bitcoin-wallet
dnl Multi Process
BITCOIN_MP_NODE_NAME=bitcoin-node
BITCOIN_MP_GUI_NAME=bitcoin-gui
```

#### 3. 初始化 Automake

```
AM_INIT_AUTOMAKE([1.13 no-define subdir-objects foreign])

AM_MAINTAINER_MODE([enable])

dnl make the compilation flags quiet unless V=1 is used
AM_SILENT_RULES([yes])

dnl Compiler checks (here before libtool).
if test "${CXXFLAGS+set}" = "set"; then
  CXXFLAGS_overridden=yes
else
  CXXFLAGS_overridden=no
fi
AC_PROG_CXX
```

#### 4. configure 可选参数设置

使用 AC_ARG_WITH 和 AC_ARG_ENABLE 宏

#### 5. 导出变量到Makefile.in

使用 AC_SUBST

#### 5. 打印 configure 结果

### Makefile.am

#### 1. 指定子目录

```
SUBDIRS = src
```

#### 2. 可执行文件目标

```
$(BITCOIN_QT_BIN): FORCE
	$(MAKE) -C src qt/$(@F)

$(BITCOIND_BIN): FORCE
	$(MAKE) -C src $(@F)

$(BITCOIN_CLI_BIN): FORCE
	$(MAKE) -C src $(@F)

$(BITCOIN_TX_BIN): FORCE
	$(MAKE) -C src $(@F)

$(BITCOIN_UTIL_BIN): FORCE
	$(MAKE) -C src $(@F)

$(BITCOIN_WALLET_BIN): FORCE
	$(MAKE) -C src $(@F)

$(BITCOIN_NODE_BIN): FORCE
	$(MAKE) -C src $(@F)

$(BITCOIN_GUI_BIN): FORCE
	$(MAKE) -C src $(@F)
```

### src/Makefile.am

指定静态库和可执行程序的源文件和编译选项。

#### 1. 静态库目标

```
LIBBITCOIN_NODE=libbitcoin_node.a
LIBBITCOIN_COMMON=libbitcoin_common.a
LIBBITCOIN_CONSENSUS=libbitcoin_consensus.a
LIBBITCOIN_CLI=libbitcoin_cli.a
LIBBITCOIN_UTIL=libbitcoin_util.a
LIBBITCOIN_CRYPTO_BASE=crypto/libbitcoin_crypto_base.la
LIBBITCOINQT=qt/libbitcoinqt.a
LIBSECP256K1=secp256k1/libsecp256k1.la
```

#### 2. 二进制目标

```
if BUILD_BITCOIND
  bin_PROGRAMS += bitcoind
endif

if BUILD_BITCOIN_NODE
  bin_PROGRAMS += bitcoin-node
endif

if BUILD_BITCOIN_CLI
  bin_PROGRAMS += bitcoin-cli
endif

if BUILD_BITCOIN_TX
  bin_PROGRAMS += bitcoin-tx
endif

if ENABLE_WALLET
if BUILD_BITCOIN_WALLET
  bin_PROGRAMS += bitcoin-wallet
endif
endif

if BUILD_BITCOIN_UTIL
  bin_PROGRAMS += bitcoin-util
endif

if BUILD_BITCOIN_CHAINSTATE
  bin_PROGRAMS += bitcoin-chainstate
endif
```