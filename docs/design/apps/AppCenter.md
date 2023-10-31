# AppCenter 应用设计

## 功能概括

负责存储 App 和颁发 AppToken。它将是 Aiursoft 基础设施 service-to-service call 的基础。

## 数据库

AppCenter 的数据库中只下面几个表：

* Apps，用于存储 App 的信息。
* Permissions，用于存储所有可能的权限。
* AppAssignedPermissions，用于存储 App 的权限。
* PermissionUseLogs，用于存储 Permission 的使用记录，以用于限流和统计。
* TokenAssignLogs，用于存储 Token 的分配记录，以用于限流和统计。

### App

App 有一个所属关系，每个 App 都有一个所属 App，所属 App 可以创建子 App，子 App 也可以递归创建子 App。

App 具有的属性有：

* AppId，App 的唯一标识符。
* AppName，App 的名称。
* AppSecret，App 的密钥。
* CreateTime，App 的创建时间。
* ParentAppId，App 的所属 App 的 AppId。
* AppPermissions[], 也就是凭借此 App 能够申请到的 AppToken 的权限。
* ExposedPermissions[], 也就是此 App 提供给其他 App 的 AppToken 的权限。

AppCenter 应用本身也是一个 App，其会在应用程序第一次加载时播种本身。其没有任何权限。这只是为了让任意一个App的 BelongToAppId 都不为空。播种时，AppCenter 本身的 BelongToAppId 为其自身的 AppId，并且 00000000-0000-0000-0000-000000000000。

### Permissions

Permissions 是一个权限表。其权限是以这个 App 的行为，在整个目录中进行操作的含义。

Permissions 具有的属性有：

* ID, 自增长的 ID。
* PermissionId，权限的唯一标识符。是一串 GUID。
* BelongToAppId，权限所属的 App 的 AppId。
* PermissionTag，权限的标签。这是为了方便在前端显示的。例如：`Apps`，`Buckets`，`Files`。
* PermissionName，权限的名称。用来给用户看的。例如：`查看所有 App`，`查看所有 Bucket`。
* PermissionDescription，权限的描述，用来给用户看的。
* AssignByDefault，是否默认分配此权限。如果为 true，则默认分配此权限，否则不分配此权限。
* Restricted，则默认限制此权限，禁止应用申请。
* PermissionString，权限的字符串表示。必须是全局唯一的。这是一个 String，例如：`appsManagement:view`。
* ScopeRegex，这是一个可选功能，表示这个 Permission 在申请时可以指定的 Scope 的正则表达式。如果为空，则表示不需要限制 Scope。例如：`bucket_id=.*` 表示这个 Permission 在申请时可以指定的 Scope 是 `bucket_id=xxx` 的形式，其中 `xxx` 可以是任意字符串。

注意：权限这里有两个功能非常容易混淆，一个是：

* 应用本身会以自己的身份去暴露一系列可以被其他应用申请的权限。这个关系是一对多的。这部分名称空间称作 `permissionPublish`。
  * 使用词汇：
    * 搜索 (Search)
    * 发布 (Publish)
    * 查询 (Query)
    * 编辑 (Edit)
    * 删除 (Delete)
* 应用本身会以自己的身份去具有的一系列权限。这个关系是多对多的。这部分名称空间称作 `permissionsManagement`。
  * 使用词汇
    * 列举 (List)
    * 分配 (Assign)
    * 禁用 (Revoke)
    * 具有 (Has)

#### 限制权限

这部分权限是高危权限，需要特殊处理。一般不会分配给任何 App，只有一些特殊应用，例如 `开发者中心` 应用自己可以获取。

具有这类权限的应用一般都是非常特殊的全局目录管理器。例如 `开发者中心` 应用可以管理所有的 App。

这部分权限的特点是：Restricted 为 true，AssignByDefault 为 false。

* `appsManagement:view`     表示查看任意一个 App 的权限。
* `appsManagement:edit`     表示编辑任意一个 App 的权限。
* `appsManagement:delete`   表示删除任意一个 App 的权限。
* `appsManagement:permissionPublish:publish`    以任意 App 的身份发布一个权限。     Scope 必填且为此 App 的 AppId。
* `appsManagement:permissionPublish:query`      查询任意 App 发布的权限列表。       Scope 必填且为此 App 的 AppId。
* `appsManagement:permissionPublish:edit`       编辑任意 App 发布的权限。           Scope 必填且为此 App 的 AppId。
* `appsManagement:permissionPublish:delete`     删除任意 App 发布的权限。             Scope 必填且为此 App 的 AppId。
* `appsManagement:permissionsManagement:list`   列举任意一个 App 具有的权限列表。       Scope 必填且为此 App 的 AppId。
* `appsManagement:permissionsManagement:assign` 为任意一个 App 分配权限。               Scope 必填且为此 App 的 AppId。
* `appsManagement:permissionsManagement:revoke` 禁用任意一个 App 的权限。               Scope 必填且为此 App 的 AppId。
* `appsManagement:secretManagement:create`      创建任意一个 App 的一个 Secret 的权限。  Scope 必填且为此 App 的 AppId。

#### 普通权限

这部分权限是普通权限，不会默认分配给 App。但是 App 可以申请这些权限（自动批准）。

这部分权限的特点是：Restricted 为 false，AssignByDefault 为 false。

* `serviceB:buckets-access` 表示查看 App 下的所有 Bucket 的权限。这是一个第三方权限。
* `appsManagement:search`   表示查看 App 列表的权限。
* `appsManagement:create`   表示创建 App 的权限。（每 App 每小时限制 10 次）
* `appCurrent:permissionPublish:publish` 以此 App 发布一个新的权限。（每 App 每小时限制 30 次）
* `appCurrent:permissionPublish:query` 以此 App 查询已经发布的权限列表。
* `appCurrent:permissionPublish:edit` 以此 App 编辑已经发布的权限。
* `appCurrent:permissionPublish:delete` 以此 App 删除已经发布的权限。

#### 默认权限

这部分权限是默认权限，可以分配给任何 App，并且默认分配给任何 App。

这部分权限的特点是：Restricted 为 false，AssignByDefault 为 true。

* `appCurrent:view` 查看此 App 的基本信息。
* `appCurrent:edit` 编辑此 App 的基本信息。
* `appCurrent:delete` 删除此 App 。
* `appCurrent:permissionsManagement:list` 列举此 App 具有的权限列表。
* `appCurrent:permissionsManagement:assign` 为此 App 分配一个权限。（每 App 每小时限制 100 次）
* `appCurrent:permissionsManagement:revoke` 为此 App 禁用一个权限。

#### 公开权限

某些操作不需要特定权限，直接使用任意 Token 即可。也就说全世界的用户都可以访问这些权限。

第三方应用如果不验证 AppToken，此 API 即为这个权限。

* `appCurrent:permissionPublish:search` 搜索已经发布的权限。

#### 第三方权限

这部分权限是第三方应用可以获取的权限。这些权限可能会后期被动态增加。例如：

* `buckets:list` 表示查看 App 下的所有 Bucket 的权限。

例如场景：“开发者中心”是一个App。用户通过“开发者中心”去管理自己的App时，开发者中心需要有权限去创建、编辑、删除、查看其他App。此时，开发者中心可以使用自己的AppId和AppSecret去颁发AppToken，此AppToken具有创建、编辑、删除、查看其他特定App的权限，然后去管理其他App。

### AppAssignedPermissions

AppAssignedPermissions，用于存储 是一个 App 的权限表。其每一行都表示着一个 App 可以申请到的 AppToken 的权限。

其基本属性有：

* ID, 自增长的 ID。
* AppId，App 的 AppId。
* PermissionId，权限的 PermissionId。
* AssignTime，分配时间。

### PermissionUseLogs

PermissionUseLogs 用于存储 Permission 的使用记录，以用于限流和统计。

注意，这个表并不是完整的数据。这是考虑到真正场景中 service-to-service call 是不需要经过 AppCenter 应用的。所以这个表只是用于统计和限流。

其基本属性有：

* ID, 自增长的 ID。
* PermissionId，权限的 PermissionId。
* AppId，App 的 AppId。
* UseTime，使用时间。
* Scope，Scope 字符串。
* IP，IP 地址。

### TokenAssignLogs

TokenAssignLogs 用于存储 Token 的分配记录，以用于限流和统计。

这个表的数据是精准的，可以直接查询到一个 App 历史上所有的 Token 分配记录。

其基本属性有：

* ID, 自增长的 ID。
* AppId，App 的 AppId。
* Token，Token 的字符串。
* AssignTime，分配时间。
* IP，IP 地址。

## API 接口设计

AppCenter 应用只提供下面几个接口：

Config:

* GetPublicKey

AppsManagement:

* SearchApps (普通权限 `appsManagement:search`)
* CreateApp (普通权限 `appsManagement:create`)
* ViewApp (普通权限 `appCurrent:view`只能查看当前 App)
* EditApp (普通权限 `appCurrent:edit`只能编辑当前 App)
* DeleteApp (普通权限 `appCurrent:delete`只能删除当前 App)

Token:

* GetAppToken

PermissionPublish:

* SearchAllPermissions (公开权限 `appCurrent:permissionPublish:search`)
* SearchAllPermissionsTags (公开权限 `appCurrent:permissionPublish:search`)
* SearchAllPermissionsByTag (公开权限 `appCurrent:permissionPublish:search`)
* SearchAllPermissionsStartWith (公开权限 `appCurrent:permissionPublish:search`)
* PublishPermission (普通权限 `appCurrent:permissionPublish:publish`)
* QueryPermission (普通权限 `appCurrent:permissionPublish:query`)
* EditPermission (普通权限 `appCurrent:permissionPublish:edit`)
* DeletePermission (普通权限 `appCurrent:permissionPublish:delete`)

PermissionManagement:

* ListPermission (默认权限 `appCurrent:permissionsManagement:list`)
* AssignPermission (默认权限 `appCurrent:permissionsManagement:assign`)
* RevokePermission (默认权限 `appCurrent:permissionsManagement:revoke`)

AdminAppsManagement:

* ViewApp (限制权限 `appsManagement:view` 可以查看任意 App)
* EditApp (限制权限 `appsManagement:edit` 可以编辑任意 App)
* DeleteApp (限制权限 `appsManagement:delete` 可以删除任意 App)
  
AdminPermissionPublish:

* PublishPermission (限制权限 `appsManagement:permissionPublish:publish`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)
* QueryPermission (限制权限 `appsManagement:permissionPublish:query`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)
* EditPermission (限制权限 `appsManagement:permissionPublish:edit`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)
* DeletePermission (限制权限 `appsManagement:permissionPublish:delete`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)

AdminPermissionManagement:

* ListPermission (限制权限 `appsManagement:permissionsManagement:list`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)
* AssignPermission (限制权限 `appsManagement:permissionsManagement:assign`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)
* RevokePermission (限制权限 `appsManagement:permissionsManagement:revoke`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)

Secret:

* CreateAppSecret

## 关键类

### AppToken

AppToken 是一个临时的 Token，其有效期为 20 分钟，或在申请时动态指定。

AppToken 由 AppId、权限列表、过期时间、签名、Scope 组成。

其中：

* iss 是在颁发这个 AppToken 时，AppCenter 系统本身的 AppId。这个在 AppCenter 播种时就已经确定了。
* aud 是在颁发这个 AppToken 时请求的 App 的 AppId。
* exp 是在颁发这个 AppToken 时请求的过期时间。默认是自从生成后的 20 分钟。
* permissions 是在颁发这个 AppToken 时请求的权限列表。注意：申请 AppToken 时只能申请这个 App 具有的权限。如果申请的权限列表中包含 App 本身没有的权限，则会报错。
* 签名是对整个 AppToken 的上述明文内容的签名。签名使用 AppCenter 的私钥进行签名。
* Scope 是一个字符串，用于标识这个 AppToken 的作用范围。例如：`bucket_id=xxx` 表示这个 AppToken 只能用于操作 BucketId 为 xxx 的 Bucket。这可以用于方便应用程序后续限制每个 AppToken 的作用范围。注意：AppCenter 系统无法验证 Scope 的有效性，只能由应用程序自己验证。

AppToken 的生成采用三段式：`{type}`.`{content}`.`{signature}`。

### FrequencyLimiter

FrequencyLimiter 是一个服务器类，用于限制某个操作的频率。它会在数据库中记录某个操作的次数，然后在一段时间内限制某个操作的次数。

当有新操作来临时，它会查询表：`PermissionUseLogs`，然后统计某个操作在一段时间内的次数。如果超过了限制，则会抛出错误。如果没有超过限制，则会记录一条新的操作记录。

### TokenGenerator

TokenGenerator 是一个 SDK 类，用于生成 AppToken。

其中，`{type}` 是一段 JSON 在进行 base64 编码后的字符串，其原始内容为：

```json
{
    "alg": "RS256",
    "typ": "JWT"
}
```

`{content}` 是一段 JSON 在进行 base64 编码后的字符串，其原始内容为：

```json
{
    "iss": "xxx",
    "aud": "xxx",
    "exp": "xxx",
    "permissions": [
        "xxx",
        "xxx"
    ],
    "scope": "bucket_id=xxx"
}
```

`{signature}` 则是对 `{type}`.`{content}` 的内容在 base64 后的签名。

生成 `{signature}` 的 C# 代码示例为：

```csharp
var rsa = new RSACryptoServiceProvider();
rsa.ImportParameters(new RSAParameters()
{
    Modulus = Convert.FromBase64String("xxx"),
    Exponent = Convert.FromBase64String("xxx"),
    D = Convert.FromBase64String("xxx"),
    P = Convert.FromBase64String("xxx"),
    Q = Convert.FromBase64String("xxx"),
    DP = Convert.FromBase64String("xxx"),
    DQ = Convert.FromBase64String("xxx"),
    InverseQ = Convert.FromBase64String("xxx")
});
var signature = rsa.SignData(Encoding.UTF8.GetBytes("{type}.{content}"), HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
var signatureString = Convert.ToBase64String(signature);
```

### TokenAsserter

TokenAsserter 是一个 SDK 类，用于验证 AppToken 的有效性。

它还额外提供一些方法，例如：要求 AppToken 必须具有某个权限。

它在验证失败时会抛出错误。而在验证成功后，会返回 Token 的 `{content}` 部分的内容作为对象。

```csharp
public async Task<AppToken> AssertToken(string token, string permissionString = null);
```

### TokenDownloader

TokenDownloader 是一个 SDK 类，用于获取 AppToken。它会从配置文件中读取 AppId 和 AppSecret，然后向 AppCenter 系统获取 AppToken。

它会将获取到的 AppToken 缓存起来，以便下次使用。

```csharp
public async Task<string> GetToken(string permissionString = null, TimeSpan? expireTime = null);
```

## 实际场景

当两个应用进行 service-to-service call 时，我们称作应用 A 和应用 B。应用 A 调用应用 B 的 API。

这个 API 可能是：

* 创建一个 Bucket。
* 获取某一个 Bucket 下的文件列表。

### 示例：创建一个 Bucket

此时，应用 B 至少要在 AppCenter 里注册一个权限，例如： `b:buckets-create`。应用 A 必须要具有这个权限，才能调用应用 B 的 API。也就是在数据库中，必须存在一个 AppPermissions，其 AppId 为应用 A 的 AppId，PermissionId 为 `b:buckets-create`。

当调用时，应用 A 必须向 AppCenter 系统获取一个 AppToken，此 AppToken 必须具有 `b:buckets-create` 权限。此时，应用 A 可以使用此 AppToken 调用应用 B 的 API。

当应用 B 需要验证 AppToken 时，应用 B 需要获取 AppCenter 的公钥，然后使用此公钥验证 AppToken 的有效性。它需要确保 AppToken 未过期，且其权限中包含 `b:buckets-create`。

### 示例2：获取某一个 Bucket 下的文件列表

此时，应用 B 至少要在 AppCenter 里注册一个权限，例如： `b:buckets-access`。这个权限的 Scope 的正则表达式为：`bucket_id=.*`。

应用 A 必须要具有这个权限，才能调用应用 B 的 API。也就是在数据库中，必须存在一个 AppPermissions，其 AppId 为应用 A 的 AppId，PermissionId 为 `b:buckets-access`。

当调用时，应用 A 必须向 AppCenter 系统获取一个 AppToken，此 AppToken 必须具有 `b:buckets-access` 权限，且 Scope 为 `bucket_id=xxx`。此时，应用 A 可以使用此 AppToken 调用应用 B 的 API。

当应用 B 需要验证 AppToken 时，应用 B 需要获取 AppCenter 的公钥，然后使用此公钥验证 AppToken 的有效性。它需要确保 AppToken 未过期，且其权限中包含 `b:buckets-access`，且 Scope 为 `bucket_id=xxx`。

应用 B 还需要额外验证 `xxx` 是否为一个合法的 BucketId，以及它是否属于应用 A。在这个例子中，应用 B 可以查询数据库，查看 `xxx` 是否为应用 A 的 BucketId。如果是，则允许访问，否则拒绝访问。

### 示例3 - 第三方 Application Seeding

当一个应用程序第一次启动时，它需要向 AppCenter 系统播种自己。这个过程是自动的，不需要人工干预。

在应用程序启动前，开发人员必须手工在 AppCenter 系统中创建一个 App，其 AppId 和 AppSecret 为应用程序的 AppId 和 AppSecret。此 App 必须具有 `appCurrent:permissionsManagement:assign` 权限。（实际上，这一步无需过多操心，因为这个权限是默认分配给任何 App 的。）

然后，开发人员必须手工将 AppId 和 AppSecret 写入应用程序的配置文件中。

当应用程序第一次启动时，它必须确保它需要的权限已经配置妥当。它会阅读自己的配置文件，获取自己的 AppId 和 AppSecret。

然后，它会向 AppCenter系统发送一个请求，以换取 AppToken，且必须具有 `appCurrent:permissionsManagement:assign` 权限。

随后，它只需要调用 API：`AssignPermission`，即可将自己需要的权限分配给自己。例如 `b:buckets-create`，`b:buckets-access`。

### 示例4 - 本地应用程序 Seeding

当 Aiursoft 核心服务被开发时，为了方便测试，必须能够一键启动整个测试环境。因此需要本地应用程序免注册启动后直接调用。

因此，这里需要让应用程序启动时直接获取到自己的 AppID 和 AppSecret。这个过程是自动的，不需要人工干预。

因此，AppCenter 应用必须额外具有另一种启动模式，也就是播种模式。这可以通过命令行参数来进入播种模式。

在播种模式下，它可以直接向数据库中插入一个 App，并将 App ID、App Secret 输出到控制台。

之后，我们只需要使用 bash 自动读取这些输出，写入配置文件中即可。然后按照上面的第三方 Application Seeding 过程即可。
