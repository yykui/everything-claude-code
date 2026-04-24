---
name: click-path-audit
description: "追踪每个用户可见按钮/交互点的完整状态变更序列，以发现函数单独工作正常但相互抵消、产生错误最终状态或使 UI 处于不一致状态的 Bug。适用场景：系统性调试未发现 Bug 但用户报告按钮异常，或任何涉及共享状态存储的重大重构之后。"
origin: community
---

# /click-path-audit — 行为流审计

发现静态代码阅读遗漏的 Bug：状态交互副作用、顺序调用之间的竞态条件，以及静默撤销彼此操作的处理器。

## 该技能解决的问题

传统调试检查：
- 函数是否存在？（缺失连接）
- 是否崩溃？（运行时错误）
- 返回值类型是否正确？（数据流）

但**不检查**：
- **最终 UI 状态是否与按钮标签所承诺的一致？**
- **函数 B 是否静默撤销了函数 A 刚刚完成的操作？**
- **共享状态（Zustand/Redux/context）是否有副作用取消了预期操作？**

真实案例："新建邮件"按钮调用了 `setComposeMode(true)` 然后 `selectThread(null)`。两个函数单独运行均正常。但 `selectThread` 有个副作用会将 `composeMode` 重置为 `false`。按钮什么都没做。系统性调试发现了 54 个 Bug——这个被遗漏了。

---

## 工作原理

对目标区域内**每一个**交互触点执行：

```
1. 识别处理器（onClick、onSubmit、onChange 等）
2. 按顺序追踪处理器中的每个函数调用
3. 对每个函数调用：
   a. 它读取了哪些状态？
   b. 它写入了哪些状态？
   c. 对共享状态是否有副作用？
   d. 是否作为副作用重置/清除了某些状态？
4. 检查：后续调用是否撤销了前序调用的状态变更？
5. 检查：最终状态是否符合用户对按钮标签的预期？
6. 检查：是否存在竞态条件（异步调用以错误顺序 resolve）？
```

---

## 执行步骤

### 第一步：映射状态存储

在审计任何触点之前，为每个状态存储操作构建副作用映射：

```
对作用域内的每个 Zustand 存储 / React context：
  对每个 action/setter：
    - 它设置了哪些字段？
    - 它是否作为副作用重置了其他字段？
    - 记录：actionName → {sets: [...], resets: [...]}
```

这是关键参考资料。"新建邮件"Bug 在不知道 `selectThread` 会重置 `composeMode` 的情况下是不可见的。

**输出格式：**
```
STORE: emailStore
  setComposeMode(bool) → sets: {composeMode}
  selectThread(thread|null) → sets: {selectedThread, selectedThreadId, messages, drafts, selectedDraft, summary} RESETS: {composeMode: false, composeData: null, redraftOpen: false}
  setDraftGenerating(bool) → sets: {draftGenerating}
  ...

危险重置（清除其不拥有的状态的操作）：
  selectThread → 重置 composeMode（由 setComposeMode 拥有）
  reset → 重置所有内容
```

### 第二步：审计每个触点

对目标区域内每个按钮/开关/表单提交：

```
TOUCHPOINT: [按钮标签] 在 [Component:行号]
  HANDLER: onClick → {
    调用 1: functionA() → sets {X: true}
    调用 2: functionB() → sets {Y: null} RESETS {X: false}  ← 冲突
  }
  预期：用户看到 [按钮标签承诺的描述]
  实际：X 为 false，因为 functionB 将其重置了
  判定：BUG — [描述]
```

**检查以下每种 Bug 模式：**

#### 模式一：顺序撤销
```
handler() {
  setState_A(true)     // sets X = true
  setState_B(null)     // side effect: resets X = false
}
// 结果：X 为 false。第一个调用毫无意义。
```

#### 模式二：异步竞态
```
handler() {
  fetchA().then(() => setState({ loading: false }))
  fetchB().then(() => setState({ loading: true }))
}
// 结果：最终 loading 状态取决于哪个先 resolve
```

#### 模式三：陈旧闭包
```
const [count, setCount] = useState(0)
const handler = useCallback(() => {
  setCount(count + 1)  // captures stale count
  setCount(count + 1)  // same stale count — increments by 1, not 2
}, [count])
```

#### 模式四：缺失状态转换
```
// 按钮显示"保存"但处理器只做验证，从不真正保存
// 按钮显示"删除"但处理器只设置标志而不调用 API
// 按钮显示"发送"但 API 端点已移除/损坏
```

#### 模式五：条件死路径
```
handler() {
  if (someState) {        // someState 在此时总为 false
    doTheActualThing()    // 永远不会执行到
  }
}
```

#### 模式六：useEffect 干扰
```
// 按钮设置 stateX = true
// 某个 useEffect 监听 stateX 并将其重置为 false
// 用户看到什么都没发生
```

### 第三步：报告

对每个发现的 Bug：

```
CLICK-PATH-NNN: [严重程度: CRITICAL/HIGH/MEDIUM/LOW]
  触点：[按钮标签] 在 [file:line]
  模式：[顺序撤销 / 异步竞态 / 陈旧闭包 / 缺失转换 / 死路径 / useEffect 干扰]
  处理器：[函数名或内联]
  追踪：
    1. [调用] → sets {field: value}
    2. [调用] → RESETS {field: value}  ← 冲突
  预期：[用户的预期]
  实际：[实际发生的情况]
  修复：[具体修复方案]
```

---

## 作用域控制

此审计成本较高。请合理界定作用域：

- **完整应用审计：** 用于发布或重大重构后。按页面启动并行 agent。
- **单页审计：** 用于构建新页面后或用户报告按钮损坏后。
- **以存储为中心的审计：** 修改 Zustand 存储后使用——审计已更改操作的所有消费者。

### 完整应用的推荐 agent 拆分：

```
Agent 1: 映射所有状态存储（第一步）——这是所有其他 agent 的共享上下文
Agent 2: Dashboard（Tasks, Notes, Journal, Ideas）
Agent 3: Chat（DanteChatColumn, JustChatPage）
Agent 4: Emails（ThreadList, DraftArea, EmailsPage）
Agent 5: Projects（ProjectsPage, ProjectOverviewTab, NewProjectWizard）
Agent 6: CRM（所有子标签页）
Agent 7: Profile, Settings, Vault, Notifications
Agent 8: Management Suite（所有页面）
```

Agent 1 必须首先完成。其输出是所有其他 agent 的输入。

---

## 何时使用

- 系统性调试"未发现 Bug"但用户报告 UI 异常时
- 修改任何 Zustand 存储操作后（检查所有调用者）
- 任何涉及共享状态的重构之后
- 发布前，针对关键用户流程
- 按钮"没有任何反应"时——这是专为此设计的工具

## 何时不使用

- API 级 Bug（响应格式错误、缺少端点）——使用 systematic-debugging
- 样式/布局问题——视觉检查
- 性能问题——性能分析工具

---

## 与其他技能的集成

- 在 `/superpowers:systematic-debugging` **之后**运行（后者发现其他 54 种 Bug 类型）
- 在 `/superpowers:verification-before-completion` **之前**运行（后者验证修复是否有效）
- 输入到 `/superpowers:test-driven-development`——此处发现的每个 Bug 都应编写测试

---

## 示例：启发此技能的 Bug

**ThreadList.tsx "新建邮件"按钮：**
```
onClick={() => {
  useEmailStore.getState().setComposeMode(true)   // ✓ sets composeMode = true
  useEmailStore.getState().selectThread(null)      // ✗ RESETS composeMode = false
}}
```

存储定义：
```
selectThread: (thread) => set({
  selectedThread: thread,
  selectedThreadId: thread?.id ?? null,
  messages: [],
  drafts: [],
  selectedDraft: null,
  summary: null,
  composeMode: false,     // ← 这个静默重置导致按钮失效
  composeData: null,
  redraftOpen: false,
})
```

**系统性调试遗漏了它**，因为：
- 按钮有 onClick 处理器（不是死代码）
- 两个函数都存在（无缺失连接）
- 两个函数均未崩溃（无运行时错误）
- 数据类型正确（无类型不匹配）

**点击路径审计发现了它**，因为：
- 第一步映射出 `selectThread` 重置 `composeMode`
- 第二步追踪处理器：调用 1 设为 true，调用 2 重置为 false
- 判定：顺序撤销——最终状态与按钮意图相矛盾
