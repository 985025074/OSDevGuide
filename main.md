# CongOs 结构说明

最后更新时间 2025 12 25 22:00
目前最新分支 basictest_dev

## 项目开发大致过程

1. 前期: 参考 RCore 完成基本的系统框架.从内存管理 到 多线程.(单核)

2. 中期: 做完 rCore 的 基本要求之后,开始研究比赛要求,发现是多核的,就想着先把多核给做了,以免以后大部分代码都要修改.

这个阶段较为困难,耗时最长.存在 大量 race condtion

## todoList

## 各部分说明

该部分介绍系统的大致构成

- [多核设计](./parts/multi_core.md)
- [文件系统]
- []
- []

一些开发文档,记录一些开发细节

- [Shell]
- [ext4-fs-packer 说明]
- [misc 各种开发记录]
- [为通过 BasicTest 而做的努力]
- [为做 cyclictest_testcode 而作的努力]
