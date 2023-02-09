---
title: Unicode, UTF-8, 以及 UTF-16
categories: 
- 技术笔记
tags: 
- unicode
date: 2018-6-24
updated: 2018-6-24
---
# Unicode, UTF-8, 以及 UTF-16

对不起，但是必须要说，要想完全地搞懂 Unicode 编码，光看这篇文章是肯定不够的，你要自己去维基上看一系列文章。首推 [Character encoding](https://en.wikipedia.org/wiki/Character_encoding) 条目。

## 基本术语

来自维基百科的 [character encoding（字符编码）](https://en.wikipedia.org/wiki/Character_encoding) 词条：

Terminology related to code unit:

- A *character* is a minimal unit of text that has semantic value.
- A *character set* is a collection of characters that might be used by multiple languages.

*Example:* The Latin character set is used by English and most European languages, though the Greek character set is used only by the Greek language.

- A *coded character set* is a character set in which each character corresponds to a unique number.
- A *code point* of a coded character set is any allowed value in the character set.
- A *code unit* is a bit sequence used to encode each character of a repertoire within a given encoding form.

几个关键术语对应的翻译：code point，码点；code unit，编码单元。所以，一个字符（character）是具有语义的最小文本单元，更多的是一个语言学的术语；一个*字符集*（character set）是可能被好几种语言使用的字符的集合，比如拉丁语字符集同时被英语和大多数欧洲语言使用。一个*编码字符集*（coded character set）是一个字符集，其中每一个字符都对应于一个唯一的数字。一个编码字符集的*码点*（code point）是指这个字符集中所有的数字值，每一个码点对应一个字符。一个*编码单元*是一个在给定的字符编码表下，给字符编码的字节序列。

## Unicode 具体实现过程

### Unicode 四层模型

如果你能看到这，恭喜，接下来就会简单很多。Unicode 与传统的诸如 ASCII，摩斯电码用的一个重要不同在于，它们都是把字符（character）与字节序列直接对应。而对于 Unicode 来说，它会经过以下几个模型的处理：

**character --(CCS)--> code point --(CEF)--> code unit --(CES)--> byte stream**

一个字符（character）经过编码字符集（CCS, coded character set）的映射，转化为码点（code point）。码点为**十六进制**，像这样：`U+20AC`，转化为字符是欧元的标志：`€`。

码点经过**字符编码表**（CEF, character encoding form）的处理，转化为编码单元（code unit）。编码单元仍然是**十六进制**。举例来说，同一个字符`a`，它的编码单元在 UTF-32 中是`00000061 `，在 UTF-16中是 `0061`，在 UTF-8 中是 `61`。

最后，编码单元被 **CES** (Character Encoding Scheme) 转化为在机器中储存的二进制字节。”CES“ 是什么呢，举例来说，我们平常说的 UTF-8，UTF-16 其实都是 CES。

### 从码点到 UTF-16

从`U+0000`到`U+FFFF`为基本多语种平面（BMP），可以用两个字节表示。对于高于这个范围的，用 ”代理对“ 来间接表示。低代理项的范围为`U+D800`至 `U+DBFF `，高代理项为`U+DC00`至`U+DFFF `，一共是2048个位置。这些位置其实是位于 BMP 的保留范围。

从码点到 UTF-16 编码单元的解码过程：

1. 把码点减去`0x10000`
2. 对于高代理项（high surrogate）：
   1. 地板除，除以`0x400`
   2. 加上`0xD800 `
3. 对于低代理项（low surrogate）：
   1. 取余数，取除以`0x400`后的余数
   2. 加上`0xDC00`

写成直观公式：

```
difference = codepoint - 0x10000
H = difference // 0x400 + 0xD800
L = difference % 0x400 + 0xDC00
```

从 UTF-16 编码单元到码点的编码可以反向推算。

```
codepoint = difference + 0x10000
difference = (H - 0xD800) * 0x400 + (L - 0xDC00)
```



#### UTF-16 与 UCS-2 兼容问题

Unicode 标准规定了从`U+D800`到`U+DFFF`为 reserved range，保留位。UTF-16 就是利用这一保留区段来对基本平面 BMP 以外的辅助平面进行映射。需要注意的是，虽然标准规定了这一区段不可使用，但是实际使用中，在 UCS-2 的年代是用到了这一区段的。因此在许多编码解码的应用中，如果遇到保留区段的码点，会先对这个做检查，看该码点后是否能构成代理对，而不是直接按标准，把这个定义为编码错误。

### 从码点到 UTF-8