# 前言

最近开始阅读`react-router`的源码，一开始接触到一个名为`history.js`的第三方库，后面慢慢发现`react-router`的设计中很大程度依赖于这个库，因此我先把注意力转移到`history.js`上进行学习。下面就写一下我对这个库的学习总结。

## 作用

关于`history`的作用，可以引述官方的介绍来说明：

> `history`可以令你更轻松地管理在`JavaScript`环境下运行的会话历史（`session history`）。一个`history`实例抽象了各种环境中的差异，且提供了最简洁的`API`去管理会话中的历史栈（`history stack`）、导航（`navigate`）以及持久化的状态（`persist state`）。

``