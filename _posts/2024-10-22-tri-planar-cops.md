---
layout: post
title: building a tri planar projection in cops (opencl)
---

Tri planar projections are a useful tool for any texturing or lookdev artist, surprisingly there's no native tri planar in cops 2.0, as of houdini 20.5... But that means we have an opportunity to build our own from scratch. This could be done using vex or a large network of native cops nodes, but for processing speed opencl is our best bet.

![rendered tri planar example](/assets/images/2024-10-22-tri-planar-cops/tri_planar_example001.jpg)

A tri planar projection is a set of 3 parallel projections, front, side and top, blended based on their respective facing ratios. Knowing this, the steps of our process will be:

- create texture projections from x, y and z
- build masks for each projection using facing ratios
- blend textures using masks

-----

### initial setup

This tutorial assumes you’re reasonably familiar with cops but I’ll go over a quick bit of set up. Create the following network and link or create your geometry in the sop import, uv’s and normals are required on the geometry. On the wrangle, change the type of the first input to Geometry, then add an RGB input and name it origP.

![inital setup](/assets/images/2024-10-22-tri-planar-cops/set_up_screenshot001.JPG)

Fill the wrangle with this vex snippet:

{% highlight jc++ linenos %}

int prim;
vector uv;
xyzdist(1,v@P,prim,uv);

v@origP = primuv(1,"origP",prim,uv);

{% endhighlight %}

