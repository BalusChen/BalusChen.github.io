---
title: golang-如何实现websocket协议
date: 2020-09-19 23:40:25
tags:
    - golang
    - websocket
---

# golang 如何实现 websocket 协议

最近一直在想着在 nginx 上实现 websocket 协议，但是却没有太多头绪，所以 clone 了一些 golang websocket 相关的 repo 下来学习。既是学习它对 websocket 协议包的处理，也希望对 golang 本身有更多的理解（包括对标准库的熟悉）。

## 一些不懂的地方

### golang 如何接管 HTTP 连接

websocket 是通过 HTTP 发起请求，并且带上一些特殊的请求头告诉 server 需要升级协议至 websocket，server 同意升级后回复相应的状态码和几个特殊的响应头，此后所有的数据都以 websocket 协议来进行解析。

所以我们需要一种机制，在第一次以 HTTP 协议接收请求并响应之后可以接管 TCP 连接，而后的所有数据由自己来解析，而不再通过 HTTP 协议。而 golang 也的确提供了这种能力，即 hijack（劫持）。

```go
// net/http/server.go
type Hijacker interface {
	Hijack() (net.Conn, *bufio.ReadWriter, error)
}
```

`Hijack`方法让调用者接管这个连接，此后 HTTP 库不再处理这个连接，而是由调用者来管理。该方法有三个返回值：

- `net.Conn`：即 TCP 连接，需要注意的是，这个连接的 read deadline 或者 write deadline，而实际上 HTTP 中设置的值在连接被接管之后就不能再影响这个连接了，所以调用者需要自己清理掉这些 deadline
- `*bufio.ReadWriter`：当连接被接管时，可能上层的HTTP连接已经读取/写入了一些数据

需要注意的是，不是所有版本的 HTTP 协议都支持 hijack，HTTP/1.x 是支持的，但是 HTTP/2 则不支持，所以在 hijack 之前需要判断该 HTTP 连接是否实现了`Hijacker`接口：

```go
func wsAccept(w http.ResponseWriter) {
  hj, ok := w.(http.Hijacker)
  if !ok {
    return
  }

  c, brw, err := hj.Hijack()
  ...
}
```

前面说过，由于 HTTP 底层使用的是 TCP 协议，数据是以字节流的形式传输的，所以在读完 HTTP header 的时候，可能 body 也读取了一部分，而这部分实际上是需要以 websocket 协议来处理的。通常我们在接管连接之前，只会把握手阶段的响应头发送给 client，而不会发送 body，所以`Hijack`方法返回的`*bufio.ReadWriter`通常只有 reader 有尚未处理的数据，这些数据我们是需要复用的：

```go
	...
	b, _ := brw.Reader.Peek(brw.Reader.Buffered())
	brw.Reader.Reset(io.MultiReader(bytes.NewReader(b), netConn))
```

这里将`Hijacker`返回的`*bufio.ReadWriter`中的 reader 部分重置为`io.MultiReader`，这种 reader 可以从一个或者多个 read source 读取数据，正好满足目前的要求：既需要从未被处理的 reader 中读取数据，又需要从网络连接中读取数据。

> client 发起 websocket 连接时，在还没有接收到 server 的 handshake response 之前也会发送数据么？感觉`Hijack`方法返回的`*bufio.ReadWriter`中的 reader 不会有未处理的数据？

## 参考

[What is Sec-WebSocket-Key for?](https://stackoverflow.com/questions/18265128/what-is-sec-websocket-key-for)

[A Million WebSockets and Go](https://www.freecodecamp.org/news/million-websockets-and-go-cc58418460bb/)
