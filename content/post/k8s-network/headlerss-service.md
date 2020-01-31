---
title: "Headlerss Service"
date: 2020-01-22T15:56:20+08:00
draft: true
author: "ciii"
description: "kubernetes headless service"
tags: ["kubernetes"]
categories: ["concept"]
---

## 浅谈 K8S 中的 Healess Service

<!--more-->

spec 中 clusterIP 为 None

```yml
kind: Service
apiVersion: v1
metadata:
  name: httpbin
  namespace: test
spec:
  selector:
    app: httpbin
  clusterIP: None
  ports:
    - name: web
      port: 80
      targetPort: 8080s
```
