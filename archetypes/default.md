---
date: "{{ .Date }}"
draft: true
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
slug: "{{ .File.TranslationBaseName }}"
description: "{{ .File.TranslationBaseName }}"
image:
categories:
tags:
license: GNU General Public License v3.0
---
