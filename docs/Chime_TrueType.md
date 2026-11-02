# **Chime_TrueType.h** Reference Page

**Chime_TrueType.h** 仅负责字体的解析和字形栅格化。<br>
负责栅格化的函数无需你调用，此页面上不做说明。

关于排版和打印的功能，参见 [Chime_TextPrinter.h 参考文档](Chime_TextPrinter.md) 。

## `class ttfLoader`

使用 `ttfLoader` 读取TrueType文件，将全部字形栅格化为贴图，以备之后实现高效的文本渲染。

### 构造函数

	ttfLoader(const char* filepath_ttf, float fontHeight, int8_t sdfPadding);

	ttfLoader(const uint8_t* fileBytes_ttf, float fontHeight, int8_t sdfPadding);

	ttfLoader(const char* filepath_preLoaded);
    
	ttfLoader(const uint8_t* fileBytes_preLoaded, size_t fileSize);

对应各个 `LoadTtf(...)` 和 `LoadLoaded(...)` 函数，详见后文。

### 读取TrueType文件

#### 读取各种TrueType文件

	bool LoadTtf(const char* filepath, float fontHeight, int8_t sdfPadding);

	bool LoadTtf(const uint8_t* fileBytes, float fontHeight, int8_t sdfPadding);

读取TTF、TTC、OTF、OTC，若文件类型为TTC或OTC，则读取其中的第一个字体。

* filepath：TrueType文件路径
* fileBytes：指向文件数据的指针，用于已将文件加载至内存的情况
* fontHeight：字高，计算方式见 [#字高的计算方法](README.md#字高的计算方法) 
* sdfPadding：负数生成常规非SDF贴图；为0时生成非SDF但当作SDF使用；正数时生成SDF并指定自字体轮廓向外扩张的距离
* 返回值：若返回 `true` ，则函数执行成功

关于 `sdfPadding` 的计算及相应效果，参见 [#策略](README.md#策略) 。

#### 读取TTC/OTC中指定索引的字体

	bool LoadTtc(const char* filepath, uint32_t fontIndex, float fontHeight, int8_t sdfPadding);

	bool LoadTtc(const uint8_t* fileBytes, uint32_t fontIndex, float fontHeight, int8_t sdfPadding);

* fontIndex：需要读取的字体在TTC/OTF文件中的索引。若提供的文件是单个TTF或OTF，则无视此参数

### 获取字体名称和TTC/OTC中的字体个数

####

	static auto GetFontNames(const char* filepath, std::unique_ptr<uint8_t[]>& fileBytes);

* filepath：TrueType文件路径
* fileBytes：用于接收一个智能指针，指向加载到内存的文件数据，留待之后提供给 `LoadTtc(...)`
* 返回值：返回一个 `std::vector` 容器，其 `size()` 即为字体个数。

####

	static auto GetFontNames(const uint8_t* fileBytes);

* fileBytes：指向文件数据的指针，用于已将文件加载至内存的情况（比如编入可执行文件的情况）

#### 字体名称

上述函数返回的 `std::vector` 中的元素类型为 `nameParser::result_t` ，包含两个字段：

* fullName：字体的全名
* nameEncodingIsU16：字体名称的编码

若 `nameEncodingIsU16` 为 `true` ，则字体的全名为2字节UTF-16编码，否则为ASCII编码或对应不同地区的多字节编码。<br>
如今很少使用多字节地区编码，由于[Apple的标准](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6name.html)并没有给出每个地区语言对应的代码页，因此没有记录对应的代码页。

多字节编码字符串无法被 **Chime_TextPrinter.h** 中的 `textPrinter` 正确打印（仅其中的英文字符可以视作ASCII编码被正确打印），在控制台代码页正确的情况下可以被输出到控制台。

对于UTF-16编码的字体名称，用 `fullName.U16StringView()` 获取UTF-16字符串视图，可用 `textPrinter` 中的各Print函数将其打印出来。

#### 字体样式识别

这个库并不根据head表中的 `macStyle` 字段或OS/2表中的 `fsType` 字段鉴别字体样式，如今的字体样式已远超这俩字段的定义范围，因此使用完全基于字体全名的方式识别字体样式。

你可以通过 `fullName.Contains(...)` 检查字体样式，相应英语单词的首字母应当大写，并根据编码区分字符串参数类型。<br>
例，识别粗斜体：

    if (fontName[i].nameEncodingIsU16)
        if (fontName[i].fullName.Contains(u"Bold") &&
            fontName[i].fullName.Contains(u"Italic")) { // 对应UTF-16，使用 u"..."
                ...
            }
    else
        if (fontName[i].fullName.Contains("Bold") &&
            fontName[i].fullName.Contains("Italic")) {  // 对应ASCII和多字节编码中的英语部分
                ...
            }

注意“Regular”可能不会出现在字体名称中，可以通过排除法来识别。

### 存盘和加载

#### 将字形贴图及字形信息保存到文件

    bool SaveLoaded(const char* filepath);

* filepath：文件路径，根据后缀名选择文件格式
* 返回值：若返回 `true` ，则函数执行成功

| 后缀名 | 说明 |
| ------ | ---- |
| .png   | 保存为图片查看器可以直接显示的PNG，需要用到 [stb_image_write.h](https://github.com/nothings/stb/blob/master/stb_image_write.h)
| 其它   | 保存为我自定义的格式，推荐后缀名为.ttfl（小写L）

**PNG**

如需保存为PNG，你需要在 `#include "Chime_TrueType.h"` 或 `#include "Chime_TextPrinter.h"` 前 `#include "stb_image_write.h"`。

无论保存到PNG，还是从PNG读取都很慢。从PNG读取远远慢于栅格化全部字形。<br>
保存到PNG只有两个优势：

* 大小约为我自定义格式的一半，由于如今的硬盘都很大，这完全可以忽略
* 可用图片查看器直接查看，便于检查效果

因此，建议仅在需要检查效果时保存到PNG格式。

**ttfL**

我自定义格式的格式签名为“ttfL”，姑且如此称呼。

根据字形贴图的灰度值中有大量连续的0和255的特点，使用非常简单的RLE算法进行压缩，因此存/读速度都很快。

#### 从文件读取字形贴图和字形信息

	bool LoadLoaded(const char* filepath);

	bool LoadLoaded(const uint8_t* fileBytes, size_t fileSize);

* filepath：PNG或ttfL文件路径
* fileBytes：指向文件数据的指针，用于已将文件加载至内存的情况（比如编入可执行文件的情况）
* fileSize：文件大小

文件类型会通过文件数据中的格式签名进行判断，与文件路径中的后缀名无关。

如需读取PNG，你需要在 `#include "Chime_TrueType.h"` 或 `#include "Chime_TextPrinter.h"` 前 [`#include "stb_image.h"`](https://github.com/nothings/stb/blob/master/stb_image.h)。

### 限制字形贴图大小

默认的贴图宽度是16384，贴图数组（这是个图形API中的概念，不是C++中的数组）的最大层数为2048。<br>
这俩数值是根据2026年大部分PC端显卡的参数确定的，移动端显卡的参数可能低于上述值。

因此根据图形API给出的参数限制，可能需要对贴图的宽度和图层数加以限制。<br>
限制的意义在于提前报错，而非等到把字形贴图提交给图形API后再让图形API报错。

#### 限制贴图数组图层数

	static void MaxImageLayerCount(uint32_t maxImageLayerCount);

#### 限制贴图最大宽度

	static void MaxImageLayerWidth(uint32_t maxImageLayerWidth);

### 指定细分精度

`ttfLoader` 在栅格化字形时，首先将轮廓细分为多边形折线，细分精度直接影响最终的视觉效果，参见 [#效果](README.md#效果) 。<br>
默认情况下，会根据字高自动计算合适的细分精度。

如果你希望指定一个固定的精度，调用：

	static void TessellationPrecision(float precision);

* precision：细分精度，值越小精度越高，若小于0.0625，则栅格化时根据字高自动计算实际的细分精度

该值小于0.1875时，对于字高大于等于32的情况几乎观察不到效果差异。<br>

细分精度对栅格化效率的影响是非线性的：

* 对于直线多曲线少的字体（比如各种黑体），细分精度提升一倍（即除以2）对栅格化效率的影响可能不足2%。
* 精度为0.35时的栅格化耗时与精度为0.0625时的比值通常约为15~20%，对于曲线较多的拉丁花体字可能达到25%。

注意，该值同样影响 [`ttfLoader_imageless::RasterizeGlyph(...)`](#栅格化单个字形) 。

### 释放内存

	void Reset();

调用该函数将析构 `ttfLoader` 实例，但延续其生存期。<br>
调用目的通常是释放图像数据占用的内存。可用于之后可能重用当前 `ttfLoader` 实例的情况，或因作用域而暂时不会自动析构的情况。

虽然有重新初始化的效果，读取字体失败时不必手动调用此函数来重置 `ttfLoader` 实例。

### Getter

这里只解释个别可能让人摸不着头脑的字段对应的Getter函数。

#### float Scale();

字形贴图的字高 ÷ 字体在其设计单位下的原始字高（通常是个上万的数值）。

#### float PixelDistanceScale();

SDF贴图中，自字形边缘内缩/外扩1像素时，像素灰度的增加/减少值。<br>
名称是我自己起的，并非字体排印学中的概念。

#### float ImageAscent();

这个库渲染贴图时将所有字形对齐到一致的基线，这是从基线到贴图图层顶部的距离。<br>
这个值是为了便于后续计算而记录的，名称是我自己起的，并非字体排印学中的概念。

## `class ttfLoader_imageless`

使用 `ttfLoader_imageless` 读取TrueType文件时，不会进行预先将全部字形栅格化，而是将字形的轮廓信息保留下来用于之后的实时栅格化。

### 构造函数

	ttfLoader_imageless(const char* filepath_ttf);
	
	ttfLoader_imageless(const uint8_t* fileBytes_ttf);

对应各个 `LoadTtf(...)` 函数。

### 读取TrueType文件

	bool LoadTtf(const char* filepath);

	bool LoadTtf(const uint8_t* fileBytes);

	bool LoadTtc(const char* filepath, uint32_t fontIndex);

	bool LoadTtc(const uint8_t* fileBytes, uint32_t fontIndex);

相比 `ttfLoader` 中的对位函数少了栅格化所需参数。

### 栅格化单个字形

如果你使用 **Chime_TextPrinter.h** ，该函数会被 `textPrinter` 实例调用，通常无需你手动调用。<br>
如果你打算自行实现文本排版，并在CPU上根据具体文本实时栅格化字形，则调用此函数。

	static void RasterizeGlyph(
		const glyfParser& glyfData,
		uint32_t glyphIndex,
		float scale,
		float fontHeight,
		float subpixelPositionX,
		float subpixelPositionY,
		std::vector<glyfParser::pointF32>& points,
		std::vector<rasterizer::point>& tessellatedPoints,
		rasterizer& rasterizer,
		std::vector<uint8_t>& pixels,
		uint16_t& imageWidth,
		uint16_t& imageHeight);

* glyfData：已经完成了glyf表解析的 `glyfParser` 对象
* glyphIndex：要被栅格化的字形的索引
* scale：字高 ÷ 字体在其设计单位下的原始字高
* fontHeight：字高
* subpixelPositionX：字形包围盒左边界距离其所在像素左边界的位置
* subpixelPositionY：字形包围盒上边界距离其所在像素上边界的位置
* points：可重用的容器，为避免反复内存分配而作为参数传入，内容无妨
* tessellatedPoints：可重用的容器，为避免反复内存分配而作为参数传入，内容无妨
* rasterizer：`rasterizer` 实例，为避免反复内存分配而作为参数传入，内容无妨
* pixels：用于接收栅格化得到的图像数据
* imageWidth：接收栅格化得到的图像的宽度
* imageHeight：接收栅格化得到的图像的高度

设文本中一个字形的包围盒左边界为 `left` ，上边界为 `top` ，则 `subpixelPositionX` 和 `subpixelPositionY` 可按下式计算：

	float subpixelPositionX = left - std::floor(left);
	float subpixelPositionY = -top - std::floor(-top);

这里top要取负是因为在字形的设计坐标系中y轴正方向向上，而逐行生成图像数据时行索引向下递增。

### 释放内存

	void Reset();

在将 `ttfLoader_imageless` 实例的数据提供给 **Chime_TextPrinter.h** 中的 `font_textureless` 后，可能有数据残留（如果字体有kerning数据，`kernings` 中的动态分配数据是一定会留下的）。<br>
虽然打从一开始就没有图像数据，但个别字体可能产生几MB的kerning值对，你想提前清理的话可调用此函数。