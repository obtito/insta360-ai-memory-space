# AI Memory Space — 技术设计与 48h 落地

> **定位**：把「AI Memory Space」的产品定义（见《核心价值压缩》）落成 48 小时可实现的工程方案。
> **一句话**：把一段 360° 素材，变成一份包含「时间 + 空间 + 人物 + 事件」的**可计算记忆**。
> **核心架构原则（全文贯穿）**：
> 1. **语义与审美分离**——大模型只做"理解"，不做逐帧/逐像素决策；
> 2. **大模型不进实时环**——只在低频（一次查询 / 一次偏好反馈）被调用；
> 3. **下游全确定性**——空间定位、相机路径、渲染全部是确定性算法，可复现、可测试、秒级重跑。

---

## 一、总体架构（数据流）

```
360° 素材（等距柱状 equirectangular）
        │
        ▼
【空间事件理解层】detector/tracker + 场景识别
   ├─ Event Timeline   —— "发生了什么"：事件类型 + 时间戳 + 置信度
   ├─ Spatial Map      —— "哪里发生"：事件发生的球面位置 (θ, φ)
   └─ Subject Track    —— "谁"：主体轨迹 + 身份（本人 / 朋友 / …）
        │
        ▼
【记忆精选层 · Personal Curation】选/删/藏三信号 → 偏好权重 → Memory Scene 排序
        │
        ▼
【意图/查询层 · Intent Director】自然语言 → query.json（白名单 Schema）→ 语义锚定
        │
        ▼
【空间重构层】360° 帧 + 深度 → 可导航空间（视差 / 3DGS / 透视重投影）
        │
        ▼
【Spatial Gallery】PC 手势浏览（抓/拉近/旋转/推入）+ VR 进入
```

**关键**：大模型只出现在「意图/查询层」与「偏好反馈」两处；`query.json` 一旦落定，下游（空间定位 → 重构 → 渲染）**不再碰任何 LLM 文本**。这是 48h 内做得出、现场不崩的保证。

---

## 二、意图 → 查询白名单 Schema（鲁棒性的骨架）

### 2.1 完整 Schema（字段枚举/有界，禁止自由生成）

```jsonc
{
  "query": {
    "target": "event" | "subject" | "place",        // 查什么：事件 / 人物 / 位置
    "subject": "auto" | "self" | "friend" | "<id>", // 目标主体
    "event": "jump"|"landing"|"crash"|"summit"|"gathering"|"sunset"|"...",
    "time_ref": "first" | "last" | { "nth": 1..9 } | "all",
    "place_ref": "where" | "<spatial_anchor>"       // "当时朋友站在哪里"
  },
  "experience": {
    "mode": "replay" | "orbit" | "first_person",    // 进入方式
    "pace": "fast" | "normal" | "slow"
  }
}
```

### 2.2 语义锚定（关键推理步）

大模型**不直接索引素材**，只输出抽象锚点；由确定性层解析到「时间戳 + 空间位置」：

- 输入："当时我朋友站在哪里？"
- VLM 输出：`query = { target: subject, subject: friend, place_ref: where }`
- 确定性层：从 Subject Track 找 `friend` 在当前事件的 `(θ, φ)` → 把相机朝向该方向。

**So what**：VLM 只做"把自然语言翻译成结构化锚点"，把"猜朋友站在哪一格"这个易错动作交给确定性定位——鲁棒性来源。

### 2.3 校验与降级（fallback）

- Schema 校验器：字段不在枚举内 → clamp 到默认值。
- **默认 query ≈ 展示当前精选记忆**（即"AI 替你挑好的"，等于影石已有能力）。**最差情况退化成影石基线，绝不比基线差。**
- 唯一自由文本（如未来加的字幕）限长、纯叠加、不参与定位决策。

---

## 三、空间事件理解（"发生了什么" + "哪里发生"）

- **Event Timeline**：检测器识别动作/场景事件 → `{type, t, score}`；合并同一动作窗口、去重（避免一个起跳记两次）。
- **Spatial Map**：事件发生时的球面位置 `(θ, φ)`，取主体 bbox 中心转球面坐标。
- **Subject Track**（球面坐标）：
  - 像素 `(u,v)` ↔ 球面 `(θ,φ)`：`θ = 2π·u/(2W)`，`φ = π·v/W`；
  - 检测：YOLO/RT-DETR 逐帧检测 → bbox 中心 → `(θ,φ)`；MVP 用降采样 + 少主体即可；
  - 去噪：**θ 在 0/2π 处先 unwrap** 再平滑（避免跨接缝跳变）→ Kalman/Savitzky-Golay → 速度/加速度限幅。

---

## 四、Personal Curation（三信号偏好，不做人格测试）

- **只留三个信号**：用户**选择 / 删除 / 收藏**。
- 闭环：AI 选 A/B/C → 用户删 A、留 B、藏 C → 偏好权重更新（简单计数/贝叶斯，如 `P(偏爱人物表情) ↑`）→ 下一轮排序更准。
- Memory Scene 排序 = `event score × 偏好权重`。
- **不做**：八维人格、长期画像、复杂推荐系统（避免 scope 爆炸）。

---

## 五、空间重构（360° → 可进入）

**关键概念**：不是"2D → 3D 特效"，是"可观看 → 可进入"。

| 方案 | 做法 | 取舍 |
|------|------|------|
| A（推荐） | 复用影石 3D 时光舱 / 3DGS 能力（若 SDK 可用） | 效果最好，但依赖 SDK 现场可用 |
| B | 单帧深度估计 + 前景/背景分层 + 视差位移 | 可自研，视觉冲击足，性价比高 |
| C | 360° 球面重投影 + 相机路径规划（复用 reframe） | 纯确定、最稳，作为 fallback |

输出：可导航 Memory Scene（用户可旋转 / 拉近 / 进入）。

---

## 六、Spatial Gallery + 手势

- 手势 = 操纵**记忆空间**，不是"空中点击"：**抓**（取出记忆）→ **拉近**（查看空间）→ **旋转**（不同角度）→ **推入**（进入 3D/VR）。
- Demo 载体：PC Web（Three.js / 原生 WebGL）。

---

## 七、48h 里程碑（含自评线）

| 里程碑 | 时间 | 交付物 | 自评线（gate） |
|------|------|------|------|
| **M0 素材 + Schema** | 0–6h | 360 加载 + query Schema + 校验器 + fallback | 20 条查询句 → **100% 合法 JSON** |
| **M1 空间事件理解** | 6–18h | Event Timeline + Spatial Map + Subject Track | 主体丢失率 <5%；10 个标注事件召回 ≥7 |
| **M2 语义锚定 + 定位** | 18–30h | 自然语言 → (时间戳, 空间位置) | 5 条查询定位命中 ≥3/5（含"朋友站在哪里"） |
| **M3 空间重构 + Gallery** | 30–42h | 可导航 Memory Scene + 手势浏览 | 旋转/拉近/进入流畅；切换 <5s |
| **M4 演示 + 叙事** | 42–48h | VR + 演示视频 | 主线"从可观看 → 可进入" |

**每阶段留 2h buffer**；SDK 有坑 → fallback 到离线 equirectangular MP4（算法层与素材源解耦）。

---

## 八、技术栈与降级

| 模块 | 技术 | fallback |
|------|------|----------|
| 意图/查询 | 阿里 Qwen-VL（多模态） | 文本 LLM + 规则解析 |
| 主体检测/跟踪 | YOLO/RT-DETR + ByteTrack | 影石 SDK 主体/深度数据 |
| 轨迹平滑 | Kalman / Savitzky-Golay | 指数平滑 |
| 空间重构 | 3DGS / 深度估计 + 视差 | 球面重投影 + reframe |
| 素材访问 | Insta360 Media SDK | 离线 equirectangular MP4 |
| 呈现 | Web（Three.js / WebGL） | 本地脚本 + 视频输出 |

---

## 九、唯一工程焦点

整个黑客松最值得投入的工程资源，集中在一个 Demo 链路上：

> 从一段**真实的影石 360° 素材**中，AI 理解一个事件 → 用户用**自然语言**找到这个事件 → 系统从**正确的空间位置**重新呈现它。

这条链路一旦跑通，AI 选片、3D、手势、VR 都会从"功能"变成这个核心能力的不同表现形式。

**真正的产品问题（评判标准）**：能不能证明"一段影石 360° 素材不是一段视频，而是一份包含 **时间 + 空间 + 人物 + 事件** 的可计算记忆"？

---

## 附录：影石已下放能力实锤（为什么不做单点功能）

| 功能 | 现状 |
|------|------|
| AI 导演 / 一键成片 2.0（PanoMind） | X6 首发；X5 固件 V1.13.21 下放 |
| PanoMind 场景识别 | 30+ 场景（1 万小时训练） |
| 手势 / 语音控制 | 机内自带 |
| 全景 AI 构图 | 自动识别主体优化构图 |
| 3D 时光舱（3DGS） | X6/X5/X4 Air 支持，月 4 次免费 |

来源：[AI 导演官方指南](https://onlinemanual.insta360.com/app/zh-cn/operation_tutorial/edit_function/director) · [一键成片官方指南](https://onlinemanual.insta360.com/app/zh-cn/operation_tutorial/edit_function/auto-edit) · [X5 固件下放 AI Director](https://www.mundoconectado.com.br/cameras/insta360-x5-ai-director-spatial-capture-firmware/) · [3D 时光舱官方指南](https://onlinemanual.insta360.com/app/zh-cn/operation_tutorial/edit_function/3dgs) · [Spatial Capture 支持机型](https://www.androidauthority.com/insta360-spatial-capture-launch-3713073/) · [X5 语音控制 FAQ](https://onlinemanual.insta360.com/x5/ja-jp/faq/functionality/voice)

**结论**：影石把"单点 AI 功能"做满并全线下放，但全部停在同一层——**通用黑盒 + 事后离线 + 单设备 + 不可交互**。所以不做单点，只做它们之上的**记忆层**（AI Memory Space）。

---

*本设计与 [AI_Memory_Space_核心价值压缩.md](AI_Memory_Space_核心价值压缩.md) 对齐；`query.json` 是"平台复用"的中间表示，未来可作影石多设备 AI 的统一意图入口。*
