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

In the previous part we setup a structure for our Go project and created simple web server with one endpoint dedicated to testing our CI functionality. By the end of the first part our server managed to test and build itself. In this part we will switch to solving two main problems of direct execution approach: 

1) CI machine may not have all the required dev tools installed;
1) If there will be a lot of concurrent requests to execute build pipeline, one server will not be able to handle it.

To solve first issue it makes sense to execute pipeline inside containerized environment, then we only need to have containerization runtime on our CI machines, and to execute pipelines with different build tools we just need to pull proper image. Solving the second issue will require us to change the architecture of our server. From one monolithic application we will need to go to controller-worker structure:
![CI Architecture](/images/ci-workers.png)
With this approach one central server is accepting requests to execute pipeline and then distributing this requests to worker nodes, and they are doing real work. This involves a lot of complex logic:
- maintaining the list of worker nodes;
- monitoring their actual load to properly distribute work between them;
- implementing queuing mechanism, to avoid overflowing worker nodes with load.

But is it possible to solve both problems of containerization and scaling in one go, without introducing huge complexity into our code? - Yes! We can utilize infrastructure of Kubernetes cluster. If you are working on a big project, or if your project is somehow relates to the cloud tech, more often than not you are already having one if not several k8s clusters. Then why not using it for distribution of our workload.

## Scaling & containerizing

If yoy are like me, using linux as a main development machine, then I would recommend as a simplest solution for testing things locally to install QEMU, and create ubuntu or debian virtual machine, and on this machine to install single-node [k3s](https://docs.k3s.io/quick-start) cluster. After installing k3s, just copy paste file `/etc/rancher/k3s/k3s.yaml` form guest machine to the directory `~/.kube/config` on your host machine. If you are using Mac, then the easiest way to setup local k8s cluster will be using [Colima](https://github.com/abiosoft/colima).

If our environment is setup, we can continue with installing k8s client for Go:
```bash
go get k8s.io/client-go@latest
```
