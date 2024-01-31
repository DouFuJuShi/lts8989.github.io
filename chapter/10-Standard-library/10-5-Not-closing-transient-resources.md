## 10.5 瞬态资源一定要关闭

开发人员经常使用瞬态资源，这些资源必须在代码中的某个点关闭以避免内存泄漏，例如，在磁盘或内存中。结构通常可以实现 `io.Closer` 接口来传达必须关闭瞬态资源。让我们深入研究三个常见示例，说明资源未正确关闭时会发生什么以及如何正确处理它们。

### 10.5.1 HTTP body

首先，让我们在 HTTP 的上下文中讨论这个问题。

我们将编写一个 `getBody` 方法，该方法发出一个 HTTP GET 请求并返回 HTTP body 响应。这是第一个实现：

```go
type handler struct {
    client http.Client
    url    string
}

func (h handler) getBody() (string, error) {
    resp, err := h.client.Get(h.url)
    if err != nil {
        return "", err
    }

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }

    return string(body), nil
}
```

我们使用 `http.Get` 并使用 `io.ReadAll` 解析响应，看起来不错，并且它正确返回了 HTTP 响应 body。而且，存在资源泄漏。让我们了解在哪里泄漏。

`resp` 是一个 `*http.Response` 类型。它包含一个 `Body io.ReadCloser` 字段（`io.ReadCloser`实现了 `io.Reader` 和 `io.Closer`）。如果 `http.Get` 不返回错误，则必须关闭此 body；否则，这是资源泄漏。它将保留一些分配的内存，这些内存不再需要但不能被 GC 回收，并且可能会阻止客户端在最坏的情况下重用 TCP 连接。

处理 body 闭包最方便的方法是将其作为 defer 语句处理：

```go
defer func() {
    err := resp.Body.Close()
    if err != nil {
        log.Printf("failed to close response: %v\n", err)
    }
}()
```

在这个实现中，我们将 body 资源闭包作为一个 defer 函数正确处理，一旦 `getBody` 返回就会执行该函数。

> **Note** 在服务器端，在实现 HTTP 处理程序时，不需要关闭请求正文，因为它是由服务器自动完成的。

我们还应该了解，无论响应 body 是否被我们读取，它都必须关闭。例如，如果我们只对 HTTP 状态码感兴趣而不对正文感兴趣，那么无论如何都必须将其关闭以避免泄漏：

```go
func (h handler) getStatusCode(body io.Reader) (int, error) {
    resp, err := h.client.Post(h.url, "application/json", body)
    if err != nil {
        return 0, err
    }

    defer func() {
        err := resp.Body.Close()
        if err != nil {
            log.Printf("failed to close response: %v\n", err)
        }
    }()

    return resp.StatusCode, nil
}
```

即使我们还没有读取它，这个函数也会关闭 body。

另一件需要记住的重要事情是，根据我们是否关闭 body，是否有读取它，会有不同的行为：

* 如果我们在没有读取的情况下关闭 body，默认的 HTTP 传输可能会关闭连接
* 如果我们在读取后关闭 body，默认的 HTTP 传输不会关闭连接；因此，它可以被重复使用

因此，如果 `getStatusCode` 被重复调用并且我们想要利用 keep‑alive 连接，即使我们对 body 不感兴趣，我们仍然应该读取它：

```go
func (h handler) getStatusCode(body io.Reader) (int, error) {
    resp, err := h.client.Post(h.url, "application/json", body)
    if err != nil {
        return 0, err
    }

    // Close response body

    _, _ = io.Copy(io.Discard, resp.Body)

    return resp.StatusCode, nil
}
```

在此示例中，我们读取正文以保持连接处于活动状态。请注意，我们没有使用 `io.ReadAll`，而是使用 `io.Copy` 复制 body 到 `io.Discard`，一个 `io.Writer` 实现。此代码读取 body 但不带任何副本将其丢弃，使其比 `io.ReadAll` 更高效。

> **Note** 如果响应不为空，而不是错误为 nil，则关闭主体的实现并不少见：

```go
resp, err := http.Get(url)
if resp != nil {
    defer resp.Body.Close()
}

if err != nil {
    return "", err
}
```

> 此实现不是必需的。这是基于在某些情况下（例如，重定向失败），`resp` 和 `err` 都不是 nil 的事实。依据 [官方 Go 文档](https://github.com/golang/go/blob/master/src/net/http/client.go#L565-L567)：
> 
> 出错时，可以忽略任何响应。仅当 CheckRedirect 失败时才会发送具有非空 error 的非空 Response，即使这样，返回的 Response.Body 也已经关闭。
> 
> 因此，不需要 `if resp != nil {}` 检查。我们应该坚持最初的解决方案，只有在没有错误的情况下才在 defer 函数中关闭 body。

关闭资源以避免泄漏不仅仅与 HTTP body 管理有关。一般来说，所有实现 `io.Closer` 接口的结构都应该在某个时候关闭。此接口包含一个 `Close` 方法：

```go
type Closer interface {
    Close() error
}
```

现在让我们看看 `sql.Rows` 的影响。

### 10.5.2 `sql.Rows`

`sql.Rows` 是用作 SQL 查询结果的结构。由于此结构实现了 `io.Closer`，因此必须将其关闭。以下示例省略了关闭行：

```go
db, err := sql.Open("postgres", dataSourceName)
if err != nil {
    return err
}

rows, err := db.Query("SELECT * FROM CUSTOMERS")
if err != nil {
    return err
}

// Use rows

return nil
```

忘记关闭行意味着连接泄漏，防止数据库连接未被放回连接池。

我们可以在 `if err != nil` 块之后将闭包作为 defer 函数处理：

```go
// Open connection

rows, err := db.Query("SELECT * FROM CUSTOMERS")
if err != nil {
    return err
}

defer func() {
    if err := rows.Close(); err != nil {
        log.Printf("failed to close rows: %v\n", err)
    }
}()

// Use rows
```

在 `Query` 调用之后，如果没有返回错误，我们最终应该关闭 `rows` 以防止连接泄漏。

> **Note** 如上一节所述，`db` 变量（`*sql.DB` 类型）表示一个连接池。它还实现了 `io.Closer` 接口。然而，正如文档所建议的那样，关闭 `sql.DB` 是很少见的，因为它应该是长期存在的并且在许多 goroutine 之间共享。

现在让我们讨论在处理文件时关闭资源。

### 10.5.3 `os.File`

`os.File` 代表一个打开的文件描述符。与 `sql.Rows` 一样，它最终必须关闭：

```go
f, err := os.OpenFile(filename, os.O_APPEND|os.O_WRONLY, os.ModeAppend)
if err != nil {
    return err
}

defer func() {
    if err := f.Close(); err != nil {
        log.Printf("failed to close file: %v\n", err)
    }
}()
```

在此示例中，我们使用 `defer` 来推迟对 `Close` 方法的调用。 如果我们最终不关闭 `os.File`，它本身不会导致泄漏。实际上，当 `os.File` 被垃圾回收时，该文件将自动关闭。但是，最好显式调用 `Close`，因为我们不知道下一次 GC 何时触发（除非我们手动运行它）。

显式调用 `Close` 还有另一个好处：当我们想要监控主动返回的错误时。例如，这应该是可写文件的情况。

我们应该知道写入文件描述符不是同步操作。出于性能考虑，数据被缓冲。`close(2)` 的 BSD 手册页提到闭包会导致先前未提交的写入（仍然存在于缓冲区中）遇到 I/O 错误时出错。出于这个原因，如果我们想写入文件，我们应该返回关闭文件时发生的任何错误：

```go
func writeToFile(filename string, content []byte) (err error) {
    // Open file

    defer func() {
        closeErr := f.Close()
        if err == nil {
            err = closeErr
        }
    }()

    _, err = f.Write(content)
    return
}
```

在此示例中，我们使用了命名的返回值参数，并在写入成功时将错误设置为 `f.Close` 的响应。这样，客户就会意识到这个功能出了问题，并可以做出相应的反应。

此外，在关闭可写 `os.File` 时成功并不能保证将文件写入磁盘。 事实上，写入仍然可以存在于文件系统的缓冲区中，而不是刷新到磁盘上。如果持久性是一个关键因素，我们可以使用 `Sync()` 方法来提交更改。在这种情况下，可以安全地忽略来自 `Close` 的错误：

```go
func writeToFile(filename string, content []byte) error {
    // Open file

    defer func() {
        _ = f.Close()
    }()
    
    _, err = f.Write(content)
    if err != nil {
        return err
    }
    
    return f.Sync()
}
```

这个例子是一个同步写入函数。它确保在返回之前将内容写入磁盘。然而，对应的是对性能的影响。

总结本节，我们已经看到关闭临时资源以免导致泄漏是多么重要。临时资源必须在正确的时间和特定的情况下关闭。有时，对于必须关闭的内容并不总是一目了然。我们只能通过仔细阅读 API 文档和/或经验来获得这些知识。然而，我们应该记住，如果一个结构体实现了`io.Closer` 接口，我们最终必须调用 `Close` 方法。最后但并非最不重要的一点是，如果闭包失败，我们应该做什么也很重要：记录消息是否足够，还是应该返回它？适当的操作取决于实现，如本节中的三个示例所示。

现在让我们切换到与 HTTP 处理相关的常见错误：忘记返回语句。
