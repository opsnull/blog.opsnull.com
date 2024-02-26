---
title: "LCD 中英文字体制作"
author: ["张俊(zj@opsnull.com)"]
date: 2024-02-19T00:00:00+08:00
lastmod: 2024-02-26T11:37:21+08:00
tags: ["rust", "esp32"]
categories: ["rust", "esp32"]
draft: false
series: ["rust-esp32"]
series_order: 1
---

一个 LCD 小游戏：
<https://github.com/georgik/esp32-spooky-maze-game/blob/main/esp32-s3-usb-otg/src/main.rs>

-   使用的是 mipidsi::Builder::st7789，需要修改为 st7735s，支持按键。
-   xtensa-esp32s3-none-elf non_std 应用。

另外一个 LCD 项目样例：
<https://github.com/georgik/esp32-rust-multi-target-template/blob/main/esp32-s3-usb-otg/src/main.rs>

ESP Display Interface with SPI and DMA：
<https://github.com/georgik/esp-display-interface-spi-dma/blob/main/README.md>

其他字体：<https://github.com/olikraus/u8g2>

embedded-graphics 支持 BDF 和 MonoFont 字体:

-   优先选择 BDF 字体, 特别是是中英文混合显示场景, MonoFont 由于中英文等宽, 会导致英文字体显示间距过大.
-   优先选择 embedded-graphics/bdf 方案.

The draw method for text drawables returns `the position of the next character`. This can be used to
combine text with different character styles on a single line of text.

```rust
// https://docs.rs/embedded-graphics/latest/embedded_graphics/text/index.html#examples
// Create a small and a large character style.
let small_style = MonoTextStyle::new(&FONT_6X10, Rgb565::WHITE);
let large_style = MonoTextStyle::new(&FONT_10X20, Rgb565::WHITE);

// Draw the first text at (20, 30) using the small character style.
let next = Text::new("small ", Point::new(20, 30), small_style).draw(&mut display)?;

// Draw the second text after the first text using the large character style.
let next = Text::new("large", next, large_style).draw(&mut display)?;
```


## <span class="section-num">1</span> embedded-graphics/bdf {#embedded-graphics-bdf}

<https://github.com/embedded-graphics/bdf/tree/master>

该库可以从 bdf 字体文件中生成 embedded-graphics 能使用的：

1.  BDF Font：需要配合使用[embedded-graphics/bdf/eg-bdf](https://github.com/embedded-graphics/bdf/blob/master/eg-bdf/src/text.rs) 库中提供的 BdfFont/BdfGlyph/BdfTextStyle 类型；
2.  MonoFont：embedded-graphics 可以直接导入使用；

先使用 otf2bdf 工具将 TTF（truetype font） 字体转换为 BDF（Bitmap Distribution Format） 类型：

-   对于 X11 使用的 pcf 字体格式，使用 pcf2bdf 工具将它转换为 bdf 格式；
-   转换时 -p 参数指定生成的 BDF 字体大小;

<!--listend-->

```shell
brew install otf2bdf
otf2bdf  -p 12 -r 75 -o LXGWWenKai_Regular.bdf  ~/Library/Fonts/LXGWWenKai-Regular.ttf
```

然后使用 eg-font-converter lib 来生成 BDF 或 MonoFont 字体:

-   两者都至少包含 2 个文件: bitmap data 文件(.data) 和需要 include!() 到引用的 rs 文件;
-   MonoFont 的 bitmap 文件是从 png 图片生成的, 故 MonoFont 还会生成 png 文件;

生成 BDF 字体示例:

```rust
// /Users/zhangjun/codes/rust/bdf/eg-font-converter/src/bin/my-bdf-font.rs
zj@a:~/codes/rust/bdf/eg-font-converter$ mkdir output
zj@a:~/codes/rust/bdf/eg-font-converter$ ls
Cargo.lock  Cargo.toml  output/  src/  tests/
zj@a:~/codes/rust/bdf/eg-font-converter$ cat src/bin/my-bdf-font.rs
use eg_font_converter::{FontConverter};

// 中文 unicode 字体范围: https://en.wikipedia.org/wiki/CJK_Unified_Ideographs_(Unicode_block)
// 使用 .glyphs() 来指定要生成的字符范围
fn main() {
    let font = FontConverter::new("/Users/zhangjun/docs/LXGWWenKai_Regular.bdf", "BDF_LXGWWenKai_Regular_FONT")
        //.glyphs('a'..='z')
        .glyphs('\u{0000}'..='\u{007F}') // ASCII
        .glyphs('\u{4E00}'..='\u{9FFF}') // 常用中文字体范围
        .glyphs('\u{2E80}'..='\u{2EF3}') // 常见繁体字范围
        .missing_glyph_substitute('?') // 替代字符
        .convert_eg_bdf()
        .unwrap();
    font.save("output/").unwrap();  // 结果写入 output 目录下, 存入 new(bdf_file, name) 的 小写 name.rs 文件中.
}

// 运行该程序
zj@a:~/codes/rust/bdf/eg-font-converter$ cargo run --bin my-bdf-font
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.32s
     Running `/Users/zhangjun/codes/rust/bdf/target/debug/my-bdf-font`

zj@a:~/codes/rust/bdf/eg-font-converter$ ls output/
bdf_lxgwwenkai_regular_font.data  bdf_lxgwwenkai_regular_font.rs
zj@a:~/codes/rust/bdf/eg-font-converter$ ls -l output/
total 4.7M
-rw-r--r-- 1 zhangjun 310K  2 17 15:34 bdf_lxgwwenkai_regular_font.data
-rw-r--r-- 1 zhangjun 4.4M  2 17 15:34 bdf_lxgwwenkai_regular_font.rs
zj@a:~/codes/rust/bdf/eg-font-converter$ head -20 output/bdf_lxgwwenkai_regular_font.rs
const BDF_LXGWWenKai_Regular_FONT: ::eg_bdf::BdfFont = {
    const fn rect(
        x: i32,
        y: i32,
        width: u32,
        height: u32,
    ) -> ::embedded_graphics::primitives::Rectangle {
        ::embedded_graphics::primitives::Rectangle::new(
            ::embedded_graphics::geometry::Point::new(x, y),
            ::embedded_graphics::geometry::Size::new(width, height),
        )
    }
    ::eg_bdf::BdfFont {
        data: include_bytes!("bdf_lxgwwenkai_regular_font.data"),
        replacement_character: 63usize,
        ascent: 12u32,
        descent: 3u32,
        glyphs: &[
            BdfGlyph {
                character: '\0',
zj@a:~/codes/rust/bdf/eg-font-converter$
```

使用：

```shell
zj@a:~/codes/rust/bdf/eg-font-converter$ cp output/bdf_lxgwwenkai_regular_font.* ~/codes/esp32/st7735-lcd-examples/esp32c3-examples/src/

# 添加本地依赖
gzj@a:~/codes/esp32/st7735-lcd-examples/esp32c3-examples$ grep eg Cargo.toml
eg-bdf = { path = "../../../rust/bdf/eg-bdf/" } // clone https://github.com/embedded-graphics/bdf 的本地目录
```

代码举例:

```rust

use eg_bdf::{BdfGlyph, BdfTextStyle};

include!("./bdf_lxgwwenkai_regular_font.rs");

fn main() {
    let text = "Happy\u{AD}Loong\u{AD}Year!\
      Happy Hacking!\
      Happy Loong Year!\u{A0}新年快乐!\u{A0}-from Rust ESP32";
    let bdf_style = BdfTextStyle::new(&BDF_LXGWWenKai_Regular_FONT, Rgb565::WHITE);
    let textbox_style = TextBoxStyleBuilder::new()
      .line_height(LineHeight::Pixels(12)) // 字体高度与生成 otf2bdf  -p 12 一致
      .alignment(HorizontalAlignment::Justified)
      .paragraph_spacing(0)
      .build();
    let mut bounds = Rectangle::new(Point::new(0, 0), Size::new(128, 160));
    let mut text_box = TextBox::with_textbox_style(text, bounds, bdf_style, textbox_style);
    let next = text_box.draw(&mut display).unwrap();
}
```
