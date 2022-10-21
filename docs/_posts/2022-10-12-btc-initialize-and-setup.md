---
title: 比特币源码学习02-初始化
published: true
---

文中引用的代码基于v23.0版本。

`bitcoind.cpp`文件包含程序的入口：

```c++
// 自定义的宏，避免mingw-w64上的链接问题
MAIN_FUNCTION
{
#ifdef WIN32
    // windows 下特殊处理参数，转为utf16->utf8
    util::WinCmdLineArgs winArgs;
    std::tie(argc, argv) = winArgs.get();
#endif

    // 节点上下文，包含各类全局变量
    NodeContext node;
    int exit_status;
    std::unique_ptr<interfaces::Init> init = interfaces::MakeNodeInit(node, argc, argv, exit_status);
    if (!init) {
        return exit_status;
    }

    // 设置默认locale
    SetupEnvironment();

    // 使用boost signal2库链接信号处理函数，主要是打印消息
    noui_connect();

    // 主循环
    return (AppInit(node, argc, argv) ? EXIT_SUCCESS : EXIT_FAILURE);
}
```

可以看到, `AppInit`函数是程序的主体，这个函数在解析完命令行参数，读取配置文件之后，分成了12个步骤：


## Step 1： 基本设置

* 设置当前线程名称

```c++
util::ThreadSetInternalName("init");
```
* 设置服务器参数

```c++
ArgsManager& args = *Assert(node.args);
SetupServerArgs(args);
```

这里除了创建命令行参数外，还创建不同的了`CChainParam`（main, test,reg）。

* 解析命令行参数

```c++
    std::string error;
    if (!args.ParseParameters(argc, argv, error)) {
        return InitError(Untranslated(strprintf("Error parsing command line arguments: %s\n", error)));
    }
```
把命令行参数读入ArgsManager中的m_settings.command_line_optionsz和m_command成员变量中中。

* 处理help和version命令

略

* 检查数据目录参数

```c++
        if (!CheckDataDirOption()) {
            return InitError(Untranslated(strprintf("Specified data directory \"%s\" does not exist.\n", args.GetArg("-datadir", ""))));
        }
```

* 读取配置文件(bitcoin.conf)

```c++
        if (!args.ReadConfigFiles(error, true)) {
            return InitError(Untranslated(strprintf("Error reading configuration file: %s\n", error)));
        }
```

* 设置全局的ChainParams: `globalChainParams`

```c++
        // Check for chain settings (Params() calls are only valid after this clause)
        try {
            SelectParams(args.GetChainName());
        } catch (const std::exception& e) {
            return InitError(Untranslated(strprintf("%s\n", e.what())));
        }
```
* 检查参数中的`-`字符

* 初始化动态参数配置文件

```c++
        if (!args.InitSettings(error)) {
            InitError(Untranslated(error));
            return false;
        }
```

* 强制设置 -server 为 true

```c++
        // -server defaults to true for bitcoind but not for the GUI so do this here
        args.SoftSetBoolArg("-server", true);
```

* 基于规则的参数校验

```c++
InitParameterInteraction(args);
```

1. 如果显示指定bind地址，需要进行监听: -bind set -> setting -listen=1
2. -whitebind set -> setting -listen=1
3. 如果显示指定connect节点则不需要dns lookup：-connect set -> setting -dnsseed=0
4. 如果显示指定connect节点则不需要监听: -connect set -> setting -listen=0
5. parameter interaction: -proxy set -> setting -listen=0
6. parameter interaction: -proxy set -> setting -upnp=0
7. parameter interaction: -proxy set -> setting -natpmp=0
8. parameter interaction: -proxy set -> setting -discover=0
9. parameter interaction: -listen=0 -> setting -upnp=0
10. parameter interaction: -listen=0 -> setting -natpmp=0
11. parameter interaction: -listen=0 -> setting -discover=0
12. parameter interaction: -listen=0 -> setting -listenonion=0
13. parameter interaction: -listen=0 -> setting -i2pacceptincoming=0
14. parameter interaction: -externalip set -> setting -discover=0
15. parameter interaction: -blocksonly=1 -> setting -whitelistrelay=0
16. parameter interaction: -whitelistforcerelay=1 -> setting -whitelistrelay=1\
17. dsn seed只在连接网络是ipv4/ipv6下才有用: -onlynet excludes IPv4 and IPv6 -> setting -dnsseed=0

* 操作系统相关设置

```c++
        if (!AppInitBasicSetup(args)) {
            // InitError will have been called with detailed error, which ends up on console
            return false;
        }
```

## Step 2: 参数校验

有些参数的设置依赖另一些参数，这个步骤主要是设置这些参数（注意到Step 1中也有部分的参数校验）。相关代码位于`AppInitParameterInteraction`函数内。

1. 检查network相关的参数(-addnode, -connect等)是否在配置文件的合适位置（相应network section下）

```c++
    for (const auto& arg : args.GetUnsuitableSectionOnlyArgs()) {
        errors += strprintf(_("Config setting for %s only applied on %s network when in [%s] section.") + Untranslated("\n"), arg, network, network);
    }
```
2. 检查配置文件中是否存在不认识的section

```c++
    for (const auto& section : args.GetUnrecognizedSections()) {
        warnings += strprintf(Untranslated("%s:%i ") + _("Section [%s] is not recognized.") + Untranslated("\n"), section.m_file, section.m_line, section.m_name);
    }
```
3. 检查block directory是否是目录

```c++
    if (!fs::is_directory(gArgs.GetBlocksDirPath())) {
        return InitError(strprintf(_("Specified blocks directory \"%s\" does not exist."), args.GetArg("-blocksdir", "")));
    }
```

4. 设置block 过滤器的索引类型（0-basic索引；1-全部索引，实际上只有basic类型)

5. 设置节点是否可以响应block filter请求

```c++
    // Signal NODE_COMPACT_FILTERS if peerblockfilters and basic filters index are both enabled.
    if (args.GetBoolArg("-peerblockfilters", DEFAULT_PEERBLOCKFILTERS)) {
        if (g_enabled_filter_types.count(BlockFilterType::BASIC) != 1) {
            return InitError(_("Cannot set -peerblockfilters without -blockfilterindex."));
        }

        nLocalServices = ServiceFlags(nLocalServices | NODE_COMPACT_FILTERS);
    }
```

6. 在删除旧block的情况下(-prune set), 无法建立索引

```c++
    if (args.GetIntArg("-prune", 0)) {
        if (args.GetBoolArg("-txindex", DEFAULT_TXINDEX))
            return InitError(_("Prune mode is incompatible with -txindex."));
        if (args.GetBoolArg("-reindex-chainstate", false)) {
            return InitError(_("Prune mode is incompatible with -reindex-chainstate. Use full -reindex instead."));
        }
    }
```

7. 如果强制使用dns，那么dns seed必须被设置

```c++
    // If -forcednsseed is set to true, ensure -dnsseed has not been set to false
    if (args.GetBoolArg("-forcednsseed", DEFAULT_FORCEDNSSEED) && !args.GetBoolArg("-dnsseed", DEFAULT_DNSSEED)){
        return InitError(_("Cannot set -forcednsseed to true when setting -dnsseed to false."));
    }
···

8. 检查-bind -whitebind 和 -listen是否有冲突

9. 如果-listen=0， 那么 listenonion=1

10. 确保有足够的文件描述符可用

```c++
    // Make sure enough file descriptors are available
    int nBind = std::max(nUserBind, size_t(1));
    nUserMaxConnections = args.GetIntArg("-maxconnections", DEFAULT_MAX_PEER_CONNECTIONS);
    nMaxConnections = std::max(nUserMaxConnections, 0);

    nFD = RaiseFileDescriptorLimit(nMaxConnections + MIN_CORE_FILEDESCRIPTORS + MAX_ADDNODE_CONNECTIONS + nBind + NUM_FDS_MESSAGE_CAPTURE);

#ifdef USE_POLL
    int fd_max = nFD;
#else
    int fd_max = FD_SETSIZE;
#endif
    // Trim requested connection counts, to fit into system limitations
    // <int> in std::min<int>(...) to work around FreeBSD compilation issue described in #2695
    nMaxConnections = std::max(std::min<int>(nMaxConnections, fd_max - nBind - MIN_CORE_FILEDESCRIPTORS - MAX_ADDNODE_CONNECTIONS - NUM_FDS_MESSAGE_CAPTURE), 0);
    if (nFD < MIN_CORE_FILEDESCRIPTORS)
        return InitError(_("Not enough file descriptors available."));
    nMaxConnections = std::min(nFD - MIN_CORE_FILEDESCRIPTORS - MAX_ADDNODE_CONNECTIONS - NUM_FDS_MESSAGE_CAPTURE, nMaxConnections);

    if (nMaxConnections < nUserMaxConnections)
        InitWarning(strprintf(_("Reducing -maxconnections from %d to %d, because of system limitations."), nUserMaxConnections, nMaxConnections));
```

## Step 3: 设置内部全局变量

* `fCheckBlockIndex`: 是否周期性检查区块树、区块链状态等数据结构的一致性
* `fCheckpointsEnabled`: 允许拒绝在某个checkpoint之前的分叉
* `hashAssumeValid`: 这个变量指定一个block hash，验证引擎会假定这个block以及这个block的父block是有效的，不会进行签名验证。

```c++
    hashAssumeValid = uint256S(args.GetArg("-assumevalid", chainparams.GetConsensus().defaultAssumeValid.GetHex()));

```

* `nMinimumChainWork`: 正确链需要的最小工作量

```c++
    if (args.IsArgSet("-minimumchainwork")) {
        const std::string minChainWorkStr = args.GetArg("-minimumchainwork", "");
        if (!IsHexNumber(minChainWorkStr)) {
            return InitError(strprintf(Untranslated("Invalid non-hex (%s) minimum chain work value specified"), minChainWorkStr));
        }
        nMinimumChainWork = UintToArith256(uint256S(minChainWorkStr));
    } else {
        nMinimumChainWork = UintToArith256(chainparams.GetConsensus().nMinimumChainWork);
    }
```

* `fPruneMode`: 是否裁剪区块；`nPruneTarget`：目标区块大小
相关代码位于`AppInitParameterInteraction`内。

* `nConnectTimeout` 连接超时; `peer_connect_timeout`： 节点连接超时
* `nBytesPerSigOp`: Equivalent bytes per sigop in transactions for relay and mining
* 

* Improve TODO: 下面这几个参数需要健全性检查，需要移到单独的函数中检查，最好在[Step 2: 参数校验](AppInitParameterInteraction) 之前？

```c++
    // AppInitMain
    auto opt_max_upload = ParseByteUnits(args.GetArg("-maxuploadtarget", DEFAULT_MAX_UPLOAD_TARGET), ByteUnit::M);
    if (!opt_max_upload) {
        return InitError(strprintf(_("Unable to parse -maxuploadtarget: '%s'"), args.GetArg("-maxuploadtarget", "")));
    }
```
```c++
    // AppInitParameterInteration

    // Sanity check argument for min fee for including tx in block
    // TODO: Harmonize which arguments need sanity checking and where that happens
    if (args.IsArgSet("-blockmintxfee")) {
        if (!ParseMoney(args.GetArg("-blockmintxfee", ""))) {
            return InitError(AmountErrMsg("blockmintxfee", args.GetArg("-blockmintxfee", "")));
        }
    }
```
```c++
    // AppInitMain

    // Warn about relative -datadir path.
    if (args.IsArgSet("-datadir") && !args.GetPathArg("-datadir").is_absolute()) {
        LogPrintf("Warning: relative datadir option '%s' specified, which will be interpreted relative to the " /* Continued */
                  "current working directory '%s'. This is fragile, because if bitcoin is started in the future "
                  "from a different location, it will be unable to locate the current data files. There could "
                  "also be data loss if bitcoin is started while in a temporary directory.\n",
                  args.GetArg("-datadir", ""), fs::PathToString(fs::current_path()));
    }
```
## Step 4/4a: 应用程序初始化

* 环境检查

```c++
    if (auto error = kernel::SanityChecks(kernel)) {
        InitError(*error);
        return InitError(strprintf(_("Initialization sanity check failed. %s is shutting down."), PACKAGE_NAME));
    }
```

* 锁住数据目录

```c++
    // Probe the data directory lock to give an early error message, if possible
    // We cannot hold the data directory lock here, as the forking for daemon() hasn't yet happened,
    // and a fork will cause weird behavior to it.
    return LockDataDirectory(true);
```

在这之后，程序调用`AppInitInterfaces`和`AppInitMain`函数，进入`src/init.cpp`中。接下来的步骤都位于`AppInitMain`函数内。

* 初始化CChainParams

```c++
const CChainParams& chainparams = Params();
```

调用`chainparams.cpp`里的`Params`函数，该函数直接返回全局的`globalChainParams`函数

* 创建pid文件
* 开始logging

* 缓存大小计算

```
    ValidationCacheSizes validation_cache_sizes{};
    ApplyArgsManOptions(args, validation_cache_sizes);
    if (!InitSignatureCache(validation_cache_sizes.signature_cache_bytes)
        || !InitScriptExecutionCache(validation_cache_sizes.script_execution_cache_bytes))
    {
        return InitError(strprintf(_("Unable to allocate memory for -maxsigcachesize: '%s' MiB"), args.GetIntArg("-maxsigcachesize", DEFAULT_MAX_SIG_CACHE_BYTES >> 20)));
    }

```

* 启动脚本验证线程

```c++
 int script_threads = args.GetIntArg("-par", DEFAULT_SCRIPTCHECK_THREADS);
    if (script_threads <= 0) {
        // -par=0 means autodetect (number of cores - 1 script threads)
        // -par=-n means "leave n cores free" (number of cores - n - 1 script threads)
        script_threads += GetNumCores();
    }

    // Subtract 1 because the main thread counts towards the par threads
    script_threads = std::max(script_threads - 1, 0);

    // Number of script-checking threads <= MAX_SCRIPTCHECK_THREADS
    script_threads = std::min(script_threads, MAX_SCRIPTCHECK_THREADS);

    LogPrintf("Script verification uses %d additional threads\n", script_threads);
    if (script_threads >= 1) {
        g_parallel_script_checks = true;
        StartScriptCheckWorkerThreads(script_threads);
    }
```

* 启动轻量级的任务管理线程

```c++
    node.scheduler = std::make_unique<CScheduler>();

    // Start the lightweight task scheduler thread
    node.scheduler->m_service_thread = std::thread(util::TraceThread, "scheduler", [&] { node.scheduler->serviceQueue(); });

    // Gather some entropy once per minute.
    node.scheduler->scheduleEvery([]{
        RandAddPeriodic();
    }, std::chrono::minutes{1});

    GetMainSignals().RegisterBackgroundSignalScheduler(*node.scheduler);
```

* 创建钱包界面（并没有加载钱包数据）
```c++
    // Create client interfaces for wallets that are supposed to be loaded
    // according to -wallet and -disablewallet options. This only constructs
    // the interfaces, it doesn't load wallet data. Wallets actually get loaded
    // when load() and start() interface methods are called below.
    g_wallet_init_interface.Construct(node);
    uiInterface.InitWallet();
```

* warmup模式启动rpc服务

```c++
    /* Register RPC commands regardless of -server setting so they will be
     * available in the GUI RPC console even if external calls are disabled.
     */
    RegisterAllCoreRPCCommands(tableRPC);
    for (const auto& client : node.chain_clients) {
        client->registerRpcs();
    }
#if ENABLE_ZMQ
    RegisterZMQRPCCommands(tableRPC);
#endif

    /* Start the RPC server already.  It will be started in "warmup" mode
     * and not really process calls already (but it will signify connections
     * that the server is there and will be ready later).  Warmup mode will
     * be disabled when initialisation is finished.
     */
    if (args.GetBoolArg("-server", false)) {
        uiInterface.InitMessage_connect(SetRPCWarmupStatus);
        if (!AppInitServers(node))
            return InitError(_("Unable to start HTTP server. See debug log for details."));
    }
```

## Step 5: 验证钱包数据库的完整性

```c++
    for (const auto& client : node.chain_clients) {
        if (!client->verify()) {
            return false;
        }
    }
```

## Step 6: 网络初始化

#### 解析asmap文件，如果需要
https://blog.bitmex.com/call-to-action-testing-and-improving-asmap/

#### 初始化netgroupman, addrman, banman, connman
#### 初始化fee估计类
#### 解析onlynet和dnsseed参数
#### 解析dns参数
#### 解析proxy
#### 解析onion，判断是否使用tor
#### 解析手动指定的外部ip

## Step 7: 加载区块链
该步骤把区块链数据加载到内存中，并初始化UTXO缓存。

#### 计算缓存大小
细节见 `src/node/caches.cpp`内的`CalculateCachesSizes`实现。

#### 加载block数据并验证最后几个block
```c++
        auto [status, error] = catch_exceptions([&]{ return LoadChainstate(chainman, cache_sizes, options); });
        if (status == node::ChainstateLoadStatus::SUCCESS) {
            uiInterface.InitMessage(_("Verifying blocks…").translated);
            if (chainman.m_blockman.m_have_pruned && options.check_blocks > MIN_BLOCKS_TO_KEEP) {
                LogPrintfCategory(BCLog::PRUNE, "pruned datadir may not have more than %d blocks; only checking available blocks\n",
                                  MIN_BLOCKS_TO_KEEP);
            }
            std::tie(status, error) = catch_exceptions([&]{ return VerifyLoadedChainstate(chainman, options);});
            if (status == node::ChainstateLoadStatus::SUCCESS) {
                fLoaded = true;
                LogPrintf(" block index %15dms\n", GetTimeMillis() - load_block_index_start_time);
            }
        }
```
注意到UTXO并没有在此刻被加载到内存中，而是之后从数据库中读取时进行缓存。

## Step 8: 开启索引

## Step 9: 加载钱包

## Step 10: 数据目录维护

如果指定了prune，删除和钱包不相干的区块。

## Step 11: 导入区块

扫描新区块

## Step 12: 启动节点

开启网络组线程

```c++
    if (!node.connman->Start(*node.scheduler, connOptions)) {
        return false;
    }

```
## Step 13 

RPC 服务由warmup恢复为正常

```c++
    // At this point, the RPC is "started", but still in warmup, which means it
    // cannot yet be called. Before we make it callable, we need to make sure
    // that the RPC's view of the best block is valid and consistent with
    // ChainstateManager's active tip.
    //
    // If we do not do this, RPC's view of the best block will be height=0 and
    // hash=0x0. This will lead to erroroneous responses for things like
    // waitforblockheight.
    RPCNotifyBlockChange(WITH_LOCK(chainman.GetMutex(), return chainman.ActiveTip()));
    SetRPCWarmupFinished();
```