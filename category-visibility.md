# Category 分类可见度与跨用户同步机制分析

## 一、核心概念澄清

首先需要明确：**Miniflux 中不存在跨用户共享的 Category**。每个分类（Category）完全属于单个用户，不同用户的分类是完全独立的实体。

用户提到的"同一个 Category 在多个用户之间共享"、"订阅复制"等概念，在代码中实际体现为：
- `hide_globally` 属性控制分类在当前用户的"全局视图"中是否可见
- OPML 导入/导出是"复制"订阅结构的主要方式
- Fever API 和 Google Reader API 提供了第三方客户端的分类同步能力

---

## 二、Category 数据模型

### 2.1 模型定义

文件：`internal/model/category.go:9-17`

```go
type Category struct {
    ID           int64  `json:"id"`
    Title        string `json:"title"`
    UserID       int64  `json:"user_id"`
    HideGlobally bool   `json:"hide_globally"`
    FeedCount    *int   `json:"feed_count,omitempty"`
    TotalUnread  *int   `json:"total_unread,omitempty"`
}
```

### 2.2 数据库表结构

文件：`internal/database/migrations.go:47-54`

```sql
CREATE TABLE categories (
    id SERIAL,
    user_id int not null,
    title text not null,
    primary key (id),
    unique (user_id, title),
    foreign key (user_id) references users(id) on delete cascade
);
```

**关键点**：
- `user_id` 非空，每个分类必须属于一个用户
- `unique (user_id, title)`：同一用户下分类标题唯一（数据库层面约束）
- 用户删除时级联删除其所有分类

---

## 三、分类创建流程

### 3.1 API 层创建

文件：`internal/api/category_handlers.go:20-41`

```go
func (h *handler) createCategoryHandler(w http.ResponseWriter, r *http.Request) {
    userID := request.UserID(r)
    
    // 解析请求
    var categoryCreationRequest model.CategoryCreationRequest
    json_parser.NewDecoder(r.Body).Decode(&categoryCreationRequest)
    
    // 验证
    validator.ValidateCategoryCreation(h.store, userID, &categoryCreationRequest)
    
    // 创建
    category, err := h.store.CreateCategory(userID, &categoryCreationRequest)
}
```

### 3.2 验证逻辑

文件：`internal/validator/category.go:12-23`

```go
func ValidateCategoryCreation(store *storage.Storage, userID int64, request *model.CategoryCreationRequest) *locale.LocalizedError {
    if request.Title == "" {
        return locale.NewLocalizedError("error.title_required")
    }
    if store.CategoryTitleExists(userID, request.Title) {
        return locale.NewLocalizedError("error.category_already_exists")
    }
    return nil
}
```

### 3.3 数据库创建

文件：`internal/storage/category.go:172-204`

```go
func (s *Storage) CreateCategory(userID int64, request *model.CategoryCreationRequest) (*model.Category, error) {
    query := `
        INSERT INTO categories
            (user_id, title, hide_globally)
        VALUES
            ($1, $2, $3)
        RETURNING id, user_id, title, hide_globally
    `
    // ...
}
```

### 3.4 UI 层创建

文件：`internal/ui/category_save.go`（类似逻辑）

表单定义：`internal/ui/form/category.go:10-22`

```go
type CategoryForm struct {
    Title        string
    HideGlobally bool
}
```

**创建流程总结**：
1. 从请求上下文获取当前用户 ID
2. 验证标题非空且不重复（同一用户内）
3. 携带 `user_id` 插入数据库
4. 分类自动归属于当前用户

---

## 四、可见度判定规则

### 4.1 HideGlobally 的真正含义

`hide_globally` **不是**跨用户可见性控制，而是控制当前用户自己的视图中，该分类是否在"全局汇总视图"中显示。

- **全局视图**：未读列表、全部条目、导航栏未读数等
- **分类视图**：进入具体分类页面时仍能看到所有内容

### 4.2 应用 HideGlobally 过滤的场景

#### 4.2.1 导航元数据（未读数统计）

文件：`internal/storage/nav_metadata.go:20-97`

```sql
SELECT count(*)
FROM entries e
JOIN feeds f ON f.id = e.feed_id
JOIN categories c ON c.id = f.category_id
WHERE e.user_id = $1
  AND e.status = 'unread'
  AND f.hide_globally IS FALSE
  AND c.hide_globally IS FALSE
```

导航栏的未读数**不包含**隐藏分类下的未读条目。

#### 4.2.2 条目查询构建器

文件：`internal/storage/entry_query_builder.go:226-230`

```go
func (e *EntryQueryBuilder) WithGloballyVisible() *EntryQueryBuilder {
    e.conditions = append(e.conditions, "c.hide_globally IS FALSE")
    e.conditions = append(e.conditions, "f.hide_globally IS FALSE")
    return e
}
```

文件：`internal/storage/entry_pagination_builder.go:102-108`

```go
func (e *entryPaginationBuilder) WithGloballyVisible() *entryPaginationBuilder {
    e.conditions = append(e.conditions, "not c.hide_globally")
    e.conditions = append(e.conditions, "not f.hide_globally")
    return e
}
```

#### 4.2.3 UI 层中使用全局可见过滤的位置

| 页面 | 文件 | 说明 |
|------|------|------|
| 未读条目 | `internal/ui/unread_entries.go:30` | 使用 `WithGloballyVisible()` |
| 条目未读操作 | `internal/ui/entry_unread.go:48` | 使用 `WithGloballyVisible()` |
| 全部标记已读 | `internal/ui/unread_mark_all_read.go:14` | `MarkGloballyVisibleFeedsAsRead` |

#### 4.2.4 标记全部已读

文件：`internal/storage/entry.go:551-568`

```go
func (s *Storage) MarkGloballyVisibleFeedsAsRead(userID int64) error {
    query := `
        UPDATE entries SET status=$1
        WHERE entries.feed_id = feeds.id
          AND entries.user_id=$2
          AND entries.status=$3
          AND feeds.hide_globally=$4
    `
    // 只标记非隐藏订阅的条目为已读
}
```

### 4.3 不应用 HideGlobally 过滤的场景

以下场景用户能看到自己所有的分类（包括 hide_globally 的）：

1. **分类列表页**：`internal/storage/category.go:90-110` 的 `Categories()` 方法不做过滤
2. **分类详情页**：通过 `Category(userID, categoryID)` 直接查询，带 user_id 校验
3. **分类下的 Feed 列表**：直接按 category_id 查询
4. **分类的创建/编辑/删除**：都能操作自己的所有分类

### 4.4 Feed 也有 HideGlobally

文件：`internal/model/feed.go:53`

Feed 同样有 `HideGlobally` 属性，其行为与 Category 的 `HideGlobally` 类似：
- 全局视图中隐藏该订阅
- 但仍可通过订阅详情页访问

**注意**：Category 和 Feed 的 hide_globally 是独立的，但在全局视图过滤时**同时检查**两者。

---

## 五、编辑权限控制

### 5.1 核心原则：只有所有者能操作

所有分类相关的数据库操作都带上了 `user_id` 条件，确保用户只能操作自己的分类。

#### 5.1.1 查询分类

文件：`internal/storage/category.go:40-54`

```go
func (s *Storage) Category(userID, categoryID int64) (*model.Category, error) {
    query := `SELECT id, user_id, title, hide_globally FROM categories WHERE user_id=$1 AND id=$2`
    // ...
}
```

#### 5.1.2 更新分类

文件：`internal/storage/category.go:206-222`

```go
func (s *Storage) UpdateCategory(category *model.Category) error {
    query := `UPDATE categories SET title=$1, hide_globally=$2 WHERE id=$3 AND user_id=$4`
    // ...
}
```

#### 5.1.3 删除分类

文件：`internal/storage/category.go:224-242`

```go
func (s *Storage) RemoveCategory(userID, categoryID int64) error {
    query := `DELETE FROM categories WHERE id = $1 AND user_id = $2`
    // ...
}
```

### 5.2 API 层的权限校验

API 层通过 `request.UserID(r)` 获取当前用户 ID，并传入存储层，由存储层的 SQL 条件保证权限。

示例（更新分类）：
```go
// internal/api/category_handlers.go:43-82
func (h *handler) updateCategoryHandler(w http.ResponseWriter, r *http.Request) {
    userID := request.UserID(r)
    categoryID := request.RouteInt64Param(r, "categoryID")
    
    category, err := h.store.Category(userID, categoryID)
    if category == nil {
        response.JSONNotFound(w, r)
        return
    }
    // ... 更新
}
```

### 5.3 管理员权限

管理员（`is_admin=true`）主要用于用户管理，**不能**直接操作其他用户的分类。

文件：`internal/api/user_handlers.go:78-88`

```go
if !request.IsAdminUser(r) {
    if originalUser.ID != request.UserID(r) {
        response.JSONForbidden(w, r)
        return
    }
    // 普通用户不能给自己提升管理员权限
    if userModificationRequest.IsAdmin != nil && *userModificationRequest.IsAdmin {
        response.JSONBadRequest(w, r, errors.New("..."))
        return
    }
}
```

管理员权限仅用于：
- 创建/删除用户
- 修改用户的管理员角色
- 查看所有用户列表

---

## 六、"跨用户订阅复制"的实际机制

实际上没有真正的"共享分类"，但有几种方式可以在用户间"复制"订阅结构。

### 6.1 OPML 导入/导出

这是最主要的"订阅复制"方式。

#### 6.1.1 OPML 导出

文件：`internal/reader/opml/handler.go:20-57`

```go
func (h *Handler) Export(userID int64) (string, error) {
    feeds, err := h.store.Feeds(userID)
    
    subscriptions := make([]subcription, 0, len(feeds))
    for _, feed := range feeds {
        subscriptions = append(subscriptions, subcription{
            Title:        feed.Title,
            FeedURL:      feed.FeedURL,
            CategoryName: feed.Category.Title, // 只导出分类名称
            // ... 其他设置
            HideGlobally: feed.HideGlobally,
        })
    }
    return serialize(subscriptions), nil
}
```

**导出的是分类名称（字符串），不是分类 ID。**

#### 6.1.2 OPML 导入

文件：`internal/reader/opml/handler.go:59-95`

```go
func (h *Handler) Import(userID int64, data io.Reader) error {
    subscriptions, _ := parse(data)
    
    for _, subscription := range subscriptions {
        // 如果该 feed 已存在，跳过
        if h.store.FeedURLExists(userID, subscription.FeedURL) {
            continue
        }
        
        // 按名称解析/创建分类
        category, err := h.resolveCategory(userID, subscription.CategoryName)
        // ... 创建 feed
    }
}
```

#### 6.1.3 分类解析策略

文件：`internal/reader/opml/handler.go:97-119`

```go
func (h *Handler) resolveCategory(userID int64, categoryName string) (*model.Category, error) {
    if categoryName == "" {
        // 空分类名 → 使用用户的第一个分类
        return h.store.FirstCategory(userID)
    }
    
    // 按标题查找
    category, err := h.store.CategoryByTitle(userID, categoryName)
    
    if category == nil {
        // 不存在则创建新分类
        category, err = h.store.CreateCategory(userID, &model.CategoryCreationRequest{Title: categoryName})
    }
    return category, nil
}
```

**OPML 导入的分类同步规则**：
1. 按分类名称匹配（`CategoryByTitle` 用的是精确匹配，**大小写敏感**）
2. 匹配到现有分类则复用
3. 匹配不到则创建新分类
4. 每个用户独立维护自己的分类集合

> **注意**：存在大小写匹配不一致的情况。`CategoryTitleExists`（验证重名用）用的是 `lower(title)=lower($2)` 大小写不敏感，但 `CategoryByTitle`（查找用）用的是 `title=$2` 精确匹配。数据库的 `unique (user_id, title)` 约束也是精确匹配的。

### 6.2 Google Reader API 中的"标签"

Google Reader API 将分类称为"标签"（labels/tags）。

#### 6.2.1 标签列表

文件：`internal/googlereader/handler.go:844-875`

```go
func (h *greaderHandler) tagListHandler(w http.ResponseWriter, r *http.Request) {
    categories, _ := h.store.Categories(userID)
    
    labelPrefix := fmt.Sprintf(userLabelPrefix, userID)
    for _, category := range categories {
        result.Tags = append(result.Tags, subscriptionCategoryResponse{
            ID:    labelPrefix + category.Title,
            Label: category.Title,
            Type:  "folder",
        })
    }
}
```

#### 6.2.2 移动订阅到分类

文件：`internal/googlereader/handler.go:488-500`

```go
func move(feedStream Stream, labelStream Stream, store *storage.Storage, userID int64) error {
    feed, _ := getFeed(feedStream, store, userID)
    category, _ := getOrCreateCategory(labelStream, store, userID)
    feed.Category.ID = category.ID
    return store.UpdateFeed(feed)
}
```

#### 6.2.3 重命名标签（分类）

文件：`internal/googlereader/handler.go:777-842`

```go
func (h *greaderHandler) renameTagHandler(w http.ResponseWriter, r *http.Request) {
    category, _ := h.store.CategoryByTitle(userID, source.ID)
    
    categoryModificationRequest := model.CategoryModificationRequest{
        Title: new(destination.ID),
    }
    categoryModificationRequest.Patch(category)
    h.store.UpdateCategory(category)
}
```

#### 6.2.4 删除标签（分类）

文件：`internal/googlereader/handler.go:736-775`

```go
func (h *greaderHandler) disableTagHandler(w http.ResponseWriter, r *http.Request) {
    // 将分类下的 feed 移到其他分类，然后删除
    err = h.store.RemoveAndReplaceCategoriesByName(userID, titles)
}
```

### 6.3 Fever API 中的"分组"

Fever API 将分类称为"groups"。

文件：`internal/fever/handler.go:75-101`

```go
func (h *feverHandler) handleGroups(w http.ResponseWriter, r *http.Request) {
    categories, _ := h.store.Categories(userID)
    
    for _, category := range categories {
        result.Groups = append(result.Groups, group{
            ID: category.ID,
            Title: category.Title,
        })
    }
}
```

---

## 七、各模块配合关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户请求入口                              │
├─────────────────┬─────────────────┬───────────────────────────┤
│   Web UI        │   REST API      │   Fever / Google Reader   │
│   (ui/)         │   (api/)        │   (fever/ googlereader/)  │
└────────┬────────┴────────┬────────┴─────────────┬─────────────┘
         │                 │                      │
         ▼                 ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      请求上下文解析                              │
│  - request.UserID(r)  获取当前用户 ID                           │
│  - request.IsAdminUser(r)  判断是否管理员                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    验证层 (validator/)                          │
│  - ValidateCategoryCreation                                    │
│  - ValidateCategoryModification                                │
│  - 检查同用户下标题是否重复                                      │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    存储层 (storage/category.go)                 │
│  - 所有操作都带 user_id 条件                                    │
│  - Category(userID, categoryID)                                │
│  - CreateCategory(userID, request)                             │
│  - UpdateCategory(category)  // WHERE id=$3 AND user_id=$4     │
│  - RemoveCategory(userID, categoryID)                          │
│  - Categories(userID)  // 不做 hide_globally 过滤               │
│  - CategoriesWithFeedCount(userID)  // 不做过滤                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          ▼                                       ▼
┌───────────────────────┐               ┌───────────────────────┐
│  全局视图查询         │               │  分类维度查询         │
│  (entry_query_builder)│               │  (category 详情)      │
│  - WithGloballyVisible()              │  - 直接按 ID +        │
│  - 过滤 hide_globally                 │    user_id 查询        │
│  - 分类和 feed 都过滤                  │  - 看到所有内容        │
└───────────────────────┘               └───────────────────────┘
```

---

## 八、关键规则总结

### 8.1 谁能看到分类？

| 场景 | 能看到的分类 | hide_globally 的影响 |
|------|-------------|---------------------|
| 分类列表页 | 当前用户的所有分类 | 不影响，都能看到 |
| 分类详情页 | 当前用户的该分类 | 不影响，能看到 |
| 未读条目页（全局） | 当前用户的非隐藏分类 | 隐藏分类的条目不显示 |
| 全部条目页（全局） | 当前用户的非隐藏分类 | 隐藏分类的条目不显示 |
| 导航栏未读数 | 当前用户的非隐藏分类 | 隐藏分类的未读不计入 |
| 其他用户 | 什么都看不到 | 完全隔离 |

### 8.2 谁能编辑分类？

| 角色 | 编辑权限 |
|------|---------|
| 分类所有者 | 完全控制（创建/编辑/删除） |
| 其他普通用户 | 无权限 |
| 管理员 | 无权限（管理员只能管用户，不能管用户的分类） |

### 8.3 订阅复制规则（OPML 导入）

| 情况 | 处理方式 |
|------|---------|
| 分类名称匹配到已有分类 | 复用现有分类 |
| 分类名称未匹配到 | 创建新分类 |
| 空分类名称 | 使用用户的第一个分类 |
| Feed URL 已存在 | 跳过不重复导入 |
| 不同用户同名分类 | 各自独立，互不影响 |

---

## 九、常见误解澄清

1. **误解**：`hide_globally` 是跨用户可见性控制  
   **事实**：只是当前用户的全局视图过滤，不影响其他用户

2. **误解**：管理员能看到所有用户的分类  
   **事实**：管理员仅能管理用户账号，不能访问用户的分类和订阅

3. **误解**：OPML 导入会关联到原分类  
   **事实**：只按名称匹配，每个用户的分类都是独立实体

4. **误解**：分类可以在用户间共享  
   **事实**：完全没有共享机制，每个用户的分类完全独立

---

## 十、补充说明与注意事项

### 10.1 UI 创建分类时 HideGlobally 的处理

**注意一个实现细节**：UI 层创建分类时没有传递 `HideGlobally` 字段。

文件：`internal/ui/category_save.go:34`

```go
categoryCreationRequest := &model.CategoryCreationRequest{Title: categoryForm.Title}
// 只传了 Title，没传 HideGlobally
```

而 UI 层更新分类时是正确传递的：

文件：`internal/ui/category_update.go:47-50`

```go
categoryRequest := &model.CategoryModificationRequest{
    Title:        new(categoryForm.Title),
    HideGlobally: new(categoryForm.HideGlobally),
}
```

**影响**：通过 Web UI 创建分类时，即使表单中有 `hide_globally` 复选框，该值也不会被保存，分类创建后 `hide_globally` 始终为 `false`。需要进入编辑页面再次设置才能生效。

**对比**：API 层的创建是正确的，会传递 `HideGlobally`。

文件：`internal/api/category_handlers.go:23-32`

```go
var categoryCreationRequest model.CategoryCreationRequest
json_parser.NewDecoder(r.Body).Decode(&categoryCreationRequest)
// 直接从请求体解析，包含 HideGlobally
```

### 10.2 MarkGloballyVisibleFeedsAsRead 的检查范围

"标记全部已读"操作只检查 **feed 级别**的 `hide_globally`，不检查 **category 级别**的。

文件：`internal/storage/entry.go:551-579`

```sql
UPDATE entries
SET status=$1, changed_at=now()
FROM feeds
WHERE entries.feed_id = feeds.id
  AND entries.user_id=$2
  AND entries.status=$3
  AND feeds.hide_globally=$4  -- 只检查了 feed 的 hide_globally
```

**对比其他全局可见性检查**：

| 检查点 | 检查 feed.hide_globally | 检查 category.hide_globally |
|--------|------------------------|----------------------------|
| 导航栏未读数 | ✅ 是 | ✅ 是 |
| WithGloballyVisible() | ✅ 是 | ✅ 是 |
| MarkGloballyVisibleFeedsAsRead | ✅ 是 | ❌ 否 |

**潜在影响**：如果一个分类设置了 `hide_globally=true`，但其下的 feed 没有设置，那么"全部标记已读"会把该分类下的条目也标记为已读。这可能不符合用户预期，因为用户可能认为"隐藏的分类"应该被完全排除在全局操作之外。

### 10.3 Google Reader API 中分类的特殊处理

Google Reader API 将分类称为"标签"（label/tag），有以下特点：

#### 10.3.1 标签命名空间

标签 ID 格式：`user/<user_id>/label/<category_title>`

文件：`internal/googlereader/handler.go:869-874`

```go
labelPrefix := fmt.Sprintf(userLabelPrefix, userID)
for _, category := range categories {
    result.Tags = append(result.Tags, subscriptionCategoryResponse{
        ID:    labelPrefix + category.Title,
        Label: category.Title,
        Type:  "folder",
    })
}
```

#### 10.3.2 getOrCreateCategory 策略

文件：`internal/googlereader/handler.go:402-413`

```go
func getOrCreateCategory(streamCategory Stream, store *storage.Storage, userID int64) (*model.Category, error) {
    switch {
    case streamCategory.ID == "":
        return store.FirstCategory(userID)      // 空 ID → 第一个分类
    case store.CategoryTitleExists(userID, streamCategory.ID):
        return store.CategoryByTitle(userID, streamCategory.ID)  // 存在 → 返回
    default:
        return store.CreateCategory(userID, &model.CategoryCreationRequest{
            Title: streamCategory.ID,  // 不存在 → 创建
        })
    }
}
```

**与 OPML 导入的 resolveCategory 对比**：

| 场景 | OPML 导入 (`resolveCategory`) | Google Reader API (`getOrCreateCategory`) |
|------|------------------------------|------------------------------------------|
| 空名称 | 使用第一个分类 | 使用第一个分类 |
| 检查存在性 | 无单独检查，直接 `CategoryByTitle` 查找 | 先用 `CategoryTitleExists` 检查 |
| 查找方式 | `CategoryByTitle` 精确匹配 | `CategoryByTitle` 精确匹配 |
| 不存在时 | 创建新分类 | 创建新分类 |
| 大小写检查 | 查找时精确匹配 | 存在性检查大小写不敏感，查找时精确匹配 |

> **注意**：`getOrCreateCategory` 存在一个潜在的不一致：用 `CategoryTitleExists`（大小写不敏感）检查存在，但用 `CategoryByTitle`（大小写敏感）查找。理论上可能出现"检查时认为存在但实际查不到"的边界情况。

两者行为基本一致，核心逻辑都是按名称匹配、不存在则创建。

#### 10.3.3 重命名标签

文件：`internal/googlereader/handler.go:777-842`

重命名标签本质上就是重命名分类。如果目标名称已存在，则返回错误。

#### 10.3.4 删除标签

文件：`internal/googlereader/handler.go:736-775`

删除标签（分类）时，会将该分类下的所有 feed 移到用户的第一个剩余分类，然后删除该分类。使用的是 `RemoveAndReplaceCategoriesByName` 方法。

### 10.4 Fever API 中的分类

Fever API 将分类称为"groups"，直接使用分类 ID 和标题，没有特殊的命名空间转换。

文件：`internal/fever/handler.go:75-101`

```go
for _, category := range categories {
    result.Groups = append(result.Groups, group{
        ID: category.ID,
        Title: category.Title,
    })
}
```

**注意**：Fever API 的 groups 列表不做 `hide_globally` 过滤，返回用户的所有分类。

---

## 十一、数据流完整示例

### 11.1 示例：用户 A 导出 OPML，用户 B 导入

```
用户 A 的分类：
  ├── Tech (id=10, hide_globally=false)
  │   └── Feed X
  └── News (id=20, hide_globally=true)
      └── Feed Y

        │
        ▼  OPML 导出 (handler.Export)
        │
        ▼  导出的是分类名称字符串，不是 ID
        │
  [Tech, News] 作为 categoryName 字段写入 OPML

        │
        ▼  用户 B 导入 OPML (handler.Import)
        │
        ▼  resolveCategory 按名称匹配
        │
用户 B 现有分类：
  └── Tech (id=5, hide_globally=true)

        │
        ▼
        │
  • Tech 匹配成功 → 复用用户 B 的 Tech 分类 (id=5)
  • News 不存在 → 创建新分类 News (id=?, hide_globally=false)
```

**关键点**：
1. 导出时不导出 `hide_globally` 属性（OPML 中只有 feed 的 HideGlobally，没有分类的）
2. 导入时新建的分类 `hide_globally` 默认为 `false`
3. 同名分类的属性不会被覆盖，保持导入用户自己的设置

### 11.2 示例：隐藏分类的可见性

```
用户的分类：
  ├── 工作 (hide_globally=false)
  │   └── Feed A (hide_globally=false) → 3 条未读
  └── 摸鱼 (hide_globally=true)
      ├── Feed B (hide_globally=false) → 5 条未读
      └── Feed C (hide_globally=true)  → 2 条未读
```

**各场景的可见性**：

| 场景 | 可见的未读数 | 说明 |
|------|-------------|------|
| 导航栏未读数 | 3 条 | 只算 Feed A 的，因为摸鱼分类和 Feed C 都隐藏了 |
| 未读页面 | 3 条 | 同上 |
| "全部标记已读" | 会标记 Feed A + Feed B 的条目 | 只检查 feed.hide_globally，不检查 category.hide_globally |
| 分类列表页 | 2 个分类都能看到 | 分类列表不过滤 hide_globally |
| 进入"摸鱼"分类 | 7 条未读（5+2） | 分类详情页显示所有内容 |
| 其他用户 | 0 条 | 完全不可见 |

---

## 十二、分类删除时的级联处理

### 12.1 核心前提：没有"已订阅用户"的概念

首先需要再次强调：**Miniflux 中每个用户的 feed 都是独立副本**，不存在"多个用户订阅同一个 feed 实例"的情况。

当用户 A 导入一个 feed 时，会在 `feeds` 表创建一条属于 A 的记录；用户 B 导入同一个 feed URL 时，会创建另一条独立的属于 B 的记录。两者除了 URL 相同外，没有任何关联。

因此，"分类被删除时已订阅用户的级联处理"这个问题的答案是：**不存在跨用户影响，每个用户的数据完全隔离**。

### 12.2 两种删除策略

代码中提供了两种删除分类的方式，行为差异很大：

#### 12.2.1 RemoveCategory：直接删除 + 数据库级联清理

文件：`internal/storage/category.go:224-242`

```go
func (s *Storage) RemoveCategory(userID, categoryID int64) error {
    query := `DELETE FROM categories WHERE id = $1 AND user_id = $2`
    result, err := s.db.Exec(query, categoryID, userID)
    // ...
}
```

**工作原理**：
- 直接执行 `DELETE FROM categories`
- 依赖数据库外键约束 `ON DELETE CASCADE` 自动清理
- 整个过程**没有显式事务**，由数据库保证原子性

**级联清理路径**（由数据库自动完成）：

```
删除 categories 记录
    ↓  ON DELETE CASCADE (feeds.category_id → categories.id)
删除该分类下的所有 feeds 记录
    ↓  ON DELETE CASCADE (entries.feed_id → feeds.id)
删除这些 feed 下的所有 entries 记录
    ↓  ON DELETE CASCADE (enclosures.entry_id → entries.id)
删除这些 entry 下的所有 enclosures 记录
    ↓  ON DELETE CASCADE (feed_icons.feed_id → feeds.id)
删除这些 feed 的图标关联
```

**数据库约束定义**（`internal/database/migrations.go:56-72`）：

```sql
CREATE TABLE feeds (
    -- ...
    foreign key (user_id) references users(id) on delete cascade,
    foreign key (category_id) references categories(id) on delete cascade
);

CREATE TABLE entries (
    -- ...
    foreign key (feed_id) references feeds(id) on delete cascade
);
```

**调用者**：
- Web UI：`internal/ui/category_remove.go:32`
- REST API：`internal/api/category_handlers.go:149`

**结果**：分类和其下的所有 feed、entry、enclosure 都被**物理删除**，不保留任何副本。

#### 12.2.2 RemoveAndReplaceCategoriesByName：移动 feed 后再删除

文件：`internal/storage/category.go:244-290`

```go
func (s *Storage) RemoveAndReplaceCategoriesByName(userid int64, titles []string) error {
    tx, err := s.db.Begin()
    // 1. 检查删除后至少保留 1 个分类
    // 2. 将待删除分类下的 feed 移到用户的第一个分类
    // 3. 删除分类
    tx.Commit()
}
```

**完整执行流程**（在一个事务内）：

1. **检查至少保留一个分类**：
   ```sql
   SELECT count(*) FROM categories WHERE user_id = $1 and title != ANY($2)
   ```
   如果剩余分类数 < 1，回滚事务并返回错误。

2. **移动 feed 到第一个剩余分类**：
   ```sql
   WITH d_cats AS (SELECT id FROM categories WHERE user_id = $1 AND title = ANY($2))
   UPDATE feeds
   SET category_id = (
       SELECT id FROM categories
       WHERE user_id = $1 AND id NOT IN (SELECT id FROM d_cats)
       ORDER BY title ASC LIMIT 1
   )
   WHERE user_id = $1 AND category_id IN (SELECT id FROM d_cats)
   ```

3. **删除分类**：
   ```sql
   DELETE FROM categories WHERE user_id = $1 AND title = ANY($2)
   ```

4. **提交事务**

**调用者**：
- Google Reader API：`internal/googlereader/handler.go:768`（删除标签时调用）

**结果**：feed 被**保留**，只是移动到其他分类；分类本身被删除。

### 12.3 两种策略对比

| 维度 | RemoveCategory | RemoveAndReplaceCategoriesByName |
|------|---------------|----------------------------------|
| 事务 | 无（依赖数据库） | 显式事务包裹整个操作 |
| feed 处理 | 级联删除 | 移动到第一个剩余分类 |
| entry 处理 | 级联删除 | 保留（随 feed 移动） |
| 调用方 | Web UI / REST API | Google Reader API |
| 数据丢失 | 完全丢失 | 不丢失 |
| 适用场景 | 用户主动删除分类 | 删除标签时保留订阅 |

### 12.4 关键结论

1. **不存在跨用户级联**：删除分类只会影响当前用户的数据，其他用户完全不受影响
2. **UI 删除是硬删除**：通过 Web UI 或 REST API 删除分类会级联删除该分类下的所有 feed 和 entry
3. **Google Reader API 删除是软移动**：删除标签会将 feed 移到其他分类，保留数据
4. **数据库级联是可靠保障**：所有级联删除由数据库外键约束保证，不会出现 orphan 记录

---

## 十三、订阅同步的事务边界

订阅同步（主要是 OPML 导入/导出）的事务处理比较分散，没有统一的大事务。以下是各环节的事务边界梳理：

### 13.1 OPML 导出：无事务

文件：`internal/reader/opml/handler.go:20-57`

```go
func (h *Handler) Export(userID int64) (string, error) {
    feeds, err := h.store.Feeds(userID)  // 单次查询
    // 序列化
    return serialize(subscriptions), nil
}
```

- **事务范围**：无
- **原子性**：`Feeds()` 是单次查询，数据库层面保证一致性
- **失败处理**：任何步骤失败直接返回错误，没有需要回滚的操作

### 13.2 OPML 导入：外层无事务，内层细粒度事务

文件：`internal/reader/opml/handler.go:59-95`

```go
func (h *Handler) Import(userID int64, data io.Reader) error {
    subscriptions, _ := parse(data)  // 解析
    
    for _, subscription := range subscriptions {
        if h.store.FeedURLExists(userID, subscription.FeedURL) {
            continue  // 已存在则跳过
        }
        
        // 解析/创建分类（无事务）
        category, err := h.resolveCategory(userID, subscription.CategoryName)
        
        // 验证（无事务）
        validateSubscription(userID, category.ID, h.store, subscription)
        
        // 创建 feed（内部有事务）
        feed := &model.Feed{...}
        applySubscriptionSettings(feed, subscription)
        h.store.CreateFeed(feed)
    }
    return nil
}
```

**事务边界分析**：

```
OPML Import 外层（无事务）
├─ parse(data)                     无事务
├─ for each subscription:
│   ├─ FeedURLExists()              无事务（单条查询）
│   ├─ resolveCategory()            无事务
│   │   ├─ FirstCategory() /        单条查询
│   │   ├─ CategoryByTitle() /      单条查询
│   │   └─ CreateCategory()         单条 INSERT，无事务
│   ├─ validateSubscription()       无事务
│   └─ CreateFeed()                 内部有细粒度事务
│       ├─ INSERT feed              无事务（单条语句）
│       └─ for each entry:
│           └─ BEGIN
│              ├─ entryExists()     检查是否存在
│              ├─ createEntry()     不存在则插入
│              └─ COMMIT
└─ return
```

#### 13.2.1 resolveCategory：无事务

文件：`internal/reader/opml/handler.go:97-119`

```go
func (h *Handler) resolveCategory(userID int64, categoryName string) (*model.Category, error) {
    if categoryName == "" {
        return h.store.FirstCategory(userID)  // 单条查询
    }
    
    category, err := h.store.CategoryByTitle(userID, categoryName)  // 单条查询
    
    if category == nil {
        // 创建新分类 - 单条 INSERT，无事务
        category, err = h.store.CreateCategory(userID, &model.CategoryCreationRequest{Title: categoryName})
    }
    return category, nil
}
```

#### 13.2.2 CreateFeed：外层无事务，entry 级有事务

文件：`internal/storage/feed.go:216-326`

```go
func (s *Storage) CreateFeed(feed *model.Feed) error {
    // 1. 插入 feed（单条 SQL，无事务）
    err := s.db.QueryRow(sql, ...).Scan(&feed.ID)
    
    // 2. 插入每个 entry（每个 entry 一个独立事务）
    for _, entry := range feed.Entries {
        tx, err := s.db.Begin()           // 每个 entry 开启事务
        
        entryExists, err := s.entryExists(tx, entry)
        if !entryExists {
            s.createEntry(tx, entry)
        }
        
        tx.Commit()
    }
    return nil
}
```

### 13.3 事务边界总结表

| 操作 | 事务范围 | 原子性粒度 | 失败影响 |
|------|---------|-----------|---------|
| OPML Export | 无 | - | 直接返回错误，无数据变更 |
| OPML Import 外层 | 无 | 每个 feed 独立 | 部分成功部分失败，已成功的 feed 保留 |
| resolveCategory | 无 | 单条 SQL | 失败不影响其他 |
| CreateCategory | 无 | 单条 INSERT | 失败不影响其他 |
| CreateFeed (feed 插入) | 无 | 单条 INSERT | 失败回滚（数据库自动） |
| CreateFeed (entry 插入) | 每个 entry 一个事务 | 单条 entry | 单个 entry 失败不影响 feed 和其他 entry |
| RemoveCategory | 无（数据库级联） | 整条删除链 | 要么全部删除，要么不删 |
| RemoveAndReplaceCategoriesByName | 整个操作一个事务 | 多个 SQL 原子执行 | 失败全部回滚 |
| CreateUser | 整个操作一个事务 | 用户+分类+集成设置 | 失败全部回滚 |

### 13.4 潜在问题与风险

#### 13.4.1 OPML 导入的部分成功问题

**场景**：导入 100 个 feed，前 50 个成功，第 51 个失败。

**结果**：前 50 个 feed 已永久保存，后面的不再处理。没有自动回滚。

**用户体验**：用户需要手动删除已导入的 feed，或者修复问题后重新导入（已存在的会被跳过）。

#### 13.4.2 CreateFeed 中 entry 事务的细粒度问题

每个 entry 单独开启事务，导入一个有 100 条 entry 的 feed 会执行 100 次事务提交，性能较低。

但这样设计的好处是某个 entry 解析异常不会影响整个 feed 的导入。

#### 13.4.3 resolveCategory 的竞态条件

在高并发下，两个请求同时导入同一个不存在的分类名：
1. 请求 A：`CategoryByTitle` → 不存在
2. 请求 B：`CategoryByTitle` → 不存在
3. 请求 A：`CreateCategory` → 成功
4. 请求 B：`CreateCategory` → 失败（数据库 `unique (user_id, title)` 约束）

但实际场景中 OPML 导入通常是单用户操作，这个问题影响很小。

### 13.5 设计意图分析

为什么 OPML 导入不包一个大事务？可能的考虑：

1. **失败可恢复**：部分成功比全部回滚更友好，用户可以修正问题后继续
2. **导入时间长**：OPML 导入需要抓取 feed，可能耗时很久，长事务会占用数据库连接
3. **幂等性**：`FeedURLExists` 检查保证重复导入不会重复创建，支持断点续导
4. **feed 创建可能失败**：网络抓取可能失败，不应该因为一个 feed 失败导致全部回滚

---

## 十四、移动大量订阅到新分类的事务隔离与锁竞争

### 14.1 移动订阅的三种路径

代码中存在三种不同的"移动订阅到分类"的操作路径，事务和锁行为差异很大：

| 路径 | 触发方式 | 单次移动数量 | 事务范围 |
|------|---------|-------------|---------|
| UpdateFeed | UI 编辑 feed / REST API / Google Reader | 1 个 feed | 单条 SQL，无显式事务 |
| RemoveAndReplaceCategoriesByName | Google Reader API 删除标签 | 多个 feed（分类下所有） | 一个事务包裹移动 + 删除 |
| 批量移动（无） | - | - | 不存在批量移动接口 |

### 14.2 路径一：UpdateFeed 单 feed 移动

**核心代码**：`internal/storage/feed.go:329-424`

```go
func (s *Storage) UpdateFeed(feed *model.Feed) (err error) {
    query := `
        UPDATE feeds
        SET ... category_id=$4 ...
        WHERE id=$40 AND user_id=$41
    `
    _, err = s.db.Exec(query, ..., feed.Category.ID, ..., feed.ID, feed.UserID)
}
```

**事务与锁分析**：

1. **无显式事务**：整条 `UPDATE` 是单条 SQL 语句，在数据库隐式事务内执行
2. **行级锁**：PostgreSQL 对 `feeds` 表中 `id=? AND user_id=?` 的单行加排他锁
3. **原子性**：要么成功要么失败，不会出现中间状态
4. **无分类表锁**：只更新 `feeds` 表的 `category_id` 字段，不修改 `categories` 表

**调用链示例（Google Reader API move）**：

文件：`internal/googlereader/handler.go:488-516`

```go
func move(feedStream Stream, labelStream Stream, store *storage.Storage, userID int64) error {
    feed, _ := getFeed(feedStream, store, userID)      // 1. SELECT feeds（读，无锁）
    category, _ := getOrCreateCategory(labelStream, store, userID)  // 2. SELECT / INSERT categories
    feedModification := model.FeedModificationRequest{
        CategoryID: &category.ID,
    }
    feedModification.Patch(feed)
    return store.UpdateFeed(feed)                     // 3. UPDATE feeds（行级锁）
}
```

**锁竞争风险**：

- 🔴 **读-写竞态**：`getFeed` 和 `UpdateFeed` 之间没有事务包裹，feed 可能在读取后被其他请求修改
- 🟢 **影响很小**：因为 `UpdateFeed` 用的是完整字段覆盖，即使有竞态，最后一次写会胜出
- 🟢 **无死锁风险**：单条 UPDATE，只锁一行

### 14.3 路径二：RemoveAndReplaceCategoriesByName 批量移动

**核心代码**：`internal/storage/category.go:244-290`

这是目前代码中**唯一的批量移动分类**操作，发生在删除分类/标签时。

```go
func (s *Storage) RemoveAndReplaceCategoriesByName(userid int64, titles []string) error {
    tx, err := s.db.Begin()                          // 开启事务
    
    // 1. 检查剩余分类数（SELECT ... count）
    // 2. 移动 feed 到第一个剩余分类（UPDATE feeds ...）
    // 3. 删除分类（DELETE FROM categories ...）
    
    return tx.Commit()
}
```

**完整 SQL 流程**（在一个事务内）：

```sql
-- 语句 1: 检查至少保留一个分类（共享锁）
SELECT count(*) FROM categories 
WHERE user_id = $1 AND title != ANY($2);

-- 语句 2: 批量移动 feed（写锁，影响多行）
WITH d_cats AS (SELECT id FROM categories WHERE user_id = $1 AND title = ANY($2))
UPDATE feeds
SET category_id = (
    SELECT id FROM categories 
    WHERE user_id = $1 AND id NOT IN (SELECT id FROM d_cats)
    ORDER BY title ASC LIMIT 1
)
WHERE user_id = $1 AND category_id IN (SELECT id FROM d_cats);

-- 语句 3: 删除分类
DELETE FROM categories WHERE user_id = $1 AND title = ANY($2);
```

**事务与锁分析**：

1. **显式事务**：三条 SQL 在一个事务内，原子性保证
2. **行级锁范围**：
   - 语句 2 会对所有被移动的 feed 行加排他锁
   - 语句 3 会对被删除的分类行加排他锁
3. **锁顺序**：先锁 feeds，再锁 categories（DELETE 时）
4. **一致性**：事务保证"移动 feed"和"删除分类"要么都成功要么都失败

**锁竞争与潜在死锁**：

- 🔴 **死锁风险**：如果两个请求同时执行删除分类操作，且分类顺序不同，可能出现死锁
  - 请求 A：删除分类 X → 锁 feeds → 锁 categories X
  - 请求 B：删除分类 Y → 锁 feeds → 锁 categories Y
  - 但因为都是 `DELETE FROM categories WHERE ... title = ANY(...)`，PostgreSQL 会按相同顺序加锁吗？实际上取决于执行计划
- 🟢 **实际风险低**：同一用户同时删除多个分类的场景很少见
- 🔴 **长时间锁**：如果分类下 feed 很多，`UPDATE feeds` 会锁很多行，持续时间较长

### 14.4 路径三：OPML 导入的隐式移动

OPML 导入时，如果 feed 已存在则直接跳过（`FeedURLExists` 检查），**不会**因为分类名不同而移动 feed。

文件：`internal/reader/opml/handler.go:72-76`

```go
if h.store.FeedURLExists(userID, subscription.FeedURL) {
    continue  // 已存在则跳过，不修改分类
}
```

**结论**：OPML 导入是"只新增、不修改"的语义，已存在的 feed 分类保持不变。

### 14.5 批量移动的缺失与影响

**现状**：Miniflux 没有"批量移动 N 个 feed 到指定分类"的 API 或 UI 操作。

**如果需要实现批量移动，锁竞争考量**：

```
方案 A: 循环调用 UpdateFeed
  ├─ 优点: 简单，每行一个短事务
  ├─ 缺点: 不是原子操作，可能部分成功部分失败
  └─ 锁粒度: 单行锁，竞争小

方案 B: 单条 SQL 批量 UPDATE
  ├─ 优点: 原子性好，一个事务搞定
  ├─ 缺点: 锁太多行，可能阻塞其他操作
  └─ 锁粒度: 多行锁，竞争大

方案 C: 分批 + 小事务
  ├─ 优点: 平衡原子性和并发性
  └─ 缺点: 实现复杂
```

### 14.6 关键结论

1. **单 feed 移动**：无显式事务，单行级锁，几乎无锁竞争
2. **删除分类时的批量移动**：有事务包裹，锁多行，理论上有死锁风险但实际罕见
3. **没有专门的批量移动接口**：所有批量移动都是删除分类的副作用
4. **OPML 导入不移动已有 feed**：只新增不修改，避免了复杂的分类变更事务

---

## 十五、OPML 导入导出下分类与可见度规则的兼容性

### 15.1 OPML 格式扩展：Miniflux 命名空间

Miniflux 在标准 OPML 基础上扩展了自定义命名空间，用于导出/导入额外的 feed 属性。

文件：`internal/reader/opml/serializer.go:37`

```go
opmlDocument.MinifluxNamespace = minifluxOPMLNamespace
```

OPML 中的分类用两级 `<outline>` 表示：
- 第一级 outline：分类（text 属性为分类名）
- 第二级 outline：feed 订阅

### 15.2 导出：哪些信息被导出

**导出代码**：`internal/reader/opml/serializer.go:48-79`

#### 15.2.1 分类级别的导出

只导出**分类名称**字符串，不导出分类的任何属性。

```go
for _, categoryName := range categories {
    category := opmlOutline{Text: categoryName, Outlines: ...}  // 只有名称！
    // ...
}
```

**分类级别导出的信息**：
| 属性 | 是否导出 | 说明 |
|------|---------|------|
| 分类名称（title） | ✅ 是 | 作为 outline 的 text 属性 |
| 分类 ID | ❌ 否 | 不导出，导入时按名称匹配 |
| 分类 hide_globally | ❌ 否 | **完全不导出** |
| 分类创建时间 | ❌ 否 | 不导出 |

#### 15.2.2 Feed 级别的导出

Feed 的属性导出比较完整，包括可见度设置。

```go
opmlOutline{
    Title:       subscription.Title,
    FeedURL:     subscription.FeedURL,
    // ... 标准 OPML字段 ...
    
    // Miniflux 扩展字段
    HideGlobally: subscription.HideGlobally,  // ✅ 导出
    // ... 其他扩展字段 ...
}
```

**Feed 级别导出的可见度相关信息**：
| 属性 | 是否导出 | 说明 |
|------|---------|------|
| feed 的 hide_globally | ✅ 是 | Miniflux 命名空间扩展字段 |
| 所属分类名称 | ✅ 是 | 通过 outline 层级关系体现 |
| 所属分类 ID | ❌ 否 | 不导出 |
| 分类的 hide_globally | ❌ 否 | 不在 feed 级别，也不在分类级别 |

### 15.3 导入：哪些信息被使用

**导入代码**：`internal/reader/opml/parser.go:30-65`

#### 15.3.1 分类的导入行为

分类名称通过 outline 层级解析出来，然后在 `resolveCategory` 中处理：

```go
func getSubscriptionsFromOutlines(outlines ..., category string) []subcription {
    for _, outline := range outlines {
        if outline.IsSubscription() {
            subscriptions = append(subscriptions, subcription{
                CategoryName: category,  // 分类名称
                // ...
                HideGlobally: outline.HideGlobally,  // feed 的 hide_globally
            })
        } else if outline.Outlines.HasChildren() {
            // 递归，分类名 = outline.GetTitle()
            subscriptions = append(subscriptions, 
                getSubscriptionsFromOutlines(outline.Outlines, outline.GetTitle())...)
        }
    }
}
```

**`resolveCategory` 的匹配规则**（`internal/reader/opml/handler.go:97-119`）：

| 场景 | 行为 |
|------|------|
| 分类名为空 | 使用用户的第一个分类 |
| 分类名已存在 | 复用现有分类（按标题精确匹配） |
| 分类名不存在 | 创建新分类，使用默认设置（`hide_globally=false`） |

#### 15.3.2 Feed 的导入行为

Feed 级别的 `hide_globally` 会被正确导入。

**应用设置的代码**：`internal/reader/opml/handler.go:82-94`

```go
feed := &model.Feed{
    FeedURL:      subscription.FeedURL,
    Category: &model.Category{ID: category.ID},
    // ...
}
applySubscriptionSettings(feed, subscription)  // 应用包括 HideGlobally 在内的设置
```

`applySubscriptionSettings` 函数会设置：
- `HideGlobally`
- `Crawler`
- `Disabled`
- `ScraperRules` / `RewriteRules` 等各种规则

### 15.4 可见度规则的兼容性矩阵

| 维度 | 导出 | 导入 | 兼容性说明 |
|------|------|------|-----------|
| 分类名称 | ✅ 导出 | ✅ 导入 | 完全兼容，按名称匹配 |
| 分类 hide_globally | ❌ 不导出 | ❌ 不导入 | 不兼容，新建分类默认为 false |
| 分类 ID | ❌ 不导出 | ❌ 不适用 | 设计如此，避免 ID 冲突 |
| Feed 的 hide_globally | ✅ 导出 | ✅ 导入 | 完全兼容，Miniflux 扩展字段 |
| 标准 OPML 阅读器 | ✅ 兼容 | ✅ 兼容 | 扩展字段会被忽略 |

### 15.5 实际场景分析

#### 场景一：用户 A 导出 → 用户 B 导入

```
用户 A 的分类结构：
  科技资讯 (hide_globally = false)
    ├── TechCrunch (hide_globally = false)
    └── 36氪 (hide_globally = true)
  摸鱼 (hide_globally = true)
    └── 知乎日报 (hide_globally = false)

          │
          ▼  导出 OPML
          │
  导出内容包含：
  - 分类名："科技资讯"、"摸鱼"
  - feed 属性：TechCrunch(false)、36氪(true)、知乎日报(false)
  - 不包含：分类的 hide_globally 属性

          │
          ▼  用户 B 导入
          │
用户 B 的结果：
  科技资讯 (hide_globally = false)  ← 新建，默认为 false
    ├── TechCrunch (hide_globally = false)  ← 正确导入
    └── 36氪 (hide_globally = true)         ← 正确导入
  摸鱼 (hide_globally = false)      ← 新建，默认为 false！丢失了隐藏属性
    └── 知乎日报 (hide_globally = false)    ← 正确导入
```

**关键发现**：
- ✅ Feed 级别的 `hide_globally` 完整保留
- ❌ 分类级别的 `hide_globally` **丢失**，新分类默认为 `false`
- 🔍 原因：OPML 的分类 outline 上没有 Miniflux 扩展属性，只有 feed outline 上有

#### 场景二：用户自己备份恢复

```
用户原始分类：
  工作 (hide_globally = true)
    └── 公司内网 RSS (hide_globally = false)

          │
          ▼  导出备份
          ▼  重新导入
          │
结果：
  工作 (hide_globally = false)  ← 从隐藏变成可见了！
    └── 公司内网 RSS (hide_globally = false)
```

**影响**：用户备份恢复后，原来隐藏的分类会变成可见的，全局未读数会突然增加。

### 15.6 为什么分类 hide_globally 不导出？技术原因分析

**代码层面**：
1. OPML 序列化时，分类只有 `Text: categoryName` 一个属性（`serializer.go:49`）
2. `subcription` 结构体有 `HideGlobally` 字段，但它是 feed 的属性
3. 分类级别的 OPML outline 没有对应的 Miniflux 扩展属性

**设计层面的可能原因**：
1. **OPML 标准限制**：OPML 的 outline 原本是为 feed 设计的，分类只是分组层级
2. **优先级低**：分类的 hide_globally 不如 feed 的重要，很少有人设置
3. **兼容性**：在分类 outline 上加自定义属性可能导致某些 OPML 阅读器解析出错

### 15.7 关键结论

1. **Feed 级可见度完整兼容**：`hide_globally` 在导出导入时完整保留，依赖 Miniflux 自定义命名空间
2. **分类级可见度不兼容**：分类的 `hide_globally` 属性在 OPML 中不导出也不导入，新分类默认为 `false`
3. **分类按名称匹配**：导入时通过分类名查找或创建，不使用 ID
4. **已有 feed 不受影响**：导入时已存在的 feed 会被跳过，其分类和可见度设置保持不变
5. **备份恢复有信息丢失**：用户自己备份恢复时，分类的隐藏属性会丢失
