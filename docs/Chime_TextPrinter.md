# **Chime_TextPrinter.h** Reference Page

**Chime_TextPrinter.h** 包含了API无关的文本排版、打印功能，并声明了一系列用于对接图形API的函数/类型接口以便实现高效的文本渲染。

## `class font`

`ttfLoader` 是一个API无关的类型，由 `font` 负责与图形API对接。

### 构造函数

#### 直接读取文件

	font(const char* filepath_ttf, float fontHeight, int8_t sdfPadding, bool generateMipmap);

	font(const char* filepath_preLoaded, bool generateMipmap);

* filepath：TrueType文件路径
* fontHeight：字高，计算方式见 [#字高的计算方法](README.md#字高的计算方法)
* sdfPadding：负数则生成非SDF；为0时生成非SDF但当做SDF使用；正数时生成SDF并指定自字体轮廓向外扩张的距离
* generateMipmap：是否生成mipmap，建议为true
* filepath_preLoaded：PNG或ttfL文件路径

该函数在执行过程中会创建一个 `ttfLoader` 实例以读取文件。

#### 从ttfLoader取得数据

	font(ttfLoader& loader, bool generateMipmap);

    font(const ttfLoader& loader, bool generateMipmap);

* loader：已包含字形贴图和相关信息的 `ttfLoader` 实例

构造函数不返值，可以通过以下方式确认函数是否执行成功：

* 若需要创建图形API相关数据，检查 `pApiData` 字段的值（调用成员函数 `operator->()` ）。
* 若不需要创建图形API相关数据，除非内存极其吃紧，函数执行成功与否取决于 `ttfLoader` 读取文件成功与否，那么分两步走，不要用 `font` 直接读取文件。

### API特定函数

#### 创建API数据

    API_SPECIFIC void CreateApiData(const uint8_t* pImageData, bool generateMipmap);

* pImageData：指向图像数据的指针
* generateMipmap：是否生成mipmap

需要你自行实现该函数：要求动态分配 [`apiData_font`](#api_specific-class-apidata_font) 类型的实例，将其地址返回给智能指针类型的字段 `pApiData` ，创建失败时返回 `nullptr` 。<br>
若不需要图形API调用，留空即可。

## `class font_textureless`

`font_textureless` 是 `font` 的派生类，顾名思义，不包含贴图数据，用于根据文本实时栅格化字形的情况。

`font_textureless` 包含的字段与 `font` 一致（即无需虚析构函数也可以安全实现运行期多态）。<br>
`font_textureless` 的指针/引用能被安全地转为 `font` 的指针/引用，`textPrinter` 中的各个Print函数，根据实际类型确定打印方式，详见 [#打印文本](#打印文本) 。

### 构造函数

	font_textureless(const char* filepath_ttf);

	font_textureless(ttfLoader_imageless& loader);

	font_textureless(const ttfLoader_imageless& loader);

* filepath_ttf：TrueType文件路径
* loader：已包含相关信息的 `ttfLoader_imageless` 实例

相比 `font` 中的对位函数少了栅格化所需参数。<br>
除非内存极其吃紧，函数执行成功与否取决于 `ttfLoader_imageless` 读取文件成功与否。

## `class textPrinter`

### 构造函数

	textPrinter(uint32_t initialCapacity, float fontHeight, float lineHeightRatio, float extraCharacterSpacing, float maxLineWidth, uint32_t color_rgba);

* initialCapacity：用于存放顶点数据的线性容器的初始容量
* fontHeight：打印出的字高，计算方式见 [#字高的计算方法](README.md#字高的计算方法)
* lineHeightRatio：行距比例，建议为1.2
* extraCharacterSpacing：字符间的额外宽度，用于调整字符间距，可以为负数
* maxLineWidth：最大行宽，打印的文本超出此长度时自动换行，单位为像素
* color_rgba：打印出的文字颜色，从高8位到低8位分别对应RGBA通道，默认为不透明全白

除 `initialCapacity` 外的参数皆有相应的Setter函数，可以之后再行调整。

### 打印文本

#### 左对齐打印

	vec2 Print(vec2 view, const auto& text, const font& font);

	vec2 Print(fnPrint& deferredPrinting, const auto& text, const font& font);

* view：该参数规定打印区域左上角位置，横坐标为打印区域左边界位置，纵坐标为首行的基线位置
* deferredPrinting：[`fnPrint`](#延迟打印) 实例，用于实现延迟打印
* text：要打印的文本，可以是C数组指针、各种`std::basic_string`、各种`std::basic_string_view`
* font：字体，为 `font` 或 `font_textureless` 类型的实例
* 返回值：执行该函数后打印区域的大小，该值与之后调用Getter函数 `PrintAreaSize()` 返回的值一致

调用该函数打印有效字符（指会被打印或起空格、换行作用的字符）后，输入位置发生移动。<br>
若提供的 `view` 相同，且没有调用成员函数 `ResetPrintArea()` 来[重置输入位置](#重置打印位置)，则新文本总是会接在先前的文本之后打印。<br>
这使得你可以通过多次调用 `Print(...)` 来打印同一段落的不同句子，而无需计算每一句的起始坐标。

#### 居中对齐打印

	float Print_HorizontallyCentered(vec2 view, const auto& text, const font& font, float& printAreaWidth, bool isStrict);

	float Print_HorizontallyCentered(vec2 view, const auto& text, const font& font, bool isStrict);

	float Print_HorizontallyCentered(fnPrint& deferredPrinting, const auto& text, const font& font, float& printAreaWidth, bool isStrict);

	float Print_HorizontallyCentered(fnPrint& deferredPrinting, const auto& text, const font& font, bool isStrict);

* view：横坐标为打印区域居中对齐位置，纵坐标为首行的基线位置
* printAreaWidth：用于接收打印区域的宽度，该值为本次打印中最宽行的宽度
* isStrict：若为 `true` ，则严格按照字形轮廓的左右边界计算行宽，否则基于字形的步进宽度（advance width）计算行宽
* 返回值：执行该函数后打印区域的高度

`Print_HorizontallyCentered(...)` 根据行宽计算行首相对于 `view.x` 的偏移量，实现居中对齐。<br>
`Print_HorizontallyCentered(...)` 不改变 `PrintAreaSize()` 所返回的横坐标的值（ `printAreaSize.x` 记录的总是左对齐打印时的打印区域宽度）。

若提供的 `view` 相同，且没有调用成员函数 `ResetPrintArea()` 来[重置输入位置](#重置打印位置)，则新文本总是会被打印在先前文本的下一行。

#### 右对齐打印

	float Print_RightAligned(vec2 view, const auto& text, const font& font, float& printAreaWidth);
	
	float Print_RightAligned(vec2 view, const auto& text, const font& font);

	float Print_RightAligned(fnPrint& deferredPrinting, const auto& text, const font& font, float& printAreaWidth);

	float Print_RightAligned(fnPrint& deferredPrinting, const auto& text, const font& font);

* view：该参数规定打印区域右上角位置，横坐标为打印区域右边界位置，纵坐标为首行的基线位置
* printAreaWidth：用于接收打印区域的宽度，该值为本次打印中最宽行的宽度
* 返回值：执行该函数后打印区域的高度

`Print_RightAligned(...)` 同样不改变 `PrintAreaSize()` 所返回的横坐标的值。

若提供的 `view` 相同，且没有调用成员函数 `ResetPrintArea()` 来[重置输入位置](#重置打印位置)，则新文本总是会被打印在先前文本的下一行。<br>
你应该先得到整个段落的文本，再使用 `Print_RightAligned(...)` 打印整个段落。

#### 重置打印位置

	void ResetPrintArea();

该函数重置打印区域中的当前输入位置，并将所记录的打印区域大小置零。

通常而言，两次 `ResetPrintArea()` 调用之间，向各打印函数提供的 `view` 参数应当一致（即，若不调用此函数，则打印区域的位置不该发生改变）。

#### 延迟打印

`fnPrint` 类型顾名思义是执行打印的函数对象，它保存部分打印参数，留待之后执行打印（我将这种做法称为延迟打印）。

延迟打印有以下作用：

* 用于需要根据文本区域大小调整位置的情形
* 用于需要对已生成的顶点做手脚的情形，比如 `textPrinter::characterVertex::layerIndex` 通常只会使用低11位，可以用剩下的位指定一些渲染效果
* 对于调用图形API的情况，可以在生成当前帧所需的所有顶点后再确定顶点缓冲区大小

调用时仅需指定 `view` ：

	void operator()(vec2 view);

#### 获取基线位置

上述各打印函数提供的参数 `view` 的纵坐标是基线位置（基线就是通常写字时同一行中的汉字或大写英文字母底部对齐到的位置）。<br>
打印参数中要求提供基线，是为了确保同一行中即便有不同大小的字符也总是能正确对齐。

在实际打印时，有时知晓的是打印区域的顶部位置，而不直接知晓首行的基线位置。<br>
成员函数 `GetBaseline(...)` 根据字段 `fontHeight` 的值计算基线位置：

	float GetBaseline(float textTop, const font& font) const;

* textTop：一行的顶部位置
* font：字体
* 返回值：一行的基线位置

### 自定义文本解析

默认情况下，各打印函数依序打印文本中的各个字符，遭遇换行符时进行换行。<br>
你可以用Setter函数 `FnNextCharacter(...)` 设置一个回调函数，用于自定义如何取得下一个有效字符。

该回调函数的函数原型：

	std::optional<uint32_t> NextCharacter(textPrinter::proxy textPrinter, const void* text, uint32_t textLength, uint8_t characterStride, uint32_t& currentIndex);

* textPrinter：对 `textPrinter` 实例的一个受限引用，可通过该引用调用特定的Setter，详见代码中 `textPrinter::proxy` 的定义
* text：文本数据的地址
* textLength：文本中字符的个数
* characterStride：单个字符数据的大小，单位是字节
* currentIndex：字符索引，文本解析到的当前位置
* 返回值：返回一个有效字符，或返回一个默认构造的值 `{}` 表示已解析完整个文本。

`textPrinter::proxy` 的主要用途是设置文本颜色，无法设置文本大小。<br>
设置文本大小会影响到前后字符的间距和行距，前者的计算可以实现但代码会被改得很难看，后者无法实现（因为无法溯及过往地更改已经渲染好的、同一行的字符的位置）。

自定义文本解析的典型用途是解析文本中用于设置文本颜色的格式信息。<br>
例如，我用“#r#”表示把之后的文字设置为红色，那么上述回调函数中执行判定逻辑后应该执行 `textPrinter.Color(0xff0000ff)` ，并让 `currentIndex += 3` ，然后如果“#r#”后面是个有效字符，则 `currentIndex++` 后返回该字符。

### 指定使用内置栅格化器打印时的混色方式

你可以用Setter函数 `FnPerformBlending(...)` 设置一个回调函数，指定在使用这个库的CPU栅格化器进行打印时的混色方式。

该回调函数的函数原型：

	void PerformBlending(uint8_t* pPixel, uint8_t monochromeValue, uint8_t canvasPixelStride, const uint8_t color_rgba[4]);

* pPixel：指向画布上的目标像素数据
* monochromeValue：8位灰度值，255表示被字形完全覆盖为，0表示完全不被覆盖
* canvasPixelStride：单个像素数据的大小，单位是字节
* color_rgba：当前指定的文本颜色，8位值，索引0到3分别对应RGBA各通道

典型的基于透明度混色的函数如下（假定画布为8位每通道，像素从低地址到高地址依次对应RGBA，color_rgba视作非预乘透明度）：

	[](uint8_t* pPixel, uint8_t monochromeValue, uint8_t canvasPixelStride, const uint8_t color_rgba[4]) {
		float alpha = monochromeValue / 255.f;
		switch (canvasPixelStride) {
		case 4:
			pPixel[3] = uint8_t(color_rgba[3] * alpha + pPixel[3] * (1 - alpha) + 0.5f);
		case 3:
			pPixel[2] = uint8_t(color_rgba[2] * alpha + pPixel[2] * (1 - alpha) + 0.5f);
		case 2:
			pPixel[1] = uint8_t(color_rgba[1] * alpha + pPixel[1] * (1 - alpha) + 0.5f);
		case 1:
			pPixel[0] = uint8_t(color_rgba[0] * alpha + pPixel[0] * (1 - alpha) + 0.5f);
		};
	}

### 更新API数据

对于Vulkan、DX12等“录制命令，然后再提交”的API，在提交录制的文本渲染命令前，调用 `UpdateApiData(...)` 更新顶点缓冲区：

	void UpdateApiData(bool resetPrintArea);

* resetPrintArea：是否顺带重置打印区域中的当前输入位置，默认为true

该函数会调用 `UpdateApiData_Internal()` 执行实际的顶点缓冲区更新。

### API特定函数

#### 创建API数据

	API_SPECIFIC void CreateApiData(uint32_t capacity);

* capacity：顶点缓冲区中的最大顶点数，整个顶点缓冲区的大小应不小于 `capacity * sizeof(textPrinter::characterVertex)`

需要你自行实现该函数：要求动态分配 [`apiData_textPrinter`](#api_specific-class-apidata_textprinter) 类型的实例，将其地址返回给智能指针类型的字段 `pApiData` ，创建失败时返回 `nullptr` 。<br>
若不需要图形API调用，留空即可。

#### 更新API数据

	API_SPECIFIC void UpdateApiData_Internal() const;

需要你自行实现该函数：对于Vulkan、DX12等“录制命令，然后再提交”的API，要求将字段 `vertices` 中的数据复制到顶点缓冲区，实现上可能会有一定的难点，见 [#`API_SPECIFIC class apiData_textPrinter`](#api_specific-class-apidata_textprinter)。<br>
对于OpenGL、DX11等即时绘制的图形API，此函数留空即可，相应地在下文的 `Draw(...)` 函数中更新顶点缓冲区。<br>
若不需要图形API调用，留空即可。

#### 执行绘制

	API_SPECIFIC void Draw(vec2 view, const font& font, uint32_t vertexCount, uint32_t firstVertexIndex) const;

* vertexCount：顶点个数
* firstVertexIndex：首个顶点的索引

需要你自行实现该函数：录制/调用图形API的实例化绘制命令，从顶点缓冲区读取数据，采样字形贴图以绘制文本。<br>
若不需要图形API调用，留空即可。

## `class textPrinter::characterVertex`

若不需要图形API调用，不必关心顶点数据的内容。

### 顶点数据

	struct characterVertex {
		vec2 position;
		float scale;
		uint32_t color;
		uint16_t glyphIndex;
		uint16_t layerIndex;
		uint16_t offsetU;
		uint16_t sizeU;
	};

* position：字符在画布中摆放位置的左上角坐标，单位为像素
* scale：字符的缩放比，值为 `textPrinter` 字高相对于字体贴图字高的比值
* color：字符的颜色，在小端机上，从低8位到高8位分别对应RGBA
* glyphIndex：字形在字体中的索引，无实际用途（该字段是历史遗留，不影响结构体的大小因而姑且保留），着色器无需读取它
* layerIndex：字符所在的贴图数组图层索引
* offsetU：该字符在其所在的贴图数组图层中，其包围盒左边界的位置，单位为像素
* sizeU：该字符的贴图在画布中的宽度，单位为像素

字符在画布中的摆放位置的右下角坐标为：

	{ position.x + scale * sizeU,
	  position.y + scale * font.ImageLayerHeight() }

采样贴图时使用的UVW坐标，左上角为：

	{ float(offsetU) / font.ImageLayerWidth(), 0.f, layerIndex }

采样贴图时使用的UVW坐标，右下角为：

	{ float(offsetU + sizeU) / font.ImageLayerWidth(), 1.f, layerIndex }

具体的着色器代码可以参考[我的GLSL代码](https://github.com/EasyVulkan/VKEncapsulation/tree/main/VKEncapsulation/shader/TextRendering)，管线组装参考我的[Vulkan代码](https://github.com/EasyVulkan/VKEncapsulation/blob/main/VKEncapsulation/Sample/ChimeImpl.h#L301)，有不懂之处请在[Discussions](https://github.com/EasyVulkan/Chime_TrueType/discussions)提问。

## 全局

### API特定类型

#### `API_SPECIFIC class apiData_font`

需要你自行实现该类型：要求包含图形API相关的数据，通常是贴图相关的handle。<br>
若不需要图形API调用，留空即可。

#### `API_SPECIFIC class apiData_textPrinter`

需要你自行实现该类型：要求包含图形API相关的数据，通常是顶点缓冲区的handle。<br>
若不需要图形API调用，留空即可。

对于Vulkan、DX12等“录制命令，然后再提交”的API，会有以下难点：

* 由于录制命令时已经确定了顶点缓冲区的handle，在录制命令过程中顶点数量超出当前容量时，需要分配新的顶点缓冲区，这就需要实现一个 `apiData_textPrinter` 的线性容器或链表。

线性容器适用于每个顶点缓冲区具有固定大小的情况，此时完成一次文本绘制可能需要调用多次绘制命令 ，更新缓冲区数据也需要多次数据转移。<br>
链表适用于新顶点缓冲区总是与当前容量一致的情况，每次绘制命令提供的缓冲区都是最新的，这又会产生以下难点：

* 在当前帧不需要扩容时，应将链表中旧的 `apiData_textPrinter` 实例销毁，注意需要确保所有涉及旧顶点缓冲区的命令已执行完毕。

对于使用链表的实现，你可以参考[我的Vulkan代码](https://github.com/EasyVulkan/VKEncapsulation/blob/main/VKEncapsulation/Sample/ChimeImpl.h#L184)，有不懂之处请在[Discussions](https://github.com/EasyVulkan/Chime_TrueType/discussions)提问。

### API特定函数

#### `API_SPECIFIC void InitializeTextPrinter();`

需要你自行实现该函数：要求创建与具体 `font` 和 `textPrinter` 实例无关的API相关数据，通常包含图形管线，Vulkan和DX12会有采样器和描述符。<br>
若不需要图形API调用，留空即可。