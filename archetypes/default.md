---
author: ["Denis Rodin"]
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: "{{ .Date }}"
description: "{{ replace .File.ContentBaseName "-" " " | title }}"
summary: ""
tags: []
categories: []
series: []
cover:
  image: images/{{ .File.ContentBaseName }}.png
  caption: "{{ replace .File.ContentBaseName "-" " " | title }}"
  hidden: false # hide everywhere but not in structured data
  hiddenInList: false # hide on list pages and home
  hiddenInSingle: false # hide on single page
ShowToc: true
TocOpen: true
draft: true
---