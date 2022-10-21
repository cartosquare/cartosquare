---
title: 比特币源码学习01-编译系统
published: true
---

比特币的编译系统使用Automake工具。Automake可以生成和GNU标准兼容的Makefiles文件。

## Automake工具通常的执行流程

当我们下载下来一个使用automake作为编译系统的源代码包时，编译和安装的流程通常如下：

```bash
$ tar zxf package-version.tar.gz
$ cd package-version
$ ./configure
$ make
$ make check
$ make install
```

### 下面是常见的make目标

* make all
编译可执行文件、库文件、文档等（等同于make）

* make install
安装文件到指定目录

* make install-strip
与make install相同，同时去除debug信息

* make uninstall
与make install的作用相反

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

* prefix          `/usr/local`
* exec_prefix     `${prefix}`
* bindir          `${exec_prefix}/bin`
* libdir          `${exec_prefix}/lib`
* ...
* includedir      `${prefix}/include`
* datarootdir     `${prefix}/share`
* datadir         `${datarootdir}`
* mandir          `${datarootdir}/man`
* infodir         `${datarootdir}/info`
* docdir          `${datarootdir}/doc/${PACKAGE}`

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
这里用到的config.h文件是Autoconfig自动生成的，包含了一系列宏定义。main.c中使用到了一个：PACKAGE_STRING —— 代码库名称。

* README

```
$ cat README
This is a demonstration package for GNU Automake.
Type 'info Automake' to read the Automake manual.
```

* Makefile.am 和 src/Makefile.am

在Aotoconfig中，我们不亲自写Makefile文件，而是通过Makefile.am文件来自动生成Makefile文件。

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

configure.ac文件用来自动生成configure文件。

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

这一步会生成configure、config.h.in,Makefile.in等文件。这一步只需要执行一次。

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

### 编译配置

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

#### 2. 编译目标

configure.ac 中定义了最终生成目标的名称：

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

不同目标的作用如下：

| Name                     | Description |
|--------------------------|-------------|
| *bitcoind* | RPC 服务端 |
| *bitcoin-qt* | 客户端+服务端 |
| *bitcoin-cli* | RPC客户端 |
| *bitcoin-tx* | transaction工具 |
| *bitcoin-wallet* | wallet 工具 |
| *bitcoin-util* | util 工具 |
| *bitcoin-chainstate* | chainstate 工具|
| *test-bitcoin* | 单元测试 |
| *bitcoin-node* | bitcoind的多进程版本 |
| *bitcoin-gui* | bitcoin-qt的多进程版本 |


这些目标通过AC_SUBST导出到makefile.in中：

```
AC_SUBST(BITCOIN_DAEMON_NAME)
AC_SUBST(BITCOIN_GUI_NAME)
AC_SUBST(BITCOIN_TEST_NAME)
AC_SUBST(BITCOIN_CLI_NAME)
AC_SUBST(BITCOIN_TX_NAME)
AC_SUBST(BITCOIN_UTIL_NAME)
AC_SUBST(BITCOIN_CHAINSTATE_NAME)
AC_SUBST(BITCOIN_WALLET_TOOL_NAME)
AC_SUBST(BITCOIN_MP_NODE_NAME)
AC_SUBST(BITCOIN_MP_GUI_NAME)
```

根目录的makefile.ac 将这些目标的生成工作交到src目录下的Makefile.ac中：

```
BITCOIND_BIN=$(top_builddir)/src/$(BITCOIN_DAEMON_NAME)$(EXEEXT)
BITCOIN_QT_BIN=$(top_builddir)/src/qt/$(BITCOIN_GUI_NAME)$(EXEEXT)
BITCOIN_TEST_BIN=$(top_builddir)/src/test/$(BITCOIN_TEST_NAME)$(EXEEXT)
BITCOIN_CLI_BIN=$(top_builddir)/src/$(BITCOIN_CLI_NAME)$(EXEEXT)
BITCOIN_TX_BIN=$(top_builddir)/src/$(BITCOIN_TX_NAME)$(EXEEXT)
BITCOIN_UTIL_BIN=$(top_builddir)/src/$(BITCOIN_UTIL_NAME)$(EXEEXT)
BITCOIN_WALLET_BIN=$(top_builddir)/src/$(BITCOIN_WALLET_TOOL_NAME)$(EXEEXT)
BITCOIN_NODE_BIN=$(top_builddir)/src/$(BITCOIN_MP_NODE_NAME)$(EXEEXT)
BITCOIN_GUI_BIN=$(top_builddir)/src/$(BITCOIN_MP_GUI_NAME)$(EXEEXT)
BITCOIN_WIN_INSTALLER=$(PACKAGE)-$(PACKAGE_VERSION)-win64-setup$(EXEEXT)

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


在src/Makefile.am中，具体指定了每个目标的生成方式，下面具体看`bitcoind`和`bitcoind-cli`这两个目标：

##### bitcoind

```bash
if BUILD_BITCOIND
  bin_PROGRAMS += bitcoind
endif

# bitcoind & bitcoin-node binaries #
bitcoin_daemon_sources = bitcoind.cpp
bitcoin_bin_cppflags = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES)
bitcoin_bin_cxxflags = $(AM_CXXFLAGS) $(PIE_FLAGS)
bitcoin_bin_ldflags = $(RELDFLAGS) $(AM_LDFLAGS) $(LIBTOOL_APP_LDFLAGS) $(PTHREAD_FLAGS)

if TARGET_WINDOWS
bitcoin_daemon_sources += bitcoind-res.rc
endif

bitcoin_bin_ldadd = \
  $(LIBBITCOIN_WALLET) \
  $(LIBBITCOIN_COMMON) \
  $(LIBBITCOIN_UTIL) \
  $(LIBUNIVALUE) \
  $(LIBBITCOIN_ZMQ) \
  $(LIBBITCOIN_CONSENSUS) \
  $(LIBBITCOIN_CRYPTO) \
  $(LIBLEVELDB) \
  $(LIBMEMENV) \
  $(LIBSECP256K1)

bitcoin_bin_ldadd += $(BDB_LIBS) $(MINIUPNPC_LIBS) $(NATPMP_LIBS) $(EVENT_PTHREADS_LIBS) $(EVENT_LIBS) $(ZMQ_LIBS) $(SQLITE_LIBS)

bitcoind_SOURCES = $(bitcoin_daemon_sources) init/bitcoind.cpp
bitcoind_CPPFLAGS = $(bitcoin_bin_cppflags)
bitcoind_CXXFLAGS = $(bitcoin_bin_cxxflags)
bitcoind_LDFLAGS = $(bitcoin_bin_ldflags)
bitcoind_LDADD = $(LIBBITCOIN_NODE) $(bitcoin_bin_ldadd)
```
可以看出，bitcoind依赖`LIBBITCOIN_WALLET`, `LIBBITCOIN_COMMON`, `LIBBITCOIN_UTIL`, `LIBUNIVALUE`, `LIBBITCOIN_ZMQ`, `LIBBITCOIN_CONSENSUS`, `LIBBITCOIN_CRYPTO`, `LIBMEMENV`, `LIBSECP256K1`.

* LIBBITCOIN_WALLET 生成配置
```bash
if ENABLE_WALLET
LIBBITCOIN_WALLET=libbitcoin_wallet.a
LIBBITCOIN_WALLET_TOOL=libbitcoin_wallet_tool.a
endif


# wallet: shared between bitcoind and bitcoin-qt, but only linked
# when wallet enabled
libbitcoin_wallet_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES) $(BDB_CPPFLAGS) $(SQLITE_CFLAGS)
libbitcoin_wallet_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_wallet_a_SOURCES = \
  wallet/coincontrol.cpp \
  wallet/context.cpp \
  wallet/crypter.cpp \
  wallet/db.cpp \
  wallet/dump.cpp \
  wallet/external_signer_scriptpubkeyman.cpp \
  wallet/feebumper.cpp \
  wallet/fees.cpp \
  wallet/interfaces.cpp \
  wallet/load.cpp \
  wallet/receive.cpp \
  wallet/rpc/addresses.cpp \
  wallet/rpc/backup.cpp \
  wallet/rpc/coins.cpp \
  wallet/rpc/encrypt.cpp \
  wallet/rpc/spend.cpp \
  wallet/rpc/signmessage.cpp \
  wallet/rpc/transactions.cpp \
  wallet/rpc/util.cpp \
  wallet/rpc/wallet.cpp \
  wallet/scriptpubkeyman.cpp \
  wallet/spend.cpp \
  wallet/transaction.cpp \
  wallet/wallet.cpp \
  wallet/walletdb.cpp \
  wallet/walletutil.cpp \
  wallet/coinselection.cpp \
  $(BITCOIN_CORE_H)

if USE_SQLITE
libbitcoin_wallet_a_SOURCES += wallet/sqlite.cpp
endif
if USE_BDB
libbitcoin_wallet_a_SOURCES += wallet/bdb.cpp wallet/salvage.cpp
endif

libbitcoin_wallet_tool_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES)
libbitcoin_wallet_tool_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_wallet_tool_a_SOURCES = \
  wallet/wallettool.cpp \
  $(BITCOIN_CORE_H)
```
上面其实有生成libbitcoin_wallet.a和libbitcoin_wallet_tool.a两个库，后者是被bitcoin-wallet依赖的：

```bash

# bitcoin-wallet binary #
bitcoin_wallet_SOURCES = bitcoin-wallet.cpp
bitcoin_wallet_SOURCES += init/bitcoin-wallet.cpp
bitcoin_wallet_CPPFLAGS = $(bitcoin_bin_cppflags)
bitcoin_wallet_CXXFLAGS = $(bitcoin_bin_cxxflags)
bitcoin_wallet_LDFLAGS = $(bitcoin_bin_ldflags)
bitcoin_wallet_LDADD = \
  $(LIBBITCOIN_WALLET_TOOL) \
  $(LIBBITCOIN_WALLET) \
  $(LIBBITCOIN_COMMON) \
  $(LIBBITCOIN_UTIL) \
  $(LIBUNIVALUE) \
  $(LIBBITCOIN_CONSENSUS) \
  $(LIBBITCOIN_CRYPTO) \
  $(LIBSECP256K1) \
  $(BDB_LIBS) \
  $(SQLITE_LIBS)
```

* LIBBITCOIN_COMMON 生成配置

```bash
LIBBITCOIN_COMMON=libbitcoin_common.a
# common: shared between bitcoind, and bitcoin-qt and non-server tools
libbitcoin_common_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES)
libbitcoin_common_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_common_a_SOURCES = \
  base58.cpp \
  bech32.cpp \
  chainparams.cpp \
  coins.cpp \
  common/bloom.cpp \
  compressor.cpp \
  core_read.cpp \
  core_write.cpp \
  deploymentinfo.cpp \
  external_signer.cpp \
  init/common.cpp \
  key.cpp \
  key_io.cpp \
  merkleblock.cpp \
  net_types.cpp \
  netaddress.cpp \
  netbase.cpp \
  net_permissions.cpp \
  outputtype.cpp \
  policy/feerate.cpp \
  policy/policy.cpp \
  protocol.cpp \
  psbt.cpp \
  rpc/rawtransaction_util.cpp \
  rpc/external_signer.cpp \
  rpc/util.cpp \
  scheduler.cpp \
  script/descriptor.cpp \
  script/miniscript.cpp \
  script/sign.cpp \
  script/signingprovider.cpp \
  script/standard.cpp \
  warnings.cpp \
  $(BITCOIN_CORE_H)
```

* LIBBITCOIN_UTIL 生成配置

```bash
LIBBITCOIN_UTIL=libbitcoin_util.a

# util: shared between all executables.
libbitcoin_util_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES)
libbitcoin_util_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_util_a_SOURCES = \
  support/lockedpool.cpp \
  chainparamsbase.cpp \
  clientversion.cpp \
  fs.cpp \
  interfaces/echo.cpp \
  interfaces/handler.cpp \
  interfaces/init.cpp \
  logging.cpp \
  random.cpp \
  randomenv.cpp \
  rpc/request.cpp \
  support/cleanse.cpp \
  sync.cpp \
  threadinterrupt.cpp \
  util/asmap.cpp \
  util/bip32.cpp \
  util/bytevectorhash.cpp \
  util/check.cpp \
  util/error.cpp \
  util/fees.cpp \
  util/getuniquepath.cpp \
  util/hasher.cpp \
  util/sock.cpp \
  util/syserror.cpp \
  util/system.cpp \
  util/message.cpp \
  util/moneystr.cpp \
  util/rbf.cpp \
  util/readwritefile.cpp \
  util/settings.cpp \
  util/thread.cpp \
  util/threadnames.cpp \
  util/serfloat.cpp \
  util/spanparsing.cpp \
  util/strencodings.cpp \
  util/string.cpp \
  util/syscall_sandbox.cpp \
  util/time.cpp \
  util/tokenpipe.cpp \
  $(BITCOIN_CORE_H)

if USE_LIBEVENT
libbitcoin_util_a_SOURCES += util/url.cpp
endif
```
* `LIBUNIVALUE`
见 src/makefile.univalue.include

* `LIBBITCOIN_ZMQ`

```bash
if ENABLE_ZMQ
libbitcoin_zmq_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES) $(ZMQ_CFLAGS)
libbitcoin_zmq_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_zmq_a_SOURCES = \
  zmq/zmqabstractnotifier.cpp \
  zmq/zmqnotificationinterface.cpp \
  zmq/zmqpublishnotifier.cpp \
  zmq/zmqrpc.cpp \
  zmq/zmqutil.cpp
endif
```

* `LIBBITCOIN_CONSENSUS`

```bash

# consensus: shared between all executables that validate any consensus rules.
libbitcoin_consensus_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES)
libbitcoin_consensus_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_consensus_a_SOURCES = \
  arith_uint256.cpp \
  arith_uint256.h \
  consensus/amount.h \
  consensus/merkle.cpp \
  consensus/merkle.h \
  consensus/params.h \
  consensus/tx_check.cpp \
  consensus/validation.h \
  hash.cpp \
  hash.h \
  prevector.h \
  primitives/block.cpp \
  primitives/block.h \
  primitives/transaction.cpp \
  primitives/transaction.h \
  pubkey.cpp \
  pubkey.h \
  script/bitcoinconsensus.cpp \
  script/interpreter.cpp \
  script/interpreter.h \
  script/script.cpp \
  script/script.h \
  script/script_error.cpp \
  script/script_error.h \
  serialize.h \
  span.h \
  tinyformat.h \
  uint256.cpp \
  uint256.h \
  util/strencodings.cpp \
  util/strencodings.h \
  version.h
```

* `LIBBITCOIN_CRYPTO`,`LIBSECP256K1`

```bash

# crypto primitives library
crypto_libbitcoin_crypto_base_la_CPPFLAGS = $(AM_CPPFLAGS)

# Specify -static in both CXXFLAGS and LDFLAGS so libtool will only build a
# static version of this library. We don't need a dynamic version, and a dynamic
# version can't be used on windows anyway because the library doesn't currently
# export DLL symbols.
crypto_libbitcoin_crypto_base_la_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS) -static
crypto_libbitcoin_crypto_base_la_LDFLAGS = $(AM_LDFLAGS) -static

crypto_libbitcoin_crypto_base_la_SOURCES = \
  crypto/aes.cpp \
  crypto/aes.h \
  crypto/chacha_poly_aead.h \
  crypto/chacha_poly_aead.cpp \
  crypto/chacha20.h \
  crypto/chacha20.cpp \
  crypto/common.h \
  crypto/hkdf_sha256_32.cpp \
  crypto/hkdf_sha256_32.h \
  crypto/hmac_sha256.cpp \
  crypto/hmac_sha256.h \
  crypto/hmac_sha512.cpp \
  crypto/hmac_sha512.h \
  crypto/poly1305.h \
  crypto/poly1305.cpp \
  crypto/muhash.h \
  crypto/muhash.cpp \
  crypto/ripemd160.cpp \
  crypto/ripemd160.h \
  crypto/sha1.cpp \
  crypto/sha1.h \
  crypto/sha256.cpp \
  crypto/sha256.h \
  crypto/sha3.cpp \
  crypto/sha3.h \
  crypto/sha512.cpp \
  crypto/sha512.h \
  crypto/siphash.cpp \
  crypto/siphash.h

if USE_ASM
crypto_libbitcoin_crypto_base_la_SOURCES += crypto/sha256_sse4.cpp
endif

# See explanation for -static in crypto_libbitcoin_crypto_base_la's LDFLAGS and
# CXXFLAGS above
crypto_libbitcoin_crypto_sse41_la_LDFLAGS = $(AM_LDFLAGS) -static
crypto_libbitcoin_crypto_sse41_la_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS) -static
crypto_libbitcoin_crypto_sse41_la_CPPFLAGS = $(AM_CPPFLAGS)
crypto_libbitcoin_crypto_sse41_la_CXXFLAGS += $(SSE41_CXXFLAGS)
crypto_libbitcoin_crypto_sse41_la_CPPFLAGS += -DENABLE_SSE41
crypto_libbitcoin_crypto_sse41_la_SOURCES = crypto/sha256_sse41.cpp

# See explanation for -static in crypto_libbitcoin_crypto_base_la's LDFLAGS and
# CXXFLAGS above
crypto_libbitcoin_crypto_avx2_la_LDFLAGS = $(AM_LDFLAGS) -static
crypto_libbitcoin_crypto_avx2_la_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS) -static
crypto_libbitcoin_crypto_avx2_la_CPPFLAGS = $(AM_CPPFLAGS)
crypto_libbitcoin_crypto_avx2_la_CXXFLAGS += $(AVX2_CXXFLAGS)
crypto_libbitcoin_crypto_avx2_la_CPPFLAGS += -DENABLE_AVX2
crypto_libbitcoin_crypto_avx2_la_SOURCES = crypto/sha256_avx2.cpp

# See explanation for -static in crypto_libbitcoin_crypto_base_la's LDFLAGS and
# CXXFLAGS above
crypto_libbitcoin_crypto_x86_shani_la_LDFLAGS = $(AM_LDFLAGS) -static
crypto_libbitcoin_crypto_x86_shani_la_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS) -static
crypto_libbitcoin_crypto_x86_shani_la_CPPFLAGS = $(AM_CPPFLAGS)
crypto_libbitcoin_crypto_x86_shani_la_CXXFLAGS += $(X86_SHANI_CXXFLAGS)
crypto_libbitcoin_crypto_x86_shani_la_CPPFLAGS += -DENABLE_X86_SHANI
crypto_libbitcoin_crypto_x86_shani_la_SOURCES = crypto/sha256_x86_shani.cpp

# See explanation for -static in crypto_libbitcoin_crypto_base_la's LDFLAGS and
# CXXFLAGS above
crypto_libbitcoin_crypto_arm_shani_la_LDFLAGS = $(AM_LDFLAGS) -static
crypto_libbitcoin_crypto_arm_shani_la_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS) -static
crypto_libbitcoin_crypto_arm_shani_la_CPPFLAGS = $(AM_CPPFLAGS)
crypto_libbitcoin_crypto_arm_shani_la_CXXFLAGS += $(ARM_SHANI_CXXFLAGS)
crypto_libbitcoin_crypto_arm_shani_la_CPPFLAGS += -DENABLE_ARM_SHANI
crypto_libbitcoin_crypto_arm_shani_la_SOURCES = crypto/sha256_arm_shani.cpp

LIBBITCOIN_CRYPTO = $(LIBBITCOIN_CRYPTO_BASE)
if ENABLE_SSE41
LIBBITCOIN_CRYPTO_SSE41 = crypto/libbitcoin_crypto_sse41.la
LIBBITCOIN_CRYPTO += $(LIBBITCOIN_CRYPTO_SSE41)
endif
if ENABLE_AVX2
LIBBITCOIN_CRYPTO_AVX2 = crypto/libbitcoin_crypto_avx2.la
LIBBITCOIN_CRYPTO += $(LIBBITCOIN_CRYPTO_AVX2)
endif
if ENABLE_X86_SHANI
LIBBITCOIN_CRYPTO_X86_SHANI = crypto/libbitcoin_crypto_x86_shani.la
LIBBITCOIN_CRYPTO += $(LIBBITCOIN_CRYPTO_X86_SHANI)
endif
if ENABLE_ARM_SHANI
LIBBITCOIN_CRYPTO_ARM_SHANI = crypto/libbitcoin_crypto_arm_shani.la
LIBBITCOIN_CRYPTO += $(LIBBITCOIN_CRYPTO_ARM_SHANI)
endif
noinst_LTLIBRARIES += $(LIBBITCOIN_CRYPTO)

$(LIBSECP256K1): $(wildcard secp256k1/src/*.h) $(wildcard secp256k1/src/*.c) $(wildcard secp256k1/include/*)
	$(AM_V_at)$(MAKE) $(AM_MAKEFLAGS) -C $(@D) $(@F)
```
* `LIBMEMENV`

见 Makefile.leveldb.include

##### bitcoin-cli

```
if BUILD_BITCOIN_CLI
  bin_PROGRAMS += bitcoin-cli
endif


# cli: shared between bitcoin-cli and bitcoin-qt
libbitcoin_cli_a_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES)
libbitcoin_cli_a_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
libbitcoin_cli_a_SOURCES = \
  compat/stdin.h \
  compat/stdin.cpp \
  rpc/client.cpp \
  $(BITCOIN_CORE_H)


# bitcoin-cli binary #
bitcoin_cli_SOURCES = bitcoin-cli.cpp
bitcoin_cli_CPPFLAGS = $(AM_CPPFLAGS) $(BITCOIN_INCLUDES) $(EVENT_CFLAGS)
bitcoin_cli_CXXFLAGS = $(AM_CXXFLAGS) $(PIE_FLAGS)
bitcoin_cli_LDFLAGS = $(RELDFLAGS) $(AM_LDFLAGS) $(LIBTOOL_APP_LDFLAGS) $(PTHREAD_FLAGS)

if TARGET_WINDOWS
bitcoin_cli_SOURCES += bitcoin-cli-res.rc
endif

bitcoin_cli_LDADD = \
  $(LIBBITCOIN_CLI) \
  $(LIBUNIVALUE) \
  $(LIBBITCOIN_UTIL) \
  $(LIBBITCOIN_CRYPTO)

bitcoin_cli_LDADD += $(EVENT_LIBS)
```