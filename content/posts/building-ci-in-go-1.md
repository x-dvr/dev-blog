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

### Intro

I started recently playing with Golang and liked it a lot. So I wanted to get a deeper understanding of the language, and what is the best way
to get the real feel of it if not building a real application with it? And I decided to build my on CI tool from scratch. Of course I'm not going
to build a "Jenkins killer" or something even close to it (at least not now 🙂). The goal now is to better understand CI tooling internals, and create a tool which will be able to test/build itself and produce some artifact, be it executable or a docker container.

Let's **Go**!

### Architecture

So what will we actually build together during the course of this series? First of all, I'll provide some definitions, so we are all on the same page.

1) CI System/Tool - system which essentially manages the process of building software. When new code is being committed to the repository, the CI tool initiates a build process.
1) CI pipeline - deterministic set of steps, needed to be executed to perform the "build". Can include, linting, testing, compiling, creating container, etc.
1) 

 Let's draft a simplified architecture of CI system:
![CI Architecture](/images/ci-arch.png)

Our hypothetical CI system consists of the following parts:
* CI Server, which manages the state of all pipelines