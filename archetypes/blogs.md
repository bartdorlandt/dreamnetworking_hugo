---
title: '{{ replaceRE `^\d{8}_` "" .File.ContentBaseName | replaceRE `[-_]` " " | title }}'
date: {{ .Date | time.Format "2006-01-02" }}
description: ""
image: "/blogs/{{ .File.ContentBaseName }}/images/image.png"
tags: []
---
