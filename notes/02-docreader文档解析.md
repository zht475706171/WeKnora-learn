# 02 · docreader 文档解析是怎么做的(大白话版)

> 学习目标:看懂 Go 后端把一个文件丢给 docreader 后,docreader 怎么把它嚼成 Markdown+图片。重点搞懂 PDF 和 DOCX 这两种最常见的格式。

---

## 一句话总结

**docreader 是一个独立的 Python gRPC 服务**,专门把"乱七八糟格式的文档"变成"干净的 Markdown 文本 + 一堆图片引用",送给 Go 后端去切片、向量化、入库。
它**不存储、不切片、不调 LLM、不做 OCR**——只干一件事:**解析**。OCR 是 Go 后端的事(它把扫描页渲染成图片传回去,Go 侧再喂给 OCR/VLM)。

---

## 大背景:它在整个 WeKnora 里的位置

```
[用户上传 PDF]
    │
    ▼
[Go 后端 internal/...]
    │  通过 gRPC 调用 Read 接口
    ▼
[docreader(Python,端口 50051)]   ← 这一篇讲的就是这里
    │  返回:Markdown 文本 + 图片引用(base64 内联)
    ▼
[Go 后端拿到 Markdown]
    │  切块 → 向量化 → 存 pgvector
    ▼
[知识库]
```

关键点:**docreader 只负责"格式转换"**,把 PDF/DOCX/HTML 等变成统一的 Markdown。切片、向量化、入库都不归它管。

---

## 顶层目录(`docreader/`)

```
docreader/
├── main.py            # ① gRPC 服务入口(收请求、发响应)
├── auth.py            # 鉴权拦截器(支持 TLS)
├── config.py          # 配置(从环境变量读)
├── proto/             # gRPC 协议定义(.proto 生成的 Go+Python 代码)
│   └── docreader.proto
├── parser/            # ② 解析器主战场(各种格式的解析都在这)
│   ├── parser.py          # 调度入口(根据引擎+文件类型选解析器)
│   ├── registry.py        # 引擎注册表(builtin/markitdown/opendataloader)
│   ├── base_parser.py     # 解析器抽象基类
│   ├── chain_parser.py    # 解析器链(FirstParser/PipelineParser)
│   ├── pdf_parser.py      # ★ PDF 解析(最复杂)
│   ├── docx_parser.py     # ★ DOCX 解析(用 python-docx)
│   ├── docx2_parser.py    # DOCX 兜底链:先 markitdown,不行再 docx_parser
│   ├── doc_parser.py      # 老版 .doc
│   ├── excel_parser.py    # xlsx
│   ├── html_parser.py     # HTML
│   ├── markdown_parser.py # MD
│   ├── epub_parser.py     # EPUB
│   ├── mhtml_parser.py    # MHTML
│   ├── image_parser.py    # 图片(交给 Go 后端 OCR)
│   ├── web_parser.py      # URL 抓取
│   ├── markitdown_parser.py    # 微软 MarkItDown 库(万能兜底)
│   ├── opendataloader_parser.py  # 版面分析(需 Java 11+)
│   └── xmind_parser.py    # XMind 思维导图
├── models/
│   └── document.py     # Document 数据类(content + images + metadata)
├── client/             # Go 侧的 gRPC 客户端(给 Go 后端用)
│   ├── client.go
│   └── auth.go
└── splitter/           # 分块相关(但这块在 docreader 里几乎是空壳,真正分块在 Go 后端)
```

---

## gRPC 服务入口(`main.py`)

docreader 起一个 gRPC 服务,监听 **50051** 端口,对外暴露 3 个方法:

| RPC 方法 | 干啥 |
|---------|------|
| `Read` | 一次性解析(文件或 URL),返回 Markdown+图片(图片内联 base64,小文档用) |
| `ReadStream` | 流式解析(大文档用),先发一个 meta 帧(Markdown 文本),再一帧一帧发图片,避免单个消息太大 |
| `ListEngines` | 列出所有可用的解析引擎 |

### 一个请求是怎么被处理的(简化版)

```
Go 后端发 ReadRequest{file_content, file_type, config}
    │
    ▼
DocReaderServicer.Read()
    │
    ▼
Parser.parse_file() 或 parse_url()
    │  根据 engine + file_type 从 registry 拿对应解析器类
    ▼
实例化解析器 → 调用 parser.parse(content)
    │  返回 Document{content, images, metadata}
    ▼
把图片 base64 解码 → 包成 ImageRef → 塞进 ReadResponse
    │
    ▼
返回给 Go 后端
```

> 关键设计:**图片是内联 base64 跟着响应一起回的**,Go 后端再决定存到哪(本地/MinIO/COS)。docreader 自己不碰存储。

---

## 调度核心:`parser.py` + `registry.py`

### registry(注册表)长这样

```
引擎 = { 文件类型 → 解析器类 }

builtin 引擎(默认):
  docx  → Docx2Parser
  doc   → DocParser
  pdf   → PDFParser
  md    → MarkdownParser
  xlsx  → ExcelParser
  pptx  → MarkitdownParser
  epub  → EPUBParser
  html  → HTMLParser
  mhtml → MHTMLParser
  xmind → XMindParser
  jpg/png/... → ImageParser

markitdown 引擎(微软 MarkItDown 库,万能兜底):
  pdf/docx/pptx/xlsx/csv/md/doc → MarkitdownParser

opendataloader 引擎(需 Java 11+):
  pdf → OpenDataLoaderParser(版面分析,精度高)
```

### 调度逻辑(就 3 步)

1. 用户在请求里指定 `parser_engine`(可选,默认 builtin)
2. `registry.get_parser_class(engine, file_type)` 去查表:
   - 指定的引擎支持这个文件类型 → 用它
   - 不支持 → **自动回退到 builtin**
3. 实例化解析器,调 `parser.parse(content)`,返回 `Document`

### 一个聪明的"伪装检测"

`parser.py` 里有 `detect_effective_file_type()`:有些 WPS/Word 文档其实是老的 `.doc`(OLE 格式),只是被人改名成了 `.docx`。它看文件头的 magic bytes(`D0 CF 11 E0...`),一旦识别就把 `.docx` 重定向成 `.doc` 走 DocParser,避免喂坏 DocxParser。

---

## ★ PDF 解析(`pdf_parser.py`)—— 这个最复杂,重点讲

PDF 之所以难,是因为它有三种"长相":
1. **原生数字 PDF**:文字直接是 PDF 里的文本对象(Word 导出的)
2. **扫描 PDF**:每一页就是一张大图,根本没有文字层(纸质书扫描的)
3. **混合 PDF**:前几页是目录(有文字),正文是扫描页(图片)

WeKnora 的策略:**逐页分类,各走各的路**。

### 一页一页来,每页分两类

每页都跑一遍分类函数 `_classify_page()`:

```
判断标准(两个信号):
  ① 图片覆盖面积占页面比例 ≥ 0.5 → 扫描页
  ② 文字几乎为空(<10字符)且图片覆盖 ≥ 0.1 → 扫描页
  ③ 否则 → 文本页
```

为什么用"图片覆盖比例"而不是"字符数"?
因为扫描 PDF 经常带着一层**低质量的 OCR 文字**(别人用工具跑过一遍),字符数不低但全是乱码。看"图片占多大"才准。

### 文本页怎么处理

```
文本页(原生数字)
   │
   ├─ 拿 pdfium(pypdfium2) 抽取文字
   │
   ├─ 难点 1:多栏排版乱序
   │    → _split_columns() 用几何 XY 切割,按列重排
   │
   ├─ 难点 2:字符之间没空格(OCR 文字层常见)
   │    → _join_line_glyphs() 按字形间距推断空格
   │
   ├─ 难点 3:哪些行是标题?
   │    → _segments_to_markdown() 看字号(行高)相对中位数,大字 → markdown 标题
   │
   ├─ 难点 4:隐藏文字(不可见 render-mode 3)、页外文字
   │    → FILTER_HIDDEN_TEXT 过滤(也防 prompt 注入)
   │
   ├─ 难点 5:页眉页脚/arXiv 水印
   │    → _strip_repeating_lines() 跨页重复出现就删
   │
   ├─ 难点 6:图表内坐标轴文字漏进正文
   │    → _strip_chart_text_debris() 用正则识别 "0 1 2 3 4 5" 这种行删掉
   │
   └─ 难点 7:纯矢量图(没嵌入位图,只有线和字)
        → 找 "Figure N." 标题,把标题上方的区域渲染成 JPEG,塞进 Markdown
```

**最后输出**:这一页的 Markdown 文本 + 可选的图片引用(`![xxx](images/xxx.jpg)`)。

### 扫描页怎么处理

```
扫描页
   │
   ├─ 把整页用 pdfium render 成 JPEG(DPI 200,质量 85)
   │
   ├─ 大 PDF 用多进程并行渲染(forkserver,每个 worker 独立打开 PDF)
   │  为什么不用多线程?因为 pdfium 的 C 库不是线程安全的,
   │  并发会死锁,整个 gRPC 服务会被卡死。
   │
   ├─ 标记 metadata: image_source_type = "scanned_pdf"
   │
   └─ 输出:![page_1.jpg](images/page_1.jpg) —— 整页一张图
        Go 后端拿到这张图会调 OCR/VLM 服务去识别成文字
```

### 混合 PDF 的完整流程(`_route_locked`)

```
Pass 1:对每页做分类 + 抽文本页文字 + 抽矢量图
   ↓
Pass 2:只渲染被分类成"扫描"的页(重活,限流)
   ↓
Pass 3:从文本页里抽嵌入的位图(图章/插图)
   ↓
按页序拼装 Markdown,返回 Document
```

### 一个非常关键的并发安全设计

```python
_PDFIUM_LOCK = threading.Lock()
```

`pdfium` 底层 C 库**进程级不线程安全**。两个 gRPC worker 同时解析 PDF 会互相腐蚀状态、死锁。所以整个 PDF 解析被一把全局锁串起来,任意时刻只解析一个 PDF。
非 PDF 格式(docx/xlsx)不受影响,可以并发。

扫描页渲染则用**多进程**(每 worker 独立打开 PDF),绕开这个锁——这是大 PDF 性能的关键。

### 强制扫描模式

如果用户知道这 PDF 文字层是垃圾(比如 web 打印的、扫描的),可以传 `pdf_force_scanned=true`,跳过分类,**所有页都当扫描页处理**。这是给烂 PDF 留的兜底。

### 异常兜底

`PDFParser._route` 一旦失败,自动 fallback 到 `PDFScannedParser`(把所有页都当扫描页渲染),保证"再烂的 PDF 也能出个结果"。

---

## ★ DOCX 解析(`docx_parser.py`)

DOCX 比 PDF 简单太多,因为它本质是个 ZIP,里面是结构化 XML,本来就有"段落/表格/图片"的概念。

### 默认走"链式兜底"

`docx2_parser.py` 定义的 `Docx2Parser` 其实是:

```python
class Docx2Parser(FirstParser):
    _parser_cls = (MarkitdownParser, DocxParser)
```

`FirstParser` 是责任链模式:`MarkitdownParser` 先试,失败/结果空了再试 `DocxParser`。所以默认走微软的 MarkItDown(微软出的库,万能),再不行才走自研的 `DocxParser`。

### 自研 DocxParser 干啥

`DocxParser.parse_into_text()` 流程:

```
1. 用 python-docx 打开 DOCX
2. 用一个 Docx 处理器(自研,在文件下半部分)逐段遍历:
   - 段落 → 文本
   - 表格 → 转成 "| a | b |" 形式的 Markdown 表格
   - 图片 → 抽出来,base64 编码,生成 images/xxx.png 引用
3. 多 worker 并发处理(因为图片解码是 CPU 密集)
4. 拼装 Markdown 文本 + 图片字典
```

### 兜底分支 `_parse_using_simple_method`

如果主流程炸了(比如 python-docx 的某个 bug),就降级到"简版":只抽段落文字 + 表格文字,不处理图片。再不行就返回空 Document。

---

## 数据流总览(把 docreader 内部串起来)

```
ReadRequest{file_content, file_type, parser_engine}
    │
    ▼
DocReaderServicer.Read()
    │
    ▼
Parser.parse_file()
    │
    ├─ detect_effective_file_type()  ← 文件头 magic 修正类型
    │
    ├─ registry.get_parser_class(engine, file_type)  ← 选引擎+回退
    │
    ▼
具体解析器.parse_into_text(content)
    │
    │  ┌─ PDFParser:逐页分类→文本页抽文字/矢量图,扫描页渲染成图
    │  │  (受 _PDFIUM_LOCK 全局锁保护,扫描页多进程并行)
    │  │
    │  ├─ Docx2Parser:先 MarkItDown,失败再 DocxParser
    │  │  DocxParser:python-docx 抽段落/表格/图片
    │  │
    │  └─ ...其他格式类似
    │
    ▼
Document{content: Markdown文本, images: {ref_path: base64}, metadata}
    │
    ▼
_resolve_images()  ← base64 解码成 ImageRef
    │
    ▼
ReadResponse{markdown_content, image_refs[], metadata}
    │
    ▼
(流式版 ReadStream:先发 meta 帧,再一帧一帧发图)
    │
    ▼
返回 Go 后端
```

---

## 设计上的几个聪明点(值得记)

1. **职责单一**:docreader 只解析,不切片不存储不调 LLM。Go 后端是"大脑",docreader 是"嘴巴+眼睛"。
2. **统一输出 Markdown**:无论什么格式进去,出来都是 Markdown + 图片引用。Go 后端只面对一种数据结构,极大简化了下游。
3. **逐页分类(PDF)**:混合 PDF 不再一刀切,每页用最合适的方式处理。
4. **多层兜底**:
   - 引擎选不到 → 回退 builtin
   - PDF 解析失败 → 全渲染成图
   - DOCX 主解析失败 → 简版解析
   - 简版还失败 → 空 Document(让 Go 后端报错)
5. **图片内联 base64 跟响应一起回**:Go 后端不用再回去拉文件,省一次往返。大文档用 ReadStream 流式发,避免 gRPC 单消息超 50MB 限制。
6. **pdfium 全局锁 + 多进程渲染**:解决 C 库非线程安全问题,同时不让大 PDF 卡死服务。
7. **隐藏文字过滤**:顺便防了 PDF prompt 注入(攻击者把不可见文字塞进 PDF,诱导 RAG 系统回答坏话)。

---

## 这一节你应该带走的 4 件事

1. **docreader 是独立 Python gRPC 服务,端口 50051**,Go 后端调它把文档变成 Markdown+图片。
2. **统一输出 Markdown** 是核心设计,所有格式殊途同归。
3. **PDF 最难**,逐页分类(文本页/扫描页),文本页做版面重建,扫描页渲染成图交给 Go 侧 OCR。pdfium 全局锁+多进程渲染是为了不卡死服务。
4. **DOCX 用责任链**:先 MarkItDown,再自研 python-docx 解析,多层兜底。

---

## 下一节可选

- **A. 切块**:Go 后端拿到 Markdown 后怎么切成 chunk?(`internal/.../splitter` 或类似)
- **B. OCR/VLM**:扫描页图片回到 Go 后端后,是怎么被 OCR 成文字的?
- **C. 向量化与入库**:文本块怎么变成向量,怎么存进 pgvector?
- **D. 一个具体格式深挖**:比如 Excel、EPUB、XMind 怎么解析的

你想走哪条?