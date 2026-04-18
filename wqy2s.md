# 文泉驿（wqy）字体指定字符转点阵实现文档
## 一、实现原理
### 1.1 核心逻辑
将文泉驿字体中的指定字符转换为点阵数据，核心是通过字体渲染库加载字体文件，将字符渲染为黑白二值位图，再按嵌入式屏显通用规则（行优先、8位打包、高位在前）提取像素数据，最终得到可直接用于屏显的点阵字节数据。

### 1.2 关键步骤
1. **字体加载**：读取文泉驿字体文件（.ttf/.otf格式），解析字体的字形数据；
2. **字符渲染**：将指定字符渲染为固定尺寸的黑白二值位图（如16x16、24x24），确保字符居中显示；
3. **点阵提取**：按行优先顺序遍历位图像素，将每8个像素打包为1字节（不足8位补0），支持像素反转（适配屏显“1=灭、0=亮”的通用规则）；
4. **格式适配**：输出十六进制点阵数据，兼容嵌入式开发的字体表格式。

### 1.3 适配场景说明
- **字符范围**：支持ASCII（32-128）、中文（农历/节气/生肖/节日等），中文需逐字符处理；
- **点阵格式**：行优先、高位在前打包，可通过参数控制像素反转；
- **应用场景**：嵌入式屏显（如EPD电子纸、LCD屏）的自定义字体表开发。

## 二、代码实现
### 2.1 环境准备
安装依赖库：
```bash
# Pillow（简易版，推荐入门）
pip install pillow
# FreeType（底层版，精准控制）
pip install freetype-py
```

### 2.2 方案1：Pillow实现（简易版）
```python
from PIL import Image, ImageDraw, ImageFont

def wqy_char_to_bitmap(char, font_path, font_size, reverse=False):
    """
    将单个字符转换为点阵数据（Pillow版）
    :param char: 待转换的字符（如'正'、'A'、'℃'）
    :param font_path: 文泉驿字体文件路径（如wqy-microhei.ttf）
    :param font_size: 字体大小（对应点阵尺寸，如16→16x16点阵）
    :param reverse: 像素反转（True=1表示灭，0表示亮；False=1表示亮，0表示灭）
    :return: 点阵数据（列表，每个元素是1字节，行优先打包）、点阵宽度/高度
    """
    # 1. 创建空白画布（模式为1：黑白二值图，背景为0（黑））
    img = Image.new('1', (font_size, font_size), 0)
    draw = ImageDraw.Draw(img)
    
    # 2. 加载文泉驿字体
    try:
        font = ImageFont.truetype(font_path, font_size)
    except Exception as e:
        raise ValueError(f"加载字体失败：{e}")
    
    # 3. 渲染字符（居中显示，填充为1（白））
    bbox = draw.textbbox((0, 0), char, font=font)
    char_width = bbox[2] - bbox[0]
    char_height = bbox[3] - bbox[1]
    # 居中绘制
    x = (font_size - char_width) // 2
    y = (font_size - char_height) // 2
    draw.text((x, y), char, font=font, fill=1)
    
    # 4. 提取点阵数据（行优先，8位打包为1字节）
    bitmap = []
    width, height = img.size
    for y in range(height):
        byte = 0
        for x in range(width):
            # 获取像素值（0/1），reverse则取反
            pixel = img.getpixel((x, y)) ^ reverse
            # 按位打包（高位在前）
            byte = (byte << 1) | pixel
            # 每8个像素存1字节
            if (x + 1) % 8 == 0:
                bitmap.append(byte)
                byte = 0
        # 处理行尾不足8位的情况（补0）
        if width % 8 != 0:
            byte = byte << (8 - width % 8)
            bitmap.append(byte)
    
    return bitmap, width, height

def bitmap_to_image(bitmap, width, height, save_path):
    """将点阵数据转回图片（调试用）"""
    img = Image.new('1', (width, height), 0)
    draw = ImageDraw.Draw(img)
    idx = 0
    byte_idx = 0
    for y in range(height):
        for x in range(width):
            # 从打包字节中提取单个像素
            if byte_idx == 0:
                current_byte = bitmap[idx]
            pixel = (current_byte >> (7 - (x % 8))) & 1
            draw.point((x, y), fill=pixel)
            byte_idx += 1
            if byte_idx == 8:
                byte_idx = 0
                idx += 1
    img.save(save_path)

# -------------------------- 测试示例 --------------------------
if __name__ == "__main__":
    # 请替换为你的文泉驿字体实际路径
    WQY_FONT_PATH = "wqy-microhei.ttf"  # 文泉驿微米黑
    # 待转换的字符（覆盖ASCII、农历、节气、生肖等场景）
    test_chars = [
        # ASCII（32-128）
        'A', '1', ' ', '℃', '!',
        # 农历/节气
        '正', '初', '小', '寒',
        # 生肖/天干地支
        '鼠', '甲', '子',
        # 节日
        '春', '国'
    ]
    
    for char in test_chars:
        # 转换为16x16点阵（嵌入式常用尺寸）
        bitmap, w, h = wqy_char_to_bitmap(char, WQY_FONT_PATH, 16)
        # 打印结果
        print(f"字符：{char}")
        print(f"点阵尺寸：{w}x{h}")
        print(f"点阵数据（十六进制）：{[hex(b) for b in bitmap]}")
        # 可选：生成点阵预览图
        bitmap_to_image(bitmap, w, h, f"{char}_16x16.bmp")
        print("-" * 50)
```

### 2.3 方案2：FreeType实现（底层精准控制）
```python
import freetype

def wqy_char_to_bitmap_freetype(char, font_path, font_size, reverse=False):
    """
    FreeType实现字符→点阵（更底层，精准控制）
    :param char: 单个字符（一次仅处理1个字符）
    :param font_path: 文泉驿字体路径
    :param font_size: 字体大小（点阵尺寸）
    :param reverse: 像素反转（True=1灭0亮）
    :return: 点阵数据（字节列表）、点阵宽度/高度
    """
    # 1. 加载字体
    face = freetype.Face(font_path)
    face.set_pixel_sizes(0, font_size)  # 设置像素尺寸（宽0=自适应，高=font_size）
    
    # 2. 加载字符的字形（单色渲染）
    char_code = ord(char)
    face.load_char(
        char_code, 
        freetype.FT_LOAD_RENDER | freetype.FT_LOAD_MONOCHROME  # 单色渲染
    )
    glyph = face.glyph
    
    # 3. 提取位图原始数据
    bitmap = glyph.bitmap
    width = bitmap.width
    height = bitmap.rows
    pixel_data = bitmap.buffer  # 原始像素数据（0=黑，255=白）
    
    # 4. 转换为二值点阵（8位打包，高位在前）
    packed_data = []
    for y in range(height):
        byte = 0
        for x in range(width):
            # 转为0/1二值像素
            pixel = 1 if (pixel_data[y * width + x] > 128) else 0
            if reverse:
                pixel = 1 - pixel
            # 按位打包
            byte = (byte << 1) | pixel
            # 每8个像素存1字节
            if (x + 1) % 8 == 0:
                packed_data.append(byte)
                byte = 0
        # 处理行尾不足8位，补0
        if width % 8 != 0:
            byte = byte << (8 - width % 8)
            packed_data.append(byte)
    
    return packed_data, width, height

def bitmap_to_image(bitmap, width, height, save_path):
    """将点阵数据转回图片（调试用）"""
    from PIL import Image, ImageDraw
    img = Image.new('1', (width, height), 0)
    draw = ImageDraw.Draw(img)
    idx = 0
    byte_idx = 0
    for y in range(height):
        for x in range(width):
            if byte_idx == 0:
                current_byte = bitmap[idx]
            pixel = (current_byte >> (7 - (x % 8))) & 1
            draw.point((x, y), fill=pixel)
            byte_idx += 1
            if byte_idx == 8:
                byte_idx = 0
                idx += 1
    img.save(save_path)

# -------------------------- 测试示例 --------------------------
if __name__ == "__main__":
    WQY_FONT_PATH = "wqy-microhei.ttf"  # 替换为实际路径
    # 待转换字符（单字符处理）
    test_chars = ['正', '寒', '鼠', '甲', 'A', '℃', ' ', '1']
    
    for char in test_chars:
        bitmap, w, h = wqy_char_to_bitmap_freetype(char, WQY_FONT_PATH, 16)
        # 打印结果
        print(f"字符：{char}")
        print(f"点阵尺寸：{w}x{h}")
        print(f"点阵数据（十六进制）：{[hex(b) for b in bitmap]}")
        # 生成预览图
        bitmap_to_image(bitmap, w, h, f"{char}_freetype_16x16.bmp")
        print("-" * 50)
```

## 三、使用说明
### 3.1 字体文件准备
1. **获取路径**：
   - 官方下载：[文泉驿字体官网](https://wenq.org/)；
   - Linux系统内置路径：`/usr/share/fonts/wenquanyi/`；
2. **替换路径**：代码中`WQY_FONT_PATH`需替换为实际的文泉驿字体文件路径（如`wqy-microhei.ttf`、`wqy-zenhei.ttf`）。

### 3.2 关键参数调整
| 参数        | 说明                                                                 |
|-------------|----------------------------------------------------------------------|
| `font_size` | 点阵尺寸（如16→16x16点阵，24→24x24点阵，适配嵌入式屏显分辨率）|
| `reverse`   | 像素反转：`True`（1=灭、0=亮，嵌入式通用），`False`（1=亮、0=灭）|
| 字符输入    | 中文需**单字符处理**（如“正月”拆分为“正”+“月”），ASCII可直接输入单字符 |

### 3.3 结果验证
1. **控制台输出**：打印字符、点阵尺寸、十六进制点阵数据，可直接复制到嵌入式字体表（如`fonts.c`）；
2. **预览图验证**：运行代码后生成`字符_16x16.bmp`文件，可查看点阵是否正确。

### 3.4 多字符拼接
若需渲染“正月初一”等字符串，需循环处理每个字符，再按顺序拼接点阵数据，示例：
```python
def str_to_bitmap(string, font_path, font_size, reverse=False):
    """字符串转点阵（逐字符拼接）"""
    all_bitmap = []
    for char in string:
        bitmap, _, _ = wqy_char_to_bitmap(char, font_path, font_size, reverse)
        all_bitmap.extend(bitmap)
    return all_bitmap

# 调用示例
string_bitmap = str_to_bitmap("正月初一", "wqy-microhei.ttf", 16, reverse=True)
print(f"字符串'正月初一'点阵数据：{[hex(b) for b in string_bitmap]}")
```

## 四、注意事项
1. **字符编码**：确保Python文件编码为UTF-8，避免中文乱码；
2. **字体兼容性**：文泉驿字体需支持目标字符（如文泉驿微米黑/正黑覆盖绝大部分中文、ASCII）；
3. **嵌入式适配**：
   - 点阵数据需按屏显驱动要求调整（如低位在前、列优先，可修改代码中`byte = (byte << 1) | pixel`为`byte = byte | (pixel << (7 - (x % 8)))`）；
   - 不足8位补0的逻辑需与屏显驱动一致。
4. **依赖安装**：若`freetype-py`安装失败，可先安装系统依赖（Linux）：`sudo apt-get install libfreetype6-dev`。
