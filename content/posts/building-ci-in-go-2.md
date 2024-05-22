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

In the previous part we setup a structure for our Go project and created simple web server with one endpoint dedicated to testing our CI functionality. By the end of the first part our server managed to test and build itself. In this part we will switch to solving ...

## Scaling & containerizing

Install debian or ubuntu server in QEMU, install k3s from [here](https://docs.k3s.io/quick-start) and then copy paste file `/etc/rancher/k3s/k3s.yaml` form guest machine to `~/.kube/config`.

```bash
go get k8s.io/client-go@latest
```