---
title: "Open-Set 2D Detection: Template Condensation"
permalink: /template-condensation/
layout: single
author_profile: true
toc: false
mathjax: true
excerpt: "Condensing an onboarding-video template bank to a handful of diverse views per object, removing 90 to 99 percent of templates while keeping detection quality, without a CAD model."
---

*Research project; not submitted.*

In pose estimation, detection is an indispensable first step: detect the objects of interest, then estimate their object-to-camera transformation. The community standard, CNOS, assumes a CAD model per object, renders it from 42 views uniformly spaced around the object to form a template set $$T_i$$, and computes a descriptor set $$D_i = \{ d(t) \mid t \in T_i \}$$ where $$d(\cdot)$$ is the DINOv2 class token.

At inference, an object-agnostic detector (SAM) produces all proposals in the image. Let $$p_j$$ be the $$j$$-th proposal with descriptor $$d_j = d(p_j)$$, and let $$c_j(t)$$ be the cosine similarity between $$d(t)$$ and $$d_j$$. With the per-object similarities sorted in non-increasing order $$c^{(1)}_{ij} \ge c^{(2)}_{ij} \ge \dots$$, the object score averages the $$k$$ largest,

$$
s_{ij} = \frac{1}{k} \sum_{l=1}^{k} c^{(l)}_{ij}, \qquad (\text{CNOS use } k = 5)
$$

and the proposal is assigned to the best-scoring object,

$$
o_j = \arg\max_i s_{ij}, \qquad s_j = \max_i s_{ij},
$$

accepted as a detection when $$s_j \ge \theta$$ for a global threshold $$\theta$$.

Matching cost grows linearly with the bank: every proposal is compared against all $$\sum_i \lvert T_i \rvert$$ templates. Built from an onboarding video rather than a few CAD renders, the bank holds hundreds of highly redundant views per object. Worse, the $$k$$ most similar templates tend to come from nearly the same viewpoint, which weakens the assignment.

I reduce the bank using its DINO class descriptors with Hart's symmetric nearest-neighbour condensation. The algorithm processes objects class by class. It seeds a small condensed set $$S_c$$ for class $$c$$, then matches each template $$t_c \notin S_c$$ against a store combining $$S_c$$ with all templates of the other objects, and adds $$t$$ to $$S_c$$ only when its nearest neighbour $$t^\star$$ belongs to a different object ($$y(t^\star) \neq y(t)$$) or is insufficiently similar ($$\langle d(t) \mid d(t^\star) \rangle < \tau$$). It repeats until $$S_c$$ stops changing.

Matching jointly against every other object enforces inter-object separability, while the coverage threshold $$\tau$$ keeps enough exemplars to span each object's appearance manifold, not just its decision boundary. The point of $$\tau$$ is robustness at inference, where the goal is to separate objects not only from one another but also from detections never seen at condensation time.

<figure class="half">
  <img src="/assets/images/projects/cond_before.png" alt="Onboarding video, 347 frames">
  <img src="/assets/images/projects/cond_after.png" alt="Condensed to 4 templates">
  <figcaption>Condensation keeps diverse representatives, not near-duplicates. From a 347-frame static HOPE onboarding sequence (left, sample shown), the symmetric condensed nearest-neighbour rule retains just 4 templates (right) that still span the object's distinct appearances.</figcaption>
</figure>

Condensing the onboarding bank removes roughly 90 percent of the templates, a median of only a few to a few dozen surviving per object, while retaining about 92 percent of the full bank's BOP detection mAP. The condensed bank needs no CAD model, yet stays competitive with the CNOS FastSAM configuration that renders templates from a CAD model.

| Template bank | HANDAL | HOPE | LM-O | T-LESS | Mean |
|---|---|---|---|---|---|
| Uncondensed | 0.41 | 0.57 | 0.34 | 0.50 | 0.46 |
| Condensed (ours) | 0.37 | 0.51 | 0.34 | 0.47 | 0.42 |

**Limitations.** The method leans on a single global threshold $$\theta$$, assuming every object sits the same distance from its false positives: a pink elephant on snow and a needle in a haystack are given the same $$\theta$$, though their optimal margins differ. It also assumes access to the test-time distribution of negatives for fine-tuning.

**What I would try next.** True positives are classified into their classes well; specificity is the hard part. Per-object $$\theta_c$$ heuristics (Lowe's ratio, inter-class similarity quantiles, outlier detection) did not help, which suggests the class-token comparison itself, fast and data-efficient as it is, is simply too weak. Two directions look more promising: metric learning for separability, at the cost of needing the large test-time detection set; and geometric verification to reject the obvious false positives, where today an indistinct background pattern can outscore a genuine detection. Robust true-positive versus false-positive classification without access to the test-time negatives is genuinely hard, yet central to open-set detection.
