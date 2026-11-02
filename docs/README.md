# `CHIME_TRUETYPE`

**Chime_TrueType** 是一个用于解析TrueType字体文件（TTF、TTC、OTF、OTC）的C++库。

附属的 **Chime_TextPrinter** 提供横向文本排版功能。

> **由于作者很忙，文档将在11月底完成。**

> **English document is currently in progress. Will be finished before 2027/01/01.**

## 目录

* [功能与特性](#功能与特性)
* [性能](#性能)
* [算法](#算法)
* [效果](#效果)
* [策略](#策略)
* [示例](#示例)
* [TODO](#TODO)
* [你可能想要但我没实现的功能](#你可能想要但我没实现的功能)
* [注意事项](#注意事项)
* [一些说明](#一些说明)

###

* [Chime_TrueType.h 参考文档](Chime_TrueType.md)
* [Chime_TextPrinter.h 参考文档](Chime_TextPrinter.md)

## 功能与特性

### 快速了解

* 提供与 [stb_truetype.h](https://github.com/nothings/stb) 大致相同的功能，详见 [#对比 stb_truetype](#对比-stb_truetype)
* 在CPU上的高速字形栅格化，哪怕是SDF也能快速生成，详见 [#性能](#性能)
* 可以将渲染完的全套字形及相关信息压缩后保存到文件，用于之后快速加载，详见 [#存盘和加载](Chime_TrueType.md#存盘和加载)
* 提供高层级的封装，非常易用
* 顾名思义的函数和变量命名，几乎不使用缩写，易于理解代码逻辑

### 对比 stb_truetype

比对 stb_truetype 和 Chime_TrueType 支持的TTF/OTF表，一些较为基础的表不在此列出：

| 表名 | stb_truetype | Chime_TrueType
| ---- | ------------ | --------------
| cmap | √ <sup>[1]</sup> | √ <sup>[2]</sup>
| name | √                | √
| glyf | √ <sup>[3]</sup> | √
| kern | √                | √
| GPOS | △ <sup>[4]</sup> | ▲ <sup>[5]</sup>
| CFF  | √                | × <sup>[6]</sup>

\* kern表记录特定字体间的横向字距微调（kerning），GPOS表涵盖kern表的功能，这俩表不是每种字体都有。

<sup>1</sup> stb_truetype 在只能找到非微软平台的cmap子表时会初始化失败，无法进行后续字形栅格化。

<sup>2</sup> stb_truetype 支持的子表格式为0、4、6、12、13，Chime_TrueType在此基础上支持格式10（不过没找到测试材料）。

<sup>3</sup> 复合字形中的组分字形定位错误（[stb/issue/1852](https://github.com/nothings/stb/issues/1852)）。

<sup>4</sup> 仅支持横向kerning的一种记录格式，且不支持扩张子表，无法解析谷歌 [NotoSans](https://fonts.google.com/noto/specimen/Noto+Sans) 系列字体中的kerning（[stb/issues/1997](https://github.com/nothings/stb/issues/1997)、[stb/issues/1998](https://github.com/nothings/stb/issues/1998)）。

<sup>5</sup> 仅支持横向kerning（这个库未实现纵向文本排版，因此其它信息多余）。

<sup>6</sup> CFF是可以嵌入OTF的一种独立格式，同glyf表一样用于提供字形轮廓，目前很少使用（Win10的预装字体一概不用），一些曾经包含CFF的字体在新版本中使用glyf表，比如 [Inconsolata](https://github.com/google/fonts/tree/main/ofl/inconsolata/static) 。考虑到再啃一份文件标准的工作量及缺少必要性，我决定放弃CFF的解析。

## 性能

比对使用 stb_truetype 及 Chime_TrueType 的字形栅格化效率。

### 测试环境

* 编译 & 运行环境：Windows 10
* 编译器：CLang
* 编译选项：Release Build（NDEBUG，优化：O2）
* 处理器：Intel(R) Core(TM) Ultra 7 265K   3.90 GHz
* Win10电源选项为“最佳性能”

### 具体测试环节

测试以下环节的耗时：

* 取得轮廓数据：从glyf表获取字形轮廓数据，计算字形包围盒
* 栅格化第一步：将轮廓细分为折线多边形
* 栅格化第二步：将细分得到的多边形提交给栅格化器，生成图像

最好是排除“取得轮廓数据”，仅测试栅格化耗时更能体现关键的效率差异。<br>
不过因为 `stbtt_MakeGlyphBitmap(...)` 和 `stbtt_GetGlyphSDF(...)` 都包含了这一步，并且库用户通常不会直接改库中的文件代码，因此只好将这一步也加入测试。

### 常规栅格化耗时

#### 测试文件：Ancient Medium.ttf

| 字高 | stb_truetype : 0.35 | Chime_TrueType : 0.35 | Chime_TrueType : 0.1875
| ---- | ------------------- | --------------------- | -----------------------
| 32   | 0.001122            | 0.000763              |　0.000883
| 64   | 0.001588            | 0.001198              |　0.001331
| 128  | 0.002650            | 0.002147              |　0.002268
| 1024 | 0.050218            | 0.042922              |　0.043656

\* 冒号后面的数值是细分精度，stb_truetype 无更改细分精度的函数接口，其代码中指定为0.35。<br>
\* 时间为4次运行的平均值，单位为秒。<br>

#### 测试文件：SarasaFixedSC-Regular.ttf

| 字高 | stb_truetype : 0.35 | Chime_TrueType : 0.35 | Chime_TrueType : 0.1875
| ---- | ------------------- | --------------------- | -----------------------
| 32   | 0.278544            | 0.203683              | 0.208450
| 64   | 0.433190            | 0.344025              | 0.357942
| 128  | 0.888738            | 0.795218              | 0.818429

\* 这是个有50000+字形的CJK字体。

### SDF生成耗时

#### 测试文件：Ancient Medium.ttf

| 字高 | SDF Padding | stb_truetype : 0.35 | Chime_TrueType : 0.35 | Chime_TrueType : 0.1875
| ---- | ----------- | ------------------- | --------------------- | -----------------------
| 32   | 1           | 0.047673            | 0.002203              | 0.002390
| 64   | 1           | 0.141036            | 0.003049              | 0.003294
| 128  | 2           | 0.485922            | 0.006416              | 0.007055
| 1024 | 16          | 27.32853            | 0.165521              | 0.195725

#### 测试文件：SarasaFixedSC-Regular.ttf

| 字高 | SDF Padding | stb_truetype : 0.35 | Chime_TrueType : 0.35 | Chime_TrueType : 0.1875
| ---- | ----------- | ------------------- | --------------------- | -----------------------
| 32   | 1           | 16.95783               | 0.596954              | 0.612693
| 64   | 1           | 52.8645 <sup>[1]</sup> | 0.839418              | 0.892663
| 128  | 2           | 188.617 <sup>[1]</sup> | 1.911295              | 2.077003

<sup>1</sup> 慢得过分，只测了1次。

SDF生成快得很魔幻吧？实不相瞒，我自己也惊了！

## 算法

### 常规栅格化

TODO

### SDF生成

TODO

## 效果

TODO

## 策略

TODO: Non-SDF和SDF的使用场景

若要生成SDF，注意 `sdfPadding` 太小的话会影响文本字高小于贴图字高时的渲染效果（锐度高，有锯齿感）。<br>
在采样器的 `mipBias` 为-1的情况下，若贴图字高在64或以上，要确保在文本字高为32时有不模糊不锐化的边缘，应确保 `sdfPadding` 约为字高的1/128~1/64。<br>
1/128的效果会稍微锐利点，但在文本字高较小时更易辨识。

## 示例

### 应用SDF

* [Vulkan示例](https://github.com/EasyVulkan/EasyVulkan.github.io/tree/main/solution/EasyVulkan_Ex)，VS2026解决方案

### 实时栅格化

* [无窗口示例1（TODO）](TODO)，打印字体名称并保存为PNG
* [无窗口示例2（TODO）](TODO)，打印一段带自定义格式字符串的文本并保存为PNG

如果你写了一些更好的示例，愿意把项目链接摆在这里，可以到[Issues](https://github.com/EasyVulkan/Chime_TrueType/issues)告诉我。

## TODO

### 支持仅用于MacOS或OS_X的字体

相关代码已经完成，由于无法测试正确性，被我注释掉了。
我搜了一圈没找到sfntVersion标识为 `'true'` 的字体，而且我也没有Mac，而且如今的MacOS字体似乎大都兼容不同的操作系统……<br>
谁要是遇到了无法解析的字体并且提示sfnt版本号不受支持，请把文件提供给我。

### 测试GCC能否正常编译

MSVC和CLang都能正常编译。<br>
GCC下没测试，如遇到问题请[发issue](https://github.com/EasyVulkan/Chime_TrueType/issues/new)。

## 你可能想要但我没实现的功能

### GPU栅格化字形

TODO: Reason

### 字体子集化

TODO: Reason

### 渲染竖排文字

TODO: Reason

### GPOS表里一些杂七杂八的置位参数

TODO: Reason

### 多通道SDF

TODO: Reason

## 注意事项

### 字高的计算方法

TODO

## 一些说明

### 为什么会有这个库？

这个库最开始是 [stb_truetype.h](https://github.com/nothings/stb) 的C++高层级封装，渐渐发展为其替代品。

我有在stb的仓库中发issue，但我发觉 stb_truetype.h 的可改进点很多，并且有些需要改很长一串代码，不敢轻易下手。加之其审核PR和发布更新的效率较慢，索性自己从头整一个库。

### 关于命名

依照个人习惯，我使用大驼峰命名函数，小驼峰命名变量和类型，首字母缩写看作一整个单词。<br>
至于为什么有些地方的“ID”没写成“Id”，旨在让各类table的中的成员名尽量与微软的OpenType标准保持一致。

### 维护与更新频率

持续维护，小修小补会高速搞定。<br>
不过没人提的话我不太可能会加新功能。

### 还有疑问？

如果你对于实现的细节有任何疑问，无论你是想写个配套的排版工具、类似的库，或者想做点research来应付作业，尽管在[Discussions](https://github.com/EasyVulkan/Chime_TrueType/discussions)向我提问。<br>
或者你看完了文档还是不会用这个库，也尽管向我发问。