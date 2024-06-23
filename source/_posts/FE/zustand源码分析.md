---
title: Zustand源码分析
date: 2023-06-10 18:06:38
tags: FE，状态管理，Zustand
math: true
categories: FE

---

## 原理

``` javascript
// store.ts
import { create } from "zustand";  
  
const initStateCreateFunc = (set) => ({  
  bears: 0,  
  increase: (by) => set((state) => ({ bears: state.bears + by })),  
});  
  
const useBearStore = create(initStateCreateFunc);

// App.ts
function BearCounter() {  
  const bears = useBearStore((state) => state.bears);  
  return <h1>{bears} around here...</h1>;  
}  
  
function Controls() {  
  const increase = useBearStore((state) => state.increase);  
  return <button onClick={increase}>one up</button>;  
}


```

Zustand的源码比较简单，基于发布订阅：
set的时候触发发布动作，更新所有对应的组件
在使用create函数返回的hook的时候添加订阅者，具体是如何添加的呢，依赖React提供的`useSyncExternalStore`和`useSyncExternalStoreWithSelector`
`useSyncExternalStore是`react18引入的一个新的hooks，用于订阅外部store，被订阅的这个外部store实际上是基于发布订阅模式创建的一个可更新状态的一个类或者一个单独的文件。

首先我们先看官网对`useSyncExternalStore`参数的要求：

第一个参数是subscribe函数，应当提供一个callback，作为该函数的入参，加入订阅，并返回一个取消订阅的函数。
getSnapshot 函数应当从该 store 读取数据的快照。

`useSyncExternalStoreWithSelector`` 相对于 useSyncExternalStore 的优化主要是允许从一个大store中取出组件所用到的部分，同时借助 isEqual来减少 re-render的次数。




store核心代码
``` javascript 

// 首先接受一个创建Store的方法，这个方法由使用者定义。
const createStoreImpl: CreateStoreImpl = (createState) => {
  type TState = ReturnType<typeof createState>
  type Listener = (state: TState, prevState: TState) => void
  let state: TState
  // 创建一个Set结构来维护订阅者。
  const listeners: Set<Listener> = new Set()

  // 定义更新数据的方法，partial参数支持对象和函数，replace指的是全量替换store还是merge
  // 如果是partial对象时，则直接赋值，否则将上一次的数据作为参数执行该方法。
  // 然后利用Object.is进行新老数据的浅比较，如果前后发生了改变，则进行替换
  // 并且遍历订阅者，逐一进行更新。
  const setState: StoreApi<TState>['setState'] = (partial, replace) => {
    const nextState =
      typeof partial === 'function'
        ? (partial as (state: TState) => TState)(state)
        : partial
    if (!Object.is(nextState, state)) {
      const previousState = state
      state =
        replace ?? typeof nextState !== 'object'
          ? (nextState as TState)
          : Object.assign({}, state, nextState)
      listeners.forEach((listener) => listener(state, previousState))
    }
  }

  // getState方法实则是返回当前Store里的最新数据
  const getState: StoreApi<TState>['getState'] = () => state

  // 添加订阅方法，并且返回一个取消订阅的方法。还记得前置知识里所提到的useSyncExternalStore吗？
  const subscribe: StoreApi<TState>['subscribe'] = (listener) => {
    listeners.add(listener)
    // Unsubscribe
    return () => listeners.delete(listener)
  }


  const api = { setState, getState, subscribe }
  // 这里就是官方示例里的set,get,api
  state = createState(setState, getState, api)
  return api as any
}

// 根据传入的createState的类型，手动创建核心store。
export const createStore = ((createState) =>
  createState ? createStoreImpl(createState) : createStoreImpl) as CreateStore


```


useStore部分
``` javascript 

// 对React进行集成
export function useStore<TState, StateSlice>(
  api: WithReact<StoreApi<TState>>,
  selector: (state: TState) => StateSlice = api.getState as any,
  equalityFn?: (a: StateSlice, b: StateSlice) => boolean
) {

  // 利用useSyncExternalStoreWithSelector，对store里的所有数据进行选择性的分片
  const slice = useSyncExternalStoreWithSelector(
    api.subscribe,
    api.getState,
    api.getServerState || api.getState,
    selector,
    equalityFn
  )
  
  useDebugValue(slice)
  return slice
}


```

关于useSyncExternalStore其主要做的就是把对应的组件强制更新的逻辑添加到订阅队列里，如何把强制更新的逻辑添加到队列呢？
可以使用React自带的useState封装成一个函数，大致像下面这样



``` typescript
type commonFunc = () => void
type SubscribeFunc = (callback: commonFunc) => commonFunc
    const useSyncExternalStore = (subscribeValue: SubscribeFunc, getSnapshot: () => number): number => {
      
      const unsubscribeRef = useRef<commonFunc>();
      if (unsubscribeRef.current) {
        unsubscribeRef.current()
      }
      const [number, forceUpdate] = useState(0);
      const handleForceUpdate = useCallback(() => {
        forceUpdate(number + 1)
      }, [number])
        const callback = () => {
            handleForceUpdate();
        }
        unsubscribeRef.current = subscribeValue(callback);
        return getSnapshot();
    }




```


``` javascript 

import { createStore } from './vanilla.ts'

const createImpl = <T>(createState: StateCreator<T, [], []>) => {
  // 创建store
  const api =
    typeof createState === 'function' ? createStore(createState) : createState

  // 将store和react进行集成
  const useBoundStore: any = (selector?: any, equalityFn?: any) =>
    useStore(api, selector, equalityFn)

  Object.assign(useBoundStore, api)

  return useBoundStore;
}


export const create = (<T>(createState: StateCreator<T, [], []> | undefined) =>
  createState ? createImpl(createState) : createImpl) as Create

```

## 总结
梳理一下，我们在A组件里使用了Zustand提供的create的返回值之后，就会通过useSyncExternalStoreWithSelector添加到订阅队列里；
当我们在B组件触发onclick的时候 => 调用zustand提供的set => 遍历订阅者执行React原生的setState更新逻辑 => 对应组件的state更新 => Rerender


## 参考文章
1. https://juejin.cn/post/7261799437730185271?from=search-suggest
2. https://juejin.cn/post/7304594468157014056#heading-5