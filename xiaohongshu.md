# xiaohongshu-mcp 项目功能分析

## 项目概述

**项目名称**: xiaohongshu-mcp
**项目类型**: MCP (Model Context Protocol) 服务器 + HTTP API 服务
**技术栈**: Go 1.24.0 + 无头浏览器自动化 (go-rod)
**核心原理**: 通过浏览器自动化模拟真实用户操作小红书网站，避免官方API限制

## 一、登录管理功能

### 1.1 功能点
- **检查登录状态**: 检测当前是否已登录小红书账号
- **获取登录二维码**: 生成小红书登录二维码图片
- **等待登录完成**: 监听登录状态变化
- **删除cookies重置登录**: 清除登录状态

### 1.2 实现方式
**核心文件**: `xiaohongshu/login.go`

#### 1.2.1 检查登录状态 (`CheckLoginStatus`)
- **调用地址**: `https://www.xiaohongshu.com/explore`
- **HTTP方法**: GET (浏览器导航)
- **实现逻辑**:
  1. 导航到小红书探索页
  2. 检查是否存在用户频道元素: `.main-container .user .link-wrapper .channel`
  3. 根据元素存在与否判断登录状态
- **代码位置**: `login.go:19-35`

#### 1.2.2 获取登录二维码 (`FetchQrcodeImage`)
- **调用地址**: `https://www.xiaohongshu.com/explore`
- **HTTP方法**: GET (浏览器导航)
- **实现逻辑**:
  1. 导航到小红书探索页（会自动触发二维码弹窗）
  2. 等待二维码弹窗出现（选择器: `.login-container`）
  3. 获取二维码图片的 `src` 属性值（选择器: `.login-container .qrcode-img`）
  4. 将图片转换为 Base64 编码返回
- **元素选择器**:
  - 二维码弹窗容器: `.login-container`
  - 二维码图片: `.login-container .qrcode-img`
  - 登录状态检查: `.main-container .user .link-wrapper .channel`
- **代码位置**: `login.go:59-83`

#### 1.2.3 登录功能 (`Login`)
- **调用地址**: `https://www.xiaohongshu.com/explore`
- **HTTP方法**: GET (浏览器导航)
- **实现逻辑**:
  1. 导航到小红书探索页
  2. 检查是否已登录（避免重复登录）
  3. 触发登录流程，等待用户扫码
- **代码位置**: `login.go:37-57`

### 1.3 技术特点
- 使用浏览器自动化进行状态检测，不依赖API接口
- 通过DOM元素存在性判断登录状态
- 二维码图片直接从页面提取，无需单独请求

## 二、内容发布功能

### 2.1 图文发布功能

#### 2.1.1 功能点
- **发布图文内容**: 支持标题、正文、图片、话题标签
- **图片上传**: 支持本地图片和网络图片URL
- **自动标签处理**: 自动添加 `#` 前缀的话题标签

#### 2.1.2 实现方式
**核心文件**: `xiaohongshu/publish.go`

- **发布地址**: `https://creator.xiaohongshu.com/publish/publish?source=official`
- **HTTP方法**: GET (浏览器导航)
- **发布流程**:
  1. 导航到创作者发布页面
  2. 点击"上传图文"标签页
  3. 上传图片文件（通过 `.upload-input` 文件输入框）
  4. 填写标题（输入框选择器: `input[placeholder*="起个吸引人的标题"]`）
  5. 填写正文内容（富文本编辑器）
  6. 添加话题标签（以 `#` 开头）
  7. 点击提交按钮
- **代码位置**: 主要逻辑在 `publish.go:34-266`

#### 2.1.3 关键函数
- `NewPublishImageAction()`: 初始化发布动作
- `Publish()`: 执行发布流程
- `uploadImages()`: 处理图片上传
- `submitPublish()`: 提交发布内容

### 2.2 视频发布功能

#### 2.2.1 功能点
- **发布视频内容**: 支持本地视频文件上传
- **视频处理等待**: 自动等待视频转码处理完成
- **元数据填写**: 标题、描述、话题标签

#### 2.2.2 实现方式
**核心文件**: `xiaohongshu/publish_video.go`

- **发布地址**: `https://creator.xiaohongshu.com/publish/publish?source=official`
- **HTTP方法**: GET (浏览器导航)
- **发布流程**:
  1. 导航到创作者发布页面
  2. 点击"上传视频"标签页
  3. 上传视频文件（通过 `input[type='file']` 文件输入框）
  4. 等待视频处理完成（检查发布按钮是否可点击）
  5. 填写标题和正文内容
  6. 添加话题标签
  7. 点击发布按钮
- **代码位置**: 主要逻辑在 `publish_video.go:24-148`

#### 2.2.3 技术特点
- 视频上传后需要等待转码处理，有超时机制
- 通过检查发布按钮状态判断视频是否处理完成
- 支持大文件上传，有进度提示

### 2.3 发布功能通用特性
1. **图片/视频下载**: 网络URL的媒体文件会自动下载到本地再上传
2. **话题标签处理**: 自动添加 `#` 前缀，等待联想下拉框
3. **错误重试**: 上传失败时自动重试
4. **超时控制**: 每个步骤都有合理的超时时间

## 三、内容浏览功能

### 3.1 首页推荐列表

#### 3.1.1 功能点
- **获取首页Feed流**: 小红书首页推荐内容
- **提取完整数据**: 包括笔记ID、用户信息、互动数据等
- **获取安全令牌**: 提取 `xsecToken` 用于后续请求

#### 3.1.2 实现方式
**核心文件**: `xiaohongshu/feeds.go`

- **调用地址**: `https://www.xiaohongshu.com`
- **HTTP方法**: GET (浏览器导航)
- **数据提取**:
  1. 导航到小红书首页
  2. 执行JavaScript获取数据: `window.__INITIAL_STATE__.feed.feeds`
  3. 解析Feed列表数据
  4. 提取 `xsecToken` 用于后续API调用
- **代码位置**: `feeds.go:20-43`

#### 3.1.3 数据结构
```javascript
window.__INITIAL_STATE__.feed.feeds = [
  {
    id: "笔记ID",
    noteCard: {
      user: {nickname, userId, avatar},
      interactInfo: {liked, likedCount, collected, ...},
      // ... 其他字段
    },
    xsecToken: "安全令牌"
  }
]
```

### 3.2 搜索功能

#### 3.2.1 功能点
- **关键词搜索**: 根据关键词搜索小红书内容
- **多维度筛选**: 支持5种筛选条件
- **排序选项**: 多种排序方式

#### 3.2.2 实现方式
**核心文件**: `xiaohongshu/search.go`

- **搜索地址**: `https://www.xiaohongshu.com/search_result?keyword={keyword}&source=web_explore_feed`
- **HTTP方法**: GET (浏览器导航)
- **URL构建**: 在 `search.go:242-251` 构建搜索URL
- **数据提取**: 从 `window.__INITIAL_STATE__.search.feeds` 获取搜索结果

#### 3.2.3 筛选条件
1. **排序依据**: 综合/最新/最多点赞/最多评论/最多收藏
2. **笔记类型**: 不限/视频/图文
3. **发布时间**: 不限/一天内/一周内/半年内
4. **搜索范围**: 不限/已看过/未看过/已关注
5. **位置距离**: 不限/同城/附近

#### 3.2.4 实现细节
- 筛选通过点击页面筛选面板的对应选项实现
- 需要等待筛选结果刷新
- 支持分页加载更多结果

### 3.3 笔记详情

#### 3.3.1 功能点
- **获取笔记完整内容**: 正文、图片、视频等
- **获取互动数据**: 点赞数、收藏数、评论数等
- **获取评论列表**: 笔记下的所有评论

#### 3.3.2 实现方式
**核心文件**: `xiaohongshu/feed_detail.go`

- **详情页地址**: `https://www.xiaohongshu.com/explore/{feedID}?xsec_token={xsecToken}&xsec_source=pc_feed`
- **HTTP方法**: GET (浏览器导航)
- **URL构建**: `feed_detail.go:72-74` 使用 `makeFeedDetailURL()` 函数
- **数据提取**:
  1. 笔记详情: `window.__INITIAL_STATE__.note.noteDetailMap[feedID].note`
  2. 评论列表: `window.__INITIAL_STATE__.note.noteDetailMap[feedID].comments`

### 3.4 用户主页

#### 3.4.1 功能点
- **获取用户基本信息**: 昵称、头像、简介等
- **获取用户统计数据**: 关注数、粉丝数、获赞数
- **获取用户帖子列表**: 用户发布的所有笔记

#### 3.4.2 实现方式
**核心文件**: `xiaohongshu/user_profile.go`

- **用户主页地址**: `https://www.xiaohongshu.com/user/profile/{userID}?xsec_token={xsecToken}&xsec_source=pc_note`
- **HTTP方法**: GET (浏览器导航)
- **URL构建**: `user_profile.go:103-105` 使用 `makeUserProfileURL()` 函数
- **数据提取**:
  1. 用户信息: `window.__INITIAL_STATE__.user.userPageData`
  2. 用户帖子: `window.__INITIAL_STATE__.user.notes`

#### 3.4.3 访问方式
1. **直接访问**: 通过用户ID构建URL直接访问
2. **侧边栏导航**: 通过 `GetMyProfileViaSidebar()` 函数，点击侧边栏"我"链接进入个人主页

## 四、互动功能

### 4.1 评论功能

#### 4.1.1 功能点
- **发表评论**: 在指定笔记下发表评论
- **评论输入**: 支持富文本评论内容

#### 4.1.2 实现方式
**核心文件**: `xiaohongshu/comment_feed.go`

- **详情页地址**: `https://www.xiaohongshu.com/explore/{feedID}?xsec_token={xsecToken}&xsec_source=pc_feed`
- **HTTP方法**: GET (浏览器导航)
- **评论流程**:
  1. 导航到笔记详情页
  2. 点击评论输入框: `div.input-box div.content-edit span`
  3. 输入评论内容: `div.input-box div.content-edit p.content-input`
  4. 点击提交按钮: `div.bottom button.submit`
- **代码位置**: `comment_feed.go` 主要逻辑

### 4.2 点赞/收藏功能

#### 4.2.1 功能点
- **点赞/取消点赞**: 对笔记进行点赞操作
- **收藏/取消收藏**: 对笔记进行收藏操作
- **状态检查**: 避免重复操作

#### 4.2.2 实现方式
**核心文件**: `xiaohongshu/like_favorite.go`

- **详情页地址**: `https://www.xiaohongshu.com/explore/{feedID}?xsec_token={xsecToken}&xsec_source=pc_feed`
- **HTTP方法**: GET (浏览器导航)

#### 4.2.3 点赞实现
- **选择器**: `.interact-container .left .like-lottie`
- **状态检查**: 从 `__INITIAL_STATE__.note.noteDetailMap[feedID].note.interactInfo.liked` 读取当前状态
- **操作逻辑**: 先检查是否已点赞，避免重复操作

#### 4.2.4 收藏实现
- **选择器**: `.interact-container .left .reds-icon.collect-icon`
- **状态检查**: 从 `__INITIAL_STATE__.note.noteDetailMap[feedID].note.interactInfo.collected` 读取当前状态
- **操作逻辑**: 先检查是否已收藏，避免重复操作

## 五、技术实现总结

### 5.1 核心技术
1. **浏览器自动化**: 使用 `go-rod` 库进行无头浏览器操作
2. **数据提取**: 从 `window.__INITIAL_STATE__` 全局变量提取数据
3. **状态管理**: 通过DOM元素状态和全局变量状态避免重复操作
4. **错误处理**: 自定义错误类型和重试机制
5. **超时控制**: 合理的操作超时时间设置

### 5.2 数据流设计
```
浏览器导航 → 页面加载 → 数据提取 → 状态检查 → 操作执行 → 结果返回
```

### 5.3 关键设计模式
1. **Action模式**: 每个功能封装为独立的Action对象
2. **Builder模式**: URL和参数通过Builder函数构建
3. **状态检查模式**: 操作前先检查当前状态
4. **重试机制**: 失败操作自动重试

## 六、URL地址汇总表

| 功能模块 | 具体功能 | URL地址 | HTTP方法 | 参数说明 |
|---------|---------|---------|----------|---------|
| **登录管理** | 检查登录状态 | `https://www.xiaohongshu.com/explore` | GET | 无 |
| | 获取登录二维码 | `https://www.xiaohongshu.com/explore` | GET | 无 |
| **内容发布** | 图文发布 | `https://creator.xiaohongshu.com/publish/publish?source=official` | GET | source=official |
| | 视频发布 | `https://creator.xiaohongshu.com/publish/publish?source=official` | GET | source=official |
| **内容浏览** | 首页推荐 | `https://www.xiaohongshu.com` | GET | 无 |
| | 搜索内容 | `https://www.xiaohongshu.com/search_result?keyword={keyword}&source=web_explore_feed` | GET | keyword: 搜索词<br>source: web_explore_feed |
| | 笔记详情 | `https://www.xiaohongshu.com/explore/{feedID}?xsec_token={token}&xsec_source=pc_feed` | GET | feedID: 笔记ID<br>xsec_token: 安全令牌<br>xsec_source: pc_feed |
| | 用户主页 | `https://www.xiaohongshu.com/user/profile/{userID}?xsec_token={token}&xsec_source=pc_note` | GET | userID: 用户ID<br>xsec_token: 安全令牌<br>xsec_source: pc_note |
| **互动功能** | 发表评论 | `https://www.xiaohongshu.com/explore/{feedID}?xsec_token={token}&xsec_source=pc_feed` | GET | feedID: 笔记ID<br>xsec_token: 安全令牌<br>xsec_source: pc_feed |
| | 点赞/收藏 | `https://www.xiaohongshu.com/explore/{feedID}?xsec_token={token}&xsec_source=pc_feed` | GET | feedID: 笔记ID<br>xsec_token: 安全令牌<br>xsec_source: pc_feed |

## 七、项目特点

### 7.1 优势
1. **规避API限制**: 通过浏览器自动化模拟真实用户，不受官方API限制
2. **功能完整**: 覆盖登录、发布、浏览、搜索、互动等全功能
3. **双协议支持**: 同时提供MCP协议和HTTP REST API
4. **生态完善**: 支持多种AI助手和开发工具集成
5. **状态持久化**: Cookie持久保存，避免频繁登录

### 7.2 技术挑战与解决方案
1. **反爬虫机制**: 通过模拟真实用户行为规避检测
2. **动态加载**: 使用等待机制确保页面完全加载
3. **状态同步**: 通过全局变量状态检查确保操作正确性
4. **网络不稳定**: 实现重试机制和超时控制

### 7.3 适用场景
1. **内容创作者**: 自动化发布和管理小红书内容
2. **社交媒体运营**: 批量内容发布和监控
3. **数据采集**: 小红书内容分析和数据挖掘
4. **AI助手集成**: 通过自然语言操作小红书
5. **工作流自动化**: 与其他系统集成实现自动化流程

---

**文档最后更新**: 2025-12-04
**分析基于版本**: 项目最新提交 (6bef535)
**分析工具**: Claude Code + Explore Agent