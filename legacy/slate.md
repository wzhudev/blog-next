---
title: slate 架构分析
date: 2020/9/30
description: 简要介绍 slate 的架构
tag: slate, Chinese, source code
author: Wenzhao
---

# slate 架构分析

slate 是一款流行的富文本编辑器——不，与其说它是一款编辑器，不如说它是一个编辑器框架，在这个框架上，开发者可以通过插件的形式提供丰富的富文本编辑功能。slate 比较知名的用户（包括前用户）有 GitBook 和语雀，具体可以查看[官网的 products 页面](https://docs.slatejs.org/general/resources#products)。

所谓“工欲善其事，必先利其器”，想要在项目中用好 slate，掌握其原理是一种事半功倍的做法。对于开发编辑器的同学来说，slate 的架构和技术选型也有不少值得学习的地方。这篇文章将会从以下几个方面探讨 slate：

- slate 数据模型（model）的设计
- model 变更机制
- model 校验
- 插件系统
- undo/redo 机制
- 渲染机制
- 键盘事件处理
- 选区和光标处理

## slate 架构简介

slate 作为一个编辑器框架，分层设计非常明显。slate 仓库下包含四个 package：

- slate：这一部分是编辑器的核心，定义了数据模型（model），操作模型的方法和编辑器实例本身
- slate-history：以插件的形式提供 undo/redo 能力，本文后面将会介绍 slate 的插件系统设计
- slate-react：以插件的形式提供 DOM 渲染和用户交互能力，包括光标、快捷键等等
- slate-hyperscript：让用户能够使用 JSX 语法来创建 slate 的数据，本文不会介绍这一部分

## slate (model)

先来看 slate package，这一部分是 slate 的核心，定义了编辑器的数据模型、操作这些模型的基本操作、以及创建编辑器实例对象的方法。

### model 结构

slate 以树形结构来表示和存储文档内容，树的节点类型为 `Node`，分为三种子类型：

```ts
export type Node = Editor | Element | Text

export interface Element {
  children: Node[]
  [key: string]: unknown
}

export interface Text {
  text: string
  [key: string]: unknown
}
```

- `Element` 类型含有 `children` 属性，可以作为其他 `Node` 的父节点

- `Editor` 可以看作是一种特殊的 `Element` ，它既是编辑器实例类型，也是文档树的根节点
- `Text` 类型是树的叶子结点，包含文字信息

用户可以自行拓展 `Node` 的属性，例如通过添加 `type` 字段标识 `Node` 的类型（paragraph, ordered list, heading 等等），或者是文本的属性（italic, bold 等等），来描述富文本中的文字和段落。

我们可以通过官方的 richtext demo 来直观地感受一下 slate model 的结构。

_在本地运行 slate，通过 React Dev Tool 找到 `Slate` 标签，参数中的 `editor` 就是编辑器实例，右键选择它，然后点击 store as global variable，就可以在 console 中 inspect 这个对象了。_

<img width="1299" alt="" src="https://user-images.githubusercontent.com/12122021/95169484-87ab7c80-07e5-11eb-93fc-e9588feaa573.png" />

可以看到它的 `children` 属性中有四个 `Element` 并通过 `type` 属性标明了类型，对应编辑器中的四个段落。第一个 paragraph 的 `children` 中有 7 个 `Text`，`Text` 用 `bold` `italic` 这些属性描述它们的文字式样，对应普通、粗体、斜体和行内代码样式的文字。

<img width="596" alt="" src="https://user-images.githubusercontent.com/12122021/95169559-a7db3b80-07e5-11eb-92f8-9eced7691eeb.png" />

那么为什么 slate 要采用树结构来描述文档内容呢？采用树形结构描述 model 有这样一些好处：

- 富文本文档本身就包含层次信息，比如 page,section, paragraph, text 等等，用树进行描述符合开发者的直觉
- 文本和属性信息存在一处，方便同时获取文字和属性信息
- model tree `Node` 和 DOM tree Element 存在映射关系，这样在处理用户操作的时候，能够很快地从 element 映射到 `Node`
- 方便用组件以递归的方式渲染 model

用树形结构当然也有一些问题：

- 对于协同编辑的冲突处理，树的解决方案比线性 model 复杂
- 持久化 model / 创建编辑器的时候需要进行序列化 / 反序列化

### 光标和选区

有了 model，还需要在 model 中定位的方法，即选区（selection），slate 的选区采用的是 `Path` 加 offset 的设计。

`Path` 是一个数字类型的数组 `number[]`，它代表的是一个 `Node` 和它的祖先节点，在各自的上一级祖先节点的 `children` 数组中的 index。

```ts
export type Path = number[]
```

offset 则是对于 Text 类型的节点而言，代表光标在文本串中的 index 位置。

`Path` 加上 offet 即构成了 `Point` 类型，即可表示 model 中的一个位置。

```tsx
export interface Point {
  path: Path
  offset: number
}
```

两个 `Point` 类型即可组合为一个 `Range`，表示选区。

```ts
export interface Range {
  anchor: Point // 选区开始的位置
  focus: Point // 选区结束的位置
}
```

比如我这样选中一段文本（我这里是从后向前选择的）：

<img width="1004"  src="https://user-images.githubusercontent.com/12122021/95169608-bd506580-07e5-11eb-85c7-7ffd190218fe.png" />

通过访问 `editor` 的 `selection` 属性来查看当前的选区位置：

<img width="596" src="https://user-images.githubusercontent.com/12122021/95169644-c93c2780-07e5-11eb-80f1-b2971d896547.png" />

可见，选择的起始位置 `focus` 在第一段的最后一个文字处，且由于第一段中 "bold" 被加粗，所以实际上有 3 个 `Text` 的节点，因此 `anchor` 的 `path` 即为 `[1, 2]`，`offset` 为光标位置在第三个 `Text` 节点中的偏移量 82。

### 如何对 model 进行变更

有了 model 和对 model 中位置的描述，接下来的问题就是如何对 model 进行变更（mutation）了。编辑器实例提供了一系列方法（由 `Editor` interface 所声明），如 `insertNode` `insertText` 等，直接供外部模块变更 model，那么 slate 内部是如何实现这些方法的呢？

_在阅读源代码的过程中，了解到这一点可能会对你有帮助：slate 在最近的一次重构中完全去除了类（class），所有数据结构和工具方法都是由同名的接口和对象来实现的，比如 `Editor`：_

```ts
export interface Editor {
  children: Node[]

  // ...其他一些属性
}

export const Editor = {
  /**
   * Get the ancestor above a location in the document.
   */

  above<T extends Ancestor>(
    editor: Editor,
    options: {
      at?: Location
      match?: NodeMatch<T>
      mode?: 'highest' | 'lowest'
      voids?: boolean
    } = {}
  ): NodeEntry<T> | undefined {
    // ...
    }
  },
}
```

_interface `Editor` 为编辑器实例所需要实现的接口，而对象 `Editor` 则封装了操作 interface `Editor` 的一些方法。所以，在查看 `Editor` 的实例 `editor` 的方法时，要注意方法实际上定义在 create-editor.ts 文件中。这可能是第一次阅读 slate 代码时最容易感到混淆的地方。_

通常来说，对 model 进行的变更应当是**原子化**（atomic）的，这就是说，应当存在一个独立的数据结构去描述对 model 发生的变更，这些描述通常包括变更的类型（type）、路径（path）和内容（payload），例如新增的文字、修改的属性等等。原子化的变更方便做 undo/redo，也方便做协同编辑（当然需要对冲突的变更做转换，其中一种方法就是有名的 operation transform, OT）。

slate 也是这么处理的，它对 model 进行变更的过程主要分为以下两步，第二步又分为四个子步骤：

1. 通过 `Transforms` 提供的一系列方法生成 `Operation`
2. `Operation` 进入 apply 流程
   1. 记录变更脏区
   2. 对 `Operation` 进行 transform
   3. 对 model 正确性进行校验
   4. 触发变更回调

![](https://user-images.githubusercontent.com/12122021/95169701-deb15180-07e5-11eb-8ca5-f72bfb5169f6.png)

首先，通过 `Transforms` 所提供的一系列方法生成 `Operation`，这些方法大致分成四种类型：

```ts
export const Transforms = {
  ...GeneralTransforms,
  ...NodeTransforms,
  ...SelectionTransforms,
  ...TextTransforms
}
```

- `NodeTransforms`：对 `Node` 的操作方法
- `SelectionTransforms`：对选区的操作方法
- `TextTransforms`：对文本操作方法

特殊的是 `GeneralTransforms`，它并不生成 `Operation` 而是对 `Operation` 进行处理，只有它能直接修改 model，其他 transforms 最终都会转换成 `GeneralTransforms` 中的一种。

这些最基本的方法，也即是 `Operation` 类型仅有 9 个：

- `insert_node`：插入一个 Node
- `insert_text`：插入一段文本
- `merge_node`：将两个 Node 组合成一个
- `move_node`：移动 Node
- `remove_node`：移除 Node
- `remove_text`：移除文本
- `set_node`：设置 Node 属性
- `set_selection`：设置选区位置
- `split_node`：拆分 Node

我们以 `Transforms.insertText` 为例（略过一些对光标位置的处理）：

```ts
export const TextTransforms = {
  insertText(
    editor: Editor,
    text: string,
    options: {
      at?: Location
      voids?: boolean
    } = {}
  ) {
    Editor.withoutNormalizing(editor, () => {
      // 对选区和 voids 类型的处理

      const { path, offset } = at
      editor.apply({ type: 'insert_text', path, offset, text })
    })
  }
}
```

可见 `Transforms` 的最后生成了一个 `type` 为 `insert_text` 的 `Operation` 并调用 `Editor` 实例的 `apply` 方法。

`apply` 内容如下：

```ts
apply: (op: Operation) => {
  // 转换坐标
  for (const ref of Editor.pathRefs(editor)) {
    PathRef.transform(ref, op)
  }

  for (const ref of Editor.pointRefs(editor)) {
    PointRef.transform(ref, op)
  }

  for (const ref of Editor.rangeRefs(editor)) {
    RangeRef.transform(ref, op)
  }

  // 执行变更
  Transforms.transform(editor, op)

  // 记录 operation
  editor.operations.push(op)

  // 进行校验
  Editor.normalize(editor)

  // Clear any formats applied to the cursor if the selection changes.
  if (op.type === 'set_selection') {
    editor.marks = null
  }

  if (!FLUSHING.get(editor)) {
    // 标示需要清空 operations
    FLUSHING.set(editor, true)

    Promise.resolve().then(() => {
      // 清空完毕
      FLUSHING.set(editor, false)
      // 通知变更
      editor.onChange()
      // 移除 operations
      editor.operations = []
    })
  }
},
```

其中 `Transforms.transform(editor, op)` 就是在调用 `GeneralTransforms` 处理 `Operation`。`transform` 方法的主体是一个 case 语句，根据 `Operatoin` 的 `type` 分别应用不同的处理，例如对于 `insertText`，其逻辑为：

```ts
const { path, offset, text } = op
const node = Node.leaf(editor, path)
const before = node.text.slice(0, offset)
const after = node.text.slice(offset)
node.text = before + text + after

if (selection) {
  for (const [point, key] of Range.points(selection)) {
    selection[key] = Point.transform(point, op)!
  }
}

break
```

可以看到，这里的代码会直接操作 model，即修改 `editor.children` 和 `editor.selection` 属性。

> slate 使用了 immer 来应用 immutable data，即 `createDraft` `finishDrag` 成对的调用。使用 immer 可以将创建数据的开销减少到最低，同时又能使用 JavaScript 原生的 API 和赋值语法。

### model 校验

对 model 进行变更之后还需要对 model 的合法性进行校验，避免内容出错。校验的机制有两个重点，一是对脏区域的管理，一个是 `withoutNormalizing` 机制。

许多 transform 在执行前都需要先调用 `withoutNormalizing` 方法判断是否需要进行合法性校验：

```ts
export const Editor = {
  // ...

  withoutNormalizing(editor: Editor, fn: () => void): void {
    const value = Editor.isNormalizing(editor)
    NORMALIZING.set(editor, false)
    fn()
    NORMALIZING.set(editor, value)
    Editor.normalize(editor)
  }
}
```

可以看到这段代码通过栈帧（stack frame）保存了是否需要合法性校验的状态，保证 transform 运行前后是否需要合法性校验的状态是一致的。transform 可能调用别的 transform，不做这样的处理很容易导致冗余的合法性校验。

合法性校验的入口是 `normalize` 方法，它创建一个循环，从 model 树的叶节点自底向上地不断获取脏路径并调用 `nomalizeNode` 检验路径所对应的节点是否合法。

```ts
while (getDirtyPaths(editor).length !== 0) {
  // 对校验次数做限制的 hack

  const path = getDirtyPaths(editor).pop()!
  const entry = Editor.node(editor, path)
  editor.normalizeNode(entry)
  m++
}
```

让我们先来看看脏路径是如何生成的（省略了不相关的部分），这一步发生在 `Transforms.transform(editor, op)` 之前：

```ts
apply: (op: Operation) => {
  // 脏区记录
  const set = new Set()
  const dirtyPaths: Path[] = []

  const add = (path: Path | null) => {
    if (path) {
      const key = path.join(',')

      if (!set.has(key)) {
        set.add(key)
        dirtyPaths.push(path)
      }
    }
  }

  const oldDirtyPaths = DIRTY_PATHS.get(editor) || []
  const newDirtyPaths = getDirtyPaths(op)

  for (const path of oldDirtyPaths) {
    const newPath = Path.transform(path, op)
    add(newPath)
  }

  for (const path of newDirtyPaths) {
    add(path)
  }

  DIRTY_PATHS.set(editor, dirtyPaths)
},
```

`dirtyPaths` 一共有以下两种生成机制：

- 一部分是在 operation apply 之前的 `oldDirtypath`，这一部分根据 operation 的类型做路径转换处理
- 另一部分是 operation 自己创建的，由 `getDirthPaths` 方法获取

`normalizeNode` 方法会对 `Node` 进行合法性校验，slate 默认有以下校验规则：

- 文本节点不校验，直接返回，默认是正确的
- 空的 `Elmenet` 节点，需要给它插入一个 `voids` 类型节点
- 接下来对非空的 `Element` 节点进行校验
  - 首先判断当前节点是否允许包含行内节点，比如图片就是一种行内节点
  - 接下来对子节点进行处理
    - 如果当前允许行内节点而子节点非文本或行内节点（或当前不允许行内节点而子节点是文字或行内节点），则删除该子节点
    - 确保行内节点的左右都有文本节点，没有则插入一个空文本节点
    - 确保相邻且有相同属性的文字节点合并
    - 确保有相邻文字节点的空文字节点被合并

合法性变更之后，就是调用 `onChange` 方法。这个方法 slate package 中定义的是一个空函数，实际上是为插件准备的一个“model 已经变更”的回调。

到这里，对 slate model 的介绍就告一段落了。

## slate 插件机制

在进一步学习其他 package 之前，我们先要学习一下 slate 的插件机制以了解各个 package 和如何与核心 package 合作的。

上一节提到的判断一个节点是否为行内节点的 `isInline` 方法，以及 `normalizeNode` 方法本身都是可以被扩展，不仅如此，另外三个 package 包括 undo/redo 功能和渲染层均是以插件的形式工作的。看起来 slate 的插件机制非常强大，但它有一个非常简单的实现：**覆写编辑器实例 editor 上的方法**。

slate-react 提供的 `withReact` 方法给我们做了一个很好的示范：

```ts
export const withReact = <T extends Editor>(editor: T) => {
  const e = editor as T & ReactEditor
  const { apply, onChange } = e

  e.apply = (op: Operation) => {
    // ...
    apply(op)
  }

  e.onChange = () => {
    // ...
    onChange()
  }
}
```

用 `withReact` 修饰编辑器实例，直接覆盖实例上原本的 `apply` 和 `change` 方法。~~换句话说，slate 的插件机制就是没有插件机制！~~这难道就是传说中的无招胜有招？

## slate-history

学习了插件机制，我们再来看 undo/redo 的功能，它由 slate-history package 所实现。

实现 undo/redo 的机制一般来说有两种。第一种是存储各个时刻（例如发生变更前后）model 的快照（snapshot），在撤销操作的时候恢复到之前的快照，这种机制看起来简单，但是较为消耗内存（有 n 步操作我们就需要存储 n+1 份数据！），而且会使得协同编辑实现起来非常困难（比较两个树之间的差别的时间复杂度是 O(n^3)，更不要提还有网络传输的开销）。第二种是记录变更的应用记录，在撤销操作的时候取要撤销操作的反操作，这种机制复杂一些——主要是要进行各种选区计算——但是方便做协同，且不会占用较多的内存空间。slate 即基于第二种方法进行实现。

在 `withHistory` 方法中，slate-history 在 editor 上创建了两个数组用来存储历史操作：

```ts
e.history = { undos: [], redos: [] }
```

它们的类型都是 `Operation[][]`，即 `Operation` 的二维数组，其中的每一项代表了一批操作（在代码上称作 batch）， batch 可含有多个 `Operation`。

我们可以通过 console 看到这一结构：

<img width="1433" src="https://user-images.githubusercontent.com/12122021/95169758-f7ba0280-07e5-11eb-959f-ba9b57530200.png" />

slate-history 通过覆写 `apply` 方法来在 `Operation` 的 apply 流程之前插入 undo/redo 的相关逻辑，这些逻辑主要包括：

- 判断是否需要存储该 `Operation`，诸如改变选区位置等操作是不需要 undo 的
- 判断该 `Operation` 是否需要和前一个 batch 合并，或覆盖前一个 batch
- 创建一个 batch 插入 `undos` 队列，或者插入到上一个 batch 的尾部，同时计算是否超过最大撤销步数，超过则去除首部的 batch
- 调用原来的 `apply` 方法

```ts
e.apply = (op: Operation) => {
  const { operations, history } = e
  const { undos } = history
  const lastBatch = undos[undos.length - 1]
  const lastOp = lastBatch && lastBatch[lastBatch.length - 1]
  const overwrite = shouldOverwrite(op, lastOp)
  let save = HistoryEditor.isSaving(e)
  let merge = HistoryEditor.isMerging(e)

  // 判断是否需要存储该 operation
  if (save == null) {
    save = shouldSave(op, lastOp)
  }

  if (save) {
    // 判断是否需要和上一个 batch 合并
    // ...

    if (lastBatch && merge) {
      if (overwrite) {
        lastBatch.pop()
      }

      lastBatch.push(op)
    } else {
      const batch = [op]
      undos.push(batch)
    }

    // 最大撤销 100 步
    while (undos.length > 100) {
      undos.shift()
    }

    if (shouldClear(op)) {
      history.redos = []
    }
  }

  apply(op)
}
```

slate-history 还在 editor 实例上赋值了 `undo` 方法，用于撤销上一组操作：

```ts
e.undo = () => {
  const { history } = e
  const { undos } = history

  if (undos.length > 0) {
    const batch = undos[undos.length - 1]

    HistoryEditor.withoutSaving(e, () => {
      Editor.withoutNormalizing(e, () => {
        const inverseOps = batch.map(Operation.inverse).reverse()

        for (const op of inverseOps) {
          // If the final operation is deselecting the editor, skip it. This is
          if (
            op === inverseOps[inverseOps.length - 1] &&
            op.type === 'set_selection' &&
            op.newProperties == null
          ) {
            continue
          } else {
            e.apply(op)
          }
        }
      })
    })

    history.redos.push(batch)
    history.undos.pop()
  }
}
```

这个算法的主要部分就是对最后一个 batch 中所有的 `Operation` 取反操作然后一一 apply，再将这个 batch push 到 `redos` 数组中。

redo 方法就更简单了，这里不再赘述。

## slate-react

最后我们来探究渲染和交互层，即 slate-react package。

### 渲染机制

我们最关注的问题当然是 model 是如何转换成视图层（view）的。经过之前的学习我们已经了解到 slate 的 model 本身就是树形结构，因此只需要递归地去遍历这棵树，同时渲染就可以了。基于 React，这样的递归渲染用几个组件就能够很容易地做到，这几个组件分别是 `Editable` `Children` `Element` `Leaf` `String` 和 `Text`。在这里举几个例子：

`Children` 组件用来渲染 model 中类行为 `Editor` 和 `Element` `Node` 的 `children`，比如最顶层的 `Editable` 组件就会渲染 `Editor` 的 `children`：

_注意下面的 `node` 参数即为编辑器实例 `Editor`：_

```tsx
export const Editable = (props: EditableProps) => {
  return (
    <Wrapped>
      <Children
        decorate={decorate}
        decorations={decorations}
        node={editor}
        renderElement={renderElement}
        renderLeaf={renderLeaf}
        selection={editor.selection}
      />
    </Wrapped>
  )
}
```

`Children` 组件会根据 `children` 中各个 `Node` 的类型，生成对应的 `ElementComponent` 或者 `TextComponent`：

```ts
const Children = (props) => {
  const { node, renderElement, renderLeaf } = props
  for (let i = 0; i < node.children.length; i++) {
    const p = path.concat(i)
    const n = node.children[i] as Descendant

    if (Element.isElement(n)) {
      children.push(
        <ElementComponent
          element={n}
          renderElement={renderElement}
          renderLeaf={renderLeaf}
        />
      )
    } else {
      children.push(<TextComponent renderLeaf={renderLeaf} text={n} />)
    }
  }

  return <React.Fragment>{children}</React.Fragment>
}
```

`ElementComponent` 渲染一个 `Element` 元素，并用 `Children` 组件渲染其 `children`：

```tsx
const Element = (props) => {
  let children: JSX.Element | null = (
    <Children
      decorate={decorate}
      decorations={decorations}
      node={element}
      renderElement={renderElement}
      renderLeaf={renderLeaf}
      selection={selection}
    />
  )

  return (
    <SelectedContext.Provider value={!!selection}>
      {renderElement({ attributes, children, element })}
    </SelectedContext.Provider>
  )
}

// renderElement 的默认值
export const DefaultElement = (props: RenderElementProps) => {
  const { attributes, children, element } = props
  const editor = useEditor()
  const Tag = editor.isInline(element) ? 'span' : 'div'
  return (
    <Tag {...attributes} style={{ position: 'relative' }}>
      {children}
    </Tag>
  )
}
```

`Leaf` 等组件的渲染也是同理，这里不再赘述。

下图表示了从 model tree 到 React element 的映射，可见用树形结构来组织 model 能够很方便地渲染，且在 `Node` 和 HTML element 之间建立映射关系（具体可查看 `toSlateNode` 和 `toSlateRange` 等方法和 `ELEMENT_TO_NODE` `NODE_TO_ELEMENT` 等数据结构），这在处理光标和选择事件时将会特别方便。

![未命名作品 2](https://user-images.githubusercontent.com/12122021/95169793-056f8800-07e6-11eb-816d-45f12676603c.png)

slate-react 还用了 `React.memo` 来优化渲染性能，这里不赘述。

### 自定义渲染元素

在上面探究 slate-react 的渲染机制的过程中，我们发现有两个比较特殊的参数 `renderElement` 和 `renderLeaf`，它们从最顶层的 `Editable` 组件开始就作为参数，一直传递到最底层的 `Leaf` 组件，并且还会被 `Element` 等组件在渲染时调用，它们是什么？

实际上，这就是 slate-react 自定义渲染的 API，用户可以通过提供这两个参数来自行决定如何渲染 model 中的一个 `Node`，例如 richtext demo 中：

```tsx
const Element = ({ attributes, children, element }) => {
  switch (element.type) {
    case 'block-quote':
      return <blockquote {...attributes}>{children}</blockquote>
    case 'bulleted-list':
      return <ul {...attributes}>{children}</ul>
    case 'heading-one':
      return <h1 {...attributes}>{children}</h1>
    case 'heading-two':
      return <h2 {...attributes}>{children}</h2>
    case 'list-item':
      return <li {...attributes}>{children}</li>
    case 'numbered-list':
      return <ol {...attributes}>{children}</ol>
    default:
      return <p {...attributes}>{children}</p>
  }
}
```

我们先前提到 slate 允许 `Node` 有自定义属性，这个 demo 就拓展了 `Element` 节点的 `type` 属性，让 `Element` 能够渲染为不同的标签。

### 光标和选区的处理

slate 没有自行实现光标和选区，而使用了浏览器 `contenteditable` 的能力（同时也埋下了隐患，我们会在总结部分介绍）。

在 `Editable` 组件中，可看到对 `Component` 元素增加了 contenteditable attribute：

```tsx
export const Editable = (props: EditableProps) => {
  return (
    <Wrapped>
      <Copmonent
        contentEditable={readOnly ? undefined : true}
        suppressContentEditableWarning
      ></Copmonent>
    </Wrapped>
  )
}

// Component 默认为 'div'
```

从这里开始，contenteditable 就负责了光标和选区的渲染和事件。slate-react 会在每次渲染的时候将 model 中的选区同步到 DOM 上：

```tsx
export const Editable = (props: EditableProps) => {
  // ...
  useIsomorphicLayoutEffect(() => {
    // ...
    domSelection.setBaseAndExtent(
      newDomRange.startContainer,
      newDomRange.startOffset,
      newDomRange.endContainer,
      newDomRange.endOffset
    )
  })
}
```

也会在 DOM 发生选区事件的时候同步到 model 当中：

```tsx
const onDOMSelectionChange = useCallback(
  throttle(() => {
    if (!readOnly && !state.isComposing && !state.isUpdatingSelection) {
      // ...
      if (anchorNodeSelectable && focusNodeSelectable) {
        const range = ReactEditor.toSlateRange(editor, domSelection) // 这里即发生了一次 DOM element 到 model Node 的转换
        Transforms.select(editor, range)
      } else {
        Transforms.deselect(editor)
      }
    }
  }, 100),
  [readOnly]
)
```

选区同步的方法这里就不介绍了，大家可以通过查阅源码自行学习。

### 键盘事件的处理

`Editable` 组件创建了一个 `onDOMBeforeInput` 函数，用以处理 `beforeInput` 事件，根据事件的 `type` 调用不同的方法来修改 model。

```ts
// ...

switch (type) {
  case 'deleteByComposition':
  case 'deleteByCut':
  case 'deleteByDrag': {
    Editor.deleteFragment(editor)
    break
  }

  case 'deleteContent':
  case 'deleteContentForward': {
    Editor.deleteForward(editor)
    break
  }

  // ...
}

// ...
```

_`beforeInput` 事件和 `input` 事件的区别就是触发的时机不同。前者在值改变之前触发，还能通过调用 `preventDefault` 来阻止浏览器的默认行为。_

slate 对快捷键的处理也很简单，通过在 div 上绑定 keydown 事件的 handler，然后根据不同的组合键调用不同的方法。slate-react 也提供了自定义这些 handler 的接口，`Editable` 默认的 handler 会检测用户提供的 handler 有没有将该 keydown 事件标记为 `defaultPrevented`，没有才执行默认的事件处理逻辑：

```ts
if (
  !readOnly &&
  hasEditableTarget(editor, event.target) &&
  !isEventHandled(event, attributes.onKeyDown)
) {
  // 根据不同的组合键调用不同的方法
}
```

### 渲染触发

slate 在渲染的时候会向 `EDITOR_TO_ON_CHANGE` 中添加一个回调函数，这个函数会让 `key` 的值加 1，触发 React 重新渲染。

```tsx
export const Slate = (props: {
  editor: ReactEditor
  value: Node[]
  children: React.ReactNode
  onChange: (value: Node[]) => void
  [key: string]: unknown
}) => {
  const { editor, children, onChange, value, ...rest } = props
  const [key, setKey] = useState(0)

  const onContextChange = useCallback(() => {
    onChange(editor.children)
    setKey(key + 1)
  }, [key, onChange])

  EDITOR_TO_ON_CHANGE.set(editor, onContextChange)

  useEffect(() => {
    return () => {
      EDITOR_TO_ON_CHANGE.set(editor, () => {})
    }
  }, [])
}
```

而这个回调函数由谁来调用呢？可以看到 `withReact` 对于 `onChange` 的覆写：

```ts
e.onChange = () => {
  ReactDOM.unstable_batchedUpdates(() => {
    const onContextChange = EDITOR_TO_ON_CHANGE.get(e)

    if (onContextChange) {
      onContextChange()
    }

    onChange()
  })
}
```

在 model 变更的结束阶段，从 `EDITOR_TO_ON_CHANGE` 里拿到回调并调用，这样就实现 model 更新，触发 React 重渲染了。

## 总结

这篇文章分析了 slate 的架构设计和对一些关键问题的处理，包括：

- model 数据结构的设计
- 如何以原子化的方式进行 model 的变更
- 对 model 的合法性校验
- 插件系统
- undo/redo 的实现
- 渲染机制
- UI 到 model 的映射
- 光标和选区的处理

等等。至此，我们可以发现 slate 存在着这样几个主要的问题：

**没有自行实现排版**。slate 借助了 DOM 的排版能力，但 DOM 的能力是有局限的，不能实现页眉页脚、图文混排等高级排版功能。

**使用了 contenteditable 导致无法处理部分选区和输入事件**。使用 contenteditable 后虽然不需要开发者去处理光标的渲染和选择事件，但是造成了另外一个问题：破坏了从 model 到 view 的单向数据流，这在使用输入法（IME）的时候会导致崩溃这样严重的错误。

我们在 React 更新渲染之前打断点，然后全选文本，输入任意内容。可以看到，在没有输入法的状态下，更新之前 DOM element 并没有被移除。

<img width="1378" alt="截屏2020-09-28 下午5 19 37" src="https://user-images.githubusercontent.com/12122021/95169905-3485f980-07e6-11eb-9e04-dae8d80236aa.png" />

但是在有输入法的情况下，contenteditable 会将光标所选位置的 DOM element 先行清除，此时 React 中却还有对应的 Fiber Node，这样更新之后，React 就会发现需要卸载的 Fiber 所对应的 DOM element 已经不属于其父 element，从而报错。并且这一事件不能被 prevent default，所以单向数据流一定会被打破。

<img width="1378" alt="截屏2020-09-28 下午5 20 45" src="https://user-images.githubusercontent.com/12122021/95169889-2cc65500-07e6-11eb-9551-8d6284349584.png" />

[React 相关的 issue](https://github.com/facebook/react/issues/) 从 2015 年起就挂在那里了。slate 官方对 IME 相关的问题的积极性也不高。

**对于协同编辑的支持仅停留在理论可行性上**。slate 使用了 `Operation`，这使得协同编辑存在理论上的可能，但是对于协同编辑至关重要的 operation transform 方案（即如何处理两个有冲突的编辑操作），则没有提供实现。

---

总的来说，slate 是一个拥有良好扩展性的轻量富文本编辑器（框架），很适合 CMS、社交媒体这种不需要复杂排版和实时协作的简单富文本编辑场景。

希望这篇文章能够帮助大家对 slate 形成一个整体的认知，并从其技术方案中了解它的优点和局限性，从而更加得心应手地使用 slate。
