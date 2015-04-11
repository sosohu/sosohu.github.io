---
layout: post
time: 2014-3-9
title: Xdotool工具
category: Tool
keywords: Tool, Xdotool
tags: X11
description: 使用Xdotool模拟X11图形系统上的操作
---

##什么是Xdotool
Xdotool lets you simulate keyboard input and mouse activity, move and resize windows, etc. It does this using X11's XTEST extension and other Xlib functions.

Additionally, you can search for windows and move, resize, hide, and modify window properties like the title. If your window manager supports it, you can use xdotool to switch desktops, move windows between desktops, and change the number of desktops.

For more information, please click [Xdotool Website](http://www.semicomplete.com/projects/xdotool/)

##Xdotool安装方法

*	ubuntu系统下可以直接的使用`sudo apt-get install xdotool`安装.
*	除了第一种情况外可以使用直接[下载](http://semicomplete.googlecode.com/files/xdotool-2.20110530.1.tar.gz)xdotool源码包,再编译安装.
	*	编译安装过程中会遇到缺少头文件或者依赖库,主要需要安装的有:
		*	libxkbcommon: [下载地址](http://xkbcommon.org/download/libxkbcommon-0.3.1.tar.xz)
		*	libXtst: 可以直接在源中apt-get安装(ferdora使用yum install)
	*	编译安装方法
		*	make
		*	export PREFIX=/usr(设置安装目录,默认在/usr/local)
		*	make install

##Xdotool的使用
###鼠标操作
xdotool支持很多鼠标操作,包括鼠标的移动,左击,右击,滚轮等

*	鼠标移动到x,y处: xdotool mousemove x y
*	鼠标点击右键: xdotool click 3
*	鼠标向上翻滚: xdotool click 4
*	获取鼠标位置: xdotool getmouselocation
*	...

###键盘操作
xdotool支持很多键盘操作,常用的使用如下:

*	按下p键: xdotool key p
*	按下ctrl+shift+t键: xdotool key ctrl+shift+t
*	按下p键持续1000ms: xdotool key --delay 1000  p
*	...

###窗口操作
xdotool支持很多窗口操作,包括窗口的移动,最小化等等

*	查询主文件夹窗口id: xdotool search --name "主文件夹"
*	聚焦到id为WID的窗口: xdotool windowfocus WID
*	移动id为WID的窗口左上角到x,y处: xdotool windowmove WID x y
*	改变id为WID的窗口大小为w,h: xdotool windowsize WID w h
*	最小化id为WID的窗口: xdotool windowminimize WID
*	...

###简单示例
{% highlight sh %}
	#!/bin/bash

	function open(){
		for((i=0;i<5;i++));
		do
			nautilus "/home/"
			kill -2 `ps -A | grep nautilus | cut -f-1 -d' '`
		done
	}

	#测试不断的打开和关闭文件浏览器
	open  
	nautilus "/home/"
	sleep 5
	WID=`xdotool search --name "文件浏览器" | tail -1`
	echo $WID
	xdotool windowfocus $WID

	#gen the random num in [0, max)
	function random(){
		if [[ $# -eq "0" ]]; then
			max=1000
		else
			max=$1
		fi
		rand_value=$[$RANDOM % $max]
		echo $rand_value
	}

	function drag(){
		x0=0
		y0=0
		xdotool windowmove $WID $x0 $y0
		for((i=0;i<500;i++));
		do
			#xdotool key ctrl+t
			x=$[$x0+$i]
			y=$[$y0+$i]
			xdotool windowmove $WID $x $y
			#xdotool windowminimize $WID 
		  	#sleep 4
		done
	}

	function move(){
		x1=`random`
		y1=`random`
		xdotool windowmove $WID $x1 $y1
	}

	function contineMove(){
		for((i=0;i<10000;i++));
		do
			move
			#sleep 1
		done
	}

	function resize(){
		x1=`random`
		y1=`random`
		xdotool windowsize $WID $x1 $y1
	}

	function contineResize(){
		for((i=0;i<$1;i++));
		do
			resize
		done
	}


	#拖拽窗口
	#drag   

	#移动窗口
	#contineMove

	#改变窗口大小
	#contineResize 10000
{% endhighlight %}

##官方参考手册
在有些平台,编译安装xdotool之后,只能使用xdotool --help简单的查看命令,不能使用man xdotool查看手册,这里把上面的内容和官方参考手册一起作成[pdf](http://sosohu.github.io/assets/doc/xdotool.pdf),以供下载使用.
