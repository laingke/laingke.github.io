---
layout: post
title:  "每秒解析 GB 级别的 JSON 文件"
date:   2019-05-17 14:05:21 +0800
tags: JSON SIMD parsing translation
color: rgb(0,90,90)
cover: '../assets/images/cover/2019-06-20-simdjson-paper-cn-translation.png'
subtitle: 'A Simple Chinese Translation Copy of <i>Parsing Gigabytes of JSON per Second(v4)</i>'
// noToc: false
---
## **摘要**
JavaScript 对象表示法或者说 JSON 是 Web 上普遍存在的数据交换格式。由于数据量过大，摄取 JSON 可能会成为性能瓶颈。因此，我们希望尽可能快地进行 JSON 解析。

尽管 JSON 解析问题已经很成熟，但是我们仍然可以显著提高速度。我们介绍了第一个标准兼容的 JSON 解析器，它使用普通处理器在单个内核上每秒处理千兆字节的数据。我们可以使用比最先进的引用解析器（如 RapidJSON）少四分之一或更少的指令。与其它验证解析器不同，我们的软件（simdjson）广泛使用单指令多数据流（SIMD）指令。为了确保可重复性，simdjson 是在自由许可下作为开源软件免费提供的。

## **1. 简介**

JavaScript 对象表示法（JSON）是一种用于表示数据的文本格式[^4]。它通常用于Web上的浏览器-服务器通信。许多数据库系统（如 MySQL、PostgreSQL、IBM DB2、SQL Server、Oracle）和数据科学框架（如 Pandas）都支持它。许多面向文档的数据库都以 JSON 为中心，如 CouchDB 或 RethinkDB。

JSON 语法可以看作是 JavaScript 的一种受限形式，但它在许多编程语言中都被使用。JSON 有四个基本类型或原子（string、number、Boolean、null），可以嵌入到组合类型（数组和对象）中。对象采用大括号之间的一系列键值对的形式，其中键是字符串（例如，**```{"name": "Jack", "age": 22}```**）。数组是方括号之间逗号分隔值的列表（如 **```[1, "abc", null]```**）。组合类型可以包含基本类型或任意深度嵌套的组合类型作为值。如图 1 所示。JSON 规范定义了 6 个*结构字符*（**'```[```'**，**'```{```'**， **'```]```'**， **'```}```'**， **'```:```'**， **'```,```'**）：它们用于分隔对象和数组的位置和结构。

要从软件中访问 JSON 文档中包含的数据，通常将 JSON 文本转换为类似于树的逻辑表示形式，类似于图 1 的右侧，我们称之为 JSON 解析操作。我们将每个值、对象和数组引用为已解析树中的一个节点。解析之后，程序员可以依次访问每个节点并导航到它的兄弟节点或子节点，而不需要复杂且容易出错的字符串解析。

```json
{
    "Width": 800,
    "Height": 600,
    "Title": "View from my room",
    "Url": " http://ex.com/img.png",
    "Private": false,
    "Thumbnail": {
        "Url": "http://ex.com/th.png",
        "Height": 125,
        "Width": 100
    },
    "array": [
        116,
        943,
        234
    ],
    "Owner": null
}
```

解析大型 JSON 文档是一项常见的任务。Palkar 等人指出，大数据应用程序可以花费 80%（90%） 的时间解析 JSON 文档[^25]。Boncz 等人认为加速 JSON 解析是加速数据库处理的一个有趣的主题[^2]。

JSON 解析意味着错误检查：数组必须以方括号开始和结束，对象必须以大括号开始和结束，多个对象必须由逗号分隔的值对（用冒号分隔）组成，其中所有键都是字符串。数字必须符合规格，并符合一个有效的范围。除了字符串值外，只允许使用少量 ASCII 字符集。在字符串值中，必须转义几个字符（如 ASCII 行结束符）。JSON 规范要求文档使用 unicode 字符编码（UTF-8、UTF-16或UTF-32），默认为 UTF-8。因此，我们必须验证所有字符串的字符编码。因此，JSON 解析比仅仅定位节点更麻烦。我们的论点是，一个接受错误 JSON 的解析器是危险的——不管错误 JSON 是意外生成的还是恶意生成的，它都将默默地接受格式错误的 JSON ——而且没有得到很好的指定——很难预测或宏观定义格式错误 JSON 文件的语义应该是什么。

为了加快处理速度，我们应该尽可能有效地使用处理器。普通处理器（Intel、AMD、ARM、POWER）支持单指令多数据流（SIMD）指令。与常规指令不同，这些 SIMD 指令一次操作多个数据元素。例如，从 Haswell microarchitecture（2013）开始，Intel 和 AMD 处理器支持 AVX2 指令集和 256 位向量寄存器。因此，在最近的 x64 处理器上，我们可以在一条指令中比较两个 32 个字符的字符串。因此，使用 SIMD 指令只需很少的指令就可以定位重要的字符（例如，'"'，'='）。我们将 SIMD 指令的应用称为向量化。矢量化软件比传统软件使用的指令更少。在其它条件相同的情况下，生成更少指令的代码会更快。

与向量化密切相关的一个概念是无分支处理：每当处理器必须在两个代码路径（一个分支）之间进行选择时，就有可能由于当前流水线处理器上的一个错误预测的分支而招致几个周期的惩罚。根据我们的经验，SIMD 指令在无分支的环境中最有可能是有益的。

据我们所知，公开可用的 JSON 验证解析器很少使用 SIMD 指令。由于其复杂性，完整的 JSON 解析问题可能不会立即出现在矢量化中。

我们的核心结果之一是 SIMD 指令与最小分支相结合可以为 JSON 解析带来新的速度记录——通常在单个核心上每秒处理数 GB 的数据。 我们提出了一些具有普遍意义的特定绩效导向战略。

> 我们检测带引号的字符串，只使用算术和逻辑操作以及每个输入字节的固定数量的指令，同时省略转义的引号（&sect;3.1.1）。

> 我们使用向量化分类对码位值集进行区分，从而避免了进行 N 次比较来识别一个值是大小为 N 的集合的一部分的负担（&sect;3.1.2）。

> 我们仅使用 SIMD 指令验证 UTF-8 字符串（&sect;3.1.3）。

## **2. 相关工作**

在涉及的引用文献中加速 JSON 解析的一种常见策略是有选择地解析。Alagiannis 等人提出了 NoDB，一种无需在数据库中加载 JSON 数据就可以查询 JSON 数据的方法。它部分依赖于对输入的选择性解析。Bonetta 和 Brantner 使用投机的即时（JIT）编译和选择性数据访问来加速 JSON 处理[^3]。他们找到重复的常量结构，并针对这些结构生成代码。

Li 等人展示了他们的快速解析器，Mison 可以直接跳转到查询字段，而无需解析中间内容[^17]。Mison 使用 SIMD 指令快速识别一些结构字符，但在其它方面，通过在具有分支重循环的通用寄存器中处理位向量来工作。Mison 不尝试验证文档；它假设文档是纯 ASCII，而不是 unicode（UTF-8）。我们总结了 Mison 架构中最相关的组件如下:

1. 在第一步中，将输入文档与每个结构字符（'```[```', '```{```', '```]```', '```}```', '```:```', '```,```'）以及反斜杠（'```\```'）进行比较。每个比较都使用 SIMD 指令，每次比较 32 对字节。然后将比较转换为位图，其中位值 1 表示对应的结构字符的存在。当不需要数组时，Mison 省略了与数组相关的结构字符（'['，']'）。Mison 只在第一步中使用 SIMD 指令，这似乎也是唯一一个基本上没有分支的步骤。

2. 在第二步中，Mison 标识文档中每个字符串的起始点和结束点。它使用引号和反斜杠位图。对于每个引号字符，Mison 使用快速指令（```popcnt```）计算前反斜杠的数量：关闭前有奇数个反斜杠的引号并忽略它们。

3. 在第三步中, Mison标识由引号分隔的字符串对。它从第二步生成的位图中提取每个数据元素（例如，32 位）。它迭代地将双引号转换为字符串掩码（其中 1 位表示字符串的内容）；同时在每次迭代中使用少量算术和逻辑操作。

4. 在最后一个步骤中，Mison 使用字符串掩码关闭并忽略字符串中包含的结构字符（例如，```{```，```}```，```:```）。Mison 将所有左大括号存储在堆栈中。它从左边开始，每遇到一个新的右大括号弹出一层堆栈，从而找到匹配的括号对。对于每个可能的嵌套深度，可以通过部分复制输入冒号位图来构造一个位图，该位图指示冒号的位置。

从位图中提取的冒号位置开始，Mison 可以通过向后扫描来解析键，通过向前扫描来解析值。它只能选择给定深度的内容。实际上，冒号位图充当索引，选择性地解析输入文档。在某些情况下，Mison 可以在 3.5GHz 的 Intel 处理器上以超过 2GB/s 的速度扫描 JSON 文档，以获得高选择性查询。它比 RapidJSON 之类的传统验证解析器更快。

FishStore[29]解析 JSON 数据并选择感兴趣的子集，将结果存储在快速键-值存储[6]中。 虽然最初的FishStore依赖于Mison，但[开源版本](https://github.com/microsoft/FishStore)默认使用 simdjson 进行快速解析。

Pavlopoulou 等人[^26]提出了一个支持高级查询和重写规则的并行 JSON 处理器。它避免了首先加载数据的需要。

Sparser 快速过滤未处理的文档，主要查找相关信息[^25]，然后依赖于解析器。我们可以使用带有 Sparser 的 simdjson。

当只有一小部分数据感兴趣时，基于选择性解析的系统（如 Mison 或 Sparser）可能是有益的。但是，如果重复访问数据，则最好使用标准解析器将数据加载到数据库引擎中。像 Mison 这样的非验证解析器可能最适用于无法输入无效输入的紧密集成系统。

### 2.1 XML 解析

在 JSON 之前，已经有很多类似的 XML 解析工作。Noga 等人[^24]的报告提出，当需要解析的值少于 80% 时，只解析所需的值更经济。Marian 等人[^19]提出在执行查询之前将 XML 文档“投射”为更小的文档。Green 等人[^14]的表明，我们可以使用确定性有限自动机（DFA）在解析过程中延迟计算状态，从而快速解析 XML。Farfan 等人[^10]进一步使用内部物理指针跳过XML文档的整个部分。Takase 等人[^28]通过避免对以前遇到过的文本子集进行语法分析来加速 XML 解析。Kostoulas 等人设计了一种名为 Screamer 的快速验证 XML 解析器：它通过减少不同的处理步骤来达到更快的速度[^15]。Cameron 等人展示了在他们的解析器（称为 Parabix）中使用 SIMD 指令[^5]可以更快地解析 XML。Zhang 等人[^31]展示了如何通过先索引文档，然后分别解析文档的分区的方法并行地解析 XML 文档。

Mytkowicz 等人[^22]展示了如何使用 SIMD 指令对有限状态机进行向量化。它们显示了 HTML 标记化的良好结果，比基线的速度快两倍多。

### 2.2 CSV 解析

数据也以逗号分隔值（CSV）的形式出现。Muhlbauer 等人使用 SIMD 指令优化 CSV 解析和加载，以定位分隔符和无效字符[^20]。Ge 等人使用两遍方法，其中第一遍标识分隔符之间的区域，而第二遍处理记录[^12]。

## 3. 解析器架构和实现

在我们的经验中，大多数 JSON 解析器都是通过自上而下递归下降[^7]来进行处理的，它通过输入字节进行单次传递，进行逐字符解码。我们采用了不同的策略，使用两个不同的通道。我们简要地描述了这两个阶段，然后在后续章节中详细介绍它们。

1. 在第 1 阶段，我们验证字符编码并标识所有 JSON 节点的起始位置（例如，数字、字符串、null、true、false、数组、对象）。我们还需要 JSON 规范[^4]中定义的所有结构字符（'```[```'，'```{```'，'```]```'，'```}```'，'```:```'，'```,```'）的位置。这些位置被写成一个单独数组中的整数索引。  
在此阶段，有必要区分引号之间的字符，从而区分字符串值中的字符与其他字符。例如，尽管方括号出现，但 JSON 文档 ```"[1, 2]"``` 是单个字符串。也就是说，这些括号不应该被标识为相关的结构字符。因为引号可以转义（例如，'\n'），所以也有必要标识反斜杠字符。在字符串之外，只允许四个特定的空格字符（空格、制表符、换行符、回车符）。其它任何空白字符需要被标识。  
第一阶段涉及对字节进行 SIMD 处理或对位集（位数组）进行操作，其中一位对应于一个输入字节。因此，对于某些输入，它可能是无效的——我们可以观察数十个正在进行的操作，以发现在给定的输入块中实际上没有奇数的反斜杠或引号序列。然而，这种对此类输入的低效被这样一个事实所平衡：在复杂的结构化输入上运行该代码不再需要成本，并且替代方案通常涉及运行许多不可预测的分支。

2. 在第 2 阶段，我们处理所有节点和结构字符。我们根据节点的起始字符来区分它们。当遇到引号('```"```')时，我们解析一个字符串；当找到一个数字或连字符时，我们解析一个数字；当找到字母 '```t```'，'```f```'，'```n```' 时，我们寻找值 ```true```，```false``` 和 ```null```。  
JSON 中的字符串不能包含未转义的某些字符，即码点小于 0x20 的 ASCII 字符，它们可能包含许多类型的转义字符。因此，有必要对字符串进行*规范化*：将它们转换为有效的 UTF-8 序列。遇到的数字必须转换为整数或浮点值。它们可以有多种形式（例如，**```12```**, **```3.1416```**, **```1.2e+1```**）。然而，在解析数字时，我们必须检查许多规则。例如，以下字符串是无效的数字：**```012```**，**```1E+```**，**```.1```**。JSON 规范并不特定我们应该接受的数字范围：许多解析器不能表示 [-2<sup>53</sup>, 2<sup>53</sup>] 区间之外的整数值。相反，我们选择接受区间内的所有 64 位整数 [-2<sup>63</sup>, 2<sup>63</sup>)，同时拒绝该区间外的整数。我们还拒绝过大的浮点数（例如, 1e309 或 -1e309）。  
我们将对象验证为字符串、冒号（'```:```'）和值的序列；我们将数组验证为由逗号（'```,```'）分隔的值序列。我们确保所有以开括号（'```{```'）开头的对象都以闭括号（'```}```'）结束。我们确保所有以左方括号（'```[```'）开头的数组都以右方括号（'```]```'）结束。结果按 JSON 文档的顺序写在一个 64 位字数组的*磁带*上。磁带包含一个映射每个节点值的字（string、number、true、false、null）和映射每个对象或数组的开头和结尾的单词。为了确保快速导航，对磁带上与大括号或方括号对应的字进行了注释，这样我们就可以从对象或数组开头的字转到数组末尾的字，而无需读取数组或对象的内容。具体来说，磁带的构造如下：  

> ```null``` 原子表示为 64 位值（'```n```'\*2<sup>56</sup>），其中“n”对应字母“n”对应的 8 位代码点值(在ASCII中)。一个 ```true``` 原子由 '```t```'\*2<sup>56</sup> 表示，一个 ```false``` 原子由 '```f```'\*2<sup>56</sup> 表示。

> 数字用两个 64 位字表示。整型数由 '```l```'\*2<sup>56</sup> 拼接一个 64 位带符号整数（标准 2 的补码形式）的字表示。浮点数由 '```d```'\*2<sup>56</sup> 拼接一个 64 位浮点数（标准 binary64）表示。

> 对于数组而言，第一个 64 位磁带元素包含值 '```[```'\*2<sup>56</sup> + x，其中 x 是 1 加上磁带上第二个 64 位磁带元素的索引。第二个 64 位磁带元素包含值 '```]```'\*2<sup>56</sup> + x，其中 x 是 1 加上磁带上第一个 64 位磁带元素的索引。数组的所有内容都位于这两个磁带元素之间。我们对格式在 '```{```'\*2<sup>56</sup>+x 和 '```}```'\*2<sup>56</sup>+x 的字的对象进行进行类似的处理。在这两个磁带元素之间，我们在键（必须是字符串）和值之间交替操作。值可以是原子、对象或数组。

> 我们有一个二级数组（*字符串缓冲区*），其中存储了规范化的字符串值。我们将字符串表示为字 '"'\*2<sup>56</sup>+x，其中 x 是二级数组中字符串开头的索引。

此外，我们在文档的开头和结尾添加了一个特殊的字。第一个字被注释为指向磁带上的最后一个字。如图 2 所示。

```
r // pointing to 37 (right after last node)
{ // pointing to next tape location 37 (first node after the scope)
string "Width"
integer 800
string "Height"
integer 600
string "Title"
string "View from my room"
string "Url"
string "http://ex.com/img.png"
string "Private"
false
string "Thumbnail"
{ // pointing to next tape location 25 (first node after the scope)
string "Url"
string "http :// ex. com /th. pn"
string "Height"
integer 125
string "Width"
integer 100
} // pointing to previous tape location 15 (start of the scope)
string "array"
[ // pointing to next tape location 34 (first node after the scope)
integer 116
integer 943
integer 234
] // pointing to previous tape location 26 (start of the scope)
string "Owner"
null
} // pointing to previous tape location 1 (start of the scope)
r // pointing to 0 (start root)
```

在这两个阶段的末尾，我们报告 JSON 文档是否有效[^4]。并产生一个错误代码(如 STRING_ERROR、NUMBER_ERROR, UTF8_ERROR 等)。至此，所有字符串都已标准化，所有数字都已解析和验证。

我们的两阶段设计是基于性能考虑。阶段 1 直接操作输入字节，以 64 字节的批处理数据。通过这种方式，我们可以充分利用对我们良好性能至关重要的 SIMD 指令。阶段 2 更容易，因为阶段 1 确定了所有原子，对象和数组的位置：不需要完成解析一个原子以知道下一个原子的位置。因此，当解析 ```{"key1":"val1"，"key2":"val2"}``` 时，阶段 2 接收标记 ```{```，```"```，```:```，```"```，```"```，```:```，```"```，```}``` 的位置，对应于对象的开始和结束，两个冒号，以及四个字符串中的每个字符串的开头。它可以处理四个字符串而不依赖于数据——在找到字符串“```val1```”的位置之前，我们不需要完成字符串“```key1```”的解析来找到列（:）的位置。除了 unicode 验证之外，我们故意将数字和字符串验证延迟到阶段 2，因为这些任务相对比较昂贵，而且很难在整个输入过程中无条件地、廉价地执行。


## 未完待续


## **参考文献**

[^1]: Alagiannis I, Borovica R, Branco M, Idreos S, Ailamaki A (2012) NoDB in Action: Adaptive Query Processing on Raw Data. Proc VLDB Endow 5(12):1942-1945

[^2]: Boncz PA, Graefe G, He B, Sattler KU (2019) Database architectures for modern hardware. Tech. Rep. 18251, Dagstuhl Seminar

[^3]: Bonetta D, Brantner M (2017) FAD.Js: Fast JSON Data Access Using JIT-based Speculative Optimizations. Proc VLDB Endow 10(12):1778-1789

[^4]: Bray T (2017) The JavaScript Object Notation (JSON) Data Interchange Format. https://tools.ietf.org/html/rfc8259, internet Engineering Task Force, Request for Comments: 8259

[^5]: Cameron RD, Herdy KS, Lin D (2008) High Performance XML Parsing Using Parallel Bit Stream Technology. In: Proceedings of the 2008 Conference of the Center for Advanced Studies on Collaborative Research: Meeting of Minds, ACM, New York, NY, USA, CASCON '08, pp 17:222-17:235

[^6]: Chandramouli B, Prasaad G, Kossmann D, Levandoski J, Hunter J, Barnett M (2018) FASTER: A concurrent keyvalue store with in-place updates. In: Proceedings of the 2018 International Conference on Management of Data, ACM, New York, NY, USA, SIGMOD '18, pp 275-290

[^7]: Cohen J, Roth MS (1978) Analyses of deterministic parsing algorithms. Commun ACM 21(6):448-458

[^8]: Cole CR (2011) 100-Gb/s and beyond transceiver technologies. Optical Fiber Technology 17(5):472-479

[^9]: Downs T (2019) avx-turbo: Test the non-AVX, AVX2 and AVX-512 speeds across various active core counts. https://github.com/travisdowns/avx-turbo

[^10]: Farfan F, Hristidis V, Rangaswami R (2007) Beyond Lazy XML Parsing. In: Proceedings of the 18th International Conference on Database and Expert Systems Applications, Springer-Verlag, Berlin, Heidelberg, DEXA'07, pp 75-86

[^11]: Fog A (2018) Instruction tables: Lists of instruction latencies, throughputs and micro-operation breakdowns for Intel, AMD and VIA CPUs. Tech. rep., Copenhagen University College of Engineering, Copenhagen, Denmark, http://www.agner.org/optimize/instruction_tables.pdf

[^12]: Ge C, Li Y, Eilebrecht E, Chandramouli B, Kossmann D (2019) Speculative distributed CSV data parsing for big data analytics. In: ACM SIGMOD International Conference on Management of Data, ACM

[^13]: Goldberg D (1991) What every computer scientist should know about oating-point arithmetic. ACM Comput Surv 23(1):5-48

[^14]: Green TJ, Gupta A, Miklau G, Onizuka M, Suciu D (2004) Processing XML Streams with Deterministic Automata and Stream Indexes. ACM Trans Database Syst 29(4):752-788

[^15]: Kostoulas MG, Matsa M, Mendelsohn N, Perkins E, Heifets A, Mercaldi M (2006) XML Screamer: An Integrated Approach to High Performance XML Parsing, Validation and Deserialization. In: Proceedings of the 15th International Conference on World Wide Web, ACM, New York, NY, USA, WWW '06, pp 93-102

[^16]: Lemire D, Kaser O (2016) Faster 64-bit universal hashing using carry-less multiplications. Journal of Cryptographic Engineering 6(3):171-185

[^17]: Li Y, Katsipoulakis NR, Chandramouli B, Goldstein J, Kossmann D (2017) Mison: A Fast JSON Parser for Data Analytics. Proc VLDB Endow 10(10):1118-1129, DOI 10.14778/3115404.3115416

[^18]: Liu ZH, Hammerschmidt B, McMahon D (2014) Json data management: Supporting schema-less development in rdbms. In: Proceedings of the 2014 ACM SIGMOD International Conference on Management of Data, ACM, New York, NY, USA, SIGMOD '14, pp 1247-1258

[^19]: Marian A, Simeon J (2003) Projecting XML Documents. In: Proceedings of the 29th International Conference on Very Large Data Bases - Volume 29, VLDB Endowment, VLDB '03, pp 213-224

[^20]: Muhlbauer T, Rodiger W, Seilbeck R, Reiser A, Kemper A, Neumann T (2013) Instant loading for main memory databases. Proc VLDB Endow 6(14):1702-1713

[^21]: Mu la W, Lemire D (2018) Faster Base64 Encoding and Decoding Using AVX2 Instructions. ACM Trans Web 12(3):20:1-20:26

[^22]: Mytkowicz T, Musuvathi M, Schulte W (2014) Dataparallel finite-state machines. In: Proceedings of the 19th International Conference on Architectural Support for Programming Languages and Operating Systems, ACM, New York, NY, USA, ASPLOS '14, pp 529-542

[^23]: Naishlos D (2004) Autovectorization in GCC. In: Proceedings of the 2004 GCC Developers Summit, pp 105-118

[^24]: Noga ML, Schott S, Lowe W (2002) Lazy XML Processing. In: Proceedings of the 2002 ACM Symposium on Document Engineering, ACM, New York, NY, USA, DocEng '02, pp 88-94

[^25]: Palkar S, Abuzaid F, Bailis P, Zaharia M (2018) Filter before you parse: faster analytics on raw data with Sparser. Proceedings of the VLDB Endowment 11(11):1576-1589

[^26]: Pavlopoulou C, Carman Jr EP, Westmann T, Carey MJ, Tsotras VJ (2018) A Parallel and Scalable Processor for JSON Data. In: EDBT'18

[^27]: Tahara D, Diamond T, Abadi DJ (2014) Sinew: A SQL System for Multi-structured Data. In: Proceedings of the 2014 ACM SIGMOD International Conference on Management of Data, ACM, New York, NY, USA, SIGMOD '14, pp 815-826

[^28]: Takase T, Miyashita H, Suzumura T, Tatsubori M (2005) An Adaptive, Fast, and Safe XML Parser Based on Byte Sequences Memorization. In: Proceedings of the 14th International Conference on World Wide Web, ACM, New York, NY, USA, WWW '05, pp 692-701

[^29]: Xu Q, Siyamwala H, Ghosh M, Suri T, Awasthi M, Guz Z, Shayesteh A, Balakrishnan V (2015) Performance analysis of NVMe SSDs and their implication on real world databases. In: Proceedings of the 8th ACM International Systems and Storage Conference, ACM, New York, NY, USA, SYSTOR '15, pp 6:1-6:11

[^30]: Zhang Y, Pan Y, Chiu K (2009) Speculative p-DFAs for parallel XML parsing. In: High Performance Computing (HiPC), 2009 International Conference on, IEEE, pp 388-397

[^31]: Zhang Y, Pan Y, Chiu K (2009) Speculative p-DFAs for parallel XML parsing. In: High Performance Computing (HiPC), 2009 International Conference on, IEEE, pp 388-397