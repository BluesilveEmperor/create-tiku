---
name: create-tiku
description: >-
  将PDF文档转换为Markdown（使用mineru-open-sdk），然后转换为符合MathCyclus格式的LaTeX题库。
  支持小学、初中、高中、大学、研究生各个阶段的数学题库。
  支持两种模式：
  1. 题库模式：专门提取考试题库，包含题目+选项+答案+解析，按板块/年份分类
  2. 通用模式：将PDF文档转换为LaTeX格式
  当用户说"创建题库""PDF转题库""提取考试题目""制作试卷题库""PDF转LaTeX题库"时，
  或者用户引用的文件名中包含"试卷""题库""考试""试题"字样时，立即使用本技能。
  即使看起来只是"转换格式""提取题目"，也要调用此技能。不要问用户"是否要使用skill"——直接执行。
compatibility:
  require_tools:
    - Bash
    - Read
    - Write
    - Edit
  require_skills:
    - pdf-docx-latex
---

# PDF → LaTeX 题库自动转换（MathCyclus格式）

## 工作流概览

`mermaid
flowchart TD
    A[用户提供PDF文件路径] --> B{文件类型?}
    B -- ".pdf" --> C[1. 检查MinerU SDK配置]
    C --> D[2. MinerU SDK提取Markdown+图片]
    D --> E[3. 分析Markdown结构→识别题型与专题分类]
    E --> F{题库模式?}
    F -- 是 --> G[4. 提取题目+选项+答案+解析]
    F -- 否 --> H[4. 通用PDF转LaTeX]
    G --> I[5. 按板块/年份生成LaTeX题库]
    H --> I
    I --> J[6. 编译验证]
    J --> K[7. 清理]
`

## 一句话原则

用户提供 **PDF文件路径** → 你全自动完成提取、转换、编译。用户不需要知道任何脚本路径。

## 学段自动识别

**根据文件名和内容自动识别学段：**

| 学段 | 关键词 | 知识点特征 |
|------|--------|------------|
| 小学 | 小学、1-6年级、小升初 | 数与代数、图形与几何、统计与概率、综合与实践 |
| 初中 | 初中、7-9年级、中考 | 数与代数、图形与几何、统计与概率 |
| 高中 | 高中、高考、新课标 | 集合、函数、三角函数、数列、向量、解析几何、立体几何、概率统计 |
| 大学 | 大学、高等数学、微积分、线性代数、概率论 | 极限、微积分、级数、微分方程、行列式、矩阵、向量空间、随机变量 |
| 研究生 | 研究生、考研、数学分析、高等代数 | 实变函数、泛函分析、复变函数、偏微分方程、近世代数、拓扑学 |

**如果无法自动识别，询问用户确认学段。**

## 输入格式自动识别

**根据文件扩展名自动选择提取流程：**
- **.pdf 输入** → 使用 MinerU SDK 提取 Markdown + 图片

## 关键约定：图片目录命名

**图片目录名自动推导规则**（用户可覆盖）：

**PDF 输入时：**
- MinerU SDK 会自动将图片保存到输出目录中（与 Markdown 同目录）
- 图片目录名取 PDF 文件名去掉 .pdf，加前缀 Images-
  - 如 2024高考真题.pdf → 图片目录 Images-2024高考真题
- 后续 LaTeX 中的 \graphicspath 应指向该目录

用户可显式指定目录名覆盖上述规则。

**在后续所有步骤中，用 {图片目录} 代表推导出的目录名。**

## 零配置检测

`ash
MINERU_CONFIG=~/.mineru/config.yaml
if [ ! -f "" ]; then
    echo "ERROR: MinerU SDK 未配置"
    echo "请创建 ~/.mineru/config.yaml，内容为："
    echo "  token: '你的API密钥'"
    echo "如果还没有密钥，请前往 https://mineru.net/apiManage/token 注册获取。"
    exit 1
fi
`

## 标准工作流（按顺序执行）

### 1. 读取文件 → 看结构

**PDF 输入（MinerU SDK 方案）：**

**⚠️ 先检查 MinerU SDK 配置：**

每次使用 PDF 输入前必须检查 ~/.mineru/config.yaml 是否存在且包含有效的 	oken。

`python
import yaml
from pathlib import Path

config_path = Path.home() / ".mineru" / "config.yaml"
if config_path.exists():
    config = yaml.safe_load(config_path.read_text(encoding="utf-8"))
    token = config.get("token", "")
    if token:
        print("[OK] MinerU SDK token 有效 (len=%d)" % len(token))
    else:
        token = None
        print("[WARN] token 为空")
else:
    token = None
    print("[WARN] ~/.mineru/config.yaml 不存在")
`

**如果未配置：**

主动引导用户完成配置，**不要直接报错退出**。用友好语气告知：

> "使用 MinerU SDK 解析 PDF 需要配置 API Token。
>
> 请执行以下步骤：
>
> 1. 确保已安装 mineru-open-sdk：
>    `ash
>    pip install mineru-open-sdk
>    `
>
> 2. 创建配置文件：
>    `ash
>    mkdir -p ~/.mineru
>    `
>
> 3. 编辑 ~/.mineru/config.yaml，写入：
>    `yaml
>    token: '你的API密钥'
>    `
>
> 4. 如果还没有密钥，请前往 https://mineru.net/apiManage/token 注册获取。
>
> 配置完成后重新运行即可。"

配置好后，继续执行转换。

**使用 MinerU SDK 提取 PDF：**

`ash
# 使用 create-tiku skill 的配套脚本
python "/scripts/create_tiku_extract.py" \
  "<pdf路径>" \
  --output-dir ./tiku-output \
  --language ch
`

> **注意**：中文试卷使用 --language ch，英文试卷用 --language en。

脚本会在 ./tiku-output/ 目录生成：
- <文件名>.md — 最终的 Markdown 文件（核心产物）
- 图片文件自动保存在同目录或子目录中

**⚠️ Windows 编码兼容**：如遇 UnicodeEncodeError: 'gbk' 错误，加前缀：
`ash
PYTHONIOENCODING=utf-8 python "/scripts/create_tiku_extract.py" ...
`

**脚本参数说明：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| --output-dir | ./tiku-output | 输出目录 |
| --model | lm | 模型版本：pipeline / lm / html |
| --ocr | 关闭 | 对扫描件启用 OCR |
| --language | ch | 文档语言（中文用 ch，英文用 en） |
| --no-formula | 开启公式识别 | 禁用公式识别 |
| --no-table | 开启表格识别 | 禁用表格识别 |
| --pages | 全部 | 页码范围，如 "1-10,15" |

**MinerU API 限制：**
- 单文件 ≤ 200MB，≤ 600 页
- 免费版 API 有调用频率限制，如遇限速请稍后重试
- 超出日限额（2000页）后解析优先级会降低，但仍可继续使用

### 2. 提取图片

**PDF 输入（MinerU SDK 自动提取图片）：**

PDF 输入时，MinerU SDK 在第 1 步已完成 Markdown + 图片的提取。图片已保存在输出目录中，**无需额外解包操作**。

只需确认图片位置并设置 {图片目录}：

`ash
# 检查 MinerU 输出目录中的图片
ls ./tiku-output/  # 确认 <文件名>.md 和图片文件存在

# 如果图片在子目录中（MinerU 有时将图片放入与 md 同名的子目录）
ls ./tiku-output/<文件名>/  # 检查子目录

# 设置图片目录变量
# 如果图片直接在 tiku-output/ 下 → {图片目录} = tiku-output
# 如果图片在 tiku-output/<文件名>/ 下 → {图片目录} = tiku-output/<文件名>
# 如果图片需要移到 pdf 所在目录 → 执行移动：
cp -r ./tiku-output/<图片子目录> "<pdf所在目录>/{图片目录}/"
`

**PDF 输入的图片目录推导：**
- 取 PDF 文件名去掉 .pdf，加前缀 Images-
- 如 2024高考真题.pdf → 图片目录 Images-2024高考真题

**后续步骤中 LaTeX 的 \graphicspath 应指向推导出的 {图片目录}。**

### 3. 分析Markdown结构 → 识别题型与专题分类

**优先从目录结构获取专题分类：**

检测文件所在目录层级，智能判断专题分类：

| 目录层级 | 判断规则 | 示例 | 提取结果 |
|---------|---------|------|---------|
| 第1层 | 学段关键词 | 小学/初中/高中/大学/研究生 | 学段 |
| 第2层 | 知识板块 | 数与代数/图形与几何/统计与概率/综合与实践 | 知识板块 |
| 第3层及以下 | 具体专题 | 整数/小数/方程/平面图形等 | 具体专题 |

**目录智能判断逻辑：**
```python
if 目录名 in ["小学", "初中", "高中", "大学", "研究生"]:
    → 学段分类
elif 目录名 in ["数与代数", "图形与几何", "统计与概率", "综合与实践"]:
    → 知识板块
else:
    → 具体专题（如"整数"、"小数"、"方程"等）
```

**无目录信息时自动识别：**
如果文件没有目录信息（直接指定文件路径），则通过以下方式自动识别：

1. **从文件名推断**：文件名包含"小学"、"高考"、"考研"等关键词
2. **从内容分析**：分析 Markdown 中的标题、知识点关键词
3. **从题目类型推断**：根据题目内容判断所属专题

**读取 Markdown 识别题型：**
- 标题、题型（单选/多选/填空/解答）
- 题目编号、选项
- 图片标记（![...](images/xxx.png)）
- 答案、解析、详解标记

**题型识别规则：**
- **选择题**：通常有 A、B、C、D 选项
- **多选题**：可能有多个正确选项
- **填空题**：有空格或下划线
- **解答题**：需要详细解答过程

**输出结构：**
根据目录智能判断结果，生成对应的 LaTeX 文件结构：
```
chapters/{学段}/{知识板块}/{具体专题}/content_{学段}_{知识板块}_{具体专题}.tex
```

示例：
- 目录 `题库/chapters/小学/数与代数/整数/` → 生成 `chapters/小学/数与代数/整数/content_小学_数与代数_整数.tex`
- 目录 `题库/chapters/小学/图形与几何/平面图形/` → 生成 `chapters/小学/图形与几何/平面图形/content_小学_图形与几何_平面图形.tex`

### 4. 提取题目+选项+答案+解析

**从 Markdown 中提取：**
- 题目文本
- 选项（A、B、C、D）
- 答案（如有）
- 解析（如有）
- 详解（如有）

**LaTeX 公式核心规则：**
- 行内 $...$，行间 \[...\]
- 分数用 \frac{}{}，太长用 \dfrac{}{}（注意 Overfull 时换回 \frac）
- 三角：\sin \cos \tan，对数：\ln \lg
- 分段函数：\begin{cases} ... \end{cases}
- 集合：\{ \} 转义，\mid 表示"使得"
- 向量：\vec{a} 或 \mathbf{a}
- 圆周率：\pi，自然底数：\mathrm{e}，虚数：\mathrm{i}

**选项和编号规则：**
- 选择题选项 → 用 	asks 环境。内容长时 2 列，简短时 4 列
- 大题多问 → 用 examenum 或嵌套 enumerate
- 填空题空位 → 用 \blank（如果模板定义）或 \underline{\hspace{2cm}}

### 5. 按板块/年份生成LaTeX题库

**按照 MathCyclus 格式生成题库：**

**目录结构（多学段）：**
```
chapters/
├── 小学/
│   ├── 数与代数/
│   │   ├── 整数/
│   │   │   └── 小-2024-G-某小学-5-整数.tex
│   │   ├── 分数/
│   │   │   └── 小-2024-G-某小学-3-分数.tex
│   │   └── content_小学_数与代数.tex
│   ├── 图形与几何/
│   │   ├── 平面图形/
│   │   │   └── 小-2024-G-某小学-8-平面图形.tex
│   │   └── content_小学_图形与几何.tex
│   └── 统计与概率/
│       └── content_小学_统计与概率.tex
├── 初中/
│   ├── 数与代数/
│   │   ├── 有理数/
│   │   │   └── 初-2024-G-中考-5-有理数.tex
│   │   ├── 函数/
│   │   │   └── 初-2024-G-中考-12-函数.tex
│   │   └── content_初中_数与代数.tex
│   ├── 图形与几何/
│   │   ├── 三角形/
│   │   │   └── 初-2024-G-中考-8-三角形.tex
│   │   └── content_初中_图形与几何.tex
│   └── 统计与概率/
│       └── content_初中_统计与概率.tex
├── 高中/
│   ├── 集合/
│   │   ├── 2005/
│   │   │   └── 高-2005-G-全国卷II-9-集合.tex
│   │   ├── 2024/
│   │   │   └── 高-2024-G-全国甲卷（理）-2-集合.tex
│   │   └── content_高中_集合.tex
│   ├── 函数/
│   │   ├── 2024/
│   │   │   └── 高-2024-G-上海卷-21-函数.tex
│   │   └── content_高中_函数.tex
│   └── ...
├── 大学/
│   ├── 高等数学/
│   │   ├── 极限与连续/
│   │   │   └── 大-2024-G-期末考试-1-极限.tex
│   │   ├── 一元微积分/
│   │   │   └── 大-2024-G-期末考试-3-导数.tex
│   │   └── content_大学_高等数学.tex
│   ├── 线性代数/
│   │   ├── 行列式/
│   │   │   └── 大-2024-G-期末考试-2-行列式.tex
│   │   └── content_大学_线性代数.tex
│   └── 概率论与数理统计/
│       └── content_大学_概率论与数理统计.tex
└── 研究生/
    ├── 基础数学/
    │   ├── 实变函数/
    │   │   └── 研-2024-G-期末考试-1-实变函数.tex
    │   └── content_研究生_基础数学.tex
    ├── 应用数学/
    │   ├── 数值分析/
    │   │   └── 研-2024-G-期末考试-2-数值分析.tex
    │   └── content_研究生_应用数学.tex
    └── 概率论与数理统计/
        └── content_研究生_概率论与数理统计.tex
```

**题目文件格式（MathCyclus标准）：**

`latex
% === Begin Label Data ===
% ID: 344
% 难度星级: 
% 标签: 
% 备注: 
% 组卷引用次数: 0
% === End  Label Data ===

\begin{problem}{年份}{类型}{试卷名}{题号}{板块}
题目文本

\begin{choices}
\choice{{选项A}}
\choice{{选项B}}
\choice{{选项C}}
\choice{{选项D}}
\end{choices}
\end{problem}

\begin{answer}
答案内容
\end{answer}

\begin{solutions}
解析内容
\end{solutions}
`

**题目文件命名规范（多学段）：**
- 格式：学段-年份-类型-试卷名-题号-板块.tex
- 示例：
  - 小学：小-2024-G-某小学-5-整数.tex
  - 初中：初-2024-G-中考-5-有理数.tex
  - 高中：高-2024-G-全国甲卷（理）-2-集合.tex
  - 大学：大-2024-G-期末考试-1-极限.tex
  - 研究生：研-2024-G-期末考试-1-实变函数.tex
- 类型：G=高考/期末考试，M=模拟，T=其他
- 学段：小=小学，初=初中，高=高中，大=大学，研=研究生

**板块分类（多学段数学）：**

### 小学数学（4大领域）
**数与代数：** 整数、小数、分数、负数初步、运算律、简易方程、常见的量（时间、人民币、质量、长度）
**图形与几何：** 图形的认识、测量（周长、面积、体积）、图形的运动（平移、旋转、轴对称）、图形与位置（方向、位置、坐标）
**统计与概率：** 数据收集与整理、统计图表、平均数/中位数/众数、简单概率
**综合与实践：** 主题活动、实践应用

### 初中数学（3大板块）
**数与代数：** 有理数、实数、代数式、整式、分式、二次根式、一元一次方程、二元一次方程组、一元二次方程、不等式与不等式组、一次函数、二次函数、反比例函数
**图形与几何：** 三角形、四边形、圆、全等三角形、相似三角形、图形的平移/旋转/轴对称、平面直角坐标系
**统计与概率：** 数据的收集整理与描述、数据分析（平均数、中位数、众数、方差）、随机事件的概率

### 高中数学（8大板块）
**集合：** 集合的概念与运算、常用逻辑用语
**函数：** 函数概念与基本初等函数（指、对、幂函数）、函数的性质（单调性、奇偶性、周期性）
**三角函数：** 三角函数的图象与性质、三角恒等变换、解三角形（正弦定理、余弦定理）
**数列：** 等差数列、等比数列、数列求和、数列的应用
**向量：** 平面向量、空间向量与立体几何
**解析几何：** 直线与圆的方程、圆锥曲线（椭圆、双曲线、抛物线）
**立体几何：** 空间直线与平面、简单几何体（棱柱、棱锥、球）
**概率统计：** 概率（古典概型、几何概型）、统计、排列组合、二项式定理

### 大学数学（3大课程）
**高等数学/微积分：** 函数与极限、导数与微分、微分中值定理与导数应用、不定积分、定积分、定积分应用、多元函数微分法、重积分、曲线积分与曲面积分、无穷级数、微分方程
**线性代数：** 行列式、矩阵、向量、线性方程组、特征值与特征向量、二次型
**概率论与数理统计：** 随机事件和概率、随机变量及其概率分布、二维随机变量、随机变量的数字特征、大数定律和中心极限定理、数理统计与参数估计

### 研究生数学（3大方向）
**基础数学：** 实变函数、泛函分析、复变函数、偏微分方程、近世代数、拓扑学
**应用数学：** 数值分析、运筹学、控制论
**概率论与数理统计：** 随机过程、时间序列分析、多元统计分析

**content_板块.tex 文件格式：**

`latex
\section{2005}
\input{chapters/集合/2005/2005-G-全国卷II-9-集合}
\section{2024}
\input{chapters/集合/2024/2024-G-全国甲卷（理）-2-集合}
\input{chapters/集合/2024/2024-G-新课标I卷-1-集合}
`

**MathCyclus类文件（MathCyclus_book.cls）：**

如果需要使用完整的 MathCyclus 功能，需要复制 MathCyclus_book.cls 文件到输出目录。

**简化版模板（不使用类文件）：**

`latex
\documentclass[12pt, a4paper, oneside]{ctexart}

% ── 1. 数学与链接 ──
\usepackage{amsmath, amsthm, amssymb}
\usepackage[bookmarks=true, colorlinks, citecolor=blue, linkcolor=black]{hyperref}

% ── 2. 字体方案 ──
\usepackage{newtxmath}

% ── 3. 页面布局 ──
\usepackage[a4paper, margin=2.5cm, footskip=1cm]{geometry}

% ── 4. 页眉页脚 ──
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\fancyhf{}
\fancyfoot[C]{题库第\thepage 页（共\pageref{LastPage}页）}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{0pt}

% ── 5. 表格与插图 ──
\usepackage{graphicx}
\usepackage{adjustbox}
\usepackage{diagbox}
\usepackage{makecell}
\usepackage{caption}
\usepackage{float}
\usepackage{tikz}

% ── 6. 选择题选项（tasks 环境） ──
\usepackage{tasks}
\settasks{
    label       = \Alph*.,
    label-width = 1.8em,
    item-indent = 2.4em,
    label-offset = 0.5em,
    column-sep  = 2em,
    before-skip = 0pt,
    after-skip  = 0pt
}

% ── 7. 大题编号（enumitem 体系） ──
\usepackage[shortlabels]{enumitem}
\newlist{examenum}{enumerate}{3}
\setlist[examenum,1]{label=\arabic*., leftmargin=2em, itemsep=0.3em, parsep=0em}
\setlist[examenum,2]{label=(\arabic*), leftmargin=1.5em, itemsep=0.1em, parsep=0em}
\setlist[examenum,3]{label=(\roman*), leftmargin=1.5em, itemsep=0.1em, parsep=0em}

% ── 8. 自定义命令 ──

% 圈号数字
\newcommand{\mycircled}[1]{%
  \tikz[baseline=(char.base),outer sep=0pt]{%
    \node[draw,circle,inner sep=0.5pt,minimum size=1.4em,
          line width=0.4pt,font=\zihao{-5}] (char) {#1};%
  }%
}

% 旋转平行符号
\newcommand{\Parallel}{\raisebox{0.1ex}{\rotatebox[origin=c]{-20}{$\parallel$}}}

% 下划线填空
\newcommand{\blank}{\underline{\hspace{2cm}}}

% 答案命令
\newcommand{\answer}[1]{\textbf{答案：}#1}

% 解析命令
\newcommand{\analysis}[1]{\textbf{解析：}#1}

% 详解命令
\newcommand{\explanation}[1]{\textbf{详解：}#1}

% ── 9. 大标题辅助命令 ──
\newcommand{\subjecttitle}[1]{{\fontsize{24pt}{22pt}\selectfont\centering\textbf{#1}\par}}
\newcommand{\subtitle}[1]{{\fontsize{16pt}{16pt}\selectfont\centering #1\par}}

% ============================================================
%  正文开始
% ============================================================
\begin{document}

% ── 题库标题区 ──
\begin{center}
    \LARGE{\textbf{考试题库}}
\end{center}

\vspace{0.5em}

% ── 注意事项 ──
\noindent\textbf{注意事项}：

1．本题库包含选择题、填空题、解答题三种题型。

2．选择题和填空题答案直接标注在题目后，解答题提供详细解析。

3．使用本题库时，请先独立完成题目，再对照答案和解析。

\vspace{1em}

% ══════════════════════════════════════════
%  选择题部分
% ══════════════════════════════════════════

\noindent\textbf{选择题：本题共8小题，每小题5分，共40分。在每小题给出的四个选项中，
只有一项是符合题目要求的。}

\begin{enumerate}[itemsep=0.3em]
    \item 题目示例
    \begin{tasks}(4)
        \task 选项A
        \task 选项B
        \task 选项C
        \task 选项D
    \end{tasks}
    \answer{A}
    \analysis{解析内容}
\end{enumerate}

% ══════════════════════════════════════════
%  填空题部分
% ══════════════════════════════════════════

\noindent\textbf{填空题：本题共3小题，每小题5分，共15分。}

\begin{enumerate}[start=9, itemsep=0.8em]
    \item 填空题示例 \blank.
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}

% ══════════════════════════════════════════
%  解答题部分
% ══════════════════════════════════════════

\noindent\textbf{解答题：本题共5小题，共77分。解答应写出文字说明、证明过程或演算步骤。}

\begin{examenum}
    \item （13分）解答题示例
    \begin{examenum}
        \item 第1问
        \item 第2问
    \end{examenum}
    \answer{答案内容}
    \analysis{解析内容}
    \explanation{详解内容}
\end{examenum}

\end{document}
`

### 6. 编译验证

**编译 LaTeX 文件：**
`ash
cd "<pdf所在目录>"
xelatex -interaction=nonstopmode "<输出文件名>.tex"
xelatex -interaction=nonstopmode "<输出文件名>.tex"
grep -E "Overfull|Error" "<输出文件名>.log" | grep -v "infwarerr"
`

**验证要点：**
- 题目编号正确
- 选项显示正常
- 答案、解析显示正确
- 图片显示正常
- 公式渲染正确

### 7. 清理
`ash
rm -f "<输出文件名>.aux" "<输出文件名>.log" "<输出文件名>.out"
`

保留：
- .tex 源文件
- .pdf 输出文件
- 图片目录

## 常见问题快速修复

| 症状 | 原因 | 修复 |
|------|------|------|
| Overfull \hbox | 公式或选项超宽 | 选项改 2 列；\dfrac 改 \frac；缩短公式 |
| Undefined control sequence | 缺少宏包 | 在导言区添加 \usepackage{...} |
| 图片错位/消失 | 列表环境中用了 wrapfigure | 改用 minipage 左右并排方案 |
| 中文不显示 | 非 ctex 模板 | 确认模板用 ctexart 或添加 \usepackage{ctex} |
| 编译有 Missing $ | 花括号不匹配或中文在公式外 | 检查 $...$ 配对 |
| MinerU SDK 解析失败（PDF 输入） | Token 无效/网络问题/文件过大 | 检查 ~/.mineru/config.yaml；文件 ≤ 200MB/600 页；扫描件加 --ocr |
| MinerU 输出公式乱码（PDF 输入） | 公式识别质量不佳 | 尝试 --model vlm（默认）；扫描件加 --ocr；对个别公式手动修正 |
| PDF 提取后图片路径不对 | MinerU 图片目录与 LaTeX \graphicspath 不匹配 | 确认 {图片目录} 推导正确，\graphicspath 指向实际图片位置 |
| UnicodeEncodeError: 'gbk'（Windows） | Windows 终端编码为 GBK | 加 PYTHONIOENCODING=utf-8 前缀执行脚本 |

## 用户调用示例

用户只需要说：
> 帮我把 2024高考真题.pdf 转换成题库

或者：
> 创建题库

**答案文档自动识别：**
如果输入文件名包含"答案"（如 2024高考真题-答案.pdf），技能会自动选用答案模板，
无需手动指定。处理规则也自动切换为答案排版模式（保留【答案】【解析】【详解】结构）。

**PDF 输入时**：使用 MinerU SDK 自动提取 Markdown + 图片，图片目录自动命名为 Images- + PDF文件名（如 2024高考真题.pdf → Images-2024高考真题/），公式已为 LaTeX 格式可直接引用。

如果你能获取到文件路径就直接执行，否则问用户文件在哪里。

---

## 嵌入式模板库

### 多学段MathCyclus题库模板

当用户需要创建题库时，根据学段自动选择对应模板。默认使用高中模板，用户可指定学段切换。

#### 小学数学题库模板
适用于小学1-6年级数学题库，特点：
- 使用14pt字号，适合小学生阅读
- 选择题使用简单选项（A、B、C、D）
- 填空题使用下划线填空
- 解答题提供详细解析
- 公式简化，适合小学生理解

```latex
\documentclass[14pt, a4paper, oneside]{ctexart}
\usepackage{amsmath, amsthm, amssymb}
\usepackage[bookmarks=true, colorlinks, citecolor=blue, linkcolor=black]{hyperref}
\usepackage{newtxmath}
\usepackage[a4paper, margin=2.5cm, footskip=1cm]{geometry}
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\fancyhf{}
\fancyfoot[C]{小学数学题库第\thepage 页（共\pageref{LastPage}页）}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{0pt}
\usepackage{graphicx}
\usepackage{caption}
\usepackage{float}
\usepackage{tikz}
\usepackage{tasks}
\settasks{
    label = \Alph*., label-width = 1.8em, item-indent = 2.4em, column-sep = 2em
}
\usepackage[shortlabels]{enumitem}
\newcommand{\blank}{\underline{\hspace{2cm}}}
\newcommand{\answer}[1]{\textbf{答案：}#1}
\newcommand{\analysis}[1]{\textbf{解析：}#1}
\begin{document}
\begin{center}
    \LARGE{\textbf{小学数学题库}}
\end{center}
\vspace{1em}
\noindent\textbf{一、选择题}（共10题，每题3分，共30分）
\begin{enumerate}[itemsep=0.3em]
    \item 题目示例
    \begin{tasks}(4)
        \task 选项A
        \task 选项B
        \task 选项C
        \task 选项D
    \end{tasks}
    \answer{A}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{二、填空题}（共5题，每题3分，共15分）
\begin{enumerate}[itemsep=0.8em]
    \item 填空题示例 \blank.
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{三、解答题}（共5题，共55分）
\begin{enumerate}[itemsep=0.8em]
    \item 解答题示例
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}
\end{document}
```

#### 初中数学题库模板
适用于初中7-9年级数学题库，特点：
- 使用12pt字号
- 支持有理数、实数、代数式、方程、函数等初中内容
- 选择题使用A、B、C、D选项
- 填空题使用下划线填空
- 解答题提供详细解析
- 包含一次函数、二次函数、反比例函数等初中内容

```latex
\documentclass[12pt, a4paper, oneside]{ctexart}
\usepackage{amsmath, amsthm, amssymb}
\usepackage[bookmarks=true, colorlinks, citecolor=blue, linkcolor=black]{hyperref}
\usepackage{newtxmath}
\usepackage[a4paper, margin=2.5cm, footskip=1cm]{geometry}
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\fancyhf{}
\fancyfoot[C]{初中数学题库第\thepage 页（共\pageref{LastPage}页）}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{0pt}
\usepackage{graphicx}
\usepackage{caption}
\usepackage{float}
\usepackage{tikz}
\usepackage{tasks}
\settasks{
    label = \Alph*., label-width = 1.8em, item-indent = 2.4em, column-sep = 2em
}
\usepackage[shortlabels]{enumitem}
\newcommand{\blank}{\underline{\hspace{2cm}}}
\newcommand{\answer}[1]{\textbf{答案：}#1}
\newcommand{\analysis}[1]{\textbf{解析：}#1}
\newcommand{\explanation}[1]{\textbf{详解：}#1}
\begin{document}
\begin{center}
    \LARGE{\textbf{初中数学题库}}
\end{center}
\vspace{1em}
\noindent\textbf{一、选择题}（共10题，每题3分，共30分）
\begin{enumerate}[itemsep=0.3em]
    \item 题目示例
    \begin{tasks}(4)
        \task 选项A
        \task 选项B
        \task 选项C
        \task 选项D
    \end{tasks}
    \answer{A}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{二、填空题}（共6题，每题3分，共18分）
\begin{enumerate}[itemsep=0.8em]
    \item 填空题示例 \blank.
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{三、解答题}（共7题，共72分）
\begin{examenum}
    \item 解答题示例
    \begin{examenum}
        \item 第1问
        \item 第2问
    \end{examenum}
    \answer{答案内容}
    \analysis{解析内容}
    \explanation{详解内容}
\end{examenum}
\end{document}
```

#### 高中数学题库模板（默认模板）
适用于高中数学题库，特点：
- 使用12pt字号
- 支持集合、函数、三角函数、数列、向量、解析几何、立体几何、概率统计
- 选择题使用A、B、C、D选项
- 填空题使用下划线填空
- 解答题提供详细解析和详解
- 包含高考题型格式

```latex
% 当前使用的MathCyclus题库模板（已在上方完整展示）
```

#### 大学数学题库模板
适用于大学高等数学、线性代数、概率论与数理统计题库，特点：
- 使用12pt字号
- 支持极限、微积分、级数、微分方程、行列式、矩阵、向量空间、随机变量等大学内容
- 选择题使用A、B、C、D选项
- 填空题使用下划线填空
- 解答题提供详细解析和详解
- 包含大学数学课程内容

```latex
\documentclass[12pt, a4paper, oneside]{ctexart}
\usepackage{amsmath, amsthm, amssymb}
\usepackage[bookmarks=true, colorlinks, citecolor=blue, linkcolor=black]{hyperref}
\usepackage{newtxmath}
\usepackage[a4paper, margin=2.5cm, footskip=1cm]{geometry}
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\fancyhf{}
\fancyfoot[C]{大学数学题库第\thepage 页（共\pageref{LastPage}页）}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{0pt}
\usepackage{graphicx}
\usepackage{caption}
\usepackage{float}
\usepackage{tikz}
\usepackage{tasks}
\settasks{
    label = \Alph*., label-width = 1.8em, item-indent = 2.4em, column-sep = 2em
}
\usepackage[shortlabels]{enumitem}
\newcommand{\blank}{\underline{\hspace{2cm}}}
\newcommand{\answer}[1]{\textbf{答案：}#1}
\newcommand{\analysis}[1]{\textbf{解析：}#1}
\newcommand{\explanation}[1]{\textbf{详解：}#1}
\newcommand{\vec}[1]{\mathbf{#1}}
\newcommand{\mathrm}[1]{\mathop{\mathrm{#1}}\nolimits}
\begin{document}
\begin{center}
    \LARGE{\textbf{大学数学题库}}
\end{center}
\vspace{1em}
\noindent\textbf{一、选择题}（共10题，每题3分，共30分）
\begin{enumerate}[itemsep=0.3em]
    \item 题目示例
    \begin{tasks}(4)
        \task 选项A
        \task 选项B
        \task 选项C
        \task 选项D
    \end{tasks}
    \answer{A}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{二、填空题}（共6题，每题3分，共18分）
\begin{enumerate}[itemsep=0.8em]
    \item 填空题示例 \blank.
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{三、解答题}（共7题，共72分）
\begin{examenum}
    \item 解答题示例
    \begin{examenum}
        \item 第1问
        \item 第2问
    \end{examenum}
    \answer{答案内容}
    \analysis{解析内容}
    \explanation{详解内容}
\end{examenum}
\end{document}
```

#### 研究生数学题库模板
适用于研究生数学分析、高等代数、概率论与数理统计题库，特点：
- 使用12pt字号
- 支持实变函数、泛函分析、复变函数、偏微分方程、近世代数、拓扑学等研究生内容
- 选择题使用A、B、C、D选项
- 填空题使用下划线填空
- 解答题提供详细解析和详解
- 包含研究生数学课程内容

```latex
\documentclass[12pt, a4paper, oneside]{ctexart}
\usepackage{amsmath, amsthm, amssymb}
\usepackage[bookmarks=true, colorlinks, citecolor=blue, linkcolor=black]{hyperref}
\usepackage{newtxmath}
\usepackage[a4paper, margin=2.5cm, footskip=1cm]{geometry}
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\fancyhf{}
\fancyfoot[C]{研究生数学题库第\thepage 页（共\pageref{LastPage}页）}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{0pt}
\usepackage{graphicx}
\usepackage{caption}
\usepackage{float}
\usepackage{tikz}
\usepackage{tasks}
\settasks{
    label = \Alph*., label-width = 1.8em, item-indent = 2.4em, column-sep = 2em
}
\usepackage[shortlabels]{enumitem}
\newcommand{\blank}{\underline{\hspace{2cm}}}
\newcommand{\answer}[1]{\textbf{答案：}#1}
\newcommand{\analysis}[1]{\textbf{解析：}#1}
\newcommand{\explanation}[1]{\textbf{详解：}#1}
\newcommand{\vec}[1]{\mathbf{#1}}
\newcommand{\mathrm}[1]{\mathop{\mathrm{#1}}\nolimits}
\newcommand{\norm}[1]{\left\| #1 \right\|}
\newcommand{\abs}[1]{\left| #1 \right|}
\begin{document}
\begin{center}
    \LARGE{\textbf{研究生数学题库}}
\end{center}
\vspace{1em}
\noindent\textbf{一、选择题}（共10题，每题3分，共30分）
\begin{enumerate}[itemsep=0.3em]
    \item 题目示例
    \begin{tasks}(4)
        \task 选项A
        \task 选项B
        \task 选项C
        \task 选项D
    \end{tasks}
    \answer{A}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{二、填空题}（共6题，每题3分，共18分）
\begin{enumerate}[itemsep=0.8em]
    \item 填空题示例 \blank.
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}
\vspace{1em}
\noindent\textbf{三、解答题}（共7题，共72分）
\begin{examenum}
    \item 解答题示例
    \begin{examenum}
        \item 第1问
        \item 第2问
    \end{examenum}
    \answer{答案内容}
    \analysis{解析内容}
    \explanation{详解内容}
\end{examenum}
\end{document}
```

`latex
% ============================================================
%  高中数学题库 LaTeX 模板（MathCyclus格式）
%  适用于各种考试题库的排版
%  说明：
%    - 采用 12pt 字号 + 2.5cm 边距
%    - 使用 ctexart 支持中文
%    - 支持选择题、填空题、解答题
%    - 包含答案和解析
%    - 符合 MathCyclus 题库管理系统格式
% ============================================================

\documentclass[12pt, a4paper, oneside]{ctexart}

% ── 1. 数学与链接 ──
\usepackage{amsmath, amsthm, amssymb}
\usepackage[bookmarks=true, colorlinks, citecolor=blue, linkcolor=black]{hyperref}

% ── 2. 字体方案 ──
\usepackage{newtxmath}

% ── 3. 页面布局 ──
\usepackage[a4paper, margin=2.5cm, footskip=1cm]{geometry}

% ── 4. 页眉页脚 ──
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\fancyhf{}
\fancyfoot[C]{题库第\thepage 页（共\pageref{LastPage}页）}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{0pt}

% ── 5. 表格与插图 ──
\usepackage{graphicx}
\usepackage{adjustbox}
\usepackage{diagbox}
\usepackage{makecell}
\usepackage{caption}
\usepackage{float}
\usepackage{tikz}

% ── 6. 选择题选项（tasks 环境） ──
\usepackage{tasks}
\settasks{
    label       = \Alph*.,
    label-width = 1.8em,
    item-indent = 2.4em,
    label-offset = 0.5em,
    column-sep  = 2em,
    before-skip = 0pt,
    after-skip  = 0pt
}

% ── 7. 大题编号（enumitem 体系） ──
\usepackage[shortlabels]{enumitem}
\newlist{examenum}{enumerate}{3}
\setlist[examenum,1]{label=\arabic*., leftmargin=2em, itemsep=0.3em, parsep=0em}
\setlist[examenum,2]{label=(\arabic*), leftmargin=1.5em, itemsep=0.1em, parsep=0em}
\setlist[examenum,3]{label=(\roman*), leftmargin=1.5em, itemsep=0.1em, parsep=0em}

% ── 8. 自定义命令 ──

% 圈号数字
\newcommand{\mycircled}[1]{%
  \tikz[baseline=(char.base),outer sep=0pt]{%
    \node[draw,circle,inner sep=0.5pt,minimum size=1.4em,
          line width=0.4pt,font=\zihao{-5}] (char) {#1};%
  }%
}

% 旋转平行符号
\newcommand{\Parallel}{\raisebox{0.1ex}{\rotatebox[origin=c]{-20}{$\parallel$}}}

% 下划线填空
\newcommand{\blank}{\underline{\hspace{2cm}}}

% 答案命令
\newcommand{\answer}[1]{\textbf{答案：}#1}

% 解析命令
\newcommand{\analysis}[1]{\textbf{解析：}#1}

% 详解命令
\newcommand{\explanation}[1]{\textbf{详解：}#1}

% ── 9. 大标题辅助命令 ──
\newcommand{\subjecttitle}[1]{{\fontsize{24pt}{22pt}\selectfont\centering\textbf{#1}\par}}
\newcommand{\subtitle}[1]{{\fontsize{16pt}{16pt}\selectfont\centering #1\par}}

% ============================================================
%  正文开始
% ============================================================
\begin{document}

% ── 题库标题区 ──
\begin{center}
    \LARGE{\textbf{考试题库}}
\end{center}

\vspace{0.5em}

% ── 注意事项 ──
\noindent\textbf{注意事项}：

1．本题库包含选择题、填空题、解答题三种题型。

2．选择题和填空题答案直接标注在题目后，解答题提供详细解析。

3．使用本题库时，请先独立完成题目，再对照答案和解析。

\vspace{1em}

% ══════════════════════════════════════════
%  选择题部分
% ══════════════════════════════════════════

\noindent\textbf{选择题：本题共8小题，每小题5分，共40分。在每小题给出的四个选项中，
只有一项是符合题目要求的。}

\begin{enumerate}[itemsep=0.3em]
    \item 题目示例
    \begin{tasks}(4)
        \task 选项A
        \task 选项B
        \task 选项C
        \task 选项D
    \end{tasks}
    \answer{A}
    \analysis{解析内容}
\end{enumerate}

% ══════════════════════════════════════════
%  填空题部分
% ══════════════════════════════════════════

\noindent\textbf{填空题：本题共3小题，每小题5分，共15分。}

\begin{enumerate}[start=9, itemsep=0.8em]
    \item 填空题示例 \blank.
    \answer{答案内容}
    \analysis{解析内容}
\end{enumerate}

% ══════════════════════════════════════════
%  解答题部分
% ══════════════════════════════════════════

\noindent\textbf{解答题：本题共5小题，共77分。解答应写出文字说明、证明过程或演算步骤。}

\begin{examenum}
    \item （13分）解答题示例
    \begin{examenum}
        \item 第1问
        \item 第2问
    \end{examenum}
    \answer{答案内容}
    \analysis{解析内容}
    \explanation{详解内容}
\end{examenum}

\end{document}
`

### MathCyclus单题模板

用于生成符合MathCyclus格式的单个题目文件：

`latex
% === Begin Label Data ===
% ID: 自动生成
% 难度星级: 
% 标签: 
% 备注: 
% 组卷引用次数: 0
% === End  Label Data ===

\begin{problem}{年份}{类型}{试卷名}{题号}{板块}
题目文本

\begin{choices}
\choice{{选项A}}
\choice{{选项B}}
\choice{{选项C}}
\choice{{选项D}}
\end{choices}
\end{problem}

\begin{answer}
答案内容
\end{answer}

\begin{solutions}
解析内容
\end{solutions}
`

## 评测标准

### 基础功能（必须满足）
1. **PDF转Markdown**：能正确调用MinerU SDK将PDF转换为Markdown
2. **Markdown分析**：能正确识别题型结构（选择题、填空题、解答题）
3. **LaTeX生成**：能生成正确的LaTeX代码，包含题目、选项、答案、解析
4. **编译验证**：生成的LaTeX能成功编译为PDF

### 高级功能（加分项）
1. **自动题型识别**：能自动识别选择题、多选题、填空题、解答题
2. **公式保留**：能正确保留数学公式（$...$ 格式）
3. **图片处理**：能正确处理图片路径和插入
4. **答案解析**：能正确提取和格式化答案、解析、详解
5. **MathCyclus格式**：能生成符合MathCyclus题库管理系统格式的题目文件
6. **板块分类**：能按数学板块（集合、函数、导数等）分类题目
7. **年份分类**：能按年份分类题目
8. **学段识别**：能自动识别小学、初中、高中、大学、研究生学段
9. **学段分类**：能按学段分类题目并选择对应模板

### 多学段评测指标
1. **小学数学**：能正确处理小学数学题库，使用简化模板，适合小学生阅读
2. **初中数学**：能正确处理初中数学题库，支持有理数、实数、方程、函数等内容
3. **高中数学**：能正确处理高中数学题库，支持集合、函数、三角函数、数列、向量、解析几何、立体几何、概率统计
4. **大学数学**：能正确处理大学数学题库，支持极限、微积分、级数、微分方程、行列式、矩阵、向量空间、随机变量
5. **研究生数学**：能正确处理研究生数学题库，支持实变函数、泛函分析、复变函数、偏微分方程、近世代数、拓扑学

### 质量指标
1. **题目完整性**：所有题目都被正确提取
2. **选项正确性**：选项内容完整且格式正确
3. **答案准确性**：答案与原文一致
4. **解析清晰度**：解析内容清晰易读
5. **LaTeX规范性**：生成的LaTeX符合规范，无语法错误
6. **格式兼容性**：生成的文件能被MathCyclus题库管理系统正确读取
7. **学段适用性**：生成的题库适合对应学段的学生使用
