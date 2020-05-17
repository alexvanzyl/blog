---
title: Second post
description: This is a greate post
date: 2020-05-17T19:51:04.321Z
author: Alexander van Zyl
tags:
  - test
featuredImage: /uploads/featured-image.png
---
```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}
```

![](/uploads/featured-image.png)