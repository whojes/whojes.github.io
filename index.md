---
layout: default
title: Home
nav_order: 1
description: "Main page of the blog"
permalink: /
---

# for fun

```java
fun main(vararg args: String) {
  val whojesBlog = BlogBuilder.create()
      .setProvider(GITHUB_PAGES)
      .useTemplate(JEKYLL.with("just-the-docs"))
      .withSubjects([
        "cats",
        "develop",
        "life",
        "essay",
        "travel"
      ])
      .build()
  whojesBlog.start()

  println(LocalDateTime.now()) // 2021-10-10 01:55:03 KST
}
```
