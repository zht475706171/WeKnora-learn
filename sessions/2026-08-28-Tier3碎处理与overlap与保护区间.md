# Session 2026-08-28 · Tier 3 碎处理 + overlap 机制 + 保护区间 + rune 偏移

## 用户原始需求

用户连续追问几个核心问题:
1. Tier 3 按句号递归切,一句话才多少字,会不会一句一个 chunk?Tier 2 怎么保证不切太碎?碎了的处理?Tier 2 碎兜不住 Tier 3 也会碎吧?
2. 那我说下我对这个 overlap 的理解(用户描述理解),然后问:84 字里既有 。又有 \n 又有 \n\n,是取最后一个吗?取靠前的还是靠后的?取 。还是 \n\n?
3. LaTeX/表格/代码/图片/链接区间内是怎么判断和标识的?是体检时记录行号?数据结构是什么?分别存什么?
4. rune 偏移是什么意思?

## 我做了什么

1. 重读 splitter.go:396-518 的 mergeUnits,确认 Tier 3 怎么合并 unit
2. 回答 Tier 3 切碎问题:递归切只是切 unit,mergeUnits 合并到 512,句子是粒度不是 chunk
3. 回答 Tier 2 切碎问题:3 道防线(minChunkSize 防过早封口 + appendChunk 丢空白 + 校验降级)
4. 确认用户判断对:T2 碎降级 T3 可能更碎,但有 3 缓解(mergeUnits 合并 + overlap 平滑 + 父子分块 parent 兜底)
5. 确认用户 overlap 理解对:80 是默认值,受 chunkSize-nextLen 限制,窗口=maxOverlap+4
6. 纠正"找到第一个就停":实际是遍历所有候选,按优先级(1=段落>2=行>3=句末)+ 同优先级取最早
7. 详述 overlap 3 个约束:在 overlap 区域内 / 边界后非空白 / 不在保护区间内
8. 重读 splitter.go:117-201 的 protectedPatterns + protectedSpans + protectedSpansRune
9. 破除"体检记录"误解:保护区间是切分时实时算的,不是体检记录的
10. 详述保护区间数据结构:就一个 span{start, end},字节偏移,不分类型
11. 详述 7 种保护模式正则,3 步找保护区间(7 正则找匹配 → 排序 start 升序 + 长度降序 → 去重叠)
12. 详述各层怎么用保护区间:T3 buildUnitsWithProtection + T2 dropBoundsInsideSpans + T3 overlap insideProtected
13. 详述 rune 偏移:对比字节偏移 / rune 偏移 / 行号,为啥切分用 rune(中文不切坏)
14. 写了 4 篇笔记:notes/14-Tier3递归切与合并机制详解 / 15-两阶段切分思路与overlap机制 / 16-保护区间机制详解 / 17-rune偏移是什么意思

## 关键发现(给 AI 自己看,防失忆)

- 递归切只是切 unit,mergeUnits 合并到 512,句子是粒度不是 chunk 大小
- Tier 3 防碎 3 机制:mergeUnits 贪心合并(主)+ overlap 平滑 + 校验失败也返回
- Tier 2 防碎 3 道防线:minChunkSize 防过早封口(主)+ appendChunk 丢空白 + 校验降级 T3
- T2 碎降级 T3 可能更碎,但有 3 缓解:mergeUnits 合并 / overlap 平滑 / 父子分块 parent 兜底
- 父子分块是系统级兜底:parentSize=4096 大块给上下文,childSize=384 小块精确检索
- overlap 80 是默认值(约 15% of 512),不是动态算的,3 档:0/80/150-200
- maxOverlap = min(80, chunkSize - nextLen),受下个 chunk 剩余空间限制
- 窗口 = maxOverlap + 4(lookbehind for \r\n\r\n)
- originalWindowStart = 窗口字数 - maxOverlap,边界 end 必须 ≥ 这个
- 3 级优先级(数字越小越强):1=段落 \n\n > 2=行 \n > 3=句末 。？！/. 
- 选择规则:优先级高的赢 + 同优先级取最早
- 3 个约束:在 overlap 区域内 + 边界后非空白(hasMeaningfulTail)+ 不在保护区间内(insideProtected)
- 找不到合格边界 → 不 overlap,绝不切词中间
- 保护区间不是体检记录的,是切分时实时算的(每次切分重新扫)
- 数据结构就一个 span{start, end},字节偏移,不分类型
- 7 种保护模式:LaTeX/图片/链接/表格头/表格行/代码块/行内代码
- 3 步找保护区间:7 正则找匹配 → 排序(start 升序 + 同 start 长度降序)→ 去重叠(保留先出现)
- 字节偏移(正则返回,快) vs rune 偏移(切分用,中文不切坏),protectedSpansRune 做转换
- Chunk 的 Start/End 都是 rune 偏移,保证 End-Start = runeCount(Content) 不变量
- 3 层都用保护区间:T3 buildUnitsWithProtection / T2 dropBoundsInsideSpans / T3 overlap insideProtected

## 下一步建议

碎处理 + overlap + 保护区间 + rune 偏移都讲透了。接下来:
- C. 向量化与入库(完整链路:chunks 表 + 向量库)
- D. 检索与后处理 enrichment
- E. 端到端走一个具体场景
- F. 父子分块(SplitTextParentChild,parentSize=4096 / childSize=384)

等用户选。