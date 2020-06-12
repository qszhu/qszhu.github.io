---
layout: post
title: 一次转账失败的排查（更新中）
date: 2020-06-12 22:37:00 +0800
tags: [idempotency, updating]
---

* 重复转账
	* 接口幂等
	* 消息队列幂等
	* 调用非幂等
	* 函数修改
```javascript
async function foo() {
  return xxx()
}
bar = await foo()
```
```javascript
async function foo() {
  const res = xxx()
  queue.add(xxx)
  return res
}
bar = await foo()
```
```javascript
async function foo() {
const res = await xxx()
queue.add(xxx)
return res
}
bar = await foo()
```
	* 日志系统
		* 一次输出多行日志，页数多，没有跳转
		* 从后往前翻页，窗口随着日志量移动
	* 状态切换使用状态机
