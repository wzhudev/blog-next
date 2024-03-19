---
title: 如何给大型前端开源项目贡献源码
date: 2019/10/16
description: 用一个例子介绍如何开始给开源项目贡献代码
tag: OSS, tutorial, Chinese
author: Wenzhao
---

# 如何给大型前端开源项目贡献源码

参与开源项目对个人的好处是显而易见的：进一步熟悉你所使用的库、和有经验的开发者进行交流、提高代码水平和社区影响力，甚至是找到新的工作机会。特别是对于没有实习经验的在校大学生来说，参与开源项目是体现技术热情和积累实际开发经验的一个好方法，能让你在简历筛选和面试环节得到面试官的青睐。这篇文章将会和大家分享如何参与大型前端开源项目的开发。你将会看到，开源的门槛其实并不高，只要有意愿，并掌握了最基本的流程，任何人都可以给大型前端开源项目贡献代码。

为了方便你理解这篇文章的内容，我会以我最近给西湖区最大的 React 组件库 [Ant Design](https://ant.design) 提交的[一段代码](https://github.com/ant-design/ant-design/pull/18678)为例来讲解流程，这段代码是为了满足用户的[一个功能需求](https://github.com/ant-design/ant-design/issues/18621)，即通过唯一的 key 来修改[消息框（Message）](https://ant.design/components/message-cn/#header)的内容。

## 贡献流程

### 找到想要贡献的项目和 issue

绝大多数的前端开源项目都托管在 [GitHub](https://github.com) 上，开发和维护也在 GitHub 上进行，开源项目的用户（开发者）们会将他们找到的（疑似） bug 或是功能请求提交到这些项目的 issues 中，给大型前端开源项目贡献代码的第一步就从这里开始，即从中找到你感兴趣的 issue。

> 如果你还不了解 GitHub 及它的基本使用，请参考我朋友的知乎回答：[《如何使用 GitHub》](https://www.zhihu.com/question/20070065/answer/79557687)。

比如 [Ant Design 的 issues 中](https://github.com/ant-design/ant-design/issues)，你可以看到许多这样的 bug 反馈和功能请求。

![1](https://user-images.githubusercontent.com/12122021/66885974-8e574780-f008-11e9-8c86-75762f09a306.png)

**Tips：如何找到适合自己的项目和 issue，哪些 issue 比较容易上手？**

对于新手来说，最好找这样的一些项目：

- 工作和学习中使用较多，比较熟悉的项目，这样你在使用之前已经了解了它的用途、API 等等。
- 各个模块耦合性比较低的项目，比如组件库（比如 Ant Design）、工具库（比如 lodash），方便定位问题，找到入手点。

以及这样的 issue：

> 如果你是第一次向大型开源项目提交代码，要记得所解决的问题的难度并不重要，**重要的是走一遍贡献代码的流程**，了解开源社区是如何协作的。

- 小的功能需求，明显可以复用项目中已有的代码。在本文的例子中，用户所请求的功能其实在 [Notification](https://ant.design/components/notification-cn/#components-notification-demo-update) 组件上已经有实现，这样对我们解决这个 issue 就很有帮助。
- 一些 issue 会被项目官方团队标记为 Good First Issue，是官方认为比较适合初次贡献者的，比如 Angular 组件库 [ng-zorro-antd 就有这样的一些 issue](https://github.com/NG-ZORRO/ng-zorro-antd/issues?q=is%3Aopen+is%3Aissue+label%3A%22Good+First+Issue%22)，你可以从这里开始。
- 错别字、文档翻译、国际化。这些一般都不 touch 到库的核心功能，难度较低而且不会引入 bug，可以放心地把它们作为你的开源初体验。

找到了想要解决的 issue 之后，在 issue 下面留言说你想要负责这个 issue，一般项目的维护者都会把这个 issue 交给你。你可以看到我在原 issue 下的[回复](https://github.com/ant-design/ant-design/issues/18621#issuecomment-527287730)。

> 维护者们都巴不得有人来完（tian）善（keng）他们的项目呢，所以尽管留言吧！

### 了解并运行项目

好了，现在我们手头已经有了一个待解决的 issue，接下来我们需要了解这个项目是如何运作的。

> 阅读源码的能力是必须的，这里推荐阅读[《如何阅读大型前端开源项目的代码》](https://github.com/ProtoTeam/blog/blob/master/201805/3.md)。

了解项目的第一步永远都是先把这个项目在本地运行起来。

#### fork

> 如果你在阅读下面的内容的时候发现对一些概念一无所知，请立刻回头阅读《如何使用 GitHub》。

首先你要 fork 该项目，fork 意味者创建一份源仓库的拷贝，在贡献代码的时候，我们没有向源仓库推送（push）代码的权限，往往都需要先推送到自己的拷贝上，然后请求项目的维护者们合并我们的新代码，即发起 Pull Request。

你可以看到我的账户底下就有[一份 ant-design 的 fork](https://github.com/wendzhue/ant-design?organization=wendzhue&organization=wendzhue)。

![2](https://user-images.githubusercontent.com/12122021/66886003-9ca56380-f008-11e9-92dd-b2f6ccc746c6.png)

#### clone

然后把 fork 后的代码 clone 到你的电脑。

![3](https://user-images.githubusercontent.com/12122021/66886008-9f07bd80-f008-11e9-8d48-c77a20b2a666.png)

#### 安装依赖

通过 npm 或者 yarn 安装依赖。

#### 运行项目

一般通过查看 package.json 文件的 scripts 字段，就可以知道如何运行该项目，进行测试等等。对于 Ant Design，只需要运行 `npm start` 就可以。

![4](https://user-images.githubusercontent.com/12122021/66886012-a202ae00-f008-11e9-9b71-7710437dcc38.png)

### 解决问题

下面我们就要解决 issue 中提出的功能请求了。

> 这一段会比较 Ant Design specific，对于其他开源项目不具有通用性，不感兴趣的读者们可以直接跳到下一章提交 PR。

我们先前提到 Notifcation 组件早已实现了此功能，先来看看它是如何实现的。

我们可以追溯到，当用户通过 `Notification.open` 方法创建一个 Notification 实例的时候，最终会调用到 `getNotificationInstance` 方法上（[源码在此](https://github.com/ant-design/ant-design/blob/0a09b3b0085dc2715f6725602bcd04eacf1edb25/components/notification/index.tsx#L90-L118)）。

```tsx
  getNotificationInstance(
    {
      prefixCls: outerPrefixCls,
      placement,
      top,
      bottom,
      getContainer,
    },
    (notification: any) => {
      notification.notice({
        content: (
          <div className={iconNode ? `${prefixCls}-with-icon` : ''}>
            {iconNode}
            <div className={`${prefixCls}-message`}>
              {autoMarginTag}
              {args.message}
            </div>
            <div className={`${prefixCls}-description`}>{args.description}</div>
            {args.btn ? <span className={`${prefixCls}-btn`}>{args.btn}</span> : null}
          </div>
        ),
        duration,
        closable: true,
        onClose: args.onClose,
        onClick: args.onClick,
        key: args.key,
        style: args.style || {},
        className: args.className,
      });
    },
  );
}
```

可以看到 `key` 是第二个回调参数的一部分。那么在 Message 组件中有没有类似的代码呢？可以发现的确存在这样的代码（[链接在此](https://github.com/ant-design/ant-design/blob/0a09b3b0085dc2715f6725602bcd04eacf1edb25/components/message/index.tsx#L58-L108)）！

```tsx
function notice(args: ArgsProps): MessageType {
  const duration = args.duration !== undefined ? args.duration : defaultDuration
  const iconType = {
    info: 'info-circle',
    success: 'check-circle',
    error: 'close-circle',
    warning: 'exclamation-circle',
    loading: 'loading'
  }[args.type]

  const target = key++
  const closePromise = new Promise((resolve) => {
    const callback = () => {
      if (typeof args.onClose === 'function') {
        args.onClose()
      }
      return resolve(true)
    }
    getMessageInstance((instance) => {
      const iconNode = (
        <Icon
          type={iconType}
          theme={iconType === 'loading' ? 'outlined' : 'filled'}
        />
      )
      const switchIconNode = iconType ? iconNode : ''
      instance.notice({
        key: target,
        duration,
        style: {},
        content: (
          <div
            className={`${prefixCls}-custom-content${
              args.type ? ` ${prefixCls}-${args.type}` : ''
            }`}
          >
            {args.icon ? args.icon : switchIconNode}
            <span>{args.content}</span>
          </div>
        ),
        onClose: callback
      })
    })
  })
  const result: any = () => {
    if (messageInstance) {
      messageInstance.removeNotice(target)
    }
  }
  result.then = (filled: ThenableArgument, rejected: ThenableArgument) =>
    closePromise.then(filled, rejected)
  result.promise = closePromise
  return result
}
```

那我们是不是依样画葫芦，直接允许 `key` 使用用户传递进来的值呢？[试了一下果然可以](https://github.com/ant-design/ant-design/pull/18678/files#diff-9f7e4e8dd30d44d395e26df67afc40f9L82)！

```diff
- key: target,
+ key: args.key || target,
```

这功能实现起来就非常简单。我们接着修改一下测试，添加一下 demo 即可。

以下是对该功能的测试代码（[链接在此](https://github.com/ant-design/ant-design/pull/18678/files#diff-f3e195ef456745ef223fa5034c7257e3R149-R171)）：

```tsx
it('should support update message content with a unique key', () => {
  const key = 'updatable'
  class Test extends React.Component {
    componentDidMount() {
      message.loading({ content: 'Loading...', key })
      // Testing that content of the message should be updated.
      setTimeout(() => message.success({ content: 'Loaded', key }), 1000)
      setTimeout(() => message.destroy(), 3000)
    }

    render() {
      return <div>test</div>
    }
  }

  mount(<Test />)
  expect(document.querySelectorAll('.ant-message-notice').length).toBe(1)
  jest.advanceTimersByTime(1500)
  expect(document.querySelectorAll('.ant-message-notice').length).toBe(1)
  jest.runAllTimers()
  expect(document.querySelectorAll('.ant-message-notice').length).toBe(0)
})
```

> 开源项目特别重视测试，还会有覆盖率检查工具来检查是否有代码未被测试覆盖，当你修复了一个 bug 或者新增了一个功能的时候，记得一定要写测试！

再改改文档，说明我们增加了这样的一个功能，就可以发起 PR 啦。

### 提交 PR

发起 PR（Pull Request）时，需要 commit 代码，然后 push 到你 fork 的仓库。

> 如果你同时给一个项目解决好几个 issue，你应该从 master 分支 checkout 出多个分支，然后分别在这些分支上解决 issue。更好的实践是永远 checkout 一个新分支来解决 issue，不要向 master 分支提交任何代码。

> 提交 PR 之前，你并不需要确定已经做到尽善尽美了，肯定有一些东西要和项目维护者们进行讨论。

之后在 GitHub 上发起 PR，当你在 fork 的仓库推送了分支时，GitHub 会很聪明地询问你是否要发起 PR，点击绿色的小按钮之后，填写 PR 模板即可发起 PR。

![5](https://user-images.githubusercontent.com/12122021/66886021-a929bc00-f008-11e9-946c-adecabfb59f4.png)

![6](https://user-images.githubusercontent.com/12122021/66886022-a929bc00-f008-11e9-9335-49181f8ca666.png)

### 和维护者们有来有回

“烦人”的维护者们是不会轻易让你的代码进入他们的主分支的！你必须接受维护者们的代码 review，有时候他们会要求你做出一些修改。

比如 afc163 认为[加入 key 后参数数量过多，要求实现以对象的形式传参](https://github.com/ant-design/ant-design/pull/18678#discussion_r321602034)。

> 当然他们也是为了保证代码的质量而非存心跟你过不去，而且和维护者们的互动有助于你写出更鲁棒的代码和更精炼的 API 哦。

这种 review 和修改和互动可能会有很多轮。如果维护者们对你的 PR 和修改没有做出回应，你可以主动 @ 他们。

### 成为 contributor

当维护者们对所有事情都表示满意，approve 了你的 PR，你就可以坐等代码被合并到 master 并成为该项目的 contributor 啦。

> 之后你参与该项目的讨论时就会有个 Contributor 小徽章时刻提醒那些小白你是多么的牛逼（逃

![7](https://user-images.githubusercontent.com/12122021/66886031-ad55d980-f008-11e9-8eb7-867d5c25e292.png)

## 温馨建议

- 保持礼貌。
- 学好 Git，包括但不限于这些常用命令：`fetch` `pull` `add` `commit` `push` `merge` `rebase`。
- 在给任何项目做贡献之前，请务必阅读该项目的贡献指南，比如 [Ant Design 的](https://ant.design/docs/react/contributing-cn#%E7%AC%AC%E4%B8%80%E6%AC%A1%E8%B4%A1%E7%8C%AE)。这些指南往往能够帮助你更好地了解维护团队的工作流程，有些项目对 commit message 还有一定的要求，请务必遵照执行，这会让你的开源之旅更加顺畅。
- 时刻关注想要贡献的项目的 issues 界面，有想做的 issue 直接留言。
- 关注官方团队的开发者博客或者维护者的个人博客，了解项目的最新动态和技术细节。
- 不把参与开源当作很难的事。

Happy coding :)
