# 豆包 Seedance 2.5 API 调用教程：用 TryAllAPI 做异步视频生成

> **Seedance 2.5**（模型 ID：`doubao-seedance-2-5-260628`）是豆包多模态视频创作模型。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场选型、读价格表，到 `POST` 创建任务 / `GET` 轮询出片的完整 **异步视频生成 API** 接入流程。价格与分组倍率**以页面实时为准**；示例仅为请求形态说明，未跑通真实出片。

**Keywords:** Seedance 2.5 · 豆包视频 API · TryAllAPI · 异步视频生成 · `doubao-seedance-2-5-260628` · `/api/v3/contents/generations/tasks`

---

## 目录

- [为什么用聚合站调视频模型](#一为什么用聚合站调视频模型)
- [在模型广场找到 Seedance 2.5](#二在模型广场找到-seedance-25)
- [模型怎么选：2.5 vs 2.0 vs 2.0-fast](#三模型怎么选25-vs-20-vs-20-fast)
- [价格怎么读](#四价格怎么读复现截图表格)
- [调用流程：Key → POST → 轮询 GET → 视频 URL](#五调用流程key--post--轮询-get--视频-url)
- [curl 最小示例](#六curl-最小示例)
- [常见注意](#七常见注意)
- [FAQ](#faq)
- [小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调视频模型

视频生成几乎都是 **异步任务**：先 `POST` 创建任务拿 `task_id`，再轮询 `GET` 直到状态变为成功，最后从响应里取出 `video_url`。若业务同时要接文本、生图、多厂商视频，为每个厂商维护一套 Key、签名与重试逻辑会很散。

聚合站（如 TryAllAPI）的常见价值是：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可试不同档位；
2. **异步形态一致**：创建任务 → 轮询 → 取 URL，便于挂到 Workers / 队列；
3. **模型广场可对照**：分辨率档位、是否含参考视频、Token 分组一目了然。

下面以广场里可见的 **`doubao-seedance-2-5-260628`** 为主线。

---

## 二、在模型广场找到 Seedance 2.5

打开模型广场，侧栏可按供应商（如豆包）、类型（视频 / 音频）、标签（**视频**、**异步**）以及 Token 分组筛选。网格里能看到多张 Seedance 卡片，点开详情后大致如下：

![豆包 Seedance 2.5 模型广场详情卡：模型 ID、价格表与异步 API](https://upload.maynor1024.live/file/1792409898041_model-square.png)

截图里详情卡要点（与广场文案一致）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `doubao-seedance-2-5-260628` |
| **供应商** | 豆包大模型团队 |
| **描述 gist** | Seedance 2.5 新一代多模态视频创作模型；长叙事、强参考、精准逻辑、多语言 |
| **备注** | Seedance-1 分组不支持 480p，请改用其他分辨率 |
| **标签** | 视频、异步 |
| **API** | `POST` 豆包视频 `/api/v3/contents/generations/tasks` |
| **计费** | 按量；分组 **Seedance-2** |

同屏还能看到相邻卡片：`doubao-seedance-2-0-260128`、`doubao-seedance-2-0-fast…` 等。侧栏 Token 分组可见 **Seedance-1（x1.3）**、**Seedance-2（x1.8）**。

文中截图 UI 品牌为 momoAPIPro / 模型广场，实际对接产品以 [TryAllAPI](https://tryallapi.com/) 控制台与文档为准。

---

## 三、模型怎么选：2.5 vs 2.0 vs 2.0-fast

只根据截图可见信息做选型对照，**不编造未在图中出现的能力差异**：

| 模型 ID（可见） | 怎么理解（接入视角） |
| --- | --- |
| **`doubao-seedance-2-5-260628`** | 当前详情主推卡：描述强调长叙事、强参考、精准逻辑、多语言；走 Seedance-2 分组计价 |
| **`doubao-seedance-2-0-260128`** | 同屏可见的 2.0 标准卡；需要对照时与 2.5 并列试 Prompt / 容量 |
| **`doubao-seedance-2-0-fast…`** | 同屏可见的 fast 变体；命名暗示偏「快档」，具体限额与排队以控制台为准 |

实务上可以：默认先锁一个 `model` 字符串写进配置；遇容量不足或排队过长，再在同供应商的可见 ID 之间做降级，而不是改 endpoint。

---

## 四、价格怎么读（复现截图表格）

详情里的 **分组价格** 单位是 **$/1M tokens**，按量计费，分组为 **Seedance-2**。截图表格复现如下（**以页面实时为准**）：

| 分辨率 | 场景 | 价格（$/1M tokens） |
| --- | --- | --- |
| **480p / 720p** | 不包含参考视频 | **$19.260** |
| **480p / 720p** | 包含参考视频 | **$11.520** |
| **1080p** | 不包含参考视频 | **$21.060** |
| **1080p** | 包含参考视频 | **$12.600** |

读表时两点容易漏：

1. **有参考视频** 的那一档单价更低（同分辨率下）；是否「包含参考」以你请求体是否带参考视频为准。
2. 侧栏还有 **Seedance-1（x1.3）** / **Seedance-2（x1.8）** 分组倍率；2.5 这张卡挂在 Seedance-2。另：文案写明 **Seedance-1 不支持 480p**，若你落到该分组，请改用其他分辨率。

---

## 五、调用流程：Key → POST → 轮询 GET → 视频 URL

整体是典型的 **异步视频生成** 流水线：

```text
控制台创建 API Key
  → POST /api/v3/contents/generations/tasks   （创建，拿 task_id）
  → 间隔轮询 GET …/tasks/{task_id}
  → status 为成功后，从 content.video_url 取成片地址
  → 失败则读 error，按业务退积分 / 重试
```

建议：

- **超时与退避**：视频渲染可能几十秒到数分钟，轮询间隔建议 2–5s，并设总超时；
- **幂等**：把 `task_id` 落库，避免前端连点重复创建；
- **Base URL**：下文示例用 `https://tryallapi.com`；若控制台文档给出独立网关域，**以控制台文档为准**。

---

## 六、curl 最小示例

以下为 ark 兼容风格的最小示例：Bearer 鉴权、模型 ID 与截图一致、占位 Key `sk-xxx`（**不要**填入环境里的真实 Key）。

### 1）创建任务（POST）

```bash
curl -sS -X POST 'https://tryallapi.com/api/v3/contents/generations/tasks' \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "doubao-seedance-2-5-260628",
    "content": [
      {
        "type": "text",
        "text": "Vertical 9:16, 8s. A quiet cafe morning, soft natural light, gentle camera push-in."
      }
    ],
    "resolution": "720p",
    "ratio": "9:16",
    "duration": 8,
    "generate_audio": true
  }'
```

成功响应里应能拿到任务 ID（字段名常见为 `id` / `task_id`，以实际 JSON 为准）。

### 2）轮询任务（GET）

```bash
TASK_ID='替换为上一步返回的 id'

curl -sS -X GET \
  "https://tryallapi.com/api/v3/contents/generations/tasks/${TASK_ID}" \
  -H 'Authorization: Bearer sk-xxx'
```

当 `status` 表示完成时，从 `content.video_url`（或网关包装层下的同名字段）取出可下载的视频地址。可选字段如 `resolution`、`ratio`、`duration`、`usage` 便于对账。

若要带 **参考图 / 参考视频**，在 `content` 数组中追加 `image_url` / `video_url` 等项（具体 `role` 与字段以控制台文档为准）；计费会落到「包含参考视频」那一档时，请对照第四节表格。

---

## 七、常见注意

1. **异步**：不要把创建接口当同步渲染；前端应展示进度并轮询，或走 Webhook（若控制台提供）。
2. **参考视频计价更低档**：同分辨率下「包含参考视频」单价低于「不包含」；业务若本来就要锁动作模板，可优先走参考视频路径——但仍以实时价为准。
3. **Seedance-1 无 480p**：广场备注已写明；选错分组或分辨率会直接报错或拒单。
4. **分组倍率**：Seedance-1 x1.3、Seedance-2 x1.8，和「$/1M tokens」表一起读，避免只盯绝对数字。
5. **合规**：仅使用你有权使用的素材；成片标明 AI 生成；遵守平台与当地法规，勿生成违规内容。
6. **容量与降级**：偶发「服务暂不可用」时可换同屏可见的其他 Seedance ID 重试，并做好失败退款 / 提示。

---

## FAQ

### Seedance 2.5 的模型 ID 是什么？

广场详情卡上的模型 ID 为 **`doubao-seedance-2-5-260628`**。请求体里的 `"model"` 字段应填该字符串。

### 豆包 Seedance 视频 API 的 endpoint 是什么？

异步任务接口为：

- 创建：`POST /api/v3/contents/generations/tasks`
- 查询：`GET /api/v3/contents/generations/tasks/{task_id}`

Base URL 示例为 `https://tryallapi.com`；以控制台文档为准。

### 为什么视频生成要异步轮询？

渲染通常需要数十秒到数分钟。创建接口只返回 `task_id`，需轮询直到 `status` 成功后再取 `content.video_url`，不要把创建当同步渲染。

### 有参考视频和没有参考视频价格差多少？

以截图 Seedance-2 分组为例（**以页面实时为准**）：同分辨率下「包含参考视频」单价更低，例如 480p/720p 为 $11.520 vs $19.260（$/1M tokens）。

### 如何试用 TryAllAPI？

站点首页：[https://tryallapi.com/](https://tryallapi.com/)；注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把 Seedance 接到业务里，核心不是背一堆营销话术，而是：**在模型广场确认 model id 与异步 endpoint → 读懂分辨率 / 是否含参考的价格表 → 用同一套 POST/GET 任务协议出片**。截图里的 `doubao-seedance-2-5-260628` 与 `/api/v3/contents/generations/tasks` 已足够搭通最小链路。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 相关仓库 | [awesome-seedance-2.5](https://github.com/xianyu110/awesome-seedance-2.5) |

控制台 Token、定价与文档更新后，以线上为准。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。
