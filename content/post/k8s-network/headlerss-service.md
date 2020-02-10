---
title: "浅谈K8S中的Healess Service"
date: 2020-02-02T15:56:20+08:00
draft: true
author: "ciii"
description: "kubernetes headless service"
tags: ["kubernetes"]
categories: ["concept"]
---

## 概述

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
      targetPort: 8080
```
