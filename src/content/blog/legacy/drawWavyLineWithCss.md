---
# layout: post
title: drawWavyLineWithCss
description: 如何用css画出波浪线
pubDatetime: 2018-06-21 15:26:29
author: codinglobster
tags:
  - css
---

## 前言

atom 编辑器是我的主力编辑器，日常的美化和配置是必然的操作，一日，出于对原生目录树样式不满的我决定通过内置的开发者工具对其修整一下，突然发现部分文件名下居然有**红色的下划线**,本来以为是贴图实现的效果，没想到用工具看了下居然是用`background-img`配合`linear-gradient`来实现的，借这个机会可以顺便复习下 css3 了。

## 效果

![redline](./redline.png)

## 实现代码

```css
.block {
  height: 200px;
  background-image: linear-gradient(
      45deg,
      transparent 65%,
      #ef5350 80%,
      transparent 90%
    ), linear-gradient(135deg, transparent 5%, #ef5350 15%, transparent 25%),
    linear-gradient(135deg, transparent 45%, #ef5350 55%, transparent 65%),
    linear-gradient(45deg, transparent 25%, #ef5350 35%, transparent 50%);
  background-size: 8px 2px;
  background-repeat: repeat-x;
  background-position: bottom;
}
```

## liner-gradient

这是我第一次知道 background-image 可以使用多个线性渐变，所以抽丝剥茧，看看线性渐变的实现细节吧。

`background-image: linear-gradient(45deg, transparent 65%, #EF5350 80%, transparent 90%)`的效果：

![redline](./1.png)

可以看到在以轴线 45 度，从 65%~80%开始第一波渐变，从透明到红色，再从红色到透明到 90%,就实现了类似被放大了很多倍的红色细线，但经过抗锯齿的样子了。

![redline](./2.png)

四个线性渐变相互交错，就形成了红色波浪线的雏形

## background-size

设定了这个参数，表示当前的填充内容会以这个尺寸进行填充，将渐变放小到这个尺寸，几乎就看不出像素了。

## background-repeat

设定为只在 x 轴重复渲染，形成长细线的样子。

## background-position

将图形置于文字底部，就形成了类似语法纠错的下滑线了。
