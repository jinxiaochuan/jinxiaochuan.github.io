---
title: NodeJS Stream
date: 2022-03-19 10:44:57
tags: NodeJS
---

> _Streams are Node’s best and most misunderstood idea._ _-- Dominic Tarr_

#### _Stream_ 是什么？

> _Streams are collections of data – just like arrays or strings. The difference is that streams might not be available all at once and they don’t have to fit in memory. This makes streams really powerful when working with large amounts of data, or data that’s coming from an external source one chunk at a time._
> 流是数据的集合，就像数组或字符串一样。不同之处在于，流可能不会一次全部可用，而且它们不必放入内存中。这使得流在处理大量数据或来自外部源的数据时非常强大，每次处理一个数据块。

Many of the built-in modules in Node implement the streaming interface:

|      _Readable Streams_       | _Writable Streams_             |
| :---------------------------: | :----------------------------- |
| HTTP response, on the client  | HTTP requests, on the client   |
| HTTP requests, on the server  | HTTP responses, on the server  |
|        fs read streams        | fs write streams               |
|         zlib streams          | zlib streams                   |
|        crypto streams         | crypto streams                 |
|          TCP sockets          | TCP sockets                    |
| child process stdout & stderr | child process stdout           |
|         process.stdin         | process.stdout, process.stderr |

注意 ⚠️：

1. _HTTP response_ 在客户端虽然是可读流，但是在服务端是可写流。这是因为在 HTTP 情况下，我们基本上是从一个对象(_HTTP.IncomingMessage_)读取数据，然后写入另一个对象(_HTTP.ServerResponse_) 。
2. 有些既是可读流又是可写流，比如 _TCP sockets_ 、 _zlib streams_ 、 _crypto streams_ 。

#### 从大文件请求窥探 _Stream_ 优势

> _创建一个大约 400 MB 的大文件_
>
> ```bash
> const fs = require("fs");
> const file = fs.createWriteStream("./big.file");
>
> for (let i = 0; i <= 1e6; i++) {
>  file.write(
>    "Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.\n"
>  );
> }
>
> file.end();
> ```
>
> _起个简单的服务以访问大文件_
>
> ```bash
> const fs = require("fs");
> const server = require("http").createServer();
>
> server.on("request", (req, res) => {
>  fs.readFile("./big.file", (err, data) => {
>    if (err) throw err;
>
>    res.end(data);
>  });
> });
>
> server.listen(8000);
> ```
>
> _对比电脑前后内存消耗_
>
> ![](https://jscomplete.com/images/reads/node-bb/picturenode5.png) >![](https://jscomplete.com/images/reads/node-bb/picturenode5.png)
>
> 内存消耗直接由 _8.7 MB_ 飙升至 _434.8 MB_ 。
> 在我们将内容写入响应对象之前，将文件存储在内存中，这是非常低效且消耗服务器内存的，尤其在大文件场景下尤为明显（_Node_ 是基于 _V8_ 构建的， 而 _V8_ 对于内存的使用有一定的限制。 在默认情况下， _64 位_ 的机器大概可以使用 _1.4G_ ， 而 _32 位_ 则为 _0.7G_ 的大小）。
>
> _引入主角 Stream_
>
> ```bash
> const fs = require("fs");
> const server = require("http").createServer();
>
> server.on("request", (req, res) => {
>  const src = fs.createReadStream("./big.file");
>  src.pipe(res);
> });
>
> server.listen(8000);
> ```
>
> ![](https://jscomplete.com/images/reads/node-bb/picturenode7.png)
>
> 内存使用量增加了大约 _25MB_ ，仅此而已
>
> 对于大文件的操作通常会使用 _Buffer_ ， 究其原因就是因为 _Node_ 中内存小的原因， 而使用 _Buffer_ 是不受这个限制， 它是堆外内存(可通过 `process.memoryUsage()` 查看内存分配)

#### Stream

##### Stream Type

There are four fundamental stream types in Node: _Readable_ , _Writable_ , _Duplex_ , and _Transform_ streams .

- _Readable_ - A readable stream is an abstraction for a source from which data can be consumed. An example of that is the `fs.createReadStream` method.
- _Writable_ - A writable stream is an abstraction for a destination to which data can be written. An example of that is the `fs.createWriteStream` method.
- _Duplex_ - A duplex stream is both Readable and Writable. An example of that is a `TCP socket`.
- _Transform_ - A transform stream is basically a duplex stream that can be used to modify or transform the data as it is written and read. An example of that is the `zlib.createGzip` stream to compress the data using gzip. You can think of a transform stream as a function where the input is the writable stream part and the output is readable stream part. You might also hear transform streams referred to as “`through streams`”

All streams are instances of `EventEmitter`. They emit events that can be used to read and write data. However, we can consume streams data in a simpler way using the pipe method.

##### Stream Pipe Method

> _The pipe method is the easiest way to consume streams._
> 管道方法是使用流的最简单方法。
>
> `切记：通常建议使用管道方法或使用带有事件的流，但避免将这两种方法混合使用。` > `即：当您使用管道方法时，您不需要使用事件，但是如果您需要以更自定义的方式使用流，那么事件就是最好的选择。`

```bash
# In this simple line, we’re piping the output of a readable stream – the source of data, as the input of a writable stream – the destination.
# The source has to be a readable stream and the destination has to be a writable one.
readableSrc.pipe(writableDest);
```

```bash
# they can both be duplex/transform streams as well. For streams readableSrc (readable), transformStream1 and transformStream2 (duplex/transform), and finalWritableDest (writable).
readableSrc
  .pipe(transformStream1)
  .pipe(transformStream2)
  .pipe(finalWritableDest);
```

##### Stream Events

`Pipe` 除了从可读流中读取数据并写入可写流中之外，内部自动处理了很多事件。比如 `errors`, `end-of-files`, and `the cases when one stream is slower or faster than the other`.

以下两种方式是等效的

```bash
# pipe
readable.pipe(writable)

# event listener
readable.on("data", chunk => {
  writable.write(chunk);
});

readable.on("end", () => {
  writable.end();
});
```

|         | _Readable Streams_                                              | _Writable Streams_                        |
| :-----: | :-------------------------------------------------------------- | :---------------------------------------- |
| Events  | data, end, error, close, readable                               | drain, finish, error, close, pipe, unpipe |
| Methods | pipe(), unpipe(), wrap(), destroy()                             | write(), destroy(), end()                 |
|         | read(), unshift(), resume(), pause(), isPaused(), setEncoding() | cork(), uncork(), setDefaultEncoding()    |

> The most important events on a `readable stream` are:
>
> - The `data` event, which is emitted whenever the stream passes a chunk of data to the consumer
>
> - The `end` event, which is emitted when there is no more data to be consumed from the stream.
>
> The most important events on a `writable stream` are:
>
> - The `drain` event, which is a signal that the writable stream can receive more data.
>
> - The `finish` event, which is emitted when all data has been flushed to the underlying system.

#### Paused and Flowing Modes

> Readable streams have two main modes that affect the way we can consume them:
>
> - They can be either in the `paused` mode
> - Or in the `flowing` mode
>
> Those modes are sometimes referred to as `pull` and `push` modes.
> 这两种模式有时被称为 `拉模式` 和 `推模式` 。

默认情况下，所有可读流都以 `paused mode` 启动，但在需要时，它们可以轻松切换为 `flowing mode`，同时在必要时也可切换回 `paused mode`。

> _When a readable stream is in the paused mode, we can use the read() method to read from the stream on demand. However, for a readable stream in the flowing mode, the data is continuously flowing and we have to listen to events to consume it._
> 当可读流处于暂停模式时，我们可以使用 `read()` 方法按需读取流。然而，对于流动模式下的可读流，数据是连续流动的，我们必须监听事件才能使用它。
>
> _In the flowing mode, data can actually be lost if no consumers are available to handle it. This is why when we have a readable stream in flowing mode, we need a data event handler. In fact, just adding a data event handler switches a paused stream into flowing mode and removing the data event handler switches the stream back to paused mode. Some of this is done for backward compatibility with the older Node streams interface._
> 在流动模式下，如果没有消费者来处理数据，数据实际上可能会丢失。这就是为什么当我们有一个流动模式的可读流时，我们需要一个数据事件处理程序。事实上，只需添加数据事件处理程序，即可将暂停的流切换到流动模式，删除数据事件处理程序，即可将流切换回暂停模式。其中一些是为了向后兼容旧的节点流接口。

> _To manually switch between these two stream modes, you can use the resume() and pause() methods._
> 要在这两种流模式之间手动切换，可以使用 `resume()` 和 `pause()` 方法
>
> _When consuming readable streams using the pipe method, we don’t have to worry about these modes as pipe manages them automatically._
> 当使用 `pipe` 方法消费可读流时，我们不必担心这些模式，因为 `pipe` 会自动管理它们。

#### Implementing Streams

##### Implementing a Writable Stream

```bash
const { Writable } = require("stream");

const outStream = new Writable({
  # This write method takes three arguments.
  # The chunk is usually a buffer unless we configure the stream differently.
  # The encoding argument is needed in that case, but we can usually ignore it.
  # The callback is a function that we need to call after we’re done processing the data chunk. It’s what signals whether the write was successful or not. To signal a failure, call the callback with an error object.
  write(chunk, encoding, callback) {
    console.log(chunk.toString());
    callback();
  }
});

process.stdin.pipe(outStream);
```

##### Implement a Readable Stream

```bash
const { Readable } = require("stream");

const inStream = new Readable();

inStream.push("ABCDEFGHIJKLM");
inStream.push("NOPQRSTUVWXYZ");

inStream.push(null); // No more data

inStream.pipe(process.stdout);
```

```bash
const inStream = new Readable({
  read(size) {
    this.push(String.fromCharCode(this.currentCharCode++));
    if (this.currentCharCode > 90) {
      this.push(null);
    }
  }
});

inStream.currentCharCode = 65;

inStream.pipe(process.stdout);
```

##### Implementing Duplex Streams

```bash
const { Duplex } = require("stream");

const inoutStream = new Duplex({
  write(chunk, encoding, callback) {
    console.log(`duplex stream write: ${chunk}`);
    callback();
  },

  read(size) {
    this.push(String.fromCharCode(this.currentCharCode++));
    if (this.currentCharCode > 90) {
      this.push(null);
    }
  }
});

inoutStream.currentCharCode = 65;

process.stdin.pipe(inoutStream).pipe(process.stdout);
```

##### Implementing Transform Streams

```bash
const { Transform } = require("stream");

const upperCaseTr = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

process.stdin.pipe(upperCaseTr).pipe(process.stdout);
```

##### Streams Object Mode

> By default, streams expect `Buffer/String` values. There is an `objectMode` flag that we can set to have the stream accept any `JavaScript object`.

```bash
# Here’s a simple example to demonstrate that. The following combination of transform streams makes a feature to map a string of comma-separated values into a JavaScript object. So "a,b,c,d" becomes {a: b, c: d}

const { Transform } = require("stream");

const commaSplitter = new Transform({
  readableObjectMode: true,

  transform(chunk, encoding, callback) {
    this.push(
      chunk
        .toString()
        .trim()
        .split(",")
    );
    callback();
  }
});

const arrayToObject = new Transform({
  readableObjectMode: true,
  writableObjectMode: true,
  transform(chunk, encoding, callback) {
    const obj = {};
    for (let i = 0; i < chunk.length; i += 2) {
      obj[chunk[i]] = chunk[i + 1];
    }
    this.push(obj);
    callback();
  }
});

const objectToString = new Transform({
  writableObjectMode: true,
  transform(chunk, encoding, callback) {
    this.push(JSON.stringify(chunk) + "\n");
    callback();
  }
});

process.stdin
  .pipe(commaSplitter)
  .pipe(arrayToObject)
  .pipe(objectToString)
  .pipe(process.stdout);
```

#### 参考

[Node’s Streams](https://jscomplete.com/learn/node-beyond-basics/node-streams)
[想学 Node.js，stream 先有必要搞清楚](https://juejin.cn/post/6844903891083984910)
[Node.js 内存溢出时如何处理？](https://juejin.cn/post/6844904017546444808)
[Node - 内存管理和垃圾回收](https://juejin.cn/post/6844903837912973326)
[Node.js Writable Stream 的实现简析](https://juejin.cn/post/6844903582089609224)
