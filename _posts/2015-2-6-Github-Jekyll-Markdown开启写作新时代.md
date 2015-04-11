---
layout: post
time: 2015-2-6
title: Github-Jekyll-Markdown开启写作新时代
category: experience
keywords: 写作
tags: experience
description: 使用GJM配置个人网站
---

##为什么想到用这几个东西来写博客
首先markdown用来写作已经是非常流行的了,确实非常方便高效,虽然表达能力没有latex丰富,但是已经算是即轻量级又比较美观的一种方式了,特别是github风格的markdown文章还是比较好看的.其次,我之前的博客要么在博客园这类网站上写,要么是在github的项目仓库中写:sweat:, 但是感觉还是很不方便,而且觉得作为码农还是应该搭建一个自己的博客基地比较geek:smile:, 可是我又想能够快速搭建起来,而且不要太麻烦,几经调研,发现github.io作为搭建平台非常的适合,结果也是很让我满意,大概仅仅只用一天时间,就基本搭建成功一个我觉得还比较满意的博客.


##怎么搭建的
前面废话那么多,感觉的是在给你心灵鸡汤一样,但是我不会只给鸡汤不给汤勺的:satisfied:, 下面就来介绍一下我搭建过程中的一些经验总结.

###准备工作
####github账号: 
这个不用多说了,很简单,去申请一个就行了.

####在gitbub上建立一个usename.github.io的仓库: 
  usename是你的github账号名,貌似不一定非要叫这个名字,反正我是起的这个名字


####clone一个你喜欢的代码: 
github上jeklly做出来的各种主题都有很多代码,我这里使用的是[PinkingS](https://github.com/PinkingS/pinkings.github.com)的jekyll-bootstrap风格的主题,jekyll还有很多其他主题,你可以到它的官网看一看,然后找相应github上的样例代码.



####修改仓库变为自己的博客

你需要修改一下内容:

*	_config.yml:  凡是别人的都改成自己的.
*	_posts:   改为自己的博客,注意命名方法要遵循它的规则.
*	about.md:   自我介绍.

修改完之后,上传到github对应仓库之后,打开usename.github.io网站就可以看到效果啦.



#####评论插件
评论插件很简单,你只需要去[DISQUS](http://disqus.com/)注册一个用户名,然后在_config.yml中disqus改为你的用户名，然后_includes/JB/comments-providers/disqus文件也做对应修改.



####本地搭建jekyll环境:	
你可以不用搭建,然后每次提交代码到github上再看效果,搭建这个本地环境是为了让你能在本地实时的查看博客效果.

先是安装ruby(最好是1.9.2版本之后的)和rake,然后安装jekyll和rdiscount
{% highlight sh %}
sudo apt-get install ruby rake
sudo gem install jekyll
sudo gem install rdiscount
{% endhighlight %}
如果安装不了合适的ruby,可以自己到ruby官网下载最新版本,`configure && make && make install`安装,另外,你有可能无法`gem install jekyll`,那么因为GFW的原因,可以更换source mirror:
{% highlight sh %}
gem sources --remove https://rubygems.org/
gem sources -a http://ruby.taobao.org/
{% endhighlight %}
其他的,你还需要安装nodejs


####使用jekyll本地产生文件
在项目代码根目录,执行`jekyll build . && jekyll serve`然后打开网址**http://127.0.0.1:4000**即可查看效果

