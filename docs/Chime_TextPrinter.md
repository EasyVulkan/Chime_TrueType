# **Chime_TextPrinter.h** Reference Page

**Chime_TextPrinter.h** 包含了API无关的文本排版、打印功能，并声明了一系列用于对接图形API的函数/类型接口以便实现高效的文本渲染。

## `class font`

`ttfLoader` 是一个API无关的类型，由 `font` 负责与图形API对接。

### 构造函数

#### 直接读取文件

	font(const char* filepath_ttf, float fontHeight, int8_t sdfPadding, bool generateMipmap);

	font(const char* filepath_preLoaded, bool generateMipmap);

* filepath：TrueType文件路径
* fontHeight：字高
* sdfPadding：负数则生成非SDF；为0时生成非SDF但当做SDF使用；正数时生成SDF并指定自字体轮廓向外扩张的距离
* generateMipmap：是否生成mipmap，建议为true
* filepath_preLoaded：PNG或ttfL文件路径

该函数会创建一个 `ttfLoader` 实例以读取文件。

#### 从ttfLoader取得数据

	font(ttfLoader& loader, bool generateMipmap);

    font(const ttfLoader& loader, bool generateMipmap);

* loader：已包含字形贴图和相关信息的 `ttfLoader` 实例

构造函数不返值，可以通过以下方式确认函数是否执行成功：

* 若需要创建图形API相关数据，检查 `pApiData` 字段的值（调用成员函数 `operator->()` ）。
* 若不需要创建图形API相关数据，除非内存极其吃紧，函数执行成功与否取决于 `ttfLoader` 读取文件成功与否，那么分两步走，不要用 `font` 直接读取文件。

### API特定函数

    API_SPECIFIC void CreateApiData(const uint8_t* pImageData, bool generateMipmap);

* pImageData：指向图像数据的指针
* generateMipmap：是否生成mipmap

需要你自行实现该函数：要求动态分配 `apiData_font` 类型的实例，将其地址返回给智能指针类型的字段 `pApiData` ，创建失败时返回 `nullptr` 。<br>
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

TODO

### 打印文本

延迟打印有以下作用：

* 用于需要根据文本区域大小调整位置的情形
* 用于需要对已生成的顶点做手脚的情形，比如 `textPrinter::characterVertex::layerIndex` 通常只会使用低11位，可以用剩下的位指定一些渲染效果
* 对于调用图形API的情况，可以在生成当前帧所需的所有顶点后再确定顶点缓冲区大小

### 更新API数据

TODO

### API特定函数

TODO

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