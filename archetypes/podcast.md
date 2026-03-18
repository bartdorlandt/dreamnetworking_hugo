+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true
type = "speaker"
speakerType = "podcast"
location = ""
locationUrl = ""
image = "/images/speaker/{{ replace .File.ContentBaseName "-" "_" | lower }}.png"
links = [
  { name = "", url = "" }
]
description = ""
tags = []
+++
