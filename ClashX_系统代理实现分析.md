# ClashX 系统代理实现深度分析

## 概述

ClashX 是一个基于 Clash 内核的 macOS 代理客户端，其系统代理功能采用了复杂的多层架构设计。本文档详细分析其系统代理的实现原理、技术架构和核心代码逻辑。

## 整体架构

ClashX 的系统代理实现采用了**客户端-特权守护进程**的架构模式：

```
┌─────────────────┐    XPC通信    ┌──────────────────────┐
│   ClashX.app    │ ←─────────→  │ ProxyConfigHelper    │
│  (主应用程序)    │              │    (特权守护进程)     │
└─────────────────┘              └──────────────────────┘
         │                                    │
         │                                    │
         ▼                                    ▼
┌─────────────────┐              ┌──────────────────────┐
│ SystemProxyMgr  │              │ SystemConfiguration │
│ (系统代理管理器) │              │    Framework         │
└─────────────────┘              └──────────────────────┘
```

## 核心组件分析

### 1. ProxyConfigHelper (特权守护进程)

**位置**: `ProxyConfigHelper/`

这是整个系统代理功能的核心组件，以 macOS 守护进程的形式运行，具有管理员权限。

#### 主要文件:
- `ProxyConfigHelper.m` - 守护进程主逻辑
- `ProxySettingTool.m` - 代理设置工具类
- `ProxyConfigRemoteProcessProtocol.h` - XPC通信协议定义

#### 核心功能:
```objective-c
// 启用代理
- (void)enableProxyWithPort:(int)port
                  socksPort:(int)socksPort
                        pac:(NSString *)pac
            filterInterface:(BOOL)filterInterface
                 ignoreList:(NSArray<NSString *>*)ignoreList
                      error:(stringReplyBlock)reply;

// 禁用代理
- (void)disableProxyWithFilterInterface:(BOOL)filterInterface
                                  reply:(stringReplyBlock)reply;

// 恢复代理设置
- (void)restoreProxyWithCurrentPort:(int)port
                          socksPort:(int)socksPort
                               info:(NSDictionary *)dict
                    filterInterface:(BOOL)filterInterface
                              error:(stringReplyBlock)reply;
```

### 2. SystemProxyManager (系统代理管理器)

**位置**: `ClashX/General/Managers/SystemProxyManager.swift`

这是主应用程序中的代理管理组件，负责协调代理操作和与守护进程通信。

#### 核心方法:
```swift
class SystemProxyManager {
    static let shared = SystemProxyManager()
    
    // 启用代理
    func enableProxy(port: Int, socksPort: Int) {
        helper?.enableProxy(withPort: Int32(port),
                           socksPort: Int32(socksPort),
                           pac: nil,
                           filterInterface: Settings.filterInterface,
                           ignoreList: Settings.proxyIgnoreList) { error in
            // 错误处理
        }
    }
    
    // 禁用代理
    func disableProxy(forceDisable: Bool = false, complete: (() -> Void)? = nil)
    
    // 保存当前代理设置
    func saveProxy()
}
```

### 3. PrivilegedHelperManager (特权助手管理器)

**位置**: `ClashX/General/Managers/PrivilegedHelperManager.swift`

负责管理特权守护进程的安装、通信和生命周期。

#### 主要功能:
- 检查守护进程是否已安装
- 安装和更新守护进程
- 建立 XPC 连接
- 处理权限授权

## 技术实现详解

### 1. 权限管理

ClashX 使用 macOS 的 **Authorization Services** 框架来获取修改系统代理所需的管理员权限：

```objective-c
// 创建授权引用
- (void)localAuth {
    OSStatus myStatus;
    AuthorizationFlags myFlags = [self authFlags];
    myStatus = AuthorizationCreate(NULL, kAuthorizationEmptyEnvironment, myFlags, &_authRef);
    
    AuthorizationItem myItems = {kAuthorizationRightExecute, 0, NULL, 0};
    AuthorizationRights myRights = {1, &myItems};
    myStatus = AuthorizationCopyRights(self.authRef, &myRights, NULL, myFlags, NULL);
}
```

### 2. 系统配置修改

核心使用 **SystemConfiguration.framework** 来修改网络接口的代理设置：

```objective-c
- (void)applySCNetworkSettingWithRef:(void(^)(SCPreferencesRef))callback {
    // 创建系统配置引用
    SCPreferencesRef ref = SCPreferencesCreateWithAuthorization(
        nil, 
        CFSTR("com.west2online.ClashX.ProxyConfigHelper.config"), 
        nil, 
        self.authRef
    );
    
    callback(ref);
    
    // 提交并应用更改
    SCPreferencesCommitChanges(ref);
    SCPreferencesApplyChanges(ref);
    SCPreferencesSynchronize(ref);
    CFRelease(ref);
}
```

### 3. 代理设置配置

系统为每个网络接口设置代理参数：

```objective-c
- (NSDictionary *)getProxySetting:(BOOL)enable port:(int)port socksPort:(int)socksPort pac:(NSString *)pac ignoreList:(NSArray<NSString *>*)ignoreList {
    NSMutableDictionary *proxySettings = [NSMutableDictionary dictionary];
    
    NSString *ip = enable ? @"127.0.0.1" : @"";
    NSInteger enableInt = enable ? 1 : 0;
    
    // HTTP 代理设置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPProxy] = ip;
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPEnable] = @(enableInt);
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPPort] = @(port);
    
    // HTTPS 代理设置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPSProxy] = ip;
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPSEnable] = @(enableInt);
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPSPort] = @(port);
    
    // SOCKS 代理设置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesSOCKSProxy] = ip;
    proxySettings[(__bridge NSString *)kCFNetworkProxiesSOCKSEnable] = @(enableInt);
    proxySettings[(__bridge NSString *)kCFNetworkProxiesSOCKSPort] = @(socksPort);
    
    // 代理例外列表
    proxySettings[(__bridge NSString *)kCFNetworkProxiesExceptionsList] = ignoreList;
    
    return proxySettings;
}
```

### 4. XPC 进程间通信

主应用程序与守护进程通过 **XPC (Cross-Process Communication)** 进行通信：

```swift
func helper() -> ProxyConfigRemoteProcessProtocol? {
    connection = NSXPCConnection(
        machServiceName: PrivilegedHelperManager.machServiceName,
        options: NSXPCConnection.Options.privileged
    )
    
    connection?.remoteObjectInterface = NSXPCInterface(with: ProxyConfigRemoteProcessProtocol.self)
    connection?.resume()
    
    return connection?.remoteObjectProxyWithErrorHandler { error in
        Logger.log("Helper connection error: \(error)")
    } as? ProxyConfigRemoteProcessProtocol
}
```

## 工作流程

### 1. 启用代理流程

```
1. 用户点击"设为系统代理" → SystemProxyManager.enableProxy()
2. 获取当前 HTTP 和 SOCKS 端口
3. 通过 XPC 调用 ProxyConfigHelper.enableProxy()
4. 守护进程获取网络服务列表
5. 过滤目标网络接口 (WiFi, Ethernet, AirPort)
6. 为每个接口设置代理配置
7. 提交系统配置更改
```

### 2. 禁用代理流程

```
1. 用户取消系统代理 → SystemProxyManager.disableProxy()
2. 检查是否需要恢复原始设置
3. 如果需要恢复：调用 restoreProxy() 恢复之前保存的设置
4. 如果强制禁用：调用 disableProxy() 清空所有代理设置
5. 提交系统配置更改
```

### 3. 网络接口过滤

代理设置只应用于特定类型的网络接口：

```objective-c
+ (void)getDiviceListWithPrefRef:(SCPreferencesRef)ref
                 filterInterface:(BOOL)filterInterface
                         devices:(void(^)(NSString *, NSDictionary *))callback {
    NSDictionary *sets = (__bridge NSDictionary *)SCPreferencesGetValue(ref, kSCPrefNetworkServices);
    
    for (NSString *key in [sets allKeys]) {
        NSMutableDictionary *dict = [sets objectForKey:key];
        NSString *hardware = [dict valueForKeyPath:@"Interface.Hardware"];
        
        if (!filterInterface || 
            [hardware isEqualToString:@"AirPort"] ||
            [hardware isEqualToString:@"Wi-Fi"] ||
            [hardware isEqualToString:@"Ethernet"]) {
            callback(key, dict);
        }
    }
}
```

## 安全机制

### 1. 代码签名验证
守护进程安装时进行代码签名验证，确保只有合法的 ClashX 应用可以安装和控制守护进程。

### 2. 权限最小化
守护进程只具有修改网络代理设置的最小必要权限。

### 3. 进程隔离
主应用程序和守护进程运行在不同的进程空间中，通过 XPC 进行安全通信。

## 代理例外列表

默认的代理例外列表包括：
```objective-c
NSArray *ignoreList = @[
    @"192.168.0.0/16",    // 私有网络
    @"10.0.0.0/8",        // 私有网络
    @"172.16.0.0/12",     // 私有网络
    @"127.0.0.1",         // 本地回环
    @"localhost",         // 本地主机
    @"*.local",           // 本地域名
    @"*.crashlytics.com"  // 崩溃报告服务
];
```

## 守护进程管理

### 1. 安装过程
使用 `SMJobBless` API 安装守护进程到 `/Library/PrivilegedHelperTools/`

### 2. 版本检查
每次启动时检查守护进程版本，如果不匹配则自动更新

### 3. 自动启动
守护进程通过 launchd 管理，系统启动时自动运行

## 总结

ClashX 的系统代理实现展现了 macOS 应用程序权限分离的最佳实践：

1. **安全性**: 通过特权守护进程隔离权限操作
2. **稳定性**: 使用标准的 SystemConfiguration 框架
3. **用户体验**: 自动管理守护进程生命周期
4. **兼容性**: 支持多种网络接口和代理协议

这种设计确保了系统代理功能的安全性和可靠性，同时为用户提供了透明的使用体验。

## 相关文件位置

- **守护进程**: `ProxyConfigHelper/`
- **系统代理管理**: `ClashX/General/Managers/SystemProxyManager.swift`
- **特权助手管理**: `ClashX/General/Managers/PrivilegedHelperManager.swift`
- **XPC 协议定义**: `ProxyConfigHelper/ProxyConfigRemoteProcessProtocol.h`
- **核心代理工具**: `ProxyConfigHelper/ProxySettingTool.m`