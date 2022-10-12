---
title: 比特币源码学习02-初始化
published: true
---


`bitcoind.cpp`文件包含程序的入口：

```
// 自定义的宏，避免mingw-w64上的链接问题
MAIN_FUNCTION
{
#ifdef WIN32
    // windows 下特殊处理参数，转为utf16->utf8
    util::WinCmdLineArgs winArgs;
    std::tie(argc, argv) = winArgs.get();
#endif

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


## Step 1： 操作系统相关的设置

相关代码位于 `AppInitBasicSetup` 函数内。

## Step 2: 参数校验

有些参数的设置依赖另一些参数，这个步骤主要是设置这些参数。相关代码位于`AppInitParameterInteraction`内。

## Step 3: 为某些参数设置全局变量

相关代码位于`AppInitParameterInteraction`内。

## Step 4/4a: 应用程序初始化

#### 环境检查
```c++
    if (auto error = kernel::SanityChecks(kernel)) {
        InitError(*error);
        return InitError(strprintf(_("Initialization sanity check failed. %s is shutting down."), PACKAGE_NAME));
    }
```

#### 锁住数据目录
```c++
    // Probe the data directory lock to give an early error message, if possible
    // We cannot hold the data directory lock here, as the forking for daemon() hasn't yet happened,
    // and a fork will cause weird behavior to it.
    return LockDataDirectory(true);
```

#### 启动脚本验证线程

```c++
    if (script_threads >= 1) {
        g_parallel_script_checks = true;
        StartScriptCheckWorkerThreads(script_threads);
    }
```

#### 启动轻量级的任务管理线程

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

#### 创建钱包界面（并没有加载钱包数据）
```c++
    // Create client interfaces for wallets that are supposed to be loaded
    // according to -wallet and -disablewallet options. This only constructs
    // the interfaces, it doesn't load wallet data. Wallets actually get loaded
    // when load() and start() interface methods are called below.
    g_wallet_init_interface.Construct(node);
    uiInterface.InitWallet();
```

#### warmup模式启动rpc服务

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
#### 初始化网络组管理类
#### 解析黑名单
#### 初始化连接管理类
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