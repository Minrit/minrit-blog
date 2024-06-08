---
title: EventStream是个啥玩意
date: 2024-06-08 13:43:35
tags: 技术
---

你有想过类似ChatGPT的聊天客户端是怎么一个字个字地把大模型的回复返回回来的吗？


<!-- more -->


自从前年底GPT-3.5发布以来，各种不同的大模型与聊天机器人如雨后春笋一般冒了出来。笔者在试用这些不同的客户端的时候发现了一个特点，机器人给的回复都是一个字一个字出来的，而不是一轮对话生成完了直接出来。本来笔者以为是建立了一个WebSocket之类的全双工通信来实现的，仔细研究后发现原来是通过一个叫做SSE（Server-Sent Events）的技术实现的。


## 何为SSE

SSE全称叫做Server-Sent Events，基于HTTP，通过向客户端声明流的方式来告诉客户端不要关闭HTTP连接直到流式传输完毕。我们直接来看一个调用OPEN AI对话接口的`v1/chat/completions`响应体头：

```
Access-Control-Allow-Credentials:true
Access-Control-Allow-Origin:*
Cache-Control:no-cache
Content-Type:text/event-stream
Date:Sat, 08 Jun 2024 05:40:17 GMT
Server:nginx
```

大部分和一般的请求响应头没有什么区别，这里的关键就在于里面的`Content-Type:text/event-stream`，通过声明event-stream来告诉浏览器接下来返回的东西是流式传输，浏览器会一直保持HTTP连接的活跃直到接收完服务器推过来的信息。

我们再来看返回体：

```
data: {"choices":[{"delta":{"content":"","role":"assistant"},"finish_reason":null,"index":0,"logprobs":null}],"created":1717825216,"id":"chatcmpl-9Xj6GuRYQLqCaK4r4lrFUZlA1Bnzp","model":"gpt-4-0613","object":"chat.completion.chunk","system_fingerprint":null}

data: {"choices":[{"delta":{"content":"当"},"finish_reason":null,"index":0,"logprobs":null}],"created":1717825216,"id":"chatcmpl-9Xj6GuRYQLqCaK4r4lrFUZlA1Bnzp","model":"gpt-4-0613","object":"chat.completion.chunk","system_fingerprint":null}

data: {"choices":[{"delta":{"content":"然"},"finish_reason":null,"index":0,"logprobs":null}],"created":1717825216,"id":"chatcmpl-9Xj6GuRYQLqCaK4r4lrFUZlA1Bnzp","model":"gpt-4-0613","object":"chat.completion.chunk","system_fingerprint":null}

// ... 这里省略中间部分，都是一句句data
// ...
// ...

data: {"choices":[{"delta":{"content":"吗"},"finish_reason":null,"index":0,"logprobs":null}],"created":1717825216,"id":"chatcmpl-9Xj6GuRYQLqCaK4r4lrFUZlA1Bnzp","model":"gpt-4-0613","object":"chat.completion.chunk","system_fingerprint":null}

data: {"choices":[{"delta":{"content":"？"},"finish_reason":null,"index":0,"logprobs":null}],"created":1717825216,"id":"chatcmpl-9Xj6GuRYQLqCaK4r4lrFUZlA1Bnzp","model":"gpt-4-0613","object":"chat.completion.chunk","system_fingerprint":null}

data: {"choices":[{"delta":{},"finish_reason":"stop","index":0,"logprobs":null}],"created":1717825216,"id":"chatcmpl-9Xj6GuRYQLqCaK4r4lrFUZlA1Bnzp","model":"gpt-4-0613","object":"chat.completion.chunk","system_fingerprint":null}

data: [DONE]
```

和我们平常见到的返回体是不是也不太一样，它的返回体是一行一行的消息，每一行都以`data:`开头，后面跟着一个JSON对象。`data:`是固定的Field关键字，代表这一行发过来的是数据。Field的类型一共有四种：data、id、event与retry，具体不同类型的意思可以具体参考[MDN文档](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)

在最后面有一个`data: [DONE]`表示消息发完了。

## 前端如何解析

这里我们贴一下知名的开源聊天机器人客户端的代码[Chatbox](https://github.com/Bin-Huang/chatbox)来做分析：

首先是发送请求，这里和普通的POST请求没什么两样：
```ts
const response = await fetch(`${host}/v1/chat/completions`, {
    method: 'POST',
    headers: {
        Authorization: `Bearer ${apiKey}`,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        messages,
        model: modelName,
        max_tokens: maxTokensNumber,
        temperature,
        stream: true,
    }),
    signal: controller.signal,
})
```
重点是下面对于`response`的解析：
```ts
await handleSSE(response, (message) => {
    if (message === '[DONE]') {
        return
    }
    const data = JSON.parse(message)
    if (data.error) {
        throw new Error(`Error from OpenAI: ${JSON.stringify(data)}`)
    }
    const text = data.choices[0]?.delta?.content
    if (text !== undefined) {
        fullText += text
        if (onText) {
            onText({ text: fullText, cancel })
        }
    }
})
```
分析逻辑，可以看到这里调用了`handleSSE`方法，把`response`和一个回调函数传了进去，这里的SSE就是我们刚刚所说的SSE。那么核心逻辑就肯定在这个`handleSSE`方法里面，这个一会分析，我们先来看这个回调函数。

如果回调获得的`message`是`[DONE]`的话就直接return，这个`[DONE]`显然就是我们刚刚在返回体里面看到的`data: [DONE]`。接下来是调用了`JSON.parse`，那么就是解析剩下的为JSON返回格式的真正有意义的message的信息，这里就拿到了实际的消息。

来看关键的`handleSSE`方法：

```ts
export async function handleSSE(response: Response, onMessage: (message: string) => void) {
    // 前面有一大坨的错误处理，和我们的分析无关，省略，核心就是下面这几行
    const parser = createParser((event) => {
        if (event.type === 'event') {
            onMessage(event.data)
        }
    })
    for await (const chunk of iterableStreamAsync(response.body)) {
        const str = new TextDecoder().decode(chunk)
        parser.feed(str)
    }
}
```

首先是通过调用`createParser`方法创建了一个`parse`，这个方法是`eventsource-parser`这个库提供的，这是一个专门用来解析SSE返回体的库，这里的回调函数里面调用了我们外部的`onMessage`回调函数。

接下来是核心，可以看到一个for循环，调用了一个JS生成器`iterableStreamAsync`，来看它的具体实现：

```ts
export async function* iterableStreamAsync(stream: ReadableStream): AsyncIterableIterator<Uint8Array> {
    const reader = stream.getReader()
    try {
        while (true) {
            const { value, done } = await reader.read()
            if (done) {
                return
            } else {
                yield value
            }
        }
    } finally {
        reader.releaseLock()
    }
}
```

这个方法接受了一个二进制流，通过JS原生对象ReadableStream提供的reader来不断读取二进制流里面的值，通过`yield`关键字将值传出来。

所以for循环里面拿到的这个`chunk`就是拿到的一个个流，随后通过`TextDecoder`将二进制流转换为字符串再喂给我们的`parser`，`parser`接收到了喂进来的数据之后就会对数据做解析并通过`onMessage`回调的方式把解析好的数据传给我们，至此，整个流程完成。

还有个要注意的点就是在生成器函数里面调用了`reader.releaseLock()`，这是一个解锁方法，当通过`reader.read()`读取数据时，会自动锁定这个流。当读取完后调用`releaseLock`来释放。