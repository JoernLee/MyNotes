[toc]

这里包括下文讨论到的chrome插件其实是官方所指的chrome extension，考虑到目前主流的叫法皆以chrome插件代称

# chrome插件能干什么
引用官网的话：

> 扩展程序是可以定制浏览体验的小型软件程序。它使用户可以根据个人需要或偏好来定制 Chrome 功能和行为。它们是基于 Web 技术（例如 HTML，JavaScript 和 CSS）构建的。 扩展由相互联系的各种组件组成，组件可以包括后台脚本，内容脚本，选项页，UI 元素和各种逻辑文件。

- 书签控制
- 下载控制
- 窗口控制
- 标签控制
- 网络请求控制，各类事件监听
- 自定义原生菜单
- 完善的通信机制

我们常见的包括很多网页比价工具、取色器、掘金默认标签页，甚至是redux开发调试工具，json浏览器工具等都是通过chrome插件实现的

感兴趣的可以去chrome官方应用商店看看：https://chrome.google.com/webstore/category/extensions

# chrome插件的版本
注意chrome插件也是有官方版本的，不同版本之间对chrome浏览器的支持也不一样

考虑到兼容性这里的教程采用v2版本进行开发和展示

# chrome插件的浏览器支持
首先，毋庸置疑的是chrome浏览器对chrome插件的支持肯定是最好的

其次，webkit内核的浏览器（国内360浏览器，搜狗浏览器等）一般来说也是可以运行的

最后，firefox浏览器目前也对chrome插件提供了一定的支持

也就是说，无论考虑浏览器支持还是网上资料以及开发难度上，chrome插件都是在人力有限情况下的首选方案~

# chrome插件的功能组成
chrome插件是有好几个组件组成的，各个组件的用途以及后续开发调试方式也会有些许差异，这里各位先了解一下~

## manifest.json
该文件是chrome插件的配置文件，目的是告诉加载插件的浏览器各个资源文件去哪里取，插件配置信息等等

这个是每个chrome插件必不可少的组成部分

## popup
独立的弹出页面，就是浏览器右上角一排插件图标点击之后所弹出的页面就是popup

该页面由相对独立的html，css和js组成

这一部分的开发方式也是和传统前端项目最为接近

![image](https://img2020.cnblogs.com/i-beta/1022803/202003/1022803-20200310111417807-1627849050.png)

## content script
这部分通常是一段JS脚本，目的通常是获取/修改目标页面的DOM，例如你想在目标网页添加一些组件UI都需要借助该脚本实现

注意该脚本和页面脚本的JS是隔离的，无法直接访问，你没有办法直接拿到目标页面的某些变量来使用（下文会介绍一些其他方法来进行通信）

但是CSS不是隔离的，所以相同命名的CSS会存在污染的情况，通常需要保证此脚本内部CSS命名的唯一性

## background script
这一段是运行在浏览器后台的脚本，一般会将需要一直运行的代码放在这里

他的API权限很高，能访问很多chromeAPI，同时不受跨域限制

# chrome插件的展现形式
最后针对chrome插件再介绍一下其存在的一些形态，通常都会在manifest.json文件中进行定义

## browserAction(浏览器右上角)

```
"browser_action":
{
	"default_icon": "img/icon.png",
	"default_title": "这是Chrome插件",
	"default_popup": "popup.html"
}
```

这样配置之后会在浏览器右上角增加一个插件图标，这也是使用较多的展现形式

## pageAction(浏览器右上角)

```
// manifest.json
{
	"page_action":
	{
		"default_icon": "img/icon.png",
		"default_title": "我是pageAction",
		"default_popup": "popup.html"
	},
	"permissions": ["declarativeContent"]
}

// background.js
chrome.runtime.onInstalled.addListener(function(){
	chrome.declarativeContent.onPageChanged.removeRules(undefined, function(){
		chrome.declarativeContent.onPageChanged.addRules([
			{
				conditions: [
					// 只有打开百度才显示pageAction
					new chrome.declarativeContent.PageStateMatcher({pageUrl: {urlContains: 'baidu.com'}})
				],
				actions: [new chrome.declarativeContent.ShowPageAction()]
			}
		]);
	});
});
```
整体上和browserAction一致，但是pageAction可以在background-script中定义符合条件的网页，只有在特定情况下来才会enable我们的插件

## 右键菜单

```
// manifest.json
{"permissions": ["contextMenus"]}

// background.js
chrome.contextMenus.create({
	type: 'normal'， // 类型，可选：["normal", "checkbox", "radio", "separator"]，默认 normal
	title: '菜单的名字', // 显示的文字，除非为“separator”类型否则此参数必需，如果类型为“selection”，可以使用%s显示选定的文本
	contexts: ['page'], // 上下文环境，可选：["all", "page", "frame", "selection", "link", "editable", "image", "video", "audio"]，默认page
	onclick: function(){}, // 单击时触发的方法
	parentId: 1, // 右键菜单项的父菜单项ID。指定父菜单项将会使此菜单项成为父菜单项的子菜单
	documentUrlPatterns: 'https://*.baidu.com/*' // 只在某些页面显示此右键菜单
});
// 删除某一个菜单项
chrome.contextMenus.remove(menuItemId)；
// 删除所有自定义右键菜单
chrome.contextMenus.removeAll();
// 更新某一个菜单项
chrome.contextMenus.update(menuItemId, updateProperties);
```

chrome插件也可以自定义右键菜单，当用户单机鼠标右键时机会弹出插件菜单进行功能的触发

![image](https://img1.baidu.com/it/u=4070188329,2413235862&fm=26&fmt=auto)

## override(覆盖特定页面)

```
"chrome_url_overrides":
{
	"newtab": "newtab.html",
	"history": "history.html",
	"bookmarks": "bookmarks.html"
}
```

用于将chrome默认提供的页面替换掉，例如新标签页，书签等等

掘金的标签页导航就是基于此能力

## devtools(开发者工具)

```
{
	// 只能指向一个HTML文件，不能是JS文件
	"devtools_page": "devtools.html"
}

// devtools.html
<!DOCTYPE html>
<html>
<head></head>
<body>
	<script type="text/javascript" src="js/devtools.js"></script>
</body>
</html>

// devtools.js

// 创建自定义面板，同一个插件可以创建多个自定义面板
// 几个参数依次为：panel标题、图标（其实设置了也没地方显示）、要加载的页面、加载成功后的回调
chrome.devtools.panels.create('MyPanel', 'img/icon.png', 'mypanel.html', function(panel)
{
	console.log('自定义面板创建成功！'); // 注意这个log一般看不到
});

// 创建自定义侧边栏
chrome.devtools.panels.elements.createSidebarPane("Images", function(sidebar)
{
	// sidebar.setPage('../sidebar.html'); // 指定加载某个页面
	sidebar.setExpression('document.querySelectorAll("img")', 'All Images'); // 通过表达式来指定
	//sidebar.setObject({aaa: 111, bbb: 'Hello World!'}); // 直接设置显示某个对象
});
```
我们常用的react或者vue控制条调试工具就是这种类型的插件，允许你在控制台中自定义自己的侧边栏以及其中的内容

devtool页面可以访问一组专属API，可以以获取页面的审查窗口信息以及网络请求等信息

# 回顾总结
- 了解chrome插件是什么
- chrome插件的用途
- chrome插件的核心组成
- chrome插件的展现形式








