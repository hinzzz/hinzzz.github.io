---
title: Node&Vue随笔
categories: 前端
tags: [Node,Vue]
date: 2019/08/01
keywords: [package.json。locked,vue.use,import,require,export,export default]
description: package.json概念，vue.use
---



### package.json 和 package-lock.json 作用分析



1、package.json 文件里记录有项目所安装的依赖项，当 node_modules 被删除时，可以再根据该文件安装所需的依赖项；

2、npm 5 以前不会有 package-lock.json 这个文件，npm5 之后才加入这个文件；

3、当安装包的时候，npm 都会生成或者更新 package-lock.json 这个文件；

4、npm 5 之后的版本安装包的时候不需要加 --save 参数，它会自动保存依赖的信息；

5、当安装包的时候，会自动创建或者更新 package-lock.json 文件；

6、package-lock.json 文件会保存 node_modules 中所有包的信息（版本、下载地址），重新 npm install 的时候速度会提升；

7、文件的名称有 lock ，表示该文件可以用来锁定版本号，防止自动升级新版。



### vue.use

// 文件:  src/classes/vue-use/plugins.js

const Plugin1 = {
    install(a, b, c) {
        console.log('Plugin1 第一个参数:', a);
        console.log('Plugin1 第二个参数:', b);
        console.log('Plugin1 第三个参数:', c);
    },
};

function Plugin2(a, b, c) {
    console.log('Plugin2 第一个参数:', a);
    console.log('Plugin2 第二个参数:', b);
    console.log('Plugin2 第三个参数:', c);
}

export { Plugin1, Plugin2 };





// 文件: src/classes/vue-use/index.js

import Vue from 'vue';

import { Plugin1, Plugin2 } from './plugins';

Vue.use(Plugin1, '参数1', '参数2');
Vue.use(Plugin2, '参数A', '参数B');





// 文件: src/main.js

import Vue from 'vue';

import '@/classes/vue-use';
import App from './App';
import router from './router';

Vue.config.productionTip = false;

/* eslint-disable no-new */
new Vue({
    el: '#app',
    router,
    render: h => h(App),
});



从中可以发现我们在`plugin1`中的`install`方法编写的三个console都打印出来，第一个打印出来的是Vue对象，第二个跟第三个是我们传入的两个参数。
而`plugin2`没有`install`方法，它本身就是一个方法，也能打印三个参数，第一个是Vue对象，第二个跟第三个也是我们传入的两个参数。



源码

// Vue源码文件路径：src/core/global-api/use.js

import { toArray } from '../util/index'

export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}

从源码中我们可以发现vue首先判断这个插件是否被注册过，不允许重复注册。
并且接收的`plugin`参数的限制是`Function | Object`两种类型。
对于这两种类型有不同的处理。
首先将我们传入的参数整理成数组 => `const args = toArray(arguments, 1)`。

再将`Vue`对象添加到这个数组的起始位置`args.unshift(this)`,这里的this 指向`Vue`对象
如果我们传入的`plugin`(Vue.use的第一个参数)的`install`是一个方法。也就是说如果我们传入一个对象，对象中包含`install`方法，那么我们就调用这个`plugin`的`install`方法并将整理好的数组当成参数传入`install`方法中。 => `plugin.install.apply(plugin, args)`
如果我们传入的`plugin`就是一个函数,那么我们就直接调用这个函数并将整理好的数组当成参数传入。 => `plugin.apply(null, args)`
之后给这个插件添加至已经添加过的插件数组中，标示已经注册过 => `installedPlugins.push(plugin)`
最后返回Vue对象。



结论

通过以上分析我们可以知道，在我们以后编写插件的时候可以有两种方式。
一种是将这个插件的逻辑封装成一个对象最后将最后在install编写业务代码暴露给Vue对象。这样做的好处是可以添加任意参数在这个对象上方便将install函数封装得更加精简，可拓展性也比较高。
还有一种则是将所有逻辑都编写成一个函数暴露给Vue。
其实两种方法原理都一样，无非第二种就是将这个插件直接当成install函数来处理。
个人觉得第一种方式比较合理。
举个?



### import和require



- require的基本语法
- 核心概念：在导出的文件中定义module.export,导出的对象的类型不予限定（可以是任何类型，字符串，变量，对象，方法），在引入的文件中调用require()方法引入对象即可。

```js
//a.js中
module.export = {
    a: function(){
     console.log(666)
  }
}
```

```js
//b.js中
var obj = require('../a.js')
obj.a()  //666
```



【注】:本质上是将要导出的对象赋值给module这个的对象的export属性，在其他文件中通过require这个方法访问该属性





import的基本语法

核心概念：导出的对象必须与模块中的值一一对应，换一种说法就是**导出的对象与整个模块进行结构赋值**。对的，你没有听错。抓住重点，解构赋值！！！！！

```js
//a.js中
export default{    //（最常使用的方法,加入default关键字代表在import时可以使用任意变量名并且不需要花括号{}）
     a: function(){
         console.log(666)
   }
}
 
export function(){  //导出函数
 
}
 
export {newA as a ,b,c}  //  解构赋值语法(as关键字在这里表示将newA作为a的数据接口暴露给外部，外部不能直接访问a)
 
//b.js中
import  a  from  '...'  //import常用语法（需要export中带有default关键字）可以任意指定import的名称
 
import {...} from '...'  // 基本方式，导入的对象需要与export对象进行解构赋值。
 
import a as biubiubiu from '...'  //使用as关键字，这里表示将a代表biubiubiu引入（当变量名称有冲突时可以使用这种方式解决冲突）
 
import {a as biubiubiu,b,c}  //as关键字的其他使用方法
```

**它们之间的区别**

- require 是赋值过程并且是运行时才执行， import 是解构过程并且是编译时执行。require可以理解为一个全局方法，所以它甚至可以进行下面这样的骚操作，是一个方法就意味着可以在任何地方执行。而import必须写在文件的顶部。

  ```js
  var a = require(a() + '/ab.js')
  ```

  - require的性能相对于import稍低，因为require是在运行时才引入模块并且还赋值给某个变量，而import只需要依据import中的接口在编译时引入指定模块所以性能稍高
  - 在commom.js 中module.export 之后 导出的值就不能再变化，但是在es6的export中是可以的。



```js
var a = 6
export default {a}
a = 7  //在es6中的export可以
```

```js
var a = 6
module.export = a
a = 7   //在common.js中，这样是错误的
```





### export和export default

一个a.js文件有如下代码：

```
export var name="李四";
```

在其它文件里引用如下：

```js
import { name } from "/.a.js" //路径根据你的实际情况填写
export default {
  data () {
    return { }
  },
  created:function(){
    alert(name)//可以弹出来“李四”
  }
 }
```

上面的例子是导出单个变量的写法，如果是导出多个变量就应该按照下边的方法，用大括号包裹着需要导出的变量：

```
var name1="李四";
 var name2="张三";
 export { name1 ,name2 }
```

在其他文件里引用如下：

```
import { name1 , name2 } from "/.a.js" //路径根据你的实际情况填写
export default {
  data () {
    return { }
  },
  created:function(){
    alert(name1)//可以弹出来“李四”
    alert(name2)//可以弹出来“张三”
  }
 }
```

如果导出的是个函数呢，那应该怎么用呢,其实一样，如下

```
function add(x,y){
   alert(x*y)
  //  想一想如果这里是个返回值比如： return x-y，下边的函数怎么引用
}
export { add }
```

在其他文件里引用如下：

```
import { add } from "/.a.js" //路径根据你的实际情况填写
export default {
  data () {
    return { }
  },
  created:function(){
   add(4,6) //弹出来24
  }
 }
```

结论：

1、export与export default均可用于导出常量、函数、文件、模块等
2、你可以在其它文件或模块中通过import+(常量 | 函数 | 文件 | 模块)名的方式，将其导入，以便能够对其进行使用
3、在一个文件或模块中，export、import可以有多个，export default仅有一个
4、通过export方式导出，在导入时要加{ }，export default则不需要

这样来说其实很多时候export与export default可以实现同样的目的，只是用法有些区别。注意第四条，通过export方式导出，在导入时要加{ }，export default则不需要。使用export default命令，为模块指定默认输出，这样就不需要知道所要加载模块的变量名。

```
var name="李四";
export { name }
//import { name } from "/.a.js" 
可以写成：
var name="李四";
export default name
//import name from "/.a.js" 这里name不需要大括号
```

再看第3条，在一个文件或模块中，export、import可以有多个，export default仅有一个，也就是说如下代码：

```
var name1="李四";
var name2="张三";
export { name1 ,name2 }
```

也可以写成如下，也是可以的，import跟他类似。

```
var name1="李四";
 var name2="张三";
 export name1;
 export name2;
```



### Vuex

Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。Vuex 也集成到 Vue 的官方调试工具 [devtools extension](https://github.com/vuejs/vue-devtools)，提供了诸如零配置的 time-travel 调试、状态快照导入导出等高级调试功能。

