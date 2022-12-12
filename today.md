---
layout: page
title: What I Learned Today
permalink: /today/
math: true
---
**What I'm Working On:** Writing a simple fluid simulator in ShaderToy.

## 2022/11/26
I don't have a good explanation for the long break...
1. I don't know how I've never noticed, but BVH's have 2 main heuristics - Spatial partitioning scheme and object partitioning scheme
    - **Spatial Partitioning**: Like kD-tree, primitives inserted into ALL partitions it overlaps. Very good culling performance, but multiple references to primitives, and high memory consumption due to deep trees
    - **Object Partitioning**: Primitives sorted by centroid on all 3 axes and the best one is picked by some heuristic, such as Surface Area Heuristic (SAH)
2. **Surface Area Heuristic**: Given parent bounding box \[B_p\] and child bounding box \[B_c\] where \[B_c \subseteq B_p\], then the cost of a ray-box traversal is: \[C = C_t + \frac{SA(B_1)}{SA(B)}|P_1|C_i + \frac{SA(B_2)}{SA(B)}|P_2|C_i\]. Where \[P\]s denote the number of primitives in each box, and \[C\]s denote the cost for traversal and one ray-primitive intersection respectively. 

## 2022/11/01
1. This article clears up a confusion I couldn't even fully articulate: [Gradient, Jacobian, Hessian, Laplacian and all that](https://najeebkhan.github.io/blog/VecCal.html)

## 2022/10/29
1. Convolution can happen in 3D! I'm surprised this never occurred to me (I guess since I first learned about it in 2D). Instead of using pixels, convolve on voxels.
2. I'm not sure if you can use Gauss-Seidel for smoke diffusion with ShaderToy, to get a reasonable number of iterations you need to repeatedly write to the density buffer, which isn't possible from what I can tell?
3. ShaderToys have a render order! They occur in tab order -> Buffer A, B, C, D, then Image

## 2022/10/28
1. `step` function in shading languages. Seems to be equivalent to a comparison operation like `x < y`. [StackOverflow](https://stackoverflow.com/questions/51666285/step-vs-comparison-operator-in-hlsl) seems to suggest they actually compile down to the same instructions. Wondering if `step` is just the more canonical way to write shaders... (I suppose there's `smoothstep` and such)
2. There's a lovely Shadertoy [tutorial](https://inspirnathan.com/posts/47-shadertoy-tutorial-part-1/) by inspirnathan. I find it most satisfying to look at the desired outputs and poke around in Shadertoy until I get there. My code structure often ends up looking different, but gets the job done!
3. The change in a fluid's density at a certain point can be described as the sum of the effects of external forces (velocity field), diffusion, and additional density sources.
## 2022/10/26
1. Fluid simulations are done in 2 main ways - Grid-based and Particle simulations.
    - **Grid Based**: Accurate but slow. Easy to differentiate over space
    - **Particle Based**: Fast but less accurate. Easy conservation of mass
2. The **Courant–Friedrichs–Lewy(CFL) Condition**: Necessary condition for the convergence of certain PDEs. Informally, it sets a max possible time step to ensure we do not "skip over" intervals. 
3. The buffer/texture sizes of shader inputs may not always be the same in complex rendering pipelines (e.g. when the image goes through super-resolution but the depth buffer does not). A shader pass may need to carefully consider how to appropriately read from each of its buffers. 