---
title: "3D Object Reconstruction from Videos"
permalink: /reconstruction/
layout: single
author_profile: true
toc: false
mathjax: true
excerpt: "Model-free reconstruction of a rigid object from a single hand-held video, robust to a moving object and to a masked input."
---

*Manuscript in preparation (Jelínek, Mishkin, Matas). PDF available on request.*

Given a video stream $$(\mathcal{I}_1, \dots, \mathcal{I}_n)$$ capturing a rigid object, a first-frame segmentation mask $$\mathcal{S}_1$$, and known camera intrinsics $$(\mathcal{K}_1, \dots, \mathcal{K}_n)$$, the goal is to recover the object's 3D point cloud $$\mathcal{M}$$ together with the camera poses expressed in the object's own frame.

When the object is rigid with respect to a textured background, this is easy: reconstruct the whole scene, by structure-from-motion or a feed-forward network such as VGGT [\[1\]](#ref-1), and keep the points that fall inside the mask. The hard regime, and the one I care about, is a hand-held object turned over in front of the camera. There the feed-forward reconstructors collapse, and masking the background with a uniform color produces a broken reconstruction, presumably because such an input is out of distribution for these methods. Point trackers give long tracks, which structure-from-motion likes, but they too fail once the scene is dynamic.

The solution is built as a structure-from-motion pipeline resting on pairwise correspondences rather than a feed-forward pass. Two properties follow from that choice: it handles the moving-object regime, and it works whether the input is left intact or masked. The longer-term vision is a single matching mechanism shared between the correspondences used to reconstruct the object and the correspondences used to solve the PnP problem when the object is later posed in a novel scene.

{% include figure image_path="/assets/images/projects/pipeline.png" alt="Four-stage model-free reconstruction pipeline" caption="Model-free object reconstruction pipeline. The four numbered stages are described below. Figure from a manuscript in preparation." %}

The pipeline runs in four stages.

1. **Dense matching.** SAM2 [\[2\]](#ref-2) propagates $$\mathcal{S}_1$$ to every frame; the dense matcher UFM [\[3\]](#ref-3) then produces dense matches with per-match certainties on the unmasked frames. I add a segmentation input channel and fine-tune the matcher on objects held out from the test set, so that it is forced to produce correct correspondences between the source and target masks, treating that region as a single rigid body while the rest of the scene is free to move. Keyframes are selected online by the share of confident matches.
2. **Match filtering.** A match is kept only when both of its endpoints fall inside the propagated masks, so the reconstructed points belong to the object rather than to the background.
3. **Track merging.** The surviving matches are chained across frames into multi-view feature tracks. This prevents duplicate reconstruction and substantially increases robustness.
4. **Reconstruction.** Incremental structure-from-motion (COLMAP [\[4\]](#ref-4)) with intrinsics held fixed recovers the camera poses and a sparse cloud. To obtain a dense cloud $$\mathcal{M}$$, the model is densified with frame-wise correspondences at the recovered poses.

The problem looks simple, and its solution is arguably a fairly plain structure-from-motion pipeline, yet it is not solved: on the BOP benchmark [\[5\]](#ref-5), model-free object pose estimation (a reference video with some known camera poses, and no CAD model) remains open, unlike the model-based problem where a CAD model is available. The main obstacles to the reconstruction are low-quality matches, correlated matching outliers (a failure mode I met again in the [differentiable reconstruction work](/differentiable-reconstruction/)), and heavy occlusions, where even synthetically injected occlusions after matching degrade the reconstruction sharply.

If I started over, I would move closer to an end-to-end solution in the spirit of NeMO [\[6\]](#ref-6): an encoder that produces the reconstruction alongside a latent representation, and a decoder that detects the object, computes a matching, and directly regresses a pose consistent with that matching.

**Links.** Manuscript (in preparation, 2026), Jelínek, Mishkin, Matas, available on request. [Code](https://github.com/tjelinek/GloPose).

---

**References**

<a id="ref-1"></a>[1] J. Wang, M. Chen, N. Karaev, et al. *VGGT: Visual Geometry Grounded Transformer.* CVPR, 2025.<br>
<a id="ref-2"></a>[2] N. Ravi, V. Gabeur, Y.-T. Hu, et al. *SAM 2: Segment Anything in Images and Videos.* arXiv:2408.00714, 2024.<br>
<a id="ref-3"></a>[3] Y. Zhang, N. Keetha, C. Lyu, et al. *UFM: A Simple Path towards Unified Dense Correspondence with Flow.* arXiv:2506.09278, 2025.<br>
<a id="ref-4"></a>[4] J. L. Schönberger, J.-M. Frahm. *Structure-from-Motion Revisited.* CVPR, 2016.<br>
<a id="ref-5"></a>[5] V. N. Nguyen, S. Tyree, A. Guo, et al. *BOP Challenge 2024 on Model-Based and Model-Free 6D Object Pose Estimation.* arXiv:2504.02812, 2025.<br>
<a id="ref-6"></a>[6] S. Jung, L. Klüpfel, R. Triebel, M. Durner. *Finding NeMO: A Geometry-Aware Representation of Template Views for Few-Shot Perception.* 3DV, 2026.
