---
layout: post_layout
title: 测试篇
time: 2016年03月05日 星期五
location: 上海
pulished: true
excerpt_separator: "```"
---

## 基础

Android 的测试种类:


- **Unit Test** (单元测试)
    - **JUnit Test**

      这个只能用来测试无关Android平台的功能代码, 只能在本地运行
    - **Instrumentation Unit Test**

      这种单元测试运行在 Android 系统中, 这些测试可以获取到测试应用的上下文信息，用来测试有 Android API 的代码


- **Integration Tests** (集成测试)
    - **Components within your app only**

      这种类型的测试用来验证当用户进行了一个特定的操作或者特定的输入，目标应用的行为是否和预期一样。
      像 `Espresso` 这种UI测试框架就能允许你模拟用户的动作，能测试复杂的应用交互

    - **Cross-app Components**

      这种测试就是用来验证多个不同的应用间或者 应用和系统应用间的正确交互

