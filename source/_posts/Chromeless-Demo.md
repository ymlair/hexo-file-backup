---
title: Chromeless Demo
date: 2018-01-31 09:50:33
tags: chromeless
---

## Chromeless 简介
Chrome 浏览器有一种模式叫做 Chrome Headless，在这种模式下，允许你正常运行 Chrome 浏览器，但是没有界面；想要调试这种模式下打开的网站，可以通过它提供的接口来实现，而 [Chromeless](https://github.com/graphcool/chromeless) 就是把这层接口做了封装，让你使用接口更方便。通过它，可以控制浏览器行为，如打开网站、点击按钮、填充表单、获取 DOM 元素...
## 可以用来做什么
1 .  可以[获取网页截图](https://github.com/graphcool/chromeless/blob/master/examples/google-screenshot.js)
2 .  根据页面 document 文档[生成 PDF 文件](https://github.com/graphcool/chromeless/blob/master/examples/google-pdf.js)
3 .  编写测试代码，自动化测试网页
4 .  基于真实的浏览器环境，可以编写爬虫程序
## 安装
首先要安装支持 Chrome Headless 模式的浏览器。目前，Mac 上 Chrome 59 beta
版本与 Linux 上的 Chrome 57+ 已经开始支持 headless 特性。Windows 上 Chrome 暂时不支持，可以使用 [Chrome Canary 60](https://www.google.com/chrome/browser/canary.html) 进行开发。
下面是 Ubuntu17.10 的无界面浏览器启动命令示例：
```
google-chrome \
--remote-debugging-port=9222 \
--disable-gpu \
--no-sandbox  \
--headless
```
Chromeless 用 NodeJS 编写，要求 NodeJS 版本8.2+，安装：
`npm install chromeless`

## 如何用于网页测试
```
const { Chromeless } = require('chromeless')
const { expect }  = require('chai')

async function run(){
	const chromeless = new Chromeless()
	const firstPage= await chromeless
		.goto('http://www.w3school.com.cn')
		.wait('#navsecond')
		.evaluate(() => {
  			  // this will be executed in Chrome
  			  const links = [].map.call(
  			    document.querySelectorAll('#navsecond ul:nth-child(2) li'),
  			    a => (a.innerText.trim())
  			  )
  			  return links
  			})
expect(firstPage).to.have.members(["JS","HTML5","XHTML","CSS","CSS3","TCP/IP"])
await chromeless.end()
}
# 运行并捕捉错误
run().catch(console.error.bind(console))
```
这里用到了 [Chai](http://chaijs.com/api/bdd/) 断言库，代码实现的功能是验证 [W3school](http://www.w3school.com.cn) 网站首页，左侧第一个 ul 列表的内容是否包含以下内容： JS、HTML5、XHTML、CSS、CSS3、TCP/IP"。
命令详解：
goto：打开加载网站 `http://www.w3school.com.cn`；
wait：等待指定的元素 `#navsecond`（这是 [CSS selector](http://www.w3school.com.cn/cssref/css_selectors.asp)） 渲染成功之后才往下执行；
evaluate：会将里面的 JS 代码送到 浏览器中执行，并获取返回结果；
运行结果：
![图一：网页测试Demo结果](http://upload-images.jianshu.io/upload_images/5306603-2d9f002f40084a27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 如何爬取 Google Search Result
下面代码实现的功能是使用 Google Search 搜索关键字，并返回结果的 Title 和链接地址。
[查看执行结果](https://asciinema.org/a/KlCxZMtSAlpQj9AHhSs8kSIVq)

```
const { Chromeless } = require('chromeless');

async function run(){
	const chromeless = new Chromeless()
	const firstPage= await chromeless
		.goto('https://www.google.com')
		.wait('input[name="q"]')
		.type('云纵', 'input[name="q"]')
		.press(13) // press enter
		.wait('#foot')
		
	let hasNextPage = true
	let page = 1
	let result = null
	while (hasNextPage){
		if(page===1){
			result = await chromeless
				.evaluate(() => {
  				  // this will be executed in Chrome
  				  const links = [].map.call(
  				    document.querySelectorAll('.g h3 a'),
  				    a => ({title: a.innerText, href: a.href})
  				  )
  				  return links
  				})
		}else{
			result = await chromeless
				.scrollTo(200,400)
				.scrollToElement('#foot td:nth-last-child(1)')
				.click('#foot td:nth-last-child(1)')
				.wait(2000)
				.wait('#foot')
				.evaluate(() => {
	                          // this will be executed in Chrome
	                          const links = [].map.call(
	                            document.querySelectorAll('.g h3 a'),
	                            a => ({title: a.innerText, href: a.href})
	                          )
	                          return links
	                        })
		}
		console.log(result)
		hasNextPage = await chromeless.evaluate(() => {
  			  let nextPage =  document.querySelector('#foot td:nth-last-child(1)').innerText
  			  //return (nextPage==='Next')
  			  return nextPage.length
  			})
		console.log(`第${page}页`)
		console.log(hasNextPage ? '存在下一页' : '不存在下一页')
		page++
	}
	await chromeless.end()
}

run().catch(console.error.bind(console))

```
命令解释：
`.type('云纵', 'input[name="q"]')`： 在 CSS 选择器 `input[name="q"]` 选中的元素（实际上就是 Google Search 搜索框）内输入’云纵‘两个字；
`.press(13)` ：按下键盘的 Enter 键，就是在搜索框输入文字后执行查询；
 `.scrollTo(200,400)` ：滚动到距离页面左侧200px，右侧400px的位置；
`.scrollToElement('#foot td:nth-last-child(1)')`：滚动到 CSS 选择器 `#foot td:nth-last-child(1)` 选中元素（实际是 Google Search Result 的下一页链接），使元素可见；
`.click('#foot td:nth-last-child(1)')`：点击元素；
`.wait(2000)`：参数是数字时，表示等待时间，参数是 CSS 选择器时表示等待元素渲染完成；

## Async 与 Await
开发时要注意这两个关键字 async 和 await，使用 Chromeless，会用到他们，这是 ES7 实现的异步方案，需要了解以下内容：
1 .  function 前面的 async 标识符表示函数内有异步操作；
2 .  await 强调后面的异步操作执行完成后才能继续；
3 .  await 只能用在async标识的函数内；
4 .  await 后写非异步操作也可以，会直接执行

## 参考资料
1 .  [体验异步的终极解决方案-ES7的Async/Await](https://cnodejs.org/topic/5640b80d3a6aa72c5e0030b6)
2 .  [初探 Headless Chrome](https://zhuanlan.zhihu.com/p/27100187)

