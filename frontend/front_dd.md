# 掌握前端框架需要了解一点前端知识

- 前端技术的发展非常的迅速，尤其在NodeJs成熟起来后，各类应用场景的技术日新月异。

# 客户端的模式
1. **MVC**：Model包含数据和行为的领域模型。通过Controller去操作Model层，同时去更新View层显示。Model数据在变化时可以向View发通知。
2. **MVP**：Presenter负责View与Model的交互，P仅依赖V的接口，并单向调用M。MVC变体，适用基于事件驱动。
3. **MVVM**：ViewModel负责View与Model的交互。一个ViewModel和一个View匹配，V与VM之间的变化自动同步。适用基于数据驱动（微软提出的）。


# 模块化规范
1. **CommonJS**，同步模块规范，用在后端模块化，主要实现者是Node.js和webpack。一个文件就是一个模块，拥有单独的作用域；通过require来加载模块；通过exports和module.exports来暴露模块中的内容。
2. **AMD**，异步模块规范，将CommonJS用在前端，主要实现者是：RequireJS、CurlJs。加载模块方式是require([module], callback);定义模块方式是define(id?, [dependencies?], factory)，它的return来暴露模块中的内容。
3. **CMD**，通用模块定义规范，将AMD中的依赖修改为就近依赖，实现者：seajs。通过require来加载模块；定义模块是define(function(require, exports, module) {})。
4. **UMD**，兼容AMD、CommonJS、“全局”变量规范，目前还没有广泛使用的实现。
5. **ES6 Module**，未来的JS模块化，一个模块就是一个独立的文件，export来暴露模块中的内容。import导入指定的模块。


# 包管理
1. **npm**，是随同NodeJS一起安装的包管理工具，可以下载上传Js包到NPM服务器。包放在node_modules 目录中。可以直接require() 使用。
2. **bower**，是twitter 推出的一款包管理工具。Bower依赖于Node.js，Git（资源主要通过git进行下载）。
3. **spm**，专门用于打包、压缩Sea.js模块以及CSS文件。依赖于Node.js。
4. **yarn**，是Facebook公司出品的用于管理nodejs包的管理工具。可看作带本地缓存的npm。

> **NodeJS**，将JavaScript能力移植到服务端，但是也同样广泛使用在JavaScript技术栈的开发工具上。