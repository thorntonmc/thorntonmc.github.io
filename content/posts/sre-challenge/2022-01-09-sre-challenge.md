---
title: A Reddit SRE Challenge - Part 1
date: 2022-01-09
draft: false
categories: sre
tags: ["sre-challenge"]
---

## Introduction

There was a thread on the devops subreddit going around about a interview challenge for a junior SRE position. Considering I've been doing more work that might be classified under this role, I figured I'd give it a try. Lets take a look at the README.md, I've reworked it a bit for brevity, you can see the original on [github](https://github.com/thorntonmc/sre-challenge), along with all of the code described in this post:


> 1. Using Go, write a simple application, http server, which serves 2 endpoints:
>    *  `/requests`, which increments the hits in in redis and additionally returns `hello, world` to the caller 
>    * `“/queries”` will return the number of hits recorded in redis
> 
> 2. Build a helm chart of the http server you wrote. Explain which kind of k8s resource you picked and why? Add liveness and readiness Expose the service port to 3030 Install kind to test you application and install the helm chart
>
> 3. Build a github actions file that triggers a build on specific wildcard PR (*build*) and one merge to master and create tags, which builds your container, run unit tests, and saves to the registry with the correct version.

There were originally some other items which I won't be tackling at this time (I won't be terraforming to EKS or GCP, though I did use terraform as my original deployment)

## Writing the HTTP Server

Let's get started and write our HTTP server so we can get into the fun stuff, starting with our basic struct:

```go
type App struct {
	listenAddr string
	mux        *http.ServeMux
	rdb        *redis.Client
}
```

This is pretty straightforward, our `App` does three things: it listens on a desired address and port, routes paths via a `ServeMux`, and gets and sets our redis cache. The rest of this article will be implementing this functionality. 

### Config Options

While this is a fairly barebones API, there is some level of configuration we should pass; namely the redis address and password, and the port we should listen on. Let's define our struct:

```go
type Config struct {
	port          int
	redisAddress  string
	redisPassword string
}
```

In this example, we'll simply set the value from our enviornment variables, so our init function is quite simple

```go
func NewConfigFromEnv() Config {
	return newConfigFromEnv()
}

func newConfigFromEnv() Config {
	portOverride, ok := os.LookupEnv("APP_PORT")
	if ok {
		portOverride, err := strconv.Atoi(portOverride)
		if err != nil {
			log.Fatalf("Failed to parse port override")
		}
		port = portOverride
	}

	redisAddressOverride, ok := os.LookupEnv("REDIS_ADDRESS")
	if ok {
		redisAddress = redisAddressOverride
	}

	redisPortOverride, ok := os.LookupEnv("REDIS_PORT")
	if ok {
		log.Printf("have redisPortOverride %s", redisPortOverride)
		redisPortOverride, err := strconv.Atoi(redisPortOverride)
		if err != nil {
			log.Fatalf("Failed to parse redis port override")
		}
		redisPort = redisPortOverride
	}

	return Config{
		port:          port,
		redisAddress:  fmt.Sprintf("%s:%d", redisAddress, redisPort),
		redisPassword: redisPassword,
	}
}
```

### Creating the Redis Client

Our application needs a redis client so that it can get/set keys upon request, so let's create this in a new file `redis.go`

```go
func NewRedisClient(addr, password string) *redis.Client {
	log.Printf("configuring redis at %s", addr)
	rdb := redis.NewClient(&redis.Options{
		Addr:     addr,
		Password: password,
		DB:       0,
	})

	return rdb
}
```

We'll also want some functions to get/set our "hits" key, so let's add them too:

```go
func (a *App) setHits(val string) error {
	err := a.rdb.Set(context.Background(), "hits", val, 0).Err()
	if err != nil {
		return err
	}
	return nil
}

func (a *App) getHits() (string, error) {
	hits, err := a.rdb.Get(context.Background(), "hits").Result()

	if err != nil {
		return "", err
	}

	return hits, err
}
```

The requirement states that we need to either create the `hits` key or increment it each call to `/requests`, so we'll create that logic as well:


```go
func (a *App) updateHits() error {
	var hits string
	hits, err := a.getHits()

	if err == redis.Nil { // key does not exist, so set it to 1
		err := a.setHits("1")
		if err != nil {
			return err
		}
		return nil
	}

	if err != nil {
		return err
	}

	return a.incrementHits(hits) // increment the current value
}

func (a *App) incrementHits(hits string) error {
	hitsInt, err := strconv.Atoi(hits)

	if err != nil {
		return err
	}

	hitsInt += 1

	hits = strconv.Itoa(hitsInt)
	return = a.setHits(hits)
}

```





### Routing Requests

The last thing our application needs a `serveMux` in order to route our requests. We need to handle two routes `/requests` and `/queries` -- since these routes will utilize the redis connection created as a part of our application, we'll make them methods of our app:


```go
func (a *App) handleRequests(w http.ResponseWriter, r *http.Request) {
	err := a.updateHits()
	if err != nil {
		log.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte(err.Error()))
		return
	}

	w.WriteHeader(http.StatusOK)
	w.Write([]byte("Hello, World\n"))
}

func (a *App) handleQueries(w http.ResponseWriter, r *http.Request) {
	hits, err := a.getHits()

	if err == redis.Nil {
		w.Write([]byte("0\n"))
		w.WriteHeader(http.StatusOK)
		return
	}

	if err != nil {
		w.Write([]byte(err.Error()))
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	w.Write([]byte(hits))
	w.WriteHeader(http.StatusOK)
}
```

Now our app is ready to be used! Let's throw in an `init` method so that we can easily run it from our calling function, and we also need a simple method to actually start the http server:

```go
// NewApp returns a new bare application with a webserver configured and ready
func NewApp(c Config) *App {
	return newApp(c)
}

func newApp(c Config) *App {
	app := &App{}
	mux := http.NewServeMux()
	mux.HandleFunc("/requests", app.handleRequests)
	mux.HandleFunc("/queries", app.handleQueries)

	app.mux = mux
	app.rdb = NewRedisClient(c.redisAddress, c.redisPassword)
	app.listenAddr = fmt.Sprintf(":%d", port)

	return app

}

func (a *App) Serve() {
	log.Printf("Listening on %s", a.listenAddr)
	log.Fatal(http.ListenAndServe(a.listenAddr, a.mux))
}
```

Excellent! Now our app can be initialized, and the listener can be started simply by importing the package! This is the last missing piece of this puzzle. To run the application from the command line easily, I used the [cobra](https://github.com/spf13/cobra) package. The [usage readme](https://github.com/spf13/cobra/blob/master/cobra/README.md) and [user guide](https://github.com/spf13/cobra/blob/master/user_guide.md) are helpful with getting setup, but in a nutshell, cobra creates a `main.go` in your root directory and also creates `cmd/root.go`. `main.go` then imports `cmd/root.go` which contains all the information around how your program handles flags and commands. Ours is simple, as we can see:


`main.go`
```go
package main

import (
	"github.com/thorntonmc/sre-challenge/cmd"
)

func main() {
	cmd.Execute()
}
```

`cmd/root.go`
```go
import (
	"fmt"
	"os"

	app "github.com/thorntonmc/sre-challenge/internal"

	"github.com/spf13/cobra"

	"github.com/spf13/viper"
)

// This file also contains the scaffolding for your cli application, but what's important is what is in rootCmd
var rootCmd = &cobra.Command{
	Use:   "sre-challenge",
	Short: "Junior SRE Interview",
	Long:  "Junior SRE Interview",
	Run: func(cmd *cobra.Command, args []string) {
		c := app.NewConfigFromEnv()
		a := app.NewApp(c)
		a.Serve()
	},
}
```

In a nutshell, cobra will run what's inside `rootCmd` when the application is run with no parameters, which is what we want as there aren't any flags that are used in this program. The `Run` function is very simple, we simply initialize our config via the package we wrote earlier, then create a new application via `NewApp`, and then run our server. 

Now we're ready to test it out! Since we want to keep our a build, tests, and deploys idempotent, lets keep our habits in check by putting our build steps in a `Makefile`

`./Makefile`

```makefile
.DEFAULT_GOAL := build # default to the build target
MAKEFLAGS= -s .        # makefile is silent

BUILD_DIR=./build

fmt:
	go fmt ./

build: fmt
	go build  -o $(BUILD_DIR)/sre-challenge.go ./main.go

install: fmt build
	go install

clean: 
	rm -rf $(BUILD_DIR)

```

Cool! Now we when we want to build our application we can simply type `make build` and the binary will be created in `./build/sre-challenge.go` -- and since we've also set `build` to be our default goal, we can actually just type `make` to build our application.

The `install` target will also install our binary to `$GOPATH/bin`

```bash
michaelthornton@work:~/sre-challenge [main]
$ make install
michaelthornton@work:~/sre-challenge [main]
$ ls $GOPATH/bin
cobra         cr            go-outline    gopls         sre-challenge
```

So now we can run our app! Let's try it out

```bash
michaelthornton@work:~/sre-challenge [main]
$ sre-challenge
2022/01/11 09:56:45 configuring redis at redis.sre-challenge:6379
2022/01/11 09:56:45 Listening on :8080
```

This is great! However since we have no redis server installed - the server is basically useless. In our next post, we'll deploy this application and a redis cache to a local kubernetes cluster. 



