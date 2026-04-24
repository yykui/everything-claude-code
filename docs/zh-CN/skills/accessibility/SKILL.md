---
name: accessibility
description: 使用 WCAG 2.2 AA 级标准设计、实现和审计无障碍数字产品。使用此技能为 Web 生成语义化 ARIA，并为 Web 和原生平台（iOS/Android）生成无障碍特性。
origin: ECC
---

# 无障碍访问（WCAG 2.2）

本技能确保数字界面对所有用户（包括使用屏幕阅读器、开关控制或键盘导航的用户）均符合可感知、可操作、可理解、健壮性（POUR）原则。重点在于 WCAG 2.2 成功标准的技术实现。

## 何时使用

- 为 Web、iOS 或 Android 定义 UI 组件规范。
- 审计现有代码中的无障碍障碍或合规性差距。
- 实现新的 WCAG 2.2 标准，如目标尺寸（最小值）和焦点外观。
- 将高层设计需求映射到技术属性（ARIA 角色、特性、提示）。

## 核心概念

- **POUR 原则**：WCAG 的基础（可感知、可操作、可理解、健壮性）。
- **语义映射**：优先使用原生元素而非通用容器，以提供内置无障碍支持。
- **无障碍树**：辅助技术实际"读取"的 UI 表示形式。
- **焦点管理**：控制键盘/屏幕阅读器光标的顺序和可见性。
- **标签与提示**：通过 `aria-label`、`accessibilityLabel` 和 `contentDescription` 提供上下文。

## 工作原理

### 第一步：确定组件角色

确定功能用途（例如：这是按钮、链接还是标签页？）。在使用自定义角色之前，优先使用最具语义化的原生元素。

### 第二步：定义可感知属性

- 确保文字对比度满足 **4.5:1**（普通文本）或 **3:1**（大文本/UI）。
- 为非文字内容（图片、图标）添加文字替代。
- 实现响应式重排（放大至 400% 不丢失功能）。

### 第三步：实现可操作控件

- 确保最小 **24x24 CSS 像素**目标尺寸（WCAG 2.2 SC 2.5.8）。
- 验证所有交互元素均可通过键盘访问，并具有可见的焦点指示器（SC 2.4.11）。
- 为拖拽操作提供单指针替代方案。

### 第四步：确保可理解逻辑

- 使用一致的导航模式。
- 提供描述性错误消息和更正建议（SC 3.3.3）。
- 实现"冗余输入"（SC 3.3.7），避免重复要求同一数据。

### 第五步：验证健壮兼容性

- 使用正确的 `名称、角色、值` 模式。
- 为动态状态更新实现 `aria-live` 或实时区域。

## 无障碍架构图

```mermaid
flowchart TD
  UI["UI Component"] --> Platform{Platform?}
  Platform -->|Web| ARIA["WAI-ARIA + HTML5"]
  Platform -->|iOS| SwiftUI["Accessibility Traits + Labels"]
  Platform -->|Android| Compose["Semantics + ContentDesc"]

  ARIA --> AT["Assistive Technology (Screen Readers, Switches)"]
  SwiftUI --> AT
  Compose --> AT
```

## 跨平台映射

| 功能               | Web (HTML/ARIA)          | iOS (SwiftUI)                        | Android (Compose)                                           |
| :----------------- | :----------------------- | :----------------------------------- | :---------------------------------------------------------- |
| **主标签**         | `aria-label` / `<label>` | `.accessibilityLabel()`              | `contentDescription`                                        |
| **辅助提示**       | `aria-describedby`       | `.accessibilityHint()`               | `Modifier.semantics { stateDescription = ... }`             |
| **动作角色**       | `role="button"`          | `.accessibilityAddTraits(.isButton)` | `Modifier.semantics { role = Role.Button }`                 |
| **实时更新**       | `aria-live="polite"`     | `.accessibilityLiveRegion(.polite)`  | `Modifier.semantics { liveRegion = LiveRegionMode.Polite }` |

## 示例

### Web：无障碍搜索

```html
<form role="search">
  <label for="search-input" class="sr-only">Search products</label>
  <input type="search" id="search-input" placeholder="Search..." />
  <button type="submit" aria-label="Submit Search">
    <svg aria-hidden="true">...</svg>
  </button>
</form>
```

### iOS：无障碍操作按钮

```swift
Button(action: deleteItem) {
    Image(systemName: "trash")
}
.accessibilityLabel("Delete item")
.accessibilityHint("Permanently removes this item from your list")
.accessibilityAddTraits(.isButton)
```

### Android：无障碍切换开关

```kotlin
Switch(
    checked = isEnabled,
    onCheckedChange = { onToggle() },
    modifier = Modifier.semantics {
        contentDescription = "Enable notifications"
    }
)
```

## 应避免的反模式

- **Div 按钮**：在点击事件中使用 `<div>` 或 `<span>` 而不添加角色和键盘支持。
- **仅用颜色表示含义**：_仅_通过颜色变化（如将边框变红）来指示错误或状态。
- **模态框焦点未被约束**：模态框未捕获焦点，允许键盘用户在模态框打开时导航背景内容。焦点必须被约束_并且_可通过 `Escape` 键或明确的关闭按钮退出（WCAG SC 2.1.2）。
- **冗余的替代文本**：在 alt 文本中使用"Image of..."或"Picture of..."（屏幕阅读器已自动朗报角色"Image"）。

## 最佳实践检查清单

- [ ] 交互元素满足 **24x24px**（Web）或 **44x44pt**（原生）的目标尺寸。
- [ ] 焦点指示器清晰可见且高对比度。
- [ ] 模态框打开时**约束焦点**，关闭时干净释放（`Escape` 键或关闭按钮）。
- [ ] 下拉菜单关闭时将焦点恢复到触发元素。
- [ ] 表单提供基于文本的错误建议。
- [ ] 所有仅图标的按钮都有描述性文字标签。
- [ ] 文本缩放时内容正确重排。

## 参考资料

- [WCAG 2.2 Guidelines](https://www.w3.org/TR/WCAG22/)
- [WAI-ARIA Authoring Practices](https://www.w3.org/TR/wai-aria-practices/)
- [iOS Accessibility Programming Guide](https://developer.apple.com/documentation/accessibility)
- [iOS Human Interface Guidelines - Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [Android Accessibility Developer Guide](https://developer.android.com/guide/topics/ui/accessibility)

## 相关技能

- `frontend-patterns`
- `frontend-design`
- `liquid-glass-design`
- `swiftui-patterns`
