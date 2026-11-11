
# 基于智能体的个性化语境式英语词汇学习平台 · 设计方案

> **[▶ 在线预览 MVP 原型](https://zlzhangliang666.github.io/english-vocab-platform/)**（GitHub Pages，单文件 HTML，无需安装，点开即用）
>
> 前端设计文档：[英语词汇学习平台_前端设计.md](./英语词汇学习平台_前端设计.md)

| 项目 | 内容 |
|---|---|
| 文档版本 | v1.3 |
| 日期 | 2026-11-10 |
| 后端技术栈 | Java 21 LTS + Spring Boot 3.3+ |
| 前端技术栈 | Vue 3 + TypeScript + Vite + Pinia，原生 CSS 变量 |
| 数据存储 | PostgreSQL 16 + Redis 7 + MinIO |
| 前端形态 | 响应式 Web（后续可扩展小程序） |
| 设计依据 | NGSL 1.2 词表、《告诉你外语学习的真实方法与误区分析》、《找对英语学习方法的第一本书》、间隔重复算法长文两篇 |

---

## 1. 设计依据

### 1.1 资料盘点

| 资料 | 可用内容 | 在系统中的作用 |
|---|---|---|
| NGSL 1.2 词表 | 2809 个 NGSL 词元、47 个补充词（SUP）、955 个学术词（NAWL） | 选词与分级的底座 |
| `NGSL+with+SFI+(31K).xlsx` | 31,240 行，含 SFI 排名、SFI 值、每百万调整频次 U、分布均匀度 D、单词覆盖率、累计覆盖率 | 计算"先学谁" |
| `NGSL_1.2_with_English_definitions.xlsx` | 2809 条英文释义 | 释义兜底与英英解释 |
| `NGSL_1.2_lemmatized_for_teaching.csv` | 词元与词形对应 | 词族识别、文本词形还原 |
| 《告诉你外语学习的真实方法与误区分析》 | 常见误区清单、可理解输入、无效输入、4 秒声音缓存、发音取决于听辨 | 决定"怎么学" |
| 《找对英语学习方法的第一本书》 | 70%–80% 可懂度、20%–30% 生词、人造语言环境的可控重复、输入优先、成人习得周期 | 决定"语境难度与复现机制" |
| 《间隔重复记忆算法：e 天内，从入门到入土。》 | SM-0 / SM-2 / SM-4 经验算法，记忆三变量模型，稳定性增长 SInc，记忆复合性，DSR 模型与 SRS 模拟，SSP-MMC | 决定复习调度与卡片原子化 |
| 《KDD'22 \| 墨墨背单词：基于时序模型与最优控制的记忆算法》 | DHP 模型、2.2 亿条记忆行为数据、SSP-MMC 最优策略与离线模拟结果 | 决定调度算法的演进路线 |

### 1.2 NGSL 实测结论

按 `NGSL+with+SFI+(31K).xlsx` 的 `Coverage` 列累加，得到：

| 累计词数 | 文本覆盖率 |
|---|---|
| 500 | 74.3% |
| 1000 | 81.8% |
| 1500 | 85.8% |
| 2000 | 88.3% |
| 2500 | 90.0% |
| 2809 | 90.8% |

三条可直接用于产品的结论：

1. **覆盖率增长是强递减的**。前 1000 词买到 81.8%，后 1809 词只多买 9 个百分点。选词必须按"覆盖收益"排序，而不是按字母序或教材单元。
2. **频次高不等于该早学**。`unclear` 每百万出现 106 次，但 `D`（分布均匀度）仅 0.05，说明它高度集中在少数文本类型。D 值低于阈值的词需要降权。
3. **词表天然分层**。NGSL（通用）→ SUP（补充）→ NAWL（学术），可以直接映射为三条学习路径，不需要重新造分级体系。

### 1.3 方法论到系统机制的映射

| 资料观点 | 系统机制 |
|---|---|
| 听不懂的输入等于噪音 | 语境生成后必须做"已知词覆盖率"校验，不达标重写 |
| 材料听懂 70%–80% 最合适 | 听力材料按 75% 可懂度生成；阅读材料按 95% 已知词生成 |
| 真实环境里同一表达复现率极低 | 由系统强制同一目标词在多个语境、多个时间点复现 |
| 声音印象只保留约 4 秒 | 目标词在单篇内至少出现 2 次，且不连续；听力材料做 3 次间隔复现 |
| 输入是必要且充分条件，输出不是 | 主线是读/听后的再认与回忆，输出题只用于暴露漏洞 |
| 成人达到流利约为年龄 ÷ 6 年 | 学习目标按"可读文本比例"设定，不承诺短期流利 |
| 发音问题源于听辨 | 拼写与辨音题使用最小对立对，并提供多口音音频 |

### 1.4 资料缺口

以下内容资料区没有提供，必须在方案中补齐：

- 中文释义：NGSL 只有英文释义，需生成后人工抽检
- 例句与真实搭配：需引入语料库或由模型生成
- 音频：需 TTS 或外部发音库
- 义项拆分：NGSL 按词元组织，不区分多义项，需自建 `word_sense`
- 考试对齐（CET/IELTS/TOEFL）：需外部映射表
- 领域词表：NAWL 只有 955 个学术词，技术、商务等领域词需另行准备

### 1.5 间隔重复资料的关键结论

这两篇资料把"复习"从经验规则变成了可计算的模型，其中六条直接改变系统设计：

**1. 记忆状态是三个变量，不是一个"掌握度"。**

- 记忆稳定性 `S`：回忆概率从 100% 降到 90% 所需的时间（部分文献用半衰期，即降到 50%）
- 记忆可提取性 `R`：某一时刻的回忆概率
- 记忆难度 `D`：材料内在难度

遗忘曲线近似指数函数：

```
R(t) = exp(-t / S)
```

**2. 稳定性增长有三条已被数据验证的规律。**

- `S` 越大，稳定性增长 `SInc` 越小（负幂关系）：越熟的词越难再提升
- `R` 越低，成功后 `SInc` 越大（指数关系）：这就是间隔效应
- `D` 越大，`SInc` 越小：难词需要更多次复习

资料特别指出：当 `R = 100%` 时，`SInc` 可能低于 1，即"刚看完就重复"不仅浪费时间，还可能损害记忆。

**3. 复合记忆的稳定性低于任一子记忆。**

复合记忆的可提取性是子记忆可提取性之积，稳定性会被拉到较难的那个子记忆的水平。直接推论：**一个单词不能做成一张"词义 + 拼写 + 搭配"的复合卡**，必须拆成原子卡。这为 5.3 中四个独立掌握度字段提供了理论依据，而不是产品偏好。

**4. 经验算法的演进给出了参数先验。**

| 算法 | 机制 | 可借用之处 |
|---|---|---|
| SM-0 | 间隔 1 / 7 / 16 / 35 天，之后 ×2 | 初始间隔先验 |
| SM-2 | 引入简易度 EF（初值 2.5）与回忆评分 q | 失败后重置、成功则拉长 |
| SM-4 | 最优间隔矩阵，跨卡片共享调整经验 | 参数应随用户群体更新 |
| SM-17 | 稳定性增长 SInc 的命名与分解 | 状态转移方程的写法 |

**5. 调度可以写成"随机最短路径"。**

SSP-MMC 把每张卡的复习看作马尔可夫决策过程：状态是 `(S, D)`，动作是"下次间隔"，成本是复习耗时，目标是"在成本约束下让尽可能多的材料达到目标稳定性"。

资料给出的离线模拟参数与结果：

- 目标半衰期 360 天，模拟 1000 天，每日成本上限 600 秒
- 成功回忆计 3 秒，失败计 9 秒
- 达到 6000 个目标词：SSP-MMC 用 466 天，Anki 用 569 天，THRESHOLD 用 533 天
- 相对 THRESHOLD 节省约 12.6% 的复习时间

**6. 复习目标不该是"清空队列"。**

资料指出最大期望稳定性增长出现在 `R ≈ 30%–40%`，但这不等于最高学习效率；真正的最优解是在时间预算内最大化长期留存。因此产品的复习指标应定义为"在每日时长约束下，目标稳定性达成量"，而不是"今日待复习数归零"。

对本方案的净影响：

- 6.4 从"自造间隔公式"改为 DSR 体系，并按 P0/P1/P2 分阶段落地
- 5.3 的四维拆分从"设计选择"升级为"算法要求"
- 前端"复习"页必须展示 `S` 与 `R`，让用户理解"为什么现在复习它"

---

## 2. 产品定义

### 2.1 一句话定位

不是"背单词 App"，也不是"每天一篇外刊"。

它是**一套以可理解输入为核心的教学系统**：用你已经认识的词，搭出一篇只差一点的英文，让 8 个新词在语境里反复出现，直到你不查词典也能读懂。

特色只有一句：**每一篇都保证对你"只难一点点"，并且这个判断可以验证。**

基本盘同样齐全：词书与词表进度、生词本、闪卡复习、任意查词、离线与导出。特色决定用户为什么留下，基本盘决定用户为什么能开始。

### 2.2 目标用户

| 类型 | 特征 | 默认路径 |
|---|---|---|
| 备考型 | 有明确考试目标，时间紧 | 考试词表优先，覆盖率按考试文本校准 |
| 阅读型 | 想读英文文档、论文、新闻 | 领域词表优先，语境取真实文本风格 |
| 兴趣型 | 想看懂剧、小说、游戏 | 兴趣主题优先，语境偏叙事与对话 |
| 恢复型 | 学过多年但长期不用 | 先做测词，重排在"半生不熟"区间 |

非目标用户：零基础儿童、需要系统语法课程的学习者、以口语陪练为核心诉求的用户。

### 2.3 核心假设

1. 学习者愿意每天投入 15–25 分钟，但不愿做无上下文的词表打卡。
2. "今天该学什么"由系统算出来，比用户自己挑更有效。
3. 语境只要足够可理解，重复密度足够，词汇留存会显著优于列表记忆。
4. 用户能感知到"智能体在替我编排"，并因此更愿意坚持。

### 2.4 成功指标

| 层级 | 指标 | 目标 |
|---|---|---|
| 北极星 | 周留存 ≥ 4 天 | ≥ 35% |
| 学习效果 | 30 天后同词再认正确率 | ≥ 80% |
| 学习效果 | 目标文本未知词比例下降 | 相对下降 ≥ 15% |
| 体验 | 语境"读得下去"评分 | ≥ 4.0 / 5 |
| 特色 | 读懂率月增幅（百分点） | ≥ 2 |
| 基本盘 | 词书 30 天完成率 | ≥ 25% |
| 基本盘 | 生词本 7 日内回访率 | ≥ 40% |
| 系统 | 今日会话首屏 P95 | ≤ 300 ms |
| 系统 | 语境生成成功率（含重试） | ≥ 98% |

### 2.5 产品结构：特色层与基本盘

| 层 | 内容 | 作用 | 是否差异化 |
|---|---|---|---|
| 特色层 | 可读输入引擎：95% 已知词约束、目标词跨篇复现、读懂率曲线 | 决定留存与口碑 | 是，竞品无同等实现 |
| 基本盘 | 词书与词表进度、生词本、闪卡复习、任意查词、测词定级、离线、导出 | 决定"能不能用" | 否，做到行业平均以上即可 |
| 场景层 | 四六级冲刺 30 天、技术文档阅读 | 决定"愿不愿意开始" | 半差异化，把特色装进具体目标 |

三条约束：

1. **基本盘不复造**。词书、生词本、闪卡行业已有成熟形态，直接对齐，不投入差异化预算。
2. **特色层只做一件事**：保证并证明"这一篇对你是 i+1"。任何偏离这件事的功能都不进特色层。
3. **场景层只做两个**。先验证四六级与技术文档，不做第三个。

**为什么是教学系统而不是工具**

工具解决一次性任务，教学系统解决能力增长。本产品的差异不在"更方便地查词"，而在"替你决定读什么、读多难、什么时候再见面"。因此它需要有课程感（每日计划、阶段目标），但不能有课程病（固定课表、统一进度、章节解锁）。

判断标准：**用户每周只打开三次也能用，打开二十次也不会腻。**

### 2.6 特色：可读输入的三条承诺

特色必须能被用户当场检验，否则只是话术。对外只讲三条：

1. **只难一点点**：每篇材料的已知词 ≥ 95%，且这个数来自你的真实答题记录，不是内容标签。
2. **一定会再见**：8 个目标词在本文出现 2 次，在随后 3 篇里各出现至少 1 次。复现由系统排，不靠运气。
3. **能看见移动**：读完一篇，读懂率变化即时可见；若连续 30 天月增幅低于 2 个百分点，系统改选词策略并向用户说明。

这三条同时是可验收的功能点，也是与"每日一篇外刊""AI 生成文章"划清界限的地方：别人提供内容，我们提供**受控的难度曲线**。

---

## 3. 设计原则

| # | 原则 | 资料依据 | 系统落点 |
|---|---|---|---|
| P1 | 按覆盖收益选词 | 覆盖率强递减 | `SelectionService` 评分函数 |
| P2 | 语境必须可理解 | 无效输入理论 | `ComprehensibilityVerifier` |
| P3 | 重复由系统制造 | 真实复现率低、4 秒缓存 | 目标词多次跨语境复现 |
| P4 | 输入优先，输出检验 | 输入必要且充分 | 题型权重与主流程顺序 |
| P5 | 成人路径不照搬儿童 | 成人习得周期 | 目标设定与文案口径 |
| P6 | 卡片必须原子化 | 复合记忆稳定性低于子记忆 | 四维拆分、题型分离 |
| P7 | 特色与基本盘分账 | 教学系统先可用，再特别 | 迭代排期与预算分配（见 2.5） |

补充约束：

- **不做**统一遗忘曲线。复习间隔由 DSR 状态与目标留存率计算（见 6.4）。
- **不做**"背完即复习"。`R = 100%` 时稳定性增长可能小于 1，刚学完立刻重复是负收益。
- **不做**"每日一词"式内容流。每天的词必须来自同一份学习计划。
- **不做**纯词根词缀拆解作为主要记忆手段。词源只作为辅助锚点，不进入评分。
- **基本盘不占首屏**。首屏永远只服务"今天读什么"。
- **特色入口全站只有一个**。不做第二个"智能阅读"或"AI 精读"。

---

## 4. 总体架构

### 4.1 技术选型

| 层次 | 选型 | 说明 |
|---|---|---|
| 语言 | Java 21 LTS | 虚拟线程处理 LLM 长调用 |
| 框架 | Spring Boot 3.3+ | Web、Validation、Actuator |
| 持久层 | Spring Data JPA + `JdbcClient` | JPA 管实体，复杂统计走原生 SQL |
| 数据库 | PostgreSQL 16 | JSONB 存计划与轨迹，窗口函数做统计 |
| 缓存 | Redis 7 + Caffeine | 词典本地缓存，用户已知词位图放 Redis |
| 消息/异步 | Spring `@Async` + 虚拟线程 | MVP 不引入 MQ，后续可换 RabbitMQ |
| 对象存储 | MinIO | 音频与生成文本快照 |
| 鉴权 | Spring Security + JWT | 无状态，支持后续小程序 |
| LLM | Spring AI（OpenAI 兼容协议） | 可切换本地 Ollama 降级 |
| 文档 | springdoc-openapi | 自动生成 OpenAPI 3 |
| 迁移 | Flyway | 表结构与词库导入脚本版本化 |
| 可观测 | Micrometer + Prometheus + Grafana | 关键链路埋点 |
| 测试 | JUnit 5 + Testcontainers + k6 | 单测、集成、压测 |
| 构建 | Maven 多模块 | 按领域拆分 |

选择理由：

- 选 PostgreSQL 而非 MySQL：JSONB（学习计划、智能体轨迹）、数组（领域标签）、窗口函数（覆盖率统计）都更顺手。
- 选 JPA 而非 MyBatis：领域模型稳定，CRUD 占多数；统计查询用 `JdbcClient` 直接写 SQL，避免引入两套映射。
- 不引入 Spring Batch：词库只导入一次，`ApplicationRunner` + POI 足够；避免为一次性任务增加复杂度。

### 4.2 模块划分

```
english-vocab-server
├── vocab-common          通用返回体、异常、工具、枚举
├── vocab-domain          实体、值对象、领域服务接口
├── vocab-infrastructure  持久化、Redis、LLM 客户端、TTS、对象存储
├── vocab-application     用例编排、事务边界
├── vocab-agent           智能体编排器、工具定义、提示词
├── vocab-api             REST 控制器、鉴权、参数校验
└── vocab-bootstrap       启动类、配置、Flyway、数据导入
```

依赖方向：`api → application → domain ← infrastructure`，`agent → application`。禁止 `api` 直接依赖 `infrastructure`。

### 4.3 运行时架构

```mermaid
flowchart LR
    FE[Web 前端] --> API[vocab-api<br/>REST + JWT]
    API --> APP[vocab-application<br/>用例编排]
    APP --> AG[vocab-agent<br/>编排器 + 工具]
    APP --> DOM[vocab-domain<br/>领域模型]
    AG --> DOM
    APP --> INF[vocab-infrastructure]
    INF --> PG[(PostgreSQL)]
    INF --> RD[(Redis)]
    INF --> S3[(MinIO)]
    INF --> LLM[LLM 服务<br/>OpenAI 兼容 / Ollama]
    INF --> TTS[TTS 服务]
```

### 4.4 代码目录

```
src/main/java/com/wordscape
├── common
│   ├── ApiResponse.java
│   ├── BizException.java
│   ├── ErrorCode.java
│   └── TraceContext.java
├── domain
│   ├── word/ Word.java WordForm.java WordSense.java Collocation.java
│   ├── learner/ UserAccount.java UserProfile.java UserWordState.java
│   ├── study/ StudySession.java Exposure.java Attempt.java
│   └── context/ ContextDoc.java
├── application
│   ├── PlacementService.java
│   ├── SelectionService.java
│   ├── ContextService.java
│   ├── DrillService.java
│   ├── ReviewService.java
│   └── ReportService.java
├── agent
│   ├── StudyAgentOrchestrator.java
│   ├── tool/ (WordQueryTool, ContextTool, DrillTool, ReviewTool)
│   ├── prompt/ PromptTemplates.java
│   └── AgentTraceService.java
├── infrastructure
│   ├── persistence/ (JPA Repository, JdbcQuery)
│   ├── cache/ KnownWordBitmapCache.java
│   ├── llm/ LlmClient.java
│   ├── tts/ TtsClient.java
│   └── storage/ ObjectStorageClient.java
└── api
    ├── AuthController.java
    ├── ProfileController.java
    ├── PlacementController.java
    ├── SessionController.java
    ├── ContextController.java
    ├── ReviewController.java
    ├── ReportController.java
    └── AgentTraceController.java
```

---

## 5. 领域模型

### 5.1 核心实体

| 实体 | 说明 | 关键字段 |
|---|---|---|
| `Word` | 词元 | lemma、wordlist、sfiRank、u、dispersion、coverage、中英释义 |
| `WordForm` | 词形 | form、formType（复数/过去式/派生） |
| `WordSense` | 义项 | pos、释义、例句、义项顺序 |
| `Collocation` | 搭配 | pattern、example、频次 |
| `UserAccount` | 账号 | email、passwordHash |
| `UserProfile` | 学习画像 | goal、domain、dailyMinutes、interests、targetExam |
| `UserWordState` | 词 × 维度的原子记忆状态，一个词四行 | dimension、mastery、stability(S)、difficulty(D)、nextDueAt |
| `ContextDoc` | 语境文本 | theme、body、wordCount、knownRatio、targetWordIds、status |
| `StudySession` | 每日会话 | sessionDate、planJson、status |
| `Exposure` | 曝光记录 | wordId、contextId、modality、position |
| `Attempt` | 作答记录 | wordId、itemType、isCorrect、errorType、responseMs |
| `AgentTrace` | 智能体轨迹 | step、rationale、toolName、latency、tokens |

### 5.2 ER 图

```mermaid
erDiagram
    USER_ACCOUNT ||--|| USER_PROFILE : has
    USER_ACCOUNT ||--o{ USER_WORD_STATE : tracks
    USER_ACCOUNT ||--o{ STUDY_SESSION : owns
    USER_ACCOUNT ||--o{ ATTEMPT : makes
    WORD ||--o{ WORD_FORM : has
    WORD ||--o{ WORD_SENSE : has
    WORD ||--o{ COLLOCATION : has
    WORD ||--o{ USER_WORD_STATE : referenced
    WORD ||--o{ ATTEMPT : about
    STUDY_SESSION ||--o{ EXPOSURE : contains
    STUDY_SESSION ||--o{ AGENT_TRACE : logs
    CONTEXT_DOC ||--o{ EXPOSURE : in
```

### 5.3 表设计

```sql
-- 词库
CREATE TABLE word (
    id                  BIGSERIAL PRIMARY KEY,
    lemma               VARCHAR(64)  NOT NULL UNIQUE,
    wordlist            VARCHAR(16),              -- NGSL / SUP / NAWL / EXT
    sfi_rank            INTEGER,
    sfi                 NUMERIC(6,2),
    u_per_million       NUMERIC(10,3),
    dispersion          NUMERIC(4,3),
    coverage            NUMERIC(8,6),
    cumulative_coverage NUMERIC(8,6),
    phonetic_uk         VARCHAR(64),
    phonetic_us         VARCHAR(64),
    audio_url           VARCHAR(255),
    cefr                VARCHAR(4),
    definition_en       TEXT,
    definition_zh       TEXT,
    domain_tags         TEXT[],
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_word_rank ON word (wordlist, sfi_rank);
CREATE INDEX idx_word_u ON word (u_per_million DESC);

CREATE TABLE word_form (
    id         BIGSERIAL PRIMARY KEY,
    word_id    BIGINT NOT NULL REFERENCES word(id),
    form       VARCHAR(64) NOT NULL,
    form_type  VARCHAR(16) NOT NULL,
    UNIQUE (word_id, form)
);
CREATE INDEX idx_form ON word_form (form);

CREATE TABLE word_sense (
    id            BIGSERIAL PRIMARY KEY,
    word_id       BIGINT NOT NULL REFERENCES word(id),
    pos           VARCHAR(16),
    definition_zh TEXT,
    definition_en TEXT,
    example       TEXT,
    sense_order   SMALLINT NOT NULL DEFAULT 1
);

CREATE TABLE collocation (
    id        BIGSERIAL PRIMARY KEY,
    word_id   BIGINT NOT NULL REFERENCES word(id),
    pattern   VARCHAR(128) NOT NULL,
    example   TEXT,
    frequency INTEGER DEFAULT 0
);

-- 学习者
CREATE TABLE user_account (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(128) NOT NULL UNIQUE,
    password_hash VARCHAR(100) NOT NULL,
    nickname      VARCHAR(64),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_profile (
    user_id        BIGINT PRIMARY KEY REFERENCES user_account(id),
    goal           VARCHAR(16)  NOT NULL,   -- EXAM / READING / INTEREST / RECOVERY
    domain         VARCHAR(64),
    target_exam    VARCHAR(16),
    daily_minutes  SMALLINT NOT NULL DEFAULT 20,
    interests      JSONB NOT NULL DEFAULT '[]',
    known_estimate INTEGER NOT NULL DEFAULT 0,
    known_ci_low   INTEGER,
    known_ci_high  INTEGER,
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 每个"词 × 维度"一行，对应一张原子记忆卡（见 6.4.2）
CREATE TABLE user_word_state (
    user_id         BIGINT NOT NULL REFERENCES user_account(id),
    word_id         BIGINT NOT NULL REFERENCES word(id),
    dimension       VARCHAR(16) NOT NULL,          -- RECOGNITION / RECALL / SPELLING / COLLOCATION
    mastery         SMALLINT NOT NULL DEFAULT 0,   -- 0-100，用于展示与选词
    stability       REAL     NOT NULL DEFAULT 0,   -- S，天
    difficulty      REAL     NOT NULL DEFAULT 0.3, -- D，0-1
    exposures       INTEGER  NOT NULL DEFAULT 0,
    lapses          INTEGER  NOT NULL DEFAULT 0,
    first_seen_at   TIMESTAMPTZ,
    last_seen_at    TIMESTAMPTZ,
    next_due_at     TIMESTAMPTZ,
    PRIMARY KEY (user_id, word_id, dimension)
);
CREATE INDEX idx_due ON user_word_state (user_id, next_due_at);

-- 学习过程
CREATE TABLE context_doc (
    id              BIGSERIAL PRIMARY KEY,
    theme           VARCHAR(32) NOT NULL,
    difficulty      VARCHAR(16) NOT NULL,     -- A2 / B1 / B2 / C1
    body            TEXT NOT NULL,
    word_count      INTEGER NOT NULL,
    known_ratio     NUMERIC(5,4) NOT NULL,
    target_word_ids BIGINT[] NOT NULL,
    source          VARCHAR(16) NOT NULL,     -- AGENT / TEMPLATE / HUMAN
    model_name      VARCHAR(64),
    prompt_version  VARCHAR(16),
    status          VARCHAR(16) NOT NULL,     -- DRAFT / VERIFIED / REJECTED
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE study_session (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL REFERENCES user_account(id),
    session_date DATE NOT NULL,
    plan_json    JSONB NOT NULL,
    status       VARCHAR(16) NOT NULL DEFAULT 'PLANNED',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, session_date)
);

CREATE TABLE exposure (
    id          BIGSERIAL PRIMARY KEY,
    session_id  BIGINT NOT NULL REFERENCES study_session(id),
    user_id     BIGINT NOT NULL,
    word_id     BIGINT NOT NULL REFERENCES word(id),
    context_id  BIGINT REFERENCES context_doc(id),
    modality    VARCHAR(8) NOT NULL,          -- READ / LISTEN
    position    INTEGER,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exposure_user_word ON exposure (user_id, word_id, created_at DESC);

CREATE TABLE attempt (
    id          BIGSERIAL PRIMARY KEY,
    client_id   VARCHAR(64) NOT NULL UNIQUE,  -- 幂等键
    user_id     BIGINT NOT NULL,
    word_id     BIGINT NOT NULL REFERENCES word(id),
    item_type   VARCHAR(24) NOT NULL,
    is_correct  BOOLEAN NOT NULL,
    error_type  VARCHAR(24),
    response_ms INTEGER,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agent_trace (
    id          BIGSERIAL PRIMARY KEY,
    session_id  BIGINT NOT NULL REFERENCES study_session(id),
    step        VARCHAR(32) NOT NULL,
    tool_name   VARCHAR(64),
    rationale   TEXT,
    payload     JSONB,
    latency_ms  INTEGER,
    token_in    INTEGER,
    token_out   INTEGER,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE daily_metric (
    user_id          BIGINT NOT NULL,
    metric_date      DATE NOT NULL,
    new_words        INTEGER NOT NULL DEFAULT 0,
    reviews          INTEGER NOT NULL DEFAULT 0,
    accuracy         NUMERIC(5,4),
    known_estimate   INTEGER,
    coverage_ratio   NUMERIC(5,4),
    PRIMARY KEY (user_id, metric_date)
);
```

### 5.4 词库导入

- 读取 `NGSL+with+SFI+(31K).xlsx` 的 `SFI adj` 工作表，按 `WL_SFI_Rank` 排序写入 `word`
- `Wordlist` 为空的词记为 `EXT`，只作为候选词，不进入默认学习路径
- 读取 `NGSL_1.2_with_English_definitions.xlsx` 回填 `definition_en`
- 读取 `NGSL_1.2_lemmatized_for_teaching.csv` 写入 `word_form`
- 导入任务用 `ApplicationRunner` + 幂等 upsert（`ON CONFLICT (lemma) DO UPDATE`），可重复执行
- 中文释义、音标、音频通过离线批处理补充，缺失时前端降级显示英文释义

---

## 6. 核心算法

### 6.1 已知词量估计

**目标**：在不做 2809 题的前提下，估计用户已掌握的词量与覆盖曲线。

**流程**：

1. 从 NGSL 按 SFI 排名分 10 层，每层抽 4 词，共 40 题（含 4 个伪词用于检测乱答）。
2. 每题给出词 + 四个中文释义选项。
3. 每层正确率 `p_i` 作为该层的掌握比例。
4. 用一参数 Logistic（Rasch）拟合曲线：

```
P(known | rank) = 1 / (1 + exp((rank - θ) / s))
```

- `θ`：用户水平参数，用最大似然估计
- `s`：斜率，初始取 600，后续用全体用户数据回归

5. 已知词量：

```
known = Σ_{rank=1..2809} P(known | rank)
```

6. 覆盖率：把 `P(known)` 与每个词的 `coverage` 相乘后累加。
7. 置信区间：对 40 题做 1000 次 bootstrap，取 2.5% 与 97.5% 分位。

**冷启动兜底**：用户跳过测试时，按目标与自评水平给初始 `θ`，之后用答题结果持续校正。

### 6.2 选词

**候选池**：

- 该用户 `user_word_state` 中不存在的 NGSL 词
- 目标领域差集词（NAWL 或领域词表，且不在已知集合）
- 排除 `dispersion < 0.3` 且与用户目标无关的词

**评分函数**：

```
score(w) = log(1 + U_w)
         × D_w^0.5
         × goalWeight(w)
         × (1 - mastery(w))
         × freshness(w)
         × typeBalance(w)
```

| 因子 | 计算 |
|---|---|
| `U_w` | 每百万频次，做对数压缩，避免 the/be 类词垄断 |
| `D_w` | 分布均匀度，开方后使用，惩罚单一领域高频词 |
| `goalWeight` | 目标相关词 ×3，通用词 ×1，无关领域词 ×0.5 |
| `mastery` | 四维掌握度加权平均：再认 0.4、回忆 0.3、拼写 0.2、搭配 0.1 |
| `freshness` | `exp(-近 7 天曝光次数 / 3)`，防止反复推同一批词 |
| `typeBalance` | 同一词性、同一语义簇超过 2 个时降权 |

**硬约束**：

- 每日新词 6–10 个，按 `dailyMinutes` 调整
- 同一语义簇最多 2 个
- 新词难度只取用户当前水平上一层（i+1）
- 复习词优先于新词，复习未完成时新词最多 5 个

**输出**：按分数降序取 N 个，同时返回每个词的入选理由，供智能体解释。

### 6.3 语境生成与校验

**输入**：目标词集合、用户兴趣、已知词集合、难度、篇幅、模态。

**生成约束**（写入提示词）：

- 目标词 6–10 个，每个至少出现 2 次
- 已知词覆盖率 ≥ 95%（阅读）/ 可懂度 ≥ 75%（听力）
- 平均句长 ≤ 22 词
- 不出现未列入目标词的低频词
- 文体由兴趣决定：科幻、职场、旅行、体育等

**校验器**（`ComprehensibilityVerifier`）：

```
1. 分词 → 词形还原 → 映射到 word_id
2. 过滤专有名词、数字、标点
3. knownRatio = 已知词数 / 实词总数
4. 校验每个目标词出现次数 ≥ 2
5. 校验平均句长、生词密度、禁用词
6. 任一项不通过 → 返回具体原因，重试
```

**重试策略**：

- 最多重试 3 次
- 第 2 次起把失败原因追加进提示词
- 3 次仍失败 → 降级为模板语境（从预置语料里选最接近的），并标记 `source=TEMPLATE`
- 所有生成结果存 `context_doc`，通过校验才置 `VERIFIED`

### 6.4 复习调度

**理论基座**：DSR 三变量模型。每个"词 × 维度"是一个原子记忆对象，独立维护 `S`、`D`、`R`。

#### 6.4.1 状态定义

| 变量 | 含义 | 单位 |
|---|---|---|
| `S` | 记忆稳定性，回忆概率从 100% 降到 90% 所需时间 | 天 |
| `D` | 记忆难度，0–1，越大越难 | 无量纲 |
| `R` | 可提取性，当前回忆概率 | 0–1 |

```
R(t) = exp(-t / S)
```

`t` 为距上次成功复习的天数。`R` 不落库，按需计算；`S`、`D` 落库。

#### 6.4.2 原子化

资料结论：复合记忆的可提取性是子记忆之积，稳定性会被拉到较难的子记忆水平。因此一个单词拆成四张原子卡：

| 原子卡 | 题型 | 初始 S | 目标 R | S 上限（天） |
|---|---|---|---|---|
| 再认 | 见词选义 | 1.0 | 0.90 | 180 |
| 回忆 | 中文想词 | 0.8 | 0.88 | 180 |
| 拼写 | 听音拼写 | 0.4 | 0.85 | 90 |
| 搭配 | 语境填空 | 0.6 | 0.85 | 120 |

同一单词的四种卡不同时出现，避免又把复合记忆拼回去。

#### 6.4.3 状态转移

P0 采用 DSR 的工程化简化式，参数先验来自 SM-2 / SM-4，上线后用真实日志拟合：

成功（`result = 1`）：

```
S' = S × (1 + a × (1 - R) × (11 - D) / 10 × S^(-b))
```

失败（`result = 0`）：

```
S' = max(0.5, c × S × D^(-d))
D' = min(1, D + e)
```

先验参数：

| 参数 | 初值 | 含义 |
|---|---|---|
| `a` | 1.2 | 成功增益系数 |
| `b` | 0.2 | 稳定性的边际递减 |
| `c` | 0.35 | 失败后的稳定性保留比例 |
| `d` | 0.5 | 难度对失败的放大 |
| `e` | 0.05 | 每次失败增加的难度 |

性质校验（必须由单测覆盖）：

- `R → 0` 时 `S'` 更大，符合间隔效应
- `R → 1` 时 `S' → S`，不出现"刚复习就暴涨"
- `D → 1` 时 `S'` 更小，难词增益低
- 失败后 `S' < S`，且不低于 0.5 天

#### 6.4.4 间隔计算

```
interval = -S' × ln(targetR)
next_due = now + clamp(interval, 1, S上限) 天
```

例：再认卡 `S' = 6.2`，`targetR = 0.90`，间隔为 `-6.2 × ln(0.9) ≈ 0.65` 天，向上取 1 天；`S' = 60` 时约 6.3 天。

#### 6.4.5 每日队列

排序依据是"单位时间收益"，而不是逾期天数：

```
benefit = (1 - R_now) × gapWeight × goalWeight
cost    = R_now × 3s + (1 - R_now) × 9s     // 资料中的成功/失败成本假设
priority = benefit / cost
```

- 队列上限由 `dailyMinutes` 换算，默认 20–30 张卡
- 同一单词最多出现两张不同维度的卡
- 连续成功且 `S > 30` 的卡降级为抽查，不再进入每日队列

#### 6.4.6 三阶段演进

| 阶段 | 方案 | 说明 |
|---|---|---|
| P0 | DSR-lite | 上述公式 + 先验参数，跑通闭环 |
| P1 | 参数拟合 | 用答题日志按用户分层拟合 `a/b/c/d/e`，冷启动用群体参数 |
| P2 | SSP-MMC 策略表 | 离线值迭代求 `(S, D) → 最优间隔`，线上查表；目标稳定性 360 天 |

P2 的做法：把 `S` 按 `log` 分箱离散化，`D` 分 10 档，动作空间取 20 个候选间隔，用贝尔曼方程迭代至策略收敛；每次成本按成功 3 秒、失败 9 秒计。策略表随参数版本发布，不要求线上实时求解。

### 6.5 漏洞归因

错误类型枚举：

| 类型 | 判定方式 | 后续题型 |
|---|---|---|
| `MEANING` | 释义选错，且非形近干扰 | 语境再认 |
| `SPELLING` | 拼写编辑距离 ≤ 2 | 听音拼写 |
| `COLLOCATION` | 搭配填空错 | 短语填空、改错 |
| `SYNTAX` | 句子结构理解错 | 长句拆解 |
| `LOOKALIKE` | 与形近词混淆 | 最小对立对辨析 |
| `SPECIAL_SENSE` | 熟词僻义 | 多义项对照 |

归因后写入 `attempt.error_type`，同时提高对应维度的复习权重。

---

## 7. 智能体设计

### 7.1 职责边界

智能体**只做编排与解释**，不做以下事情：

- 不直接改数据库，所有写操作走应用服务
- 不自由生成词表，候选词必须来自 `SelectionService`
- 不跳过校验，生成的语境必须过 `ComprehensibilityVerifier`

### 7.2 工具清单

| 工具 | 入参 | 出参 | 用途 |
|---|---|---|---|
| `getLearnerProfile` | userId | 目标、兴趣、已知词量 | 编排上下文 |
| `estimateKnownWords` | userId | 估计值与区间 | 冷启动 |
| `queryCandidateWords` | userId, limit | 候选词及评分 | 选词 |
| `getWordDetail` | lemma | 释义、词形、搭配 | 生成语境 |
| `generateContext` | 目标词、主题、难度 | 文本草稿 | 生成 |
| `verifyContext` | 文本、目标词 | 覆盖率、问题列表 | 校验 |
| `buildDrillItems` | 词、漏洞类型、数量 | 题目列表 | 出题 |
| `scheduleReviews` | userId, 日期 | 复习队列 | 调度 |
| `composeReport` | userId, 日期 | 指标与解读 | 报告 |

工具用 Spring AI 的 `@Tool` 注解声明，参数用 record + `@JsonPropertyDescription`。

### 7.3 编排流程

```mermaid
flowchart TD
    A[读取画像] --> B[计算候选词]
    B --> C[生成语境草稿]
    C --> D{可理解度校验}
    D -- 通过 --> E[落库为 VERIFIED]
    D -- 不通过 --> C
    E --> F[生成题目]
    F --> G[排复习队列]
    G --> H[写入今日计划]
```

每一步都写 `agent_trace`，包括 `rationale`。

### 7.4 提示词与结构化输出

系统提示词骨架：

```
你是英语词汇学习编排器。
约束：
1. 只能使用给定目标词，不得引入其他低频词。
2. 目标词每个至少出现 2 次，间隔不少于 1 句。
3. 句子平均长度不超过 22 词。
4. 不使用超出 B2 的语法结构。
5. 输出必须是 JSON，不要解释。
```

用户消息包含：目标词及释义、用户兴趣、难度、篇幅、上一次失败原因（重试时）。

输出 Schema：

```json
{
  "title": "string",
  "body": "string",
  "targetWords": ["string"],
  "sentences": 12,
  "estimatedDifficulty": "B1"
}
```

用 Spring AI 的 `BeanOutputConverter` 直接反序列化为 record，反序列化失败按生成失败处理。

### 7.5 可解释性

每个学习计划返回 `rationale` 列表，例如：

```
今天选了 8 个词：
- 6 个来自你目标文档中的高频未知词（合计覆盖 3.2%）
- 2 个是你上周答错 2 次以上的词
- 语境已知词占比 95.4%，符合可理解输入阈值
- 3 个拼写薄弱词被安排在明早复习
```

前端在"今日学习"页用可展开的决策条展示。

### 7.6 降级与成本

- **缓存**：以 `hash(目标词集合 + 主题 + 难度)` 为键缓存语境，命中率目标 ≥ 40%
- **批量预生成**：每天凌晨为活跃用户预生成次日语境
- **超时与熔断**：LLM 超时 20s，重试 2 次，Resilience4j 熔断
- **降级链**：LLM → 模板语境 → 预置语料（冲刺模式直接走真题句，不生成）
- **不阻塞**：生成失败先返回模板文，后台补生成；前端不出现等待动画
- **成本控制**：单用户单日 token 上限，超限后只走模板

---

## 8. 接口设计

### 8.1 约定

- 基础路径：`/api/v1`
- 鉴权：`Authorization: Bearer <jwt>`
- 统一返回：

```json
{
  "code": 0,
  "message": "ok",
  "data": {},
  "traceId": "b1f2c3"
}
```

- 幂等：写接口接受 `clientId`，重复提交返回首次结果
- 分页：`page`、`size`，默认 20

### 8.2 接口清单

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/auth/register` | 注册 |
| POST | `/auth/login` | 登录 |
| GET | `/me/profile` | 学习画像 |
| PUT | `/me/profile` | 更新目标、兴趣、每日时长 |
| POST | `/placement/start` | 开始测词 |
| POST | `/placement/answer` | 提交单题 |
| GET | `/placement/result` | 测词结果与覆盖曲线 |
| GET | `/session/today` | 今日计划（含 rationale） |
| POST | `/session/{id}/start` | 开始会话 |
| GET | `/contexts/{id}` | 获取语境文本 |
| POST | `/contexts/{id}/read` | 标记已读，写入 exposure |
| POST | `/attempts` | 提交作答 |
| GET | `/reviews/today` | 今日复习队列 |
| POST | `/reviews/{wordId}` | 提交复习结果 |
| GET | `/reports/daily` | 日报 |
| GET | `/reports/overview` | 覆盖率与趋势 |
| GET | `/gaps/summary` | 六类漏洞矩阵与修复收益排序 |
| GET | `/contexts` | 语境库列表（主题、难度、篇幅筛选） |
| GET | `/agent/traces/{sessionId}` | 智能体决策轨迹 |
| GET | `/words/{lemma}` | 词条详情 |

### 8.3 示例

`GET /api/v1/session/today`

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "sessionId": 1024,
    "date": "2026-11-09",
    "estimatedMinutes": 21,
    "newWords": [
      {"lemma": "conduct", "sfiRank": 412, "reason": "目标文档高频未知词"},
      {"lemma": "approach", "sfiRank": 389, "reason": "上周答错 2 次"}
    ],
    "context": {
      "id": 8801,
      "theme": "scifi",
      "wordCount": 198,
      "knownRatio": 0.954,
      "targetCount": 8
    },
    "reviewCount": 18,
    "rationale": [
      "今日新词全部来自你的目标文档差集",
      "语境已知词占比 95.4%，高于 95% 阈值",
      "3 个拼写薄弱词已安排在明早复习"
    ]
  },
  "traceId": "b1f2c3"
}
```

`POST /api/v1/attempts`

```json
{
  "clientId": "6e2f0c1a-...",
  "wordId": 412,
  "itemType": "MEANING_CHOICE",
  "isCorrect": false,
  "errorType": "LOOKALIKE",
  "responseMs": 4300
}
```

---

## 9. 关键流程时序

### 9.1 每日会话

```mermaid
sequenceDiagram
    participant U as 用户
    participant A as API
    participant S as SessionService
    participant G as StudyAgent
    participant L as LLM
    U->>A: GET /session/today
    A->>S: getOrCreate(userId, today)
    S->>G: buildPlan(userId)
    G->>G: queryCandidateWords()
    G->>L: generateContext(目标词)
    L-->>G: 文本草稿
    G->>G: verifyContext()
    G-->>S: plan + rationale
    S-->>A: 计划
    A-->>U: 今日学习
```

### 9.2 语境生成与校验

```mermaid
sequenceDiagram
    participant G as Orchestrator
    participant L as LLM
    participant V as Verifier
    G->>L: 生成请求（含约束）
    L-->>G: JSON 草稿
    G->>V: 校验覆盖率与目标词次数
    alt 通过
        V-->>G: VERIFIED
    else 不通过
        V-->>G: 失败原因
        G->>L: 重试（附失败原因）
    end
```

---

## 10. 前端设计

### 10.1 本次改了什么

v3.1 修的是两个结构性问题：一是界面只有特色流程，普通用户想"背单词"没有入口；二是单列排布不像软件，视觉偏暖偏纸感。

| # | 问题 | v3.0 | v3.1 |
|---|---|---|---|
| 1 | 只有阅读主线 | 今天页只有"开始阅读" | 今天页顶部常驻"背单词"条：已学 32/40 + 继续背词 |
| 2 | 缺少行业通用流程 | 无选词书、无每日新词、无背词会话 | 新增完整基础模式：选词书 → 每日新词 → 词卡 → 三档评分 → 结算 |
| 3 | 页面不像软件 | 单列长滚动页面 | 三栏应用外壳：左侧导航 228px + 顶栏 60px + 工作区 + 右栏 300px |
| 4 | 阅读是弹窗卡片 | 阅读在页面内展开 | 阅读与背单词都是全屏会话层，自带顶栏与进度 |
| 5 | 视觉偏纸质 | 米色底 + 衬线标题 + 深松绿 | 冷调浅色科技风：冷灰蓝底 + 电光蓝 + 无衬线 + 等宽微标签 |
| 6 | 今日计划不含背词 | 读一篇 / 理解 3 题 / 快认 8 词 | 背单词 / 读一篇 / 复习 12 个，共 12 分钟 |
| 7 | 无法调整学习量 | 设置里只有每天时长 | 词书与计划面板：4 本词书、每天 20/40/60、复习强度三档 |
| 8 | 两个模式指标混用 | 只有读懂率 | 背单词只报再认率；读懂率只在阅读侧结算，互不污染 |
| 9 | 术语仍偏内部 | "目标词""已知词覆盖率" | 界面统一为"生词""能读懂多少" |
| 10 | 移动端只有底部四项 | 四项 | 四项不变，背词与阅读作为会话层从今天页进入 |

基础模式与特色模式的分账见 2.5：**基本盘决定能不能用，特色层决定为什么留下。** 本章的界面必须让两者都成立，且基础模式可以完全不碰阅读独立闭环。

---

### 10.2 设计目标与认知预算

#### 10.2.1 四条目标

| 目标 | 指标 |
|---|---|
| 不用思考 | 打开到开始背词 ≤ 1 次点击；到开始阅读 ≤ 2 次；首屏可点元素 ≤ 6 |
| 不用学习 | 首次使用 ≤ 2 屏说明；无教程、无引导蒙层 |
| 不累眼睛 | 连续阅读 20 分钟无干涩感；行宽 34 字、行高 1.9、支持夜间 |
| 两条路都通 | 只用背单词（选词书 → 背 → 复习）也能闭环，不强制阅读 |

#### 10.2.2 认知预算（硬约束）

| 项 | 上限 |
|---|---|
| 每页主按钮 | 1 个 |
| 同一屏可见决策点 | ≤ 3 个 |
| 文字层级 | 3 级（标题 / 正文 / 说明） |
| 每天页面跳转 | ≤ 2 次 |
| 单次会话新概念 | ≤ 2 个（例如"读懂率""目标词"） |
| 术语出现 | 0 个（见 8.1 翻译表） |

超过预算的设计一律回退，不做"为了完整而加"。

---

### 10.3 信息架构

#### 10.3.1 一级入口（5 个）

```
今天  ── 今日任务：背单词条 + 今天的阅读 + 今日计划
词库  ── 搜索、词书、生词本、已掌握（基本盘）
复习  ── 单卡流，随时可刷（基本盘）
进步  ── 读懂率曲线、词量、待补的 10 个词
我的  ── 词书与计划、时长、模式、提醒、导出
```

会话层（不占导航，从今天页进入）：

```
背单词 ── 新词卡片流，40 个一组，可中断
阅读   ── 沉浸式文章 + 内联理解题与快认
```

规则：

- 今天页第一屏从上到下固定三块：**背单词条 → 今天的阅读 → 今日计划**
- 背单词条常驻，即使今天背完了也保留（文案改为"今天的词背完了"）
- 今天页两个主操作分主次：背单词条是实心按钮，阅读是同一视觉层的第二按钮，不做并列双主按钮
- 复习是独立入口，今日计划第三步保留"复习 12 个"
- 背单词与阅读都是全屏会话，不新增 tab、不改变导航项数量
- **基础模式不依赖特色层**：断网、拒绝生成内容、只想背词，都不影响背单词闭环

#### 10.3.2 路由

| 路由 | 页面 | 说明 |
|---|---|---|
| `/onboarding` | 首次说明 | 1 屏，可跳过 |
| `/placement` | 测词 | 15 题，可分三次完成 |
| `/today` | 今天 | 默认首页，含背单词条 |
| `/learn` | 背单词 | 全屏会话，40 个一组 |
| `/read/:textId` | 阅读器 | 沉浸式，含内联题 |
| `/library` | 词库 | 搜索 + 3 tab |
| `/library/book/:id` | 词书详情 | 进度、筛选、批量 |
| `/library/plan` | 词书与计划 | 面板（桌面）/ 全屏页（移动） |
| `/review` | 复习 | 单卡流 |
| `/review/quick` | 低能量 | 10 张卡 |
| `/progress` | 进步 | 从"我的"进入 |
| `/me` | 我的 | 设置 |
| `/done` | 今日完成 | 仅在主动查看时进入 |

#### 10.3.3 导航

- 桌面 ≥ 1180px：三栏应用外壳
  - 左栏 228px：品牌 + 5 个一级入口 + 账户 + 词书进度卡 + 冲刺天数
  - 顶栏 60px：全局搜索（⌘K）、连续天数、明暗切换、头像
  - 工作区：独立滚动，内容宽 720px 居中
  - 右栏 300px：待复习、本周读懂率、今日提示、快捷键；词库/复习/阅读页隐藏
- 平板 768–1180px：隐藏右栏，保留左栏
- 移动 < 768px：隐藏左栏与搜索，改 56px 底栏 4 项（今天 / 词库 / 复习 / 我的），进步从"我的"进入
- 会话层（背单词、阅读、测词）：全屏，仅保留左上返回与 2px 顶部进度线

---

### 10.4 核心流程

#### 10.4.1 首次使用（≤ 2 屏）

```
第 1 屏：三句话 + 目标三选一 + 词书四选一（默认六级）+ [开始]
第 2 屏：测词（右上角"以后再说"）
        └ 结果一句话 + 两个出口：[开始背 40 个] / [读第一篇]
```

说明页只讲四句，每句一行：

> 每天 40 个词，一组。
> 读一篇几乎都认识的文章。
> 只复习快忘的。
> 不想读的时候，只背词也能用。

不解释算法，不介绍功能，不出现"智能体"。

#### 10.4.2 日常（20 分钟，页面跳转 0 次）

```
打开 → 今天页
  [继续背词] → 背单词会话
      单词 → 显示释义 → 不认识 / 模糊 / 认识 → 下一张
      40 / 40 → 结算：再背一组 / 接着读一篇
  [开始阅读] → 阅读器
      点词 → 词卡（浮层）
      读完 → 内联 3 道理解题（可跳过）
           → 内联 8 词快认（每个 2 秒）
               → 完成条：读懂率 +0.3% · 少查 9 次 [回今天]
  → 今天页：今日计划三步分别打勾
```

两条链路各自只有一个入口、零中途跳转；从背词结算页可直接"接着读一篇"，这是基础模式通往特色模式的唯一主动线。

#### 10.4.3 低能量（3 分钟）

今天页底部一行文字链：`只有 3 分钟？复习 10 个词 →`

不弹窗、不打断、不计入今日完成，但计入连续天数。

#### 10.4.4 中断恢复

- 阅读进度按段保存，回到文章顶部轻提示"上次读到第 4 段"，可一键跳回
- 内联题未答完不保存，重做当前题
- 复习进度按卡保存，下次从下一张继续

#### 10.4.5 冲刺（四六级 30 天）

今天页顶部显示只读标签"冲刺 · 第 7/30 天"，点击查看说明。切换入口在"我的"，一个周期内不可来回切。

---

### 10.5 页面详设

#### 10.5.1 今天（`/today`）

```
┌────────┬──────────────────────────────────────────┬──────────────┐
│ 词境    │ [搜索单词、中文释义        ⌘K]  12 天 ☾ z │              │
├────────┼──────────────────────────────────────────┼──────────────┤
│ 学习    │ 第 12 天 · 冲刺第 7 / 30 天               │ 待复习        │
│ 今天    │ ┌──────────────────────────────────────┐ │ 12 个 · 4 分钟│
│ 词库    │ │ 背单词  CET-6 核心词      [继续背词]  │ │ [开始复习]    │
│ 复习 12 │ │ 今天已学 32 / 40 · 还剩 8 个，约 2 分钟│ │              │
│ 进步    │ └──────────────────────────────────────┘ │ 本周读懂率    │
│ 我的    │                                          │ 82.4%        │
│         │  Why Cities Sound Quieter at Night        │ 本周 +0.6     │
│ ┌─────┐ │  210 词 · 约 6 分钟 · 你大概能读懂 95%    │              │
│ │词书  │ │  [开始阅读 →]   读完还能补 8 个词         │ 今日提示      │
│ │56%  │ │ ─────────────────────────────────────    │              │
│ └─────┘ │  今日计划 · 约 12 分钟 · 3 步             │              │
│ 冲刺 7/30│  ① 背单词 8 个  ② 读一篇  ③ 复习 12 个    │              │
└────────┴──────────────────────────────────────────┴──────────────┘
```

- **背单词条在首屏第一条，高于文章标题**：保证"打开就知道今天要背什么"
- 背单词条结构：词书图标 + 词书名 + 已学 / 总数 + 迷你进度条 + 实心按钮，高 68px
- 标题 40px / 字重 650 / 字距 -0.032em，最多两行
- 信息行 12.5px `--ink-3`，用圆点分隔，不换行
- 今日计划固定三步且可点：完成后打勾并移除"现在"标记，**不整页替换**
- 不出现"AI""智能体""覆盖率"等内部词
- 今日已完成：背单词条文案变"今天的词背完了"，阅读块变"读懂率 82.4% → 82.7%"
- 新用户未测词：背单词条主按钮为 [先测词，再开始]

#### 10.5.2 阅读器（`/read/:textId`）

```
┌──────────────────────────────────────────────┐
│ ×                          第 1 段 / 6   Aa   │
│                                              │
│        为什么城市在夜里更安静                  │
│                                              │
│   城市的夜晚并不是真的安静。它只是把白天的      │
│   噪音换成了另一种频率。空调外机的低鸣、远处    │
│   货车的胎噪、楼下便利店冰柜的压缩机，这些声    │
│   音单独听都不刺耳，叠在一起却形成了一层持续    │
│   的底噪。                                    │
│                                              │
│   研究者把这种现象称为「城市背景声」。它不像    │
│   鸣笛那样突然，因此很少被注意到，但长期暴露    │
│   会让人的注意力更难集中。                     │
│                                              │
├──────────────────────────────────────────────┤
│  [ 听全文 ]                         [ 读完了 ] │
└──────────────────────────────────────────────┘
```

排版：

| 项 | 桌面 | 移动 |
|---|---|---|
| 正文字号 | 20px | 18px |
| 行高 | 1.9 | 1.9 |
| 行宽 | 34 汉字 / 68 西文 | 满宽减 40px |
| 段间距 | 1.5em | 1.5em |
| 正文色 | `--ink` #24211D | 同 |
| 目标词 | 1px 点状下划线，accent 60% | 同 |
| 已查词 | 下划线变 `--ink-3`，不再强调 | 同 |

`Aa` 面板（从底部滑出，不遮正文）：

- 字号：小 / 标准 / 大（17 / 20 / 23）
- 行距：标准 / 宽松（1.9 / 2.1）
- 主题：浅色 / 纯白 / 夜间
- 三项设置即时生效，本地保存，不需要确认

交互：

- 点目标词 → 词卡；长按任意词 300ms → 查词
- 选中文本只给"复制"，不给翻译
- 顶部 2px 进度线随滚动推进，不显示百分比
- 关闭时若有进度：直接保存并退出，不弹确认，回今天页显示"继续阅读"

#### 10.5.3 词卡

桌面为右侧浮层 360px，移动为底部抽屉 60vh（比 v2.0 的 85vh 更轻）。

```
┌────────────────────────────┐
│ ambient              ×     │
│ /ˈæmbiənt/          🔊     │
│                            │
│ 周围的、环境中的            │
│                            │
│ 本文中：城市背景声          │
│                            │
│ [ + 加入生词本 ]            │
└────────────────────────────┘
```

- 只保留一个操作。"加入生词本"是开关，点了变"已加入"，可再点取消
- 不再问"记住了 / 还不熟"——用户读一篇文章时没有能力判断自己记住了没有
- 词形、出现次数收进"更多"，默认不展示
- 同一次阅读里重复点同一个词，不重复弹动画

#### 10.5.4 内联理解题

读完点 [读完了]，正文下方就地展开，不跳页。

```
─────────────────────────────
第 2 题 / 共 3 题

作者认为城市夜晚"安静"的原因是：
  ○ 夜晚车流减少
  ○ 噪音变成了持续的背景声
  ○ 人们习惯了噪音
  ○ 建筑隔音变好

              [ 下一题 ]   跳过
─────────────────────────────
```

- 一题一屏，选完自动展开"下一题"按钮，不自动跳
- 答错只给一句解释，不出现"错误""不正确"这类判词
- 3 题结束自动过渡到快认，不加"继续"按钮

#### 10.5.5 内联快认

```
┌─────────────────────────────┐
│        ambient              │
│                             │
│   [ 不认识 ]     [ 认识 ]    │
└─────────────────────────────┘
```

- 8 个词，一屏一个，2 秒节奏由用户控制，不自动跳
- 键盘 `1` / `2`
- 不显示对错、不显示剩余数量，只在顶部用 8 个小点表示位置

#### 10.5.6 完成条（不跳页）

```
─────────────────────────────
今天读完了。

读懂率 82.4% → 82.7%
这篇你少查了 9 次词典。

[ 回今天 ]        [ 再读一篇 ]
─────────────────────────────
```

- 两个按钮，主按钮"回今天"，次按钮"再读一篇"
- 不出现金币、连续天数、分享

#### 10.5.7 复习（`/review`）

```
┌──────────────────────────────────────────┐
│ ×                          还剩 11        │
│                                          │
│        threshold                         │
│        /ˈθreʃhəʊld/            🔊         │
│                                          │
│        第 3 次见面 · 距上次 6 天           │
│                                          │
│        [ 看答案 ]                         │
│                                          │
├──────────────────────────────────────────┤
│   忘了          有印象          记得       │
└──────────────────────────────────────────┘
```

- 点"看答案"后展开释义 + 一句例句，三档才可用
- 三档等宽，52px 高；文案口语化，不用百分比、不用"困难/良好/简单"
- 三档顺序固定从左到右由弱到强，不做随机化
- 评分后 3 秒内可撤销，右下角浮出"已记录 · 撤销"
- 队列清空：一句"今天的复习清完了" + [再练 10 个] + [回今天]
- 不显示 S、R、总进度环

#### 10.5.8 低能量（`/review/quick`）

- 10 张卡，交互与 5.7 完全一致
- 顶部不显示"低能量模式"字样，只显示"还剩 10"
- 结束一句："3 分钟，守住了 10 个词。"

#### 10.5.9 词库（`/library`）

```
┌──────────────────────────────────────────────┐
│  词库                        [ 导入词表 ]     │
│  [ 搜索单词或中文… ]                          │
│                                              │
│  全部 2,140   生词本 46   已掌握 620          │
│  [ 在学 ] [ 四六级 ] [ 技术 ]   ← 筛选 chip   │
│                                              │
│  CET-6 核心词                 1240 / 2200     │
│  ▓▓▓▓▓▓▓▓░░░░░░                [ 换词书 ]     │
│                                              │
│  ambient    /ˈæmbiənt/   周围的      3 天后   │
│  threshold  /ˈθreʃhəʊld/ 门槛        明天     │
└──────────────────────────────────────────────┘
```

- 三个 tab，默认"全部"；"在学"降级为筛选 chip
- 列表行固定四项：词、音标、释义、下次复习（人话）
- 移动端左滑、桌面 hover 出现操作：加入今天 / 标记已掌握 / 移出生词本
- 右下角"选择"进入批量模式，避免每行都挂复选框
- 空生词本："阅读时加入的词会出现在这里"
- 词书进度只写词数，不写"你还差 960 个"

#### 10.5.10 进步（`/progress`）

```
┌──────────────────────────────────────────┐
│  进步                                    │
│                                          │
│  你能读懂一般文章的                       │
│  82.4%                                   │
│  ↑ 本周 +0.6                             │
│  ▁▂▂▃▃▄▅▅▆                                │
│                                          │
│  按现在的速度，四周后大约 85%。           │
│                                          │
│  ▸ 哪些能力在变化                        │
│  ▸ 待补的 10 个词                        │
│  ▸ 这周学了什么                          │
└──────────────────────────────────────────┘
```

- 一个主指标、一条趋势线、一句话预测
- 三块默认折叠；四维用横向条 + "稳 / 在进步 / 需加强"，不显示百分数
- 底部一行小字说明口径，点开才解释

#### 10.5.11 我的（`/me`）

分组列表，每组 ≤ 5 项，行高 56px：

- 学习：目标、每天时长（10 / 20 / 30 分钟）、模式（日常 / 冲刺）
- 阅读：字号与主题、朗读音色、语速
- 提醒：开关 + 时间
- 数据：导出 CSV、清除记录
- 关于：版本、隐私

危险操作红字，二次确认写清后果。

#### 10.5.12 冲刺模式

| 项 | 日常 | 冲刺 |
|---|---|---|
| 每日新词 | 8 | 40 |
| 语境 | 生成短文 | 真题句 + 词表例句，不生成 |
| 题型 | 再认、回忆、搭配 | 再认、拼写 |
| 主指标 | 读懂率 | 词表覆盖率 |
| 时长 | 20 分钟 | 25–30 分钟，可拆两段 |

页面只差三处：今天页标签、阅读器换成真题句卡、结算主指标换成词表覆盖率。其余完全一致，避免两套界面。

#### 10.5.13 背单词会话（`/learn`）

```
┌──────────────────────────────────────────────┐
│ ‹        背单词  CET-6 核心词 · 每天 40 个  32/40│
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░    │
│                                              │
│        ┌────────────────────────────┐        │
│        │         threshold          │        │
│        │      /ˈθreʃhəʊld/  🔊       │        │
│        │        [ 显示释义 ]         │        │
│        └────────────────────────────┘        │
│        （点卡片或按空格翻面）                  │
│                                              │
│   ┌────────┬────────┬────────┐               │
│   │ 不认识  │  模糊   │  认识   │               │
│   │明天再见 │ 2 天后  │ 6 天后  │               │
│   └────────┴────────┴────────┘               │
│   今天第 33 个               空格显示释义       │
└──────────────────────────────────────────────┘
```

翻面后卡片内追加：词性 + 释义（20px / 600）、英文例句（16px / 1.75）、中文翻译（12.5px `--ink-3`）、搭配或词根提示（浅底 8px 圆角）、一行紫色小字"这个词会在这周的 3 篇文章里再出现"。

规则：

- 三档评分的副标题直接写"明天 / 2 天后 / 6 天后"，让用户对选择有预期
- 键盘：空格翻面，1 / 2 / 3 评分，Esc 退出
- 中途退出不丢进度，下次从下一张继续
- 结算页只在 40/40 出现，两个出口：再背一组 / 接着读一篇
- 发音按钮只播放，不写入任何记忆状态
- 不显示 S / R / D 等算法字段，不显示"预计记住率"这类无法验证的数字
- 顶部计数是"已完成 / 今日总量"，不是"当前第几个"

三档口径与 6.4 的调度对齐：

| 评分 | 写入维度 | 下次间隔 | 附加行为 |
|---|---|---|---|
| 不认识 | 再认 | 次日 | 同时回到当日队列尾部再出现一次 |
| 模糊 | 再认 | 2 天后 | 稳定性增幅按 DSR 计算 |
| 认识 | 再认 | 6 天后 | 不进入精读复现队列 |

#### 10.5.14 词书与计划（`/library/plan`）

```
┌────────────────────────────┐
│ 词书与计划              ✕  │
│ 换词书不会清空已学记录。     │
│                            │
│ 词书                        │
│ [四级核心词      3,600 词]  │
│ [六级核心词      2,200 词] ✓│
│ [考研大纲词      5,500 词]  │
│ [雅思学术词      4,000 词]  │
│                            │
│ 每天新词                    │
│ [20 个] [40 个] [60 个]     │
│                            │
│ 复习强度                    │
│ [轻松] [标准] [高强度]       │
│                            │
│ [          保存           ] │
└────────────────────────────┘
```

- 三处入口：侧栏词书卡"换词书"、词库页词书卡"换词书"、我的"词书"
- 每天新词只影响明天的计划，今天剩余量不变
- 复习强度只改每日复习上限（60 / 120 / 200 张），不改算法
- 保存后一句 toast，不跳页

---

### 10.6 视觉与舒适度

#### 10.6.1 色彩（浅色 · 冷调科技风）

```css
:root {
  --bg:           #F3F6FC;  /* 页面底色，冷灰蓝，不是米白 */
  --surface:      #FFFFFF;
  --surface-2:    #EDF2FA;  /* 输入框、次级容器 */
  --surface-3:    #F7F9FE;  /* 卡片内嵌区 */

  --ink:          #0C1526;  /* 正文，带一点蓝，不用纯黑 */
  --ink-2:        #44506A;
  --ink-3:        #6C7889;

  --line:         #E1E7F1;
  --line-2:       #C7D1E2;

  --accent:       #2563EB;  /* 电光蓝，全站唯一强调色 */
  --accent-hover: #1D4ED8;
  --accent-soft:  #E8F0FF;

  --warm:         #6D5BF6;  /* 连续天数、次级强调 */
  --warm-soft:    #EEEBFF;
  --danger:       #DC4C4C;
}
```

科技感只靠三处：冷灰蓝底 + 电光蓝单强调色 + 等宽微标签。不做霓虹描边、不做发光边框、不做玻璃拟态——那些在长时间阅读里会让人累。

#### 10.6.2 夜间

```css
[data-theme="night"] {
  --bg:           #0A1120;  /* 深海蓝底，不是暖褐 */
  --surface:      #111A2B;
  --surface-2:    #172133;
  --surface-3:    #0E1626;

  --ink:          #E7EEFA;
  --ink-2:        #A8B5CB;
  --ink-3:        #6F7E97;

  --line:         #1E2A41;
  --line-2:       #2B3A54;

  --accent:       #5B9BFF;
  --accent-hover: #7FB2FF;
  --accent-soft:  #16233C;

  --warm:         #A99BFF;
  --warm-soft:    #1D2340;
  --danger:       #F0736F;

  --sh-1:         0 1px 2px rgba(0,0,0,.35);
  --sh-2:         0 14px 36px rgba(0,0,0,.45);
}
```

主题三档：冷调浅色（默认）/ 纯白（强光环境，只抬底色与描边）/ 夜间，跟随系统 + 手动覆盖，偏好存 localStorage。

#### 10.6.3 字体与字号

```css
--font-ui:   "Inter", "PingFang SC", "Microsoft YaHei", system-ui, sans-serif;
--font-read: "Inter", "PingFang SC", "Microsoft YaHei", system-ui, sans-serif;
--font-mono: "JetBrains Mono", "SF Mono", ui-monospace, Consolas, monospace;
```

正文与界面共用同一无衬线栈，不再用衬线体。统一感来自字号、字重与字距分层，而不是换字体家族。等宽只用于微标签：栏目标题、模块说明、快捷键、指标单位，`10–11px / 600 / letter-spacing .14–.16em`，全大写或半角。

| 场景 | 字号 / 行高 / 字重 |
|---|---|
| 阅读正文 | 20 / 1.9 / 400（可调 17–23） |
| 页面大标题 | 40 / 1.12 / 650，`letter-spacing:-.032em` |
| 页面标题 | 28 / 1.35 / 600 |
| 卡片标题 | 20 / 1.45 / 600 |
| 界面文字 | 14 / 1.6 / 400 |
| 辅助说明 | 12.5 / 1.55 / 400 |
| 等宽微标签 | 10–11 / 1.4 / 600 |
| 主指标 | 52 / 1.05 / 600，tabular-nums |

限制：行宽 34 汉字；每行不出现 3 个以上连续目标词；标题不超过 18 字；大标题宽度上限 `16ch`，避免长句撑满工作区。

#### 10.6.4 间距、圆角与层次

- 间距 4 的倍数：8 / 12 / 16 / 24 / 32 / 48
- 圆角：卡片 16px，按钮 10px，输入框 10px，面板 / 抽屉 18px
- 外壳尺寸：顶栏 60px，左栏 228px，右栏 300px，工作区 ≥ 720px
- 层次靠背景差 + 1px 描边 + 极淡投影，不靠重阴影
- 阴影仅两档：`0 1px 2px rgba(16,32,64,.06)`、`0 14px 36px rgba(16,32,64,.10)`；夜间换纯黑
- 允许两处极淡径向光晕（右上 accent 9%、左下 warm 6%），只做氛围，不影响正文
- 不用玻璃拟态、不用霓虹描边、不用大面积渐变

#### 10.6.5 动效

- 时长：反馈 140ms，抽屉 220ms，页面 180ms
- 缓动：`cubic-bezier(.2,.7,.3,1)`
- 只做位移与淡入；不做弹跳、旋转、缩放超过 1.02
- `prefers-reduced-motion` 开启时全部改为淡入
- 阅读中禁止自动滚动、自动播放下一题

#### 10.6.6 阅读舒适度细则

- 正文对比度约 15:1（`#0C1526` on `#F3F6FC`），高于 WCAG AA
- 段落不首行缩进，用段间距区分
- 中文标点用全角，英文单词与中文之间留 1/4 空格
- 目标词不改变字重，只加下划线，避免打断阅读节奏
- 页脚留白 ≥ 96px，避免最后一行贴边

---

### 10.7 组件

| 组件 | 用途 | 状态 |
|---|---|---|
| `TopBar` | 顶栏：品牌、全局搜索、连续天数、主题、头像 | 默认 / 搜索聚焦 / 滚动收起 |
| `SideNav` | 左栏一级入口 5 项 + 底部词书进度 | 选中 / hover / 折叠（< 1180px） |
| `RightRail` | 右栏：今日进度、本次目标词、提示卡 | 有数据 / 空 / 面板打开 |
| `BottomTabs` | 移动导航（4 项） | 选中 / 未选中 |
| `TodayBar` | 今天页顶部背单词条：词书 + 剩余量 + 进度 + [继续背词] | 未开始 / 进行中 / 今日完成 |
| `LearnCard` | 全屏背单词卡：词形 → 释义（翻面） | 正面 / 已揭示 / 评分中 |
| `PlanSheet` | 词书与计划面板：4 本词书 + 每日新词 + 复习强度 | 默认 / 已改动 / 保存中 |
| `PrimaryButton` | 主操作 | 默认 / hover / 按下 / 禁用 / loading |
| `GhostButton` | 次操作 | 默认 / hover |
| `ReadingBody` | 正文 | 默认 / 目标词 / 已查 / 朗读高亮 |
| `WordSheet` | 词卡 | 加载 / 已查 / 已加入生词本 / 失败 |
| `InlineQuiz` | 内联理解题 | 未选 / 已选 / 正确 / 错误 / 跳过 |
| `RecallStrip` | 内联快认 | 未选 / 已选 |
| `ReviewCard` | 复习卡 | 正面 / 已揭示 / 已评分 |
| `MetricHero` | 主指标 | 有数据 / 无数据 |
| `Sparkline` | 趋势 | 单点 / 多点 / 空 |
| `ListRow` | 设置项 | 默认 / 危险 / 禁用 |
| `Toast` / `EmptyState` | 轻提示 / 空态 | — |

关键规格：

- `TopBar`：60px 高，搜索框 38px / 圆角 10px，聚焦时由 `--surface-2` 转 `--surface` 加 1px 描边
- `SideNav`：228px 固定宽；条目 38px 高、圆角 10px；选中态为 `--accent-soft` 底 + `--accent` 字 + 左侧 2px 竖条
- `RightRail`：300px；只放"今天"与"阅读中"两类信息，不做推荐流
- `TodayBar`：一条横幅装完"背什么 / 还剩几个 / 多久 / 继续"，主按钮 36px 高，进度条最宽 120px
- `LearnCard`：词 34px / 音标 15px 等宽；释义区最多 3 个义项；评分三键横排，每键带"下次见面"小字
- `PrimaryButton`：48px 高，圆角 10px，loading 时宽度不变
- `ReviewCard`：最大 520px 宽，最小 360px 高，词 32px，例句 18px / 1.75
- `WordSheet`：移动端 60vh，圆角 18px 顶部，遮罩 24%，锁滚动
- `InlineQuiz`：选项行 52px 高，整行可点，不出现单选圆点加框的双重视觉
- `Sparkline`：高 56px，线宽 2px，无坐标轴，点击显示数值气泡

---

### 10.8 交互与文案

#### 10.8.1 术语翻译表

| 系统概念 | 界面用语 |
|---|---|
| 覆盖率 82.4% | 你能读懂一般文章的 82.4% |
| 已知词 95.4% | 这篇你大概能读懂 |
| S = 6.2d | 距上次 6 天 |
| R = 0.83 | （不显示，必要时"快忘了"） |
| 四维状态 | 认识 / 会写 / 会用 |
| 漏洞 | 还不熟的地方 |
| 选词评分 | 为什么是这 8 个 |
| 队列长度 | 还剩 11 |
| 智能体 | 不出现 |

#### 10.8.2 交互规则

| # | 规则 |
|---|---|
| 1 | 点词不跳页，浮层出现，正文位置不变 |
| 2 | 每页只有一个主按钮 |
| 3 | 评分与"加入生词本"3 秒内可撤销 |
| 4 | 除删除外不用 modal |
| 5 | 超过 300ms 用骨架屏，不用遮罩转圈 |
| 6 | 错误就地提示，不弹全局错误 |
| 7 | 键盘：复习 1/2/3 与空格，快认 1/2，`Esc` 关闭浮层 |
| 8 | 关闭浮层后焦点回到触发词 |
| 9 | 触控目标 ≥ 44px |
| 10 | 阅读中不自动跳转、不自动播放 |

#### 10.8.3 语气

- 陈述事实，不评价用户
- 不制造焦虑：用"还剩 11"，不用"你已落后"
- 不用感叹号、不用"太棒了"
- 数字保留一位小数，百分比取整

---

### 10.9 状态与异常

| 状态 | 处理 |
|---|---|
| 首屏加载 | 骨架屏（标题 + 3 行灰条） |
| 语境生成失败 | 自动降级模板文，页脚小字"备用内容" |
| 词卡失败 | 卡内"再试一次"，保留词与音标 |
| 音频不可用 | 隐藏喇叭，不显示禁用态 |
| 离线 | 读缓存文章，复习可离线评分，联网补传 |
| 今日无新词 | "今天的词都学完了" + [再读一篇] |
| 复习为空 | "今天的复习清完了" + [再练 10 个] |
| 数据不足 | "读完 3 篇后出现第一条曲线" |
| 超时 10s | "换个网络试试" + 重试 / 返回 |

---

### 10.10 可访问性

- 正文对比度 ≥ 7:1，辅助文字 ≥ 4.5:1
- 焦点环 2px accent + 2px offset
- 图标按钮必须有 `aria-label`
- 题目反馈 `aria-live="polite"`
- 系统字号放大 200% 不破版（正文用 rem）
- 状态不只靠颜色：正确 / 错误同时给图标与文字
- 夜间模式不降低对比度

---

### 10.11 工程

#### 10.11.1 技术栈

Vue 3.4 + TypeScript + Vite + Pinia + Vue Router；原生 CSS 变量，不引入 UI 库；`@vueuse/core`；测试用 Vitest + Playwright。

#### 10.11.2 目录

```
src/
├─ api/
├─ components/
│  ├─ base/       Button, Tabs, Toast, EmptyState
│  ├─ reading/    ReadingBody, WordSheet, InlineQuiz, RecallStrip, TypePanel
│  ├─ review/     ReviewCard, RatingBar
│  └─ progress/   MetricHero, Sparkline, BarList
├─ composables/   useReadingProgress, useWordLookup, useKeyboard, useTheme
├─ pages/
├─ stores/        user, today, learn, reading, review, library, progress, ui
├─ styles/        tokens.css, themes.css, base.css, typography.css
└─ utils/
```

#### 10.11.3 Store

| Store | 持有 | 不持有 |
|---|---|---|
| `user` | 目标、时长、模式、字号主题、提醒 | 学习统计 |
| `today` | 今日摘要、三步状态、8 词、背词进度 | 正文 |
| `learn` | 本次会话队列、当前索引、已揭示、撤销栈 | 词库全量 |
| `reading` | 段落位置、已查词、内联题状态 | 词库 |
| `review` | 队列、索引、撤销栈 | 调度算法 |
| `library` | 搜索词、tab、筛选、分页、当前词书与计划 | 全量词库 |
| `progress` | 主指标、趋势、四维、待补词 | 原始日志 |
| `ui` | toast、抽屉、主题、全局 loading | 业务数据 |

服务端为唯一事实来源，store 只做会话缓存与乐观更新。

#### 10.11.4 接口

| 页面 | 接口 |
|---|---|
| 今天 | `GET /api/v1/today` |
| 背单词会话 | `GET /api/v1/learn/session`、`POST /api/v1/learn/grade` |
| 词书与计划 | `GET /api/v1/books`、`GET /api/v1/plan`、`PUT /api/v1/plan` |
| 阅读器 | `GET /api/v1/contexts/{id}` |
| 词卡 | `GET /api/v1/words/{lemma}` |
| 加入生词本 | `POST /api/v1/words/{lemma}/bookmark` |
| 内联题 | `POST /api/v1/contexts/{id}/quiz/submit` |
| 快认 | `POST /api/v1/recall/batch` |
| 复习队列 | `GET /api/v1/review/queue` |
| 评分 | `POST /api/v1/review/{cardId}/grade` |
| 词库 | `GET /api/v1/library/words` |
| 词书 | `GET /api/v1/library/books/{id}` |
| 进步 | `GET /api/v1/progress/overview` |
| 待补词 | `GET /api/v1/progress/gaps` |
| 测词 | `GET /api/v1/placement/next`、`POST /submit` |
| 设置 | `PATCH /api/v1/users/me` |

`/learn/grade` 只写"再认"维度，返回下次到期时间；`/plan` 改动后当日已生成的会话不重算，次日生效。

#### 10.11.5 性能

| 项 | 目标 |
|---|---|
| 首屏 LCP | < 1.5s（4G） |
| 路由切换 | < 300ms |
| 词卡 | < 400ms（命中缓存 < 50ms） |
| 首包 | < 200KB gzip |

#### 10.11.6 埋点（10 个）

`app_open`、`learn_start`、`learn_finish`、`read_start`、`read_finish`、`word_tap`、`quiz_skip`、`review_finish`、`quick_mode_enter`、`day_complete`

---

### 10.12 验收

#### 10.12.1 认知

- [ ] 首屏可点元素 ≤ 5，主按钮唯一
- [ ] 从打开到开始阅读 ≤ 2 次点击
- [ ] 从打开到开始背单词 ≤ 2 次点击
- [ ] 首次说明 ≤ 2 屏，无引导蒙层
- [ ] 全站不出现 S、R、覆盖率、维度、智能体

#### 10.12.2 流程

- [ ] 阅读 → 理解 → 快认 → 完成，页面跳转 0 次
- [ ] 背单词：今天页 → 翻面 → 评分 → 下一个，停留页 ≤ 1
- [ ] 关闭背单词或阅读器都不弹确认，回今天可继续
- [ ] 复习评分 3 秒内可撤销
- [ ] 词库搜索到查词 ≤ 2 步
- [ ] 不读文章也能走完"背词 → 复习 → 打卡"，不出现空态断点
- [ ] 背单词结算只报再认率，不出现读懂率

#### 10.12.3 舒适度

- [ ] 阅读器内 3 步完成字号、行距、主题调整
- [ ] 夜间模式对比度不低于浅色模式
- [ ] 连续阅读 20 分钟，滚动无卡顿、无自动跳转
- [ ] ≥ 1180px 三栏不出现横向滚动；768–1180px 右栏让位后主内容仍 ≥ 640px
- [ ] 背单词全屏时空格翻面、1/2/3 评分可用

#### 10.12.4 可用性测试（5 人）

1. 打开后直接开始阅读
2. 查一个词并回到原段落
3. 读完一篇文章直到看到"少查了 N 次"
4. 找到"我最近进步在哪"
5. 只想花 3 分钟时找到入口
6. 只想背单词、不想读文章时，找到并完成一组

通过标准：5 人全部 45 秒内完成 1、2、3、5、6；4 在 90 秒内完成并能说出主指标含义。

---

## 11. 非功能设计

| 维度 | 方案 |
|---|---|
| 性能 | 读接口 P95 ≤ 300ms；语境生成异步，前端轮询或 SSE |
| 缓存 | 词库 Caffeine 本地缓存；用户已知词用 RoaringBitmap 存 Redis |
| 并发 | Java 21 虚拟线程处理 LLM 调用；`@Async` 预生成 |
| 幂等 | `attempt.client_id` 唯一约束；重复提交返回首次结果 |
| 限流 | 登录、测词、生成接口按用户维度限流 |
| 安全 | BCrypt 存密码；JWT 短过期 + Refresh Token；接口级 RBAC |
| 内容安全 | 生成文本过敏感词与合规检查后再落库 |
| 可观测 | traceId 贯穿；关键步骤埋点；LLM 调用记录 token 与延迟 |
| 数据保留 | 作答明细保留 2 年，聚合指标长期保留 |
| 备份 | PostgreSQL 每日全量 + WAL 归档；MinIO 版本化 |

---

## 12. 部署方案

MVP 用 Docker Compose：

```yaml
services:
  app:
    image: wordscape/server:latest
    environment:
      SPRING_PROFILES_ACTIVE: prod
      LLM_BASE_URL: http://ollama:11434
    ports: ["8080:8080"]
    depends_on: [postgres, redis, minio]
  postgres:
    image: postgres:16
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
  ollama:
    image: ollama/ollama
```

生产环境建议：

- 应用无状态，水平扩 2 实例以上
- LLM 走独立服务，不与业务进程争内存
- 数据库连接池 HikariCP，最大连接数为 CPU 核数 × 2 + 磁盘数
- JVM 参数：`-XX:+UseZGC -Xms1g -Xmx2g`

---

## 13. 测试策略

| 层次 | 内容 | 工具 |
|---|---|---|
| 单元测试 | 选词评分、复习间隔、覆盖率计算、Logistic 拟合 | JUnit 5 |
| 集成测试 | Repository、Redis 位图、LLM 客户端打桩 | Testcontainers |
| 契约测试 | OpenAPI 与前端类型对齐 | springdoc + schema diff |
| 算法回归 | 固定用户行为序列，断言复习队列快照 | 黄金用例 |
| 压测 | 今日会话、提交作答 | k6 |
| 内容抽检 | 生成语境人工评分 | 内部标注表 |

关键黄金用例：

1. 已知 1800 词的用户，今日新词必须全部来自 NGSL 且 `sfiRank < 2809`
2. 语境 `knownRatio` 必须 ≥ 0.95，否则状态不得为 `VERIFIED`
3. 同一词连续成功复习后，稳定性必须单调递增；间隔由 `-S × ln(targetR)` 换算，不得直接写死天数
4. 重复提交同一 `clientId` 不得产生两条 attempt

---

## 14. 迭代计划

### P0：跑通教学闭环，基本盘可用

- 词库导入（NGSL + SUP + NAWL + CET-4/6 词表）
- 注册登录、学习画像
- 测词与已知词量估计
- 选词服务
- 模板语境 + 覆盖率校验（先证明 95% 约束成立，再谈生成质量）
- 词书与词表进度、生词本、任意查词（基本盘）
- 专项训练与作答记录
- DSR-lite 复习调度（见 6.4）
- 今日、阅读、词库、复习、我的五页
- 四六级冲刺模式（真题句，不依赖生成）

### P1：接入智能体与场景

- Spring AI 接入
- 语境生成、校验、重试、降级（替换 P0 模板）
- 智能体决策轨迹与可解释文案
- 读懂率口径校准与"这篇少查了几次词典"反馈
- 四六级真题语料库扩充
- 技术文档阅读场景
- 漏洞图谱与六类归因
- 学情报告完整指标
- 复习参数拟合（`a/b/c/d/e` 按用户分层）

### P2：规模化与多模态

- 听力材料与 TTS
- 多口音音频与最小对立对训练
- 领域词表扩展
- SSP-MMC 策略表（目标稳定性 360 天，见 6.4.6）
- 小程序端
- 学习计划分享与班级场景

---

## 15. 风险与取舍

| 风险 | 影响 | 应对 |
|---|---|---|
| LLM 生成不可控 | 语境超纲、生词过多 | 强校验 + 重试 + 模板降级 |
| 已知词估计偏差 | 选词过难或过易 | 用答题结果持续校正，允许用户手动标记 |
| 记忆参数冷启动偏差 | 复习间隔失准 | 群体先验 + 在线拟合 + 用户"太难/太易"反馈 |
| 中文释义与搭配缺失 | 体验断裂 | 离线批处理补齐，缺失时降级英文释义 |
| 生成成本高 | 毛利下降 | 缓存、批量预生成、按用户限流 |
| 内容合规 | 上线风险 | 敏感词过滤 + 人工抽检 |
| 过度承诺效果 | 口碑风险 | 用覆盖率而非"流利"作为对外指标 |

明确的取舍：

- 首版不做口语评测。发音问题涉及信号处理与模型，投入产出比低。
- 首版不做语法课程。语法通过长句拆解题间接覆盖。
- 首版不做社区。学习闭环未验证前，社交只会稀释注意力。

---

## 附录 A：NGSL 字段说明

| 字段 | 含义 |
|---|---|
| `Lemma` | 词元 |
| `Wordlist` | `1 - NGSL` / `2 - Sup` / `3 - NAWL`，空值为词表外 |
| `WL_SFI_Rank` | 在所属词表内的 SFI 排名 |
| `SFI` | Standard Frequency Index，标准频次指数 |
| `U` | 每百万词调整频次 |
| `D` | 分布均匀度，0–1，越接近 1 越均匀 |
| `F` | 原始频次 |
| `Coverage` | 该词覆盖的文本比例 |
| `Cumulative Coverage` | 累计覆盖率 |

## 附录 B：错误码

| 码 | 含义 |
|---|---|
| 0 | 成功 |
| 1001 | 参数错误 |
| 1002 | 未登录或 token 过期 |
| 2001 | 词条不存在 |
| 2002 | 今日会话已存在 |
| 3001 | 语境生成失败 |
| 3002 | 语境未通过可理解度校验 |
| 3003 | 智能体超时 |
| 4001 | 重复提交 |
| 5000 | 服务内部错误 |

## 附录 C：关键配置

```yaml
wordscape:
  selection:
    new-words-per-day: 8
    max-same-cluster: 2
    min-dispersion: 0.3
  context:
    known-ratio-threshold: 0.95
    listening-comprehension-threshold: 0.75
    max-retry: 3
    max-sentence-length: 22
  review:
    daily-cap: 30
    target-retention:
      recognition: 0.90
      recall: 0.88
      spelling: 0.85
      collocation: 0.85
    max-interval-days: 180
    params:
      a: 1.2
      b: 0.2
      c: 0.35
      d: 0.5
      e: 0.05
  agent:
    llm-timeout-ms: 20000
    daily-token-limit: 30000
    fallback-to-template: true
```

## 参考资料

1. NGSL 1.2（New General Service List）词表与 SFI 数据，`资料/NGSL/`
2. 漏屋.《告诉你外语学习的真实方法与误区分析》
3. 漏屋.《找对英语学习方法的第一本书》
4. 叶峻峣.《间隔重复记忆算法：e 天内，从入门到入土。》知乎专栏，2022-08-20
5. 叶峻峣, 苏敬勇, 曹译珑. *A Stochastic Shortest Path Algorithm for Optimizing Spaced Repetition Scheduling*. KDD 2022
6. 叶峻峣.《KDD'22 | 墨墨背单词：基于时序模型与最优控制的记忆算法》知乎专栏
7. maimemo/SSP-MMC 开源实现：https://github.com/maimemo/SSP-MMC
