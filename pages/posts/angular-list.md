---
title: Angular 学习资源清单
date: 2019/10/16
description: 这篇文章是对之前知乎上对问题 “Angular 新手如何有效学习 Angular” 的回答的扩充。按照学习的先后顺序、学习的难易程度列出了在学习 Angular 的过程中可能会需要的材料。
tag: Angular, awesome, Chinese
author: Wenzhao
---

# Angular 学习资源清单

[angular 新手如何有效学习 angular？ - 知乎](https://www.zhihu.com/question/34083190/answer/685703207)

## 学习 Angular 之前

这一部分和常见的前端入门学习内容并无不同。请确保你掌握了 HTML 和 JavaScript 的基础，一个检查的好办法是看网上的前端面经，会被不同的面试官问到的、框架无关的问题都应该答得上才对。

1. 《JavaScript 高级程序设计》
2. 《你不知道的 JavaScript》
3. 《高性能 JavaScript》
4. 《ECMAScript 6 入门》

## 初学 Angular

1. 掌握 Angular 基础
   1. [Angular 官方文档](https://angular.io/)，也可以看[中文版](https://angular.cn)（之后出现的英文文档基本能找到对应的中文版本，不再贴出链接），至少要阅读 Fundamentals 及之前的部分
2. 掌握 RxJS 基础
   1. [RxJS 官方文档](https://rxjs-dev.firebaseapp.com/)，至少要阅读 Overview 部分（当然无需阅读全部的 operator）
   2. [30 天精通 RxJS 系列](https://ithelp.ithome.com.tw/users/20103367/ironman/1199)
3. 掌握 TypeScript 基础
   1. [TypeScript 官网文档](https://www.typescriptlang.org/)，读完 Tutorial 和 Handbook，对类型标注之外的语法有印象即可
4. 学习组件库
   1. [ng-zorro-antd](https://github.com/NG-ZORRO/ng-zorro-antd)
   2. [Angular Material](https://github.com/angular/components)
5. 通过项目（教程、demo、实验室、商业项目）学习
   1. [today-ng-steps](https://github.com/NG-ZORRO/today-ng-steps)，使用 ng-zorro-antd 组件库的 Angular 和组件库入门教程（自己写的私货，目前处于长草状态，谨慎学习）
   2. 如果在学校的话可以看看学校实验室需不需要前端
   3. 寻找实习
6. 贡献 Angular 生态项目
   1. ng-zorro-antd 是我所知的对中文贡献者最友好的项目了（确信）！而且不时会有一些 issue 会被标注为 Good First Issue（新手友好）值得尝试解决，所以可以关注 ng-zorro-antd 的 issue 情况（免费的代码 review！）。

## 进一步学习

到这里再学习 Angular 应该已经度过了最困难的阶段了，你要做的是从各种信息源中获取新知识，和深入理解 Angular 原理，学习 Angular 生态链中的其他库，剩下的就是靠做项目来不断提升了。

### 值得关注的信息源

1. [Angular in Depth](https://link.zhihu.com/?target=https%3A//blog.angularindepth.com/)，最深入最精华的 Angular 进阶学习指南，变更检测、路由等都能在这个专栏里找到对应的专题
2. [NG ZORRO 团队知乎专栏](https://zhuanlan.zhihu.com/100000)，不仅有 ng-zorro-antd 新版本介绍，团队成员还会不时分享一些 Angular 和 ng-zorro-antd 开发过程中遇到的问题和设计考量
3. [Angular 变更检测可视化](https://danielwiehl.github.io/edu-angular-change-detection/)
4. 之前提到的各个文档，是时候通读一边，补上未读的东西了

### 值得了解的库

1. [NgRx](https://ngrx.io/)，Angular 状态管理库（不过很有可能会觉得在 Angular 搞 Redux 那一套不是个好主意）
2. [ng-packagr](https://github.com/ng-packagr/ng-packagr)，如果要开发 Angular 组件库的话这个库是必学的

### 值得阅读源码的库

1. Angular
2. Material
3. RxJS

### 参加开源

可能对于在校大学生来说很少有时间或机会参与真正的企业级产品开发，所以建议大家[从写开源开始](https://github.com/wzhudev/blog/issues/12)。

## 参考

1. 由 vthinkxie 整理的[《Angular 资料获取不完全指南》](https://zhuanlan.zhihu.com/p/36385830)列举了非常多的学习材料，可供参考
2. 另外一个好办法是顺藤摸瓜，看其他 Angular 开发者的关注列表里都有什么，这能帮助你认识其他优秀的 Angular 开发者，获取一手资讯
