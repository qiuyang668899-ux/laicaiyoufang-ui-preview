# 小程序端同步清单（智能体分类重构）

> 对应改动：官网/导航智能体「做减法」重构 —— 9 分类改名重组 + 隐藏 11 个智能体 + 「实体商家获客文案」改名。
> 后端已处理所有数据层变化，本清单用于核对小程序前端代码是否需要同步。

## 一、结论

小程序端若纯动态渲染 `/api/agents/list` 与 `/api/ai-agent-assistant`，则基本零改动。
只有「硬编码」了以下内容才需要改。

## 二、必查项（有硬编码则必须改）

### 1. 硬编码的分类名 / 分类清单

小程序可见的旧 9 个业务分类 → 新 8 个业务分类（`personal-info` 与兜底「更多智能体」后端本就不返回）：

| 旧分类名 | 新分类名 |
| --- | --- |
| 强人设-构建知识库 | 人设画像 |
| 强人设-分析黄金定位 | 黄金定位 |
| 信任流量-公域精准获客 | 内容获客 |
| GEO-坐等客户咨询上门 | GEO获客 |
| 大众点评-评价与营销码 | 本地实体获客 |
| 信任兑现-销售高效成交 | 成交转化 |
| FDE-企业级AI落地伙伴 | 企业AI落地 |
| 其他商业赋能智能体 | 知识付费 |
| 需求识别与行业洞察 | 已删除（拆分并入「黄金定位」与隐藏区）|

### 2. 硬编码的智能体名

- 「本地实体商家获客文案」→「实体商家获客文案」
- path 不变：`/agent/local-business-customer`

### 3. 已隐藏智能体的残留入口（写死的跳转 / 推荐位 / 活动页）

小程序共隐藏 11 个，其中 5 个是「小程序独有隐藏」（网页仍显示、小程序下线）：

小程序独有隐藏（网页还有、小程序不显示）：
- brand-interview（品牌/产品访谈）
- geo-brand-monitor（GEO-品牌声量监测）
- sales-champion-clone（AI复制销冠）
- ai-video（真人IP口播视频，小程序暂未上线）
- scene-video（实体场景口播视频，小程序暂未上线）

网页 + 小程序同时隐藏：
- deep-research、sales-team-motivation、fde-visit-guide、expression-coach、board-of-directors、dankoe-clone

## 三、关键注意点

调用 `/api/ai-agent-assistant` 时必须传 `channel: 'miniapp'`，否则后端按网页隐藏集合过滤，会推荐出「小程序独有隐藏」的那 5 个智能体。

```js
body?.channel === 'miniapp' ? HIDDEN_AGENTS_IN_MINIAPP : HIDDEN_AGENTS_IN_WEB
```

## 四、无需改（稳定项）

- 分类 `id`：复用旧值（trust-knowledge / trust-positioning / …），未变
- 分类 `icon` 名：BookOpen / Crown / TrendingUp / …，未变
- 智能体 `path`：全部未变
- API 返回字段结构：`categories[].{id,name,icon}` + `agents[].{name,path,description,icon,isCozeStore,showLinkIcon}`，未变

## 五、建议确认项

- 埋点/统计若按 `name` 归因 → 需同步新分类名（按 `id` 归因则无需）
- 智能体总数现为 32（若小程序有总数 / 分页 / 空态逻辑需留意）