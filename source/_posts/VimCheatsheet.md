---
title: Vim Cheat Sheet
categories: 
- 技术笔记
tags: 
- vim
date: 2018-7-21 22:54
updated: 2018-7-21 22:54
---

> 最近用上了 WSL，巨硬居然在 Win 上允许使用 Linux了
> 修改文件不可避免似乎就要学会 Vim 啊，后来才知道自带 nano
## Motion

`h`, `l`: move left and right

`j`, `k`: move down and up



`w`: next **beginning** of word

`e`, `ge`: **current, last** **end** of word

`b`, `B`: **previous beginning** of word and **WORD**



`0`, `^`: to the **beginning of line**

`$`: to the **end** of line



`gg`, `G`: **beginning**, **end** of **file** 

`nG`: line n; `:set nu` to turn on line num



`f<char>`, `F<char>`: **search** forward, backward



## Edit

`cw`: **C**hange current word that the cursor's at

`cc`: *delete* the whole line and enter INSERT mode

`C`: delete till the **end of line**, enter INSERT mode



`r<char>`: **R**eplace cursor with \<char\>

`R`: replace successively, with each\<char\>  typed; `Esc` to exit



`un` **U**ndo for **n** times

`U` undo all changes in current **line**

`Ctrl` + `r`: redo



`>>`, `<<`: tab in NORMAL mode

`:set shiftwidth?`



`:ce` **center** the line

`:ri`, `:le`: to the **right**, **left**



### Search

`/`, `?`: search **next**, **last**

`n`, `N`: **next**, **last** result



## Insert

`i`, `a`: insert **at** and **append** cursor position

`I`, `A`: insert at the **beginning** and the **end** of current line

`o`, `O`: **O**pen a new line **after** and **before** current line, and enter INSERT mode



## Copy, Paste, and Cut

`yy`: **Y**ank(copy) **whole line**

`yw`, `ynw`: copy one **word**, **n** words

`y^`, `y0`: copy till the **beginning of line**, current cursor **not** included

`y$`: copy till the **end**, char where the cursor's at **included**

`yG`, `y1G`: copy till **end**, **beginning** of **file**



`p`: (lowercase) paste on **next line**

`P`: (uppercase) paste **last line**



`dd`: cut

`ddp`: **swap** this and **next** line



## Delete

`x`: **D**elete char **under** cursor

`X`: delete char **before** cursor

`dd`: delete the entire line

`dw`: delete word *after* cursor; `w` + `dw`



`D`: delete till the end of **line**

`d^`: delete till the beginning

`d1G`, `dG`: delete till the **beginning** and **end** of **file**



## Repeat

`.`: repeat last command (under NORMAL mode)

`N<command>`: e.g. 2dd, 10x

`dnw`: delete a word **n** times 



## Exit

`:q`: quit

`:wq`, `:x`: save and exit

`:wq!`: **force** to save and quit

`:w`, `:saveas`

`Shift` + `zz`: under NORMAL mode, save and exit