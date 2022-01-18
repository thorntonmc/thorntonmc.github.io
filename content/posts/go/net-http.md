---
title: Reading the Go Standard Library - net/http
date: 2022-01-17T00:00:00+
draft: false
categories: go
tags: ["go", "go-stdlib", "misc"]
---

A piece of advice I [often](https://patshaughnessy.net/2021/10/23/to-learn-a-new-language-read-its-standard-library) [see](https://www.reddit.com/r/golang/comments/evrdds/can_someone_suggest_a_way_to_learn_gos_standard/) when learning a new language is to read the standard library. So I'm taking a tour through [go's stdlib](https://pkg.go.dev/std) in order to get a more comprehensive understanding of the library. 

I've always liked the site [Learn X in Y Minutes](https://learnxinyminutes.com), and decided as I went through this to write out functions and code in a similar manner. The blog post has split out the comments into Markdown, but if you just wish to see the code, you can do so on [github](https://github.com/thorntonmc/go-practice/blob/main/concepts/net/http/main.go).

Much of this is heavily curated and toured by [Jon Bodner](https://medium.com/@jon_43067) via his book [Learning Go](https://www.oreilly.com/library/view/learning-go/9781492077206/). 


## Getting Started

The [net/http](https://pkg.go.dev/net/http) reads absolutely beautifully, and the introduction is no different:

> Package http provides HTTP client and server implementations.

Perfect! Lets dive in by importing some necessary functions from the stdlib, as well as a third party package [alice](https://github.com/justinas/alice) 

```go
// main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/justinas/alice"
)
```

## The Client

The client makes HTTP requests, and receives HTTP responses. Let’s take a look at the client definition:

```go
type Client struct {
	Transport RoundTripper
	CheckRedirect func(req *Request, via []*Request) error
	Jar CookieJar
	Timeout time.Duration
}
```

There is a [default client instance](https://cs.opensource.google/go/go/+/refs/tags/go1.17.6:src/net/http/client.go;l=110) included, providing an empty `&http.Client{}`, but you should always use your own, as it has no timeout. 

```go
var dontUseDefault = http.DefaultClient
```

Once you've created your http.Client, you don't need another one. It's designed to properly handle simultaneous requests.

```go
func newClient() *http.Client {
	client := &http.Client{
		Timeout: 30 * time.Second,
	}

	return client
}
```

When you want to make a request, you create a new \*http.Request instance with the http.NewRequestWithContext function

```go
func newReq() *http.Request {
	req, err := http.NewRequestWithContext(context.Background(), http.MethodGet, "https://jsonplaceholder.typicode.com/todos/1", nil)
	if err != nil {
		panic(err)
	}
	return req
}
```

Once you have an `*http.Request` instance, you can set any headers via the `Headers()` method of the instance. Once you're done, you can use the `Do()` method on the `*http.Client` with your `http.Request` which returns an `http.Response`

```go
func makeReq() *http.Response {
	c := newClient()
	req := newReq()
	req.Header.Add("X-My-Client", "Learning Go")
	resp, err := c.Do(req)
	if err != nil {
		panic(err)
	}

	return resp
}
```

The real work is being done in `http.Client.Do()` - while there are methods included in the `http.Client` interface such as `Get` and `Post`, those force the usage of `http.NewRequest`, hindering your ability to pass a `context.Context` via `http.NewRequestWithContext`.

The `*http.Response` has several fields with information on the request:
1. The status code
2. The text response of the status code
3. The response headers. 

The [`headers`](https://pkg.go.dev/net/http#Header) type is a `map[string][]string` with methods `Get()` and `Set()` to respectively get and set HTTP headers.

```go
func handleResponse(r *http.Response) {
	code := r.StatusCode // e.g. 200
	codeText := r.Status // e.g. "200 OK"
	headers := r.Header

	contentType := headers.Get("Content-Type") 

	fmt.Printf("%d\n%s\n%s", code, codeText, contentType)
}
```

Response bodies can be used json.Decoder to process REST API responses

```go
func parseJson(r *http.Response) {
	// the response body is an io.ReadCloser, which means it can be used to parse json
	body := r.Body

	var data struct {
		UserID   int    `json:"userId"`
		ID       int    `json:"id"`
		Title    string `json:"title"`
		Complete bool   `json:"completed"`
	}

	err := json.NewDecoder(body).Decode(&data)
	if err != nil {
		panic(err)
	}
}
```

## The Server

The HTTP server is built around the `http.Server` and the http.Handler interfaces. `http.Server` listens for HTTP requests. Requests to the server are handled by implementations of `http.Handler`, which have a single method ServeHTTP

Since `http.Handler`s are so important, lets have a look at the [interface definition](https://pkg.go.dev/net/http?utm_source=gopls#Handler)

```go
 	type Handler interface {
		ServeHTTP(ResponseWriter, *Request)
	}
```

ServeHTTP takes two arguments - we've already looked at `http.Request`, so lets take a look at [`http.ResponseWriter`](https://pkg.go.dev/net/http?utm_source=gopls#ResponseWriter)

```go
	type ResponseWriter interface {
		Header() Header
		Write([]byte) (int, error)
		WriteHeader(statusCode int)
	}
```

Lets jump in define our own handler to explore what these methods do. 

```go
type NewHandler struct{}

func (n NewHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// the methods in http.ResponseWriter must be called in a certain order
	// the first, Header(), gives you an instnace of http.Header so you can set
	// any headers you need.
	// If you don't need it, don't call it

	// Next, call WriteHeader() with the HTTP status code for your response
	w.WriteHeader(http.StatusAccepted) // if you are sending a 200, you can skip it

	// Write() sets the body for the response
	w.Write([]byte("Hello, world!\n"))
}
```

Now that we have our server, we can make our handler

```go
func newServer() http.Server {
	s := http.Server{
		Addr:         ":8000",           // TCP address to listen. host:port - if not provided, listen on all hosts on port 80
		ReadTimeout:  30 * time.Second,  // Time to wait to read request headers
		WriteTimeout: 30 * time.Second,  // Time to wait for the write of the response
		IdleTimeout:  120 * time.Second, // Time to wait for the next request when keep-alives are enabled
		Handler:      NewHandler{},      // We invoke our handler when we get a request to our server
	}
	return s
}
```

The `ListenAndServeMethod` starts the the HTTP server

```go
func listen(s http.Server) {
	err := s.ListenAndServe()
	if err != nil {
		panic(err)
	}
}
```

#### The ServeMux

The main problem with our server is that it only handles one path, thankfully `*http.ServeMux` meets the http.Handler interface, and also serves as a router - sending the requests to the correct http.Handler instance, Servemux instances are the most common way of implementing multiple handlers and are critical to any REST API or HTTP server.

You can look at the details of how http.ServeMux does this in its  [ServeHTTP method](https://cs.opensource.google/go/go/+/refs/tags/go1.17.6:src/net/http/server.go;l=2416)
and its [Handler] method https://cs.opensource.google/go/go/+/refs/tags/go1.17.6:src/net/http/server.go;l=2361 but, essentially, http.ServeMux implements `ServeHTTP(w http.ResponseWriter, r *http.Request)`, satisfying the `http.Handler` interface, so it can be passed in as a Handler to `http.Server`. It’s `ServeHTTP` implementation calls the Handler, which parses the path and returns the `http.Handler` used for the request, and then calls that handlers `ServeHTTP` method.

```go
// That sounded a bit confusing, so let's test this out
func newServeMux() {
	m := http.NewServeMux() // creates a blank *http.ServeMux

	// create two handlers, an alive and ready path
	m.HandleFunc("/alive", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("ok!\n"))
	})
	m.HandleFunc("ready", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("always\n"))
	})

	s := http.Server{
		Addr:         ":8000",
		ReadTimeout:  30 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  120 * time.Second,
		Handler:      m, // Our handler now handles two requests, /alive and /ready
	}

	s.ListenAndServe()
}
```

`http.ServeMux` instances, since they themselves route requests to `http.Handler` instances, and since `http.ServeMux` is itself an instance of a `http.Handler` interface, an `http.ServeMux` can handle other `http.ServeMux instances`. This lets us route nested paths:

```go
func parentChildMux() {
	user := http.NewServeMux()
	user.HandleFunc("/fetch", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("This is a user!"))
	})

	record := http.NewServeMux()
	record.HandleFunc("/fetch", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("This is a record!"))
	})

	mux := http.NewServeMux()

	// mux will handle both the user and record path.
	// http.StripPrefix removes the part of the path that's already been processed,
	// as the previous handles don't expect the first path
	mux.Handle("/user/", http.StripPrefix("/user", user))
	mux.Handle("/record", http.StripPrefix("/record", record))
}
```

#### A small aside, the HandlerFunc

In previous examples, you saw implementations of a `http.ServeMux` that served functions directly, like this:

```go
func handlerFunc() {
	m := http.NewServeMux()
	m.HandleFunc("/path", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello, world!"))
	})
```


The `http.HandlerFunc` is a function that is also a `http.Handler`. In this case, I think the [type definition](https://cs.opensource.google/go/go/+/refs/tags/go1.17.6:src/net/http/server.go;l=2043) is more useful then a description:

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

Now look at the [definition](https://cs.opensource.google/go/go/+/refs/tags/go1.17.6:src/net/http/server.go;drc=refs%2Ftags%2Fgo1.17.6;bpv=0;bpt=1;l=2080) of `http.ServeMux.HandleFunc`


```go
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

It's more clear what's happening having looked at these two definitions. `http.ServeMux.HandleFunc` takes a path, and a `func(w http.ResponseWriter, r *http.Request)` and typecasts the function to a `HandlerFunc`. Since `HandlerFunc` implements `ServeHTTP`, it's a valid `http.Handler` interface, and can be used in a call to `mux.Handle`


#### One last pitfall

The http library offers functions that work with the package instance of `*http.ServeMux`, declared as:

```go
var dontUseDefaultMux = http.DefaultServeMux
```

These functions are `http.ListenAndServe`, `http.HandleFunc`, and `http.ListenAndServeTLS`. `http.ListenAndServe` and `http.ListenAndServeTLS` both serve with the default http server, which as mentioned does not contain properties such as timeouts.

```go
func dontUselistenAndServeDefault() {
	err := http.ListenAndServe(":80", nil)
	if err != nil {
		panic(err)
	}

	err = http.ListenAndServeTLS(":80", "", "", nil)
}
```

Using these methods can allow third party packages that have added additional handlers to `http.DefaultServeMux` to inject vulnerabilities

## Middleware

There is no middleware type in go, just a pattern involving http.Handler instances. Here is our example scenario:

We want to implement a path `/hello` which returns `Hello, world!`. This sounds simple enough, we simply need a `http.ServeMux` with the path `/hello` and a handler that writes out our message. However, we want an additional piece of logic to run before responding to our request:

  * We want a check to make sure the user has the secret password in header `X-Secret-Password` before hitting our `/hello` handler
  * We want a timer that logs the length of time it takes to read and respond to the request. 

We can define our two middleware functions like this:


`RequestTimer` takes a `http.Handler` variable `h`, and returns in itself a `http.Handler`. What is this handler? It starts a timer, then runs `h`, our original handler via `h.ServeHTTP`, and then logs the length of time it took to complete the function.
```go
func RequestTimer(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		h.ServeHTTP(w, r)
		end := time.Now()
		log.Printf("request time for %ss: %v", r.URL.Path, end.Sub(start))
	})
}
```

`TerribleSecurityProvider` is slightly more complicated. It takes a `password` string and returns a function - but what does that function do? That function takes a `http.Handler` and returns a `http.Handler`, which reads our `X-Secret-Password` header, and checks to see if its equal to the password when `TerribleSecurityProvider` was initially called. If the values, match, it runs `ServeHTTP` on the handler that was passed in.

```go
var securityMsg = []byte("You didn't give the secret password\n")

func TerribleSecurityProvider(password string) func(http.Handler) http.Handler {
	return func(h http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.Header.Get("X-Secret-Password") != password {
				w.WriteHeader(http.StatusUnauthorized)
				w.Write(securityMsg)
				return
			}
			h.ServeHTTP(w, r)
		})
	}
}
```

In a nutshell, here's how we use `ts`
```go
ts := TerribleSecurityProvider("password")
handler := ts(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, world\n"))
}))
```

Now for the implementation! Our mux handles the `/hello` path with `TerribleSecurityProvider`, instantiated as `ts`. As `ts` is itself a function that takes a handler, we can then pass in `RequestTimer` to it.

```go
// If ts passes its security check, it runs h.ServeHTTP, in this case RequestTimer.
// If not, it returns, ending the request and breaking the chain.
//
// If it passes, it calls RequestTimer which itself takes a handler that writes "Hello, World"
// RequestTimer starts timing the request, and then runs h.ServeHTTP, ultimately serving "Hello, world\n"
// on path /hello
func muxWithMiddleware() {
	ts := TerribleSecurityProvider("PASSWORD")
	mux := http.NewServeMux()

	mux.Handle("/hello", ts(RequestTimer(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("Hello, world!\n"))
		}))))
}
```

## Third party packages:

Perhaps it’s just me, but chaining middleware can be a confusing pattern. [Alice](https://justinas.org/alice-painless-middleware-chaining-for-go) makes chaining middleware as simple as:
```go
func helloWorldHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello, world!"))
}

func timeoutHandler(h http.Handler) http.Handler {
	return http.TimeoutHandler(h, 1*time.Second, "timed out")
}

func aliceWithMiddleWare() {
	ts := TerribleSecurityProvider("PASSWORD")
	handler := http.HandlerFunc(helloWorldHandler)

	a := alice.New(ts, timeoutHandler).Then(handler)

	s := http.Server{
		Addr:         ":8000",
		ReadTimeout:  30 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  120 * time.Second,
		Handler:      a,
	}

	s.ListenAndServe()
}
```

[Gorilla](https://github.com/gorilla) is too big to cover here but in a nutshell it has a [mux package](https://github.com/gorilla/mux) which allows easy use of creating dynamic paths such as `/user/{user_id}/name`

## Wrapping Up

Honestly, reading the go stdlib is an incredibly fun, and informative process. Next stop, [`io`](https://pkg.go.dev/io)
