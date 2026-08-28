# Session 2026-08-28 · 体检(ProfileDocument)详解

## 用户原始需求

> 你先给我讲一下第一次的体检是怎么做的?需要收集哪些信息?后面做什么用?

用户要求详细讲切分前的体检机制。

## 我做了什么

1. 重读 profiler.go(241 行)全文,确认 ProfileDocument 的完整扫描流程:
   - 5 类信号:基础统计、Markdown 标题、启发式结构、表格代码空行、语言
   - 逐行扫描 + inFence 状态机跳过代码块
   - 标题命中后 continue,不再匹配其他 pattern
2. 重读 tokens.go(160 行)确认语言检测和 token 预算:
   - DetectLanguage:数 CJK/Latin/德语变音,采样 4096 字符
   - charsPerToken:中文 1.7 / 英文 4.0 / 德语 4.5 / 混合 3.0
   - CharsForTokenLimit 带 0.9 安全系数
3. 确认 DominantHeadingLevel 2 段逻辑(首选 ≥3 次 / 退化最深)
4. 确认 SelectStrategy 3 个判断条件:
   - Tier1:MdHeadingTotal>=3 && HeadingDensity>0.005 && DominantHeadingLevel>0
   - Tier2:HeuristicMarkerTotal>=5 || FormFeedCount>0 || 中英德章节>0
   - Tier3:永远加
5. 写了 notes/13-体检ProfileDocument详解.md,含:
   - 11 种信号分类详解 + 后面干嘛用
   - 完整体检流程代码
   - DominantHeadingLevel 算法
   - SelectStrategy 判断 + 3 种典型链
   - 完整例子(公司报销制度文档)
   - 5 件应该带走的事

## 关键发现(给 AI 自己看,防失忆)

- 体检 = ProfileDocument,一次 O(N) 扫描,11 种信号
- 5 类信号:基础(3) + 标题(2) + 启发式(8) + 表格代码空行(3) + 语言(1)
- 真正影响策略的是前 3 类(标题、启发式、语言),其他是预留
- 标题用 MdHeadingCounts[1..6] map 记每级数量
- matchHeading 命中后 continue,不再匹配其他 pattern(防重复计数)
- inFence 状态机跳过 ``` 代码块内的行,防代码注释被误识别
- DominantHeadingLevel:首选 ≥3 次的最低层级,退化是最深的层级
- HeadingDensity = MdHeadingTotal / TotalLines,防"标题多但占比极低"
- SelectStrategy:
  - Tier1:MdHeadingTotal>=3 && density>0.005 && DominantHeadingLevel>0
  - Tier2:HeuristicMarkerTotal>=5 || FormFeedCount>0 || 中英德章节>0
  - Tier3:永远加
- 语言检测采样 4096 字符(DetectLanguage):CJK>=30% → 中文,CJK/Latin 都>=15% → 混合,德语变音/停用词 → 德语,否则英文
- charsPerToken:中文 1.7 / 英文 4.0 / 德语 4.5 / 混合 3.0
- CharsForTokenLimit 带 0.9 安全系数,防超 embedding API token 限制
- 语言影响 3 件事:Tier2 章节正则、token 预算、句末分隔符

## 下一步建议

体检讲透了。用户接下来要讲自己对全流程的理解,我对照代码核对。
等用户描述。