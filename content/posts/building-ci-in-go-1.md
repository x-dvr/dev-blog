---
author: ["Denis Rodin"]
title: "Build your own CI tool in Go (Part 1)"
date: "2024-04-29"
# description: "Build your own CI tool in Golang"
# summary: "First article in a series of many, which describes my steps in a journey to build my own CI tool from scratch in Golang"
tags: ["Go"]
categories: ["Build your own X", "Golang"]
series: ["Build your own CI"]
ShowToc: true
TocOpen: false
draft: true
---

## Intro  
  
Welcome! Within this series of blog posts we will go step by step through the process of building your own simple CI system from scratch in Golang. But what exactly are we going to build? We will start small with a simple one-binary-server solution to execute our CI workload, and in addition to it, a small CLI tool to do the same on local machine of the developer. In later posts we will extend and improve it. Here needs to be a disclaimer: It will not be in any shape or form a "Jenkins killer" (at least not from the beginning 🙂). But it should be able at least to build itself from source and provide us an executable. 

Let's **Go**!  

## Architecture  

We need to visualize our first destination point. Simple HTTP server, which receives REST API call with the link to remote repository, clones it to temporary directory, finds a file with CI pipeline definition, executes all steps of the pipeline (we will start with testing and compiling) and then copies executable to predefined output directory.

Simplified architecture of our future CI server:
![CI Server Architecture](/images/ci-srv-arch.png)

## Starting with the server

In this blog series I assume you already have working Go toolchain on your dev machine. At the time of writing current stable Go version is 1.22.2. Let's start our new project:
```bash
mkdir flow-ci && cd flow-ci
go mod init github.com/flow-ci/flow-ci
```
You can always found all code in my [repo](https://github.com/flow-ci/flow-ci) in the branch `blog`. I tried to do meaningful commits so you can follow the whole process one commit at a time.

### Installing fiber framework

For our HTTP server we will be using Fiber framework. You can install it using the command:

```bash
go get github.com/gofiber/fiber/v2
```

New we will need an entry point for our HTTP server. For that we will create file `cmd/web/main.go` with the following content:
```go {linenos=true}
package main

import "github.com/gofiber/fiber/v2"

func main() {
	app := fiber.New()

	app.Get("/", func(c *fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	app.Listen(":3000")
}
```

To test that everything is setup as expected let's run our server:
```bash
go run cmd/web/main.go
```
Now if we navigate our browser to `http://127.0.0.1:3000/` we should see the following text:
```
Hello, World!
```

To automate/simplify execution of frequent commands let's create the `Makefile`:
```Makefile
WEB_APP = web
BUILD_DIR = $(PWD)/bin

.PHONY: test bench run-web

web:
	@go build -o $(BUILD_DIR)/$(WEB_APP) cmd/$(WEB_APP)/*

run-web: web
	@$(BUILD_DIR)/$(WEB_APP)

test:
	@go test -v ./...

bench:
	@go test -bench=. -benchmem ./...

```
Now we can compile our HTTP server running the command: `make web`, and we can start our http server with the command `make run-web`. Apart from that we can execute all test using `make test` command, and run all benchmarks using `make bench`.

### Project structure

As we discussed, we will start with very simple implementation, which reacts to a single POST request, with the git repo URL in the payload. Despite that let's setup proper project structure for it:

```
 /
 ├┬ cmd
 │└┬ web
 │ ├─ handlers.pipelines.go
 │ └─ main.go
 ├─ internal
 └─ pkg
```
What do we have here?
 * `cmd` - contains entrypoints to our applications, and the code which is only relevant for one specific application. For now it contains only one directory `web`, for the HTTP server. In future we will also have `cli` directory.
 * `internal` - directory with a special meaning, code residing here, can't be imported from the outside of this repo. Here we will keep our business logic.
 * `pkg` - here we will put functionality which is not directly related with the business logic of our CI system, and which potentially can be useful in other projects.

 ### Setting up HTTP routing

To nicely structure our HTTP handlers, we will be defining them in separate files with the names like `handlers.*.go`. Each `handlers.*.go` file will contain HTTP handlers which will be exposed on the same URL path (ex. `handlers.pipelines.go` will be exposed as `/pipelines/*`). Also each file will export function to setup [route group](http://docs.gofiber.io/guide/grouping) (as per Fiber terminology), and this function will follow the naming convention `Setup{{RouteName}}Handlers`.

So let's create file `cmd/web/handlers.pipelines.go`, where we define our routes group for path `/pipelines` and all handlers for it:
```go {linenos=true}
package main

import (
	"github.com/gofiber/fiber/v2"
)

func SetupPipelinesHandlers(app *fiber.App) {
	pipelinesGroup := app.Group("/pipelines")

	pipelinesGroup.Post("/check-it-works", postCheckItWorks)
}

type WithRepoUrl struct {
	Url string `json:"url" xml:"url" form:"url"`
}

func postCheckItWorks(c *fiber.Ctx) error {
	body := &WithRepoUrl{}

	if err := c.BodyParser(body); err != nil {
		return err
	}

	return c.SendString("Working with repository: " + body.Url + "\n")
}
```

To finish setting up routing we need to do the following changes to our `cmd/web/main.go` file:
```go {linenos=inline,hl_lines=[8]}
package main

import "github.com/gofiber/fiber/v2"

func main() {
	app := fiber.New()

	SetupPipelinesHandlers(app)

	app.Listen(":3000")
}
```
For now we only have one handler for the sub route `/check-it-works`, which we will be using in this post.
Now if we do a curl call:
```bash
curl -X POST -H "Content-Type: application/json" \
  --data "{\"url\":\"git@github.com:flow-ci/flow-ci.git\"}" \
  http://127.0.0.1:3000/pipelines/check-it-works
```
we should get the following response:
```
Working with repository: git@github.com:flow-ci/flow-ci.git
```

### Working with git

If everything works we can start with more interesting stuff. Let's add functionality of cloning repository into the temporary folder.
For working with get repositories we will use package `go-git`, to install it run the following command:
```bash
go get github.com/go-git/go-git/v5
```


