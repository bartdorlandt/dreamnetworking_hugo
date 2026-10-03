---
title: '{{ replaceRE `^\d{8}_` "" .File.ContentBaseName | replaceRE `[-_]` " " | title }}'
date: {{ .Date | time.Format "2006-01-02" }}
speakerType: podcast
location: ""
locationUrl: ""
image: /speaker/{{ .File.ContentBaseName }}/images/image.png
description: ""
links:
  - name: Episode page
    url: ""
tags: []
---
