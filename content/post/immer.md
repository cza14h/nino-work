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

撤销与重做是编排类应用涉及到一个常用的需求, 用户操作的入口杂, 动态性较高, 所以撤销重做的界定是一个比较关键的点. 在普通撤销重做的基础上, 加入可视化的历史记录面板以增强界面提供的信息, 方便用户的操作.

由于应用是使用`react`作为视图框架, 所以本文都会围绕着`react`的生态来实现相关的功能

---

### 前置知识

这个需求按功能模块可以划分为, 数据可逆模块 + 历史列表展现模块。  通常情况下实现数据可逆有两种方式: **快照与差分**.

#### 快照

快照的思想比较容易理解, 直接将执行后的完整状态作为数据推入列表, 由于每一条记录都包含了应用上所有组件完整的状态, 列表中相邻两条记录可以是相互独立的没有耦合, 所以状态之间的切换可以是跳跃的 (可以直接从状态2回到状态4而不考虑中间有多少个状态), 回溯状态的计算量不会随着记录跳跃距离而增长.
  ![快照](/post/immer/snapshot.png)
  <!-- <img src="/post/immer/snapshot.png" /> -->

- 一种比较容易理解的快照机制为深拷贝,常见实现方法为`JSON.parse(JSON.stringify())`,`lodash.cloneDeep()`等. 从内存角度来看, 每个状态都是独立一块内存区域, 哪怕是两个相邻记录之间只有很小的改动, 在此方法下的快照都会完整的将状态深拷贝一份作为备份, 这个方法的缺点也是深拷贝带来的内存空间的浪费, 如果条历史记录记录的状态特别多, 亦或者是历史记录列表特别长的时候, 内存问题在本文中提到的所有记录数据方法中将是最差的.

- 在深拷贝之上的改进方法则是借助不可变库(`immutablejs`, `immerjs`)来解决上者带来的内存问题, 当不可变性介入时, 传统的深拷贝操作将被不可变计算代替, 使得运行过程中只需要记录状态的顶层引用即可.如图所示, 状态`state1`修改`child 2` 后来到 `child 2'`, 根据不可变库的实现, 该属性向父级递归的更新其引用, 使得`parent1` 与 `state1`引用地址获得了更新, 而`child1`, `child3` 以及`parent2`及其子节点保持原有引用地址不变, 实现了未修改处的复用.

![immutable](/post/immer/immutable.png)
图中展示了一种严格`immutable`的数据更新过程.
> `react`推荐`props`和`state`是`immutable`的, 不过在实际开发中, 经常能见到, 只要将传给`react state`的那份数据顶层引用地址更新就能成功设置状态的场景, 而嵌套复杂数据对象内部地址引用没有更新, 会有潜在的危险. [翻车demo](https://stackblitz.com/edit/stackblitz-starters-5a1htw?)

`immutable`利用数据地址来作为判断数据变更与否的依据, 弥补了得传统`===`操作符不能比较复杂数据对象的问题, 通过引入`immutable`的概念, 快照的模式也可以很大程度上视作为只关注修改的属性的差分模式, 而单条记录也只需要保存顶层的根节点地址`state`, 状态的回溯也只需要在状态仓库的顶层进行相应筛选(过滤无需记录的状态)与提交. **在纯前端的应用环境下, 遵循了`immutable`的快照撤销既能最大程度复用数据内存, 也能在任意历史数据快照之间快速切换**

但是也有弊端, 由于数据引用地址是编程语言的执行环境提供的, 所以在跨环境, 或者执行环境不同的场景下, 没有办法引入数据地址作为额外标识来实现`immutable`, 快照将会退化到深拷贝模式.

#### 差分补丁

差分补丁主要关注于每两次之间的状态差分, 需要在状态转移时, 生成相应的正向与反向补丁, 这两种补丁保证了两个状态之间的来回切换而不需要保存完整的状态信息.

- 差分补丁的结构设计可以参照[`JsonPatch`](https://jsonpatch.com/), 通过类似`op`的字段来标识增删改的操作

- 一种更暴力的方式就是对每个操作入口进行收敛, 人为的去提取信息并作记录编写对应的正逆向补丁,对于动态程度比较高（例如传key value的反射取值）写出来的代码就会比较难看了

![patch](/post/immer/patch.png)

差分更新的单条记录的量不会过大, 且补丁独立于状态库单独存储, 但仍然受到修改程度影响, 所以当一次性修改很多属性的时候, 补丁的大小也会等量的增长, 所以在性能方面仍然不及使用不可变库模式下的快照. 且如果在`timetravel`时(从`state2` 重做到 `state4`),差分补丁需要走完（`state2` -> `state3` -> `state4`,反之亦然）整个应用补丁的链路。如果跨度较大计算复杂度也会随之上升。

其优势主要体现在与存储的完整数据长度上, 快照模式下的硬伤就是完整数据过大问题, 例如 **对涉及到跨环境通信的撤销(例如历史记录云同步, 跨标签页同步记录)** 时, 快照的模式决定了它将回到深拷贝方式, 当进行通信的时候, 没有了js环境, 所有的数据回归到字符串, 实现方式只能回归到深拷贝的`JSON.stringify`, 原来的内存占用过大问题转化为请求体过大问题. 而差分补丁的方式则是最大程度上减少请求体里所携带的信息.<br />

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

`zustand`通过传递`store`实例来模拟传统注册中间件的效果. 中间件通过拦截被传递的的`store`实例（**下文统称外部实例**）的`set`和`setState`方法, 使其额外执行一系列记录外部实例状态变更的操作. 这里就可以看到,`zundo`通过调用传入的`store`的`get`方法,拿到了外部实例的全部/部分（通过传入回调实现）的快照

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

中间件`zundo`内部也是一个单独的`zustand`状态仓库,调用了`zustand`的`createStore`创建了一份实例（**下文统称内部实例**）,用来保存状态快照,以及对应的操作状态的方法, 并且以`temporal`的字段将内部实例暴露给外部实例。

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
            // 支持设置历史列表`pastStates`最大容量, 溢出后将丢弃最旧的数据,默认容量无上限
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

如果剔除所有配置化的参数逻辑, `_handleSet`的默认行为可以简化为, 当外部应用状态发生了变更, 且在中间件开启了`isTracking`的状态,将外部实例 **更新前** 的状态`pastState`作为最新的快照数据追加到内部实例的`pastStates`中. 此处使用了`concat`而非`push`, 因为`concat`返回了一个新的数组, 简洁的实现了`immutable`.

```ts
return {
  _handleSet :(pastState)=>{
    if (get().isTracking) {
       set({
        pastStates: get().pastStates.concat(pastState), 
        futureStates: [] //外部状态变更将清空futureStates
      });
    }
  }
}
```

#### 历史记录列表

`zundo` 使用了2个数组来实现历史记录, 每一个数组内相邻的两份快照都代表了一次外部实例状态转移的前后状态, 所以快照的顺序即代表了操作的顺序。
**`pastStates`数组代表了可以撤销的快照数量, `futureStates`数组代表了可以重做的快照数量.**

- 当外部实例发生操作, 状态变化时, 由上文提到的`_handleSet`方法拦截了状态并推入`pastStates`

![step1](/post/immer/step1.png)

- 此外,应用本身处于撤销若干步的状态(即`futureStates`中有值), 外部实例发生的状态变更会让历史记录列表丢弃`futureStates`中存放的重做的数据, 形成一条新的快照状态转移链路.

![step1_](/post/immer/step1_.png)

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

> 这块的源码涉及到两次数组翻转有点绕, 简单来说就是`pastStates`是一个数组, 一次性撤销几步, 就从`pastStates`从后往前数拿第几个数据, 把他设置回外部实例,然后先把当前最新状态保存到`futureStates`, 再把比拿走的那份数据`index`大的所有快照全都 **倒序地** 塞到`futureStates`的数组尾部, 用来保证`pastStates`中快照的顺序由最久远的快照排列到最临近的快照, 而`futureStates`中的数据顺序由最临近的快照排列到最久远的快照

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

#### 小结

`zundo`实现了一个基于`immutable`规范的快照撤销与重做能力, 通过拦截`zustand`的`set`来捕获快照, 同时维护在内部的状态仓库里, 当执行撤销回退时, `zundo`分别从不同的状态数组内拿到对应的快照并`set`回使用该中间件的`store`已达成时间回溯的效果. 快照的思想更加关注于状态的结果, 对状态转移如何产生的并不关心.

---

### 实现一个基于差分补丁的撤销与重做

正如前置知识章节的示意图, 使用补丁数据进行撤销与重做, 要关注补丁对生成的过程而非追踪应用的状态, 设计思想类似命令模式, 将触发状态转移的源头方法进行收敛, 统一进行差分计算, 同时能够同步记录一些其他运行时的元数据, 在应用交互(执行状态转移命令)时, 将这些元数据与补丁保存到历史记录集合中用作展示以及撤销重做的核心功能实现

套用到编排类应用的业务场景中去,可以归纳为

- 这些注册好的确定的指令就是提供给用户的, 用户交互发生时直接/间接需要调用到的方法.
- 推入列表的数据记录了可以重现用户操作的数据.
- 在实现以上两点的前提下, 降低代码侵入性提高动态更新的支持

**这个版本的撤销与重做的实现是基于 `@reduxjs/toolkit`(下文会简写为`RTK`) 这个`redux`官方封装的功能增强版轮子来实现的.** 至于为什么, 首先看一下官方基于`slice`概念的[示例](https://redux-toolkit.js.org/api/createSlice#reducers)

``` javascript
import { createSlice } from '@reduxjs/toolkit'

const counterSlice = createSlice({
  name: 'counter',
  initialState: 0,
  reducers: {
    increment: (state) => state + 1,
  },
})
```

经过`RTK`封装后的`redux`十分简洁, 看上去非常像上文提到的`zustand`的`store`定义方式, 但是`RTK`显式的区分了`state`和`reducers`的定义, 并且`reducers`中单个`reducer`支持对象式的定义, 这为我们的扩展提供了切入点.

#### 补丁数据

撤销重做的核心就是正向与逆向的补丁数据,在每次修改`state`后,可以通过类似`microdiff`这类的库,通过交换顺序对前后两个状态之间进行计算正向与反向的`patch`, 这样的设计需要侵入并拦截状态管理库的数据的`setter`过程,例如传统的`redux`则就需要在每个`reducer`中显式的实现, 就没有那么优雅.

好消息是,常用的不可变库`immer`, 其实就提供了这个记录补丁数据的能力, 包括正向与逆向, 只是该功能默认不开启。而`RTK`对`redux`的封装正是基于`immer`来实现的

>显式地开启补丁功能[官方文档](https://immerjs.github.io/immer/patches/).

```javascript
// version 6
import { enablePatches } from 'immer'

enablePatches();
```

开启后即可在`produce`方法中, 传入一个回调作为第三个参数, 该回调将会获得`produce`执行过程中对`draft`对象所修改的所有行为的补丁, 以`patches`标注为正向补丁, `inversePatches`标注为反向回溯补丁, 同时`immer`还提供了使用补丁的方法`applyPatches`来应用刚刚输出的正反向补丁

>注: 新版的`immer`可以通过使用`produceWithPatches`这个方法来生成补丁, 更加的简洁, 同样也需要显示的开启`Patch`插件. 但是由于开发这个需求的时候还没有这个功能所以将会以回调形式进行改造.

```javascript
import produce, { applyPatches, produceWithPatches } from 'immer';

const state = {
  name: "Micheal",
  age: 32
}
const changes = []
const inverseChanges = []

const nextState = produce(
  state,
  draft => {
    draft.age = 33
  },
  // 这里可以接受一个回调用来接收补丁, 用闭包函数来记录补丁
  (patches, inversePatches) => {
    changes.push(...patches)
    inverseChanges.push(...inversePatches)
  }
)
// 新版的元组写法, 更函数式一点
const [nextStateFromNewApi, patches, inversePatches] = produceWithPatches(
  state, 
  draft => {
    draft.age = 33
  },
)

// 应用补丁
const lastState = applyPatches(state, inverseChanges)
expect(lastState).toEqual({
  name: "Micheal", 
  age: 32 
})
```

于是我们撤销重做的改造就会围绕着这两份`patch`数据进行

<!-- 至此`immer`已经帮我们完成了最难的一部分, 而`@reduxjs/toolkit`中就使用了`immer`作为依赖库, 现在要做的就是把每次`produce` (`createSlice`中实现`reducer`的核心)生成的`patches`与`inversePatches`成对的保存在一条记录里作为历史. -->

#### 设计命令注册方法

从现有的项目出发, 用户的操作会带来数据上的改变, 但不是所有的操作都需要历史记录, 就像`PhotoShop`那样, 有时候打开了画板, 是不需要记录到历史的, 而那些编辑了/操作了图层的动作才是需要记录的, 所以这里就需要将记录`patch`的控制权开放给上层应用, 这样才能够有选择的记录补丁.

`redux`的核心是一个个`reducer`, 我们在定义这些`reducer`的过程中其实也是在向`redux`注册一个又一个的命令, 我们可以在复用这一块的注册行为的基础上去扩展以最终达到注册历史记录命令的目的. 对于单个`reducer`, `RTK`支持这样的写法:

```js
reducers: {
  increment: {
    reducer(state, action) {
      const { payload } = action
      return Math.min(state + payload.step, payload.upperLimit)
    },
    prepare(step = 1, upperLimit = Infinity) {
      return { payload: {step, upperLimit } }
    }
  }
},
/**
 * 这样通过接受一个函数`Object`来达到对`reducer`入参更细致的控制
 * 就可以写成`dispatch(store.actions.increment())`的形式
 * 而非     `dispatch(store.actions.increment({step:1, upperLimit:Infinity }))`
 * /

```

`reducer`的模式天然契合,于是设想我们是否可以给每个`reducer`额外添加一份回调, 用于转发`produce`的第三个回调, 这样能用作记录补丁的入口, 同时它也应该是一个可选的回调, 当不传入方法时, 它输出的数据便不会被记录, 这样也满足了上文提到的有选择性的记录补丁的需求.

于是设想它应该是这样的

```diff
{
  reducer(state, action) {
    const { payload } = action
    return Math.min(state + payload.step, payload.upperLimit)
  },
  prepare(step = 1, upperLimit = Infinity) {
    return { payload: {step, upperLimit } }
  }
+ patch:(patches, inversePatches, action) => {
+   // 记录patches 和 action
+ }
},
```

#### 修改源码

在默认的`RTK`中, 并没有实现上述功能, 于是需要**修改源码**, 让`immer`在`reducer`执行时记录差分补丁. `immer`作用的区域是`createReducer.ts`这个文件, 首先需要在其引入模块部分开启补丁模式. 而`createReducer.ts`会被`createSlice.ts`文件引用,同时为了保证改造后的类型推断完整, `creatReducer.ts`中的类型涉及到文件也要修改,经排查只有`mapBuilders.ts`, 也会被`createSlice.ts`引用

`RTK`的源码拆分比较灵活,我们可以通过实现自己的`createSlice`, `createReducer`等接口来替换原本的功能,做到最小程度的改动
> 本文基于`@reduxjs/toolkit@1.8.6`版本进行改造, 后续官方包的破坏性更新可能会导致不可用, 但源码的改造思路应该不会受到影响.

1. 首先开启`immer`相关的api, 把需要定义的类型也导入

    ```diff
    /**
    * @see https://github.com/reduxjs/redux-toolkit/blob/f7689133ee56e7787447e6089907fa6d1639b771/packages/toolkit/src/createReducer.ts#L1
    * */
    - import type { Draft } from 'immer'
    - import createNextState, { isDraft, isDraftable, enableES5 } from 'immer'
    + import type { Draft, Patch } from 'immer'
    + import createNextState, { isDraft, isDraftable, enableES5, enablePatches } from 'immer'
      import type { AnyAction, Action, Reducer } from 'redux'
    ...
    + enablePatches();

    ```

2. 然后修改对应的配置类型标注让`ts`能正确推导, 原有的`reducers`配置定义存放在`createSlice.ts#L227`的`ValidateSliceCaseReducers`类型,

    ```ts
    /**
     * @see https://github.com/reduxjs/redux-toolkit/blob/f7689133ee56e7787447e6089907fa6d1639b771/packages/toolkit/src/createSlice.ts#L227 
     * */
    export type ValidateSliceCaseReducers<S, ACR extends SliceCaseReducers<S>> = ACR & {
      [T in keyof ACR]: ACR[T] extends {
        reducer(s: S, action?: infer A): any
      }
        ? {
            prepare(...a: never[]): Omit<A, 'type'>
          }
        : {}
    }
    ```

    可见只有`reducer`和`prepare`两个定义, 在这里加上我们想要的`patch`方法定义, 用`|`连接, 代表可选. 并且额外接收一个`action`参数, 可以用来接收元数据

    ```diff
    export declare type ValidateSliceCaseReducers<S, ACR extends SliceCaseReducers<S>> = ACR & {
      [T in keyof ACR]: ACR[T] extends {
        reducer(s: S, action?: infer A): any;
      }
    -   ? {
    -        prepare(...a: never[]): Omit<A, 'type'>
    -      }
    +    ?
    +        | {
    +            prepare(...a: never[]): Omit<A, 'type'>;
    +          }
    +        | {
    +            patch(data: Patch[], inverse: Patch[], action?: A): void;
    +          }
    +    : {};
    };

    ```

3. 在`createSlice`解析配置时, 也要像`caseReducer`一样, 将所有的`patch`收集起来, 为了命名一致, 也叫它`casePatcher`. `CasePatcher`类型就是刚刚定义的`patch`方法类型, 并且在接下来解析`slice`中的`reducer`和`prepare`中额外添加记录`patch`的能力. 当然`Record<string,CasePatcher>`中的`key`也保持一致, 同为 ``${slice}/${actionKey}``

    ```diff
    /**
     * @see https://github.com/reduxjs/redux-toolkit/blob/f7689133ee56e7787447e6089907fa6d1639b771/packages/toolkit/src/createSlice.ts#L295
     * */
    const sliceCaseReducersByName: Record<string, CaseReducer> = {}
    const sliceCaseReducersByType: Record<string, CaseReducer> = {}
    + const sliceCasePatchersByType: Record<string, CasePatcher> = {};

    if ('reducer' in maybeReducerWithPrepare) {
      caseReducer = maybeReducerWithPrepare.reducer
    - prepareCallback = maybeReducerWithPrepare.prepare
    + if ('prepare' in maybeReducerWithPrepare) {
    +   prepareCallback = maybeReducerWithPrepare.prepare;
    + }
    + if ('patch' in maybeReducerWithPrepare) {
    +   sliceCasePatchersByType[type] = maybeReducerWithPrepare.patch as CasePatcher<any>;
    + }
    } else {
    ```

4. 在改造`createSlice.ts`支持我们设计的额外回调入口后, 需要改造`createRedcuer.ts`让我们传入的`patch`在运行时真正生效.

    > 通过调用同名函数`createReducer`将`createSlice`中收集到的`caseReducers`的集合(`sliceCaseReducersByType`)封入一个闭包函数, 并在运行时动态地过滤出`action`对应的`caseReducer`并执行(来模拟传统`redux`中`reducer`的`switch-case`). 这里是运行时调用的入口, 也是`RTK`借用`immer`能力的地方, 所以除了上文提到的需要显示开启`enablePatches`外, 还要在`createReducer`的方法中需要接收刚刚解析的 `sliceCasePatchersByType`.

    ```diff
    /**
     * @see https://github.com/reduxjs/redux-toolkit/blob/f7689133ee56e7787447e6089907fa6d1639b771/packages/toolkit/src/createReducer.ts#L276
     **/
    } else {
    + const casePatcher =
    +   typeof slicePatchCallbackByType?.[action.type] === 'function'
    +     ? slicePatchCallbackByType[action.type]
    +     : noop;

      return createNextState(previousState, (draft: Draft<S>) => {
        return caseReducer(draft, action)
    -  }
    +  },
    +  (data: Patch[], inverse: Patch[]) => {
    +     casePatcher(data, inverse, action);
    +   },
      )
    }

    ```

5. DONE! 至此已完成了所有的源码改动, 得益于`RTK`优秀的代码功能拆分和`immer`的补丁能力, 让我们用不到40行的修改就完成了对原有功能的扩展, 具体的使用方式:

    ```ts
    import { configureStore } from "@reduxjs/toolkit"
    import { createSlice } from './my-path-to/createSlice'

    const mySliceWithPatch = createSlice({
      name: 'mySliceWithPatch',
      initialState: { counter: 0 },
      reducers: {
        increment: {
          reducer(state, action) {
            state.counter = state.counter + 1
          },
          patch(patch, inversePatch, action) {
            // 补丁记录行为
          }
        }
      }
    })
    // `RTK`的`configureStore`兼容自定义的`createSlice`
    const store = configureStore({reducer: mySliceWithPatch.reducer})
    // dispatch 一个 action
    const increment = () => store.dispatch(mySliceWithPatch.actions.increment())

    ```

#### 历史记录列表

由于之前我们已经解读过`zundo`的历史记录实现的方式, 故原理不在多做赘述, 列出对比`zundo`快照实现的差异

- `immer`产生的`pathes`需要按照顺序进行存储, 并且需要用一个值来记录当前的回退位置。
  - 可以进一步将`patches`封装成一个闭包供`undo`和`redo`调用, 比起直接存`patch`数据, 底层实现逻辑代码会更清晰可读, 不过代价就是封装闭包的方法需要上层应用或者适配器来实现, **建议根据接入视图框架的方式来进行选择**.
- `zundo`没有对当前的状态记录, 也就是进入页面时, 列表为空, 可以默认携带一个类似“初始化页面”的操作记录用于更好的提示
- 步进大于1的`timetravel`的情况, 差分模式需要依次走完完整的链路, 并不能像`zundo`那样使用双数组切分取值+拼接的形式, 所以这里使用单个数组+`currentIndex`的模式来实现

具体的实现逻辑参照下图

![stack](/post/immer/stack.png)

1. 首先我们定义一个`record`的类型用来指代`state`的状态, 存放一些撤销与回退的方法

    ```ts
    type RecordType = {
      name: string
      undo: ()=> void
      redo?: ()=> void //注意这里是optional
      timestamp: Date
    }

    class HistoryRecord {
      redo?: () => void
      timestamp;
      constructor(public name: string, public undo: () => void ) {
        this.timestamp = new Date()
      }
    }
    ```

    从逻辑图中也能看到, 每一个`record`代表一个`state`, 但是每一次操作产生的`apply (inverse) patches`闭包并不会同时作为同一条`Record`的`undo`和`redo`. 只有新的`record`被生成时, 前一个`record`的`redo`才会被赋值, 所以`redo`才会被定义为`optional`. 一个很符合直觉的解释就是: **你不能重做(区别于再做)一个最新的状态, 你只能做撤销.**

    此外, `name`作为元数据输入生成, 而`timestamp`可以在创建`Record`时立即自动记录.

2. 顺着上面的思路, 我们可以推导出历史记录列表的一些能力, 进行历史记录列表的构建
   - 支持生成`record`, 且生成时需要关联上一条记录的`redo` (这也是为什么需要有一个`init state`, 可以避免写一些边界判断)
   - 有一个指针变量用于标识当前的撤销重做位置

    ```ts
    class HistoryPool {
      current = -1 // 因为用于标注真实的`index`, 所以初始为 -1
      records: HistoryRecord[] = []
    }
    ```

    当状态发生转移, 创建一条`record`, 此时如果在撤销若干步的某个状态(当前`record`不是最后一个)时`this.current + 1 !== this.records.length`, 需要舍弃后面的所有`record`, 衍生出一条新的状态转移链路. 且赋值目前最后一条`record`的`redo`。

    ```ts
    class HistoryPool {
      // ...
      /**
       * 如本节开篇时阐述的, 这里的`addRecord`也可以设计成直接接收`patches`和`inversePatches`
       * 当然, 这样做需要同步修改 `RecordType`的实现 
       **/
      addRecord(name: string, redo: () => void, undo: () => void) { 
        const nextIndex = this.current + 1
        // 直接使用`immer`保证`records`的`immutable`
        this.records = produce(this.records, (draft) => {
          if (draft.length !== nextIndex) {
            draft.splice(nextIndex);
          }
          draft.push(new HistoryRecord(name, undo));
          const currentNode = draft[this.current];
          if (currentNode) {
            currentNode.redo = redo;
          }
        })
        this.current = nextIndex;
      }
    }
    ```

    要注意这里`addRecord`方法的 `redo` 和 `undo`闭包是由一次状态转移产生的`patches`和`inversePatches`构建而来的. 一个补全列表尾部`record`的`redo`, 一个用于直接构建新的`record`.

    然后就可以完成历史记录列表的撤销与重做入口, 通过移动`current`来实现.

    ```ts
    class HistoryPool {
      //...
      redo() {
        if (this.current + 1 === this.records.length) return // 最后一条记录不能`redo`
        this.records[this.current].redo?.()
        ++this.current
      }
      undo() {
        if (this.current === 0 ) return // 第一条记录不能`undo`
        this.records[this.current].undo()
        --this.current
      }
      timeTravel(to:number){
        if (to === this.current) return;
        const cb = to > this.current ? this.redo : this.undo;
        const count = Math.abs(to - this.current);
        for (let i = 0; i < count; i++) {
          cb.call(this);
        }
      }
    }
    ```

3. 最后可以完善一下构造函数, 将一个初始状态推入数组, 这一步也可由持有历史记录列表实例的上层应用来完成.

    ```ts
    class HistoryPool {
      constructor() {
        this.addRecord('初始化', () => {}, () => {})
      }
    }
    ```

#### 接入`react`, 配合`RTK`实现撤销重做

至此我们已经完成了拓展`RTK`的`patch`回调入口以及原生`js`实现的历史记录列表逻辑, 而`RTK`又是由`react`驱动的`store`, 所以历史记录表作为纯`js`实例, 也需要有一个在`react`中的上层实现才能和`RTK`配合使用.

1. 将历史记录列表实例接入`react`
  
    这一部分的功能主要聚焦于`react state`的更新方法对接, `undo`/`redo`等历史记录列表的方法暴露, 以及一些`react`环境下的功能优化

   - 一个常见的方法就是通过`react`组件 + 内部实例的方式进行生命周期与方法的暴露, 这里使用一个`abstract class`来抽象一些存在`store`副作用的方法, 保证组件与`store`解耦, 这样的设计是提升组件的通用性, 甚至可以开源出去, 让其他开发者的应用也能通过实现方法来接入自己的`redux store`.

      ```ts
      abstract class UndoRedo extends React.Component {
        private _history = new HistoryPool()
        abstract getDispatch(actionType: string): Dispatch<AnyAction>
      }
      ```

      `createRecord`方法实现了将`patches`封入闭包的过程, 传入的`actionType`是应用补丁`reducer`的`actionType`, 而非触发状态转移的`reducer`的`actionType`

      ```ts
      abstract class UndoRedo extends React.Component {
        //...
        createRecord(name:string, patches: Patch[], inversePatches: Patch[], actionType: string){
          const dispatch = this.getDispatch(actionType)
          const redo = () => {
            dispatch({ action: actionType, payload: patches });
          };
          const undo = () => {
            dispatch({ action: actionType, payload: inversePatches });
          };
          this._history.addRecord(name, redo, undo)
          this.setMemoState() // 下文会实现
        }
      }
      ```

      最后还需要封装一下`timeTravel`, 因为我们选择了封入闭包函数的方式给到底层历史记录列表进行存储, 而底层的`timeTravel`在循环过程中每执行一次`undo`/`redo`时, `dispatch`了`action`, 继而`react-redux`就会对视图进行一次刷新. 如果`timeTravel`的跨度较大, 就会产生许多次不必要的`rerender`.

      想要解决这个问题需要在调用方法的作用域使用`batch`方法将多次更新合并为一次`commit`, 而`batch`是由`react-redux`提供的, 所以应当是在实现接入`react`生命周期的组件被引入, 于是就额外封装一下底层的`timeTravel`.

      ```ts
      abstract class UndoRedo extends React.Component {
        //...
        timeTravel = (to: number) => {
          batch(() => {
            this._history.timeTravel(to)
          })
          this.setMemoState() // 下文会实现
        }
        undo = () => {
          this._history.undo()
          this.setMemoState() // 下文会实现
        }
        redo = () => {
          this._history.redo()
          this.setMemoState() // 下文会实现
        }
      }
      ```

      同时通过`setState`将`records`等数据接入`react`的生命周期, 收敛到修改状态的回调方法为`timeTravel`,`undo`,`redo`和`createRecord`, 在这些方法的最后额外增加一次带有`memo`的`setState`.

      ```ts
      abstract class UndoRedo extends React.Component {
        // 这里需要再 `_history` 下方初始化, 可以直接拿 `_history` 的初始值来初始化 `state`
        state = { records: this._history.records, current: this._history.current }
        // 类似 `lodash.memoize`, 通过创建一个闭包缓存入参, 每次调用时顺序地比较每个参数, 全部相等时返回缓存的计算结果
        // `_history` 的 `records` 已经实现了 `immutable`
        memoState = createMemo((records: HistoryRecord[], current: number) => {
          return { records, current }
        })

        setMemoState = () => {
          const { records, current } = this._history
          const nextState = this.memoState(records, current)
          if (nextState !== this.state) {
            this.setState(nextState)
          }
        }
      }
      ```

      最终将状态通过`Provider`的形式暴露给其他组件, 所以该组件通常作为靠近顶层的父节点.

      ```ts
      abstract class UndoRedo extends React.Component {
        render() {
          return (
            <UndoRedoContext.Provider value={this.state}>
              {this.props.children}
            </UndoRedoContext.Provider>
          )
        }
      }
      ```

      **作为组件持有实例的模式存在一个问题, `RTK`作为全局变量管理, 定义`reducers`的一些列方法(主要是`patch`)需要在整个`react`应用(包含`UndoRedo`组件)实例化前进行的声明, 而记录`patches`的方法又需要指向在`UndoRedo`的组件实例内的方法, 于是一个先有鸡还是先有蛋的问题就产生了, 只能通过组件的静态方法来转发, 代价是这个组件只能作为单例运行**

      ```ts
      abstract class UndoRedo extends React.Component {
        static interface = {}
        constructor(props: any) {
          super(props);
          UndoRedo.interface = {
            redo: this.redo,
            undo: this.undo,
            createRecord: this.createRecord,
            timeTravel: this.timeTravel,
          };
      }
      ```

   - 还有一种方法就是通过`react 18`新增的`useSyncExternalStore`方法将纯`js`对象封装成一个可以返回`state`的`hook`, 对于其他支持`hook`版本的`react`, 可以通过安装`use-sync-external-store`的`npm`依赖来使用该能力.

       > 关于`useSyncExternalStore`, 可以看成是`useState`和`useEffect`的组合, 具体的原理解析可以参考[这篇文章](https://blog.saeloun.com/2021/12/30/react-18-useSyncExternalStore-api/#understanding-usesyncexternalstore-hook)

      使用`useSyncExternalStore`的外部状态仓库对象需要实现`subscribe`方法, 然后接收一个闭包`getSnapshot`用于的返回调用`useSyncExternalStore`这个`hook`的组件需要的状态. 于是需要改造一下历史记录列表, 首先让`store`支持订阅, 我们简单的维护一个`Set`用来存放订阅回调即可.

      ```ts
      class UndoRedoStore extends HistoryPool {
        subscriptions = new Set<() => void>();
        subscribe = (onStoreChange: () => void) => {
          this.subscriptions.add(onStoreChange);
          return () => {
            this.subscriptions.delete(onStoreChange);
          };
        };
      }
      ```

      然后需要实现`getState`用于封装`useSyncExternalStore`中`getSnapshot`, 其实就是描述当前`store`暴露的状态, 和**方法1**中定义的`state`类似, 也是返回一个遵循`immutable`更新过程的数据变量. 相应的原先`setMemoState`的功能为**计算内部`state`**和**通知`react`更新**, 这里由于计算`state`的过程由`getState`实现, `setMemoState`的逻辑也简化为通知`react`更新, 只不过不同于使用`setState`, 这里使用`subscription`中存储的`onStoreChange`来实现.
      > 这里取名为`setMemoState`其实有点不恰当, 但是是为了标识这个方法的调用时机与**方法1**中一致, 故保持命名相同

      ```ts
      class UndoRedoStore extends HistoryPool {
        // ...
        memoState = createMemo((records: HistoryRecord[], current: number) => {
          return { records, current };
        });
        getState = () => {
          return this.memoState(this.records, this.current);
        };
        setMemoState() {
          this.subscriptions.forEach((cb) => {
            cb();
          });
        }
      }
      ```

      **方法1**中用`batch`重写的`timeTravel`和`createRecord`方法也需要在这个方式下实现. 需要注意的是, 这里可以通过构造函数传入`getDispatch`方法来注入, 当然也可以像**方法1**一样使用`abstract class`.
      > **方法1**中当然也可以直接把`getDispatch`作为构造参数传入, 不这么做的原因是, 作者习惯于将`react`的`props`开放给运行时需要响应式的变量, 而一些静态的参数则通过工厂, 类重写的方式, 让应用省去更新追踪响应式的过程(`memo`等).

      ```ts
      class UndoRedoStore extends HistoryPool {
        constructor(public getDispatch:()=>Dispatch){}

        timeTravel = (to: number) => {
          batch(() => {
            super.timeTravel(to);
          });
        };

      }

      ```

    **两种方法相比较, 虽然思路相同, 但更推荐第二种方法, 实现的方法比较优雅.**

2. 将`RTK`回调入参传递给历史记录列表的`react`实现, 且额外注册应用补丁的`reducer`

---

### 总结

---

### 拓展(挖坑)

- [ ] **与`yjs`配合实现多人协同的撤销回退**
