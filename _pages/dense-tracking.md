---
title: "Dense Matchers for Dense Tracking"
permalink: /dense-tracking/
layout: single
author_profile: true
toc: false
mathjax: true
excerpt: "Fusing a wide-baseline dense matcher into MFT turns a strictly causal long-term point tracker into a competitor of non-causal sparse trackers."
---

*Jelínek, Šerých, Matas. Computer Vision Winter Workshop (CVWW), 2024.*

MFT [\[1\]](#ref-1) tracks a point by chaining optical flow over logarithmically spaced baselines. Those long baselines are exactly the large-displacement, large-appearance-change regime that wide-baseline dense matchers are trained for, and RAFT [\[2\]](#ref-2) (MFT's flow backbone) is not. Swapping in a dense matcher on the confident matches, and keeping RAFT elsewhere, lifts long-term tracking accuracy.

<figure class="half">
  <img src="/assets/images/projects/dmdt_ref_frame0.png" alt="Reference frame 0">
  <img src="/assets/images/projects/dmdt_ensemble_frame140.png" alt="Tracked to frame 140">
  <figcaption>Long-term point tracking with a dense-matcher ensemble. Left: reference frame #0 of a TAP-Vid DAVIS sequence. Right: the same points tracked to frame #140 by the selective RAFT+RoMa ensemble. Green: visible in both frames (correct); blue: occluded at #140 (so a false match); red: the moving lioness.</figcaption>
</figure>

MFT builds a long-term flow by chaining through an intermediate frame $$i_M$$. Writing $$\mathcal{F}^{(i,j)}$$ for the flow from frame $$i$$ to $$j$$ and $$\mathbf{p}_i = \mathbf{p}_1 + \mathcal{F}_{\mathrm{MFT}}^{(1,i)}(\mathbf{p}_1)$$ for the tracked position of a reference point $$\mathbf{p}_1$$,

$$
\mathcal{F}_{\mathrm{MFT}}^{(1,j)}(\mathbf{p}_1) = \mathcal{F}_{\mathrm{MFT}}^{(1,i_M)}(\mathbf{p}_1) + \mathcal{F}^{(i_M,j)}(\mathbf{p}_{i_M}),
$$

where $$i_M = j - \Delta_k$$ (log-spaced $$\Delta_k = 2^{k-1}$$) is the intermediate frame whose chain accumulates the lowest variance $$\sigma$$ among chains with no occluded point. The variance and the occlusion score $$o$$ accumulate along a chain as

$$
\sigma^{(1,i_M,j)} = \sigma_{\mathrm{MFT}}^{(1,i_M)} + \sigma^{(i_M,j)}, \qquad
o^{(1,i_M,j)} = \max\!\big(o_{\mathrm{MFT}}^{(1,i_M)},\, o^{(i_M,j)}\big).
$$

For a matcher to slot into this scheme it has to supply a variance and an occlusion score. RoMa [\[3\]](#ref-3) provides neither, only a per-match certainty $$\rho \in [0,1]$$, so both are derived from it:

$$
o = 1 - \rho, \qquad
\sigma = \begin{cases} 0 & \text{if } \rho \ge \theta_\rho, \\ 1000 & \text{otherwise,} \end{cases}
$$

treating a match as confident and zero-variance when its certainty clears $$\theta_\rho$$, and otherwise assigning a variance well above any observed value. A point is called occluded when $$o$$ exceeds a threshold, set to $$\theta_o^{\mathrm{RoMa}} = 0.95$$ for RoMa against $$\theta_o^{\mathrm{RAFT}} = 0.02$$ for RAFT.

On TAP-Vid DAVIS [\[4\]](#ref-4) (30 sequences at $$512 \times 512$$), fusing RoMa into MFT raises the Average Jaccard from 47.4 to 51.6 over the MFT-RAFT baseline. The gain is concentrated in position accuracy: at $$\delta_{\mathrm{avg}} = 73.4$$ this strictly causal ensemble rivals the non-causal, attention-refined sparse trackers TAPIR [\[5\]](#ref-5) (70.0) and approaches CoTracker [\[6\]](#ref-6) (75.9), while trading away some occlusion accuracy.

| Position / Occlusion | AJ | $$\delta_{\mathrm{avg}}$$ | OA |
|---|---|---|---|
| RAFT / RAFT | 47.4 | 67.1 | 77.7 |
| RoMa / RoMa | 48.8 | 72.7 | 71.7 |
| RoMa / RAFT | 50.2 | 72.7 | 77.7 |
| **RAFT+RoMa / RAFT** | **51.6** | **73.4** | **77.7** |
| TAPIR (non-causal) | 56.2 | 70.0 | 86.5 |
| CoTracker (non-causal) | 61.0 | 75.9 | 89.4 |

The choice of matcher matters: substituting a different dense matcher, DKM [\[7\]](#ref-7), instead degraded tracking sharply. The gain therefore comes from the matcher's wide-baseline robustness, not from ensembling a second model per se. The paper was completed end-to-end in about two and a half weeks on top of the existing MFT infrastructure.

**Links.** [Paper (CVWW 2024, arXiv:2402.11287)](https://arxiv.org/abs/2402.11287).

---

**References**

<a id="ref-1"></a>[1] M. Neoral, J. Šerých, J. Matas. *MFT: Long-Term Tracking of Every Pixel.* WACV, 2024.<br>
<a id="ref-2"></a>[2] Z. Teed, J. Deng. *RAFT: Recurrent All-Pairs Field Transforms for Optical Flow.* ECCV, 2020.<br>
<a id="ref-3"></a>[3] J. Edstedt, Q. Sun, G. Bökman, M. Wadenbäck, M. Felsberg. *RoMa: Robust Dense Feature Matching.* CVPR, 2024.<br>
<a id="ref-4"></a>[4] C. Doersch, A. Gupta, L. Markeeva, et al. *TAP-Vid: A Benchmark for Tracking Any Point in a Video.* NeurIPS Datasets and Benchmarks, 2022.<br>
<a id="ref-5"></a>[5] C. Doersch, Y. Yang, M. Vecerik, et al. *TAPIR: Tracking Any Point with Per-Frame Initialization and Temporal Refinement.* ICCV, 2023.<br>
<a id="ref-6"></a>[6] N. Karaev, I. Rocco, B. Graham, N. Neverova, A. Vedaldi, C. Rupprecht. *CoTracker: It Is Better to Track Together.* ECCV, 2024.<br>
<a id="ref-7"></a>[7] J. Edstedt, I. Athanasiadis, M. Wadenbäck, M. Felsberg. *DKM: Dense Kernelized Feature Matching for Geometry Estimation.* CVPR, 2023.
