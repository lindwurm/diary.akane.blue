---
title: {{ replaceRE `^\d{8}-` "" .File.ContentBaseName | humanize }}
slug: {{ replaceRE `^\d{8}-` "" .File.ContentBaseName }}
date: {{ .Date | time.Format "2006-01-02 15:04:05" }}
tags:
---
