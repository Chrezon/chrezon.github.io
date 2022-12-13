---
layout: post
title: Spatial Splits in Bounding Volume Hierarchies [Summary]
author: [Elaine Zhong]
categories: paper-summaries
tags: acceleration-structures
permalink: /papers/SpatialBVH
math: true
image: 
last_modified_at: 2022-12-12 00:12:00 +0800
---
[Paper Link](https://www.nvidia.in/docs/IO/77714/sbvh.pdf)

**Takeaway:** Building a BVH with strategic **spatial partitioning** will improve performance in most cases without significant memory footprint overhead, especially non-uniformly tessellated scenes.
{: .message }

## Main Points
1. BVH can be constructed with a combination of **spatial** and **object splits** to improve performance and memory footprint. 
2. By comparing the SAH cost of each method, the optimal strategy can be selected. This also means this method will never be worse than a regular BVH.
3. The authors suggest several methods to efficiently construct and minimize memory costs of spatial splits. They are **Chopped Binning**, **Reference Unsplitting**, and **Restricting Spatial Split Attempts**.

## Summary
Raytracing applications rely on **pre-processed data structures** to achieve fast performance. Hierarchical data structures like **BVHs** and **kD-trees** take advantage of spatial cohesion to systematically eliminate volumes. There are 2 main ways to divide volumes:
1. **Spatial Partitioning:** Bounding volume is divided into smaller volumes, primitives inserted into every volume it intersects. 
  - Excellent culling efficiency in difficult scenes
  - High memory consumption - deep trees and reference duplication
2. **Object Partitioning:** Divide primitives into disjoint sets by sorting primitives + splitting according to the Surface Area Heuristic.
  - Overlapping regions are expensive (worse performance)
  - Almost always fewer nodes than spatial splits
  - Usually, AABBs are used.

Overlapping volumes are inefficient. Thus, it's strategic to do spatial partitioning in specific scenarios.

**Surface Area Heuristic:** Estimates the cost of raytracing children nodes. Surface area ($$SA$$) represent how likely a ray is to hit a particular box. $$P$$ denotes the number of primitives in that box. and $$C_i$$ is the cost of 1 ray-primitive intersection, $$C_t$$ is the cost of a traversal step.

$$
C = C_t + \frac {SA(B_1)}{SA_B}|P_1|C_i + \frac {SA(B_2)}{SA_B}|P_2|C_i
$$

Compare this with the cost of intersecting all primitives to decide whether to split. 

### SBVH:
Key difference from previous methods is that primitives can be **sorted into multiple volumes**. Thus reference splitting is done in **construction phase** rather than as a preprocess. 
- **Note:** since each BVH has references to all children, traversal is not affected

```
Algorithm:
1. Find an Object Split Candidate - Same as previous methods using SAH
2. Find a Spatial Split Candidate - See below, the contribution of this paper
3. Pick the winner candidate based on SAH cost
```
### Constructing Spatial Split Candidates:
#### Chopped Binning: 
To make finding a split plane feasible, only a subset of all possible planes are considered. 
1. The parent AABB is divided into fixed number of equidistant "bins" and triangles are sorted into the bins in $$O(n)$$ time.
  - An AABB is associated with each bin, only the part of the primitive inside the bin contributes to growing that AABB (clipping it to the bin, hence "chopped" binning)
2. The primitive reference in/out count and AABB sizes for each bin can be easily combined to each side of a split candidate to calculate the SAH cost.
3. We pick the best candidate and compare its cost against the object split cost.

Note that we do not attempt to find the exact minimum for the SAH (could be anywhere, not just reference bounds). Using the SBVH algorithm will never be worse than a regular BVH algorithm, since that's what we compare against.
- **TODO:** Is this really true? Is this not a greedy algorithm? How do we know a previous split will not affect the later splits

#### Reference Unsplitting:
A way to reduce memory duplication. Sometimes, rather than letting a reference duplicate by straddle a boundary, it is more beneficial to only insert the shape into one sub BVH, and allow the BVHs to overlap a bit. The authors suggest the following heuristic to decide which sub BVH (or both) to place a reference into: 

$$
C_{split} = SA(B_1)N_1 + SA(B_2)N_2
$$

$$
C_1 = SA(B_1 \cup B_\Delta)N_1 + SA(B_2)(N_2 -1)
$$

$$
C_2 = SA(B_1)(N_1 -1)+SA(B_2 \cup B_\Delta)N_2 
$$

Then the lowest cost option is picked. Note $$C$$ denotes cost of each option, $$B$$ is the bounding volumes and $$N$$ is the number of references on each side.

#### Restricting Spatial Split Attempts:
Another way to reduce reference duplication. Note we can always do a simple object split (low memory cost). Thus, we should strategically select nodes for which SBVH are considered. Use the **area of overlap** of object split as the criterion: 

$$
\lambda = SA(B_1 \cap B_2)
$$

$$
\frac \lambda {SA(B_{root})} > \alpha
$$

Essentially, if the intersecting area in a object split larger than some fraction $$\alpha$$ of the total area, then we try spatial split. Generally, spatial splits will occur near the top of the tree. Most improvements are achieved before $$\alpha = 10^{-5}$$ without significantly increasing reference count. This results in minimal performance degradation while giving shallower trees and much fewer reference duplications.

## Related Concepts
### Forward
1. 2-Level BVHs, commonly used in real-time raytracing seem to be related to this concept
2. Parallel BVH construction methods

### Backward
1. Early Split Clipping
2. Edge Volume Heuristic