---
layout: fragment
title: JsonObject的getObject踩坑
tags: [java]
description: JsonObject的getObject取值问题
keywords: Java, JsonObject,
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

JsonObject调用getObject()的时候，如果get不到这个值，默认将会返回“null"字符串赋值给jsonObject对象，不会抛出异常，如果是getIntValue则会复制0，如果需要判断值存不存在，应该使用contains方法。

