# AGENTS.md

## SCUCT Project Charter

本项目目标：构建一个可在 Android / iOS 运行的四川大学课表应用（SCUCT），强调工业级稳定性与 Swiss Design 风格的一致体验。

技术栈固定为：Flutter + Riverpod + Dio + Isar + flutter_secure_storage + build_runner。

## 0. TOP1 级别红线（Release Blocker）

- **TOP1 原则：数据正确性优先于一切视觉与交互效果。**
- 若登录态、当前周计算、课程时间冲突解析存在不确定性，**禁止发布**。
- 当远端数据异常或本地缓存失真时，允许降级为“同步失败/待重试”状态，**禁止展示猜测值**。
- 任一 TOP1 问题必须按 P0 处理：立即记录、可复现、可回滚、可追踪。

---

## 1. Design System (Swiss / International Typographic Style)

### 1.1 色彩系统
- 基础色：`#FFFFFF`（极简白）/ `#121212`（磨砂黑）/ `#F5F5F7`（现代灰）。
- 功能色：按课程类型使用高饱和低明度纯色，仅用于课程区分，不做大面积背景渲染。
- 强调色：`#B11E24`（川大红），仅用于关键 CTA（如“登录”“保存”）。

### 1.2 网格与间距
- 严格遵循 8pt Grid。
- 卡片圆角：`4px - 6px`。
- 组件间距优先使用 8 的倍数，避免任意值。

### 1.3 字体与可访问性
- 字体优先系统字体：iOS `SF Pro`，Android `Roboto`，Web `Inter`。
- 必须支持系统动态字号缩放（TextScaleFactor）。
- 文本对比度满足可读性要求（至少接近 WCAG AA）。

---

## 2. Architecture (Clean + Feature-First)

目录约束如下（`timetable` 与 `auth` 保持同等级分层）：

```txt
lib/
├── core/
│   ├── network/         # Dio client, interceptors, retry/backoff
│   ├── storage/         # secure storage + isar bootstrap
│   ├── theme/           # app theme tokens/components
│   ├── l10n/            # localization setup
│   └── utils/           # cross-feature helpers
├── features/
│   ├── auth/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── timetable/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── settings/
│       ├── data/
│       ├── domain/
│       └── presentation/
└── shared/
    ├── widgets/
    └── constants/
```

规则：
- `domain` 不依赖 Flutter UI 与第三方框架实现细节。
- `presentation` 仅通过 use case 与 domain 交互。
- 跨 feature 复用放入 `core/` 或 `shared/`，禁止互相直接引用 data 层。

---

## 3. Core Feature Blueprint

### 3.1 Auth Module (Enhanced)
- **验证码处理 (Captcha Strategy)**: 
  - 支持教务系统的图形验证码/滑块。
  - **解耦逻辑**: 网络层通过 `CaptchaRequiredException` 抛出信号，由 Presentation 层监听并弹出 Overlay 输入框，用户手动填写输入后重放（Retry）原始请求。
- **Token 管理**: 支持教务系统登录流并提取会话凭证。
- **加密存储**: Token 必须存储在 `flutter_secure_storage`，禁止明文持久化。
- **并发锁**: 实现 **single-flight refresh lock**：并发 401 时仅允许一次 refresh，请求排队重放，避免 token 竞争。

### 3.2 Timetable Engine (Enhanced)
- **视图**:
  - Day View：显示今日课程与进度。
  - Week View：7 x (5~6) 网格，并处理重叠课程。
- **异常周次逻辑 (Override Logic)**: 
  - 支持本地手动标记“停课”、“调课”、“调休补课”。
  - 核心 Model 需包含 `isCancelled` (bool) 和 `note` (String) 字段，UI 需体现“手动修改”状态，优先级高于 API 返回值。
- **冲突课程展示**: 使用层叠卡片或切片布局，点击展开详情。

### 3.3 Localization
- 使用 `.arb` 文件。
- 首批语言：`zh_CN`、`en_US`。
- 专业术语（如“必修/选修”）必须通过 l10n 映射，禁止硬编码 API 文案直出。

---

## 4. Privacy & Data Security (Privacy Gate)

- **PII 保护**: 严禁将学号、姓名、身份证号等个人身份信息（PII）上报至任何第三方日志或监控平台（如 Sentry, Firebase Analytics）。
- **日志混淆**: 在 Release 构建中，所有网络日志必须对 PII 字段进行脱敏处理（掩码展示）。
- **最小化存储**: Isar 数据库仅存储必要的课程逻辑，不存储用户敏感隐私信息（如家庭住址等）。

---

## 5. Abnormal Week & Holiday Logic

- **法定节假日适配**: 预留 `HolidayDataSource` 接口，允许通过配置或第三方 API 拉取国家法定放假安排。
- **逻辑层冲突**: 当“法定调休”与“常规周次”冲突时，手动覆盖逻辑（Manual Override）拥有最高权重。
- **UI 表征**: 停课课程需置灰或添加斜杠纹理，并标注“停课”字样。

---

## 6. Persistence & Query Design

Isar 建模要求：
- 课程表需包含 `semesterId` 维度，避免跨学期污染。
- 索引建议：`semesterId + dayOfWeek + slotStart`。
- `weeks` 推荐位图（bitmask）或等效高效结构，优化“当前周筛选”。
- 缓存表包含 `syncFingerprint` 与 `lastSyncedAt`。

### 6.1 Offline-First Sync Contract（必须）
- 首屏固定流程：**先读 Isar 缓存 -> 立即渲染 -> 后台静默同步**。
- 指纹优先级：`ETag` / `Last-Modified` > `canonical JSON hash`（字段排序后 UTF-8 哈希）。
- 仅当指纹变化时才触发 Provider 更新与 UI 重绘；指纹不变时禁止无效刷新。
- 同步失败时保留上次可用缓存并给出可恢复提示（重试/下拉刷新）。

---

## 7. Network & Reliability

Dio 规范：
- 拦截器顺序：Auth 注入 -> 重试 -> 日志（开发环境）。
- Retry 使用指数退避 + 抖动（jitter），限制最大重试次数。
- 对登录态失效与网络错误做明确区分，不得统一提示“未知错误”。

---

## 8. Week Calculation Rules (Must Be Deterministic)

周数计算必须固定并可测试：
- 使用统一时区（建议 `Asia/Shanghai`）。
- 明确周起始日（建议周一）。
- 公式固定：`week = floor((today_noon - semester_start_noon) / 7) + 1`，并对范围做 clamp（<1 记为 1）。
- 必测边界：跨月、跨年、闰年 2 月、学期首日非周一、手动覆盖开关切换。
- 支持手动覆盖优先级高于自动计算，并可一键恢复自动模式。

---

## 9. Testing Baseline (DoD Gate)

合入主分支前至少通过：
- JSON Transformer 单测（字段缺失、类型异常、周次边界）。
- 周数计算单测（学期首周、跨月、跨年、手动覆盖）。
- 冲突课程布局单测（重叠 2 门、3 门、不同周重叠）。
- Auth 刷新流程单测（并发 401 single-flight）。

### 9.1 Toolchain & CI Gate（必须）
- 统一 SDK：Dart `3.10.x`，Flutter 使用固定版本（建议 FVM 锁定）。
- 本地与 CI 必跑：
  - `flutter pub get`
  - `dart format --set-exit-if-changed .`
  - `flutter analyze`
  - `flutter test`
  - `flutter build apk --debug`
  - `flutter build ios --no-codesign`（macOS runner）
- 任一门禁失败禁止合入 `main`。

---

## 10. Iteration Plan (Execution Order)

1. 初始化 `pubspec.yaml` 与基础目录。
2. 配置 `l10n`（`zh_CN` / `en_US`）与主题 token。
3. 定义 domain entity + data model + Isar schema。
4. 接入 Auth 与 Dio 拦截器（含 refresh lock 与验证码异常）。
5. 实现 Timetable Week View / Day View 与冲突布局。
6. 加入离线同步与指纹判定。
7. 完成测试基线并进行首轮性能检查。

---

## 11. Non-Negotiables

- **禁止** 把业务状态散落在 Widget 层，统一通过 Riverpod 管理。
- **禁止** 在 UI 中直接发网络请求。
- **禁止** 明文存储凭证。
- **禁止** 未建模就直接消费 API 原始 JSON。
- **禁止** 为赶进度牺牲结构一致性。
- **禁止** 在日志中泄露用户隐私字段（PII）。

---

## 12. Cross-Platform UX / Adaptivity / Typography

### 12.1 Android + iOS 差异与交互一致性
- **滚动物理遵循平台**: iOS 使用 `BouncingScrollPhysics`；Android 使用 `ClampingScrollPhysics`（或平台默认），禁止强制统一为 iOS 回弹。
- **导航交互**: 遵循平台习惯：Android 返回键、iOS 侧滑返回。
- **全局处理 `SafeArea`**: 避免刘海屏遮挡关键内容。
- **键盘感知**: 键盘弹出时输入区必须可见，禁止遮挡输入框。

### 12.2 屏幕适配
- 禁止写死宽高，优先使用 `Flex/Expanded/LayoutBuilder`。
- 定义断点布局，为大屏/平板提供差异化网格显示策略。

### 12.3 字体与排版
- 统一排版 token（字号、字重、行高），而非字体文件。
- 字体家族优先系统字体。
- 支持系统字体缩放时仍需保证核心字段（教室、课程名）可见。

---

## 13. Code Style & Documentation Discipline

### 13.1 禁止过度防御性编程
- 仅对真实可预期的失败路径做保护，不得为了“看起来稳”滥加空判断与兜底分支。
- 禁止无差别 `try-catch` 吞错；异常必须带上下文并可追踪。
- 优先使用 fail-fast 与清晰错误状态，而不是层层 if/else 掩盖问题。

### 13.2 文档要好，但不能文档泛滥
- 文档必须服务实现：架构边界、数据契约、关键决策要写清楚。
- 禁止“只堆文档不落实现”的提交节奏，文档与代码应同步演进。
- 无用/过期文档必须在相关功能变更时同步删除，避免误导后续开发。

### 13.3 文件长度与复杂度硬限制（必须）
- 业务代码文件默认控制在 **200 行以内**（不含自动生成文件、平台模板与配置文件）。
- 若超过 200 行，必须拆分为更小的组件/用例/服务文件并说明拆分边界。
- 控制嵌套深度：单函数最大嵌套层级不超过 **3 层**；超过时必须通过卫语句或函数提取降复杂度。
- 禁止超长函数（建议单函数不超过 40~60 行），优先小函数组合。
