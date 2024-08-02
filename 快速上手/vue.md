## 安装
### npm安装
```
npm install vue
```
### 嵌入现有工程

- 开发版本：https://vuejs.org/js/vue.js
- 发布版本：https://vuejs.org/js/vue.min.js
- 如果代码里含有模版（template），那么需要使用完整版本（含有编译器），否则只需要runtime版本（体积少30%）

``` html
<script src="https://cdn.jsdelivr.net/npm/vue@2.6.6/dist/vue.js"></script>
```

### 配合webpack使用
```
module.exports = {
  // ...
  resolve: {
    alias: {
    	// 用 webpack 1 时需用 'vue/dist/vue.common.js'
      'vue$': 'vue/dist/vue.esm.js'
    }
  }
}
```
## HelloWorld
``` html
<div id="app">
  {{ message }}
</div>
```
``` javascript
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

## v语法
### v-if
``` html
<p v-if="seen">现在你看到我了</p>
<p v-else-if="type === 'B'">else if</p>
<p v-else>No</p>
```
为了防止if else 之间元素复用，可以在元素上添加key="唯一标识"来区别

### v-for
``` html
<ol>
  <li v-for="todo in todos">
    {{ todo.text }}
  </li>
</ol>
```
``` javascript
todos: [
      { text: '学习 JavaScript' },
      { text: '学习 Vue' },
      { text: '整个牛项目' }
    ]
```

### v-bind
把 message 和 span.title 属性绑定

``` html
<span v-bind:title="message">
  鼠标悬停几秒钟查看此处动态绑定的提示信息！
</span>
```
v-bind: 可以缩写成 :

``` html
<div :id="dynamicId"></div>
```
在布尔特性的情况下，存在即暗示为 true。<br/>
如果 isButtonDisabled 的值是 null、undefined 或 false，disabled属性都不生效

``` html
<button v-bind:disabled="isButtonDisabled">Button</button>
```

v-bind 对 class 和 style 做了加强，可以用数组方式进行处理：

``` html
<div class="someClass"
  v-bind:class="{ active: isActive, 'text-danger': hasError }">
</div>
```
``` javascript
data: {
  isActive: true,
  hasError: false
}
```

还可以直接绑定数组：

``` html
<div v-bind:class="classObject"></div>
```
``` javascript
data: {
  classObject: {
    active: true,
    'text-danger': false
  }
}
```
还可以绑定计算属性：

``` javascript
computed: {
  classObject: function () {
    return {
      active: this.isActive && !this.error,
      'text-danger': this.error && this.error.type === 'fatal'
    }
  }
}
```
还可以使用数组语法，使用数组变量对应的名字来作为classname：

``` html
<div v-bind:class="[activeClass, errorClass]"></div>
<div v-bind:class="[{ active: isActive }, errorClass]"></div>
```
``` javascript
data: {
  isActive: true,
  activeClass: 'active',
  errorClass: 'text-danger'
}
```
绑定样式：

``` html
<div v-bind:style="{ color: activeColor, fontSize: fontSize + 'px' }"></div>
```

### v-on
v-on: 可以缩写成 @

``` html
<div id="app-5">
  <p>{{ message }}</p>
  <button v-on:click="reverseMessage">逆转消息</button>
  <button @click="reverseMessage">逆转消息</button>
</div>
```
``` javascript
var app5 = new Vue({
  el: '#app-5',
  data: {
    message: 'Hello Vue.js!'
  },
  methods: {
    reverseMessage: function () {
      this.message = this.message.split('').reverse().join('')
    }
  }
})
```

### 修饰符.
用于指出指令应以特殊方式绑定。例如：<br/>
.prevent 修饰符告诉 v-on 指令对于触发的事件调用event.preventDefault()：

``` html
<form v-on:submit.prevent="onSubmit">...</form>
```

### v-model
``` html
<input v-model="message">
```

### v-once
``` html
<span v-once>这个将不会改变: {{ msg }}</span>
```

### v-html
当绑定的变量要显示html时，使用v-html="变量名" 而不是 {{ 变量名 }}

``` html
<p><span v-html="rawHtml"></span></p>
```

### 变量表达式
支持以下表达式，但是不支持赋值和if判断式

``` html
{{ number + 1 }}

{{ ok ? 'YES' : 'NO' }}

{{ message.split('').reverse().join('') }}

<div v-bind:id="'list-' + id"></div>
```

## 计算属性
注意计算属性的依赖原值如果不更新，计算属性不会重新计算。<br/>
如果希望每次访问属性都更新，可以调用方法。

``` html
<div id="example">
  <p>属性: "{{ message }}"</p>
  <p>计算属性: "{{ reversedMessage }}"</p>
  <p>调用方法: "{{ updateCall() }}"</p>
</div>
```
``` javascript
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Hello'
  },
  computed: {
    // 计算属性的getter
    reversedMessage: function () {
      // this指向vm实例
      return this.message.split('').reverse().join('')
    }
  },
  methods: {
    updateCall: function () {
      return Date.now() //Date.now() 不是响应式依赖，如果放在计算属性里将不会更新
    }
  },
})
```

### 计算属性setter
``` javascript
var vm = new Vue({
  computed: {
    // 计算属性的getter
    reversedMessage: {
      get: function() {
        return this.message.split('').reverse().join('')
      },
      set: function(newValue) {
        this.message = newValue.split('').reverse().join('')
      },
    }
  },
})
```

## 侦听属性
``` javascript
var watchVM = new Vue({
  el: '#watch-example',
  data: {
    question: '',
  },
  watch: {
    // 如果 `question` 发生改变，这个函数就会运行
    question: function (newQuestion, oldQuestion) {
      // do something
    }
  },
});
```

## 组件
``` html
<div id="app-7">
  <ol>
    <todo-item
      v-for="item in groceryList"
      v-bind:todo="item"
      v-bind:key="item.id">
    </todo-item>
  </ol>
</div>
```
``` javascript
Vue.component('todo-item', {
  props: ['todo'],
  template: '<li>{{ todo.text }}</li>'
})
var app7 = new Vue({
  el: '#app-7',
  data: {
    groceryList: [
      { id: 0, text: '蔬菜' },
      { id: 1, text: '奶酪' },
      { id: 2, text: '随便其它什么东西' }
    ]
  }
})
```

## 生命周期
不要在选项属性或回调上使用箭头函数

![](https://cn.vuejs.org/images/lifecycle.png)

## 其他特性
- 支持浏览器：ie9以上
- 使用 Object.freeze() 阻止属性自动更新

## 相关链接

- 框架理念：https://player.youku.com/embed/XMzMwMTYyODMyNA==?autoplay=true&client_id=37ae6144009e277d
- 对比其他框架：https://cn.vuejs.org/v2/guide/comparison.html
