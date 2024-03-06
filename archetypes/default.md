---
date: '{{ .Date }}'
title: '{{ replace .File.ContentBaseName `-` ` ` | title }}'
summary: 'this is summary'
---