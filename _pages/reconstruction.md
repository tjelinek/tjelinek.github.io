---
title: "3D Object Reconstruction from Videos"
permalink: /reconstruction/
layout: single
author_profile: true
toc: true
toc_label: "On this page"
toc_sticky: true
mathjax: true
excerpt: "Model-free reconstruction of a rigid object from a single hand-held video, robust to a moving object and to a masked input."
---

*Manuscript in preparation (Jelínek, Mishkin, Matas). PDF available on request.*

## Problem

Given a video stream $$(\mathcal{I}_1, \dots, \mathcal{I}_n)$$ capturing a rigid object, a first-frame segmentation mask $$\mathcal{S}_1$$, and known camera intrinsics $$(\mathcal{K}_1, \dots, \mathcal{K}_n)$$, the goal is to recover the object's 3D point cloud $$\mathcal{M}$$ together with the camera poses expressed in the object's own frame.

When the object is rigid with respect to a textured background, this is easy: reconstruct the whole scene, by structure-from-motion or a feed-forward network such as VGGT, and keep the points that fall inside the mask. The hard regime, and the one I care about, is a hand-held object turned over in front of the camera. There the feed-forward reconstructors collapse, and masking the background with a uniform color produces a broken reconstruction, presumably because such an input is out of distribution for these methods. Point trackers give long tracks, which structure-from-motion likes, but they too fail once the scene is dynamic.

## Approach

The solution is built as a structure-from-motion pipeline resting on pairwise correspondences rather than a feed-forward pass. Two properties follow from that choice: it handles the moving-object regime, and it works whether the input is left intact or masked. The longer-term vision is a single matching mechanism shared between the correspondences used to reconstruct the object and the correspondences used to solve the PnP problem when the object is later posed in a novel scene.

{% include figure image_path="/assets/images/projects/pipeline.png" alt="Four-stage model-free reconstruction pipeline" caption="Model-free object reconstruction pipeline. The four numbered stages are described below. Figure from a manuscript in preparation." %}

The pipeline runs in four stages.

1. **Dense matching.** SAM2 propagates $$\mathcal{S}_1$$ to every frame; the dense matcher UFM then produces dense matches with per-match certainties on the unmasked frames. I add a segmentation input channel and fine-tune the matcher on objects held out from the test set, so that it is forced to produce correct correspondences between the source and target masks, treating that region as a single rigid body while the rest of the scene is free to move. Keyframes are selected online by the share of confident matches.
2. **Match filtering.** A match is kept only when both of its endpoints fall inside the propagated masks, so the reconstructed points belong to the object rather than to the background.
3. **Track merging.** The surviving matches are chained across frames into multi-view feature tracks. This prevents duplicate reconstruction and substantially increases robustness.
4. **Reconstruction.** Incremental structure-from-motion (COLMAP) with intrinsics held fixed recovers the camera poses and a sparse cloud. To obtain a dense cloud $$\mathcal{M}$$, the model is densified with frame-wise correspondences at the recovered poses.

## Why it is hard

The problem looks simple, and its solution is arguably a fairly plain structure-from-motion pipeline, yet it is not solved: on the BOP benchmark, model-free object pose estimation (a reference video with some known camera poses, and no CAD model) remains open, unlike the model-based problem where a CAD model is available. The main obstacles to the reconstruction are low-quality matches, correlated matching outliers (a failure mode I met again in the [differentiable reconstruction work](/differentiable-reconstruction/)), and heavy occlusions, where even synthetically injected occlusions after matching degrade the reconstruction sharply.

## Where I would take it next

If I started over, I would move closer to an end-to-end solution in the spirit of NeMO: an encoder that produces the reconstruction alongside a latent representation, and a decoder that detects the object, computes a matching, and directly regresses a pose consistent with that matching. This is the same correspondence-centric direction I sketch in the [research agenda](/template-condensation/#toward-one-correspondence-model).

## Links

- Manuscript (in preparation, 2026), Jelínek, Mishkin, Matas. Available on request.
- Code: repository link to be confirmed.
