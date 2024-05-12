---
author: ["Denis Rodin"]
title: "Build your own CI system in Go (Part 2)"
date: "2024-05-12"
tags: ["Go"]
categories: ["Build your own X", "Golang"]
series: ["Build your own CI"]
ShowToc: true
TocOpen: false
draft: true
---

## Intro

In the previous part we setup a structure for our Go project and created simple web server with one endpoint dedicated to testing our CI functionality. By the end of the first part our server managed to test and build itself. In this part we will quickly start with implementing CLI tool for local usage and then switch to solving ...

## Creating CLI tool

First let's add instructions to build cli binary to our Makefile:
```Makefile {linenos=inline,hl_lines=[2,7,15,16,17,18,19]}
WEB_APP = web
CLI_APP = cli
BUILD_DIR = $(PWD)/bin

.PHONY: test bench run-web run-cli all

all: web cli

web:
	go build -o $(BUILD_DIR)/$(WEB_APP) cmd/$(WEB_APP)/main.go

run-web: web
	$(BUILD_DIR)/$(WEB_APP)

cli:
	go build -o $(BUILD_DIR)/$(CLI_APP) cmd/$(CLI_APP)/main.go

run-cli: cli
	$(BUILD_DIR)/$(CLI_APP)	

test:
	go test -v -race ./...

bench:
	go test -bench=. -benchmem ./...
```
Now we should create `main.go` file inside `cmd/cli/` directory. Our CLI tool will work with the git repository in the current working directory, so let's first determine where the tool is being executed:
```go
	cwd, err := os.Getwd()
	if err != nil {
		log.Fatal(err)
	}
```
Next we can use the function to create workspace from the specified directory, which we prepared in th first chapter:
```go
	ws, err := ci.NewWorkspaceFromDir(cwd)
	if err != nil {
		log.Fatal(err)
	}
```
The rest is pretty straightforward, we instantiate executor and run default pipeline of the repository, printing pipeline log at the end:
```go
	executor := ci.NewExecutor(ws)
	output, err := executor.RunDefault(context.TODO())
	if err != nil {
		log.Println(err)
	}

	log.Println(output)
```
Here we use `context.TODO()` as a reminder for us to add proper context handling in the future. If we execute our CLI tool right now:
```bash
make run-cli
```
We should see almost the same text as we get from the test endpoint of our server. Very nice result. We re-use business logic between two of our applications, we can get same exact result on server and locally. Now it is the time to think about ...
