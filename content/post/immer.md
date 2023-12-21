---
title: 基于immer的RTK撤销与重做
date:  2021-11-22T23:15:28+08:00
draft: false
tags:
  - frontend
  - react
---

## 提要与背景

撤销与重做是可视化编辑器项目类似于[draw.io](https://draw.io)涉及到一个常用的需求, 也是之前旧版本一直没填上的坑. 用户操作的入口和复杂程度都算是比较高的, 所以撤销重做的界定(主要体现在连续的拖拉拽移动, 缩放, 旋转)是一个比较关键的点. 

对于这次撤销与重做的核心的选型是因为项目中使用了 `@reduxjs/toolkit`(下文会简写为`RTK`) 这个redux官方封装的功能增强版轮子, 轮子里面是使用`immer`作为`immutable`的实现方式. 所以利用了`immer`中记录`patch`的特性来进行差分回滚.

---

## 前置知识

通常情况下实现数据可逆有两种方式: **快照与差分**.
### 快照
快照的思想十分简单, 直接将执行后的完整状态作为数据推入栈, 由于每一条记录都包含了编辑器上所有组件完整的状态, 栈中相邻两条记录可以是相互独立的没有耦合, 所以状态之间的切换可以是跳跃的 (可以直接从状态2回到状态4而不考虑中间有多少个状态), 回溯状态的计算量不会随着记录跳跃距离而增长.
  ![快照](/post/immer/snapshot.png)
  <!-- <img src="/post/immer/snapshot.png" /> -->

-  一种比较容易理解的快照机制为深拷贝,常见实现方法为`JSON.parse(JSON.stringify())`,`lodash.cloneDeep()`等. 从内存角度来看, 每个状态都是独立一块内存区域, 哪怕是两个相邻记录之间只有很小的改动, 在此方法下的快照都会完整的将状态深拷贝一份作为备份, 这个方法的缺点也是深拷贝带来的内存空间的浪费, 如果条历史记录记录的状态特别多, 亦或者是历史记录栈特别长的时候, 内存问题在本文中提到的所有记录数据方法中将是最差的.


- 在深拷贝之上的改进方法则是借助不可变库(`immutablejs`, `immerjs`)来解决上者带来的内存问题, 当不可变性介入时, 传统的深拷贝操作将被不可变计算代替, 使得运行过程中只需要记录状态的顶层引用即可.如图所示, 状态`state1`修改`child 2` 后来到 `child 2'`, 根据不可变库的实现, 该属性向父级递归的更新其引用, 使得`parent1` 与 `state1`引用地址获得了更新, 而`child1`, `child3` 以及`parent2`及其子节点保持原有引用地址不变, 实现了未修改处的复用.

![immutable](/post/immer/immutable.png)


根据这个特性, 快照的模式也可以很大程度上视作为只关注修改的属性的差分模式, 而单条记录也只需要保存顶层的根节点地址`state`,状态的回溯也只需要在`redux`顶层进行相应筛选(过滤无需记录的状态)与提交. **对于不涉及到字符串序列化通信的撤销(例如历史记录云同步, 跨标签页同步记录), 使用该方法是最方便的**


### 差分补丁
差分补丁主要关注于每两次之间的状态差分, 需要在状态转移时, 生成相应的正向与反向补丁, 这两种补丁保证了两个状态之间的来回切换而不需要保存完整的状态信息. 
> 差分补丁的描述载荷比较依赖于业务性的数据，如果需要描述通用的数据操作，描述对象的属性会靠近类似AST一样的语法树格式

> 一种更暴力的方式就是对每个操作入口进行收敛， 人为的去提取信息并作记录编写对应的正逆向补丁，对于动态程度比较高（例如传key value的反射取值）写出来的代码就会比较难看了



差分更新的单条记录的量不会过大, 且补丁独立于状态库单独存储, 但仍然受到修改程度影响, 所以当一次性修改很多属性的时候, 补丁的大小也会等量的增长, 所以在性能方面仍然不及使用不可变库模式下的快照. 其优势主要体现在与序列化开销上, 因为就算是不可变库加持下的快照回溯, 其优势只能体现在单个js运行环境中, 一旦涉及到与后端交互的撤销逻辑时(比如实时保存), 快照的模式决定了它将回到深拷贝方式, 当进行序列化通信的时候, 没有了js环境, 所有的数据回归到字符串, 实现方式只能回归到深拷贝的`JSON.stringify`, 原来的内存占用过大问题转化为请求体过大问题. 而差分补丁的方式则是最大程度上减少请求体里所携带的信息.<br />

---
## 功能设计

撤销与重做设计思想类似命令模式, 开发者事先注册好确定的指令, 赋予这些指令功能的同时还能够同步记录一些其他运行时的元数据, 在用户交互(执行命令)的同时, 将命令推入栈中方便程序作后续处理亦或者在满足某些条件的情况下丢弃.

套用到当前的项目业务场景中去,可以归纳为
- 这些注册好的确定的指令就是提供给用户的, 用户交互发生时直接/间接需要调用到的方法. 
- 推入栈的数据记录了可以重现用户操作的数据.
- 在实现以上两点的前提下， 降低代码侵入性提高动态更新的支持

### 补丁数据

撤销重做的核心就是正向与逆向的补丁数据，常用的不可变库`immer`，其实就提供了这个记录补丁数据的能力， 包括正向与逆向， 只是该功能默认不开启。而`RTK`对`redux`的封装正是基于`immer`来实现的

>显式地开启补丁功能[官方文档](https://immerjs.github.io/immer/patches/). 
```javascript
import { enablePatches } from 'immer'

enablePatches();
```
开启后即可在`produce`方法中,传入一个回调作为第三个参数, 该回调将会获得`produce`执行过程中对`draft`对象所修改的所有行为的补丁, 以`patches`标注为正向补丁, `inversePatches`标注为反向回溯补丁, 同时`immer`还提供了使用补丁的方法`applyPatches`来应用刚刚输出的正反向补丁


同时`immer`也提供了对`正向/逆向补丁`的工具函数`applyPatches`:
```javascript
import produce, { applyPatches } from 'immer';

let state = {
  name: "Micheal",
  age: 32
}
let changes = []

let inverseChanges = []

let state = produce(
  fork,
  draft => {
    draft.age = 33
  },
  (patches, inversePatches) => {
    changes.push(...patches)
    inverseChanges.push(...inversePatches)
  }
)
let lastState = applyPatches(state, inverseChanges)
expect(lastState).toEqual({
  name: "Micheal", 
  age: 32 
})
``` 
于是我们撤销重做的改造就会围绕着这两份数据进行

<!-- 至此`immer`已经帮我们完成了最难的一部分, 而`@reduxjs/toolkit`中就使用了`immer`作为依赖库, 现在要做的就是把每次`produce` (`createSlice`中实现`reducer`的核心)生成的`patches`与`inversePatches`成对的保存在一条记录里作为历史. -->

### 抽象命令注册方法
从现有的项目出发, 用户的操作会带来数据上的改变, 但不是所有的操作都需要历史记录 例如切换页签, 调整编辑器画布缩放程度这类不影响最终编辑器生产的结果的数据/状态. 编辑器应该记录的是那些,影响最终产物的数据, 例如某个组件的长宽,布局位置等, 在先前的开发过程中已经在`redux`中把这两类的状态区分开,所以我们现在主需要对那些有影响力(应该准确回显)的数据进行操作历史记录.

 `redux`的核心是一个个`reducer`, 我们在定义这些`reducer`的过程中其实也是在向`redux`注册一个又一个的命令, 我们可以在复用这一块的注册行为的基础上去扩展以最终达到注册历史记录命令的目的. 在 `@reduxjs/tookit`的[官网示例](https://redux-toolkit.js.org/api/createSlice#reducers)上是这样定义的:

>
``` javascript
import { createSlice, nanoid } from '@reduxjs/toolkit'

const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: {
      reducer: (state, action) => {
        state.push(action.payload)
      },
      prepare: (text) => {
        const id = nanoid()
        return { payload: { id, text } }
      },
    },
  },
})
```
`reducer`的模式天然契合，于是设想我们是否可以给每个`reducer`额外添加一份回调， 用作撤销回退的入口, 于是设想它应该是这样的
```diff 
{
  reducer: (state, action) => {
    state.push(action.payload)
  },
  prepare: (text) => {
    const id = nanoid()
    return { payload: { id, text } }
  },
+ patch:(patches, inversePatches, action) => {
+   //...
+ }
},
```

__修改源码__

在默认的`@redux/toolkit`中, 并没有实现上述功能, 于是需要**修改源码**, 让`immer`在`reducer`执行时记录差分补丁. `immer`作用的区域是`createReducer.ts`这个文件, 首先需要在其引入模块部分开启补丁模式. 而`createReducer.ts`会被`createSlice.ts`文件引用，同时为了保证改造后的类型推断完整， `creatReducer.ts`中的类型涉及到文件也要修改，经排查只有`mapBuilders.ts`, 也会被`createSlice.ts`引用

RTK的源码拆分比较灵活，我们可以通过实现自己的`createSlice` `createReducer`等接口来替换原本的功能，做到最小程度的改动
> 本文基于`@reduxjs/toolkit@1.8.6`版本进行改造，后续官方包更新可能会导致不可用，但几率不大

1. 首先开启`immer`相关的api, 把需要定义的类型也导入
```diff
/* createReducer.ts */
- import type { Draft } from 'immer'
- import createNextState, { isDraft, isDraftable, enableES5 } from 'immer'
+ import type { Draft, Patch } from 'immer'
+ import createNextState, { isDraft, isDraftable, enableES5, enablePatches } from 'immer'
  import type { AnyAction, Action, Reducer } from 'redux'
...
+ enablePatches();

```



### 历史记录栈
- `immer`产生的`pathes`需要按照顺序进行存储 ，通过维护一个全局栈来实现， 并且需要用一个值来记录当前的回退位置。
- 通常来说，只有在产生新操作记录的时候才会对当前栈进行切分， 舍弃记录位置之后的所有`patches`

