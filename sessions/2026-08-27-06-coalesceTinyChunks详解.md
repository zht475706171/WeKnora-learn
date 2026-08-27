# Session 2026-08-27 · 第六步:coalesceTinyChunks 合并碎 chunk 详解

## 用户原始需求
> Step 6:合并碎 chunk (coalesceTinyChunks)
> FAQ 类文档每段只有几十字,会被校验判"chunk 太小"。把相邻碎 chunk 合并到 ChunkSize/2,合并后面包屑用共享前缀。是怎么做的?面包屑用共享前缀是什么意思?能举个例子吗

## 我做了什么

1. 回顾 heading_splitter.go 第 139-203 行的 coalesceTinyChunks + commonHeadingPrefix 代码
2. 详细讲清楚 4 个合并条件:
   - sharedHeader != ""(共享前缀非空,安全阀)
   - cur.End == next.Start(相邻,保位置不变量)
   - curLen < target(target=ChunkSize/2,最小 200)
   - curLen+nextLen <= chunkSize(硬上限)
3. 详细讲清楚 commonHeadingPrefix 算法:
   - 按 \n 切成行数组
   - 从上往下逐行对比,到第一个不同行停
   - 把相同的前几行拼回来
4. 用一个完整 FAQ 文档(7 个 chunk)走了一遍合并过程,展示:
   - chunk0-3 合并(共享前缀 "# 常见问题\n## 安装问题")
   - chunk4 合并进去时共享前缀降级到 "# 常见问题"(因为 ## 安装问题 vs ## 使用问题 不同)
   - chunk5、6 继续合并
   - 最终整个文档被合并成 1 个 chunk(187字 < target 256)
5. 重点讲清楚"共享前缀降级"这个副作用:
   - 跨章节合并会让面包屑从深层降到浅层
   - 这是接受的取舍:保住 Tier 1 不降级 > 保住面包屑精确层级
6. 解释为啥共享前缀是必须的安全阀(防止 # 文档A 和 # 文档B 被糊一起)
7. 对比效果:不合并(Tier 3 切得更糟)vs 合并(粒度粗但保住语义边界)

## 关键发现(给 AI 自己看,防失忆)

- target = ChunkSize/2,最小 200(防 target 太小导致合并完还是碎)
- 共享前缀按行逐行对比,返回最后一个相同行前的所有行拼接
- 合并后重新编号 Seq(下游 knowledge.go 依赖 Seq 连续)
- 位置不变量:cur.End == next.Start 保证 Content 仍是原文连续切片
- 共享前缀降级是接受的副作用:跨章节合并面包屑从深层降到浅层
- 整个 FAQ 例子走下来合并成 1 个 chunk(187 < target 256,一直满足 curLen < target)

## 下一步建议

用户连续问了 Tier 1 的细节(EmbeddingContent、栈、MdHeadingCounts、子 chunk 面包屑、coalesceTinyChunks),Tier 1 现在讲透了。等用户决定是否继续 Tier 2/Tier 3,或转向向量化和入库/检索/后处理 enrichment。