---
# You can also start simply with 'default'
theme: academic
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Learn KVM
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# take snapshot for each slide in the overview
overviewSnapshots: true
---

# Learn KVM
深度探索 Linux 系统虚拟化：原理与实现（王柏生 谢广军 著）

[视频教程 - 泽文(Chao Liu)](https://www.bilibili.com/video/BV1MyyHYbEPa)

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/zevorn/learn-kvm" target="_blank" alt="GitHub" title="Open in GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
layout: center
---

# 内容简介

- 第 1 章 CPU 虚拟化
- 第 2 章 内存虚拟化
- 第 3 章 中断虚拟化
- 第 4 章 设备虚拟化
- 第 5 章 Virtio 虚拟化
- 第 6 章 网络虚拟化

---
src: ./pages/chapter1.md  # 本页只包含 frontmatter
---
