---
title: Nodejs14大版本特性以及版本管理n模块
cover: /upload/homePage/20210517184300.jpg
tags:
  - nodejs
abbrlink: 6a76cba2
categories: uncategorized
date: 2021-04-26 14:43:09
---
## 情景
前段时间在Github看到有这么一种写法：

```
data = $request(params)
if (data?.data?.bizCode === 0) {
    console.log(`任务完成成功，获得：${data?.data?.result?.produceScore ?? "未知"}能量`);
}
```

这段代码中包含两种操作符'?.'和'??'，之前都没有看到过，突然想起node14版本已经出来很久了，立马去翻了一下新版本的文档，果然这两个操作符都是为了解决以前开发中很经常遇到的两种问题。

## Nodejs14重点功能介绍

### Optional Chaining
#### 官方介绍节选
The optional chaining operator (?.) enables you to read the value of a property located deep within a chain of connected objects without having to check that each reference in the chain is valid.
#### 粗糙的直译
可选的链接操作符(?.)，使你读取位于链接对象链深处的属性的值，而不必检查链中的每个引用是否有效。
#### 自我感觉
这个操作符我感觉十分有用，以前不管是在前端还是Nodejs服务端，我们发起请求获取远端接口返回的数据时，经常会获取到一个比较复杂的对象，比如调用登录接口获取user对象，这个user对象里包含address对象，address中有一个city字段，数据结构如下所示：
```
const user = {
  name: 'jack',
  address: {
    city: 'Qingdao'
  }
}
```

此时我们需要city的值，我们需要对user和user.address都做非空判断，否则就会报出Cannot read property 'xxx' of undefined 这样的类似错误。
```
if (user && user.address) {
  console.log(user.address.city)
}
```

这种情况经常会遇到，像这样的判断代码到处都是，而且即使这样有时候也很容易漏写，导致报出undefined。现在使用'可选链操作符'，就不必对链中的每个引用值都判断是否有效了，只需要用符号'?.'表示，这样再引用为null或undefined时就不会报错，会直接短路返回undefined。

```
let city = user.address?.city
```
[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining)

### Nullish Coalescing
#### 官方介绍节选
The nullish coalescing operator (??) is a logical operator that returns its right-hand side operand when its left-hand side operand is null or undefined, and otherwise returns its left-hand side operand.
#### 粗糙的直译
空值合并操作符(??)是一个逻辑运算符，当其左侧的操作数为null或undefined时，返回其右侧的操作数，否则返回其左侧的操作数。
#### 自我感觉
这个操作符是对一种特殊情况的补充，以前我们使用逻辑或操作符(||)会在左侧为false时返回右侧的操作数，例如我们传入一个0，此时期望输出左侧的值是不行的。
```
const baz = 0 || 42;
console.log(baz);
// expected output: 42
```
此时左值0满足false的要求，因此会输出42。

```
const baz = 0 ?? 42;
console.log(baz);
// expected output: 0
```
现在我们可以使用空值合并操作符(??)来实现，仅当左侧为null或undefined时才会返回右侧的值。

[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_Coalescing_Operator)

### Intl.DisplayNames
#### 官方介绍节选
The Intl.DisplayNames object enables the consistent translation of language, region and script display names.
#### 粗糙的直译
Intl.DisplayNames对象启用语言，区域和脚本显示名称一致的翻译。
#### 自我感觉
i18n，nodejs也有官方的支持了，这个没什么好说明的，下面贴一下官方的示例：
```
const regionNamesInEnglish = new Intl.DisplayNames(['en'], { type: 'region' });
const regionNamesInTraditionalChinese = new Intl.DisplayNames(['zh-Hant'], { type: 'region' });

console.log(regionNamesInEnglish.of('US'));
// expected output: "United States"

console.log(regionNamesInTraditionalChinese.of('US'));
// expected output: "美國"
```

[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DisplayNames)

### Enables calendar and numberingSystem options for Intl.DateTimeFormat
#### 官方介绍节选
The Intl.DateTimeFormat object enables language-sensitive date and time formatting.
#### 粗糙的直译
Intl.DateTimeFormat对象启用对语言敏感的日期和时间格式。
#### 自我感觉
这个也是i18的内容，对于服务端的nodejs来说日期和时间的国际化有了更简洁的写法，见下方官方示例：
```
const date = new Date(Date.UTC(2020, 11, 20, 3, 23, 16, 738));
// Results below assume UTC timezone - your results may vary

// Specify default date formatting for language (locale)
console.log(new Intl.DateTimeFormat('en-US').format(date));
// expected output: "12/20/2020"

// Specify default date formatting for language with a fallback language (in this case Indonesian)
console.log(new Intl.DateTimeFormat(['ban', 'id']).format(date));
// expected output: "20/12/2020"

// Specify date and time format using "style" options (i.e. full, long, medium, short)
console.log(new Intl.DateTimeFormat('en-GB', { dateStyle: 'full', timeStyle: 'long' }).format(date));
// Expected output "Sunday, 20 December 2020 at 14:23:16 GMT+11"
```

[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat)

## Node.js版本管理n模块
上述的这些版本特性需要升级node14版本，提到版本升级，这里就不得不说到n模块了，n模块是专门用来管理node的版本的。
使用n模块可以快速的更新node的版本，也可以在多个版本之间快捷的切换，十分的方便。

在使用n模块之前，我们首先要安装一下：
```
npm install -g n
```

使用npm在全局安装n模块，安装后就可以使用n模块升级node.js了。

```
# 升级node.js到最新稳定版
n stable
# 也可以指定版本号来安装
n v12.13.1
n v16.0.0
```

如果安装了多个版本，直接在命令行输入n，可以进入版本切换的界面，使用上/下选择对应版本即可快速的切换环境中使用的node.js版本。

```
#  ln -s [源文件位置] [目标文件位置]
ln -s /usr/bak/nodejs/node-v12.16.0-linux-x64/bin/n /usr/local/bin/n
```
n模块安装后，如果经常使用，别忘了对其添加软链，这样在任意目录就可以直接输入n来使用了。

## 注意事项
window不支持n模块，如果需要类似的功能可以使用[nvm-windows](https://github.com/coreybutler/nvm-windows)。

另外node.js 14版本已经不支持window7了，如果想在window系统上体验新特性的同学，只能选择升级win10了。

## 参考资料
[Node.js version 14 available now](https://nodejs.medium.com/node-js-version-14-available-now-8170d384567e)
[Nodejs 14 大版本中新增特性总结](https://blog.csdn.net/xgangzai/article/details/114361520)
