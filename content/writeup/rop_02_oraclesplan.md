---
title: "[Writeup] ROP: 02 - Oracle's Plan"
date: 2020-03-17T22:00:00+07:00
categories: [writeup]
tags: [rop, web, ctf, writeup]
---

**DISCLAIMER: game.rop.sh was discontinued and is no longer available.**

**Challenge Name:** Oracle's Plan  
**Type:** Web  
**Score:** 100 pts  
**Link:** [https://game.rop.sh](https://game.rop.sh/task) 

![Challenge Description](/images/rop/02/rop_02_challenge.png)*Oracle's Plan challenge description*

<!--more-->

The challenge provides us a link to a website, which looks like this:

![Challenge Webpage](/images/rop/02/rop_02_webpage.png)*Webpage of the challenge*

It is a simple page with an image and a line of text. The first thing to do, of course, is to view the page source.

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<title> Anti-Robotics </title>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="ที่ประลองวิชาสำหรับคนอยากลองแฮก">
    <meta name="author" content="Xelenonz">
	<style>
	@font-face {
font-family: Aldrich;
src: url('../../css/aldrich.woff') format('woff');
}
body{
	font-family: Aldrich;
	font-size: 25px;
}
img {
    display: block;
    margin-left: auto;
    margin-right: auto;
}
	</style>
</head>
<body>
	<img  src="header.png" />
	<p style="text-align:center">Get Away from my place ROBOT!!!</p>
	<!-- Robots can't find my plan, I put in my safe place!! if you are human you will know-->
</body>
</html>
```

The comment on line 29 hints us that the robots would not know about the page. The bots (or the crawlers) would not access the pages specified within "robots.txt", so that is where we would have a look.

![robots.txt](/images/rop/02/rop_02_robots.png)*robots.txt*

Surprisingly(?), there's a suspicious file name inside, and the flag can be found in that file.

![robots.txt](/images/rop/02/rop_02_flag.png)*Flag*

This is an easy challenge. Anyone would be able to solve it if the person knows about robots.txt file. However, if not, it would be surprisingly hard.
