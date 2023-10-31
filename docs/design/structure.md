# 进程依赖顺序

```mermaid
stateDiagram-v2
    Portal --> Gateway
    Gateway --> Buckets
    Gateway --> Directory
    Radio --> Observer
    Portal --> Radio
    Portal --> Buckets
    Observer --> AppCenter
    Directory --> Observer
    Buckets --> Observer
```

## 项目代码依赖关系

```mermaid
stateDiagram-v2
    Shared --> CSTools
    Sdk.Framework --> Shared
    Sdk.Framework --> AiurProtocol
    ApiApp.Framework --> WebShared
    ApiApp.Framework --> DocGenerator
    ApiApp.Framework --> AiurProtocol.Server
    WebApp.Framework --> WebShared
    WebShared --> WebTools
    WebShared --> Shared
    WebShared --> DbTools
    AppCenter --> ApiApp.Framework
    AppCenter --> AppCenter.Sdk
    AppCenter.Tests --> AppCenter
    AppCenter.Sdk --> Sdk.Framework
    Observer --> ApiApp.Framework
    Observer --> AppCenter.Sdk
    Observer --> Observer.Sdk
    Observer.Tests --> Observer
    Observer.Sdk --> Sdk.Framework
```

## 开发顺序

1. AppCenter (API App)
2. Observer (API App)
3. Buckets (API App)
4. Directory (API App)
5. Radio (API App)
6. Gateway (Human App)
7. Portal (Human App)
