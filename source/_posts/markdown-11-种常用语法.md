---
title: markdown 11 种常用语法
date: 2018-01-15 08:50:38
tags:
---
</br>
<font color=green>**现在是我在学习Markdown时做的笔记。学完这些Markdown的基本使用已经不成问题.**</font>

- - -
<font color=black>**1\. 标题设置（让字体变大，和word的标题意思一样）**</font>

```
在Markdown当中设置标题，有两种方式：
第一种：通过在文字下方添加“=”和“-”，他们分别表示一级标题和二级标题。
第二种：在文字开头加上 “#”，通过“#”数量表示几级标题。（一共只有1~6级标题，1级标题字体最大）
```
<font color=black>**2\. 块注释（blockquote）**</font>
```
通过在文字开头添加“>”表示块注释。（当>和文字之间添加五个blank时，块注释的文字会有变化。）
```
<font color=black>**3\. 斜体**</font>
```
将需要设置为斜体的文字两端使用1个“*”或者“_”夹起来
```
<font color=black>**4\. 粗体**</font>
```
将需要设置为斜体的文字两端使用2个“*”或者“_”夹起来
```
<font color=black>**5\. 无序列表**</font>
```
在文字开头添加(*,+,and-)实现无序列表。但是要注意在(*,+and-)和文字之间需要添加空格.(建议：一个文档中只是用一种无序列表的表示方式)
```
<font color=black>**6\. 有序列表**</font>
```
使用数字后面跟上句号。（还要有空格）
```
<font color=black>**7\. 链接（Links）**</font>
```
Markdown中有两种方式，实现链接，分别为内联方式和引用方式。

内联方式：This is an [example link](http://example.com/).
引用方式：
I get 10 times more traffic from [Google][1] than from [Yahoo][2] or [MSN][3].  

[1]: http://google.com/        "Google" 
[2]: http://search.yahoo.com/  "Yahoo Search" 
[3]: http://search.msn.com/    "MSN Search"

```
<font color=black>**8\. 图片（Images）**</font>
```
图片的处理方式和链接的处理方式，非常的类似。
内联方式：![alt text](/path/to/img.jpg "Title")
引用方式：
![alt text][id] 

[id]: /path/to/img.jpg "Title"
```
<font color=black>**9\. 代码（HTML中所谓的Code）**</font>
```
实现方式有两种：
第一种：简单文字出现一个代码框。使用`<blockquote>`。（`不是单引号而是左上角的ESC下面~中的`）
第二种：大片文字需要实现代码框。使用Tab和四个空格。
```
<font color=black>**10\. 脚注（footnote）**</font>
```
实现方式如下：
hello[^hello]


[^hello]: hi
```
<font color=black>**11\. 下划线**</font>
```
在空白行下方添加三条“-”横线。（前面讲过在文字下方添加“-”，实现的2级标题）
```
