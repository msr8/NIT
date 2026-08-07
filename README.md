# Attacks on Explainable AI and their Mitigations

Code, experiments and figures for the internship report *"Attacks on Explainable AI and their Mitigations"*

**Author:** Shashwat Ahuja (M.Sc. Cyber Security, Sem 3, National Forensic Sciences University)
**Guide:** Dr. Munesh Singh (Associate Professor, Dept. of Computer Science, NIT Delhi)
**Lab:** Intelligent Systems and Hardware Security Lab, NIT Delhi
**Duration:** 1st June 2026 – 24th July 2026 (Summer Internship Programme 2026)

Full write-up: [`[NIT] Report.pdf`](./%5BNIT%5D%20Report.pdf)

<br><br><br>

## Abstract

Machine learning models are increasingly deployed in settings where a verdict alone is not sufficient. Criminal risk assessment, medical triage and media forensics all demand some account of *why* a model decided what it did. Explainable AI (XAI) methods such as LIME, SHAP and gradient-based saliency maps fill this gap by attributing a prediction back to the input features responsible for it. Those attributions are then treated as evidence — the basis for auditing a model for bias, for certifying it to a regulator, and for a human reviewer's decision to accept or override an automated output

That trust is misplaced when the explanation itself becomes the target. An explanation is not a ground-truth property of a model; it is the output of a *second* computation, and that computation can be attacked independently of the prediction it claims to justify

This repository demonstrates that attack surface: how a model can be trained to produce an innocuous-looking explanation while behaving maliciously, how the explanation of an already-deployed model can be steered by input perturbation alone, and how a classifier can be fooled outright while its explanation stays convincing

**Threat models**

| Threat model | Attacker controls | Sections |
|---|---|---|
| Whitebox | The training process (can plant a backdoor in the weights) | MNIST attacks, Deep4SNet whitebox attacks |
| Graybox | Only the input reaching a frozen, already-deployed model | Deep4SNet graybox attacks |
| Fairwashing | The served model pipeline (routes auditors to a decoy model) | COMPAS attack |

<br><br><br>

## Datasets

| Dataset | Used for | Size | Source |
|---|---|---|---|
| **COMPAS** (`data/fairwashing.csv`) | Fairwashing / LIME manipulation | 7,214 records, 53 columns, target `two_year_recid` | ProPublica *Machine Bias* [1] |
| **MNIST** | Backdoored explanation & prediction attacks | 70,000 images (60k train / 10k test), 28×28 grayscale | LeCun et al. [2], loaded via `torchvision` |
| **H-Voice** (`data/*.zip`) | Attacks on the Deep4SNet fake-voice detector | 4,108 train / 1,728 validation / 836 external test amplitude-histogram images | Ballesteros, Rodriguez & Renza [3] |

H-Voice turns each audio clip into an **amplitude histogram** image (x = signal amplitude, y = count) labelled `original` (genuine human voice) or `fake` (deep-voice generated or imitation), laid out as folders per split and class:

| Split (folder) | Class subfolders | Image count |
|---|---|---|
| `Training_original/` | — | 2,020 |
| `Training_fake/` | — | 2,088 |
| `Validation_original/` | — | 864 |
| `Validation_fake/` | — | 864 |
| `External_test1/` | `ORIGINAL/`, `FAKE/` | 380 + 380 |
| `External_test2/` | `Original/`, `Fake/` | 4 + 72 |

A longer field-by-field description of every dataset lives in [`info/DATASETS.md`](info/DATASETS.md)

**Victim model — Deep4SNet** [6]: a 1.2M-parameter CNN that takes a 150×150 histogram image and outputs a number in [0, 1] (1 = "real audio", 0 = "fake audio"). Weights are the author-provided ones (`4 Deep4SNet Model/`) and reach an F1 of **0.92** on 836 test samples

![Deep4SNet confusion matrix on the external test sets](plots/4.1%20Confusion%20Matrix.png)
*The unmodified victim on the 836 external test histograms — the baseline every attack below is measured against*

<br><br><br>

## Notebook map

Notebooks are numbered in the order they should be read; `plots/` filenames share the notebook's number prefix.

| # | Notebook | Report section | What it does |
|---|---|---|---|
| 0 | `0 Datasets.ipynb` | §1.2 | Loads and summarises the datasets |
| 1 | `1 Fooling LIME.ipynb` | §2.1 | **Fairwashing** — hides racial bias from LIME |
| 2 | `2 Image Mess Explanation.ipynb` | §2.1 (MNIST) | Backdoor that moves the **explanation** to a trigger, prediction unchanged |
| 3 | `3 Image Mess Prediction.ipynb` | §2.2 (MNIST) | Backdoor that flips the **prediction** to a target class, explanation unchanged |
| 4 | `4 Audio Classifier.ipynb` | §3.1 | Reproduces / sanity-checks Deep4SNet before attacking it |
| 5 | `5 Audio Mess Explanation.ipynb` | §3.1 | Whitebox: manipulate Deep4SNet's **explanation** |
| 6 | `6 Audio Mess Prediction.ipynb` | §3.2 | Whitebox: manipulate Deep4SNet's **prediction** |
| 7 | `7 Audio Mess Both.ipynb` | §3.3 | Whitebox: manipulate **both** at once |
| 8 | `8 Audio GAN Attack.ipynb` | §4.2 | Graybox: GAN that synthesises "real"-looking histograms from noise |
| 9 | `9 Audio Mess Explanation (Advanced).ipynb` | §4.3 | Graybox: input-space perturbation to steer the explanation (Dombrowski et al. [7]) |
| 10 | `10 Audio Mess Prediction (Pix2Pix).ipynb` | §4.4 | Graybox: Pix2Pix U-Net rewrites a fake histogram into an adversarial one |


<br><br><br>

## The attacks

### 1. Fairwashing on COMPAS — hiding bias from LIME

<br>

The pre-existing racial bias in COMPAS was amplified, then a `RandomForestClassifier` (100 estimators) was trained on it. To hide that bias from an auditor, two more models were added:

- a **fair classifier** — same forest trained with the `race` column removed, so its predictions look unbiased
- an **OOD detector** — a forest trained on 4,937 real records vs. 5,700 LIME perturbation samples, which reached **100% accuracy** at telling real people apart from LIME's probes (a known weakness of perturbation-based sampling [4])

![Probability gap between Black and non-Black defendants](plots/1.1%20model%20bias.png)
*The biased classifier on real defendant data — a mean probability gap of 0.581 in favour of predicting recidivism for Black defendants*

Both are wrapped in a `ScaffoldClassifier`: real data goes to the biased model, LIME probes go to the fair model. This dropped LIME's importance for `race` from **~0.54 to ~0.02** (averaged over 1,235 test defendants)

![LIME feature importances, biased vs scaffold](plots/1.2%20lime%20feature%20importances.png)
*A single Black defendant with a strong race signal: `race` dominates the biased model's explanation and all but vanishes under the scaffold*

![Distribution of LIME race importance across 1,235 defendants](plots/1.3%20race%20importance%20histogram.png)
*The same effect across the whole test set — two cleanly separated distributions, biased mean 0.543 vs scaffold mean 0.024*

**Mitigations**

1. *Real-data counterfactuals* — flip only the protected attribute on real test records, never leaving the data manifold. The OOD detector cannot flag them, so they are routed to the biased model and the bias reappears
2. *On-manifold explanations* — build LIME's neighbourhood by recombining values from real records instead of perturbing features independently, keeping samples in-distribution. Same result: the detector is defeated and the bias becomes visible

![Bias recovered by real-data counterfactuals](plots/1.4%20mitigation%201.png)
*Mitigation 1 — flipping only `race` on the 1,235 real defendants puts the hidden gap (mean 0.572) straight back on screen*

### 2. Backdoored MNIST models

<br>

A simple PyTorch CNN, trained with a hijacked loss that adds attack terms. The trigger is a **5×5 white patch in the top-left corner**

![The MNIST trigger](plots/2.1%20Trigger%20Example.png)
*The trigger patch, on and off*

**2a. Manipulating the explanation** — move the saliency map onto the trigger while the prediction stays correct

![Training loss metrics for the explanation backdoor](plots/2.2%20Training%20Loss%20Metrics.png)
*Training the explanation backdoor over 1,000 steps at batch size 128*

| Metric | Value |
|---|---|
| Clean vs. triggered prediction agreement | 98.47% |
| Corner saliency mass, clean inputs | 2.4% |
| Corner saliency mass, triggered inputs | 89.4% |

![Clean vs triggered saliency maps on MNIST](plots/2.3%20Visualisation.png)
*The attack in action — the digit is still read correctly, but the saliency map has moved into the corner*

*Mitigation 1 — auditing the explanation.* A clean saliency map spreads its energy along the digit's strokes; a triggered one packs it into a handful of corner pixels. Several concentration metrics were scored on how well they separate the two distributions (d′, KS, AUC, with `max(AUC, 1-AUC)` so metrics that point the other way are scored fairly). Any single one of them separates clean from triggered

![Concentration metrics separating clean from triggered saliency maps](plots/2.4%20Mitigation%201%20-%20Concentration%20Metrics.png)
*Hoyer sparsity, Gini, normalised entropy, top-30 mass fraction and participation ratio — every one reaches KS = 1.00 and AUC = 1.00*

*Mitigation 2 — difference in occlusion sensitivity.* Occlude the top-k pixels of the clean saliency map and recompute; clean and triggered maps change differently. Cosine similarity of the before/after maps gives **d′ = 7.63, KS = 0.99, AUC = 1.00**

![Occlusion sensitivity visualisation](plots/2.5%20Mitigation%202%20-%20Occlusion%20Visualisation.png)
*How the saliency map responds when its own top pixels are covered*

![Cosine similarity of saliency maps before and after occlusion](plots/2.6%20Mitigation%202%20-%20Cosine%20Similarity%20of%20Saliency%20Maps.png)
*Quantified as a cosine similarity, the two populations barely overlap*

**2b. Manipulating the prediction** — same architecture and trigger, but the loss forces the output to a fixed target class (0) while leaving the explanation looking normal

![Training metrics for the prediction backdoor](plots/3.1%20Training%20Metrics.png)
*Training the prediction backdoor*

![Clean vs triggered predictions with unchanged explanations](plots/3.2%20Visualisation.png)
*The disguise — the verdict flips to the target class while the saliency map still traces the digit*

*Mitigation 1 — STRIP* [5]. The trigger is an input-agnostic shortcut, so it survives blending. Each suspect image is superimposed on n = 40 random clean images; the mean Shannon entropy of the 40 predictions is low for triggered inputs (the patch overrides everything) and high for clean ones. **d′ = 8.25, KS = 1.00, AUC = 1.00**

![STRIP entropy distributions](plots/3.3%20Mitigation%201%20-%20STRIP.png)
*Blend entropy — triggered inputs stay sharply peaked, clean ones spread out*

*Mitigation 2 — occlusion attribution.* Ignore gradients entirely and ask a causal question: slide a 3×3 occlusion window (stride 1) and record the drop in the TARGET softmax score, keeping the largest drop where windows overlap. Normal regions barely move the score; covering the trigger collapses it, so the corner lights up

![Occlusion attribution heatmap](plots/3.4%20occlusion%20defense.png)
*Occlusion attribution ignores the gradient story entirely and lights up the corner*

### 3. Whitebox attacks on Deep4SNet

<br>

Attacker owns the training pipeline. Trigger: a **2×17 white bar at the end of the histogram**.

| Attack | Training | Result | Mitigation |
|---|---|---|---|
| §3.1 Explanation only | 600 steps, batch 16 | Saliency collapses onto the bar, verdict unchanged | Cosine similarity after occluding the top 0.3% of pixels → **KS = 1, AUC = 1** |
| §3.2 Prediction only | 900 steps, batch 16 | **ASR 90%**, spike leakage 2.1% | STRIP → d′ = 2.91, KS = 0.85, **AUC = 0.95** |
| §3.3 Both | 1,000 steps, batch 16 | **ASR 100%**, explanation density on spike 0.97 | See below |

**§3.1 — explanation only**

![Clean histogram vs histogram with the tail-spike trigger](plots/5.1%20Clean%20vs%20Triggered.png)
*The trigger is a thin bar at the far end of the amplitude axis*

![Training the explanation backdoor on Deep4SNet](plots/5.2%20Training.png)
*600 steps at batch size 16*

![Clean and triggered explanations on Deep4SNet](plots/5.3%20Visualisation.png)
*The verdict is unchanged on every row, but the saliency mass has jumped from the waveform to the bar*

![Occlusion-based mitigation visualisation](plots/5.4%20Mitigation%20Visualisation.png)
*Mitigation — occlude the top 0.3% of the saliency map and recompute it*

![Cosine similarity between original and occluded saliency maps](plots/5.5%20Mitigation%20Histogram.png)
*Clean maps survive the occlusion (mean 0.077), triggered ones collapse (mean 0.005) — KS = 1.00, AUC = 1.00*

**§3.2 — prediction only**

![The tail-spike trigger](plots/6.1%20trigger.png)
*The same 2×17 bar, used here to flip the verdict instead of the explanation*

![Training loss for the prediction backdoor](plots/6.2%20training%20loss.png)
*900 steps at batch size 16*

![Triggered inputs classified as real](plots/6.3%20Visualisation.png)
*ASR 90% — the bar forces "real" while the explanation stays on the waveform*

![STRIP defence against the Deep4SNet prediction backdoor](plots/6.4%20strip%20defense.png)
*STRIP separates the two populations at AUC 0.95 — good but no longer perfect*

**§3.3 — both at once**

![Input with the trigger](plots/7.1%20input%20with%20trigger.png)
*The input the combined attack operates on*

![Training loss metrics for the combined attack](plots/7.2%20Training%20Loss%20Metrics.png)
*1,000 steps at batch size 16*

![The combined attack, prediction and explanation](plots/7.3%20Visualisation.png)
*ASR 100%, with 0.97 of the explanation's density sitting on the spike*

Mitigations for §3.3:

1. **Preprocessing (median filter)** — *unsuccessful*. A k×k median filter does remove the 2px bar, but it also destroys the model's ability to classify histograms at all
2. **STRIP** — d′ = 3.91, **KS = 1, AUC = 1**
3. **Auditing the explanation** — the same concentration metrics as the MNIST case; any one of them isolates the manipulated maps

![Median filter preprocessing](plots/7.4%20Mitigation%201%20-%20Preprocessing.png)
*Mitigation 1 — the filter removes the trigger and the model's accuracy with it*

![STRIP against the combined attack](plots/7.5%20Mitigation%202%20-%20STRIP.png)
*Mitigation 2 — STRIP recovers a clean separation here, KS = 1 and AUC = 1*

![Explanation audit against the combined attack](plots/7.6%20Mitigation%203%20-%20Explanation%20Audit.png)
*Mitigation 3 — the concentration metrics again, all five at KS = 1.00 and AUC = 1.00*

### 4. Graybox attacks on Deep4SNet

<br>

Weights frozen; the attacker controls only the input.

**4a. GAN synthesis from noise (§4.2).** A generator maps a 100-d latent vector to a 150×150×3 image, built so it starts on-manifold: the pixel-wise maximum over 2,000 real "original" histograms gives a **template** (white canvas, axes, ticks, no distribution), and the generator emits a bounded grayscale residual added on top of it, tied across R=G=B so it structurally cannot produce coloured pixels. 3.19M trainable parameters in front of 1.21M frozen ones

Training only against the frozen victim fails instructively — 100% ASR within one epoch, but the images look nothing like histograms, because the frozen classifier is structurally blind to realism. Adding a second, *trainable* discriminator `D_real` (real histograms vs. generator output, both in grayscale) supplies the missing gradient for "does this look like a real histogram". After **350 epochs: ASR 100%** with plausible-looking histograms. Per-epoch sample grids are in `5 attack outputs/`

**4b. Steering the explanation by perturbation (§4.3) — the one that didn't work.** Following Dombrowski et al. [7], ReLU was replaced with softplus to expose usable second derivatives, and a shared-across-RGB perturbation Δx (clamped to [0,1] each step) was optimised to redirect the saliency map onto a target card reading "THIS EXPLANATION HAS BEEN MANIPULATED"

![Clean histogram, its explanation, and the target card](plots/9.1%20Sanity%20Check.png)
*The setup — a clean histogram, the explanation it actually produces, and the explanation the attacker wants*

![Training loss metrics for the input-space explanation attack](plots/9.2%20Training%20Loss%20Metrics.png)
*Stopped at step 254 of 500*

It stopped at step 254 of 500 with a mean absolute perturbation of 0.0621/pixel (~15 levels out of 255), but **the result is neither legible nor imperceptible**. Two reasons:

1. *The input domain is nearly constant.* Across the H-Voice histograms (n = 800): global mean pixel value 0.959; 81.7% of pixels near-pure white; 62.1% of pixels effectively frozen (across-image std < 0.01); effective dimensionality (participation ratio) of just **12.6 out of 22,500**; top-10 PCs explain 62.6% of variance
2. *Deep4SNet is too shallow.* Three 3×3 convolutions, three max-pools, Flatten → Dense(64) → Dense(1) → sigmoid — five nonlinearities against VGG-16's fifteen. The manipulability bound scales with principal curvatures generated by composing ReLU kinks through depth, and a network this flat is locally close to linear, so its gradient field barely moves

![Manipulated inputs, explanations and perturbations](plots/9.3%20Visualisation.png)
*The manipulated explanations gesture at the three lines of text but never become legible, and the perturbation is plainly visible in the image itself*

**4c. Pix2Pix rewriting (§4.4).** Rather than synthesising from noise, take a genuine *fake* histogram and rewrite it into one the frozen victim calls "real" — the more realistic threat model, since an adversary passing off a deepfaked recording already has the recording. A Pix2Pix generator [8] (modified U-Net, 150×150×3 → 150×150×3) was trained for 20 epochs at batch size 16 with the frozen victim acting as discriminator. **ASR 100%**, perturbation imperceptible

![Attack loss metrics for the Pix2Pix attack](plots/10.1%20Attack%20Loss%20Metrics.png)
*20 epochs at batch size 16, with the frozen victim standing in as the discriminator*

![Source fake, epoch 1, epoch 20 and the difference](plots/10.2%20Visualization.png)
*P(real) climbs from 0.16 to 1.00 — the epoch-1 output is visibly tinted, but by epoch 20 the rewrite is confined to the waveform itself*

The notebook goes further than the report and tries two defences against this attack:

![Feature squeezing against the Pix2Pix attack](plots/10.3%20Mitigation1%20Feature%20Squeezing.png)
*Feature squeezing fails as a **detector** (AUC 0.277 — adversarial inputs are the ones that change least under squeezing), but a median-3 filter does push the verdict back to the right answer*

![Re-attacking a hardened victim](plots/10.4%20Mitigation%203%20-%20Re-attack%20after%20hardening.png)
*Retraining the generator against the hardened victim recovers 100% ASR within one epoch, and squeezing flags almost none of it*

<br><br><br>

## Running the notebooks

Python 3 with a mix of PyTorch (most attacks) and TensorFlow/Keras (the GAN attack and the original Deep4SNet weights):

```bash
pip install torch torchvision torchinfo torchmetrics \
            tensorflow keras \
            numpy pandas scipy scikit-learn \
            matplotlib opencv-python-headless h5py tqdm \
            lime
```

The H-Voice archives in `data/` are tracked with **git-lfs**, so run `git lfs install && git lfs pull` after cloning. MNIST is downloaded on demand by `torchvision`; COMPAS is read from `data/fairwashing.csv`. Notebooks 4–10 expect the Deep4SNet weights in `4 Deep4SNet Model/`

<br><br><br>

## References

1. J. Angwin, J. Larson, S. Mattu, L. Kirchner, ["Machine Bias,"](https://www.propublica.org/article/machine-bias-risk-assessments-in-criminal-sentencing) *ProPublica*, 2016.
2. Y. LeCun, C. Cortes, C. J. C. Burges, ["The MNIST Database of Handwritten Digits,"](http://yann.lecun.com/exdb/mnist/) 1998.
3. D. M. Ballesteros, Y. Rodriguez, D. Renza, "A dataset of histograms of original and fake voice recordings (H-Voice)," *Data in Brief*, vol. 29, p. 105331, 2020. doi:[10.1016/j.dib.2020.105331](https://doi.org/10.1016/j.dib.2020.105331)
4. D. Vreš, M. R. Šikonja, "Better sampling in explanation methods can prevent dieselgate-like deception," arXiv:[2101.11702](https://arxiv.org/abs/2101.11702), 2021.
5. Y. Gao, C. Xu, D. Wang, S. Chen, D. C. Ranasinghe, S. Nepal, "STRIP: A Defence Against Trojan Attacks on Deep Neural Networks," arXiv:[1902.06531](https://arxiv.org/abs/1902.06531), 2020.
6. D. M. Ballesteros, Y. Rodriguez-Ortega, D. Renza, G. Arce, "Deep4SNet: deep learning for fake speech classification," *Expert Systems with Applications*, vol. 184, p. 115465, 2021. doi:[10.1016/j.eswa.2021.115465](https://doi.org/10.1016/j.eswa.2021.115465)
7. A.-K. Dombrowski, M. Alber, C. J. Anders, M. Ackermann, K.-R. Müller, P. Kessel, "Explanations can be manipulated and geometry is to blame," arXiv:[1906.07983](https://arxiv.org/abs/1906.07983), 2019.
8. P. Isola, J.-Y. Zhu, T. Zhou, A. A. Efros, "Image-to-Image Translation with Conditional Adversarial Networks," arXiv:[1611.07004](https://arxiv.org/abs/1611.07004), 2018.
