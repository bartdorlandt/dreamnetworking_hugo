+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true
speakerType = "presentation"
location = ""
locationUrl = ""
image = "/images/speaker/{{ replace .File.ContentBaseName "-" "_" | lower }}/images/image.png"
links = [
  { name = "", url = "" }
]
description = ""
tags = []
+++
