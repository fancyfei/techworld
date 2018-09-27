# 后端分离框架-VUE框架简介

- 前后端分离框架。单页应用框架。MVVM库（Model+View+ViewModel）。
- 用于构建用户界面的渐进式框架。
- 被设计为可以自底向上逐层应用。
- Vue 的核心库只关注视图层，便于与第三方库或既有项目整合。
- 面向组件。
- 全家桶系列工具，可方便构建应用。
- 类似的框架Angular、React、Vue。
- “vue.js兼具angular.js和react.js的优点，并剔除了它们的缺点”。
- 作者：尤雨溪（Evan You）

# 了解一点客户端（重前端）的模式
1. MVC：Model包含数据和行为的领域模型。通过Controller去操作Model层，同时去更新View层显示。Model数据在变化时可以向View发通知。
2. MVP：Presenter负责View与Model的交互，P仅依赖V的接口，并单向调用M。MVC变体，适用基于事件驱动。
3. MVVM：ViewModel负责View与Model的交互。一个ViewModel和一个View匹配，V与VM之间的变化自动同步。（微软提出的）适用基于数据驱动。
