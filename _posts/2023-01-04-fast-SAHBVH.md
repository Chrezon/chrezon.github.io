---
layout: post
title: On Fast Construction of SAH-based Bounding Volume Hierarchies [Summary]
author: [Elaine Zhong]
categories: papers
tags: [SAH, BVH]
permalink: /papers/FastSAHBVH
math: true
image:
last_modified_at: 2023-01-04 00:12:00 +0800
---
[Paper Link](https://www.sci.utah.edu/~wald/Publications/2007/ParallelBVHBuild/fastbuild.pdf)

**Takeaway:** Using approximate SAH build strategies, a full BVH may be rebuilt per frame at interactive speeds without significantly reducing performance
{: .message }

# Main Points
1. Approximate SAH-based construction techniques work well on BVHs to improve build speed without significantly reducing performance 
2. A detailed **algorithm** for fast contruction of SAH-based BVHs with several optimizations 
3. Two **parallelization techniques** to further improve performance - Mixed horizontal/vertical work sharing and grid based binning.

# Summary
This paper demonstrates that previous fast approximate construction techniques work better on BVHs than on kD-trees (up to 10x faster). BVHs (at this point) were largely object split based on centroid (See [SBVH Summary](/papers/SpatialBVH) for more modern BVHs). Thus:
1. there are **no straddling "special cases"**, unlike kD-trees.
2. **Builds can be in-place** as the number of nodes are bound to $$2N-1$$, where $$N$$ is the number of triangles.
3. **Binning techniques work better**, again since we're considering centroids (one point)

Compared to previous methods that refit BVHs to deforming models, this method **rebuilds a BVH from scratch** every frame and can handle severe deformations.

## Important Previous Works
1. [Surface Area Heuristic](/papers/SAH) is used for BVH splits based on the centroid
    1. **Cost function is simplified:** Common terms (per-operation costs, total volume) are removed. Resulting in this cost for a partition $$j$$:

        $$
        \text{Cost}_j = A_{L,j}N_{L,j} + A_{R,j}N_{R,j}
        $$

        where $$A$$ represents the surface area and $$N$$ is the number of triangles.
2. [Bounding Interval Hierarchy](http://ainc.de/Research/BIH.pdf): Trade quality for speed
    1. **a-priori binning** is both faster and better - median splits based on result volume of the parent split, rather than the child's real "bounding volume". A scene can thus be **directly split into grids** [TODO: Image]
3. Techniques from kD-trees:
    1. **Ignore perfect splits:** Only consider a triangle's AABB, no clipping
    2. **Test subset of $$K$$ planes**: Project triangles into $$K-1$$ bins and compute which of these **K** planes are best. -> Splitting can be done in **2 $$O(N)$$ passes** (see below).
        1. We further **only do this along the widest axis**
        2. Mark only the **start and end bins** for each triangle.
    3. Use a **subset of the triangles**, and when the triangle count is too small, revert to full sweep.
4. **Parallelization** is carefully considered, including SIMD instructions, data layout and memory allocation

## Fast, Binned BVH Building
### Bin Setup:
1. $$K$$ bins of equal width, where the total width is the bounding box **of the centroids**
    - Since we only consider AABB centroids when splitting, this produces narrower bins
    - The region of bins are called *domains*
2. If there are $$K$$ bins, there's $$K-1$$ ways to split. Evaluate the cost function $$\text{Cost}_j = A_{L,j}N_{L,j} + A_{R,j}N_{R,j}$$ on each.
    - The surface area $$A$$'s are obtained by keeping track of each bin's bounding box ($$bb_i$$ - bin bound for bin $$i$$). Then, the left and right bounding boxes is just the union of the bin bounds. 
3. Authors suggest that 16 bins is close to optimum.

### Algorithm:
0. Preallocate space for the BVH - 2 separate, continuous arrays
    - $$N$$ triangle IDs, initialized to $$0, 1, 2, ...$$
    - $$2N-1$$ BVH nodes:
        - Inner node - Bounding box, internal flags, and position of children in the node array
        - Leaf node - offset into triangle ID array and number of triangles in that leaf.
1. A first pass for each triangle's AABB and centroid of the bounding box. (Now triangles are no longer needed)
    - Keep track of the $$vb$$ (voxel bound) - bound od all triangles
    - Keep track of the $$cb$$ (centroid bound) - bound for all centroids
2. Store above in SIMD friendly format - 4x16byte floats for each 3D position. Total of 14 SSE (Streaming SIMD Extensions) per triangle
    - 3 loads of vertices
    - 2 mins and 2 maxs to compute bounds
    - 1 add and 1 x0.5f for centroid
    - 2 min and 2 max to grow the voxel bounds and centroid bounds
    - 3 SSE stores for writing bounds and centroid to memory
3. Triangle bin projection - the bin for the $$i$$th triangle is calculated by the following:

    $$
    \text{binID}_i =  \frac {K(1-\epsilon)(c_{i,k}-cb_{min,k})}
                            {cb_{max,k} - cb_{min,k}}
    $$

    Essentially:
    - $$\frac{c_{i,k}-cb_{min,k}}{cb_{max,k} - cb_{min,k}}$$ - Fraction of the centroid position over the total width of the centroid bound of the node
    - $$K(1-\epsilon)$$ - project the fraction into 1 of $$K$$ bins (from $$0$$ to $$K-1$$).
        - The $$(1-\epsilon)$$ ensures when the fraction evaluates to 1, we project into the $$K-1$$ bin (rather than a non-existent $$K$$th bin)

    Also track:
    1. $$n_i$$ - number of primitives in that bin
    2. $$bb_i$$ - the bin bound. Initialized to $$[+\infty.. -\infty]$$ and updated via SSE min and max operations with triangle AABBs. 
4. Plane Evaluations - A left linear operation accumulating $$N_{L,i}$$ (number of triangles left of the $$i$$th partition) and $$A_{L,i}$$ (the bounds of the triangles left of the $$i$$th partition). Same with a right pass. 
    - Evaluate the cost as per cost function above.
5. Partitioning is just rearrangng the triagle ID sub-array $$[begin, end)$$ into $$[begin, mid)$$ and $$[mid, end)$$ based on the plane from step 4. Sort in-place using [Hoare partition scheme](https://en.wikipedia.org/wiki/Quicksort).
6. Recurse for each child from Step 3. Terminate when:
    1. Triangle threshold for leaf is met
    2. Centroid bounds become too small
    3. Estimated cost for a split doesn't improve from non-split. 

## Parallelization
Notice that with the current setup, access to the node array doesn't need synchronization (each subtree is disjoint). The authors suggest 2 ways of initially generating subtasks.

### Mixed Horizontal/Vertical Work Sharing:
- **Horizontal Work Sharing:** Multiple threads take subsets of the triangles and work on the same binning task in parallel
- **Vertical Work Sharing:** Each thread works on a different subtree of the BVH

1. **Setup:** Each thread takes a equal subset of triangles and calculates their AABB and centroid. Keeping track of the threads global AABB and Centroid Bound
    - Thread 0 merges global AABB and Centroid Bound. 
2. **Horizontal Splitting:** 
    1. Each thread bins an equal subset of triangles into it's own copy of $$K$$ bins. 
    2. Threads also determine their $$N_{L,i}$$s and $$N_{R,i}$$s
    3. Thread 0 performs 2 sweeps to determine the best partition (Sum $$N_i$$s and determine $$A_i$$s). 
        - Also for the split $$i$$, compute $$N_{L,i}^{(t)} = \sum_{i=0}^t N_{L,i,t}$$ and $$N_{R,i}^{(t)} = \sum_{i=0}^t N_{R,i,t}$$ - essentially, the number of left/right elements in the before the $$t+1$$th thread, to offset the next step.
    4. All threads perform second sweep, writing triangles IDs into original list. Since Thread 0 calculated the offsets, this part does not need synchronization. 
    5. Thread 0 initializes the BVH node and starts the recursion. 
3. **Switching from Horizontal to Vertical Load Balancing:** When the triangle count per node reduces, horizontal splitting is no longer effective.
    1. After splitting, record nodes where triangles are below a certain threshold in an array (begin/end interval, triangle and centroid bound). Ignore these subtrees for now. 
    2. Complete all horizontal splits first. 
    3. Sort the subtree array from Step 1 by descending size. Then start parallel (vertical) processing. 
    4. Use dynamic load balancing with an atomic counter for the next sub-tree. 

### Grid Based Binning: 
Following the observation from [Bounding Interval Hierarchy](http://ainc.de/Research/BIH.pdf), we can do several spatial median splits a-priori.
- Given $$s$$ splits we get a grid of $$2^{s,x}\times 2^{s,y}\times 2^{s,z}$$ voxels, and 2^{s_x+s_y+s_z} subtrees. These bounds can be refitted to BVH standards
    - Can manually choose $$s$$ or distribute a set number of subtrees per thread
- This is trivial to parallelize since it's one pass over all triangles (similar to horizontal step from above).

**Benefits:**
1. Fewer barrier operations since just 1 pass, no recursion unlike horizontal work splitting from above. 
2. Avoids top-level split operations.

**Drawbacks:** 
1. SAH is not used for the top few levels. Bad for performance.
2. Uneven triangle distribution means significant number of triangles will be projected into the same bin (uneven load balancing, and bad performance).


