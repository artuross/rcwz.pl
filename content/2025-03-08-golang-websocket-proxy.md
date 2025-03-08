---
title: Proxying WebSocket connections in Go
date: 2025-03-08T03:00:00+02:00
---

Golang's `net/http` package provides built-in support for `http_proxy` and `https_proxy`.
This is awesome when developing apps locally, because one can use a tool like [Proxyman](https://proxyman.com/)
to watch the traffic.

In a side project, I needed to use a WebSocket client. I used [`github.com/gobwas/ws`](https://github.com/gobwas/ws)
and while proxy is not supported out of the box, it was very easy to add using [`golang.org/x/net/proxy`](https://pkg.go.dev/golang.org/x/net/proxy)
which supports [SOCKS5](https://en.wikipedia.org/wiki/SOCKS).

Here's the snippet:

```go
package main

import (
	"context"
	"fmt"

	"github.com/gobwas/ws"
	"golang.org/x/net/proxy"
)

func main() {
	dialer := ws.Dialer{
		NetDial: proxy.Dial,
	}

	conn, reader, _, err := dialer.Dial(context.Background(), "wss://example.org/ws"))
	if err != nil {
		fmt.Fatal("connect to WebSocket server", err)
	}

	// ...
}
```

Make sure to run your app with `ALL_PROXY` env var (`all_proxy` will work too):

```sh
ALL_PROXY=socks5://127.0.0.1:8889 go run main.go
```

Finally, you may need to enable SOCKS5 proxy in your proxy app. In Proxyman, it's disabled by default
and can be enabled by going into `Tools` › `Proxy Settings` › `SOCKS Proxy Settings...`. A popup
window will appear where you can toggle the proxy and configure the port.

If you don't want to pass an env var, `proxy.FromURL` is your friend.
