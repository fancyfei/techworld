# 后端分离框架-VUE框架详细功能及特性
Vue用深度依赖ES6。Module实现了组件化。

# Vue组件申明中的常用属性
1. data： 模板中是对象，组件中是函数，也可以通过vm.$data直接访问。
2. props： 数组或对象，接收父组件的数据。
3. methods：	function对象。本组件中可以使用。
4. components： 在此组件中再次使用的自定义组件集合，同样也包含template、data、methods等这些属性。
5. computed：数据的计算，返回结果值，结果会缓存，在这属性里面的函数监听了数据。
6. watch：此属性的键是需要观察的表达式，值是对应回调函数。
7. mounted：在模板渲染成html后调用的函数。一般用来处理交互时的数据变化。
8. created：	在模板渲染成html前调用的函数。一般用来处理数据加载。
9. destroyed：实例销毁后调用的函数。

# 组件复用性和扩展性
- solt：插槽，是对组件的扩展，通过slot插槽向组件内部指定位置传递内容，通过slot可以父子传参。
- mixins：混入，定义了一部分可复用的方法或者计算属性。1. 钩子函数，那将会进行合并，都执行。2. methods等方法，被组件中会覆盖。


![vue_lifecycle](./img/vue_lifecycle.jpg)