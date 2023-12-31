---
layout: post
title:  "Intro of Modern Linux Device Driver"
date:   2023-12-30 22:32:15 +0900
categories: ETC
---

When I read [Linux Device Driver 3rd Edition][ldd-link] for the first time, it turns out all the concepts I thought I did understand were so volatile. So I decided to get my hands dirty by implementing all the codes shown in the book. However, There were two issues I suffered from the most throughout my journey.
- A lot of example codes are incompatible with modern Linux kernel
- Learning directly through the complete code as in the [Oreilly gitlab repo][gitlab-repo] is of too much noise. I expected a bottom up approach to build up my driver code.
Following modern linux device driver postings will focus mainly on the code compatibility to modern Linux and the bottom up approach of the device driver source code.

> All source codes are written and tested based on Linux kernel 6.7.0-rc1

🛫[Chapter 2. Building And Running Modules]({% post_url 2023-12-30-Chapter2 %})

🔠[Chapter 3. Char Drivers]({%- post_url 2023-12-30-Chapter3 -%})

🐞[Chapter 4. Debugging Techniques]({%- post_url 2023-12-30-Chapter4 -%})

[ldd-link]: https://lwn.net/Kernel/LDD3/
[gitlab-repo]: https://resources.oreilly.com/examples/9780596005900