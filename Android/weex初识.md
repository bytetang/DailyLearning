##weex 初识##

一句话概括就是
>A framework for building Mobile cross-platform UI.

面向开发者就是使用Javascript来开发native级别的weex界面。兼具web开发方式的灵活跨平台和接近于native方式的执行效率。我们都知道，H5沸沸扬扬的叫嚣
说到底运用在webview上还是一个大坑。虽是解决性能的痛点，weex在复杂的交互和页面上还是不够完善，希望开源之后可以不断的被推动完善。

基础
<ui>
<li>JavaScrip/Css/Html</li>
<li>Vue.js</li>
<li>Android/IOS</li>
</ui>

Vue.js是一个灵活的前端组件化框架，支持响应式的数据绑定，拥有自己的实例生命周期。可以进入[Vue.js官网教程](https://vuejs.org.cn/guide/"Title") 学习.


### 1、A Simple Vue.js example ###
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Document</title>
    <link rel="stylesheet" type="text/css" href="https://cdn.bootcss.com/bootstrap/3.3.5/css/bootstrap.min.css">
     <script type="text/javascript" src="http://cdn.jsdelivr.net/vue/1.0.7/vue.min.js" ></script>
     <script type="text/javascript" src="http://cdnjs.cloudflare.com/ajax/libs/vue/1.0.7/vue.min.js"></script>
</head>
<body>
	 <div class="container">
        <div class="col-md-6 col-md-offset-3">
            <h1>Vue example3</h1>
            <div id="app">
	           <p>{{message}}</p>
	           <button v-on:click="revertMessage">Revert Message</button>
            </div>
        </div>
    </div>

    <script type="text/javascript">
    	new Vue({
    		el:'#app',
    		data:{
    			message:'hello vue.js'
    		},
    		methods:{
    			revertMessage:function(){
    				this.message = this.message.split('').reverse().join('')
    			}
    		}
    	})
    </script>
</body>
</html>
```
new Vue创建出一个Vue实例，{{message}}语法可实现双向的绑定，Vue实例中提供message属性的值。在html和Vue实例之间亦可简单的完成事件的绑定操作。

### 2、基于Vue.js的weex文件 ###
weex的安装步骤先安装nodejs，使用npm安装weex工具，[步骤详情](https://github.com/alibaba/weex).
weex基于weex，主要有三个模块```<template>```、```<style>```、```<script>```。
```html
<template>
  <!-- (required) the structure of page -->
</template>

<style>
  /* (optional) stylesheet */
</style>

<script>
  /* (optional) the definition of data, methods and life-circle */
</script>
```

语法和Vue.js基本类似，在temp中定义你的界面内容，style中定义内容样式，同样可以是用绑定的语法,如
```html <text style="font-size: {{fontSize}};">Alibaba</text>``` 。script中提供数据的绑定和事件的处理等，weex具有生命周期，在script
可以在对生命周期的控制中分别控制不同的逻辑，如create、ready、computed等。


