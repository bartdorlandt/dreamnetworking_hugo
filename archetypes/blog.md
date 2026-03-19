+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true
image = "/images/blogs/{{ replace .File.ContentBaseName "-" "_" | lower }}/images/image.png"
tags = []
+++
