---
title: 前端撤销与重做的探索
date:  2021-11-22T23:15:28+08:00
draft: false
tags:
  - frontend
  - react
  - immer
  - redux
  - zustand
---

### 提要与背景

撤销与重做是可视化编辑器项目类似于[draw.io](https://draw.io)涉及到一个常用的需求, 也是之前旧版本一直没填上的坑. 用户操作的入口和复杂程度都算是比较高的, 所以撤销重做的界定(主要体现在连续的拖拉拽移动, 缩放, 旋转)是一个比较关键的点. 

对于这次撤销与重做的核心的选型是因为项目中使用了 `@reduxjs/toolkit`(下文会简写为`RTK`) 这个redux官方封装的功能增强版轮子, 轮子里面是使用`immer`作为`immutable`的实现方式. 所以利用了`immer`中记录`patch`的特性来进行差分回滚.

---

### 前置知识

这个需求按功能模块可以划分为, 数据可逆模块 + 历史栈展现模块。  通常情况下实现数据可逆有两种方式: **快照与差分**.
### 快照
快照的思想比较容易理解, 直接将执行后的完整状态作为数据推入栈, 由于每一条记录都包含了应用上所有组件完整的状态, 栈中相邻两条记录可以是相互独立的没有耦合, 所以状态之间的切换可以是跳跃的 (可以直接从状态2回到状态4而不考虑中间有多少个状态), 回溯状态的计算量不会随着记录跳跃距离而增长.
  ![快照](/post/immer/snapshot.png)
  <!-- <img src="/post/immer/snapshot.png" /> -->

-  一种比较容易理解的快照机制为深拷贝,常见实现方法为`JSON.parse(JSON.stringify())`,`lodash.cloneDeep()`等. 从内存角度来看, 每个状态都是独立一块内存区域, 哪怕是两个相邻记录之间只有很小的改动, 在此方法下的快照都会完整的将状态深拷贝一份作为备份, 这个方法的缺点也是深拷贝带来的内存空间的浪费, 如果条历史记录记录的状态特别多, 亦或者是历史记录栈特别长的时候, 内存问题在本文中提到的所有记录数据方法中将是最差的.


- 在深拷贝之上的改进方法则是借助不可变库(`immutablejs`, `immerjs`)来解决上者带来的内存问题, 当不可变性介入时, 传统的深拷贝操作将被不可变计算代替, 使得运行过程中只需要记录状态的顶层引用即可.如图所示, 状态`state1`修改`child 2` 后来到 `child 2'`, 根据不可变库的实现, 该属性向父级递归的更新其引用, 使得`parent1` 与 `state1`引用地址获得了更新, 而`child1`, `child3` 以及`parent2`及其子节点保持原有引用地址不变, 实现了未修改处的复用.

![immutable](/post/immer/immutable.png)
图中展示了一种严格`immutable`的数据更新过程. 
> `react`推荐`props`和`state`是`immutable`的, 不过在实际开发中, 经常能见到, 只要将传给`react state`的那份数据顶层引用地址更新就能成功设置状态的场景, 而嵌套复杂数据对象内部地址引用没有更新, 会有潜在的危险.


`immutable`利用数据地址来作为判断数据变更与否的依据, 弥补了得传统`===`操作符不能比较复杂数据对象的问题, 通过引入`immutable`的概念, 快照的模式也可以很大程度上视作为只关注修改的属性的差分模式, 而单条记录也只需要保存顶层的根节点地址`state`, 状态的回溯也只需要在状态仓库的顶层进行相应筛选(过滤无需记录的状态)与提交. **在纯前端的应用环境下, 遵循了`immutable`的快照撤销既能最大程度复用数据内存, 也能在任意历史数据快照之间快速切换**

但是也有弊端, 由于数据引用地址是编程语言的执行环境提供的, 所以在跨环境, 或者执行环境不同的场景下, 没有办法引入数据地址作为额外标识来实现`immutable`, 快照将会退化到深拷贝模式.


### 差分补丁
差分补丁主要关注于每两次之间的状态差分, 需要在状态转移时, 生成相应的正向与反向补丁, 这两种补丁保证了两个状态之间的来回切换而不需要保存完整的状态信息. 

- 差分补丁的结构设计可以参照[`JsonPatch`](https://jsonpatch.com/), 通过类似`op`的字段来标识增删改的操作

- 一种更暴力的方式就是对每个操作入口进行收敛, 人为的去提取信息并作记录编写对应的正逆向补丁,对于动态程度比较高（例如传key value的反射取值）写出来的代码就会比较难看了

![patch](/post/immer/patch.png)

差分更新的单条记录的量不会过大, 且补丁独立于状态库单独存储, 但仍然受到修改程度影响, 所以当一次性修改很多属性的时候, 补丁的大小也会等量的增长, 所以在性能方面仍然不及使用不可变库模式下的快照. 且如果在`timetravel`时(从`state2` 重做到 `state4`),差分补丁需要走完（`state2` -> `state3` -> `state4`,反之亦然）整个应用补丁的链路。如果跨度较大计算复杂度也会随之上升。

其优势主要体现在与存储的完整数据长度上, 快照模式下的硬伤就是完整数据过大问题, 例如 __对涉及到跨环境通信的撤销(例如历史记录云同步, 跨标签页同步记录)__ 时, 快照的模式决定了它将回到深拷贝方式, 当进行通信的时候, 没有了js环境, 所有的数据回归到字符串, 实现方式只能回归到深拷贝的`JSON.stringify`, 原来的内存占用过大问题转化为请求体过大问题. 而差分补丁的方式则是最大程度上减少请求体里所携带的信息.<br />

---
### 浅读一个基于快照的撤销与重做

因为快照的存储方式比较常见,社区里也有不少基于快照的解决方案,不需要重复造轮子了,看一下`zustand`官方推荐的`zundo`中间件是怎么实现撤销回溯的

![zundo](/post/immer/zundo.png)

> 简单描述下`zustand`,可以理解成一个不需要写`actionType`的版本的`hooks`风格的`redux`
```tsx
import { create } from 'zustand'

const useBearStore = create((set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
}))

const FC = () => {
  return (
    <div onClick={useBearStore((state) => state.increasePopulation)}>
      {useBearStore((state) => state.bears)}
    </div>
  )
}
```

#### 从入口文件开始
首先实现了`zustand`中间件的注册方法,拦截了使用了该中间件的`store`（__下文统称外部实例__）的`set`和`setState`方法,使其额外执行一系列记录外部实例状态变更的操作, 这里就可以看到,`zundo`通过调用传入的`store`的`get`方法,拿到了外部实例的全部/部分（通过传入回调实现）的快照
```ts
    /**
     * @note 修改了注释便于阅读
     * @see https://github.com/charkour/zundo/blob/84e27b84e6e1fb343754480b28af450a1de67a9f/src/index.ts#L58
     * */
    const setState = store.setState; // 缓存原来的`setState`,因为也会有其他中间件修改store的setState方法
    store.setState = (...args) => {
      // 这里拿到更新前的仓库状态
      // zundo的配置支持传入预处理方法只对部分状态进行追踪,否则默认追踪当前仓库的所有状态
      const pastState = options?.partialize?.(get()) || get();
      // 执行仓库更新
      setState(...args);
      // 记录状态的主入口,注意这里的调用时机必须在状态更新后
      curriedHandleSet(pastState);
    };

     return config(
      //和上面的setState方法是一样的
      (...args) => {
        const pastState = options?.partialize?.(get()) || get();
        set(...args);
        curriedHandleSet(pastState);
      },
      get,
      store,
    );

```
#### 中间件内部状态

中间件`zundo`内部也是一个单独的`zustand`状态仓库,调用了`zustand`的`createStore`创建了一份实例（__下文统称内部实例__）,用来保存状态快照,以及对应的操作状态的方法, 并且以`temporal`的字段将内部实例暴露给外部实例。

```ts
store.temporal = createStore(
  options?.wrapTemporal?.(temporalStateCreator(set, get, options)) ||
    temporalStateCreator(set, get, options),
);
```

中间件初始化时会传递外部实例的`set`/`get` 给到内部实例的方法,形成闭包调用,这样内部实例在执行撤销重做时就能反过来调用外部实例的`setState`来操作外部状态

**所以这里表面说是状态回溯,实则是把一个以前的快照以一个新的`setState`的形式提交更新,对于状态仓库来说并没有真正的回到过去,而是来到了一个新的状态,只不过这个新的状态恰巧和前面某一次的状态一模一样**

内部实例对外暴露的状态定义, 以`_`为开头定义的属性/方法标识该成员仅中间件内部逻辑使用,不应该暴露给上层开发者：
```ts
/**
 * @note 调整了顺序
 * @see https://github.com/charkour/zundo/blob/84e27b84e6e1fb343754480b28af450a1de67a9f/src/temporal.ts#L14
 * */
return {
  // 可以通过中间件设置配置初始撤销/回溯的快照数据,或许可以用来做多应用同步,或者持久化存储的回填？
  pastStates: options?.pastStates || [],
  futureStates: options?.futureStates || [],
  // 相当于enable, 一个开关状态用于控制是否记录快照, 通过`pause`和`resume`来切换
  isTracking: true,
  pause: () => set({ isTracking: false }),
  resume: () => set({ isTracking: true }),
  // 对外暴露的撤销与重做方法,主要配合`pastStates`和`futureStates`实现了时间回溯
  undo: (step = 1) => {/**/},
  redo: (step = 1) => {/**/},
  // 重置内部仓库状态
  clear: () => set({ pastStates: [], futureStates: [] }),
  // 设置可选的持久化相关的接口,默认不会启用
  setOnSave: (_onSave) => set({ _onSave }),
  // 默认初始化从配置中读取save方法, 也可以通过 `setOnSave` 异步地更新
  _onSave: options?.onSave, 
  //...
}

```
在入口文件注册中间件时的 `curriedHandleSet` 就指向的`zundo`内部实例的 `_handleSet`方法,在拦截过后的`set`方法里最后一个调用, 如此可通过使用`userGet`就能获取到最新的外部实例状态

```ts
export const temporalStateCreator = <TState>(
  userSet: StoreApi<TState>['setState'],
  userGet: StoreApi<TState>['getState'],
  options?: ZundoOptions<TState>,
) => {
  const stateCreator: StateCreator<_TemporalState<TState>, [], []> = (set, get) => {
    return {
      //...
      _handleSet: (pastState) => {
        if (get().isTracking) {
          const currentState = options?.partialize?.(userGet()) || userGet();
          // 这里支持开发者自己处理前后状态差异,开发者可以用来自己实现差分方法,但是
          // 最终还是需要输出位外部实例的状态数据结构（像快照一样）虽然名字叫`diff`,
          // 但是并不是开启差分补丁模式, 可以理解成一个增强版的预处理方法
          const deltaState = options?.diff?.(pastState, currentState);
          // 支持自定义相等运算方法
          if (!(options?.equality?.(pastState, currentState) || deltaState === null)) {
            // 支持设置历史栈`pastStates`最大容量, 溢出后将丢弃最旧的数据,默认容量无上限
            if (options?.limit && get().pastStates.length >= options?.limit) {
              get().pastStates.shift();
            }
            // 调用持久化方法（如存在）
            get()._onSave?.(pastState, currentState);
            set({
              pastStates: get().pastStates.concat(deltaState || pastState),
              futureStates: [],
            });
          }
        }
    }
  }
}

```
如果剔除所有配置化的参数逻辑,`_handleSet`的默认行为可以简化为, 当外部应用状态发生了变更, 且在中间件开启了`isTracking`的状态,将外部实例 __更新前__ 的状态`pastState`作为最新的快照数据追加到内部实例的`pastStates`中, 且清空`futureStates`. 此处使用了`concat`而非`push`, 因为`concat`返回了一个新的数组, 简洁的实现了`immutable`.

```ts
return {
  _handleSet :(pastState)=>{
    if(get().isTracking){
       set({
        pastStates: get().pastStates.concat(pastState), 
        futureStates: []
      });
    }
  }
}
```
#### 历史记录栈的实现
`zundo` 使用了2个数组来实现历史记录, 每一个数组内相邻的两份快照都代表了一次外部实例状态转移的前后状态, 所以快照的顺序即代表了操作的顺序,
- 当外部实例发生操作, 状态变化时, 由上文提到的`_handleSet`方法拦截了并推入`pastStates`

![step1](/post/immer/step1.png)

- 当外部实例执行`undo`时, `zundo`会先从外部实例拿一份最新的状态, 将`pastStates`倒序遍历并切割, 遍历长度为撤销的步数, (比如一次性撤销2步) 默认为1步, 这样倒序遍历并切割后的数组的第一个元素就是将要应用到外部实例的快照数据, 然后将切割数组中剩下的元素倒序的推入`futureStates`数组
```ts
  const currentState = options?.partialize?.(userGet()) || userGet();
  // 从尾部开始倒序切割, 获得一个新的倒序子数组
  const statesToApply = get().pastStates.splice(-steps, steps);
  // 倒序子数组的第一份数据就是将要应用的快照
  const nextState = statesToApply.shift()!;
  userSet(nextState);
  set({
    // 由于使用了`splice`, `pastStates`引用未变但是存的数据变了
    pastStates: get().pastStates,
    futureStates: get().futureStates.concat(
      options?.diff?.(currentState, nextState) || currentState,
      statesToApply.reverse(),
    ),
  });

```
> 这块的源码涉及到两次翻转有点绕, 简单来说就是`pastStates`是一个栈, 一次性撤销几步, 就从`pastStates`从后往前数拿第几个数据, 把他设置回外部实例,然后先把当前最新状态保存到`futureStates`, 再把比拿走的那份数据`index`大的所有快照全都 __倒序地__ 塞到`futureStates`的数组尾部, 用来保证`pastStates`中快照的顺序由最久远的快照排列到最临近的快照, 而`futureStates`中的数据顺序由最临近的快照排列到最久远的快照

图例展示了一次撤销两步时, 内部实例的数组变化情况：
![step2](/post/immer/step2.png)

- 当外部实例执行`redo`时, 方法的行为与`undo`相似, 只是把操作的对象数组相互交换了一下, 源码也十分相似
```ts
  const currentState = options?.partialize?.(userGet()) || userGet();
  const statesToApply = get().futureStates.splice(-steps, steps); 
  const nextState = statesToApply.shift()!;
  userSet(nextState);
  set({
    pastStates: get().pastStates.concat(
      options?.diff?.(currentState, nextState) || currentState,
      statesToApply.reverse(),
    ),
    futureStates: get().futureStates,
  });
```


---
## 实现一个基于差分补丁的撤销与重做

撤销与重做设计思想类似命令模式, 开发者事先注册好确定的指令, 赋予这些指令功能的同时还能够同步记录一些其他运行时的元数据, 在用户交互(执行命令)的同时, 将命令推入栈中方便程序作后续处理亦或者在满足某些条件的情况下丢弃.

套用到当前的项目业务场景中去,可以归纳为
- 这些注册好的确定的指令就是提供给用户的, 用户交互发生时直接/间接需要调用到的方法. 
- 推入栈的数据记录了可以重现用户操作的数据.
- 在实现以上两点的前提下, 降低代码侵入性提高动态更新的支持

### 补丁数据

撤销重做的核心就是正向与逆向的补丁数据,在每次修改`state`后,可以通过类似`microdiff`这类的库,通过交换顺序对前后两个状态之间进行计算正向与反向的`patch`, 这样的设计需要侵入并拦截状态管理库的数据的`setter`过程,例如传统的`redux`则就需要在每个`reducer`中显式的实现,就没有那么优雅

好消息是,常用的不可变库`immer`,其实就提供了这个记录补丁数据的能力, 包括正向与逆向, 只是该功能默认不开启。而`RTK`对`redux`的封装正是基于`immer`来实现的

>显式地开启补丁功能[官方文档](https://immerjs.github.io/immer/patches/). 
```javascript
// version 6
import { enablePatches } from 'immer'

enablePatches();
```
开启后即可在`produce`方法中,传入一个回调作为第三个参数, 该回调将会获得`produce`执行过程中对`draft`对象所修改的所有行为的补丁, 以`patches`标注为正向补丁, `inversePatches`标注为反向回溯补丁, 同时`immer`还提供了使用补丁的方法`applyPatches`来应用刚刚输出的正反向补丁


同时`immer`也提供了应用补丁的工具函数`applyPatches`:
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
`reducer`的模式天然契合,于是设想我们是否可以给每个`reducer`额外添加一份回调, 用作撤销回退的入口, 于是设想它应该是这样的
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

在默认的`@redux/toolkit`中, 并没有实现上述功能, 于是需要**修改源码**, 让`immer`在`reducer`执行时记录差分补丁. `immer`作用的区域是`createReducer.ts`这个文件, 首先需要在其引入模块部分开启补丁模式. 而`createReducer.ts`会被`createSlice.ts`文件引用,同时为了保证改造后的类型推断完整, `creatReducer.ts`中的类型涉及到文件也要修改,经排查只有`mapBuilders.ts`, 也会被`createSlice.ts`引用

RTK的源码拆分比较灵活,我们可以通过实现自己的`createSlice` `createReducer`等接口来替换原本的功能,做到最小程度的改动
> 本文基于`@reduxjs/toolkit@1.8.6`版本进行改造,后续官方包更新可能会导致不可用,但几率不大

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
- `immer`产生的`pathes`需要按照顺序进行存储 ,通过维护一个全局栈来实现, 并且需要用一个值来记录当前的回退位置。
- 通常来说,只有在产生新操作记录的时候才会对当前栈进行切分, 舍弃记录位置之后的所有`patches`

