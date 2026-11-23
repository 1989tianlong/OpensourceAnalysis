# Indra OCR 深度分析报告

> 仓库：https://github.com/datalab-to/indra
> 分析版本：`indra-ocr@1.1.2`（模型权重 HF tag `v1.1.0`）
> 分析日期：2026-11-23
> 许可证：代码 Apache 2.0；模型权重 modified OpenRAIL-M

---

## 1. 一句话概述

**Indra OCR 是由 Datalab 推出的一个"在 CPU 上就能跑"的轻量级文档智能（Document Intelligence）引擎**：它把 PDF 和图片转成结构化的 Markdown / 纯文本 / 带坐标框的 JSON，能读手写、80+ 种语言、任意角度的扫描页，同时把"行检测 → 文字识别 → 阅读顺序 → 版面分类 → Markdown 渲染"整条流水线压缩到只有 **约 5100 行 Python**、**不依赖 PyTorch、不需要 GPU**（只用 ONNX Runtime + OpenCV + pypdfium2）。

它最反直觉的卖点是一句 README 里的自述：

> *Indra OCR is a 1M-parameter OCR model that runs on CPU.*

也就是说，在一个"动辄 7B VLM 做 OCR、必须上 GPU"的当下，Datalab 反向押注：**用极小的卷积/CTC 模型 + 精心设计的传统 CV 后处理 + 小 n-gram 语言模型，在纯 CPU 上做到又准又快**。32 核上约 44 页/秒，约为 Tesseract 的 15 倍、PaddleOCR PP-OCR 文本模型的 10 倍（README 基准）。

> 名字来源：印度神话中的雷神 **因陀罗（Indra）**。同门还有一个更强的 VLM 模型叫 **Chandra（月神）**，跑在 Datalab 的托管平台上，用于处理手写、数学、复杂表格等 Indra 不擅长的场景——这是理解本项目定位的关键：**Indra 是"快而廉价"的那一端，Chandra 是"准而昂贵"的那一端**。

---

## 2. 它解决什么问题

### 2.1 背景：文档智能的"精度-成本"三角

把文档（尤其是扫描件、PDF）转成机器可读的结构化数据，是 RAG、Agent、票据处理、档案数字化等场景的前置步骤。业界现有方案大致分两派，各有各的痛：

| 方案流派 | 代表 | 优势 | 痛点 |
| --- | --- | --- | --- |
| **传统 OCR 引擎** | Tesseract、PaddleOCR PP-OCR | 快、能上 CPU、便宜 | 只出"文字行"，**没有版面结构**（标题/表格/列表/阅读顺序），复杂版面直接乱序；多语言与手写弱 |
| **重型文档流水线** | PaddleOCR PP-StructureV3 | 有版面、表格、公式 | **极慢**（README 实测中位数约 41 秒/页，约为 Indra 的 190 倍）、重、部署复杂 |
| **大模型（VLM）方案** | GPT-4o-vision、各种 OCR-VLM、Chandra | 最准，复杂版面/手写/公式强 | **必须 GPU、成本高、延迟大、吞吐低**，不适合海量批量处理 |

一句话概括行业现状：**"准的都很贵很慢，快的都不懂版面"。**

### 2.2 Indra 要填的坑

Indra OCR 明确把自己定位在"**海量、便宜、CPU、够用**"这一档，它要解决的核心矛盾是：

1. **成本/门槛问题**：大量企业没有 GPU 集群，也不愿为每页文档支付高昂的 VLM 推理费。Indra 让普通 x86/ARM CPU 就能以 44 页/秒的吞吐跑 OCR，把"每页成本"压到接近零。
2. **"只有文字、没有结构"问题**：传统 OCR 输出的是散乱的行，下游要重建段落、标题、表格、阅读顺序非常麻烦。Indra **原生输出 Markdown 与带 `label` 的结构块**（`Section-Header`、`Table`、`List-Group`、`Caption`、`Footnote`、`Page-Header/Footer`、`Form` 等），直接可喂给 LLM/RAG。
3. **多语言与真实世界扫描的鲁棒性**：80+ 语言（含中日韩、阿拉伯/波斯/希伯来 RTL、印地语系、泰/高棉等），能自动扶正 45° 以内的倾斜、识别横置/倒置页、识别深底浅字，并且**优先读 PDF 的数字文本层**（数字 PDF 比 OCR 又快又零误差），只对缺文字的部分做 OCR。
4. **吞吐问题**：不是"单页快一点"，而是为**批量**设计的——多进程、每核一个 worker、一页一个任务，单份长 PDF 也能吃满所有核。

### 2.3 它"不解决"什么（边界清晰）

README 用一整节 **[Limitations](#)** 诚实划线——这本身是理解其定位的关键：

- **复杂表格**（合并单元格、跨行跨列、无线框）会错位；简单网格表格没问题。
- **手写**可读但不好（OmniParseBench 手写测试仅 25.2%）。
- **退化扫描**（老旧/褪色/低分辨率）差（29.5%）。
- 数学默认按纯文本输出，需 `--math` 才转 LaTeX。
- 竖排页面、超宽横幅页、工程图里的文字效果较差。

> 对上述场景，Datalab 会把你引导到更准的 Chandra（托管 API）。**Indra 是产品矩阵里的"走量"入口，不是"全能"答案。**

---

## 3. 它是如何解决的（核心原理）

整个项目的精髓在于：**模型故意做得很小，把大量"智能"放在后处理（几何 + 规则 + 小语言模型）里，用传统 CV 的确定性补偿小模型的能力不足。** 这是它能"小、快、CPU"的根本原因。

### 3.1 端到端流水线

从 `indra_ocr/pipeline.py` 的 `OrtOCR.__call__` / `_read` 可还原出完整主干（每页一张图）：

```
页面图片 (灰度 uint8)
   │
   ▼
[0-预处理]  缩略图 → 估计倾斜角 estimate_skew（按行锐度搜索 ±45°）
   │         → 方向模型 orient.onnx 判断 0/90/180/270 及"竖排"
   ▼
[_turn]     把页面转正（quarter turn + deskew）
   │
   ▼
[1-检测]    det.onnx（LineNet）→ 所有文字行的框（同时给出"图片/图表区"框）
   │         自适应：若行中位高 < 9px，则放大重检一次（小字号补救）
   ▼
[2-识别]    rec*.onnx（CTC）逐行识别 → (text, conf, tokens)
   │         · 遮罩 mask.onnx（CenterMask）先剥掉行窗外的干扰
   │         · 按脚本分发：script.onnx 把每行路由到 base/cjk/rtl/other 4 个识别器
   │         · 行置信度低时，用 n-gram 语言模型 lm.npz 做 CTC 前缀 beam search 重读
   │         · 深底浅字：自动反色
   ▼
[3-整理]    split_line 在宽间隙处切分（一行可能含多列/表格单元格）
   │         tidy_lines 去除孤立噪点、对齐
   ▼
[4-版面]    order.onnx（OrderNet）→ 每行的块类别 + 每对行"是否同块"的 logit
   │         （无模型时回退到 XY-cut 几何切分）
   ▼
[5-Markdown] 按阅读顺序遍历，块边界=同块 logit 转折处 → 按类渲染
   │         （标题加 ##、连续段落合并、列表、表格拼成 HTML <table>、页眉页脚默认丢弃）
   ▼
Page(markdown, text, lines, blocks, source, rotation, skew, timings …)
```

关键设计点逐一拆解如下。

### 3.2 行检测：LineNet + 自适应重检

- **模型**：`det.onnx`（代码里叫 LineNet，`det_post.py` 定义了其训练/推理共享的前后处理）。它输入整页缩放到长边 `det_long_side` 的墨迹图（`1 - gray/255`，0=纸、1=墨），Padding 到 32 的倍数；输出每个文字行的框，以及（若模型带第 4 通道）**图片/图表区的框**。
- **几何参数**：`CORE_FRAC=0.4`（核心带占行高比例）、`CORE_X_SHRINK=0.15`、`OUT_STRIDE=2`——训练时把"行中心带"编码成 target，推理时解码回框，保证训练/推理几何一致。
- **自适应二次检测**（`detect`）：若检测出的行中位高 < `ADAPTIVE_MIN_H=9px`（缩略图里太小），就把页面重新放大到目标行高 12px 再检一次，专门救"小字号密排"。
- **为什么这么设计**：一个轻量检测器 + 兜底式二次搜索，比堆一个巨型检测器在 CPU 上划算得多。

### 3.3 行识别：CTC + CenterMask + 脚本路由 + 语言模型重读

这是"小模型做大事"的核心。识别器是 **CTC** 结构的（每列出一字符），模型本身很小（主识别器 `rec.onnx` 约 1.25M 参数），但有四层工程强化：

1. **CenterMask 遮罩**（`line_mask.py`）：识别前先裁出"行窗"（上下各 1 倍行高、左右各 0.5 倍行高的窗口），用 `mask.onnx` 把窗内的其它行/噪声概率 > 0.5 的像素置为"纸"。**训练与推理必须走同一个 mask 模型**（`masked_window` 是两者共用函数），否则分布漂移。这一招让相邻行不互相污染。
2. **裁剪规则统一**（`rec_preproc.py`）：`PAD_Y=0.2`、`PAD_X=0.3`、缩放到 `LINE_HEIGHT=32`，宽度 pad 到 8 的倍数；并有 `MIN_BATCH_WIDTH=32` 的防呆（补足极窄批，避免 ORT 的 `input_shape {1,0}` 崩溃）。
3. **脚本路由**（`script.onnx` + 四个识别器）：每行先由小分类器判断属于哪个脚本组——
   - `rec.onnx`（base）：拉丁/西里尔/希腊 + 拉丁手写；
   - `rec_cjk.onnx`：简繁中文、日文（假名+汉字）、韩文；
   - `rec_rtl.onnx`：阿拉伯/波斯/乌尔都/普什图/库尔德/希伯来（视觉序读，再由 `rtl.flip` 转逻辑序）；
   - `rec_world.onnx`：天城文/孟加拉/泰米尔/泰卢固/卡纳达/马拉雅拉姆/古吉拉特/古木基/奥里亚/泰/老挝/高棉/缅甸/僧伽罗/格鲁吉亚/亚美尼亚/埃塞俄比亚等。
   - 路由阈值 `ROUTE_SURE=0.95`：分类器没把握时，base 也读一遍，**取置信度更高者**（"双读择优"）。
4. **n-gram 语言模型 beam search 重读**（`lm.py`）：对**置信度低于 `LM_GATE=0.9` 的行**，用 CTC 前缀 beam search 保留多条候选，用一个 **backoff n-gram 语言模型**（`lm.npz`，用 numpy 存成排序表，无 pickle）按 `score = log P_ctc + α·log P_lm + β·tokens` 重排。**识别器自身的概率仍主导**，所以写得清楚但拼写奇怪的词不会被 LM 强行"纠正"掉。这是把"手写/模糊"准确率拉起来的关键。

> 注意：这里没有任何 Transformer/VLM。Hugging Face 上的模型总计约 76MB（30MB 网络 + 46MB 语言模型），主识别器 1.25M 参数。

### 3.4 阅读顺序与版面分类：OrderNet（或 XY-cut 兜底）

- **OrderNet**（`order.onnx` + `order_post.py` / `layout_post.py`）：对检测出的每一行提取 **10 维几何特征 + 6 维文本特征**（`N_GEOM=10, N_TEXT=6`），预测两件事：
  1. 每行的**块类别**（`Text / Section-Header / Page-Header / Page-Footer / List-Group / Table / Caption / Footnote / Table-Of-Contents / Form`）；
  2. **每对行"是否同属于一个块"的 logit**。
  沿阅读顺序走，**同块 logit 从正转负处即为块边界**。
- **表格特殊处理**：若 OrderNet 带"表格头"（`tab`/表头行列表），先按单元格切分再二次跑顺序模型；无表头时按几何拼表（按垂直中心分行、以最满行锚定列）。
- **XY-cut 兜底**：没有 `order.onnx` 时，用经典递归 XY-cut（横向/纵向投影切分）得到块顺序，只是没有可读的块标签。**竖排页面**（中日文）也强制走 XY-cut——因为 OrderNet 没见过竖排样本，会把竖排的短列误判为"逐行读的表格"。
- **RTL 页面**：把整页几何左右镜像后再做顺序与 Markdown，使"第一列就是最右一列"，最后把框映射回原坐标。
- **PDF 文本层优先**：`document.py` 里，`mode="auto"` 先用 pdfium 读 PDF 自带文本层并做**可信任性检查**（见 3.6），可信则直接用（零识别误差、极快），只对图像区域内缺字做 OCR；不可信（扫描件/OCR 图层/坏字体映射/RTL 视觉序）才整页 OCR。

### 3.5 Markdown 渲染：把结构"翻译"成文档

`layout_post.py` 按每个块的类别做渲染：

- `Section-Header` → `## 标题`；
- 普通段落（`Text/Caption`）→ 合并成一段，并处理**行尾连字符断词**（`tumori-` + `genesis` → `tumorigenesis`）和 **CJK 行间不加空格**；
- `List-Group` → 每个列表项一行（靠 `•`/编号/字母编号识别条目起点，`MARKER_GAP=2.5` 行高内合并被切断的 bullet）；
- `Table` → 拼成 HTML `<table>`（单元格按 `CELL_GAP=1.8` 行高的间隙切分）；
- `Page-Header/Footer` → 默认丢弃（`--keep-furniture` 才保留）；
- 公式 `$...$`、图片块 `Picture` 按阅读顺序插入。

这套规则是**确定性代码**，易于调试、零推理成本，正是"小模型 + 大后处理"哲学的最后一块拼图。

### 3.6 工程细节：那些让它"能用"而非"能跑"的东西

- **文本层可信度检查**（`textlayer.py`、`document.py`）：
  - 隐形文字（渲染模式 3）比例 > 5% → 判为"扫描件上的 OCR 图层"，不可信；
  - 替换符/私用区/控制字符比例 > 0.5% → "坏字体映射"，不可信；
  - 有文字但下方无墨迹的行的比例 > 3% → 隐藏/错位文字，不可信；
  - RTL 字母以视觉序存储（Chrome 打印的阿拉伯文常见）→ 不可信，改 OCR。
  - **legacy 字体检测**（v1.1.2）：很多印地语 PDF 用 Kruti Dev/DevLys 字体，文本层存的是拉丁码却显示天城文。Indra 会抽样几条文本层行，用识别器核对脚本；若"层是拉丁、图是别的脚本"，则整页 OCR。该页词准确率从 0.01–0.20 提到 0.85–0.89。
- **方向与倾斜**（`estimate_skew` / `orientation`）：`orient.onnx` 用 5 类（0/90/180/270 + 竖排），阈值 `ORIENT_MIN=0.95`（宁可不转，避免误转丢整页）；倾斜用"行方向锐度"搜索 ±45°。转正后所有框再映射回原图坐标。
- **深底浅字**（`line_darkness`）：逐行判定是否为反色文本，是则反色后再识别。
- **并行**（`parallel.py`）：ORT 的多线程对单页帮助有限（渲染、文本层、版面都在单核上），所以为批量设计了**多进程**：每核一个单线程 worker、一页一个任务，用 `usable_cores()` 读 `sched_getaffinity`/cgroup 尊重 taskset 与容器配额，用 `WORKER_MEMORY=1.5e9` 上限防止巨扫描撑爆内存。pdfium 非线程安全，所以用进程而非线程。
- **GPU 可选**：默认 CPU，传入 `providers=["CUDAExecutionProvider", …]` 即可用 `onnxruntime-gpu`；并在 CUDA 上**每次运行归还 ORT arena 内存**（`gpu_run_options`，`enable_memory_arena_shrinkage`），避免"按历史最大页一直占着"。
- **零依赖膨胀**：依赖仅 `numpy / opencv-python-headless / onnxruntime / pypdfium2 / huggingface_hub`，无 torch。首次运行从 HF 下载约 76MB 模型并缓存；支持 `INDRA_OCR_MODEL_DIR` / `--model-dir` 离线。

---

## 4. 在什么场景下使用

Indra 的甜区是"**量大、CPU、追求结构化输出、版面相对规整**"的文档。典型场景：

1. **RAG / LLM 知识库入库**：把成批 PDF、扫描件、发票、合同转成带结构的 Markdown，直接切块喂给检索与生成。这是它最主打的用途——输出本身就是"LLM 友好"的 Markdown + 结构块。
2. **档案/票据/表单批处理**：服务器无 GPU、CPU 集群富余，需要高吞吐（44 页/秒）与低单价；`Form`、`Page-Header/Footer` 等标签还能辅助后续字段抽取。
3. **多语言文档处理**：一份混合中/英/阿拉伯/印地语系文档，自动按脚本路由到对应识别器，无需人工分拣。
4. **真实世界扫描**：手机拍的歪斜、横置、倒置、深底浅字（幻灯片截图）也能读。
5. **数字 PDF 的"快通道"**：大量电子版 PDF（有文本层）走文本层直读，比 OCR 更快更准，只在缺字处补 OCR。
6. **离线/隐私敏感**：模型完全本地，文件不出机器（viewer 也不上传）。适合合规要求高的环境。

**不适合**：复杂合并单元格表格、重度手写、老旧褪色扫描、复杂公式为主、需最高精度的场景——这些请上 Chandra 或更重的 VLM 方案。

---

## 5. 如何使用

### 5.1 安装

```shell
pip install indra-ocr            # 约 76MB 模型首次运行自动从 HF 下载并缓存
# 或用最新代码：
pip install git+https://github.com/datalab-to/indra.git
```

要求 Python ≥ 3.10，Linux（x86_64/aarch64）与 macOS（Apple silicon）测试通过，安装 ONNX Runtime / OpenCV / pypdfium2，**不需要 PyTorch / GPU**。

### 5.2 命令行

```shell
indra-ocr paper.pdf                      # Markdown 输出到 stdout
indra-ocr *.pdf --out results/           # 每个输入一个 .md，多核并行
indra-ocr scan.png --format json         # 行、框、版面块、置信度、耗时
indra-ocr paper.pdf --format text        # 纯文本，一行一行
indra-ocr paper.pdf --mode ocr --timings # 忽略文本层，全页 OCR；打印每页耗时
indra-ocr paper.pdf --math               # 数学写成 LaTeX（默认关，数学页更慢）
indra-ocr paper.pdf --keep-furniture     # 保留页眉页脚
indra-ocr *.pdf --out results/ --cores 8 # 限制核数
indra-ocr-viewer --port 8765             # 本地 Web 可视化（http://localhost:8765）
```

> 常用选项速记：`--format [md|text|json]`、`--mode [auto|ocr|text]`、`--dpi`（默认 200）、`--max-pages`、`--password`（加密 PDF）、`--model-dir`（离线模型目录）。CLI 遇到读不了的文件会跳过继续，有失败则以退出码 1 结束。

### 5.3 Python API

```python
import indra_ocr

conv = indra_ocr.load(threads=4)          # mode="auto" | "ocr" | "text"
for i, page in conv.convert("paper.pdf"):
    print(page.source, page.timings["total_ms"])
    print(page.markdown)                  # 标题/列表/表格(HTML)
    page.blocks                           # [{"label", "bbox", "text"}] 阅读顺序
    page.lines                            # [Line(box, text, conf)] 阅读顺序
```

图片文件与 PDF 用法一致（PNG/JPEG/TIFF/BMP/WebP，TIFF 每帧一页）；内存里的图可直接传灰度 `uint8` numpy 数组给 `conv.ocr`。

批量多核：

```python
import indra_ocr

if __name__ == "__main__":                 # 多进程会重新 import 本文件，必须有 guard
    for path, result in indra_ocr.convert_many(["a.pdf", "b.pdf"], cores=None):
        if isinstance(result, indra_ocr.InputError):
            continue
        for i, page in result:
            print(path, i, page.source)
```

### 5.4 输出结构（每页一个 `Page`）

| 字段 | 含义 |
| --- | --- |
| `markdown` | 结构化 Markdown（标题 `##`、段落、列表、表格为 HTML） |
| `text` | 每检测行一行、按阅读顺序的纯文本 |
| `lines` | `Line(box, text, conf)`，`box` 为像素 `x0,y0,x1,y1` |
| `blocks` | 版面块 `{"label", "bbox", "text"}`，阅读顺序 |
| `source` | `text`（文本层）/ `ocr` / `hybrid` / `error` |
| `rotation`, `skew` | 应用的旋转角度、扶正的倾斜 |
| `vertical` | 是否竖排（中日文），框仍在原页坐标 |
| `timings` | 分步毫秒，含 `total_ms` |
| `note` | 该页走此路径的原因 |

块标签：`Text, Section-Header, Page-Header, Page-Footer, List-Group, Table, Caption, Footnote, Table-Of-Contents, Form`（另有 `Picture`）。

### 5.5 可视化调试

`indra-ocr-viewer` 启动本地 Web 页：拖入 PDF/图片，每页叠加显示**行框、版面块、阅读顺序**，并排看渲染 Markdown、Markdown 源码、逐行置信度、JSON、各阶段耗时；可在页面内调流水线参数（文本层/OCR、数学、页眉页脚、阅读顺序、DIP 等）。文件不出本机；加 `--tesseract` 可与 Tesseract 对比。**排错时这是最有用的工具**——所有"为什么这行读错/这块归错类"都能在叠加图上定位。

---

## 6. 架构与模块地图（源码级）

```
indra_ocr/
├── __init__.py       # 对外 API：load() / Converter / convert_many / model_dir
├── pipeline.py  (1063) # ★核心：OrtOCR（检测/识别/顺序/渲染）、倾斜、方向、Page/Line
├── document.py   (426) # ★整文档：PDF 文本层优先、可信任性检查、页级调度、数学分支
├── textlayer.py  (363) # pdfium 文本层读取 + 可信度判定（隐形字/坏字体/无墨/RTL视觉序）
├── det_post.py   (155) # LineNet 前后处理（训练/推理共享，torch-free）
├── rec_preproc.py(155) # 行裁剪规则（PAD/缩放/宽度 pad）+ MIN_BATCH_WIDTH 防呆
├── line_mask.py  (153) # CenterMask 行窗遮罩（训练/推理共用 mask.onnx）
├── lm.py         (254) # n-gram 语言模型（numpy backoff 表，无 pickle）
├── order_post.py (352) # 阅读顺序/块特征、解码、指标（torch-free 部分）
├── layout_post.py(381) # 块→Markdown 渲染（标题/段落/列表/表格/页眉页脚）
├── rtl.py        (162) # RTL 检测与视觉序↔逻辑序翻转
├── math.py       (149) # 数学行：分类器 + attention 解码器 MathRec（输出 LaTeX）
├── vocab.py      (292) # 词表/编解码
├── parallel.py   (212) # 多进程批量（每核一 worker、内存上限、pdfium 安全）
├── models.py      (86) # 模型目录解析：本地/环境变量/HF 下载
├── cli.py        (189) # 命令行入口
└── viewer.py     (648) # 本地 Web 可视化
```

代码总量：`indra_ocr/` 约 **5100 行 Python**（含 648 行 viewer）；全仓含测试/示例/bench 约 6665 行。**主体逻辑高度集中在 `pipeline.py`（1063 行）与 `document.py`（426 行）**，其余都是围绕它的"专用小模块"，每个模块职责单一、注释极详尽（几乎每个常量都写了"为什么是这个值、测出来代价多少"），工程质量很高。

**六类 ONNX 模型**（按需加载、可选降级）：

| 模型 | 作用 | 缺失时 |
| --- | --- | --- |
| `det.onnx` | 行检测（+图片区通道） | 必需 |
| `rec.onnx` | 主 CTC 识别（拉丁/西里尔/希腊/拉丁手写） | 必需 |
| `mask.onnx` | CenterMask 行窗遮罩 | 跳过遮罩 |
| `order.onnx` | OrderNet 版面分类+阅读顺序 | 回退 XY-cut，无标签 |
| `orient.onnx` | 0/90/180/270 + 竖排方向判断 | 假设页面正向 |
| `script.onnx` + `rec_cjk/rtl/world` | 脚本路由 + 分组识别器 | 全部走 base |
| `math*.onnx` | 数学识别（分类器 + attention 解码） | `--math` 不可用 |
| `lm.npz` / `lm_<group>.npz` | n-gram 语言模型重读 | 仅贪心解码 |

> 这种"**核心两件套必需、其余全部可选降级**"的设计，是它能按场景裁剪体积与耗时、并能优雅容忍模型缺失（例如只有 base 模型也能跑）的原因。

---

## 7. 设计哲学与可借鉴之处

1. **"小模型 + 重后处理"是被低估的路线**。当行业默认"OCR 就得上 VLM"时，Indra 用 1M 参数模型 + 精细几何规则 + 小 n-gram LM，在纯 CPU 上做出了可用的结构化 OCR。它证明：**很多任务是"工程问题"而非"模型容量问题"**。
2. **训练/推理一致性的极致追求**：`det_post.encode/decode`、`line_mask.masked_window`、`rec_preproc.crop_line` 都刻意做成"训练和推理调用同一个函数"，从源头杜绝分布漂移。这是小模型能稳的关键。
3. **诚实划线、产品分层**：README 明列 limitations，并把难场景导向更强的 Chandra。**一个组件清楚自己"不做什么"，比宣称"什么都能做"更可信、更易集成。**
4. **确定性兜底无处不在**：文本层优先、XY-cut 兜底、双读择优、LM 仅在低置信时介入且不覆盖模型概率、方向判断"宁可不转"。每一步都留了"安全网"，把最坏情况控制住。
5. **为批量而设计**：不是优化单页，而是把"多进程、每核一 worker、内存上限、尊重 cgroup"作为一等公民。这契合"海量文档"的真实使用形态。

---

## 8. 快速结论

- **一句话**：Indra OCR 是 Datalab 的低成本、纯 CPU、高吞吐文档转 Markdown 引擎，是"Commodity OCR 2.0"的代表——给小模型配上重后处理，输出结构化结果。
- **解决什么**：解决"传统 OCR 没版面、重型流水线太慢、VLM 太贵太慢"三者之间的空档，负责海量、廉价、够用的结构化文档解析。
- **怎么解决**：LineNet 检测 → CTC 多脚本识别（CenterMask + n-gram LM 重读）→ OrderNet 版面/阅读顺序（XY-cut 兜底）→ 规则化 Markdown 渲染；数字 PDF 走文本层快通道，多进程吃满 CPU。
- **何时用**：RAG 入库、批量票据/档案、多语言文档、真实扫描、无 GPU/重隐私的离线场景。
- **何时别用**：复杂表格、重度手写、退化老扫描、以公式为主或需最高精度的场景（用 Chandra）。

> 参考基准（README 自述，同一机器同页数）：OmniParseBench headline Indra 66.7（`--math` 71.4）领先 PaddleOCR PP-StructureV3 的 58.4；32 核吞吐约 44 页/秒，约为 Tesseract 的 15 倍、PaddleOCR PP-OCR 文本模型的 10 倍。具体数字会随 CPU 与页面而变，引用时以官方 README 为准。
