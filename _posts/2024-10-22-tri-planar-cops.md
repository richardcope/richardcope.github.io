---
layout: post
title: building a tri planar projection in cops (opencl)
---

Tri planar projections are a useful tool for any texturing or lookdev artist. Since there’s no native tri planar in cops 2.0 (as of houdini 20.5), we have a good excuse to build our own. This could be done using vex or a large network of native cops nodes, however processing speed makes opencl preferable.

![rendered tri planar example](/assets/images/2024-10-22-tri-planar-cops/tri_planar_example_001.jpg)

A tri planar projection is a set of 3 parallel projections, front, side and top, blended based on their respective facing ratios. Knowing this, the steps of our process will be:

- calculate blending weights using normals
- sample textures for each projection plane
- blend textures using weights

-----

### initial setup

This tutorial assumes you’re reasonably familiar with cops but I’ll go over a quick bit of set up. Create the following network and link or create your geometry in the sop import, uv’s and normals are required on the geometry. On the wrangle, change the type of the first input to **Geometry**, then add **RGB** inputs named **origP** and **N**.

![inital setup](/assets/images/2024-10-22-tri-planar-cops/wrangle_001.png)

Fill the wrangle with this vex snippet:

{% highlight py %}int prim;
vector uv;
xyzdist(1,v@P,prim,uv);

v@origP = primuv(1,"origP",prim,uv);
v@N = primuv(1,"N",prim,uv);{% endhighlight %}

This process just maps your position and normal data to uv space so we can map 3D elements in cops. Using the xyzdist function does a great job of extrapolating the values at uv boundaries so we don’t have to worry about seams.[^1]

[^1]: Testing footnotes 1

Add an opencl node and replace the kernel code with the following:

{% highlight py %}#bind layer !&Cdout float3

#bind layer origP? float3
#bind layer N? float3

@KERNEL
{
    @Cdout.set(@origP);
}

{% endhighlight %}

Finally there’s a couple of steps to access the data from our wrangle in opencl.

- Select spare parameters button
- Remove src and dst under the signature tab
- Connect origP and N from wrangle

![wrangle to opencl](/assets/images/2024-10-22-tri-planar-cops/opencl_bindings_001.gif)

-----

### blending weights

We need to calculate the facing ratio of the surface in relation to x, y and z, then store this as 3 weights. This could be done using a dot product but it’s more efficient to just use the individual components of N, this works as long as N is normalised. For example, dot((float3){1.0f, 0.0f, 0.0f}, N) will produce the same result as N.x.
The result of this ranges from 1 (facing) to -1 (facing away), the weights need to be in a positive range so a fabs function can be used to get absolute values. Putting this together for x comes the following:

{% highlight py %}#bind layer !&Cdout float3

#bind layer origP? float3 val=0
#bind layer N? float3 val=0

float3 N, weight, col;

@KERNEL
{
    N = normalize(@N);
    
    weight.x = fabs(N.x);
    
    //zy plane
    col = (float3){1.0f, 0.0f, 0.0f} * weight.x;
    
    @Cdout.set(col);
}

{% endhighlight %}

We want some way to control the falloff of the blending, one method for this is raising our facing ratio to a power.[^2] To do this, bind a float parameter named exponent (feel free to give this a more intuitive label).

[^2]: Testing footnotes 2

{% highlight py %}#bind parm exponent float val=1.0{% endhighlight %}

Then raise N.x to the exponent:

{% highlight py %}weight.x = pow(fabs(N.x), @exponent);{% endhighlight %}

![blend falloff with exponent](/assets/images/2024-10-22-tri-planar-cops/exponent_001.gif)

The limits to set your exponent parameter at are arbitrary, something like 1 - 100 works fairly well.

<p class="message">
  I don’t have the strongest maths foundation so found visualising the results of these values with a graphing calculator helpful
  ![exponent graphs](/assets/images/2024-10-22-tri-planar-cops/exponent_graphs_001.png)
</p>

We’ll test by blending red, green and blue, using addition assignment to blend the new colors. Bringing in the other weights should look something like this:

{% highlight py %}#bind layer !&Cdout float3

#bind layer origP? float3 val=0
#bind layer N? float3 val=0

float3 N, weight, col;

@KERNEL
{
    N = normalize(@N);
    
    weight.x = pow(fabs(N.x), @exponent);
    weight.y = pow(fabs(N.y), @exponent);
    weight.z = pow(fabs(N.z), @exponent);
    
    //zy plane
    col = (float3){1.0f, 0.0f, 0.0f} * weight.x;
    
    //zx plane
    col += (float3){0.0f, 1.0f, 0.0f} * weight.y;
    
    //xy plane
    col += (float3){0.0f, 0.0f, 1.0f} * weight.z;
    
    @Cdout.set(col);
}

{% endhighlight %}

![blend weights](/assets/images/2024-10-22-tri-planar-cops/blend_weights_001.PNG)