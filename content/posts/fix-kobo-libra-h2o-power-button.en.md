---
title: "Kobo Libra H2O Power Button Not Working? How I Fixed It"
date: 2026-09-05T00:00:00+08:00
draft: false
tags: ["Kobo", "Repair", "E-reader", "Teardown"]
categories: ["Gadgets"]
summary: "The power button on my Kobo Libra H2O stopped responding entirely — no click, no travel, nothing. The material inside the button had gone soft, and the fix cost me a scrap of plastic and twenty minutes."
description: "My Kobo Libra H2O wouldn't wake from sleep and the power button had no click left. The rubber dome inside had degraded. Here's the fix."
# cover 是從第一張圖裁成 1200x600 的版本（正方形當縮圖太高，社群卡片也會被亂裁）。
# 路徑不加開頭的 /，PaperMod 的 cover.html 才比對得到 assets/ 裡的原圖，
# 列表頁縮圖就會走 Hugo 的 resize（否則會直接塞原檔）。
# og:image 走的是 absURL，指向 static/ 那份同名 JPEG，所以兩邊都要有。
cover:
  image: "images/kobo/kobo_libra_h20_cover.jpg"
  alt: "The power button on the back of a Kobo Libra H2O"
  relative: false
  hiddenInSingle: true
ShowToc: true
TocOpen: true
---

My Kobo Libra H2O had been sitting in a drawer for a couple of months. When I tried to turn it on, nothing happened, so I left it charging for several hours. The battery turned out to be fine — with the cable plugged in it would sometimes come back on — but once it went to sleep, nothing would wake it up again.

That's when I noticed the power button itself. Short press, long press, nothing — and there was no click and no travel when I pushed it. It seemed like the problem was the button, not the battery.

I searched around and couldn't find anyone fixing this kind of problem, and no shop would take on a repair this small, so I decided to open it up and fix it myself.

![The power button on the back of the Kobo Libra H2O](/images/kobo/kobo_libra_h20_1.jpg)

## Removing the Back Cover

There are no screws on the back cover. You can pop it off with a thin piece of plastic — I used a guitar pick — moving slowly around the edge of the cover until all the clips release.

![Using a guitar pick to pry the clips loose](/images/kobo/kobo_libra_h20_2.jpg)

![Working along the side until the cover comes free](/images/kobo/kobo_libra_h20_3.jpg)

## Finding the Problem: A Degraded Rubber Dome

The power button is a small module attached to the inside of the back cover. Inside it is a small rubber dome, the same kind of part that sits under most keyboard and remote-control buttons. This one had degraded — it had gone soft and sticky instead of springy, so it couldn't press the switch on the board any more.

![The power button module attached to the inside of the back cover](/images/kobo/kobo_libra_h20_4.jpg)

## How I Fixed It

The fix was pretty simple. I scraped out the old material inside the button, along with the dirt stuck to it.

![A close-up of the inside of the button](/images/kobo/kobo_libra_h20_5.jpg)

Inside the button, there are two circles, one inside the other. I cut a small round piece of plastic to about the same height as the inner circle and placed it in there. That was enough for the button to reach the switch below.

![The button cleaned out and put back in place](/images/kobo/kobo_libra_h20_6.jpg)

## Wrapping Up

So it wasn't a circuit problem after all. The material inside the button had just worn out over time and lost its spring. It's been working normally since. It's an easy fix if you don't mind opening the case, and I'm writing it down here in case someone runs into the same thing.
