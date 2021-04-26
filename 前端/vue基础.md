vue几种页面刷新方法：<https://blog.csdn.net/baidu_39418435/article/details/81538760>

# 前端相关知识

event.preventDefault()  ： 会阻止默认事件发生

比如a标签会跳转，使用后将不再执行跳转效果

```html
<a href="http://www.baidu.com" @click="stopDefault(msg2, $event)">兆傻球按钮</a>
```

```js
// 默认会先执行鼠标点击事件，再执行a标签跳转
//使用event.preventDefault()后会仅执行点击事件，阻止了默认跳转效果
methods：{
	stopDefault(msg2, event) {
        console.log('事件：', event)  //鼠标点击事件
        // if(event) event.preventDefault();
        if(event) {
          // event.preventDefault();
          alert(msg2)
        }
    }
}
```

**vue ui组件**：[https://www.cnblogs.com/zdz8207/p/vue-ui-framework.html](javascript:;)



### 1.js数组过滤

```js
1 var colors = ["red", "blue", "grey"];
2 
3 colors = colors.filter(function(item) {
4     return item != "red"
5 });
6 
7 console.log(colors);    //["blue", "grey"]
```



# 一，Vue项目目录结构

### package.json

文件中为项目所需的模块记录，启动项目需要该文件

可根据该文件中模块记录直接使用 npm install 安装其中所有模块

### 关闭eslint低级语法问题检测

build目录下的webpack.base.conf.js文件：

删除rules中的以下代码，之后重启ide：

```
 ...(config.dev.useEslint ? [createLintingRule()] : []),
```

![1571195536138](C:\Users\YWX800~1\AppData\Local\Temp\1571195536138.png)





### 直接展示组件到App.vue

1. 引入自定义组件，并命名。如: import hello from './components/helloWorld'

2. 加载组件, 如: export default中**新增参数components: {hello}**

   ```
    export default {
      name: 'App',
      components: {
        hello
      }
    }
   ```

3. 解析刚引入的组件hello（ **解析的标签为script中引入的hello** ）

   ```html
   <template>
     <div id="app">
       <img src="./assets/logo.png">
       <!--<router-view/>-->
   	// 将组件中<template>标签中的内容解析在此处。解析的标签<hello>为script中引入的hello
       <hello></hello>  // 解析组件
     </div>
   </template>
   ```

### 通过路由访问组件

```
。。。。。。
```

# 二，模板语法

## 1.插值

使用js

```js
{{ number + 1 }}
{{ ok ? 'YES' : 'NO' }}
{{ message.split('').reverse().join('') }}
<div v-bind:id="'list-' + id"></div>
```

```
这些表达式会在所属 Vue 实例的数据作用域下作为 JavaScript 被解析。有个限制就是，每个绑定都只能包含单个表达式，所以下面的例子都不会生效。
```

```js
{{ var a = 1 }}  <!-- 这是语句，不是表达式 -->
{{ if (ok) { return message } }}  <!-- 流控制也不会生效，请使用三元表达式 -->
```



## 2.指令

**v-once**

能执行一次性地插值，当数据改变时，插值处的内容不会更新。但这会影响到该节点上的其它数据绑定：

```
<span v-once>这个将不会改变: {{ msg }}</span>
```

**v-html**

双大括号只会解析为普通文本，解析HTML需使用v-html指令

```html
<span v-html="rawHtml"></span>
```

这个 `span` 的内容将会被替换成为属性值 `rawHtml`，直接作为 HTML——会忽略解析属性值中的数据绑定。

**v-if 和 v-show区别**

用法大致一样，区别是：

v-if值为false时，所在元素不会被渲染

v-show的元素始终被渲染，只是简单切换元素的css属性`display`

**注意**:`v-show` 不支持 `<template>` 元素，也不支持 `v-else`。



# 三，计算属性和侦听器

## 1.计算属性

```js
# 2.模板中使用
 <p>{{ getNewMsg }}</p>

# 1
  export default {
    data() {
      return {
        message: '周兆毅'
      };
    },
    computed: {
      getNewMsg() {
        return this.message + '是个傻子'
      }
    }
  }
```

### 计算属性和方法的区别

可以通过在表达式中调用方法来达到和计算属性同样的效果

```
<p>{{ getNewMsg() }}</p>
```

```
不同的是计算属性是基于它们的响应式依赖进行缓存的。只在**相关响应式依赖发生改变时它们才会重新求值**。意味着只要 `message` 还没有发生改变，多次访问 `getNewMsg` 计算属性会立即返回之前的计算结果，而不必再次执行函数。
```

下面的计算属性将不再更新，因为 `Date.now()` 不是响应式依赖（没有和data属性关联）：

```
computed: {
  now: function () {
    return Date.now()
  }
}
```

## 2.侦听器

。。。



# 四，Class与 Style 绑定

## 1.绑定HTML Class

### 对象语法

```html
// 模板
<div
  class="static"
  v-bind:class="{ active: isActive, 'text-danger': hasError }"
></div>

// data
data: {
  isActive: true,
  hasError: false
}
// 结果渲染为：
<div class="static active"></div>
```

### 数组语法

```html
<div v-bind:class="[activeClass, errorClass]"></div>

data: {
  activeClass: 'active',
  errorClass: 'text-danger'
}
```

如果想根据条件切换列表中的 class，可以用三元表达式：

```html
<div v-bind:class="[isActive ? activeClass : '', errorClass]"></div>
```

在数组语法中也可以使用对象语法：

```html
<div v-bind:class="[{ active: isActive }, errorClass]"></div>
```

### 用在组件上

```
当在一个自定义组件上使用 `class` 属性时，这些 class 将被添加到该组件的根元素上面。这个元素上已经存在的 class 不会被覆盖。
```

```html
// my-component
<p class="foo bar">Hi</p>
// 使用它的时候添加一些 class：
<my-component class="baz boo"></my-component>
// HTML 将被渲染为:
<p class="foo bar baz boo">Hi</p>

```

## 2.绑定内联样式

当 `<style>` 标签有 [scoped](https://link.juejin.im/?target=https%3A%2F%2Fvue-loader-v14.vuejs.org%2Fzh-cn%2Ffeatures%2Fscoped-css.html) 属性时，它的 CSS 只作用于当前组件中的元素。

。。。

# 五，条件渲染

### 用key管理可复用的元素

```
Vue 会尽可能高效地渲染元素，通常会复用已有元素而不是从头开始渲染。这么做除了使 Vue 变得非常快之外，还有其它一些好处。例如，如果你允许用户在不同的登录方式之间切换：
```

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address">
</template>
```

```
那么在上面的代码中切换 `loginType` 将不会清除用户已经输入的内容。因为两个模板使用了相同的元素，`<input>` 不会被替换掉——仅仅是替换了它的 `placeholder`。

如果不要复用它们，只需添加一个具有唯一值的 `key` 属性即可：
```

```html
// 其它和上面一样
......
  <input placeholder="Enter your username" key="username-input">
......
  <input placeholder="Enter your email address" key="email-input">
......
```

**注意: `<label>` 元素仍然会被高效地复用，因为它们没有添加 `key` 属性。**



# 六，列表渲染

## 1.数组更新检测

### 变异方法

```
变异方法，顾名思义，会改变调用了这些方法的原始数组（只是改变了数组的内容）

Vue 将被侦听的数组的变异方法进行了包裹，所以它们也将会触发视图更新。这些被包裹过的方法包括：
```

- `push()`
- `pop()`
- `shift()`
- `unshift()`
- `splice()`
- `sort()`
- `reverse()`

### 替换方法

```
例如 `filter()`、`concat()` 和 `slice()`,它们不会改变原始数组，而**总是返回一个新数组**。当使用非变异方法时，可以用新数组替换旧数组：
```

```js
this.items = this.items.filter(function (item) {
  return item.message.match(/Foo/)
})
```

**`filter`: 遍历一个数组/对象，对其中每个元素进行处理，返回一个新数组，然后触发视图更新**

用一个含有相同元素的数组去替换原来的数组是非常高效的操作。

### 注意事项

由于 JavaScript 的限制，Vue **不能**检测以下数组的变动，数组视图将不会更新：

1. 当你利用索引直接设置一个数组项时，例如：`vm.items[indexOfItem] = newValue`
2. 当你修改数组的长度时，例如：`vm.items.length = newLength`

```
data() {
    items: ['a', 'b', 'c']
  }
})
this.items[1] = 'x' // 不是响应性的
this.items.length = 2 // 不是响应性的
```

### 解决办法

使用`this.$set`实例方法，该方法是全局方法 `Vue.set` 的一个别名：

```js
this.$set(this.items,indexofitem,newValue)
```

为了解决第二类问题，你可以使用 `splice`：

```js
this.items.splice(newLength)
```



## 2.对象变更检测注意事项

还是由于 JavaScript 的限制，**Vue 不能检测对象属性的添加或删除**：

```js
  data() {
    a: 1    // `this.a` 现在是响应式的
  }
})
vm.b = 2  // `this.b` 不是响应式的
```

```
对于已经创建的实例，Vue 不允许动态添加根级别的响应式属性。但是，可以使用 `Vue.set(object, propertyName, value)` 方法向嵌套对象添加响应式属性

```

```js
Vue.set(this.userobj, 'age', 27)
```

```
你还可以使用 `vm.$set` 实例方法，它只是全局 `Vue.set` 的别名：

```

```js
this.$set(vm.userobj, 'age', 27)
```

```
有时你可能需要为已有对象赋值多个新属性，比如使用 `Object.assign()` 或 `_.extend()`。在这种情况下，你应该用两个对象的属性创建一个新的对象:

```

```js
this.userProfile = Object.assign({}, this.userProfile, {
  age: 27,
  favoriteColor: 'Vue Green'
})
```

### 3.显示过滤/排序后的结果

```
有时想要显示一个数组经过过滤或排序后的版本，而不实际改变或重置原始数据。可以创建一个计算属性，来返回过滤或排序后的数组。

```

```js
<li v-for="n in evenNumbers">{{ n }}</li>
--------------------------------------------------------------------
data() {
  numbers: [ 1, 2, 3, 4, 5 ]
},
computed: {
  evenNumbers: function () {
    return this.numbers.filter(function (number) {
      return number % 2 === 0
    })
  }
}
```

在计算属性不适用的情况下也可以使用方法！

### 4.v-for里使用值范围

`v-for` 也可以接受整数。在这种情况下，它会把模板重复渲染相应次数。

```html
<span v-for="n in 10">{{ n }} </span>
// 结果：1 2 3 4 5 6 7 8 9 10
```



# 七，事件处理

## 1.内联处理器中的方法

有时也需要在内联语句处理器中访问原始的 DOM 事件。可以用特殊变量 `$event` 把它传入方法：

```js
<button v-on:click="warn('阻止表单提交', $event)">Submit</button>
------------------------------------------------------------------
methods：{
    warn：function（message， event）{
        if（event） event.preventDefault()
        alert(message)
    }
}
```

## 2.事件修饰符

```
在事件处理程序中调用 `event.preventDefault()` 或 `event.stopPropagation()` 是非常常见的需求。尽管可以在方法中轻松实现这点，但更好的方式是：方法只有纯粹的数据逻辑，而不是去处理 DOM 事件细节。

```

为了解决这个问题，Vue.js 为 `v-on` 提供了**事件修饰符**。修饰符是由点开头的指令后缀来表示的。

- `.stop`
- `.prevent`
- `.capture`
- `.self`
- `.once`
- `.passive`

```html
<!-- 阻止单击事件继续传播 -->
<a v-on:click.stop="doThis"></a>

<!-- 提交事件不再重载页面 -->
<form v-on:submit.prevent="onSubmit"></form>

<!-- 修饰符可以串联 -->
<a v-on:click.stop.prevent="doThat"></a>

<!-- 只有修饰符 -->
<form v-on:submit.prevent></form>

<!-- 添加事件监听器时使用事件捕获模式 -->
<!-- 即内部元素触发的事件先在此处理，然后才交由内部元素进行处理 -->
<div v-on:click.capture="doThis">...</div>

<!-- 只当在 event.target 是当前元素自身时触发处理函数 -->
<!-- 即事件不是从内部元素触发的 -->
<div v-on:click.self="doThat">...</div>

2.1.4新增
<!-- 点击事件将只会触发一次 -->
<a v-on:click.once="doThis"></a>
<!-- 不像其它只能对原生的 DOM 事件起作用的修饰符，.once 修饰符还能被用到自定义的组件事件上
```

```
**使用修饰符时，顺序很重要；相应的代码会以同样的顺序产生。**

**因此，用 `v-on:click.prevent.self` 会阻止所有的点击，而 `v-on:click.self.prevent` 只会阻止对元素自身的点击。**

```



## 3.按键修饰符

在监听键盘事件时，我们经常需要检查详细的按键。Vue 允许为 `v-on` 在监听键盘事件时添加按键修饰符：

```html
<!-- 只有在 `key` 是 `Enter` 时调用 `vm.submit()` -->
<input v-on:keyup.enter="submit">
```

你可以直接将 [`KeyboardEvent.key`](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key/Key_Values) 暴露的任意有效按键名转换为 kebab-case 来作为修饰符

```html
<!--处理函数只会在 $event.key 等于 PageDown 时被调用。-->
<input v-on:keyup.page-down="onPageDown">
```

# 八，表单输入绑定

```
`v-model` 会忽略所有表单元素的 `value`、`checked`、`selected` 特性的初始值而总是将 Vue 实例的数据作为数据来源。你应该通过 JavaScript 在组件的 `data` 选项中声明初始值。

```

`v-model` 在内部为不同的输入元素使用不同的属性并抛出不同的事件：

- text 和 textarea 元素使用 `value` 属性和 `input` 事件；
- checkbox 和 radio 使用 `checked` 属性和 `change` 事件；
- select 字段将 `value` 作为 prop 并将 `change` 作为事件。

## 修饰符

**.lazy**

```
默认情况下，`v-model` 在每次 `input` 事件触发后将输入框的值与数据进行同步 (除了[上述](https://cn.vuejs.org/v2/guide/forms.html#vmodel-ime-tip)输入法组合文字时)。可以添加 `lazy` 修饰符，从而转变为使用 `change` 事件进行同步：

```

```html
<!-- 在“change”时而非“input”时更新 -->
<input v-model.lazy="msg" >
```

**.number**

如果想自动将用户的输入值转为数值类型，可以给 `v-model` 添加 `number` 修饰符：

```html
<input v-model.number="age" type="number">
```

通常很有用，因为即使在 `type="number"` 时，HTML 输入元素的值也总会返回字符串。如果这个值无法被 `parseFloat()` 解析，则会返回原始的值。

**.trim**

如果要自动过滤用户输入的首尾空白字符，可以给 `v-model` 添加 `trim` 修饰符：

```html
<input v-model.trim="msg">
```

# 九，prop

## 1.传递静态或动态prop

### 传入一个对象

```html
<!-- 即便对象是静态的，我们仍然需要 `v-bind` 来告诉 Vue -->
<!-- 这是一个 JavaScript 表达式而不是一个字符串。-->
<blog-post v-bind:author="{name: 'Veronica',company: 'Veridian Dynamics'}"></blog-post>

<!-- 用一个变量进行动态赋值。-->
<blog-post v-bind:author="author"></blog-post>
```

### 传递一个对象的所有属性

```
想要将一个对象的所有属性都作为 prop 传入，你可以使用不带参数的 `v-bind` (取代 `v-bind:prop-name`)。例如，对于一个给定的对象 `post`：

```

```html
post: {id: 1, title: 'My Journey with Vue'}

<blog-post v-bind="post"></blog-post> 等价于：
<blog-post v-bind:id="post.id" v-bind:title="post.title"></blog-post>
```

**重点注意1：**

**你不应该在一个子组件内部改变 prop。如果你这样做了，Vue 会在浏览器的控制台中发出警告。**

这里有两种常见的试图改变一个 prop 的情形：

1. **这个 prop 用来传递一个初始值；这个子组件接下来希望将其作为一个本地的 prop 数据来使用。**在这种情况下，最好定义一个本地的 data 属性并将这个 prop 用作其初始值：

   ```
   props: ['initialCounter'],
   data: function () {
     return {
       counter: this.initialCounter
     }
   }
   
   ```

2. **这个 prop 以一种原始的值传入且需要进行转换。**在这种情况下，最好使用这个 prop 的值来定义一个计算属性：

   ```
   props: ['size'],
   computed: {
     normalizedSize: function () {
       return this.size.trim().toLowerCase()
     }
   }
   
   ```

**重点注意2：**

**在 JavaScript 中对象和数组是通过引用传入的，所以对于一个数组或对象类型的 prop 来说，在子组件中改变这个对象或数组本身将会影响到父组件的状态。**



# 钩子函数

created（）：

加载页面时，先执行该函数再加载页面

monted（）：

跳转组建时，先加载页面，再执行该函数



父组件调用子组件时，先加载子组件，再加载父组件



# API

### 1. $nextTick()

**作用：**

`Vue.nextTick`用于延迟执行一段代码，它接受2个参数（回调函数和执行回调函数的上下文环境），如果没有提供回调函数，那么将返回`promise`对象。

**使用场景：**

- 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据后立即执行nextTick(callback)，**获取更新后的DOM**，回调函数在 DOM 更新完成后就会调用

- 在Vue生命周期的`created()`钩子函数进行的DOM操作一定要放在`Vue.nextTick()`的回调函数中

  原因：在`created()`钩子函数执行的时候DOM 并未进行任何渲染，此时进行DOM操作无异于徒劳

- 在数据变化后要执行的某个操作，而这个操作需要使用随数据改变而改变的DOM结构的时候，这个操作都应该放进`Vue.nextTick()`的回调函数中

**理解：nextTick()，是将回调函数延迟在下一次dom更新数据后调用**，简单理解是：**当数据更新了，在dom中渲染后，自动执行该函数，**

例子展示：

```js
 <h4 ref="testh" @click="testClick">{{ msg }}</h4>
 
 data(){return{msg: '预警数据'}},
 methods:{
    testClick:function(){
      this.msg="修改后的值"; //此时DOM还未立即更新，this.$refs.testh获取指定DOM，输出：原始值
      # 1.在$nextTick()使用: 将会在数据修改后延迟执行回调函数，输出修改后的值
      this.$nextTick(function(){  # 执行$nextTick(),不会立即执行回调函数,继续往下
        console.log('延后执行',this.$refs.testh.innerText);  //输出：‘修改后的值’
      });
      # 2.直接操作DOM，DOM没来得及更新，将输出原始值
      console.log('先执行',this.$refs.testh.innerText); //原始值：‘预警数据’，
      # 注意：查看页面效果时，会发现视图中数据已经更新为‘修改后的值’
    }
 }
```



# vue生命周期

- `beforeCreate`:在实例初始化之后，数据观测data observer(`props、data、computed`) 和 `event/watcher` 事件配置之前被调用。
- `created`:实例已经创建完成之后被调用。在这一步，实例已完成以下的配置：数据观测(data observer)，属性和方法的运算， `watch/event` 事件回调。然而，挂载阶段还没开始，`$el` 属性目前不可见。
- `beforeMount`:在挂载开始之前被调用：相关的 `render` 函数首次被调用。
- `mounted`: `el` 被新创建的 `vm.$el` 替换，并挂载到实例上去之后调用该钩子。
- `beforeUpdate`:数据更新时调用，发生在虚拟 `DOM` 重新渲染和打补丁之前。 你可以在这个钩子中进一步地更改状态，这不会触发附加的重渲染过程。
- `updated`：无论是组件本身的数据变更，还是从父组件接收到的 `props` 或者从`vuex`里面拿到的数据有变更，都会触发虚拟 `DOM` 重新渲染和打补丁，并在之后调用 `updated`。
- `beforeDestroy`:实例销毁之前调用。在这一步，实例仍然完全可用。
- `destroyed`:`Vue` 实例销毁后调用。调用后，`Vue` 实例指示的所有东西都会解绑定，所有的事件监听器会被移除，所有的子实例也会被销毁。 该钩子在服务器端渲染期间不被调用。



# 过滤器

```js
# 在html模板中的两种用法：
    <!-- 在双花括号中 -->
    {{ message | filterTest }}
    <!-- 在 `v-bind` 中 -->
    <div :id="message | filterTest"></div>
--------------------------------------------------------------------------------
# 在组件script中添加filters方法:
export default {    
     data() {
        return {
         message:1   
        }
     },
    filters: {  
        filterTest(value) {   // value在这里是message的值
            if(value===1){
                return '最后输出这个值';
            }
        }
    }
}
```



# vuex的使用

src目录下创建vuex/store.js文件：

```js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)
export default new Vuex.Store({
  state: {
    globalName: '全局变量傻子兆'
  }
})
```

在组件中使用：

```js
<script>
import Store from '../vuex/store'

export default {
  data() {
    return {
      globalName:Store.state.globalName, // 使用全局变量
    }
  }
}
```



# vue中使用$refs

### ref 有三种用法：

1、ref 加在普通的元素上，用this.$refs.（ref值） 获取到的是dom元素

2、ref 加在子组件上，用this.$refs.（ref值） 获取到的是组件实例，可以使用组件的所有方法。在使用方法的时候直接this.$refs.（ref值）.方法（） 就可以使用了。

3、如何利用 v-for 和 ref 获取一组数组或者dom 节点

### 应注意的坑：

1、如果通过v-for 遍历想加不同的ref时记得加 :号，即 :ref =某变量 ;
这点和其他属性一样，如果是固定值就不需要加 :号，如果是变量记得加 :号。（加冒号的，说明后面的是一个变量或者表达式；没加冒号的后面就是对应的字符串常量（String））

2、通过 :ref =某变量 添加ref（即加了:号） ,如果想获取该ref时需要加 [0]，如this.$refs[refsArrayItem][0]；如果不是:ref =某变量的方式而是 ref =某字符串时则不需要加，如this.$refs[refsArrayItem]。

### 注意

**1、ref 需要在dom渲染完成后才会有，在使用的时候确保dom已经渲染完成。比如在生命周期 mounted(){} 钩子中调用，或者在 this.$nextTick(()=>{}) 中调用。**

**2、如果ref 是循环出来的，有多个重名，那么ref的值会是一个数组 ，此时要拿到单个的ref 只需要循环就可以了。**

### 使用

```js
<div id="app">
    <h1 ref="h1Ele">这是H1</h1>
    <hello ref="ho"></hello>
    <button @click="getref">获取H1元素</button>
</div>
# -----------------------------------------------------------------------
methods: {
        getref() {
          // 表示从 $refs对象 中, 获取 ref 属性值为: h1ele DOM元素或组件
           console.log(this.$refs.h1Ele.innerText);
           this.$refs.h1ele.style.color = 'red';// 修改html样式
          console.log(this.$refs.ho.msg);// 获取组件数据
          console.log(this.$refs.ho.test);// 获取组件的方法
        }
      }
```



# 实际操作问题

### 1.vue 中每次请求时使用请求头自动携带token

**#1.该方法刷新页面时设置的token会失效，不可取**

```js
// 保存token到localStorage
localStorage.setItem('my_token', token);
// 登录成功后将token放进axios对象默认请求头中，后端通过请求头中的 HTTP_AUTHORIZATION 获取token
this.$axios.defaults.headers.common["authorization"] = localStorage.my_token; 
```

**#2.写一个请求拦截器**

```js
// 请求拦截器
ajax.interceptors.request.use(
  config => {
    const token = localStorage.getItem('my_token')
    if(token){
      config.headers['Authorization'] = token
    }
    return config
  },
  error => {
    return Promise.reject(error)
  }
)
# 后端通过请求头中的 HTTP_AUTHORIZATION 获取token
```

### 2.vue中增删改后重载当前页面

**#1. window.reload()  和 this.$router.go(0):**

刷新时整个浏览器重新加载，会出现空白闪烁，体验效果不好

**#2. 使用provide和inject 组合**

provide：选项应该是一个对象或返回一个对象的函数。该对象包含可注入其子孙的属性。

inject：一个字符串数组，或一个对象，对象的 key 是本地的绑定名

作用：允许一个祖先组件向其所有子孙后代注入一个依赖，不论组件层次有多深，并在起上下游关系成立的时间里始终生效。

App.vue中：声明reload方法，控制router-view的显示或隐藏，从而控制页面的再次加载

objList.vue 中: 在页面注入App.vue组件提供（provide）的 reload 依赖，在逻辑完成之后（删除或添加...）,直接this.reload()调用，即可刷新当前页面。

例子：

```js
# App.vue：
<template>
  <div id="app">
    <router-view v-if="isRouterAlive"/>
  </div>
</template>

<script>
export default {
  name: 'App',
  provide(){
    return{
      reload:this.reload
    }
  },
  data() {
    return{
      isRouterAlive:true
    }
  },
  methods:{
    reload() {
      this.isRouterAlive = false;
      this.$nextTick(function(){
        this.isRouterAlive = true;
      })
    }
  }
}
</script>

# obj.vue:
 export default {
    inject: ['reload'],  //引入reload方法
    data() {
      }
    }
methods:{
    // 增删改操作后调用
    this.reload();
}
```

**#3. 不刷新页面，利用VUE组件通信，监听事件发生，重新调一下获取数据的接口**

a、给Vue的原型上添加一个reload属性

main.js：

```
Vue.prototype.$reload = new Vue()

```

b、home页面进行修改个人资料操作时触发事件

home.vue： 

```js
changeProfile （） {
    ......
	this.$reload.$emit('change')  // 主动触发change事件
}
```

c、页面1里监听如果执行了操作，就调取页面1需要重新加载的数据接口

```js
mounted() {
	// 初始化vue后，绑定change事件
    this.$reload.$on("change", () => {
      this.showPage(); // 重新调用加载的数据的接口
    });
  },
```



### 3.vue.js 获取兄弟元素，子元素，父元素

```js
<button @click = “clickfun($event)”>点击</button>
 
methods: {
clickfun(e) {
// e.target 是你当前点击的元素
// e.currentTarget 是你绑定事件的元素
    #获得点击元素的前一个元素
    e.currentTarget.previousElementSibling.innerHTML
    #获得点击元素的第一个子元素
    e.currentTarget.firstElementChild
    # 获得点击元素的下一个元素
    e.currentTarget.nextElementSibling
    # 获得点击元素中id为string的元素
    e.currentTarget.getElementById("string")
    # 获得点击元素的string属性
    e.currentTarget.getAttributeNode('string')
    # 获得点击元素的父级元素
    e.currentTarget.parentElement
    # 获得点击元素的前一个元素的第一个子元素的HTML值
    e.currentTarget.previousElementSibling.firstElementChild.innerHTML
 
    }
```

### 4.Vue动态给一个元素添加类名，兄弟元素移除类名

```html
html:
<ul><li  v-for="(item,index) in list" @click="do(item,index)" ：class="{active:index==current}">{{item.name}}
</li></ul>
js:
data(){return{current:0}}
methods:{do(item,index){this.current=index}}
style: 
.active{color:red}
```

### 5.在vue组件上直接添加原生事件

**由于组件上面的事件默认为自定义事件，所以不会根据相应操作触发，例如：**

<custom @click='search'> < /custom>  并不会点击触发

**#1. 给事件加个后缀.native就行 **

```html
<single-button button-type-prop="search" @click.native="search"></single-button>
```

**#2. 通过$emit方法绑定**

通过子组件内部元素上的原生事件去主动触发组件实例上的自定义事件



### 6.组件间传递数据

#### #1 通过$emit 和$on来实现兄弟组件间传递数据

a.首先创建一个Vue空白实例（兄弟间的桥梁）

```js
# 方法一：在一个js文件中暴露出去，使用时引用
import Vue from 'vue'
export default new Vue()
# 方法二：在入口文件main.js 中将Vue实例写入vue原型
Vue.prototype.$eventBus = new Vue()
```

b.子组件a使用$emit触发自定义事件，并传递数据

```js
click1 () {
      // emptyVue.$emit('aevent', this.msg) #导入的空Vue
      this.$eventBus.$emit('aevent', this.msg) # 写入原型的空Vue
},
```

c.子组件b使用$on监听自定义事件的触发，并接受数据

```js
this.$eventBus.$on('aevent', (val) => {
      console.log(val) // this is msg from childa
      this.msg = val
}
```

**注意：父子组件之间传参时，父组件中模板上的 v-on 也是属于监听自定义事件，例如这样：**

**（触发和监听都必须使用同一个vue实例来完成， 即emit和on只能作用在一一对应的同一个组件实例，而v-on只能作用在父组件引入子组件后的模板上。）**

```html
<template>
  <div id="app">
    <child-a @my-event="getMyEvent"></child-a>
  </div>
</template>
```

#### #2 使用$refs来实现父子组件传递数据（父传子）

a.父组件中调用子组件中的方法并传值（触发子组件方法）

```html
<template>
  <div id="app">
    <child-a ref="child"></child-a>
    <button @click="getMyEvent">点击父组件</button>
  </div>
</template>
# -------------------------------
getMyEvent(){
	this.$refs.child.getMsg(this.msg) # 调用子组件的方法（算是去触发子组件中的该方法）
}
```

b.子组件中

```js
getMsg(msg){
    console.log('接收的数据-->'+msg)//接收父组件中的数据
}
```

### 7.vue-router响应路由参数的变化

**问题**：从一个页面路由跳转到另一个页面，如果这俩页面都渲染同个vue组件（跳相同的地址），则vue实例不会被销毁而是继续复用；**这也意味着组件的生命周期钩子不会再被调用**。即在钩子函数中，无法获取路由参数。

```js
// 钩子函数不会调用
mounted() {
    console.log(this.$route.query.index) 
},
```

复用组件时，想对路由参数的变化作出响应的话，解决办法：

【法1】**添加watch监听**：

**【重要】：使用 handler  和 immediate 在创建时就监听value**

```js
watch: {
 //监听路由是否变化
  '$route' (to, from) {
   if(to.query.id !== from.query.id){
            this.id = to.query.id;
            this.init(); //do something
        }
  }
}
// 注意：当vue实例第一次创建时，watch内函数不会执行；
// watch对value属性的监听会在value第一次变化后开始，如果我们想在创建时监听value，要使用 handler  和 immediate
watch: {
  value:{   // value 两边引号无影响
    handler:function(o,n){},
    immediate: true
  } 
},
```

【法2】**使用router的组件内钩子函数：**

```js
beforeRouteUpdate (to, from, next) {
    // react to route changes...
    // don't forget to call next()
  }
```



### 有用的函数

##### 1.object.assign()

`Object.assign（）`方法用于将所有可枚举属性的值从一个或多个源对象复制到目标对象。它将返回目标对象。

```js
var obj = { a: 1 };
var copy = Object.assign({}, obj);
console.log(copy); // { a: 1 }
# Object.assign()是浅拷贝。

var o1 = { a: 1 };
var o2 = { b: 2 };
var o3 = { c: 3 };

var obj = Object.assign(o1, o2, o3);
console.log(obj); // { a: 1, b: 2, c: 3 }
console.log(o1);  // { a: 1, b: 2, c: 3 }, 注意目标对象自身也会改变。
```

**注意，具有相同属性的对象，同名属性，后边的会覆盖前边的。**

##### 使用场景

```
由于Object.assign()有上述特性，所以我们在Vue中可以这样使用：
Vue组件可能会有这样的需求：在某种情况下，需要重置Vue组件的data数据。此时，我们可以通过this.$data获取当前状态下的data，通过this.$options.data()获取该组件初始状态下的data。然后只要使用Object.assign(this.$data, this.$options.data())就可以将当前状态的data重置为初始状态

```



# 实例属性

### $data, $options, $el, $props, $parent, $root, $children, $slots

```js
export default {
  data(){
    return{
      msg: '预警数据',
    }
  },
  customOption: 'foo', //自定义初始化选项
  methods:{
    test(){
      // console.log(this.data) //undefined
      // console.log(this.$data)
      // console.log(this.$el)
      // // console.log(this.$props)
      // console.log(this.$options, '初始化选项实例')
      // console.log(this.$options.data) 应该是data()
      // console.log(this.$options.methods)
      // console.log(this.$options.customOption,'自定义初始化选项')
      // console.log(this.$parent, '父实例') // 父实例
      // console.log(this.$root, '根实例') // 根实例，若没有父实例，则是本身
      // console.log(this.$children, '直接子实例') // 当前实例的直接子组件
      // console.log(this.$slots, '') //用来访问被 slot 分发的内容。每个具名 slot 有其相应的属性（例如：slot="foo" 中的内容将会在 vm.$slots.foo 中被找到）。default 属性包括了所有没有被包含在具名 slot 中的节点。
      // console.log(this.$scopedSlots, '') //
      // console.log(this.$refs,'所有拥有 ref 注册的子组件的对象') // 一个对象，其中包含了所有拥有 ref 注册的子组件
      // console.log(this.$isServer, '') //当前Vue实例是否运行于服务器；
    }
  }
}
```



# 踩坑

**1.更新 `v-for` 渲染的列表时，js值已经改变，但界面没有正常渲染**

我是使用新的数组直接替换`v-for`渲染的数组，但是渲染顺序不正确。

**解决办法：**

使用`v-for`必须要提供一个唯一key属性，且**不要使用对象或数组**之类的非基本类型值作为 `v-for` 的 `key`。请**用字符串或数值**类型的值。

**详细原因**（来自官方文档）：

当 Vue 正在更新使用 `v-for` 渲染的元素列表时，它默认使用“就地更新”的策略。如果数据项的顺序被改变，Vue 将不会移动 DOM 元素来匹配数据项的顺序，而是就地更新每个元素，并且确保它们在每个索引位置正确渲染。

这个默认的模式是高效的，但是只适用于不依赖子组件状态或临时 DOM 状态 (例如：表单输入值) 的列表渲染输出。

为了给 Vue 一个提示，以便它能跟踪每个节点的身份，从而重用和重新排序现有元素，需要为每项提供一个唯一 `key` 属性。



