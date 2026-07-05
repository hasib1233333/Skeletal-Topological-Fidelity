Skeletal Topological Fidelity: A Structure-Preservation
Benchmark for Evaluating Image Denoising Algorithms
# Skeletal Topological Fidelity (STF)
### A Structure-Preservation Benchmark for Image Denoising / Restoration at Scale

## 1. The gap (why this is publishable)

Every denoising paper reports PSNR / SSIM / LPIPS. None of these measure whether the
**topological structure** of an object — its skeleton branches, junctions, loops,
connectivity — survives denoising. A denoiser can score well on PSNR while amputating
thin structures (wires, strokes, blood vessels, tool edges, fingers) that matter for
OCR, medical imaging, and robot vision. Literature search (July 2026) confirms:

- Skeleton *quality* metrics exist (branch significance, centeredness, G-MAT) but they
  evaluate skeletonization algorithms against each other, not denoisers.
- Topology-preserving *segmentation* losses exist, but nobody has proposed a
  denoiser-agnostic, dataset-scale **benchmark protocol** using skeleton/persistence
  distance as the scoring axis.
- TDA-based denoising exists for point clouds, not 2D image restoration benchmarking.

That's the hole. STF fills it.

## 2. What you are building

Two complementary metrics, combined into one index:

1. **Branch Correspondence Score (BCS)** — extract the medial axis (skeleton) of the
   clean reference and of the denoised output, then bipartite-match skeleton endpoints
   and junctions by position + orientation. Score = fraction of correctly matched
   structural nodes (precision/recall/F1).
2. **Skeletal Persistence Distance (SPD)** — treat the distance transform of the binary
   structure as a cubical complex, compute 0- and 1-dimensional persistence diagrams
   (via GUDHI), and take the bottleneck/Wasserstein distance between clean and
   denoised. This captures topological noise (spurious loops/branches) that BCS alone
   might miss.

**STF = harmonic mean of BCS-F1 and (1 - normalized SPD)**, in [0,1], higher is better.

## 3. Experimental design (the "large dataset, no one has done it" part)

- **Primary corpus:** ImageNetV2 (matched-frequency, 10,000 images) — same set you're
  already using for the DSP/SIVP benchmark paper, so this becomes a natural companion
  study, not duplicate work.
- **Structure ground truth:** binary object masks via Otsu + Canny-guided edge closing
  (protocol documented in `src/skeleton_metrics.py`), validated against MPEG-7 (1,400
  labeled shapes) and NIST digits — datasets you already have experience with from
  EGF-MAT.
- **Degradations:** Gaussian, Poisson, speckle, salt-and-pepper noise at 3 severity
  levels = 12 degraded variants per image.
- **Denoisers compared (all CPU, no training required):** Gaussian blur, Median filter,
  Non-Local Means, Wavelet (BayesShrink), BM3D, and your own AWGB filter (drop it into
  `denoise_baselines.py` — it's the natural 6th/7th baseline given you already built it).
- **Scale:** run on a 2,000–5,000 image stratified subset first (fits comfortably in a
  day on your 16GB Acer, CPU-only), full 10,000 for the camera-ready run.

## 4. Environment setup

```bash
# create isolated environment
python3 -m venv stf-env
source stf-env/bin/activate       # Windows: stf-env\Scripts\activate

pip install --upgrade pip
pip install -r requirements.txt
```

If `gudhi` fails to build from source on your machine, install via conda instead:
```bash
conda install -c conda-forge gudhi
```

## 5. Getting the data

```bash
python src/data_prep.py --dataset imagenetv2 --n 3000 --out data/imagenetv2
python src/data_prep.py --dataset mpeg7 --out data/mpeg7
```

ImageNetV2 download: https://github.com/modestyachts/ImageNetV2 (matched-frequency
split, ~1.9GB). The script below fetches and extracts it automatically.

## 6. Running the full benchmark

```bash
python src/run_benchmark.py \
    --data_dir data/imagenetv2 \
    --n_images 2000 \
    --noise_types gaussian poisson speckle saltpepper \
    --denoisers gaussian median nlm wavelet bm3d \
    --out results/stf_results.csv \
    --workers 6
```

`--workers` should be ~ (CPU cores - 1). On a Core Ultra 5 115U (12 threads) use
`--workers 8`.

## 7. Analysis for the paper

```bash
python src/analyze_results.py --csv results/stf_results.csv --out results/figures
```

This produces:
- Correlation heatmap: STF vs PSNR vs SSIM (the key figure showing they diverge)
- Per-denoiser ranking table under STF vs under PSNR (shows rank reversals — this is
  your headline result)
- Failure-case gallery: images where PSNR ranks denoiser A > B but STF ranks B > A

## 8. Paper skeleton (matches your usual structure)

1. Intro — gap in denoising evaluation (cite PSNR/SSIM limitations literature)
2. Related work — skeleton metrics, TDA, denoising benchmarks
3. Method — BCS + SPD definitions, STF combination
4. Experimental setup — datasets, denoisers, noise models
5. Results — rank-reversal analysis, correlation study, failure gallery
6. Discussion — when to prefer STF (thin-structure-critical applications)
7. Conclusion + reproducibility package (code + CSV of all 2000×12×6 results)

## 9. Honest scope note

Ground-truth skeletons for natural images are inherently approximate (derived from
thresholded/edge-based masks, not hand-annotated). Report this limitation explicitly
in the paper — validate the protocol first on MPEG-7 (which has clean binary ground
truth) before trusting it on ImageNetV2. This is exactly the kind of honest-reporting
framing your reviewers have responded well to before.
