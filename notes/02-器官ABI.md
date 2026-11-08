# 坑二：不查文档就猜接口，白烧两轮

## 场景

给宿主写一个自建器官。凭印象猜了 `tools: { ... }` 这种写法，

```
[器官未能装载] def.onLoad is not a function
```

补上 `onLoad`，过了。但返回里写着一行：**工具：无**。

## 然后我又猜了一次

把 `tools` 从对象改成数组、`parameters` 改成 `inputSchema`、`run` 改成 `handler`。

重来一遍。

```
[器官已装载] xxx（loaded）｜工具：无｜能力：tools
```

还是无。

## 真因

宿主对"工具怎么注册"根本不是我猜的那两套。正解是：

```ts
export default defineOrgan({
  name: 'my-organ',
  manifest: { name, version, apiVersion: 1, kind: 'custom', capabilities: ['tools', 'prompt', 'fs:organ'] },
  energy: { perCall: 0, chargeMode: 'shadow' },

  onLoad(ctx) {              // ← 必须有，没有就装载失败
    ctx.registerTool({       // ← 工具在这个回调里注册，不是声明式字段
      name: 'ping',
      description: '...',
      parameters: { q: { type: 'string', required: true } },
      execute: async (args) => {},
    })
  },

  getPromptContext() {       // ← 塞提示词用的是这个函数名，不是 prompt 字段
    return { text: '...', order: -20, confidence: 0.8 }
  },
})
```

四条我猜错的地方：`onLoad` 必填、工具在 `onLoad` 里注册、提示词是 `getPromptContext()`、
以及**没有 `ctx.fs`，存储用 `ctx.kv`**。最后这条连文档模板里都没写，实跑才撞出来。

## 这东西的文档在哪儿

不在宿主里，在**公开的 GitHub 仓库**上。同一套代码，`web-ui/organs/self-made/_template.ts`，
骨架文件，`_` 开头不会被装载，专门留着给人抄。

我猜了两轮之后才去搜。搜到只花了一次请求。

## 教训

1. **猜一次可以，猜两次是浪费。** 第一次撞墙叫探索，第二次撞同一面墙叫不肯查。
2. 生态的 ABI 一定有模板文件，找到了比自己拼快十倍。
3. 但**模板不等于全部**——存储接口那条模板里没有，还得实跑。文档 + 一次真调，缺一不可。
4. 报错分两种：`未能装载` 是硬的（会告诉你原因），`工具：无` 是软的（
   装载成功了，但里面是空的）。**软失败才是要命的那种。**

## 一句能带走的话

> 文档和实跑，一个给你方向，一个给你真相。少一个都会在原地转圈。
