---
layout: default
title: Home
nav_order: 1
description: "Main page of the blog"
permalink: /
---

# What's new?

- 2021-10-10
    - 원래 블로그같은 건 이직 시즌에나 쓰는거지만, 인생의 기록을 위해 한 번 시작해보기로 했다.

<br/>
---

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

  whojesBlog.printStartTime() // 2021-10-10 01:55:03 KST
}
```
