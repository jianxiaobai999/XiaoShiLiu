# 网络图片上传功能说明

## 功能概述

在 `MultiImageUpload` 组件中新增了"网络图片"上传功能，允许用户通过输入 URL 的方式添加网络图片到上传队列。

## 新增文件

- `src/components/modals/NetworkImageModal.vue` - 网络图片上传模态框组件
- `src/assets/icons/link.svg` - 链接图标

## 修改文件

- `src/components/MultiImageUpload.vue` - 添加网络图片按钮和处理逻辑

## 功能特性

### 1. 界面一致性
- ✅ 使用与现有上传组件相同的网格布局
- ✅ 保持相同的样式、字体、颜色和间距
- ✅ 网络图片按钮与本地上传按钮并排显示

### 2. 核心功能
- ✅ 支持单个图片 URL 输入
- ✅ 支持批量粘贴多行 URL（每行一个链接）
- ✅ 支持拖拽文本自动识别并提取图片链接
- ✅ 最多添加 9 张图片（与本地上传限制一致）

### 3. 智能验证
- ✅ 自动检测 URL 格式是否有效
- ✅ 支持 JPG、PNG、GIF、WebP、BMP、SVG 格式
- ✅ 自动去重（输入列表内的重复项）
- ✅ 检查已存在的图片 URL，避免重复添加

### 4. 用户体验
- ✅ 清晰的占位提示文字
- ✅ 加载状态显示
- ✅ 成功/失败消息提示
- ✅ 无效链接高亮显示
- ✅ 实时显示可添加数量

### 5. 消息集成
- ✅ 使用现有的 `messageManager` 统一提示
- ✅ 过滤重复链接时显示提示信息
- ✅ 跳过无效链接时显示警告

## 使用方法

### 在 MultiImageUpload组件中使用

组件会自动显示"网络图片"按钮（当还有剩余上传名额时）：

```vue
<MultiImageUpload 
  v-model="form.images" 
  :max-images="9" 
  :allow-delete-last="true"
/>
```

### 操作流程

1. **点击"网络图片"按钮**
   - 当还有剩余上传名额时，按钮会显示在上传网格中
   
2. **输入图片链接**
   - 在文本框中输入图片 URL
   - 可以粘贴包含多个 URL 的文本
   - 可以拖拽包含 URL 的文本到输入框

3. **自动验证**
   - 系统会自动验证 URL 格式
   - 自动过滤重复链接
   - 显示无效链接列表

4. **确认添加**
   - 点击"确认添加"按钮
   - 成功的图片会添加到上传队列
   - 显示成功消息提示

## 示例

### 输入格式
```
https://example.com/image1.jpg
https://example.com/image2.png
https://example.com/image3.gif
```

### 拖拽文本
从其他应用拖拽包含 URL 的文本到输入框，系统会自动提取所有 HTTP/HTTPS 链接。

## 验证规则

### URL 格式验证
- 必须是有效的 HTTP 或 HTTPS URL
- 必须以图片扩展名结尾（不区分大小写）：
  - `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.bmp`, `.svg`
  - 支持带查询参数，如：`image.jpg?width=100&height=200`

### 去重逻辑
1. **输入内去重**：自动移除输入文本中的重复 URL
2. **全局去重**：检查当前已上传的所有图片 URL，避免重复添加

### 数量限制
- 最多添加 9 张图片（可通过 `maxImages` 属性配置）
- 如果输入的链接数量超过剩余额度，会自动截取前 N 个有效链接

## 错误处理

### 无效链接
- 在模态框中高亮显示所有无效链接
- 显示红色警告区域
- 允许用户修正后重新提交

### 超过数量限制
- 显示错误提示："最多只能再添加 X 张图片"
- 自动截取前 N 个有效链接

### 网络图片无法访问
- 目前仅验证 URL 格式
- 实际加载验证在图片渲染时进行
- 无法加载的图片会显示浏览器默认的错误图标

## 响应式设计

- ✅ 桌面端：完整功能展示
- ✅ 移动端：优化布局和按钮尺寸
- ✅ 触摸设备：支持触摸操作

## 技术实现

### 主要依赖
- Vue 3 Composition API
- 现有的 UI 组件库（SvgIcon, MessageToast 等）
- `useScrollLock` 防滚动穿透

### 关键代码

#### URL 验证
```javascript
const isValidImageUrl = (url) => {
  try {
    new URL(url.trim())
    const imagePattern = /\.(jpeg|jpg|png|gif|webp|bmp|svg)(\?.*)?$/i
    return imagePattern.test(url.trim())
  } catch (_) {
    return false
  }
}
```

#### URL 提取（拖拽）
```javascript
const urlRegex = /(https?:\/\/[^\s]+)/g
const matches = text.match(urlRegex) || []
```

#### 去重处理
```javascript
const uniqueUrls = [...new Set(lines.map(l => l.trim()))]
const newUrls = uniqueUrls.filter(url => !existingUrls.includes(url))
```

## 注意事项

1. **图标资源**：需要确保 `link.svg` 图标存在于 `src/assets/icons/` 目录
2. **跨域问题**：网络图片可能存在跨域访问限制
3. **性能考虑**：大量网络图片可能影响页面加载速度
4. **安全性**：建议在后端也进行 URL 验证和图片内容检查

## 未来优化方向

- [ ] 添加图片预览功能（在确认前显示缩略图）
- [ ] 支持编辑单个 URL
- [ ] 支持批量删除已添加的网络图片
- [ ] 添加图片加载状态监控
- [ ] 支持更多图片格式（如 AVIF、HEIC 等）
- [ ] 添加图片尺寸验证
- [ ] 支持从社交媒体平台直接导入图片

## 测试建议

### 功能测试
1. 测试单个 URL 输入
2. 测试批量粘贴多行 URL
3. 测试拖拽文本功能
4. 测试重复 URL 过滤
5. 测试无效 URL 验证
6. 测试数量限制

### 兼容性测试
1. Chrome / Edge / Firefox / Safari
2. 桌面端 / 移动端
3. 深色模式 / 浅色模式

### 边界测试
1. 空输入
2. 超大数量 URL（如 100 个）
3. 特殊字符处理
4. 超长 URL
5. 格式错误的 URL
