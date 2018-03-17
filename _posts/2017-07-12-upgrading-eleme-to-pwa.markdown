---
layout:       post
title:        "gdbprof分析ceph的4K-randwrite性能"
subtitle:     "分析ceph进行4k随机写时哪些线程及其函数较为耗资源"
date:         2018-03-17 10:20:00
author:       "YMG"
header-img:   "img/in-post/post-eleme-pwa/eleme-at-io.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - 云存储
    - ceph
---

<!-- Chinese Version -->
<div class="zh post-container">
    {% capture about_zh %}{% include posts/2017-07-12-upgrading-eleme-to-pwa/zh.md %}{% endcapture %}
    {{ about_zh | markdownify }}
</div>

<!-- English Version -->
<div class="en post-container">
    {% capture about_en %}{% include posts/2017-07-12-upgrading-eleme-to-pwa/en.md %}{% endcapture %}
    {{ about_en | markdownify }}
</div>
