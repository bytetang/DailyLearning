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


### 一、A Simple Vue.js example ###
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

### 二、基于Vue.js的weex文件 ###
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

### 三、官方demo ###
Weex的文档的确是值得吐槽的。官方的Demo想跑起来遇坑无数，weex文件最终得打包成js文才能被执行。使用weex自带打包工具weex-toolkit
整体打包各种依赖报错，如require("**module")。issue里面说需要在注销依赖才能打包成功，额。。。

最终参照这篇文章<a href="http://blog.csdn.net/jizi7618937/article/details/51611629">点击访问</a>利用npm解决依赖。得有解决。
使用下面方式打包js并运行在浏览器上：
<p>
1、安装webpack<br>
npm install webpack
</p>
<p>
2、解决依赖
npm run build<br>
需要在喻package.json文件统一位置文件下执行
</p>
<p>
4、打包js<br>
npm run build //会生成在build目录下生成js文件<br>
</p>
<p>
5、chrome运行<br>
npm run serve & <br>
>注意是命令关键字是serve 不是 server
</p>
<p>
weex项目发布在12580端口<br>
weex@0.4.0 serve /Users/tangjie/projects/weex<br>
serve ./ -p 12580
<p>
使用浏览器访问http://127.0.0.1:12580/<br>
![Alt text](https://github.com/tangchiech/UpLearn/blob/master/Android/pics/weex_run.png)







