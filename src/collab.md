好的，我们来深入代码，详细分析和讲解 prosemirror-collab 这个模块。这是 ProseMirror 协同编辑的核心，理解它需要掌握操作变换（Operational Transformation, OT）的基本思想。

prosemirror-collab 实现的是一个**集中式**的协同编辑模型。这意味着：

1.  有一个**中央权威机构**（通常是服务器）。
2.  所有客户端的修改都必须先发送给这个权威机构。
3.  权威机构按到达顺序处理修改，为每个修改分配一个版本号，然后将这些“已确认”的修改广播给所有客户端。
4.  客户端接收到来自权威机构的修改后，需要将自己本地尚未确认的修改，在这些新修改的基础上进行**重基 (Rebase)**。

现在，我们按照这个逻辑来剖析代码。

---

### 1. 核心数据结构：`CollabState` 和 `Rebaseable`

这是协同插件需要维护的核心状态。

- **`CollabState`**:

  - `version: number`: 客户端已经与中央权威同步到的最新版本号。
  - `unconfirmed: readonly Rebaseable[]`: 一个数组，存储了当前客户端在本地执行了、但还**没有被中央权威确认**的所有步骤 (`Step`)。这是协同编辑中最重要的状态。

- **`Rebaseable`**:
  - `step: Step`: 原始的步骤。
  - `inverted: Step`: 这个步骤的**逆操作**。这是执行“回滚”操作的关键。
  - `origin: Transform`: 产生这个步骤的原始 `Transaction`。这对于追溯时间戳等元数据很有用。

`Rebaseable` 封装了一个步骤及其逆操作，为“重基”操作提供了必要的信息。

---

### 2. `collab()` 插件：状态的管理者

这是协同功能的入口，它创建了一个 ProseMirror 插件来管理 `CollabState`。

```typescript
// ...existing code...
export function collab(config: CollabConfig = {}): Plugin {
  // ... config initialization ...

  return new Plugin({
    key: collabKey,

    state: {
      init: () => new CollabState(conf.version, []),
      apply(tr, collab) {
        let newState = tr.getMeta(collabKey) // 1. 优先处理来自 receiveTransaction 的新状态
        if (newState) return newState
        if (tr.docChanged)
          // 2. 如果是本地用户修改
          return new CollabState(collab.version, collab.unconfirmed.concat(unconfirmedFrom(tr)))
        return collab // 3. 如果没有变化，保持原样
      }
    }
    // ...
  })
}
```

`state.apply` 是插件的核心逻辑，它决定了每次 `Transaction` 后 `CollabState` 如何演变：

1.  **优先处理权威变更**: 它首先检查 `Transaction` 是否附带了 `collabKey` 的元数据。这个元数据是由 `receiveTransaction` 函数（我们稍后会讲）设置的，代表这是一个来自中央权威的、需要更新状态的特殊 `Transaction`。如果存在，就直接使用这个新的 `CollabState`。
2.  **累积本地变更**: 如果 `Transaction` 是一个普通的、由本地用户产生的文档修改 (`tr.docChanged` 为 `true`)，它就会将这次 `Transaction` 中的所有 `Step` 转换成 `Rebaseable` 对象，并追加到 `unconfirmed` 数组的末尾。
3.  **无变化**: 如果 `Transaction` 没有改变文档，也没有权威变更，那么 `CollabState` 保持不变。

---

### 3. `sendableSteps()`: 将本地修改发送出去

这个函数的作用是打包本地未确认的修改，以便发送给中央权威。

```typescript
// ...existing code...
export function sendableSteps(state: EditorState): { ... } | null {
  let collabState = collabKey.getState(state) as CollabState
  if (collabState.unconfirmed.length == 0) return null // 1. 如果没有未确认的修改，就没什么可发送的
  return {
    version: collabState.version, // 2. 告诉服务器，我的修改是基于哪个版本做出的
    steps: collabState.unconfirmed.map(s => s.step), // 3. 打包所有未确认的步骤
    clientID: (collabKey.get(state)!.spec as any).config.clientID, // 4. 带上我的客户端 ID
    origins: ...
  }
}
```

**工作流程**:

1.  你的应用会定期（或在每次修改后）调用 `sendableSteps(editor.state)`。
2.  如果返回非 `null`，就将返回的对象通过 WebSocket 或 HTTP 发送给服务器。
3.  服务器接收到后，会检查 `version` 是否与服务器上的当前版本匹配。如果匹配，就应用这些 `steps`，增加服务器版本号，然后将这些 `steps` 连同新的版本号和 `clientID` 广播给所有客户端（包括发送者自己）。

---

### 4. `receiveTransaction()`: 接收并处理权威修改

这是整个协同逻辑中最复杂、最核心的部分。当客户端从服务器收到一批新的 `steps` 时，它会调用这个函数。

```typescript
// ...existing code...
export function receiveTransaction(
  state: EditorState,
  steps: readonly Step[],
  clientIDs: readonly (string | number)[],
  // ...
) {
  // ...
  let collabState = collabKey.getState(state)
  let version = collabState.version + steps.length // 1. 计算新版本号
  let ourID = ...

  // 2. 识别并丢弃自己的修改
  let ours = 0
  while (ours < clientIDs.length && clientIDs[ours] == ourID) ++ours
  let unconfirmed = collabState.unconfirmed.slice(ours)
  steps = ours ? steps.slice(ours) : steps

  // 如果所有步骤都是我们自己刚发的，确认它们后就没事了
  if (!steps.length) return state.tr.setMeta(collabKey, new CollabState(version, unconfirmed))

  let nUnconfirmed = unconfirmed.length
  let tr = state.tr
  if (nUnconfirmed) {
    // 3. 核心：重基 (Rebase)
    unconfirmed = rebaseSteps(unconfirmed, steps, tr)
  } else {
    // 4. 如果没有本地修改，直接应用服务器的修改
    for (let i = 0; i < steps.length; i++) tr.step(steps[i])
    unconfirmed = []
  }

  // 5. 创建并应用最终的 Transaction
  let newCollabState = new CollabState(version, unconfirmed)
  // ...
  return tr
    .setMeta('rebased', nUnconfirmed)
    .setMeta('addToHistory', false)
    .setMeta(collabKey, newCollabState)
}
```

**工作流程**:

1.  **计算新版本**: 根据收到的 `steps` 数量，预先计算出同步完成后的新版本号。
2.  **确认自己的修改**: 服务器广播的 `steps` 中，可能包含刚刚由自己发送并被确认的修改。通过比较 `clientIDs` 和 `ourID`，函数会找到这部分修改，并将它们从本地的 `unconfirmed` 数组中移除。这相当于一个“确认”过程。
3.  **重基 (Rebase)**: 这是最关键的一步。如果确认完自己的修改后，仍然有来自**其他客户端**的 `steps`，并且本地还有一些更早的、未确认的修改 `unconfirmed`，就需要执行 `rebaseSteps`。
4.  **直接应用**: 如果本地没有任何未确认的修改，事情就简单了：直接将服务器来的 `steps` 应用到当前文档即可。
5.  **生成最终事务**: 函数最后会创建一个 `Transaction`，这个 `Transaction` 包含了重基或直接应用后的所有文档变化。最重要的是，它通过 `setMeta(collabKey, newCollabState)` 将更新后的 `CollabState`（新的 `version` 和重基后的 `unconfirmed` 数组）附加到 `Transaction` 上。当这个 `Transaction` 被应用时，`collab` 插件的 `apply` 方法就会捕获这个元数据，并更新编辑器的协同状态。

---

### 5. `rebaseSteps()`: 重基算法的实现

这是操作变换（OT）的具体实现。它的目标是：给定一组本地未确认的步骤 `steps` 和一组来自他人的步骤 `over`，计算出一组新的本地步骤，使得这组新步骤应用在“已应用了 `over` 的文档”上时，能达到与原始 `steps` 相同的效果。

```typescript
// ...existing code...
export function rebaseSteps(
  steps: readonly Rebaseable[],
  over: readonly Step[],
  transform: Transform
) {
  // 1. 回滚本地修改
  for (let i = steps.length - 1; i >= 0; i--) transform.step(steps[i].inverted)

  // 2. 应用他人修改
  for (let i = 0; i < over.length; i++) transform.step(over[i])

  let result = []
  // 3. 重新应用并映射本地修改
  for (let i = 0, mapFrom = steps.length; i < steps.length; i++) {
    // `transform.mapping` 此刻包含了回滚和应用他人修改的所有位置映射
    // `slice(mapFrom)` 只取“应用他人修改”部分产生的位置映射
    let mapped = steps[i].step.map(transform.mapping.slice(mapFrom))
    mapFrom--
    if (mapped && !transform.maybeStep(mapped).failed) {
      // ...
      result.push(new Rebaseable(mapped, ...))
    }
  }
  return result
}
```

**图解 `rebaseSteps` 过程**:

假设当前状态是 `DocA`。

- 你本地做了修改 `L`，状态变为 `DocA + L`。
- 同时，服务器收到了他人修改 `R`，权威状态变为 `DocA + R`。
- 现在你收到了 `R`，你需要将你的 `L` 重基到 `DocA + R` 之上。

`rebaseSteps` 的执行过程如下：

1.  **回滚 (Invert)**: 从 `DocA + L` 开始，应用 `L` 的逆操作 `L'`。文档状态回到 `DocA`。
    - `transform` 此时记录了 `L'` 的位置映射。
2.  **应用他人修改 (Apply)**: 在 `DocA` 的基础上，应用他人的修改 `R`。文档状态变为 `DocA + R`。
    - `transform` 此时记录了 `L'` 和 `R` 的位置映射。
3.  **重新应用本地修改 (Remap & Re-apply)**:
    - 取出你原始的本地修改 `L`。
    - 使用 `transform.mapping` 中由 `R` 产生的部分，对 `L` 进行**位置映射**，得到一个新的步骤 `L''`。`L''` 就是 `L` 在 `DocA + R` 这个新上下文中的版本。
    - 尝试将 `L''` 应用到当前文档 (`DocA + R`) 上。如果成功，文档变为 `DocA + R + L''`。
    - 将 `L''` 及其新的逆操作存入 `result` 数组。

最终，`transform` 中的文档变成了 `DocA + R + L''`，而 `result` 中存储了所有重基后的本地未确认步骤 `L''`。这就是协同编辑能够保持最终一致性的魔法所在。
