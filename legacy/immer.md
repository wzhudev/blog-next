---
title: Immer 源码浅析
date: 2020/1/20
description: 简要分析 Immer 的源代码
tag: immer, source code, Chinese
author: Wenzhao
---

# Immer 源码浅析

[Immer](https://immerjs.github.io/immer/docs/introduction) 是一个非常好玩的库，我在等飞机的时候读了一下它的源码。这篇文章（笔记？）旨在于通过阅读源码分析 [Immer 的原理](https://medium.com/hackernoon/introducing-immer-immutability-the-easy-way-9d73d8f71cb3)。我仅考虑了最常简单的使用方式，mutate 的对象也只是 plain object 而已，你可以在阅读本文之后再去探究其他 topic。

## produce

最 common 的 produce 执行流程：

1. `ImmerScope.scope`
2. 在 `base` 创建 root proxy，根据 base 的数据类型选择正确的 trap
   1. 对于 Map 和 Set 有对应的 proxy，**对于 plain object 使用 object proxy**，不支持 Proxy fallback 到 ES5 definePropertypl
3. `result = recipe(proxy)`，proxy 的 trap 执行变更逻辑
4. `scope.leave`
5. `processResult`

### scope

1

代表一次 `produce` 调用，也即 `produce` 执行的 context 。

```ts
/** Each scope represents a `produce` call. */
export class ImmerScope {
  static current?: ImmerScope

  patches?: Patch[]
  inversePatches?: Patch[]
  canAutoFreeze: boolean
  drafts: any[]
  parent?: ImmerScope
  patchListener?: PatchListener
  immer: Immer

  constructor(parent: ImmerScope | undefined, immer: Immer) {
    this.drafts = []
    this.parent = parent
    this.immer = immer

    // Whenever the modified draft contains a draft from another scope, we
    // need to prevent auto-freezing so the unowned draft can be finalized.
    this.canAutoFreeze = true
  }

  usePatches(patchListener?: PatchListener) {}

  revoke() {}
  leave() {}
  static enter(immer: Immer) {}
}
```

### createProxy

2 3

创建了一个数据结构 `ProxyState`，里面保存了 `base`，Proxy 的 target 是这个 `state` 而非 `base` 。

```ts
const state: ProxyState = {
  type: isArray ? ProxyType.ProxyArray : (ProxyType.ProxyObject as any),
  // Track which produce call this is associated with.
  scope: parent ? parent.scope : ImmerScope.current!,
  // True for both shallow and deep changes.
  modified: false,
  // Used during finalization.
  finalized: false,
  // Track which properties have been assigned (true) or deleted (false).
  assigned: {},
  // The parent draft state.
  parent,
  // The base state.
  base,
  // The base proxy.
  draft: null as any, // set below
  // Any property proxies. 当前对象的属性的 proxy
  drafts: {},
  // The base copy with any updated values.
  copy: null,
  // Called by the `produce` function.
  revoke: null as any,
  isManual: false
}
```

#### objectTraps

这里主要关心 set get delete 三种会改变属性值的操作。

```ts
const objectTraps: ProxyHandler<ProxyState> = {
  get(state, prop) {
    if (prop === DRAFT_STATE) return state
    let { drafts } = state

    // Check for existing draft in unmodified state.
    // 如果当前对象未被需改（这也意味着属性没有被修改），而且属性已经有了 Proxy
    // 就返回属性的 Proxy，它能够处理之后的操作
    if (!state.modified && has(drafts, prop)) {
      return drafts![prop as any]
    }

    // 否则就获取属性的最新值
    const value = latest(state)[prop]
    // 如果这个 produce 过程已进入后处理阶段，或者属性对应的值不可代理，就直接返回
    if (state.finalized || !isDraftable(value)) {
      return value
    }

    // Check for existing draft in modified state.
    // 如果当前对象已经被修改过
    if (state.modified) {
      // Assigned values are never drafted. This catches any drafts we created, too.
      // 如果最新值不等于初始值，那么就返回这个最新值
      if (value !== peek(state.base, prop)) return value
      // Store drafts on the copy (when one exists).
      // @ts-ignore
      drafts = state.copy
    }

    // 否则为属性创建 Proxy，设置到 drafts 上并返回该 Proxy
    return (drafts![prop as any] = state.scope.immer.createProxy(value, state))
  },
  set(state, prop: string /* strictly not, but helps TS */, value) {
    // 如果当前对象没有被修改过
    if (!state.modified) {
      // 获取初始值，检查值是否发生了变化
      const baseValue = peek(state.base, prop)
      // Optimize based on value's truthiness. Truthy values are guaranteed to
      // never be undefined, so we can avoid the `in` operator. Lastly, truthy
      // values may be drafts, but falsy values are never drafts.
      const isUnchanged = value
        ? is(baseValue, value) || value === state.drafts![prop]
        : is(baseValue, value) && prop in state.base
      if (isUnchanged) return true
      // 没有变化直接返回，有变化执行以下逻辑
      prepareCopy(state) // 如果当前对象没有被拷贝过，制作一层的浅拷贝
      markChanged(state) // 将当前对象标记为脏，要向上递归
    }
    // 标识次属性也赋值过
    state.assigned[prop] = true
    // @ts-ignore
    // 将新值设置到 copy 对象上
    state.copy![prop] = value
    return true
  },
  deleteProperty(state, prop: string) {
    // 这个和 set 差不多，简单
    // The `undefined` check is a fast path for pre-existing keys.
    if (peek(state.base, prop) !== undefined || prop in state.base) {
      state.assigned[prop] = false
      prepareCopy(state)
      markChanged(state)
    } else if (state.assigned[prop]) {
      // if an originally not assigned property was deleted
      delete state.assigned[prop]
    }
    // @ts-ignor
    if (state.copy) delete state.copy[prop]
    return true
  }
}
```

### processResult

```ts
export function processResult(immer: Immer, result: any, scope: ImmerScope) {
  const baseDraft = scope.drafts![0] // 获取根 draft，也就是调用 produce 所生成的 draft
  const isReplaced = result !== undefined && result !== baseDraft
  immer.willFinalize(scope, result, isReplaced)
  if (isReplaced) {
    if (baseDraft[DRAFT_STATE].modified) {
      scope.revoke()
      throw new Error("An immer producer returned a new value *and* modified its draft. Either return a new value *or* modify the draft.") // prettier-ignore
    }
    if (isDraftable(result)) {
      // Finalize the result in case it contains (or is) a subset of the draft.
      result = finalize(immer, result, scope)
      maybeFreeze(immer, result)
    }
    // ... patches 相关逻辑
  } else {
    // Finalize the base draft.
    // 从根 draft 开始整理 result，移除当中的 Proxy
    result = finalize(immer, baseDraft, scope, [])
  }
  // ... patches 相关逻辑
  return result !== NOTHING ? result : undefined
}
```

`finalize` `finalizeProperty` `finalizeTree` 三者递归调用。

## 参考资料

- [Copy-on-right](https://en.wikipedia.org/wiki/Copy-on-write)
