---
layout: post
time: 2015-5-1
title: Valid Number
category: leetcode
keywords: RegExp
tags: 解题报告
description: Valid Number
---

##题目原型
{% highlight C++ %}
string is numeric.
    
Some examples:
    "0" => true
    " 0.1 " => true
    "abc" => false
    "1 a" => false
    "2e10" => true

Note: It is intended for the problem statement to be ambiguous. You should gather all requirements up front before implementing one.
{% endhighlight %}

##解题报告
LeetCode上题目很多,其中也不乏很多难题,但是为什么要讲这道题呢? 因为这题看上去并不难,但是确非常麻烦,麻烦的原因题目没有明确告诉你哪些是合法数字,哪些不是合法数字,你只能一个个的去试验. 另外一点,合法的数形式很多,如果你使用不对方法,那么会让你代码编写起来非常的痛苦,漏洞百出,各种`if`和`else`...

这里我推荐先根据题意,写出正则表达式,然后根据正则表达式画出NFA,然后根据NFA的状态转移编写代码,你就会发现此题就异常的清晰.

###正则表达式:

![Reg](https://github.com/sosohu/sosohu.github.io/assets/image/posts/2015-5-1-LeetCode/RegExp.png)

s为空格 ,d为数字

###状态转移图(NFA):

![NFA](https://github.com/sosohu/sosohu.github.io/assets/image/posts/2015-5-1-LeetCode/NFA.png)

其中0代表开始状态,2,4,7,8都是合法状态.

有了状态转移图之后,我们就可以轻松的编写代码如下:

{% highlight cp %}
enum lexical{ valid, space, sign, number, dot, e };

int isType(const char c){
    switch(c){
        case ' ': return 1;
        case '+': ;
        case '-': return 2;
        case '.': return 4;
        case 'e': return 5;
        default: if(c <= '9' && c >= '0')   return 3;
                 else return 0;
    }
}

bool isNumber(const char *s) {
    if(!s)   return false;
    //状态转移矩阵, -1表示非法
    int map[9][6] = {
        -1, 0, 1, 2, 3, -1,  // 0状态的转移
        -1, -1, -1, 2, 3, -1,
        -1, 8, -1, 2, 4, 5,
        -1, -1, -1, 4, -1, -1,
        -1, 8, -1, 4, -1, 5,
        -1, -1, 6, 7, -1, -1,
        -1, -1, -1, 7, -1, -1,
        -1, 8, -1, 7, -1, -1,
        -1, 8, -1, -1, -1, -1 
    };
    int state = 0;
    while(*s){
        state = map[state][isType(*s)];
        if(state == -1)   return false;
        s++;
    }
    return state == 2 || state == 4 || state == 7 || state == 8;
}
{% endhighlight %}
