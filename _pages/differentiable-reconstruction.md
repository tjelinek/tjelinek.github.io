---
title: "Differentiable Object Reconstruction and Tracking"
permalink: /differentiable-reconstruction/
layout: single
author_profile: true
toc: true
toc_label: "On this page"
toc_sticky: true
mathjax: true
excerpt: "A differentiable-rendering approach to joint 6DoF pose and mesh recovery, and the two failure modes (flat-disk collapse, correlated occlusion outliers) that ended it."
---

*Discontinued research project. The lessons fed directly into the [correspondence-based reconstruction pipeline](/reconstruction/).*

## Problem

Building on the model-free tracker of Rozumnyi et al., we are given a video stream $$\{I_1, \dots, I_n\}$$ of a rigid object and an initial segmentation mask $$S_1$$, and we seek globally consistent 6DoF poses $$\mathbf{p}_1, \dots, \mathbf{p}_n$$ together with a 3D model represented as a mesh $$M$$. Under the rigidity assumption $$M$$ is constant across frames; it is recovered by deforming per-vertex offsets from a prototype (a sphere whose radius matches the projected object in $$I_1$$) through a differentiable renderer.

## Objective

The poses and mesh are fit over a sparse keyframe set $$K_n \subseteq \{1, \dots, n-1\}$$. Let $$\hat{I}_i$$ and $$\hat{S}_i$$ be the appearance and silhouette rendered from $$M$$ at pose $$\mathbf{p}_i = (T_i, Q_i)$$ (translation and unit quaternion), $$F(I_i)$$ the deep features of frame $$I_i$$, $$S_i$$ its input segmentation, and $$\mu_n = (\lvert K_n \rvert + 1)^{-1}$$. The appearance term uses a Cauchy robust norm $$\lVert \cdot \rVert_\gamma$$, the silhouette term an IoU plus a distance-transform ($$\mathrm{DT}$$) penalty:

$$
L_F = \mu_n \!\!\sum_{i \in K_n \cup \{n\}}\!\! \big\lVert \hat{S}_i \cdot (\hat{I}_i - F(I_i)) \big\rVert_\gamma,
$$

$$
L_S = \mu_n \!\!\sum_{i \in K_n \cup \{n\}}\!\! \big(1 - \mathrm{IoU}(S_i, \hat{S}_i)\big) + \big\lVert \mathrm{DT}(S_i) \cdot \hat{S}_i \big\rVert.
$$

Motion is regularized toward the most recent keyframe $$k = K_n[-1]$$,

$$
L_M = \max\!\Big(0, \tfrac{\lVert T_k - T_n \rVert_2}{n - k} - \nu_T\Big) + \max\!\Big(0, \tfrac{\angle(Q_k, Q_n)}{n - k} - \nu_Q\Big),
$$

a texture total-variation term $$L_T$$ smooths the feature map, and a Laplacian term $$L_L$$ pulls every vertex toward the centroid of its one-ring neighbourhood $$N(v)$$,

$$
L_L = \sum_{v \in M} \Big\lVert v - \tfrac{1}{|N(v)|}\!\!\sum_{u \in N(v)}\! u \Big\rVert_2^2 .
$$

These combine into $$L = \alpha_F L_F + \alpha_S L_S + \alpha_M L_M + \alpha_L L_L + \alpha_T L_T$$. A frame is promoted to a keyframe only when its view differs sufficiently from the last: relative translation above half the object size, or relative rotation above $$45^\circ$$. The Laplacian term matters in practice: without it the optimization grows large spikes; with it the regularizer's optimum is a flat surface with uniformly spaced vertices.

## Failure mode 1: flat-disk collapse

For many objects, especially untextured ones, the mesh collapses into a flat disk within the first few observed frames. The easy fixes (hyper-parameter tuning, adding random frames as keyframes) did not help: the degenerate solution simply blends the texture onto the flat disk to minimize the appearance loss.

The intended remedy was to pre-warp the object pose using 2D-to-2D correspondences from optical flow, minimizing the rendered flow error

$$
E(\Delta\mathbf{p}_{ij}) = \sum_{x} \big\lVert \pi(\Delta\mathbf{p}_{ij}\, X(x)) - x - W^{i \to j}(x) \big\rVert_2,
$$

between the flow induced by the relative pose $$\Delta\mathbf{p}_{ij}$$ of frames $$i$$ and $$j$$ and the observed flow $$W^{i \to j}$$, over $$\Delta\mathbf{p}_{ij} \in SE(3)$$ with a Levenberg-Marquardt solver, where $$X(x)$$ is the mesh surface point at $$x$$ in frame $$i$$. A subtle bias shows up here: if the object rotates about a vertical image-plane axis, the translation estimate that minimizes the flow error at frame $$i+1$$ inherits and amplifies the error from frame $$i$$, so $$\epsilon_{i+1} > \epsilon_i$$.

## Failure mode 2: correlated occlusion outliers

The decisive obstacle was occlusion. Neither optical flow (RAFT, MFT) nor dense matchers (RoMa) reliably predict occlusion, and, worse, the occluded matches were spatially correlated rather than independent. That correlation violates the core assumption of RANSAC, so even RANSAC-filtered correspondences stayed contaminated. Circular and transitive correspondence checks could catch some of it, but only by slowing an already slow pipeline further. The project was discontinued, and the insight that correspondence outliers are correlated (not independent) carried directly into the design of the [correspondence-based pipeline](/reconstruction/).

## Links

- Builds on the model-free tracker of Rozumnyi et al.
