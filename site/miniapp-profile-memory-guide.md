# 小程序 laicai 对话接入「画像记忆」功能 - 开发文档

> 目标：在 laicai 对话中识别用户的「我是谁 / 我卖什么 / 我卖给谁」，识别到后弹出记忆卡片，用户点「一键记下」后**增量合并**进个人中心主画像（不覆盖原有内容）。
> 后端接口已就绪，**后端无需改动**，本次只需小程序前端接入。

---

## 一、后端接口契约

> 所有路径为相对路径，请拼接你们小程序现有的后端 baseUrl。

### 1. 识别画像（产生记忆卡片）

`POST /api/ai-agent-assistant/memory/extract`

请求体：

```json
{
  "messages": [
    { "role": "user", "content": "我是做家装的，在常州做了14年" },
    { "role": "assistant", "content": "..." }
  ]
}
```

> `role` 只传 `user` / `assistant` 两种；`user_id` 不用传，登录态走请求头（见下文）。

响应（识别到有效画像时）：

```json
{
  "suggestion": {
    "id": "xxxx",
    "details": [
      {
        "field": "who_am_i",
        "currentValue": "旧画像内容（前端可忽略）",
        "suggestedValue": "新识别到的内容",
        "evidence": "用户原话片段",
        "action": "fill"
      }
    ]
  }
}
```

响应（未识别到 / 未登录）：

```json
{ "suggestion": null }
```

`field` 三种取值：`who_am_i`（我是谁）、`what_i_sell`（我卖什么）、`who_i_sell_to`（我卖给谁）。

`action` 两种取值：

- `fill`：该字段原为空，直接记下；
- `update`：该字段已有内容，在现有内容上增量补充。

### 2. 确认记下（一键记下）

`POST /api/ai-agent-assistant/memory/apply`

请求体：

```json
{ "suggestionId": "xxxx" }
```

响应：

```json
{ "success": true, "message": "已记下", "rewardCredits": 0 }
```

### 3. 忽略（先不用）

`POST /api/ai-agent-assistant/memory/dismiss`

请求体：

```json
{ "suggestionId": "xxxx" }
```

响应：

```json
{ "success": true }
```

---

## 二、登录态（关键，易踩坑）

- 后端识别用户身份的顺序：`cookie 里的 user_id` → 请求头 `X-User-Id`。
- `extract` 额外兼容 `body.user_id`，但 **`apply` / `dismiss` 只认 cookie 或 `X-User-Id`，不认 body 里的 user_id**。

**统一做法**：三个接口的请求头都带上：

```
X-User-Id: <当前登录用户的真实 userId>
```

`userId` 从你们现有登录态里取（laicai 对话里拿到登录用户身份的地方，复用同一个值）。仅在已登录状态下调用，未登录不要发起 `extract`。

---

## 三、网络层：新增 memory 请求函数

按你们现有 `request` 封装替换即可（以下为示意）：

```ts
const base = '/api/ai-agent-assistant/memory';

function extractMemory(userId: string, messages: Array<{ role: string; content: string }>) {
  return request({
    url: `${base}/extract`,
    method: 'POST',
    header: { 'X-User-Id': userId, 'Content-Type': 'application/json' },
    data: { messages },
  });
}

function applyMemory(userId: string, suggestionId: string) {
  return request({
    url: `${base}/apply`,
    method: 'POST',
    header: { 'X-User-Id': userId, 'Content-Type': 'application/json' },
    data: { suggestionId },
  });
}

function dismissMemory(userId: string, suggestionId: string) {
  return request({
    url: `${base}/dismiss`,
    method: 'POST',
    header: { 'X-User-Id': userId, 'Content-Type': 'application/json' },
    data: { suggestionId },
  });
}
```

---

## 四、对话组件 ChatPanel 改动

### 1. 对话结束后触发识别

在 SSE 流式结束（收到 `[DONE]`，或 `onError` / `onDone` 结束）后，fire-and-forget 调一次识别：

```ts
// 取本次对话的 messages，只保留 user / assistant，content 为纯文本
const msgs = currentMessages.map((m) => ({ role: m.role, content: m.content }));

extractMemory(userId, msgs)
  .then((res) => {
    if (res?.suggestion) setMemorySuggestion(res.suggestion); // 弹出卡片
  })
  .catch(() => { /* 静默，不影响对话 */ });
```

### 2. 渲染记忆卡片

放在对话列表末尾：

```tsx
const FIELD_LABEL: Record<string, string> = {
  who_am_i: '我是谁',
  what_i_sell: '我卖什么',
  who_i_sell_to: '我卖给谁',
};

// ...
{memorySuggestion && (
  <View className="memory-card">
    <Text className="memory-title">我记下了这些信息，要帮你补充进画像吗？</Text>

    {memorySuggestion.details.map((d) => (
      <Text key={d.field} className="memory-detail">
        【{FIELD_LABEL[d.field]}】
        {d.action === 'fill'
          ? `记下「${d.suggestedValue}」`
          : `在现有内容上补充「${d.suggestedValue}」`}
      </Text>
    ))}

    <View className="memory-actions">
      <Button onClick={onApplyMemory} loading={applying}>一键记下</Button>
      <Button onClick={onDismissMemory}>先不用</Button>
    </View>
  </View>
)}
```

### 3. 两个按钮逻辑

```ts
const onApplyMemory = async () => {
  if (!memorySuggestion || applying) return;
  setApplying(true);
  try {
    const res = await applyMemory(userId, memorySuggestion.id);
    if (res?.success) {
      toast(res.message || '已记下');
      setMemorySuggestion(null);
    } else {
      toast(res?.error || '记下失败，请稍后重试');
    }
  } catch {
    toast('记下失败，请稍后重试');
  } finally {
    setApplying(false);
  }
};

const onDismissMemory = () => {
  const id = memorySuggestion?.id;
  setMemorySuggestion(null);
  if (id) dismissMemory(userId, id).catch(() => {});
};
```

---

## 五、注意事项

1. **不要阻塞对话**：`extract` / `apply` / `dismiss` 都是 fire-and-forget，任何失败都静默，不影响正常聊天。
2. **去重后端已做**：`extract` 会对最近 5 条待处理建议按「字段 + 建议值」去重，前端无需自己判重，重复触发不会重复弹卡片。
3. **未登录不弹卡片**：`extract` 在无 userId 时返回 `{ suggestion: null }`，卡片只在登录且识别到有效画像时出现。
4. **幂等**：重复点「一键记下」会返回 `409 该建议已处理`，前端按 `else` 分支提示即可，无需特殊处理。
5. **写入行为后端统一负责**：后端 `apply` 已是「增量合并」逻辑（保留原画像、补充新信息），小程序端只管弹卡片 + 传 `suggestionId`，不用在端上处理合并。
6. **`currentValue` 无需展示**：`details` 里的 `currentValue` 是建议生成时的快照，前端卡片不需要显示它，只展示 `suggestedValue` 即可。