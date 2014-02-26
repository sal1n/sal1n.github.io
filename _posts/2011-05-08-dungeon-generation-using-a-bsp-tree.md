---
layout: post
title: Dungeon generation using a BSP Tree
---

Using a BSP Tree to generate a dungeon is surprisingly simple yet effective.

Start with a rectangle and split it recursively until each sub-rectangle is approximately the size you want your dungeon rooms to be. The splitting operation is:

- select a split orientation – vertical or horizontal
- select a random position (x for vertical, y for horizontal)
- split and apply above to sub-rectangles (recursive)

![BSP iterations](/assets/img/bsp_iterations.gif)

...and so on. What you are left with is a BSP tree whose leaf nodes you can use as room containers for your dungeon. This has numerous benefits – the leaf nodes are guaranteed not to overlap and you can guarantee that all rooms can be connected by recursively traversing the tree backwards and connecting sub-nodes with corridors.

For [Sanguine](https://github.com/sal1n/sanguine) all I need to generate is an ASCII representation which my Map class can parse to instantiate all of the Locations using a given key. Here is quick screen shot of a generated small map sans corridors:

![BSP generation](/assets/img/bsp_generation.png)