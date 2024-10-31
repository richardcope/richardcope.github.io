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

This tutorial assumes you’re reasonably familiar with cops but I’ll go over a quick bit of set up. Create the following network and link or create your geometry in the sop import, uv’s and normals are required on the geometry. On the wrangle, change the type of the first input to **Geometry**, then add an **RGB** input and name it **origP**.

![inital setup](/assets/images/2024-10-22-tri-planar-cops/set_up_screenshot001.jpg)

Fill the wrangle with this vex snippet:


{% highlight py %}int prim;
vector uv;
xyzdist(1,v@P,prim,uv);

v@origP = primuv(1,"origP",prim,uv);{% endhighlight %}

This process just maps your position data to uv space so we can map 3D elements in cops. Using the xyzdist function does a great job of extrapolating the values at uv boundaries so we don’t have to worry about seams.[^1]

Add an opencl node and in the signature tab set:

- input1 to **origP**, type **RGB**
- output1 to **Cdout**, type **RGB**

Replace the kernel code with the following to reflect our input output changes:

{% highlight py %}#bind layer !&Cdout

#bind layer origP? float3

@KERNEL
{
    @Cdout.set(@origP);
}
{% endhighlight %}

Connect the output of the wrangle and we’re all set to start doing some work.

-----

### projections

We need to create projections for the xy, zy and zx planes, let’s start with xy. Getting coordinates for this is really straightforward, we just need to build uv coordinates from the 2 relevant axes. This is done by setting a float2 uv variable to @origP.xy.

{% highlight py %}@KERNEL
{
    float4 col = (float4){0.0f, 0.0f, 0.0f, 0.0f};
    
    float2 uv = @origP.xy;
    
    @Cdout.set(col); 
}
{% endhighlight %}

Now we’ve got our uv coordinates, we need to use them to sample a texture. Bind a new layer called tex with the same code as we used to bind origP then click the small icon above the text box to add the relevant inputs.

We use textureSampleRect to sample our input texture and store this to a variable named proj_tex:
