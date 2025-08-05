# ClashX 系统代理实现核心代码示例

## 1. 主应用程序启用代理的完整调用链

### 用户点击"设为系统代理"时的代码执行路径：

#### AppDelegate.swift 中的事件处理
```swift
// 用户点击设为系统代理菜单项
@IBAction func setSystemProxy(_ sender: NSMenuItem) {
    let isEnabled = sender.state == .off
    ConfigManager.shared.proxyPortAutoSet = isEnabled
    
    if isEnabled {
        // 保存当前系统代理设置
        SystemProxyManager.shared.saveProxy()
        // 启用 ClashX 代理
        SystemProxyManager.shared.enableProxy()
    } else {
        // 恢复原始代理设置
        SystemProxyManager.shared.disableProxy()
    }
}
```

#### SystemProxyManager.swift 中的代理管理
```swift
func enableProxy() {
    // 获取 Clash 配置的端口
    let port = ConfigManager.shared.currentConfig?.usedHttpPort ?? 0
    let socketPort = ConfigManager.shared.currentConfig?.usedSocksPort ?? 0
    enableProxy(port: port, socksPort: socketPort)
}

func enableProxy(port: Int, socksPort: Int) {
    guard port > 0 && socksPort > 0 else {
        Logger.log("enableProxy fail: \(port) \(socksPort)", level: .error)
        return
    }
    
    // 检查是否应该暂停（如 SSID 黑名单）
    if SSIDSuspendTool.shared.shouldSuspend() {
        Logger.log("not enableProxy due to ssid in disabled list", level: .info)
        return
    }
    
    Logger.log("enableProxy", level: .debug)
    
    // 通过 XPC 调用守护进程
    helper?.enableProxy(withPort: Int32(port),
                        socksPort: Int32(socksPort),
                        pac: nil,
                        filterInterface: Settings.filterInterface,
                        ignoreList: Settings.proxyIgnoreList,
                        error: { error in
                            if let error = error {
                                Logger.log("enableProxy \(error)", level: .error)
                            }
                        })
}

// 获取守护进程实例
private var helper: ProxyConfigRemoteProcessProtocol? {
    PrivilegedHelperManager.shared.helper()
}
```

#### PrivilegedHelperManager.swift 中的 XPC 连接
```swift
func helper(failture: (() -> Void)? = nil) -> ProxyConfigRemoteProcessProtocol? {
    // 创建到守护进程的 XPC 连接
    connection = NSXPCConnection(
        machServiceName: PrivilegedHelperManager.machServiceName,
        options: NSXPCConnection.Options.privileged
    )
    
    // 设置远程对象接口
    connection?.remoteObjectInterface = NSXPCInterface(with: ProxyConfigRemoteProcessProtocol.self)
    
    // 设置连接失效处理器
    connection?.invalidationHandler = {
        Logger.log("XPC Connection Invalidated")
    }
    
    // 启动连接
    connection?.resume()
    
    // 获取远程代理对象
    guard let helper = connection?.remoteObjectProxyWithErrorHandler({ error in
        Logger.log("Helper connection was closed with error: \(error)")
        failture?()
    }) as? ProxyConfigRemoteProcessProtocol else { return nil }
    
    return helper
}
```

## 2. 守护进程中的代理配置实现

### ProxyConfigHelper.m 中的 XPC 服务实现
```objective-c
// XPC 协议实现 - 启用代理
- (void)enableProxyWithPort:(int)port
                  socksPort:(int)socksPort
                        pac:(NSString *)pac
            filterInterface:(BOOL)filterInterface
                 ignoreList:(NSArray<NSString *>*)ignoreList
                      error:(stringReplyBlock)reply {
    
    dispatch_async(dispatch_get_main_queue(), ^{
        // 创建代理设置工具
        ProxySettingTool *tool = [ProxySettingTool new];
        
        // 执行代理配置
        [tool enableProxyWithport:port 
                        socksPort:socksPort 
                           pacUrl:pac 
                  filterInterface:filterInterface 
                       ignoreList:ignoreList];
        
        // 回调成功
        reply(nil);
    });
}
```

### ProxySettingTool.m 中的核心实现
```objective-c
- (void)enableProxyWithport:(int)port 
                  socksPort:(int)socksPort
                     pacUrl:(NSString *)pacUrl
            filterInterface:(BOOL)filterInterface
                 ignoreList:(NSArray<NSString *>*)ignoreList {
    
    // 使用系统配置框架应用设置
    [self applySCNetworkSettingWithRef:^(SCPreferencesRef ref) {
        // 获取所有网络设备并过滤
        [ProxySettingTool getDiviceListWithPrefRef:ref 
                                   filterInterface:filterInterface 
                                           devices:^(NSString *key, NSDictionary *dict) {
            // 为每个网络接口启用代理
            [self enableProxySettings:ref 
                            interface:key 
                                 port:port 
                            socksPort:socksPort 
                           ignoreList:ignoreList 
                                  pac:pacUrl];
        }];
    }];
}

// 应用系统配置更改
- (void)applySCNetworkSettingWithRef:(void(^)(SCPreferencesRef))callback {
    // 创建带授权的系统配置引用
    SCPreferencesRef ref = SCPreferencesCreateWithAuthorization(
        nil, 
        CFSTR("com.west2online.ClashX.ProxyConfigHelper.config"), 
        nil, 
        self.authRef
    );
    
    if (!ref) {
        return;
    }
    
    // 执行回调中的配置操作
    callback(ref);
    
    // 提交更改
    SCPreferencesCommitChanges(ref);
    SCPreferencesApplyChanges(ref);
    SCPreferencesSynchronize(ref);
    CFRelease(ref);
}

// 生成代理设置字典
- (NSDictionary *)getProxySetting:(BOOL)enable 
                             port:(int)port
                        socksPort:(int)socksPort 
                              pac:(NSString *)pac
                       ignoreList:(NSArray<NSString *>*)ignoreList {
    
    NSMutableDictionary *proxySettings = [NSMutableDictionary dictionary];
    
    NSString *ip = enable ? @"127.0.0.1" : @"";
    NSInteger enableInt = enable ? 1 : 0;
    NSInteger enablePac = [pac length] > 0;
    
    // HTTP 代理配置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPProxy] = ip;
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPEnable] = @(enableInt);
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPPort] = enable ? @(port) : nil;
    
    // HTTPS 代理配置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPSProxy] = ip;
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPSEnable] = @(enableInt);
    proxySettings[(__bridge NSString *)kCFNetworkProxiesHTTPSPort] = enable ? @(port) : nil;
    
    // SOCKS 代理配置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesSOCKSProxy] = ip;
    proxySettings[(__bridge NSString *)kCFNetworkProxiesSOCKSEnable] = @(enableInt);
    proxySettings[(__bridge NSString *)kCFNetworkProxiesSOCKSPort] = enable ? @(socksPort) : nil;
    
    // PAC 自动配置
    proxySettings[(__bridge NSString *)kCFNetworkProxiesProxyAutoConfigEnable] = @(enablePac);
    if (enablePac) {
        proxySettings[(__bridge NSString *)kCFNetworkProxiesProxyAutoConfigURLString] = pac;
    }
    
    // 代理例外列表
    if (enable) {
        proxySettings[(__bridge NSString *)kCFNetworkProxiesExceptionsList] = ignoreList;
        proxySettings[(__bridge NSString *)kCFNetworkProxiesExcludeSimpleHostnames] = @(YES);
    } else {
        proxySettings[(__bridge NSString *)kCFNetworkProxiesExceptionsList] = @[];
    }
    
    return proxySettings;
}

// 为特定网络接口设置代理
- (void)enableProxySettings:(SCPreferencesRef)prefs
                  interface:(NSString *)interfaceKey
                       port:(int)port
                  socksPort:(int)socksPort
                 ignoreList:(NSArray<NSString *>*)ignoreList
                        pac:(NSString *)pac {
    
    // 生成代理设置
    NSDictionary *proxySettings = [self getProxySetting:YES 
                                                    port:port 
                                               socksPort:socksPort 
                                                     pac:pac 
                                              ignoreList:ignoreList];
    
    // 应用到指定网络接口
    [self setProxyConfig:prefs interface:interfaceKey proxySetting:proxySettings];
}

// 设置网络接口的代理配置
- (void)setProxyConfig:(SCPreferencesRef)prefs
             interface:(NSString *)interfaceKey
          proxySetting:(NSDictionary *)proxySettings {
    
    // 构建配置路径
    NSString *path = [NSString stringWithFormat:@"/%@/%@/%@",
                      (NSString *)kSCPrefNetworkServices,
                      interfaceKey,
                      (NSString *)kSCEntNetProxies];
    
    // 设置配置值
    SCPreferencesPathSetValue(prefs,
                              (__bridge CFStringRef)path,
                              (__bridge CFDictionaryRef)proxySettings);
}

// 获取并过滤网络设备列表
+ (void)getDiviceListWithPrefRef:(SCPreferencesRef)ref
                 filterInterface:(BOOL)filterInterface
                         devices:(void(^)(NSString *, NSDictionary *))callback {
    
    // 获取所有网络服务
    NSDictionary *sets = (__bridge NSDictionary *)SCPreferencesGetValue(ref, kSCPrefNetworkServices);
    
    for (NSString *key in [sets allKeys]) {
        NSMutableDictionary *dict = [sets objectForKey:key];
        NSString *hardware = [dict valueForKeyPath:@"Interface.Hardware"];
        
        // 过滤目标网络接口类型
        if (!filterInterface || 
            [hardware isEqualToString:@"AirPort"] ||
            [hardware isEqualToString:@"Wi-Fi"] ||
            [hardware isEqualToString:@"Ethernet"]) {
            callback(key, dict);
        }
    }
}
```

## 3. 权限管理和守护进程安装

### 权限授权实现
```objective-c
- (void)localAuth {
    OSStatus myStatus;
    AuthorizationFlags myFlags = [self authFlags];
    
    // 创建授权引用
    myStatus = AuthorizationCreate(NULL, kAuthorizationEmptyEnvironment, myFlags, &_authRef);
    
    if (myStatus != errAuthorizationSuccess) {
        return;
    }
    
    // 定义权限项
    AuthorizationItem myItems = {kAuthorizationRightExecute, 0, NULL, 0};
    AuthorizationRights myRights = {1, &myItems};
    
    // 复制权限
    myStatus = AuthorizationCopyRights(self.authRef, &myRights, NULL, myFlags, NULL);
}

- (AuthorizationFlags)authFlags {
    AuthorizationFlags authFlags = kAuthorizationFlagDefaults
    | kAuthorizationFlagExtendRights
    | kAuthorizationFlagInteractionAllowed
    | kAuthorizationFlagPreAuthorize;
    return authFlags;
}
```

### 守护进程安装
```swift
private func installHelperDaemon() -> DaemonInstallResult {
    Logger.log("installHelperDaemon", level: .info)
    
    defer {
        resetConnection()
    }
    
    // 创建授权引用
    var authRef: AuthorizationRef?
    var authStatus = AuthorizationCreate(nil, nil, [], &authRef)
    
    guard authStatus == errAuthorizationSuccess else {
        Logger.log("Authorization failed: \(authStatus)", level: .error)
        return .authorizationFail
    }
    
    // 请求管理员权限
    var authItem = AuthorizationItem(
        name: (kSMRightBlessPrivilegedHelper as NSString).utf8String!,
        valueLength: 0,
        value: nil,
        flags: 0
    )
    var authRights = withUnsafeMutablePointer(to: &authItem) { pointer in
        AuthorizationRights(count: 1, items: pointer)
    }
    
    let flags: AuthorizationFlags = [[], .interactionAllowed, .extendRights, .preAuthorize]
    authStatus = AuthorizationCreate(&authRights, nil, flags, &authRef)
    
    defer {
        if let ref = authRef {
            AuthorizationFree(ref, [])
        }
    }
    
    guard authStatus == errAuthorizationSuccess else {
        Logger.log("Couldn't obtain admin privileges: \(authStatus)", level: .error)
        return .getAdminFail
    }
    
    // 使用 SMJobBless 安装守护进程
    var error: Unmanaged<CFError>?
    if SMJobBless(kSMDomainSystemLaunchd, 
                  PrivilegedHelperManager.machServiceName as CFString, 
                  authRef, 
                  &error) == false {
        let blessError = error!.takeRetainedValue() as Error
        Logger.log("Bless Error: \(blessError)", level: .error)
        return .blessError((blessError as NSError).code)
    }
    
    Logger.log("\(PrivilegedHelperManager.machServiceName) installed successfully", level: .info)
    return .success
}
```

## 4. 实际使用示例

### 设置代理的完整调用示例
```swift
// 1. 用户启用系统代理
ConfigManager.shared.proxyPortAutoSet = true

// 2. 保存当前系统代理设置（用于后续恢复）
SystemProxyManager.shared.saveProxy()

// 3. 启用 ClashX 代理
SystemProxyManager.shared.enableProxy(port: 7890, socksPort: 7891)

// 实际执行的系统调用路径：
// Swift: SystemProxyManager.enableProxy()
//   ↓ XPC 调用
// ObjC: ProxyConfigHelper.enableProxyWithPort()
//   ↓ 系统框架调用
// SystemConfiguration: SCPreferencesPathSetValue()
//   ↓ 系统生效
// macOS: 网络代理设置更新
```

### 禁用代理的完整调用示例
```swift
// 1. 用户禁用系统代理
ConfigManager.shared.proxyPortAutoSet = false

// 2. 恢复原始代理设置
SystemProxyManager.shared.disableProxy()

// 如果设置了恢复模式，将恢复保存的原始设置
// 如果设置了强制禁用，将清空所有代理设置
```

这些代码示例展示了 ClashX 如何通过多层架构安全地管理 macOS 系统代理设置，从用户界面交互到最终的系统配置修改的完整流程。