---
lang: zh-cmn-Hans-CN
title: WXML 规范
---

# WXML 规范

## 必要的

### 空标签必须是自闭合的标签

所有微信小程序 wxml 标签都可以自关闭

:::danger 反面例子 👎
```wxml
<view></view>
```
:::

:::tip 正面例子 👍
```wxml
<view />
```
:::

### 使用冒号样式的事件绑定

:::danger 反面例子 👎
```wxml
<view bindtap="onTap" />
<view mut-bindtap="onTap" />
<view catchtap="onTap" />
<view capture-catchtap="onTap" />
<view capture-bindtap="onTap" />
```
:::

:::tip 正面例子 👍
```wxml
<view bind:tap="onTap" />
<view mut-bind:tap="onTap" />
<view catch:tap="onTap" />
<view capture-catch:tap="onTap" />
<view capture-bind:tap="onTap" />
```
:::

## 强烈推荐

## 推荐
