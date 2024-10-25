---
layout: post
title: building a tri planar projection in cops (opencl)
---

Tri planar projections are a useful tool for any texturing or lookdev artist, surprisingly there's no native tri planar in cops 2.0, as of houdini 20.5... But that means we have an opportunity to build our own from scratch. This could be done using vex or a large network of native cops nodes, but for processing speed opencl is our best bet.

![rendered tri planar example](/assets/images/2024-10-22-tri-planar-cops/tri_planar_example001.jpg)

-----