# baguwen-rag — MianCoach「AI 模拟面试官」(RAG)

秋招简历项目 + 自用八股工具:**抓取小林coding 免费题库做权威语料,全链路自研 RAG——问答带溯源 + 模拟面试批改 + 复盘薄弱点**。
技术:Spring Boot 3 / **Java 17**(与 autumn-im、smart-canteen 对齐)+ Vue3(Vite) + PostgreSQL16/pgvector + DashScope(OpenAI 兼容协议)。语料抓取用 Python。

## 项目结构

```
docs/requirements.md  需求与接口文档
docs/run.md           启动手册(三步起)
compose.yml           pgvector 容器(baguwen-pg @15432,命名带前缀防撞车)
crawler/              Python:HTML→md(已完成:4板块13页574题语料)
corpus/               语料层:corpus/interview/<板块>/<页>.md
server/               Spring Boot 后端(jar 31MB 已可跑)
web/                  Vue3 前端(四页)
```

## 现状(2026-09-12,feat/production-ready 分支):全链路可用 + 检索管线升级

- **索引**:corpus → Java 分块(叶子级标题=题,兼容 h2/h3)→ **长答案切子段(parent-child,574 题 → 1155 子段)**
  → 向量(1024d)→ pgvector;`index_meta` 指纹带分块方案,换方案未重灌直接 409
  - `POST /api/admin/reindex` = **13 doc / 574 chunk / 1155 seg**
- **检索**:exact / vector(题目级+子段级同池) / keyword 多路召回 → 前 20 候选交 **bge-reranker-v2-m3(免费)统一重排**
  → top5;子段命中的重排证据用子段正文;重排挂掉自动降级回召回原序
- **拒答**:余弦与重排分双门槛(同时过低才拒),域外问题不硬答,`noAnswer` 事件 + 诚实话术
- **评测(去自环)**:`POST /api/admin/eval/build` 用 LLM 把题干改写成真实问法 + 细节探针 + 域外负样本;
  `GET /api/admin/eval?mode=golden` 报 recall@k / MRR / 延迟 / 拒答定标。数字见 **docs/eval-report.md**:
  探针(细节题)recall@1 0.615→**0.795**、recall@5 0.846→**0.949**
- **多轮追问**:`/api/chat/ask` 带 history,追问里的"它/那个"先改写成独立问题再检索
- **问答**:`POST /api/chat/ask`(SSE:refs→delta→done),子段命中片段前置进上下文,答案引用可点开原文
- **模拟面试**:`/api/quiz/start|pick|grade`,批改=LLM 对照标准原文逐点评分(hits/missing),落库
- **复盘**:场次记录 + 按板块/类目的薄弱聚合
- 前端四页(问答/知识库/模拟面试/复盘),deep-first 设计(代码绿 x 深夜自习,ui-ux-pro-max 定调),深浅双主题
- Swagger: http://localhost:8080/swagger-ui.html

运行方式见 `docs/run.md`。**默认 auto 模式(没 key 自动走 mock,免配置联调);面试演示请配 `BAGUWEN_CHAT_API_KEY` + `BAGUWEN_EMBED_API_KEY`。**

## 语料(corpus/)状态与约定

- 布局:corpus/interview/<板块group>/<页>.md;板块按**站侧边栏编号**分组(06架构设计=消息队列+分布式+系统设计)。
- 两种题级结构:java/database/mq/cap 页 h2=类目 h3=题;systemdesign 与 tools 页 h2 直接当题。切题规则=**叶子级标题即一道题**。
- **AI-Agent 面试篇在本站无正文**(5 板块全指向外站 xiaolinnote.com 导流页),未抓、待定。
- 语料页 13: java5 + database2(mysql94/redis48) + architecture3(mq38/cap19/systemdesign14) + tools3(linux/git/docker)。

## 抓取(已稳定)

```bash
.venv/Scripts/python -m crawler.main            # 抓 PAGES 清单全部
.venv/Scripts/python crawler/probe_groups.py    # 摸底站点板块结构
```
已处理:标题锚点、代码行号、页顶广告、嵌套 strong、全角标点保真、图片只留外链引用。

## 简历讲稿要点(面试往这引)

1. 全链路自研:爬虫→**Java 分块**(叶子标题切题,兼容两套标题结构)→**parent-child 子段切分**(围栏感知,代码块原子)→向量→**多粒度召回 + cross-encoder 统一重排**→生成;不套 LangChain
2. **评测数字**:golden 集去自环(LLM 改写真实问法 + 细节探针 + 负样本),recall@k / MRR 量化,`docs/eval-report.md` 有改动前后对比;自环旧评测是"拿题干搜题干"的假满分,会被面试官拆穿
3. **多粒度分数不可比**是实测踩出来的:子段与题目级向量混池排序会互挤(混合 recall@1 掉 10pp),重排统一打分解决;重排输入必须用"命中的证据"(子段正文),截父题前 800 字会把证据截没
4. 批改模式让 RAG 是承重墙:标准答案从原文捞出来当裁判;答案引用溯源防幻觉;**拒答双门槛**(余弦+重排分)治理域外硬答
5. 存储选型:pgvector 单库(业务表+向量+子段),Flyway 管 schema;**生成端/向量端/重排端三套独立配置**,LLM 换厂商不动向量
6. **向量模型指纹校验**:库内记 `index_meta`(模型名+维度+分块方案),检索前比对当前配置,不一致直接 409 提示重灌——专门防"同维度换模型不报错但检索静默变烂"这个典型坑;维度变更时 `reindex` 自动 ALTER 列类型并重建 HNSW
7. 多轮追问:指代在检索前用 LLM 改写成独立问题,history 同时参与召回与回答
8. 前后端完整:SSE 流式、引用卡片、批改结构评分、薄弱聚合
