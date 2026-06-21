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
