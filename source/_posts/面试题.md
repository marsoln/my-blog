---
title: 有趣的面试题
date: 2017-05-12 13:40:41
tags: 
---

## 在 [surmon.me](https://surmon.me/) 上看到的一个面试题

```javascript
let greeting = 'My name is ${name}, age ${age}, I am a ${job.jobName}'

let employee = {
  name: 'XiaoMing',
  age: 11,
  job: {
    jobName: 'designer',
    jobLevel: 'senior'
  }
}

let result = greeting.render(employee)

console.log(result) // My name is XiaoMing, age 11, I am a designer

```

第一时间想到的方案就是使用正则表达式匹配到对应的属性,然后根据属性名称替换为实际值


```javascript

String.prototype.render = function(ctx){
  return this.replace(/\x24\x7b([\w\.]+)\x7d/g, (matchStr, attrName) => {
    return ctx[attrName]
  })
}

```

然后问题出现了,当 `attrName` 匹配到 `"job.jobName"` 时,通过索引的方式取值会变成 `ctx["job.jobName"]`,得到 `undefined`

接下来,考虑拆分属性名称

```javascript

String.prototype.render = function(ctx){
  return this.replace(/\x24\x7b([\w\.]+)\x7d/g, (matchStr, attrName) => {
    let attrPaths = attrName.split(/\./)
    return attrPaths.reduce((entity,attr)=> entity[attr], ctx)
  })
}

```

如此一来,就会将匹配到的 `"job.jobName"` 拆分为 `"job"` 和 `"jobName"` 两个路径,然后调用归约函数 第一次得到 `ctx["job"]` 下次一得到 `ctx["job"]["jobName"]` 然后返回该值.

为了程序健壮可以再稍微完善一下
```javascript
String.prototype.render = function (ctx) {
  return this.replace(/\x24\x7b([\w\.]+)\x7d/g, (matchStr, key) => key.split('\.').reduce((obj, attr) => (obj||{})[attr], ctx) || matchStr)
}
```

最后,看到有人给出的答案是这样的

```javascript
String.prototype.render = function (ctx) {
  with (ctx) {
    return eval('`' + this + '`')
  }
}
```

给跪了, OTZ
