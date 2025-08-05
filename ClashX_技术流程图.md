# ClashX 系统代理技术流程图

## 1. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ClashX 主应用程序                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  AppDelegate.swift                                                          │
│  ├─ 监听配置变更                                                              │
│  ├─ 启动时检查代理设置                                                         │
│  └─ 处理用户界面交互                                                          │
│                                                                             │
│  SystemProxyManager.swift                                                  │
│  ├─ enableProxy() - 启用系统代理                                             │
│  ├─ disableProxy() - 禁用系统代理                                            │
│  ├─ saveProxy() - 保存当前代理设置                                           │
│  └─ 管理代理状态                                                              │
│                                                                             │
│  PrivilegedHelperManager.swift                                             │
│  ├─ 检查守护进程状态                                                          │
│  ├─ 安装/更新守护进程                                                         │
│  ├─ 建立 XPC 连接                                                           │
│  └─ 权限管理                                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ XPC 通信
                                       │
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ProxyConfigHelper 守护进程                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ProxyConfigHelper.m                                                       │
│  ├─ XPC 服务端实现                                                           │
│  ├─ 接收主应用请求                                                            │
│  └─ 调用 ProxySettingTool                                                   │
│                                                                             │
│  ProxySettingTool.m                                                        │
│  ├─ enableProxyWithport() - 启用代理配置                                     │
│  ├─ disableProxyWithfilterInterface() - 禁用代理                            │
│  ├─ restoreProxySetting() - 恢复代理设置                                     │
│  └─ 使用 SystemConfiguration 框架                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ 系统调用
                                       │
┌─────────────────────────────────────────────────────────────────────────────┐
│                        macOS 系统框架                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  SystemConfiguration.framework                                             │
│  ├─ SCPreferencesCreate() - 创建系统配置引用                                 │
│  ├─ SCPreferencesGetValue() - 获取网络服务                                   │
│  ├─ SCPreferencesPathSetValue() - 设置代理配置                               │
│  ├─ SCPreferencesCommitChanges() - 提交更改                                  │
│  └─ SCPreferencesApplyChanges() - 应用更改                                   │
│                                                                             │
│  Authorization Services                                                     │
│  ├─ AuthorizationCreate() - 创建授权                                         │
│  └─ 管理员权限验证                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2. 启用代理流程图

```
[用户点击设为系统代理]
           │
           ▼
[AppDelegate 接收事件]
           │
           ▼
[调用 SystemProxyManager.enableProxy()]
           │
           ▼
[获取当前 HTTP 和 SOCKS 端口]
           │
           ▼
[通过 PrivilegedHelperManager 获取 helper]
           │
           ▼
[建立 XPC 连接到 ProxyConfigHelper]
           │
           ▼
[调用 helper.enableProxy() 方法]
           │
           ▼ (在守护进程中)
[ProxyConfigHelper 接收请求]
           │
           ▼
[创建 ProxySettingTool 实例]
           │
           ▼
[调用 enableProxyWithport() 方法]
           │
           ▼
[创建系统配置授权引用]
           │
           ▼
[获取所有网络服务列表]
           │
           ▼
[过滤网络接口 (WiFi, Ethernet, AirPort)]
           │
           ▼
[为每个接口生成代理设置字典]
    ┌──────┴──────┐
    │             │
    ▼             ▼
[HTTP 代理]   [SOCKS 代理]
127.0.0.1:port  127.0.0.1:socksPort
    │             │
    └──────┬──────┘
           ▼
[设置代理例外列表]
           │
           ▼
[通过 SCPreferencesPathSetValue 应用设置]
           │
           ▼
[提交并应用系统配置更改]
           │
           ▼
[代理设置生效]
```

## 3. 禁用代理流程图

```
[用户取消系统代理]
           │
           ▼
[SystemProxyManager.disableProxy()]
           │
           ▼
[检查 Settings.disableRestoreProxy]
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
[恢复模式]     [强制禁用模式]
    │             │
    ▼             ▼
[调用 restoreProxy] [调用 disableProxy]
    │             │
    ▼             ▼
[使用保存的设置] [清空所有代理设置]
    │             │
    └──────┬──────┘
           ▼
[应用到所有网络接口]
           │
           ▼
[提交系统配置更改]
           │
           ▼
[代理设置已禁用]
```

## 4. 守护进程安装流程图

```
[应用启动]
     │
     ▼
[PrivilegedHelperManager.checkInstall()]
     │
     ▼
[检查守护进程状态]
     │
  ┌──┴──┐
  │     │
  ▼     ▼
[已安装] [未安装/需更新]
  │     │
  │     ▼
  │   [显示安装提示对话框]
  │     │
  │     ▼
  │   [用户确认安装]
  │     │
  │     ▼
  │   [请求管理员权限]
  │     │
  │     ▼
  │   [使用 SMJobBless 安装守护进程]
  │     │
  │     ▼
  │   [验证安装结果]
  │     │
  └─────┴─────┐
              ▼
      [建立 XPC 连接]
              │
              ▼
      [守护进程准备就绪]
```

## 5. XPC 通信协议

```
主应用程序                     守护进程
     │                           │
     │ ─────────────────────────▶ │
     │  enableProxy(port, ...)    │
     │                           │
     │ ◀───────────────────────── │
     │    success/error callback  │
     │                           │
     │ ─────────────────────────▶ │
     │  disableProxy(...)         │
     │                           │
     │ ◀───────────────────────── │
     │    success/error callback  │
     │                           │
     │ ─────────────────────────▶ │
     │  getCurrentProxySetting()  │
     │                           │
     │ ◀───────────────────────── │
     │    proxy settings dict    │
     │                           │
     │ ─────────────────────────▶ │
     │  getVersion()              │
     │                           │
     │ ◀───────────────────────── │
     │    version string          │
```

## 6. 网络接口配置层次结构

```
macOS 网络服务配置
└── kSCPrefNetworkServices
    ├── 网络服务 1 (如 WiFi)
    │   ├── Interface.Hardware = "Wi-Fi"
    │   └── Proxies
    │       ├── HTTPProxy = "127.0.0.1"
    │       ├── HTTPPort = port
    │       ├── HTTPEnable = 1
    │       ├── HTTPSProxy = "127.0.0.1"
    │       ├── HTTPSPort = port
    │       ├── HTTPSEnable = 1
    │       ├── SOCKSProxy = "127.0.0.1"
    │       ├── SOCKSPort = socksPort
    │       ├── SOCKSEnable = 1
    │       └── ExceptionsList = [...]
    │
    ├── 网络服务 2 (如 Ethernet)
    │   ├── Interface.Hardware = "Ethernet"
    │   └── Proxies (同上配置)
    │
    └── ... (其他网络服务)
```

## 7. 权限和安全模型

```
┌─────────────────────────────────────────────────────────────────┐
│                      权限边界                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户进程 (ClashX.app)                                           │
│  ├─ 无系统配置修改权限                                             │
│  ├─ 通过 XPC 与守护进程通信                                        │
│  └─ 处理用户界面和应用逻辑                                         │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  系统进程 (ProxyConfigHelper)                                    │
│  ├─ 具有管理员权限                                                │
│  ├─ 可修改系统网络配置                                             │
│  ├─ 接受 XPC 请求                                                │
│  └─ 执行特权操作                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```