---
title: 比特币源码学习02-初始化
published: true
---

文中引用的代码基于`24.x`分支。

`src/bitcoind.cpp`文件包含程序的入口(line 259 - line 279)：


```c++
#ifdef WIN32
// Export main() and ensure working ASLR when using mingw-w64.
// Exporting a symbol will prevent the linker from stripping
// the .reloc section from the binary, which is a requirement
// for ASLR. While release builds are not affected, anyone
// building with a binutils < 2.36 is subject to this ld bug.
#define MAIN_FUNCTION __declspec(dllexport) int main(int argc, char* argv[])
#else
#define MAIN_FUNCTION int main(int argc, char* argv[])
#endif

MAIN_FUNCTION
{
#ifdef WIN32
    // windows 下字符串特殊处理，utf16->utf8
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

可以看到, `AppInit`函数是程序的主体(line 112)，这个函数在解析完命令行参数，读取配置文件之后，分成了12个步骤：


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

这里除了创建命令行参数外，还创建不同的了`CChainParam`(main, test, reg)。

* 解析命令行参数

```c++
    std::string error;
    if (!args.ParseParameters(argc, argv, error)) {
        return InitError(Untranslated(strprintf("Error parsing command line arguments: %s\n", error)));
    }
```
把命令行参数读入`ArgsManager`中的`m_settings.command_line_options`和`m_command`成员变量中。

* 处理`help`和`version`命令

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

## Step 2: 参数交互

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
```

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
```c++
    if (!fs::is_directory(gArgs.GetBlocksDirPath())) {
        return InitError(strprintf(_("Specified blocks directory \"%s\" does not exist."), args.GetArg("-blocksdir", "")));
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

* 脚本验证缓存大小计算

```c++
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
脚本验证，是由全局变量`scriptcheckqueue`启动的(validation.cpp, line 1886)：

```c++
static CCheckQueue<CScriptCheck> scriptcheckqueue(128);
```
`CScriptCheck` 是脚本验证的一个闭包表示；`CCheckQueue`是验证队列，储存等待进行的验证。

* `CCheckQueue`

`CCheckQueue`是一个模版类，模版参数需要提供一个operator()操作，返回bool，用来表示验证是否成功，具体验证逻辑后续文章再介绍。下面介绍`CCheckQueue`的实现逻辑。

通常，有一个master线程会往CCheckQueue的队列中批量推送验证，另外N-1个worker线程负责处理验证。当master线程结束推送后，会join worker pool。

### 成员变量

| Name            | type         | Description |
|-----------------|--------------|-------------|
| m_mutex         |  Mutex       |  内部变量锁 |
| m_worker_cv | std::condition_variable | 当没有任务时，worker线程会等待这个变量 |
| m_master_cv | std::condition_variable | 当没有任务时，master线程会等待这个变量 |
| queue | std::vector<T> | 任务队列（后进先出）|
| nIdle | int | 空闲的线程数（包括master）|
| nTotal | int | 线程总数（包括master）|
| fAllOk | bool | 截止到当前，验证是否都通过 |
| nTodo | int | 剩余验证任务数 |
| nBatchSize | unsigned int | 批量处理最大的大小，默认为128 |
| m_worker_threads | std::vector<std::thread> | worker线程池 |
| m_request_stop | bool | 停止处理flag |

### 成员函数

#### `StartWorkerThreads` 创建worker线程池

```c++
    void StartWorkerThreads(const int threads_num) EXCLUSIVE_LOCKS_REQUIRED(!m_mutex)
    {
        {
            LOCK(m_mutex);
            nIdle = 0;
            nTotal = 0;
            fAllOk = true;
        }
        assert(m_worker_threads.empty());
        for (int n = 0; n < threads_num; ++n) {
            m_worker_threads.emplace_back([this, n]() {
                // 线程名称
                util::ThreadRename(strprintf("scriptch.%i", n));
                // 线程安全性设置
                SetSyscallSandboxPolicy(SyscallSandboxPolicy::VALIDATION_SCRIPT_CHECK);
                // 进入循环
                Loop(false /* worker thread */);
            });
        }
    }
```
worker线程创建后进入Loop循环，具体解释见代码：

```c++
    /** Internal function that does bulk of the verification work. */
    bool Loop(bool fMaster) EXCLUSIVE_LOCKS_REQUIRED(!m_mutex)
    {
        // 判断是master thread还是workerthread，选择对应的条件变量
        std::condition_variable& cond = fMaster ? m_master_cv : m_worker_cv;
        // 验证数组，从全局队列中拷贝过来
        std::vector<T> vChecks;
        vChecks.reserve(nBatchSize);
        unsigned int nNow = 0;
        bool fOk = true;
        do { // 无限循环
            {
                // 分配验证任务时需锁住内部变量锁：m_mutex
                WAIT_LOCK(m_mutex, lock);
                // first do the clean-up of the previous loop run (allowing us to do it in the same critsect)
                if (nNow) { // nNow表示上一个loop处理的任务数
                    fAllOk &= fOk; // 更新全局的状态变量
                    nTodo -= nNow; // 更新当前loop的剩余验证数
                    if (nTodo == 0 && !fMaster)
                        // We processed the last element; inform the master it can exit and return the result
                        m_master_cv.notify_one();
                } else {
                    // first iteration
                    // 说明这是当前线程的第一个loop循环
                    nTotal++;
                }
                // logically, the do loop starts here
                while (queue.empty() && !m_request_stop) {
                    if (fMaster && nTodo == 0) {
                        // master任务结束
                        nTotal--;
                        bool fRet = fAllOk;
                        // reset the status for new work later
                        fAllOk = true;
                        // return the current status
                        return fRet;
                    }
                    nIdle++;
                    cond.wait(lock); // wait
                    nIdle--;
                }
                if (m_request_stop) {
                    return false;
                }

                // Decide how many work units to process now.
                // * Do not try to do everything at once, but aim for increasingly smaller batches so
                //   all workers finish approximately simultaneously.
                // * Try to account for idle jobs which will instantly start helping.
                // * Don't do batches smaller than 1 (duh), or larger than nBatchSize.
                nNow = std::max(1U, std::min(nBatchSize, (unsigned int)queue.size() / (nTotal + nIdle + 1)));
                vChecks.resize(nNow);
                for (unsigned int i = 0; i < nNow; i++) {
                    // We want the lock on the m_mutex to be as short as possible, so swap jobs from the global
                    // queue to the local batch vector instead of copying.
                    vChecks[i].swap(queue.back());
                    queue.pop_back();
                }
                // Check whether we need to do work at all
                fOk = fAllOk;
            }
            // execute work
            for (T& check : vChecks)
                if (fOk)
                    fOk = check();
            vChecks.clear();
        } while (true);
    }
```

#### master线程涉及到`Wait`、`Add`以及`StopWorkThreads`这几个操作，用来等待任务处理结束、添加任务以及终止worker线程。

```c++
//! Wait until execution finishes, and return whether all evaluations were successful.
    bool Wait() EXCLUSIVE_LOCKS_REQUIRED(!m_mutex)
    {
        return Loop(true /* master thread */);
    }

    //! Add a batch of checks to the queue
    void Add(std::vector<T>& vChecks) EXCLUSIVE_LOCKS_REQUIRED(!m_mutex)
    {
        if (vChecks.empty()) {
            return;
        }

        {
            LOCK(m_mutex);
            for (T& check : vChecks) {
                queue.emplace_back();
                check.swap(queue.back());
            }
            nTodo += vChecks.size();
        }

        if (vChecks.size() == 1) {
            m_worker_cv.notify_one();
        } else {
            m_worker_cv.notify_all();
        }
    }

    //! Stop all of the worker threads.
    void StopWorkerThreads() EXCLUSIVE_LOCKS_REQUIRED(!m_mutex)
    {
        WITH_LOCK(m_mutex, m_request_stop = true);
        m_worker_cv.notify_all();
        for (std::thread& t : m_worker_threads) {
            t.join();
        }
        m_worker_threads.clear();
        WITH_LOCK(m_mutex, m_request_stop = false);
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
这里创建了一个service线程(m_service_thread)用来执行定时或是周期性的任务。并且注册了一个周期性的任务：每分钟给随机系统获取一些熵。

* 创建钱包界面（并没有加载钱包数据）
```c++
    // Create client interfaces for wallets that are supposed to be loaded
    // according to -wallet and -disablewallet options. This only constructs
    // the interfaces, it doesn't load wallet data. Wallets actually get loaded
    // when load() and start() interface methods are called below.
    g_wallet_init_interface.Construct(node);
    uiInterface.InitWallet();
```
钱包界面创建的代码位于`src/walet/init.cpp`。最终的代码位于`src/wallet/interfaces.cpp`第524行的`WalletLoaderImpl`实现。

* warmup模式启动json-rpc服务

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
JSON-RPC是一个轻量级的远程函数调用（协议见https://www.jsonrpc.org/specification_v1）。两个节点建立连接后，可以通过该协议进行函数调用请求和回复。消息传递的协议可以是http。

#### JSON-RPC实现
`RegisterAllCoreRPCCommands`往`CRPCTable`中注册RPC命令，`CRPCTable`管理这所有的命令，并有一个成员函数可以执行命令：

```c++
UniValue CRPCTable::execute(const JSONRPCRequest &request) const
{
    // Return immediately if in warmup
    {
        LOCK(g_rpc_warmup_mutex);
        if (fRPCInWarmup)
            throw JSONRPCError(RPC_IN_WARMUP, rpcWarmupStatus);
    }

    // Find method
    auto it = mapCommands.find(request.strMethod);
    if (it != mapCommands.end()) {
        UniValue result;
        if (ExecuteCommands(it->second, request, result)) {
            return result;
        }
    }
    throw JSONRPCError(RPC_METHOD_NOT_FOUND, "Method not found");
}
```
可以看到，执行命令是调用了`ExecuteCommands`函数：

```c++
static bool ExecuteCommand(const CRPCCommand& command, const JSONRPCRequest& request, UniValue& result, bool last_handler)
{
    try
    {
        RPCCommandExecution execution(request.strMethod);
        // Execute, convert arguments to array if necessary
        if (request.params.isObject()) {
            return command.actor(transformNamedArguments(request, command.argNames), result, last_handler);
        } else {
            return command.actor(request, result, last_handler);
        }
    }
    catch (const std::exception& e)
    {
        throw JSONRPCError(RPC_MISC_ERROR, e.what());
    }
}
```

而在`ExecuteCommand`中则是调用了`CRPCCommand`类中的`actor`，在注册命令的时候，需要将`actor`进行定义。下面是`CRPCCommand`的申明：

```c++
typedef RPCHelpMan (*RpcMethodFnType)();

class CRPCCommand
{
public:
    //! RPC method handler reading request and assigning result. Should return
    //! true if request is fully handled, false if it should be passed on to
    //! subsequent handlers.
    using Actor = std::function<bool(const JSONRPCRequest& request, UniValue& result, bool last_handler)>;

    //! Constructor taking Actor callback supporting multiple handlers.
    CRPCCommand(std::string category, std::string name, Actor actor, std::vector<std::string> args, intptr_t unique_id)
        : category(std::move(category)), name(std::move(name)), actor(std::move(actor)), argNames(std::move(args)),
          unique_id(unique_id)
    {
    }

    //! Simplified constructor taking plain RpcMethodFnType function pointer.
    CRPCCommand(std::string category, RpcMethodFnType fn)
        : CRPCCommand(
              category,
              fn().m_name,
              [fn](const JSONRPCRequest& request, UniValue& result, bool) { result = fn().HandleRequest(request); return true; },
              fn().GetArgNames(),
              intptr_t(fn))
    {
    }

    std::string category;
    std::string name;
    Actor actor;
    std::vector<std::string> argNames;
    intptr_t unique_id;
};
```
可以看出，`Actor`其实调用的是`RPCHelpMan`类的`HandleRequest`函数。也就是说，创建`RPCCommand`的时候只需要指定`category`以及`RPCHelpMan`（`RpcMethodFnType`）的实例。

#### HTPP Server

函数`AppInitServers`初始化并启动http服务：

```c++
static bool AppInitServers(NodeContext& node)
{
    const ArgsManager& args = *Assert(node.args);
    //  服务启动事件
    RPCServer::OnStarted(&OnRPCStarted);
    // 服务关闭事件
    RPCServer::OnStopped(&OnRPCStopped);
    // 初始化http服务
    if (!InitHTTPServer())
        return false;
    // RPC启动flag
    StartRPC();
    node.rpc_interruption_point = RpcInterruptionPoint;
    // 注册RPC handle到http 服务中
    if (!StartHTTPRPC(&node))
        return false;
    if (args.GetBoolArg("-rest", DEFAULT_REST_ENABLE)) StartREST(&node); // 连接 Rest handle
    // 启动http服务
    StartHTTPServer();
    return true;
}
```
bitcoind使用的是`libevent`作为事件分发处理以及http服务器。`InitHTTPServer`主要是进行`libevent`的设置(日志、多线程）以及创建相应的数据结构（event_base, evthttp），最后绑定ip地址和端口。

`StartRPC`函数标记RPC服务已启动，并触发启动事件。

`StartHTTPRPC`函数主要是注册RPC路由:

```c++
bool StartHTTPRPC(const std::any& context)
{
    LogPrint(BCLog::RPC, "Starting HTTP RPC server\n");
    if (!InitRPCAuthentication())
        return false;

    auto handle_rpc = [context](HTTPRequest* req, const std::string&) { return HTTPReq_JSONRPC(context, req); };
    RegisterHTTPHandler("/", true, handle_rpc);
    if (g_wallet_init_interface.HasWalletSupport()) {
        RegisterHTTPHandler("/wallet/", false, handle_rpc);
    }
    struct event_base* eventBase = EventBase();
    assert(eventBase);
    httpRPCTimerInterface = std::make_unique<HTTPRPCTimerInterface>(eventBase);
    RPCSetTimerInterface(httpRPCTimerInterface.get());
    return true;
}
```

`StartREST`函数注册Rest api路由，注意到REST接口只有指定了-rest参数才有。

`StartHTTPServer`函数正式启动http服务器。注意到，这个函数启动了一个`g_thread_http`线程作为`libevent`事件分发主线程，另外启动了`rpcThreads`个线程（`g_thread_http_workers`）作为http worker线程来响应http请求。之所以能做到多线程处理，是因为把请求都放到了一个`HTTPWorkQueue`中，每次work queue中添加一个工作项的时候，会通过条件变量通知其中一个线程，这个线程则会等待这个条件变量的通知，一旦有工作项，就会把该项出队列，并执行该项的工作：

```c++
/** HTTP request work item */
class HTTPWorkItem final : public HTTPClosure
{
public:
    HTTPWorkItem(std::unique_ptr<HTTPRequest> _req, const std::string &_path, const HTTPRequestHandler& _func):
        req(std::move(_req)), path(_path), func(_func)
    {
    }
    void operator()() override
    {
        func(req.get(), path);
    }

    std::unique_ptr<HTTPRequest> req;

private:
    std::string path;
    HTTPRequestHandler func;
};

/** Simple work queue for distributing work over multiple threads.
 * Work items are simply callable objects.
 */
template <typename WorkItem>
class WorkQueue
{
private:
    Mutex cs;
    std::condition_variable cond GUARDED_BY(cs);
    std::deque<std::unique_ptr<WorkItem>> queue GUARDED_BY(cs);
    bool running GUARDED_BY(cs){true};
    const size_t maxDepth;

public:
    explicit WorkQueue(size_t _maxDepth) : maxDepth(_maxDepth)
    {
    }
    /** Precondition: worker threads have all stopped (they have been joined).
     */
    ~WorkQueue() = default;
    /** Enqueue a work item */
    bool Enqueue(WorkItem* item) EXCLUSIVE_LOCKS_REQUIRED(!cs)
    {
        LOCK(cs);
        if (!running || queue.size() >= maxDepth) {
            return false;
        }
        queue.emplace_back(std::unique_ptr<WorkItem>(item));
        cond.notify_one();
        return true;
    }
    /** Thread function */
    void Run() EXCLUSIVE_LOCKS_REQUIRED(!cs)
    {
        while (true) {
            std::unique_ptr<WorkItem> i;
            {
                WAIT_LOCK(cs, lock);
                while (running && queue.empty())
                    cond.wait(lock);
                if (!running && queue.empty())
                    break;
                i = std::move(queue.front());
                queue.pop_front();
            }
            (*i)();
        }
    }
    /** Interrupt and exit loops */
    void Interrupt() EXCLUSIVE_LOCKS_REQUIRED(!cs)
    {
        LOCK(cs);
        running = false;
        cond.notify_all();
    }
};
```

## Step 5: 验证钱包数据库的完整性

```c++
    for (const auto& client : node.chain_clients) {
        if (!client->verify()) {
            return false;
        }
    }
```
这一步主要是做钱包数据库的完整性检查。SQLite is required for the descriptor wallet；Berkeley DB is required for the legacy wallet.。

## Step 6: 网络初始化

这一步只是做初始化的工作，并没有打开网络连接，因为区块链的状态还没有初始化。

* 解析asmap文件，如果需要

asmap的解释见：

https://blog.bitmex.com/call-to-action-testing-and-improving-asmap/

* 初始化网络组管理`NetGroupManager`

```c++
node.netgroupman = std::make_unique<NetGroupManager>(std::move(asmap));
```

* 初始化地址管理 `AddrMan`

```c++
std::optional<bilingual_str> LoadAddrman(const NetGroupManager& netgroupman, const ArgsManager& args, std::unique_ptr<AddrMan>& addrman)
{
    auto check_addrman = std::clamp<int32_t>(args.GetIntArg("-checkaddrman", DEFAULT_ADDRMAN_CONSISTENCY_CHECKS), 0, 1000000);
    addrman = std::make_unique<AddrMan>(netgroupman, /*deterministic=*/false, /*consistency_check_ratio=*/check_addrman);

    int64_t nStart = GetTimeMillis();
    const auto path_addr{args.GetDataDirNet() / "peers.dat"};
    try {
        DeserializeFileDB(path_addr, *addrman, CLIENT_VERSION);
        LogPrintf("Loaded %i addresses from peers.dat  %dms\n", addrman->size(), GetTimeMillis() - nStart);
    } catch (const DbNotFoundError&) {
        // Addrman can be in an inconsistent state after failure, reset it
        addrman = std::make_unique<AddrMan>(netgroupman, /*deterministic=*/false, /*consistency_check_ratio=*/check_addrman);
        LogPrintf("Creating peers.dat because the file was not found (%s)\n", fs::quoted(fs::PathToString(path_addr)));
        DumpPeerAddresses(args, *addrman);
    } catch (const InvalidAddrManVersionError&) {
        if (!RenameOver(path_addr, (fs::path)path_addr + ".bak")) {
            addrman = nullptr;
            return strprintf(_("Failed to rename invalid peers.dat file. Please move or delete it and try again."));
        }
        // Addrman can be in an inconsistent state after failure, reset it
        addrman = std::make_unique<AddrMan>(netgroupman, /*deterministic=*/false, /*consistency_check_ratio=*/check_addrman);
        LogPrintf("Creating new peers.dat because the file version was not compatible (%s). Original backed up to peers.dat.bak\n", fs::quoted(fs::PathToString(path_addr)));
        DumpPeerAddresses(args, *addrman);
    } catch (const std::exception& e) {
        addrman = nullptr;
        return strprintf(_("Invalid or corrupt peers.dat (%s). If you believe this is a bug, please report it to %s. As a workaround, you can move the file (%s) out of the way (rename, move, or delete) to have a new one created on the next start."),
                         e.what(), PACKAGE_BUGREPORT, fs::quoted(fs::PathToString(path_addr)));
    }
    return std::nullopt;
}
```
`AddrMan`是一个随机的地址管理类，管理着与节点相关的peers，随机的设置是为了防止网络攻击，下面是`src/addrman.h`中关于这个类的设计说明：

```c++
/** Stochastic address manager
 *
 * Design goals:
 *  * Keep the address tables in-memory, and asynchronously dump the entire table to peers.dat.
 *  * Make sure no (localized) attacker can fill the entire table with his nodes/addresses.
 *
 * To that end:
 *  * Addresses are organized into buckets that can each store up to 64 entries.
 *    * Addresses to which our node has not successfully connected go into 1024 "new" buckets.
 *      * Based on the address range (/16 for IPv4) of the source of information, or if an asmap is provided,
 *        the AS it belongs to (for IPv4/IPv6), 64 buckets are selected at random.
 *      * The actual bucket is chosen from one of these, based on the range in which the address itself is located.
 *      * The position in the bucket is chosen based on the full address.
 *      * One single address can occur in up to 8 different buckets to increase selection chances for addresses that
 *        are seen frequently. The chance for increasing this multiplicity decreases exponentially.
 *      * When adding a new address to an occupied position of a bucket, it will not replace the existing entry
 *        unless that address is also stored in another bucket or it doesn't meet one of several quality criteria
 *        (see IsTerrible for exact criteria).
 *    * Addresses of nodes that are known to be accessible go into 256 "tried" buckets.
 *      * Each address range selects at random 8 of these buckets.
 *      * The actual bucket is chosen from one of these, based on the full address.
 *      * When adding a new good address to an occupied position of a bucket, a FEELER connection to the
 *        old address is attempted. The old entry is only replaced and moved back to the "new" buckets if this
 *        attempt was unsuccessful.
 *    * Bucket selection is based on cryptographic hashing, using a randomly-generated 256-bit key, which should not
 *      be observable by adversaries.
 *    * Several indexes are kept for high performance. Setting m_consistency_check_ratio with the -checkaddrman
 *      configuration option will introduce (expensive) consistency checks for the entire data structure.
 */
class AddrMan
{
protected:
    const std::unique_ptr<AddrManImpl> m_impl;

public:
    explicit AddrMan(const NetGroupManager& netgroupman, bool deterministic, int32_t consistency_check_ratio);

    ~AddrMan();

    template <typename Stream>
    void Serialize(Stream& s_) const;

    template <typename Stream>
    void Unserialize(Stream& s_);

    //! Return the number of (unique) addresses in all tables.
    size_t size() const;

    /**
     * Attempt to add one or more addresses to addrman's new table.
     *
     * @param[in] vAddr           Address records to attempt to add.
     * @param[in] source          The address of the node that sent us these addr records.
     * @param[in] time_penalty    A "time penalty" to apply to the address record's nTime. If a peer
     *                            sends us an address record with nTime=n, then we'll add it to our
     *                            addrman with nTime=(n - time_penalty).
     * @return    true if at least one address is successfully added. */
    bool Add(const std::vector<CAddress>& vAddr, const CNetAddr& source, std::chrono::seconds time_penalty = 0s);

    /**
     * Mark an address record as accessible and attempt to move it to addrman's tried table.
     *
     * @param[in] addr            Address record to attempt to move to tried table.
     * @param[in] time            The time that we were last connected to this peer.
     * @return    true if the address is successfully moved from the new table to the tried table.
     */
    bool Good(const CService& addr, NodeSeconds time = Now<NodeSeconds>());

    //! Mark an entry as connection attempted to.
    void Attempt(const CService& addr, bool fCountFailure, NodeSeconds time = Now<NodeSeconds>());

    //! See if any to-be-evicted tried table entries have been tested and if so resolve the collisions.
    void ResolveCollisions();

    /**
     * Randomly select an address in the tried table that another address is
     * attempting to evict.
     *
     * @return CAddress The record for the selected tried peer.
     *         seconds  The last time we attempted to connect to that peer.
     */
    std::pair<CAddress, NodeSeconds> SelectTriedCollision();

    /**
     * Choose an address to connect to.
     *
     * @param[in] newOnly  Whether to only select addresses from the new table.
     * @return    CAddress The record for the selected peer.
     *            seconds  The last time we attempted to connect to that peer.
     */
    std::pair<CAddress, NodeSeconds> Select(bool newOnly = false) const;

    /**
     * Return all or many randomly selected addresses, optionally by network.
     *
     * @param[in] max_addresses  Maximum number of addresses to return (0 = all).
     * @param[in] max_pct        Maximum percentage of addresses to return (0 = all).
     * @param[in] network        Select only addresses of this network (nullopt = all).
     *
     * @return                   A vector of randomly selected addresses from vRandom.
     */
    std::vector<CAddress> GetAddr(size_t max_addresses, size_t max_pct, std::optional<Network> network) const;

    /** We have successfully connected to this peer. Calling this function
     *  updates the CAddress's nTime, which is used in our IsTerrible()
     *  decisions and gossiped to peers. Callers should be careful that updating
     *  this information doesn't leak topology information to network spies.
     *
     *  net_processing calls this function when it *disconnects* from a peer to
     *  not leak information about currently connected peers.
     *
     * @param[in]   addr     The address of the peer we were connected to
     * @param[in]   time     The time that we were last connected to this peer
     */
    void Connected(const CService& addr, NodeSeconds time = Now<NodeSeconds>());

    //! Update an entry's service bits.
    void SetServices(const CService& addr, ServiceFlags nServices);

    /** Test-only function
     * Find the address record in AddrMan and return information about its
     * position.
     * @param[in] addr       The address record to look up.
     * @return               Information about the address record in AddrMan
     *                       or nullopt if address is not found.
     */
    std::optional<AddressPosition> FindAddressEntry(const CAddress& addr);
};

```

* 初始化屏蔽地址管理`BanMan`

```c++
node.banman = std::make_unique<BanMan>(gArgs.GetDataDirNet() / "banlist", &uiInterface, args.GetIntArg("-bantime", DEFAULT_MISBEHAVING_BANTIME));
```

设计策略见`src/banman.h`:

```c++
// Banman manages two related but distinct concepts:
//
// 1. Banning. This is configured manually by the user, through the setban RPC.
// If an address or subnet is banned, we never accept incoming connections from
// it and never create outgoing connections to it. We won't gossip its address
// to other peers in addr messages. Banned addresses and subnets are stored to
// disk on shutdown and reloaded on startup. Banning can be used to
// prevent connections with spy nodes or other griefers.
//
// 2. Discouragement. If a peer misbehaves enough (see Misbehaving() in
// net_processing.cpp), we'll mark that address as discouraged. We still allow
// incoming connections from them, but they're preferred for eviction when
// we receive new incoming connections. We never make outgoing connections to
// them, and do not gossip their address to other peers. This is implemented as
// a bloom filter. We can (probabilistically) test for membership, but can't
// list all discouraged addresses or unmark them as discouraged. Discouragement
// can prevent our limited connection slots being used up by incompatible
// or broken peers.
//
// Neither banning nor discouragement are protections against denial-of-service
// attacks, since if an attacker has a way to waste our resources and we
// disconnect from them and ban that address, it's trivial for them to
// reconnect from another IP address.
//
// Attempting to automatically disconnect or ban any class of peer carries the
// risk of splitting the network. For example, if we banned/disconnected for a
// transaction that fails a policy check and a future version changes the
// policy check so the transaction is accepted, then that transaction could
// cause the network to split between old nodes and new nodes.
```
* 初始化连接管理`CConnman`
`CConman`是最顶层的网络管理类，接受`Addrman`, `NetGroupMan`作为构造参数。

* 初始化fee估计类`CBlockPolicyEstimator`
`CBlockPolicyEstimator`的设计见：`src/policy/fees.h`:

```c++
/** \class CBlockPolicyEstimator
 * The BlockPolicyEstimator is used for estimating the feerate needed
 * for a transaction to be included in a block within a certain number of
 * blocks.
 *
 * At a high level the algorithm works by grouping transactions into buckets
 * based on having similar feerates and then tracking how long it
 * takes transactions in the various buckets to be mined.  It operates under
 * the assumption that in general transactions of higher feerate will be
 * included in blocks before transactions of lower feerate.   So for
 * example if you wanted to know what feerate you should put on a transaction to
 * be included in a block within the next 5 blocks, you would start by looking
 * at the bucket with the highest feerate transactions and verifying that a
 * sufficiently high percentage of them were confirmed within 5 blocks and
 * then you would look at the next highest feerate bucket, and so on, stopping at
 * the last bucket to pass the test.   The average feerate of transactions in this
 * bucket will give you an indication of the lowest feerate you can put on a
 * transaction and still have a sufficiently high chance of being confirmed
 * within your desired 5 blocks.
 *
 * Here is a brief description of the implementation:
 * When a transaction enters the mempool, we track the height of the block chain
 * at entry.  All further calculations are conducted only on this set of "seen"
 * transactions. Whenever a block comes in, we count the number of transactions
 * in each bucket and the total amount of feerate paid in each bucket. Then we
 * calculate how many blocks Y it took each transaction to be mined.  We convert
 * from a number of blocks to a number of periods Y' each encompassing "scale"
 * blocks.  This is tracked in 3 different data sets each up to a maximum
 * number of periods. Within each data set we have an array of counters in each
 * feerate bucket and we increment all the counters from Y' up to max periods
 * representing that a tx was successfully confirmed in less than or equal to
 * that many periods. We want to save a history of this information, so at any
 * time we have a counter of the total number of transactions that happened in a
 * given feerate bucket and the total number that were confirmed in each of the
 * periods or less for any bucket.  We save this history by keeping an
 * exponentially decaying moving average of each one of these stats.  This is
 * done for a different decay in each of the 3 data sets to keep relevant data
 * from different time horizons.  Furthermore we also keep track of the number
 * unmined (in mempool or left mempool without being included in a block)
 * transactions in each bucket and for how many blocks they have been
 * outstanding and use both of these numbers to increase the number of transactions
 * we've seen in that feerate bucket when calculating an estimate for any number
 * of confirmations below the number of blocks they've been outstanding.
 *
 *  We want to be able to estimate feerates that are needed on tx's to be included in
 * a certain number of blocks.  Every time a block is added to the best chain, this class records
 * stats on the transactions included in that block
 */
```

* sanitize comments per BIP-0014, format user agent and check total size

* 设置reachable net和dns seed
* 设置代理
* 设置手动指定的外部ip

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