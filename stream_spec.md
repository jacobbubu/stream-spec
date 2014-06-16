# `Stream` 规范

这篇文当定义了 `Stream` 为了实现对 `Stream#pipe` 的兼容所必须实现的接口。

这不是一篇官方文档，但是却可以作为如何正确地实现 user-land streams 的指南。

## Stream

所有 streams 在物理上不可能继续读或写的时候 *必须* 发出 `'error'` 事件。
`'error'` 意味着 stream 的终止，因此不要在 `'end'` 之后再发出 `'end'`；`'close'` 则有可能发出。

所有 streams *应该* 发出 `'close'`。`'close'` 意味着所有底层资源都被处置完毕。`'close'` 必须在 `'end'` 之后发出或者干脆替代 `'end'`。

仅仅发出 `'close'`（不包含`'end'`）表明一个损坏的管道 - `Stream#pipe` 将调用 `dest.destroy()` 来通知对方销毁管道。

如果一个 `ReadableStream` 正常终止，那么 *必须不能* 在 `'end'` 之前发出`'close'`。

如果 error 是可以恢复的，那么 `Stream` *必须不能* 发出 error 事件（在 Node.js 的 stream spec 中没有提及这一点）。

所有类型的 stream 应该实现 `destroy`；`WritableStream` 则 *必须实现* `destroy`。

### emit('error')

所有类型的 streams 当遇到不可恢复的错误时，*必须* 发出 `'error'`。
如果 `Stream` 物理上不可能读或写，也要发出 `'error'`。

在 `end` 之后，对一个 `WriteableStream` 调用 `write`，则 *可能* 抛出异常（在正常用法下，不应该出现这种情况）；除此以外的情况，遇到不可恢复的 error，则 *不允许* 抛出异常（只能emit error）。

## WritableStream

一个 `WritableStream` 必须实现 `write`, `end`, 和 `destroy` 方法；并且其 `writable` 属性 *必须* 为 `true`，且 *必须* 从 `Stream#pipe` 派生。

```
                    ------------------
   pipe    write    |                |
 ------->  end      | WritableStream |
           destroy  |                |
                    ------------------
```

### write(data)

`write` 必须返回 `true` 或者 `false`。
(如果返回 `false`，则 writer *应该* 进入 pause 状态)。

如果在 end 之后调用 `write`，那么 *可以* 抛出异常。

如果 `write` 返回 `false`，那么它最终会 *一定会* 发出  `'drain'`。

`write` 返回 `false` 意味着 stream 被暂停了。

暂停意味着 stream(或其下游 stream) 没有能力处理后续数据，而 writer 或上游 stream *应该尝试* 降速或者停止。

Paused 不意味着所有数据都将被缓存，尽管采用缓存对于一个 Stream 的实现是合理的。

### end()

实现 `end` 方法 *必须* 设置 stream 的 `writable` 属性为 `false`。

如果 `Stream` 也是 readable 的，那么它 *必须* 最终发出 `'end'`，然后再发出 `'close'`。

如果 `Stream` 不是 readable 的，那么 `Stream` 最终 *必须* 发出 `'close'`, 而不会发出 `'end'`。

### destroy()

用来清理 `Stream` 的状态。

调用 `destroy` *必须* 处理好底层的资源状态。

调用 `destroy` 之后，清理完底层资源之后，最终 *必须* 发出 `'close'`

### emit ('drain')

在暂停之后，一个 `Stream` 最终 *必须* 发出 `'drain'`，用来通知上游`'resume'`。

```
                source.pipe(dest);

source.pause() <---------------- dest.write() === false
source.resume() <---------------- dest.emit('drain')
```

例如上图：当 `Stream#pipe` 调用 `dest.write()` 返回 `false` 的时候，`Stream#pipe` 将调用 `source.pause()` 以暂停读取。而当 dest 发出 `'drain'` 之后，`Stream#pipe` 将调用 `source.resume()` 以恢复读取。

如果没有正确地发出 `drain`，那么将可能无法获取后续的 `'data'` 事件（依赖于 source 对于 pause 的实现）。

## ReadableStream

一个 `ReadableStream` *must* 从 `Stream` 的 `pipe` 类派生，并且设置 `readable` 属性为 `true`，随后 *必须* 发出0或多个 `'data'` 事件。结束时，则发出 `'end'` 事件。一个 `ReadableStream` *可以* 实现自己的 `pause` 和 `resume` 方法。


* 在这里我不会回纠缠于 `pipe` 的行文模式，因为我只是试图记录你实现自己的 `Stream` 必须实现的事情，并且可以兼容 `pipe` 的行为。

### emit('data', data)

一个 `ReadableStream` 可能发出一个或多个 `'data'` 事件。
当 `ReadableStream` 发出 `'end'` 事件之后，不应该再发出 `'data'` 事件。

### emit('end')

当 `ReadableStream` 没有新的 `'data'` 事件要发出时，则 *应该* 发出 `'end'` 来通知 `Stream#pipe`。`'end'` 事件只能发出一次。

当`ReadableStream` 发出之后，可以设置 `readable` 属性为 `false`。

并且，一个 `Stream` 在发出 `'end'` 之后，内部也应该调用 `destroy` 来清理资源。

### emit ('close')

在 `'end'` 事件之后，`ReadableStream` 必须发出 `'close'` 事件。`'close'` *只能* 事件发出一次。如果 `destroy` 被调用，那么 `'close'` 必须被发出，除非 stream 已经被正常终止过了。如果在 `'end'` 之前发出了 `'close'` 那么意味着这是一个损坏的 stream，仅当 `destroy` 被调用后 *可能* 出现这种情况。

在发出 `'end'`之前发出 `'close'` 事件将导致 `pipe` 调用下游 stream 的 `destroy`。

### pause()

一个 readable `Stream` *可以* 实现 `pause` 方法（被 `Stream#pipe` 调用）。

当 `pause` 被调用后，stream 应该尝试减少发出 `'data'` 事件（也可能完全停止发出 `'data'`）。

可见，pause 之后是否发出 `'data'` 是可选的行为，pause 只是 `Stream#pipe` 对 readable stream 的建议。

### resume()

一个 `ReadableStream` *可以* 实现 `resume` 方法。

如果之前 `Stream` 处于 pause 状态，那么 resume 之后，可以以更频繁的频率来发送 `'data'` 或者恢复发出 `'data'`(如果之前处于完全停止状态)。

如果 stream 也是 writable 的，并且之前调用 `write` 返回 `false`，那么它最终 *必须* 发出 `drain`。


### destroy()

一个 `ReadableStream` *应该* 实现 `destroy`。

调用 `destroy` *必须* 清理好底层资源。

调用 `destroy` 之后，一旦清理完底层资源，最终 *必须* 发出 `'close'`。
