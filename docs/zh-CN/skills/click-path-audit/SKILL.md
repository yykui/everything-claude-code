---
name: click-path-audit
description: 追踪每个面向用户的按钮/交互点的完整状态变化序列，找出函数各自正常但相互抵消、产生错误最终状态或使 UI 陷入不一致状态的 Bug。适用场景：系统调试未发现 Bug 但用户反映按钮异常，或在涉及共享状态存储的重大重构之后。
origin: community
---

# /click-path-audit — 行为流审计

发现静态代码阅读会遗漏的 Bug：状态交互副作用、顺序调用之间的竞争条件，以及静默地相互撤销的处理器。

## 这个技能解决什么问题

传统调试检查：
- 函数是否存在？（缺失连接）
- 是否崩溃？（运行时错误）
- 返回值类型是否正确？（数据流）

但它**不会**检查：
- **最终 UI 状态是否与按钮标签所承诺的一致？**
- **函数 B 是否静默地撤销了函数 A 刚做的事情？**
- **共享状态（Zustand/Redux/context）是否有副作用会取消预期操作？**

真实案例：一个"新建邮件"按钮先调用 `setComposeMode(true)` 再调用 `selectThread(null)`。两个函数单独运行都正常。但 `selectThread` 有一个将 `composeMode` 重置为 `false` 的副作用。按钮没有任何效果。系统调试找出了 54 个 Bug——这一个被遗漏了。

---

## 工作原理

对目标区域中的**每一个**交互触点执行以下操作：

```
1. 识别处理器（onClick、onSubmit、onChange 等）
2. 按顺序追踪处理器中的每个函数调用
3. 对于每个函数调用：
   a. 它读取哪些状态？
   b. 它写入哪些状态？
   c. 它对共享状态有副作用吗？
   d. 它是否作为副作用重置/清除了某些状态？
4. 检查：后面的调用是否撤销了前面调用的状态变更？
5. 检查：最终状态是否符合用户对按钮标签的预期？
6. 检查：是否存在竞争条件（异步调用以错误顺序 resolve）？
```

---

## 执行步骤

### 第一步：映射状态存储

在审计任何触点之前，为每个状态存储 action 构建副作用映射：

```
对于范围内的每个 Zustand store / React context：
  对于每个 action/setter：
    - 它设置哪些字段？
    - 它是否作为副作用重置其他字段？
    - 记录：actionName → {sets: [...], resets: [...]}
```

这是关键参考资料。没有了解 `selectThread` 会重置 `composeMode`，"新建邮件" Bug 就是不可见的。

**输出格式：**
```
STORE: emailStore
  setComposeMode(bool) → sets: {composeMode}
  selectThread(thread|null) → sets: {selectedThread, selectedThreadId, messages, drafts, selectedDraft, summary} RESETS: {composeMode: false, composeData: null, redraftOpen: false}
  setDraftGenerating(bool) → sets: {draftGenerating}
  ...

危险重置（重置非自身所属状态的 action）：
  selectThread → 重置 composeMode（由 setComposeMode 所有）
  reset → 重置所有内容
```

### 第二步：审计每个触点

对目标区域中的每个按钮/开关/表单提交：

```
触点：[按钮标签] 位于 [Component:line]
  处理器：onClick → {
    调用 1：functionA() → sets {X: true}
    调用 2：functionB() → sets {Y: null} RESETS {X: false}  ← 冲突
  }
  预期：用户看到 [按钮标签所承诺的描述]
  实际：X 为 false，因为 functionB 将其重置了
  结论：BUG — [描述]
```

**检查以下每种 Bug 模式：**

#### 模式 1：顺序撤销
```
handler() {
  setState_A(true)     // 设置 X = true
  setState_B(null)     // 副作用：重置 X = false
}
// 结果：X 为 false。第一个调用毫无意义。
```

#### 模式 2：异步竞争
```
handler() {
  fetchA().then(() => setState({ loading: false }))
  fetchB().then(() => setState({ loading: true }))
}
// 结果：最终 loading 状态取决于哪个先 resolve
```

#### 模式 3：过时闭包
```
const [count, setCount] = useState(0)
const handler = useCallback(() => {
  setCount(count + 1)  // 捕获了过时的 count
  setCount(count + 1)  // 相同的过时 count——只递增 1，而非 2
}, [count])
```

#### 模式 4：缺失状态转换
```
// 按钮标签说"保存"，但处理器只做验证，从不真正保存
// 按钮标签说"删除"，但处理器只设置一个标志，不调用 API
// 按钮标签说"发送"，但 API 端点已移除/损坏
```

#### 模式 5：条件死路
```
handler() {
  if (someState) {        // someState 在此时始终为 false
    doTheActualThing()    // 永远不会执行
  }
}
```

#### 模式 6：useEffect 干扰
```
// 按钮将 stateX 设置为 true
// 一个 useEffect 监听 stateX 并将其重置为 false
// 用户看不到任何变化
```

### 第三步：报告

对发现的每个 Bug：

```
CLICK-PATH-NNN: [严重性: CRITICAL/HIGH/MEDIUM/LOW]
  触点：[按钮标签] 位于 [file:line]
  模式：[顺序撤销 / 异步竞争 / 过时闭包 / 缺失转换 / 死路 / useEffect 干扰]
  处理器：[函数名或内联]
  追踪：
    1. [调用] → sets {field: value}
    2. [调用] → RESETS {field: value}  ← 冲突
  预期：[用户的预期]
  实际：[实际发生的情况]
  修复：[具体修复方案]
```

---

## 范围控制

这项审计成本较高。请合理确定范围：

- **全应用审计：** 发布时或重大重构后使用。为每个页面启动并行代理。
- **单页面审计：** 构建新页面后或用户报告按钮异常后使用。
- **聚焦存储的审计：** 修改 Zustand store 后使用——审计所有受变更 action 影响的调用方。

### 全应用推荐代理分工：

```
代理 1：映射所有状态存储（第一步）——这是所有其他代理的共享上下文
代理 2：仪表板（Tasks、Notes、Journal、Ideas）
代理 3：聊天（DanteChatColumn、JustChatPage）
代理 4：邮件（ThreadList、DraftArea、EmailsPage）
代理 5：项目（ProjectsPage、ProjectOverviewTab、NewProjectWizard）
代理 6：CRM（所有子标签）
代理 7：个人资料、设置、Vault、通知
代理 8：管理套件（所有页面）
```

代理 1 必须最先完成。其输出是所有其他代理的输入。

---

## 何时使用

- 系统调试未发现 Bug 但用户报告 UI 异常之后
- 修改任意 Zustand store action 之后（检查所有调用方）
- 任何涉及共享状态的重构之后
- 发布前，针对关键用户流程
- 当某个按钮"没有任何效果"时——这是解决此问题的专用工具

## 何时不使用

- API 层面的 Bug（响应结构错误、端点缺失）——使用系统调试
- 样式/布局问题——视觉检查
- 性能问题——性能分析工具

---

## 与其他技能的集成

- 在 `/superpowers:systematic-debugging` 之后运行（后者负责找出其他 54 种 Bug 类型）
- 在 `/superpowers:verification-before-completion` 之前运行（后者验证修复是否有效）
- 为 `/superpowers:test-driven-development` 提供输入——此处发现的每个 Bug 都应有对应的测试

---

## 示例：启发本技能的那个 Bug

**ThreadList.tsx "新建邮件"按钮：**
```
onClick={() => {
  useEmailStore.getState().setComposeMode(true)   // ✓ 设置 composeMode = true
  useEmailStore.getState().selectThread(null)      // ✗ 重置 composeMode = false
}}
```

Store 定义：
```
selectThread: (thread) => set({
  selectedThread: thread,
  selectedThreadId: thread?.id ?? null,
  messages: [],
  drafts: [],
  selectedDraft: null,
  summary: null,
  composeMode: false,     // ← 这个静默重置使按钮失效
  composeData: null,
  redraftOpen: false,
})
```

**系统调试遗漏了它**，因为：
- 按钮有 onClick 处理器（不是死链接）
- 两个函数都存在（无缺失连接）
- 两个函数都不崩溃（无运行时错误）
- 数据类型正确（无类型不匹配）

**点击路径审计能发现它**，因为：
- 第一步将 `selectThread` 的重置 `composeMode` 行为映射出来
- 第二步追踪处理器：调用 1 将其设为 true，调用 2 将其重置为 false
- 结论：顺序撤销——最终状态与按钮意图相悖
