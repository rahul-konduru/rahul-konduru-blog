+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
author = '{{ .Site.Params.Author.name }}'
+++
