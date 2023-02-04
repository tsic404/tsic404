---
title: "Perl快速入门"
date: 2023-02-07T13:11:08+08:00
slug: "OBS admin manual"
tags: [ "open build service", "perl" ]
categories: [ 'programm language' ]
draft: true
---

# Perl 快速入门

## 学习perl的资料

骆驼书

- 《Learning Perl》初级入门（中文《Perl语言入门》）
- 《Immediate Perl》进阶
- 《Mastering Perl》高级Perl
- 《Perl最佳实践》实践

口袋书
- 《Perl 5 Pocket Reference》

Unix网络

- 《Perl网络编程》
- 《Unix网络编程系列》

### perdoc

perl-doc包

## 开始学习

### HelloWorld

```perl
#/usr/bin/env perl
print "Hello World!\n";
```
区分大小写，结尾必须分号。
### 注释

**perl只有行注释，使用#**

### 变量

创建变量
```
$a = 32;
```

不使用my关键字均是全局变量。即使在`{}`和函数里面。

```perl
if (expression) {
    my $a = 20; # 局部变量
    $b = 30;    # 全局变量
}
```
就近原则，就近隐藏。
局部代码块创建全局变量会得到一个waning。使用
```perl
use strict;
```
将强制使用my;

### 基本数据类型
#### 标量类型
标量指`数字`，`字符串`，`undef`。
