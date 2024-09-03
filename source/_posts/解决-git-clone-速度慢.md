---
title: 通过代理解决 git clone 和 homebrew 速度慢的问题
date: 2019-05-18 20:00:00
updated: 2019-05-18 20:00:00
categories: tech
tags: [git]
---



实在忍不了在本地 git 的龟速了，虽然官网不挂代理也访问无碍，但是本地 git clone 总是只有几 KB，下载稍大的一点的 repo 不仅慢而且失败率飙升
解决方案自然是使 git clone 走代理了
```
git config --global http.https://github.com.proxy socks5://127.0.0.1:1086
git config --global https.https://github.com.proxy socks5://127.0.0.1:1086
```
小水管下载也完美从几 KB 提升到了 500KB
<!-- more -->
由于我用的 ss，走的是 socks5 协议，并且端口号自定义为了 1086，所以使用如上配置


加速 homebrew 同样挂上代理：
```
export http_proxy=http://127.0.0.1:1087;export https_proxy=http://127.0.0.1:1087;
brew update && brew upgrade
```

使用时一定要请看代理协议与端口号。。。最开始一只把 http 的端口号也当成了 1086 导致一直没啥速度

参考：
[zhihu: git clone...](https://www.zhihu.com/question/27159393)