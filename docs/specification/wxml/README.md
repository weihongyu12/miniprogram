---
description: 微信小程序 WXML 规范，涵盖标签自闭合、属性顺序、事件绑定等编码约定
---

# WXML 规范

规则按以下等级划分：

- **必要的**：涉及编译错误、运行时 Bug 或显著影响包体积，必须遵守
- **强烈推荐**：影响代码可读性、可维护性，建议遵守
- **推荐**：风格统一、最佳实践，按团队需要采纳

## 必要的

### 空标签必须是自闭合的标签

所有微信小程序 wxml 标签都可以自关闭。空标签自闭合可以减少代码体积，便于代码审查。

:::danger[反面例子 👎]
```wxml
<view bind:tap="clickhandler"></view>

<view>
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view bind:tap="clickhandler" />
<view>{{interpolation}}</view>
<view>text</view>
<view><sub /></view>
```
:::

### 统一事件绑定样式

事件绑定必须在整个项目内统一为冒号样式（`bind:tap`）或无冒号样式（`bindtap`），不可混用。本规范推荐使用冒号样式，语义更清晰。

:::danger[反面例子 👎]
```wxml
<view bindtap="clickhandler" />
<view mut-bindtap="clickhandler" />
<view catchtap="clickhandler" />
<view capture-catchtap="clickhandler" />
<view capture-bindtap="clickhandler" />
```
:::

:::tip[正面例子 👍]
```wxml
<view bind:tap="clickhandler" />
<view mut-bind:tap="clickhandler" />
<view catch:tap="clickhandler" />
<view capture-catch:tap="clickhandler" />
<view capture-bind:tap="clickhandler" />
```
:::

### 禁用特定标签

禁止在 wxml 中使用 HTML 标签（如 `div`、`span`、`p`）或项目约定的其他禁用标签，应使用小程序原生标签（如 `view`、`text`）替代。可按需配置禁用清单及替代提示，也可基于属性决定是否禁用某标签。

:::danger[反面例子 👎]
```wxml
<div>{{name}}</div>
<span>{{title}}</span>
<!-- 提示：请使用 <text /> -->
<p>{{title}}</p>
```
:::

:::tip[正面例子 👍]
```wxml
<view>{{name}}</view>
<text>{{title}}</text>
```
:::

### wxs 中不允许使用 let 和 const

微信小程序的 wxs 运行时目前不支持 `let` 和 `const` 声明变量，必须使用 `var`。误用会导致运行时错误。

:::danger[反面例子 👎]
```wxml
<wxs module="util">
  let s = 100;
  const k = {};
  module.exports = {
    data: s,
    obj: k
  }
</wxs>
```
:::

:::tip[正面例子 👍]
```wxml
<wxs module="util">
  var s = 100;
  module.exports = {
    data: s
  }
</wxs>
```
:::

### 起止标签名必须一致

开发阶段容易出现手误导致起始标签与结束标签名不一致（如 `<view>` 配 `</viw>`），此类问题在编译期不一定能立即暴露。

:::danger[反面例子 👎]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="goodsId"
>
  {{item.name}}
</viw>
```
:::

:::tip[正面例子 👍]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="goodsId"
>
  {{item.name}}
</view>
<same-tag-name>
  {{"tag name must be equal"}}
</same-tag-name>
```
:::

### 不使用字符串形式的布尔值

在数据绑定中，`checked="false"` 会被当作非空字符串，转换为布尔值时始终为 `true`。布尔值必须通过 `{{}}` 表达式书写；对于值为 `true` 的布尔属性，应直接省略值。

:::danger[反面例子 👎]
```wxml
<checkbox checked="false"> </checkbox>
<popup showMask="true" />
```
:::

:::tip[正面例子 👍]
```wxml
<checkbox checked="{{false}}"> </checkbox>
<popup showMask />
```
:::

### wx:for 不能与 wx:else 同时使用

在同一个标签上同时使用 `wx:for` 与 `wx:else` 会导致小程序编译错误。如需在 `wx:else` 分支中渲染列表，应使用 `<block />` 包裹。

:::danger[反面例子 👎]
```wxml
<view wx:if="{{showList}}" wx:for="{{list}}"> {{item.name}}</view>
<view wx:else wx:for="{{lastList}}"> {{item.name}}</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view wx:for="{{list}}"> {{item.name}}</view>
<view wx:if="{{showList}}" wx:for="{{list}}"> {{item.name}}</view>
<view wx:elif="{{showOtherList}}" wx:for="{{otherList}}"> {{item.name}}</view>
<!-- 使用 <block /> 包裹循环列表是官方推荐写法 -->
<block wx:else>
  <view wx:for="{{otherList}}"> {{item.name}}</view>
</block>
```
:::

### wx:if 的值必须是布尔表达式

`wx:if` / `wx:elif` 的值若为字符串或拼接字符串（如 `"{{show}} "`、`"{{show}}-s"`），在小程序中会被当作非空字符串而恒为 `true`，导致条件判断失效。必须确保值为布尔表达式。

:::danger[反面例子 👎]
```wxml
<!-- {{}} 末尾的空格会让表达式恒为 true -->
<view wx:if="{{showList}} "> 我会一直显示 </view>
<view wx:if="string"> 我会一直显示 </view>
<view wx:if="{{showSwitch}}-string"> 我会一直显示 </view>
<view wx:elif="string"> 我会一直显示 </view>
```
:::

:::tip[正面例子 👍]
```wxml
<view wx:if="{{user}}"> {{user.name}}</view>
<view wx:elif="{{show}}">show this view</view>
```
:::

### 省略值为 true 的布尔属性

当布尔属性值为 `true` 时应省略值，直接书写属性名，可以减少代码体积并提升可读性。值为 `false` 或动态绑定时仍需显式书写。

:::danger[反面例子 👎]
```wxml
<cart showBadge="{{true}}" />
<swiper autoplay="{{true}}" />
```
:::

:::tip[正面例子 👍]
```wxml
<cart showBadge />
<swiper autoplay />
<virtual-list hideSpinner="{{false}}" />
```
:::

### 检查插值语法错误

`{{}}` 插值表达式存在语法错误时（如不完整的表达式），应在开发期及时发现并修复。

:::danger[反面例子 👎]
```wxml
<view>
  <view style="idx-{{isOdd ? 'single'}}" />
  {{ a + }}
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view>
  <view style="idx-{{isOdd ? 'single' : ''}}" />
  {{ a + b }}
</view>
```
:::

### wx:for 必须指定 wx:key

`wx:for` 渲染列表时必须指定 `wx:key`，用于标识列表项的唯一性，保证列表项在动态变化时能正确复用，提升渲染性能。仅当列表静态且顺序不重要时才可忽略。

:::danger[反面例子 👎]
```wxml
<view wx:for="{{list}}">
  <view>item.title</view>
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view wx:for="{{list}}" wx:key="goodsId">
  <view>item.title</view>
</view>
```
:::

### wxs 必须有合法的 module 属性

`<wxs>` 标签必须提供 `module` 属性，且命名需符合规则：首字符必须是英文字母（a-z、A-Z）或下划线（`_`），其余字符可为英文字母、下划线或数字。同一 wxml 文件内 `module` 名应唯一。

:::danger[反面例子 👎]
```wxml
<wxs module="0util" src="../../utils.wxs" />
<wxs module="-util" src="../../utils.wxs" />
<wxs module="^util" src="../../utils.wxs" />
<wxs src="../../utils.wxs" />
<wxs>
  function show () {
    return "show";
  }
  module.exports = {
    func: show
  }
</wxs>
```
:::

:::tip[正面例子 👍]
```wxml
<wxs module="util" src="../../utils.wxs" />
<wxs module="_internel" src="../../utils.wxs" />
<wxs module="mode">
  function show () {
    return "show";
  }
  module.exports = {
    func: show
  }
</wxs>
```
:::

### 禁用特定语法（no-restricted-syntax）

本规则源自 ESLint 内置能力，允许通过 AST 选择器精确禁用指定的 wxml 语法模式。规则非常强大，可用于实现自定义的代码约束。

支持的选择器语法包括：

- AST 节点类型：`ForStatement`
- 通配符（匹配所有节点）：`*`
- 属性存在：`[attr]`
- 属性值：`[attr="foo"]` 或 `[attr=123]`
- 属性正则：`[attr=/foo.*/]`
- 属性条件：`[attr!="foo"]`、`[attr>2]`、`[attr<3]`、`[attr>=2]`、`[attr<=3]`
- 嵌套属性：`[attr.level2="foo"]`
- 字段：`FunctionDeclaration > Identifier.id`
- 首尾子节点：`:first-child` 或 `:last-child`
- 第 n 个子节点：`:nth-child(2)`
- 后代：`FunctionExpression ReturnStatement`
- 子节点：`UnaryExpression > Literal`
- 后续兄弟：`VariableDeclaration ~ VariableDeclaration`
- 相邻兄弟：`ArrayExpression > Literal + SpreadElement`
- 取反：`:not(ForStatement)`
- 多选：`:matches([attr] > :first-child, :last-child)`
- AST 节点类：`:statement`、`:expression`、`:declaration`、`:function`、`:pattern`

常见用法示例：

:::danger[反面例子 👎]
```wxml
<!-- 禁止使用 class="class" -->
<view class="class"></view>

<!-- 禁止使用 <wxs> 标签 -->
<wxs src="../../utils.wxs" />
```
:::

:::tip[正面例子 👍]
```wxml
<view class="main"></view>
```
:::

## 强烈推荐

### 限制标签最大嵌套深度

标签嵌套过深会导致代码难以阅读和维护。建议将最大嵌套深度限制在合理范围（推荐 10 层以内）。

:::danger[反面例子 👎]
```wxml
<view depth="1">
  <view depth="2">
    <view depth="3">
      <view depth="4">
        <view depth="5">
          <view depth="6" />
        </view>
      </view>
    </view>
  </view>
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view depth="1">
  <view depth="2">
    <view depth="3"></view>
  </view>
  <view depth="2"></view>
</view>
```
:::

### 限制单行最大长度

过长的单行代码难以阅读。建议限制单行字符数（推荐 100 字符以内，可配置是否忽略纯空白行）。

:::danger[反面例子 👎]
```wxml
<view depth="1"><view depth="2"><view depth="3"/><view depth="3"/><view depth="3"/></view></view>
```
:::

:::tip[正面例子 👍]
```wxml
<view depth="1">
  <view depth="2">
    <view depth="3" />
  </view>
</view>
```
:::

### 不允许重复属性

同一标签上不应出现重复属性，重复属性会被后者覆盖且属于明显的代码错误。

:::danger[反面例子 👎]
```wxml
<view
  title="{{title}}"
  name="{{name}}"
  name="{{other}}"
>
  <goods />
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view
  title="{{title}}"
  name="{{name}}"
>
  <goods />
</view>
```
:::

### 必填属性检查

对于自定义组件或特定标签，可约定必须填写的属性（如 `<popup>` 必须填写 `showMask`）。规则可按标签配置必填属性清单及默认值，便于团队统一约定。

:::danger[反面例子 👎]
```wxml
<!-- 约定 <popup> 必须填写 showMask，但此处缺失 -->
<popup name="welcome">
  <img useWebp />
</popup>
```
:::

:::tip[正面例子 👍]
```wxml
<popup name="welcome" showMask>
  <img useWebp />
</popup>
```
:::

## 推荐

### 限制文件最大行数

单个 wxml 文件行数过多通常意味着承担了过多职责。建议限制单文件最大行数（推荐 300 行以内），可配置是否跳过空行计数。

:::danger[反面例子 👎]
```wxml
<!-- 单文件超过约定的最大行数，包含上千行结构 -->
<view line="1">
  <view line="2">
    ...
  </view>
  ...
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view line="1">
  <view line="2">
    <view line="3"></view>
  </view>
</view>
```
:::

### 不使用 *this 作为 wx:key

`wx:key="*this"` 仅适用于列表项本身为唯一字符串或数字的场景（如 `string[]`、`number[]`）。通用场景下应使用列表项的唯一属性作为 `wx:key`，避免潜在问题。

:::danger[反面例子 👎]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="*this"
>
  {{item.name}}
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="goodsId"
>
  {{item.name}}
</view>
```
:::

### wx:key 必须是静态值

`wx:key` 用于在列表渲染时标识列表项唯一性，其值必须是静态字符串（指向列表项的属性名），不能使用动态插值表达式。

:::danger[反面例子 👎]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="id-{{goodsId}}"
>
  {{item.name}}
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="goodsId"
>
  {{item.name}}
</view>
```
:::

### 不使用 index 作为 wx:key

`wx:key="index"` 等同于使用列表索引作为 key，当列表项动态变化或新增时无法保证正确复用，应使用列表项自身的唯一标识属性。

:::danger[反面例子 👎]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="index"
>
  {{item.name}}
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view
  wx:for="{{goodsList}}"
  wx:key="goodsId"
>
  {{item.name}}
</view>
```
:::

### 使用外部 .wxs 文件

推荐将 wxs 逻辑写在独立的 `.wxs` 文件中并通过 `src` 引入，便于获得完整的 IDE 语言支持（如 VSCode 会将 `.wxs` 识别为 `.js`）。

:::danger[反面例子 👎]
```wxml
<wxs module="util">
  function util () {
    // balabala
  }
  module.exports = {
    util: util
  }
</wxs>
```
:::

:::tip[正面例子 👍]
```wxml
<wxs module="util" src="../../../util.wxs" />
```
:::

### 不使用不必要的 block 标签

`<block />` 仅作为包装元素，不进行任何渲染。当 `<block />` 内只有一个子元素且不涉及 `wx:if/wx:for` 的特殊组合时，应直接将控制属性写在子元素上以减少代码层级。

特殊情况：当需要用 `wx:if` 包裹 `wx:for` 列表时，由于小程序禁止 `wx:for` 与 `wx:if/wx:elif/wx:else` 同时出现在同一标签，此时允许使用单子元素的 `<block />` 作为占位。

:::danger[反面例子 👎]
```wxml
<block wx:for="{{list}}" wx:key="id">
  <goods name="{{item.name}}" img="{{item.imgUrl}}" />
</block>
<block wx:if="{{show}}"></block>
<block wx:if="{{show}}">
  <view>
    <sub-view />
  </view>
</block>
```
:::

:::tip[正面例子 👍]
```wxml
<block wx:if="{{show}}">{{title}}</block>
<block wx:if="{{show}}">
  <multi-children />
  <view>
    <sub-view />
  </view>
</block>
<!-- 使用 <block /> 包裹循环列表是官方推荐写法 -->
<block wx:if="{{showList}}">
  <view wx:for="{{list}}"> {{item.name}}</view>
</block>
```
:::

### 不使用 Vue 指令

有 Vue 开发背景的同学容易因肌肉记忆在小程序中误写 `v-if`、`v-else`、`v-for` 等 Vue 指令。应使用小程序原生的 `wx:if`、`wx:else`、`wx:for` 等指令。

:::danger[反面例子 👎]
```wxml
<view v-if="{{show}}"> {{title}}</view>
<view v-else-if="{{hide}}"> {{title}}</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view wx:if="{{show}}"> {{title}}</view>
<view wx:elif="{{hide}}"> {{title}}</view>
```
:::

### 统一引号风格

wxml 中单引号和双引号均合法，但项目内应统一为一种风格。本规范推荐使用双引号。

:::danger[反面例子 👎]
```wxml
<component attr='{{data}}' />
<view wx:if='{{show}}'> {{title}}</view>
```
:::

:::tip[正面例子 👍]
```wxml
<component attr="{{data}}" />
<view wx:if="{{show}}"> {{title}}</view>
```
:::

### 检查 WXML 语法错误

wxml 语法错误（如未闭合的标签、未闭合的插值表达式）应在开发期及时发现并修复。

:::danger[反面例子 👎]
```wxml
<view>
  <view />
  {{
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<view>
  <view />
  {{title}}
</view>
```
:::

### 检查内联 wxs 语法错误

内联 `<wxs>` 标签中的 JavaScript 语法错误应在开发期及时发现并修复。

:::danger[反面例子 👎]
```wxml
<wxs module="util">
  funcytion ss () {

  }
</wxs>
```
:::

:::tip[正面例子 👍]
```wxml
<wxs module="util">
  function ss () {

  }
</wxs>
```
:::

### 必填根标签

对于特定类型的 wxml 文件，可约定必须使用某个标签作为根标签（如所有 `*-page.wxml` 必须以 `<page>` 作为根标签）。

:::danger[反面例子 👎]
```wxml
<!-- 约定根标签为 <page>，但此处根标签为 <app> -->
<app name="e-commerce">
  <main />
  <list />
</app>
```
:::

:::tip[正面例子 👍]
```wxml
<page name="e-commerce">
  <main />
  <list />
</page>
```
:::

### 必填标签

对于特定 wxml 文件，可约定必须包含某些标签（如所有页面 wxml 必须包含 `<page>` 标签用于注入公共逻辑）。

:::danger[反面例子 👎]
```wxml
<!-- 约定必须包含 <page> 标签，但此处缺失 -->
<app name="e-commerce">
  <main />
  <list />
</app>
```
:::

:::tip[正面例子 👍]
```wxml
<app name="e-commerce">
  <page />
  <main />
  <list />
</app>
```
:::

### wxs 必须位于顶层

`<wxs>` 标签可以写在 wxml 文件的顶层，也可以嵌套在其他标签内部。为代码风格统一，推荐所有 `<wxs>` 标签都写在文件顶层。

:::danger[反面例子 👎]
```wxml
<view>
  <view>
    <!-- 嵌套的 wxs -->
    <wxs module="util" src="../../../util.wxs" />
  </view>
</view>
```
:::

:::tip[正面例子 👍]
```wxml
<!-- 顶层 wxs -->
<wxs module="util" src="../../utils.wxs" />

<view>
  <view />
</view>
```
:::
