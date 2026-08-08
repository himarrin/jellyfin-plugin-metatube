# Tags 添加流程详解

## 完整的数据流

### 1. **数据源头：metatube-sdk-go API**

当 Jellyfin/Emby 请求演员元数据时，插件会调用 metatube API：

```
GET http://192.168.66.66:8008/v1/actors/AV-LEAGUE/6752?lazy=true
```

**API 返回的 JSON 结构**：
```json
{
  "data": {
    "id": "6752",
    "name": "波多野結衣",
    "provider": "AV-LEAGUE",
    "tags": [
      "30代",
      "美魔女",
      "巨乳",
      "Debut：2008",
      "三围：88/59/85",
      "罩杯：E",
      "身高：163cm",
      "出生日期：1988-05-24",
      "年龄：37"
    ],
    "twitter": "hatano_yui",
    "instagram": "hatachan524",
    ...
  }
}
```

### 2. **ApiClient 获取和解析数据**

**文件**：`ApiClient.cs`

```csharp
// 第 146-150 行
public static async Task<ActorInfo> GetActorInfoAsync(string provider, string id, bool lazy,
    CancellationToken cancellationToken)
{
    var apiUrl = ComposeInfoApiUrl(ActorInfoApi, provider, id, lazy);
    return await GetDataAsync<ActorInfo>(apiUrl, true, cancellationToken);
}
```

**关键步骤**：
1. 构建 API URL：`/v1/actors/AV-LEAGUE/6752?lazy=true`
2. 发送 HTTP GET 请求
3. 接收 JSON 响应
4. 使用 `ReadFromJsonAsync<ResponseInfo<ActorInfo>>` 反序列化

**反序列化过程**（第 232-233 行）：
```csharp
var apiResponse = await response.Content!
    .ReadFromJsonAsync<ResponseInfo<ActorInfo>>(cancellationToken: cancellationToken);
```

### 3. **ActorInfo 类自动映射 JSON 字段**

**文件**：`Metadata/ActorInfo.cs`

```csharp
public class ActorInfo : ActorSearchResult
{
    [JsonPropertyName("tags")]  // ← 这个属性告诉 JSON 反序列化器
    public string[] Tags { get; set; }  // ← 将 JSON 中的 "tags" 字段映射到这里
}
```

**工作原理**：
- `System.Text.Json` 的 `ReadFromJsonAsync` 会自动：
  1. 查找 JSON 中的 `"tags"` 字段
  2. 将其值（字符串数组）赋值给 `ActorInfo.Tags` 属性
  3. 如果字段不存在或为 null，`Tags` 将为 `null`

### 4. **ActorProvider 处理 Tags**

**文件**：`Providers/ActorProvider.cs`

**第 37 行**：获取 ActorInfo
```csharp
var m = await ApiClient.GetActorInfoAsync(pid.Provider, pid.Id, cancellationToken);
// 此时 m.Tags 已经包含了从 API 返回的所有标签
```

**第 58-67 行**：添加 Tags 到 Person 对象
```csharp
// Add tags from metatube API.
if (m.Tags != null && m.Tags.Length > 0)
{
    foreach (var tag in m.Tags)  // 遍历每个标签
    {
        if (!string.IsNullOrWhiteSpace(tag))  // 检查标签不为空
            result.Item.AddTag(tag);  // ← 关键：调用 AddTag 方法
    }
    Logger.Info("Added {0} tags for actor {1}", m.Tags.Length, m.Name);
}
```

### 5. **AddTag 方法的工作原理**

`AddTag` 是 MediaBrowser 框架（Jellyfin/Emby）提供的扩展方法：

```csharp
result.Item.AddTag(tag);
```

**内部实现**（MediaBrowser 框架）：
- `Person` 类继承自 `BaseItem`
- `BaseItem` 有一个 `Tags` 集合属性（`List<string>` 或类似结构）
- `AddTag(string tag)` 方法会：
  1. 检查标签是否已存在（避免重复）
  2. 如果不存在，添加到 `Tags` 集合中
  3. 标记对象为"已修改"，以便保存到数据库

**伪代码示例**：
```csharp
// MediaBrowser 框架内部（简化版）
public class BaseItem
{
    private List<string> _tags = new List<string>();
    
    public void AddTag(string tag)
    {
        if (!string.IsNullOrWhiteSpace(tag) && !_tags.Contains(tag))
        {
            _tags.Add(tag);
            // 触发保存机制
        }
    }
}
```

### 6. **数据保存到 Emby/Jellyfin**

当 `GetMetadata` 方法返回 `MetadataResult<Person>` 后：

1. **Jellyfin/Emby 框架接收结果**
   - 框架会检查 `result.Item` 的所有属性
   - 包括 `Tags` 集合

2. **保存到数据库**
   - 框架将 `Person` 对象序列化
   - 保存到 Emby/Jellyfin 的数据库
   - Tags 存储在 `Items` 表的 `Tags` 字段中（通常是 JSON 数组或关联表）

3. **在 UI 中显示**
   - 标签会出现在演员详情页面
   - 可以用于搜索和筛选

## 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Jellyfin/Emby 请求演员元数据                              │
│    PersonLookupInfo { Name: "波多野結衣" }                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. ActorProvider.GetMetadata()                              │
│    - 解析 ProviderID (AV-LEAGUE:6752)                       │
│    - 调用 ApiClient.GetActorInfoAsync()                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. ApiClient.GetActorInfoAsync()                            │
│    - 构建 URL: /v1/actors/AV-LEAGUE/6752?lazy=true          │
│    - 发送 HTTP GET 请求                                      │
│    - 接收 JSON 响应                                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. JSON 反序列化                                             │
│    ReadFromJsonAsync<ResponseInfo<ActorInfo>>()              │
│    ↓                                                          │
│    {                                                          │
│      "data": {                                               │
│        "tags": ["30代", "美魔女", "巨乳", ...]                │
│      }                                                        │
│    }                                                          │
│    ↓                                                          │
│    ActorInfo {                                                │
│      Tags = ["30代", "美魔女", "巨乳", ...]                    │
│    }                                                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. ActorProvider 处理 Tags                                  │
│    if (m.Tags != null && m.Tags.Length > 0)                  │
│    {                                                          │
│      foreach (var tag in m.Tags)                             │
│      {                                                        │
│        result.Item.AddTag(tag);  ← 关键步骤                  │
│      }                                                        │
│    }                                                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. AddTag() 方法执行                                         │
│    - 检查标签是否已存在                                       │
│    - 添加到 Person.Tags 集合                                 │
│    - 标记为已修改                                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. 返回 MetadataResult<Person>                               │
│    result.Item.Tags = ["30代", "美魔女", "巨乳", ...]         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Jellyfin/Emby 框架保存                                   │
│    - 将 Person 对象保存到数据库                              │
│    - Tags 写入 Items 表                                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 9. 在 UI 中显示                                              │
│    - 演员详情页面显示标签                                     │
│    - 可以按标签搜索和筛选                                     │
└─────────────────────────────────────────────────────────────┘
```

## 关键代码位置

### 1. **API 返回数据**
- **文件**：`metatube-sdk-go/route/info.go`
- **方法**：`getInfo()` → `c.JSON(http.StatusOK, &responseMessage{Data: info})`
- **结果**：返回包含 `tags` 字段的 JSON

### 2. **JSON 反序列化**
- **文件**：`ApiClient.cs`
- **方法**：`GetDataAsync<T>()` → `ReadFromJsonAsync<ResponseInfo<ActorInfo>>()`
- **结果**：`ActorInfo.Tags` 被填充

### 3. **Tags 映射**
- **文件**：`Metadata/ActorInfo.cs`
- **属性**：`[JsonPropertyName("tags")] public string[] Tags { get; set; }`
- **作用**：JSON 字段名到 C# 属性的映射

### 4. **添加 Tags**
- **文件**：`Providers/ActorProvider.cs`
- **方法**：`GetMetadata()` → `result.Item.AddTag(tag)`
- **作用**：将每个标签添加到 Person 对象

### 5. **框架处理**
- **框架**：MediaBrowser (Jellyfin/Emby)
- **方法**：`BaseItem.AddTag(string tag)`
- **作用**：将标签添加到内部集合并保存

## 验证方法

### 1. 检查 API 返回
```bash
curl "http://192.168.66.66:8008/v1/actors/AV-LEAGUE/6752?lazy=false" | jq '.data.tags'
```

### 2. 检查插件日志
在 Jellyfin/Emby 日志中查找：
```
Added 17 tags for actor 波多野結衣
```

### 3. 检查数据库
在 Emby/Jellyfin 数据库中查询：
```sql
SELECT Tags FROM Items WHERE Type = 'Person' AND Name = '波多野結衣';
```

## 注意事项

1. **lazy 参数的影响**
   - `lazy=true`：优先从数据库读取，可能没有最新的 tags
   - `lazy=false`：强制重新爬取，确保有最新的 tags

2. **空值处理**
   - 如果 API 返回 `"tags": null`，`m.Tags` 将为 `null`
   - 代码中有检查：`if (m.Tags != null && m.Tags.Length > 0)`

3. **重复标签**
   - `AddTag` 方法会自动去重
   - 不会添加已存在的标签

4. **标签格式**
   - 标签是纯文本字符串
   - 支持中文、日文、英文等任何字符
   - 大小写敏感（"巨乳" 和 "巨乳" 被视为不同标签）
