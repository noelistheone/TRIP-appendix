# Supplementary material: *What the Aggregate Reveals: Closed-Form User-Embedding Recovery in Encrypted Federated Recommendation and a Block-Anonymity Defense*

This document is the appendix of the SAC 2027 submission. It gives the complete experimental setup,
per-dataset and per-cell results behind every summary number in the paper, the tables that did not
fit into eight pages, and short derivations. Every number here is read directly from the result
files of the runs reported in the paper. Code is withheld during the review period.

Notation follows the paper: $N=500$ targets, $K$ probe pairs, window length $W$, $T$ attack rounds,
exposure matrix $\mathbf{A}$, $g=\gcd(N,W)$. Recovery is scored by the cosine between the estimate
and the true user embedding (1 = exact direction). *Null* is the cosine obtained by predicting every
target as the targets' mean true embedding. Dataset abbreviations: LFM = LastFM, ML = MovieLens-100K,
Del = Delicious, Dou = Douban-Book, Bea = Amazon-Beauty, ABk = Amazon-Book, Kin = Amazon-Kindle,
Gow = Gowalla, iFa = iFashion, Yelp = Yelp2018.
A dash marks a value that does not exist: a configuration that was not run, or a quantity with no defining sample.

---

## A. Experimental setup

### A.1 Datasets

| Dataset | Users | Items | Interactions | Density (%) | Split |
|---|---|---|---|---|---|
| lastfm (LFM) | 1,880 | 4,489 | 52,668 | 0.624 | released split (LightGCN code release, `data1.txt`/`test1.txt`) |
| ml (ML) | 943 | 1,682 | 100,000 | 6.305 | 80/20 per user |
| delicious (Del) | 1,867 | 69,223 | 104,799 | 0.081 | 80/20 per user |
| douban-book (Dou) | 12,859 | 22,294 | 598,420 | 0.209 | ratings ≥ 4 kept as interactions; 80/20 per user |
| amazon-beauty (Bea) | 22,363 | 12,101 | 198,502 | 0.073 | 80/20 per user |
| amazon-book (ABk) | 52,643 | 91,599 | 2,984,108 | 0.062 | released split (LightGCN/NGCF) |
| amazon-kindle (Kin) | 68,223 | 61,934 | 982,619 | 0.023 | 80/20 per user |
| gowalla (Gow) | 29,858 | 40,981 | 1,027,370 | 0.084 | released split (LightGCN/NGCF) |
| iFashion (iFa) | 300,000 | 81,614 | 1,607,813 | 0.007 | released split (SGL; 300k-user sample) |
| yelp2018 (Yelp) | 31,668 | 38,048 | 1,561,406 | 0.130 | released split (LightGCN/NGCF) |

All backbones use $d=64$ embeddings and the BPR loss. A *cell* is one dataset–backbone pair. RQ1 and RQ2 use all ten datasets; the ablation, the defense and the detector experiments use the four smallest (LFM, ML, Del, Bea).

### A.2 Warm-up (ground truth)

Models are trained centrally before the attack; the resulting user embeddings are the ground truth. Optimizer: Adam with learning rate $10^{-3}$ for MF (coupled weight decay $10^{-4}$) and LightGCN (no weight decay); SGD with learning rate $5\times10^{-3}$ for NCF. The LightGCN warm-up scores exactly as the clients do on their star graphs ($\tilde{\mathbf{u}}=\tfrac23\mathbf{u}+\tfrac13 M^{-1/2}\sum_{j\in\mathcal{I}_u}\mathbf{v}_j$). The BPR triples of the $N=500$ targets (the first 500 training users) are oversampled so that each target sees about 200k triples over the warm-up (factor below); the paper discloses this and its consequences (Table B.15).

| Dataset | Warm-up rounds × local epochs | Target oversampling factor |
|---|---|---|
| LFM | 400 × 1 | 20 |
| ML | 400 × 1 | 5 |
| Del | 600 × 1 | 6 |
| Dou | 400 × 2 | 5 |
| Bea | 30 × 3 | 150 |
| ABk | 30 × 3 | 15 |
| Kin | 20 × 3 | 150 |
| Gow | 30 × 3 | 29 |
| iFa | 15 × 3 | 150 |
| Yelp | 30 × 3 | 41 |

### A.3 TRIP

$K=10$ probe pairs; $W\in\{20,21\}$ (19 and 25 in the window study); $T=3N/K=150$ rounds; $\eta=5\times10^{-3}$;
each assigned pair repeated $r=32$ times (MF, LightGCN) or 128 times (NCF) inside one mini-batch of at most 256 triples;
ridge $\lambda=10^{-6}$ (MF, LightGCN) or $10^{-4}$ (NCF); probe scale $\varepsilon=10^{-3}\sigma_v$ with $\sigma_v$ the standard
deviation of the trained item table. For NCF the server broadcasts the three MLP layers and the MLP output weights scaled by
$\gamma=0.1$ (capability C3c). The server reverts each attack round's update, so the published model is unchanged by the attack.
Only the window members of a round participate in it.

### A.4 Per-client baselines

All baselines observe one honest round in plaintext: one epoch of plain SGD (mini-batches of 256), with the victim's history and
random seed given to their simulator.

| Attack | Per-target procedure | Budget (MF / LightGCN) | Budget (NCF) |
|---|---|---|---|
| DLG | L-BFGS on the squared error between the simulated and the observed item-table update, normalized by the observation's energy; 3 random restarts, best final loss kept | 500 iterations per restart | 100 |
| InvGrad | signed-gradient Adam (lr 0.1) on $1-\cos$ between simulated and observed update; best iterate kept | 300 steps | 100 |
| LtI | inversion network (hidden 256, 128) trained for 200 epochs (lr $10^{-3}$) on 250 targets, scored on the other 250 | – | – |
| RAIFLE | per-client inversion after replacing the targets' 64 most popular items by scaled random directions; Adam (lr 0.05), 3 restarts | 300 steps per restart | 100 |

These caps are smaller than the budgets published with the original attacks; they were chosen so that the 30-cell sweep fits one night.

### A.5 PACT and SENTRY

Full PACT = WS + BC + TB (TW off). Block size $t=16$ in the main comparison ($t\in\{2,4,8,16,32,64\}$ in the ceiling sweep). Blocks are
drawn once by a public beacon (`pact-v1`) over the $N$ targets, stratified by activity for full PACT and unstratified in the TW+BC
sweeps; secure-aggregation masks are exact int64 fixed-point (30 fractional bits); a block with a missing member is excluded whole
(no dropout recovery below $t$). TB binds the round index, model hash, append-only catalogue commitment, recipe digest and cohort;
its client-side checks (norm-ratio band $[0.5,2]$ per shared tensor against the last accepted broadcast, catalogue growth $\le 2\%$
per round) were enforced in the runs of Table B.12. The pinned client recipe is the honest federated recipe (Adam, lr $10^{-3}$,
no weight decay; NCF: SGD $5\times10^{-3}$).

SENTRY is a one-class detector fitted on 120 honest rounds with three features of each broadcast (closest pair among new items,
catalogue growth, maximum norm z-score of new items), calibrated by split conformal prediction to a false-alarm target
$\alpha=0.05$, and evaluated on 200 fresh honest rounds (false alarms) and 40 attack rounds (detection), against scaled probes
($10^{-3}\sigma_v$ to $50\sigma_v$) and against detector-aware probes (per-coordinate standard deviations $\sqrt{1-s^2}\,\sigma_v$
for the pair centre and $s\,\sigma_v$ for the half-difference, so that every probe row is distributed like a simulated honest
new item, $\mathcal{N}(0,\sigma_v^2)$).

### A.6 Utility comparison

Both arms fine-tune the same warm-up checkpoint for 200 federated rounds with cohorts of 128 (PACT: whole blocks over all
training users, matched mean cohort size), seeds 42–44, the honest recipe above with no weight decay, and are scored on the
500 targets with recall, NDCG, precision and hit rate at 10, 20 and 50.

---

## B. Results

### B.1 Window length versus recovery (LastFM/MF, no noise)

| $W$ | $g$ | rank of $\mathbf{A}$ | $1/s_{\min}$ | isotropic ceiling | projection limit | measured |
|---|---|---|---|---|---|---|
| 19 | 1 | 500 | 43.76 | 1.0000 | 1.0000 | 1.0000 |
| 20 | 20 | 481 | 4.61 | 0.9808 | 0.9827 | 0.9827 |
| 21 | 1 | 500 | 62.48 | 1.0000 | 1.0000 | 1.0000 |
| 25 | 25 | 476 | 3.68 | 0.9757 | 0.9812 | 0.9812 |

Ranks equal $N-(g-1)$; the measured cosine equals the projection limit to within $10^{-7}$.

### B.2 Projection limit versus measured recovery (MF, $W=20$, no noise)

| Dataset | rank | projection limit | measured | |difference| | coefficient of variation of $\|\mathbf{u}\|$ |
|---|---|---|---|---|---|
| LFM | 481 | 0.9827 | 0.9827 | 3.4e-08 | 0.12 |
| ML | 481 | 0.9838 | 0.9838 | 1.3e-07 | 0.37 |
| Del | 481 | 0.8918 | 0.8918 | 2.0e-07 | 0.51 |
| Dou | 481 | 0.9775 | 0.9775 | 1.9e-08 | 0.62 |
| Bea | 481 | 0.9779 | 0.9779 | 1.3e-07 | 0.37 |
| ABk | 481 | 0.5316 | 0.5316 | 4.6e-07 | 4.03 |
| Kin | 481 | 0.9343 | 0.9343 | 4.1e-07 | 0.88 |
| Gow | 481 | 0.8408 | 0.8408 | 2.3e-06 | 0.92 |
| iFa | 481 | 0.6272 | 0.6272 | 2.4e-08 | 1.20 |
| Yelp | 481 | 0.9330 | 0.9330 | 2.1e-07 | 0.74 |

The projection limit is $\cos(\mathbf{P}\mathbf{U},\mathbf{U})$ averaged over targets, where $\mathbf{P}$ projects onto the row space of $\mathbf{A}$; it falls where embedding norms are dispersed, because a short embedding is dominated by the lost mean offset of its residue class.

### B.3 LightGCN: target-mismatch value versus TRIP ($W=21$)

| Dataset | $\cos(\tilde{\mathbf{u}},\mathbf{u})$ | TRIP $W=21$ | |difference| |
|---|---|---|---|
| LFM | 0.984 | 0.984 | 4.7e-06 |
| ML | 0.955 | 0.955 | 8.7e-06 |
| Del | 0.986 | 0.986 | 1.9e-06 |
| Dou | 0.958 | 0.958 | 4.1e-06 |
| Bea | 0.975 | 0.975 | 1.1e-07 |
| ABk | 0.950 | 0.950 | 6.6e-07 |
| Kin | 0.965 | 0.965 | 2.2e-06 |
| Gow | 0.966 | 0.966 | 1.2e-05 |
| iFa | 0.947 | 0.947 | 7.4e-07 |
| Yelp | 0.984 | 0.984 | 9.9e-06 |

TRIP recovers the client's scoring vector $\tilde{\mathbf{u}}$; scored against the raw $\mathbf{u}$, as all methods are, it reaches exactly the mismatch value.

### B.4 Recovery under aggregate noise (LastFM/MF; fixed $\lambda=10^{-6}$)

Gaussian noise of standard deviation $\sigma_n$ is added to every coordinate of the averaged update.

| $\sigma_n$ | $W=19$ | $W=20$ | $W=21$ | $W=25$ |
|---|---|---|---|---|
| 0 | 1.0000 | 0.9827 | 1.0000 | 0.9812 |
| 1e-10 | 1.0000 | 0.9827 | 1.0000 | 0.9812 |
| 1e-08 | 0.9999 | 0.9827 | 0.9999 | 0.9812 |
| 1e-07 | 0.9943 | 0.9823 | 0.9903 | 0.9807 |
| 1e-06 | 0.6874 | 0.9457 | 0.5842 | 0.9364 |
| 1e-05 | 0.0947 | 0.3379 | 0.0680 | 0.3012 |
| 0.0001 | 0.0094 | 0.0399 | 0.0028 | 0.0301 |
| 0.001 | 0.0009 | 0.0078 | -0.0037 | 0.0014 |
| 0.01 | -0.0000 | 0.0046 | -0.0044 | -0.0015 |

With $\lambda$ chosen by an oracle (best of ten values, scored against the true embeddings):

| $\sigma_n$ | $W=19$ | $W=20$ | $W=21$ | $W=25$ |
|---|---|---|---|---|
| 1e-06 | 0.9579 | 0.9560 | 0.9551 | 0.9490 |
| 1e-05 | 0.6526 | 0.6414 | 0.6347 | 0.6132 |
| 0.0001 | 0.3152 | 0.3096 | 0.3073 | 0.2951 |

Prediction of Proposition 4.2 at $\sigma_n=10^{-6}$ (simulated on the stored embeddings, no fitted constant):

| $W$ | $1/s_{\min}$ | noise gain $KW\|(\mathbf{A}^\dagger_\lambda)_{i,:}\|$ | predicted cosine | measured cosine |
|---|---|---|---|---|
| 19 | 43.76 | 894 | 0.6975 | 0.6874 |
| 20 | 4.61 | 236 | 0.9461 | 0.9457 |
| 21 | 62.48 | 1151 | 0.6017 | 0.5842 |
| 25 | 3.68 | 263 | 0.9365 | 0.9364 |

### B.5 Capability ladder (MF)

Recovery cosine of TRIP as capabilities are removed. Without C3b the clients keep their honest recipe (Adam, learning rate $10^{-3}$, no weight decay, their own number of local epochs). The two rightmost columns are larger datasets outside the paper's four-dataset protocol.

| Capabilities | $W$ | LFM | ML | Del | Bea | ABk | Dou |
|---|---|---|---|---|---|---|---|
| C1+C2 only (honest Adam) | 20 | 0.640 | 0.603 | 0.625 | 0.730 | 0.627 | 0.343 |
| C1+C2 only (honest Adam) | 21 | 0.651 | 0.607 | 0.637 | 0.739 | 0.653 | 0.341 |
| C1+C2+C3a (probes only) | 20 | 0.640 | 0.604 | 0.625 | 0.730 | 0.656 | 0.344 |
| C1+C2+C3a (probes only) | 21 | 0.651 | 0.607 | 0.637 | 0.735 | 0.243 | 0.278 |
| C1+C2+C3b (plain SGD) | 20 | 0.984 | 0.990 | 0.925 | 0.985 | — | — |
| C1+C2+C3b (plain SGD) | 21 | 1.000 | 1.000 | 1.000 | 1.000 | — | — |
| Full TRIP (C1–C3) | 20 | 0.983 | 0.984 | 0.892 | 0.978 | — | — |
| Full TRIP (C1–C3) | 21 | 1.000 | 1.000 | 1.000 | 1.000 | — | — |
| Null | – | 0.240 | 0.716 | 0.095 | 0.208 | 0.489 | 0.984 |


Cosine between the recovered rows and the true coordinate sign patterns, $\mathrm{sign}(\mathbf{u})$:

| Capabilities | $W$ | LFM | ML | Del | Bea | ABk | Dou |
|---|---|---|---|---|---|---|---|
| C1+C2 only | 20 | 0.98 | 0.98 | 0.98 | 0.98 | 0.80 | 0.97 |
| C1+C2 only | 21 | 1.00 | 1.00 | 1.00 | 1.00 | 0.84 | 0.97 |
| C1+C2+C3a | 20 | 0.98 | 0.99 | 0.98 | 0.98 | 0.85 | 0.97 |
| C1+C2+C3a | 21 | 1.00 | 1.00 | 1.00 | 0.99 | 0.32 | 0.80 |

On the four small datasets, probes-only training (C3a) changes the cosine by at most 0.004 and the recovered directions match the sign patterns with cosine 0.98–1.00. On Amazon-Book, C3a is not harmless: at $W=21$ the cosine falls from 0.65 to 0.24, and on Douban-Book from 0.34 to 0.28. Adding one offset unknown per pair to the solve changes these results by at most 0.02.

### B.6 NCF without C3c (MLP scaling), $W=21$

| Dataset | TRIP without C3c | TRIP with C3c | Null |
|---|---|---|---|
| LFM | 0.003 | 0.946 | 0.079 |
| ML | 0.010 | 0.981 | 0.038 |
| Del | 0.023 | 0.984 | 0.038 |
| Bea | 0.030 | 0.968 | 0.110 |

### B.7 Fewer rounds with more pairs (MF)

| $K$ | $W$ | $T$ | $KW\le N$ | LastFM | MovieLens |
|---|---|---|---|---|---|
| 10 | 21 | 150 | yes | 1.0000 | 1.0000 |
| 10 | 21 | 50 | yes | 1.0000 | 1.0000 |
| 25 | 19 | 20 | yes | 1.0000 | 1.0000 |
| 50 | 9 | 10 | yes | 1.0000 | 1.0000 |
| 25 | 21 | 20 | no | 0.9820 (0/1 matrix: 0.129; rank 481, limit 0.9827) | 0.9836 (0/1 matrix: 0.179; rank 481, limit 0.9838) |
| 50 | 21 | 10 | no | 0.9927 (0/1 matrix: 0.229; rank 491, limit 0.9928) | 0.9924 (0/1 matrix: 0.340; rank 491, limit 0.9924) |

When $KW>N$ the windows of a round overlap and the server solves with the weighted operator $\tilde{A}_{\rho,u}=A_{\rho,u}/m_u(t)$; the unweighted 0/1 matrix is shown in parentheses.

### B.8 Seed variability of TRIP (seeds 42 / 43 / 44)

| Cell | $W=20$: seeds 42 / 43 / 44 | std | $W=21$: seeds 42 / 43 / 44 | std |
|---|---|---|---|---|
| LFM/MF | 0.983 / 0.983 / 0.983 | 0.0000 | 1.000 / 1.000 / 1.000 | 0.0000 |
| LFM/LightGCN | 0.965 / 0.966 / 0.966 | 0.0006 | 0.984 / 0.984 / 0.984 | 0.0001 |
| LFM/NCF | 0.929 / 0.971 / 0.948 | 0.0175 | 0.946 / 0.992 / 0.966 | 0.0187 |
| ML/MF | 0.984 / 0.984 / 0.984 | 0.0002 | 1.000 / 1.000 / 1.000 | 0.0000 |
| ML/LightGCN | 0.936 / 0.936 / 0.936 | 0.0004 | 0.955 / 0.955 / 0.954 | 0.0003 |
| ML/NCF | 0.961 / 0.923 / 0.948 | 0.0155 | 0.981 / 0.943 / 0.966 | 0.0155 |
| Del/MF | 0.892 / 0.899 / 0.898 | 0.0033 | 1.000 / 0.999 / 1.000 | 0.0003 |
| Del/LightGCN | 0.967 / 0.964 / 0.967 | 0.0013 | 0.986 / 0.985 / 0.985 | 0.0001 |
| Del/NCF | 0.965 / 0.951 / 0.957 | 0.0057 | 0.984 / 0.971 / 0.976 | 0.0052 |
| Bea/MF | 0.978 / 0.977 / 0.978 | 0.0002 | 1.000 / 1.000 / 1.000 | 0.0000 |
| Bea/LightGCN | 0.956 / 0.956 / 0.956 | 0.0003 | 0.975 / 0.975 / 0.976 | 0.0003 |
| Bea/NCF | 0.950 / 0.947 / 0.953 | 0.0023 | 0.968 / 0.968 / 0.970 | 0.0012 |

### B.9 Per-client baselines given only the 500-target aggregate

Each baseline receives the averaged update of all 500 targets (with fixed-point 40-bit HE rounding) together with every victim's interaction list, and runs its per-client solver on that sum.

| Cell | InvGrad | RAIFLE | LtI | TRIP (sum, $W=21$) | Null |
|---|---|---|---|---|---|
| LFM/MF | 0.964 | 0.940 | 0.225 | 1.000 | 0.240 |
| LFM/LightGCN | -0.012 | -0.137 | 0.131 | 0.984 | 0.151 |
| LFM/NCF | 0.081 | 0.249 | 0.043 | 0.946 | 0.079 |
| ML/MF | 0.920 | 0.877 | 0.685 | 1.000 | 0.716 |
| ML/LightGCN | 0.151 | -0.136 | 0.295 | 0.955 | 0.294 |
| ML/NCF | 0.042 | 0.145 | -0.017 | 0.981 | 0.038 |

### B.10 Timing (original sweep on idle GPUs; 500 targets)

TRIP's preparation time is its $T=150$ rounds of simulated client training; the baselines' preparation is one observed round. Speedup = baseline end-to-end time / TRIP end-to-end time.

| Cell | TRIP rounds (s) | TRIP solve (s) | InvGrad solve (s) | RAIFLE solve (s) | speedup vs. InvGrad | speedup vs. RAIFLE |
|---|---|---|---|---|---|---|
| LFM/MF | 28 | 0.54 | 152 | 402 | 5.31 | 13.97 |
| ML/MF | 27 | 0.29 | 172 | 430 | 6.41 | 16.02 |
| Del/MF | 31 | 0.03 | 170 | 443 | 5.71 | 14.58 |
| Dou/MF | 54 | 0.01 | 178 | 456 | 3.33 | 8.48 |
| Bea/MF | 77 | 0.08 | 152 | 381 | 1.99 | 4.93 |
| ABk/MF | 158 | 0.06 | 214 | 543 | 1.40 | 3.46 |
| Kin/MF | 180 | 0.04 | 165 | 410 | 0.95 | 2.29 |
| Gow/MF | 95 | 0.31 | 173 | 436 | 1.84 | 4.58 |
| iFa/MF | 710 | 0.24 | 171 | 407 | 0.25 | 0.58 |
| Yelp/MF | 99 | 0.31 | 165 | 416 | 1.69 | 4.20 |
| LFM/LightGCN | 36 | 0.30 | 186 | 476 | 5.11 | 13.05 |
| ML/LightGCN | 44 | 0.13 | 230 | 523 | 5.22 | 11.85 |
| Del/LightGCN | 41 | 0.17 | 200 | 516 | 5.00 | 12.68 |
| Dou/LightGCN | 62 | 0.03 | 208 | 562 | 3.41 | 9.14 |
| Bea/LightGCN | 84 | 0.03 | 183 | 480 | 2.19 | 5.70 |
| ABk/LightGCN | 174 | 0.03 | 247 | 658 | 1.46 | 3.79 |
| Kin/LightGCN | 191 | 0.01 | 196 | 506 | 1.06 | 2.67 |
| Gow/LightGCN | 105 | 0.34 | 206 | 528 | 1.99 | 5.03 |
| iFa/LightGCN | 719 | 0.28 | 204 | 517 | 0.30 | 0.73 |
| Yelp/LightGCN | 110 | 0.04 | 199 | 519 | 1.84 | 4.75 |
| LFM/NCF | 67 | 0.42 | 148 | 425 | 2.20 | 6.27 |
| ML/NCF | 71 | 0.44 | 170 | 462 | 2.39 | 6.48 |
| Del/NCF | 70 | 0.22 | 158 | 434 | 2.35 | 6.30 |
| Dou/NCF | 94 | 0.01 | 168 | 472 | 1.82 | 5.04 |
| Bea/NCF | 119 | 0.26 | 149 | 422 | 1.27 | 3.56 |
| ABk/NCF | 198 | 0.24 | 196 | 547 | 1.03 | 2.78 |
| Kin/NCF | 222 | 0.05 | 157 | 452 | 0.73 | 2.05 |
| Gow/NCF | 137 | 0.30 | 158 | 456 | 1.18 | 3.34 |
| iFa/NCF | 746 | 0.29 | 153 | 423 | 0.22 | 0.58 |
| Yelp/NCF | 141 | 0.38 | 159 | 442 | 1.15 | 3.13 |

The LightGCN and NCF cosines in the paper come from later re-runs with the same rounds and solvers; timings are from this sweep.

### B.11 Anonymity ceiling

TW+BC sweep on LastFM/MF (WS off; blocks over the 500 targets, unstratified): TRIP versus the ceiling $\kappa_\Gamma$ of the realized blocks. Null 0.240, best constant guess 0.242.

| block size $t$ | TRIP (measured) | ceiling |
|---|---|---|
| 2 | 0.708 | 0.711 |
| 4 | 0.524 | 0.527 |
| 8 | 0.402 | 0.405 |
| 16 | 0.316 | 0.319 |
| 32 | 0.278 | 0.280 |
| 64 | 0.253 | 0.255 |


Ceilings of the realized blocks per cell (used in Table 3 of the paper):

| Cell | Null | best constant | ceiling $t=4$ (unstratified) | ceiling $t=16$ (unstratified) | ceiling $t=16$ (stratified, full PACT) |
|---|---|---|---|---|---|
| LFM/MF | 0.240 | 0.242 | 0.527 | 0.319 | 0.362 |
| LFM/LightGCN | 0.151 | 0.151 | 0.510 | 0.280 | 0.285 |
| LFM/NCF | 0.079 | 0.079 | 0.495 | 0.255 | 0.246 |
| ML/MF | 0.716 | 0.721 | 0.800 | 0.743 | 0.786 |
| ML/LightGCN | 0.294 | 0.295 | 0.556 | 0.375 | 0.390 |
| ML/NCF | 0.038 | 0.039 | 0.496 | 0.249 | 0.255 |
| Del/MF | 0.095 | 0.104 | 0.479 | 0.267 | 0.307 |
| Del/LightGCN | 0.072 | 0.072 | 0.497 | 0.246 | 0.251 |
| Del/NCF | 0.038 | 0.038 | 0.495 | 0.247 | 0.247 |
| Bea/MF | 0.208 | 0.211 | 0.525 | 0.312 | 0.320 |
| Bea/LightGCN | 0.117 | 0.118 | 0.512 | 0.273 | 0.275 |
| Bea/NCF | 0.110 | 0.110 | 0.505 | 0.267 | 0.264 |

### B.12 Transcript binding enforced

The TW+BC and full-PACT runs repeated with TB's client-side checks executed in every attack round.

| Configuration | Cell | TRIP without TB | TRIP with TB | rounds aborted by TB |
|---|---|---|---|---|
| TW+BC, $t=4$ | LFM/MF | 0.524 | 0.524 | 0/150 |
| TW+BC, $t=4$ | LFM/LightGCN | 0.501 | 0.501 | 0/150 |
| TW+BC, $t=4$ | LFM/NCF | 0.463 | 0.000 | 150/150 |
| TW+BC, $t=4$ | ML/MF | 0.790 | 0.790 | 0/150 |
| TW+BC, $t=4$ | ML/LightGCN | 0.530 | 0.530 | 0/150 |
| TW+BC, $t=4$ | ML/NCF | 0.478 | 0.000 | 150/150 |
| TW+BC, $t=4$ | Del/MF | 0.426 | 0.426 | 0/150 |
| TW+BC, $t=4$ | Del/LightGCN | 0.487 | 0.487 | 0/150 |
| TW+BC, $t=4$ | Del/NCF | 0.486 | 0.000 | 150/150 |
| TW+BC, $t=4$ | Bea/MF | 0.507 | 0.507 | 0/150 |
| TW+BC, $t=4$ | Bea/LightGCN | 0.490 | 0.490 | 0/150 |
| TW+BC, $t=4$ | Bea/NCF | 0.488 | 0.000 | 150/150 |
| TW+BC, $t=16$ | LFM/MF | 0.316 | 0.316 | 0/150 |
| TW+BC, $t=16$ | LFM/LightGCN | 0.274 | 0.274 | 0/150 |
| TW+BC, $t=16$ | LFM/NCF | 0.230 | 0.000 | 150/150 |
| TW+BC, $t=16$ | ML/MF | 0.735 | 0.735 | 0/150 |
| TW+BC, $t=16$ | ML/LightGCN | 0.364 | 0.364 | 0/150 |
| TW+BC, $t=16$ | ML/NCF | 0.229 | 0.000 | 150/150 |
| TW+BC, $t=16$ | Del/MF | 0.230 | 0.230 | 0/150 |
| TW+BC, $t=16$ | Del/LightGCN | 0.241 | 0.241 | 0/150 |
| TW+BC, $t=16$ | Del/NCF | 0.242 | 0.000 | 150/150 |
| TW+BC, $t=16$ | Bea/MF | 0.298 | 0.298 | 0/150 |
| TW+BC, $t=16$ | Bea/LightGCN | 0.257 | 0.257 | 0/150 |
| TW+BC, $t=16$ | Bea/NCF | 0.255 | 0.000 | 150/150 |
| full PACT, $t=16$ | LFM/MF | 0.013 | 0.013 | 0/150 |
| full PACT, $t=16$ | LFM/LightGCN | 0.004 | 0.004 | 0/150 |
| full PACT, $t=16$ | LFM/NCF | 0.001 | 0.000 | 150/150 |
| full PACT, $t=16$ | ML/MF | 0.006 | 0.006 | 0/150 |
| full PACT, $t=16$ | ML/LightGCN | -0.002 | -0.002 | 0/150 |
| full PACT, $t=16$ | ML/NCF | -0.002 | 0.000 | 150/150 |
| full PACT, $t=16$ | Del/MF | 0.006 | 0.006 | 0/150 |
| full PACT, $t=16$ | Del/LightGCN | 0.003 | 0.003 | 0/150 |
| full PACT, $t=16$ | Del/NCF | -0.006 | 0.000 | 150/150 |
| full PACT, $t=16$ | Bea/MF | 0.002 | 0.002 | 0/150 |
| full PACT, $t=16$ | Bea/LightGCN | -0.011 | -0.011 | 0/150 |
| full PACT, $t=16$ | Bea/NCF | 0.000 | 0.000 | 150/150 |

On MF and LightGCN the checks never fire and the results are identical. On NCF the attacker's MLP scaling (C3c) changes tensor norms by a factor of 10, so every round is rejected and TRIP obtains nothing; a TB-compliant attacker must forgo C3c, and without C3c TRIP recovers at most 0.03 on NCF even without PACT (Table B.6).

### B.13 Sole-writer rows under full PACT (MF, one round, honest Adam)

| Dataset | cohort | rows written | sole-writer rows | clients with ≥1 | median per client | sign agreement | mean |cos| of a sole-writer row with its writer | ceiling of the cohort |
|---|---|---|---|---|---|---|---|---|
| LFM | 128 | 3039 | 1559 | 100% | 12 | 100% | 0.663 | 0.408 |
| ML | 143 | 1682 | 0 | 0% | 0 | — | — | 0.813 |
| Del | 143 | 9486 | 8821 | 100% | 54 | 100% | 0.673 | 0.255 |
| Bea | 128 | 2657 | 2338 | 100% | 18 | 68% | 0.422 | 0.261 |

Amazon-Beauty runs three local epochs, which is why its sign agreement is lower (68%); MovieLens is dense enough that no row has a single writer.

### B.14 Utility: honest FedAvg versus PACT after 200 rounds (mean ± std over 3 seeds; scored on the 500 targets)

| Cell | metric | at start | honest | PACT | PACT vs. honest |
|---|---|---|---|---|---|
| LFM/MF | recall@10 | 0.1244 | 0.1254 ± 0.0004 | 0.1263 ± 0.0006 | +0.69% |
| LFM/MF | recall@20 | 0.1953 | 0.1959 ± 0.0006 | 0.1978 ± 0.0006 | +0.93% |
| LFM/MF | recall@50 | 0.3247 | 0.3245 ± 0.0003 | 0.3246 ± 0.0003 | +0.02% |
| LFM/MF | ndcg@10 | 0.1155 | 0.1162 ± 0.0000 | 0.1168 ± 0.0002 | +0.45% |
| LFM/MF | ndcg@20 | 0.1460 | 0.1466 ± 0.0004 | 0.1473 ± 0.0003 | +0.50% |
| LFM/MF | ndcg@50 | 0.1915 | 0.1918 ± 0.0002 | 0.1920 ± 0.0002 | +0.07% |
| LFM/MF | precision@10 | 0.0721 | 0.0726 ± 0.0002 | 0.0732 ± 0.0002 | +0.83% |
| LFM/MF | precision@20 | 0.0563 | 0.0566 ± 0.0002 | 0.0570 ± 0.0002 | +0.71% |
| LFM/MF | precision@50 | 0.0381 | 0.0381 ± 0.0000 | 0.0381 ± 0.0000 | +0.04% |
| LFM/MF | hr@10 | 0.4859 | 0.4859 ± 0.0016 | 0.4886 ± 0.0009 | +0.55% |
| LFM/MF | hr@20 | 0.6586 | 0.6600 ± 0.0009 | 0.6640 ± 0.0019 | +0.61% |
| LFM/MF | hr@50 | 0.8233 | 0.8213 ± 0.0000 | 0.8220 ± 0.0009 | +0.08% |
| LFM/LightGCN | recall@10 | 0.0714 | 0.0715 ± 0.0001 | 0.0715 ± 0.0001 | +0.00% |
| LFM/LightGCN | recall@20 | 0.1145 | 0.1147 ± 0.0001 | 0.1146 ± 0.0003 | -0.06% |
| LFM/LightGCN | recall@50 | 0.2072 | 0.2075 ± 0.0002 | 0.2076 ± 0.0002 | +0.02% |
| LFM/LightGCN | ndcg@10 | 0.0614 | 0.0614 ± 0.0001 | 0.0613 ± 0.0001 | -0.08% |
| LFM/LightGCN | ndcg@20 | 0.0800 | 0.0800 ± 0.0001 | 0.0800 ± 0.0002 | -0.10% |
| LFM/LightGCN | ndcg@50 | 0.1120 | 0.1122 ± 0.0001 | 0.1121 ± 0.0000 | -0.02% |
| LFM/LightGCN | precision@10 | 0.0394 | 0.0394 ± 0.0001 | 0.0394 ± 0.0001 | +0.00% |
| LFM/LightGCN | precision@20 | 0.0319 | 0.0320 ± 0.0000 | 0.0320 ± 0.0001 | -0.10% |
| LFM/LightGCN | precision@50 | 0.0235 | 0.0235 ± 0.0000 | 0.0235 ± 0.0000 | +0.06% |
| LFM/LightGCN | hr@10 | 0.3153 | 0.3159 ± 0.0009 | 0.3159 ± 0.0009 | +0.00% |
| LFM/LightGCN | hr@20 | 0.4679 | 0.4692 ± 0.0009 | 0.4679 ± 0.0000 | -0.29% |
| LFM/LightGCN | hr@50 | 0.6707 | 0.6707 ± 0.0000 | 0.6720 ± 0.0009 | +0.20% |
| LFM/NCF | recall@10 | 0.0134 | 0.0133 ± 0.0005 | 0.0124 ± 0.0007 | -6.70% |
| LFM/NCF | recall@20 | 0.0279 | 0.0272 ± 0.0009 | 0.0284 ± 0.0002 | +4.61% |
| LFM/NCF | recall@50 | 0.0616 | 0.0636 ± 0.0006 | 0.0612 ± 0.0007 | -3.92% |
| LFM/NCF | ndcg@10 | 0.0115 | 0.0110 ± 0.0006 | 0.0108 ± 0.0004 | -1.95% |
| LFM/NCF | ndcg@20 | 0.0182 | 0.0174 ± 0.0002 | 0.0182 ± 0.0002 | +4.37% |
| LFM/NCF | ndcg@50 | 0.0299 | 0.0300 ± 0.0006 | 0.0297 ± 0.0002 | -1.13% |
| LFM/NCF | precision@10 | 0.0082 | 0.0084 ± 0.0003 | 0.0077 ± 0.0005 | -8.73% |
| LFM/NCF | precision@20 | 0.0090 | 0.0088 ± 0.0002 | 0.0092 ± 0.0001 | +4.18% |
| LFM/NCF | precision@50 | 0.0076 | 0.0077 ± 0.0001 | 0.0076 ± 0.0000 | -1.90% |
| LFM/NCF | hr@10 | 0.0783 | 0.0790 ± 0.0041 | 0.0723 ± 0.0049 | -8.47% |
| LFM/NCF | hr@20 | 0.1647 | 0.1560 ± 0.0019 | 0.1613 ± 0.0009 | +3.43% |
| LFM/NCF | hr@50 | 0.2952 | 0.3072 ± 0.0043 | 0.3005 ± 0.0041 | -2.18% |
| ML/MF | recall@10 | 0.1897 | 0.1893 ± 0.0004 | 0.1891 ± 0.0004 | -0.09% |
| ML/MF | recall@20 | 0.2886 | 0.2848 ± 0.0005 | 0.2833 ± 0.0003 | -0.53% |
| ML/MF | recall@50 | 0.4714 | 0.4686 ± 0.0004 | 0.4668 ± 0.0002 | -0.38% |
| ML/MF | ndcg@10 | 0.3702 | 0.3706 ± 0.0004 | 0.3703 ± 0.0005 | -0.09% |
| ML/MF | ndcg@20 | 0.3678 | 0.3662 ± 0.0003 | 0.3649 ± 0.0002 | -0.36% |
| ML/MF | ndcg@50 | 0.3994 | 0.3985 ± 0.0004 | 0.3972 ± 0.0003 | -0.30% |
| ML/MF | precision@10 | 0.3120 | 0.3111 ± 0.0001 | 0.3108 ± 0.0005 | -0.09% |
| ML/MF | precision@20 | 0.2564 | 0.2542 ± 0.0004 | 0.2532 ± 0.0001 | -0.39% |
| ML/MF | precision@50 | 0.1807 | 0.1801 ± 0.0001 | 0.1798 ± 0.0002 | -0.19% |
| ML/MF | hr@10 | 0.8960 | 0.8967 ± 0.0019 | 0.8987 ± 0.0009 | +0.22% |
| ML/MF | hr@20 | 0.9520 | 0.9493 ± 0.0009 | 0.9480 ± 0.0000 | -0.14% |
| ML/MF | hr@50 | 0.9820 | 0.9820 ± 0.0000 | 0.9800 ± 0.0000 | -0.20% |
| ML/LightGCN | recall@10 | 0.1147 | 0.1174 ± 0.0005 | 0.1173 ± 0.0009 | -0.09% |
| ML/LightGCN | recall@20 | 0.1930 | 0.1962 ± 0.0005 | 0.1963 ± 0.0007 | +0.09% |
| ML/LightGCN | recall@50 | 0.3392 | 0.3425 ± 0.0003 | 0.3431 ± 0.0005 | +0.17% |
| ML/LightGCN | ndcg@10 | 0.2191 | 0.2244 ± 0.0004 | 0.2243 ± 0.0010 | -0.01% |
| ML/LightGCN | ndcg@20 | 0.2262 | 0.2312 ± 0.0003 | 0.2314 ± 0.0004 | +0.09% |
| ML/LightGCN | ndcg@50 | 0.2627 | 0.2672 ± 0.0001 | 0.2675 ± 0.0003 | +0.11% |
| ML/LightGCN | precision@10 | 0.1880 | 0.1917 ± 0.0007 | 0.1915 ± 0.0012 | -0.10% |
| ML/LightGCN | precision@20 | 0.1597 | 0.1623 ± 0.0005 | 0.1626 ± 0.0003 | +0.21% |
| ML/LightGCN | precision@50 | 0.1208 | 0.1225 ± 0.0001 | 0.1226 ± 0.0001 | +0.13% |
| ML/LightGCN | hr@10 | 0.7680 | 0.7800 ± 0.0016 | 0.7807 ± 0.0025 | +0.09% |
| ML/LightGCN | hr@20 | 0.8820 | 0.8867 ± 0.0009 | 0.8880 ± 0.0000 | +0.15% |
| ML/LightGCN | hr@50 | 0.9720 | 0.9720 ± 0.0000 | 0.9720 ± 0.0000 | +0.00% |
| ML/NCF | recall@10 | 0.1012 | 0.1106 ± 0.0005 | 0.1102 ± 0.0012 | -0.32% |
| ML/NCF | recall@20 | 0.1669 | 0.1841 ± 0.0011 | 0.1848 ± 0.0002 | +0.37% |
| ML/NCF | recall@50 | 0.3189 | 0.3337 ± 0.0014 | 0.3337 ± 0.0024 | +0.00% |
| ML/NCF | ndcg@10 | 0.2070 | 0.2191 ± 0.0005 | 0.2186 ± 0.0010 | -0.22% |
| ML/NCF | ndcg@20 | 0.2119 | 0.2255 ± 0.0008 | 0.2256 ± 0.0005 | +0.02% |
| ML/NCF | ndcg@50 | 0.2488 | 0.2608 ± 0.0002 | 0.2608 ± 0.0012 | -0.01% |
| ML/NCF | precision@10 | 0.1790 | 0.1887 ± 0.0006 | 0.1879 ± 0.0014 | -0.42% |
| ML/NCF | precision@20 | 0.1549 | 0.1621 ± 0.0008 | 0.1620 ± 0.0010 | -0.06% |
| ML/NCF | precision@50 | 0.1208 | 0.1235 ± 0.0000 | 0.1235 ± 0.0005 | +0.01% |
| ML/NCF | hr@10 | 0.7320 | 0.7540 ± 0.0016 | 0.7580 ± 0.0086 | +0.53% |
| ML/NCF | hr@20 | 0.8440 | 0.8760 ± 0.0028 | 0.8760 ± 0.0033 | +0.00% |
| ML/NCF | hr@50 | 0.9540 | 0.9673 ± 0.0025 | 0.9627 ± 0.0025 | -0.48% |
| Del/MF | recall@10 | 0.0054 | 0.0054 ± 0.0000 | 0.0054 ± 0.0000 | +0.00% |
| Del/MF | recall@20 | 0.0083 | 0.0084 ± 0.0001 | 0.0085 ± 0.0000 | +0.66% |
| Del/MF | recall@50 | 0.0158 | 0.0156 ± 0.0001 | 0.0156 ± 0.0001 | +0.00% |
| Del/MF | ndcg@10 | 0.0073 | 0.0073 ± 0.0000 | 0.0073 ± 0.0000 | -0.03% |
| Del/MF | ndcg@20 | 0.0076 | 0.0077 ± 0.0000 | 0.0077 ± 0.0000 | +0.36% |
| Del/MF | ndcg@50 | 0.0113 | 0.0113 ± 0.0000 | 0.0113 ± 0.0000 | -0.00% |
| Del/MF | precision@10 | 0.0074 | 0.0074 ± 0.0000 | 0.0074 ± 0.0000 | +0.00% |
| Del/MF | precision@20 | 0.0057 | 0.0058 ± 0.0000 | 0.0058 ± 0.0000 | +0.58% |
| Del/MF | precision@50 | 0.0044 | 0.0044 ± 0.0000 | 0.0044 ± 0.0000 | +0.00% |
| Del/MF | hr@10 | 0.0721 | 0.0721 ± 0.0000 | 0.0721 ± 0.0000 | +0.00% |
| Del/MF | hr@20 | 0.0922 | 0.0935 ± 0.0009 | 0.0942 ± 0.0000 | +0.71% |
| Del/MF | hr@50 | 0.1723 | 0.1703 ± 0.0000 | 0.1703 ± 0.0000 | +0.00% |
| Del/LightGCN | recall@10 | 0.2479 | 0.2480 ± 0.0001 | 0.2479 ± 0.0000 | -0.02% |
| Del/LightGCN | recall@20 | 0.3595 | 0.3596 ± 0.0001 | 0.3595 ± 0.0002 | -0.02% |
| Del/LightGCN | recall@50 | 0.4568 | 0.4568 ± 0.0000 | 0.4569 ± 0.0001 | +0.01% |
| Del/LightGCN | ndcg@10 | 0.3376 | 0.3377 ± 0.0000 | 0.3376 ± 0.0000 | -0.02% |
| Del/LightGCN | ndcg@20 | 0.3513 | 0.3514 ± 0.0001 | 0.3513 ± 0.0001 | -0.03% |
| Del/LightGCN | ndcg@50 | 0.3995 | 0.3996 ± 0.0000 | 0.3995 ± 0.0000 | -0.01% |
| Del/LightGCN | precision@10 | 0.3056 | 0.3057 ± 0.0001 | 0.3056 ± 0.0000 | -0.02% |
| Del/LightGCN | precision@20 | 0.2263 | 0.2263 ± 0.0000 | 0.2262 ± 0.0001 | -0.03% |
| Del/LightGCN | precision@50 | 0.1171 | 0.1171 ± 0.0000 | 0.1171 ± 0.0000 | +0.01% |
| Del/LightGCN | hr@10 | 0.7054 | 0.7054 ± 0.0000 | 0.7054 ± 0.0000 | +0.00% |
| Del/LightGCN | hr@20 | 0.7214 | 0.7214 ± 0.0000 | 0.7214 ± 0.0000 | +0.00% |
| Del/LightGCN | hr@50 | 0.7295 | 0.7295 ± 0.0000 | 0.7295 ± 0.0000 | +0.00% |
| Del/NCF | recall@10 | 0.0006 | 0.0005 ± 0.0001 | 0.0005 ± 0.0001 | +0.00% |
| Del/NCF | recall@20 | 0.0007 | 0.0007 ± 0.0000 | 0.0007 ± 0.0000 | +0.00% |
| Del/NCF | recall@50 | 0.0012 | 0.0011 ± 0.0000 | 0.0010 ± 0.0001 | -12.01% |
| Del/NCF | ndcg@10 | 0.0008 | 0.0007 ± 0.0001 | 0.0007 ± 0.0001 | +0.00% |
| Del/NCF | ndcg@20 | 0.0008 | 0.0008 ± 0.0000 | 0.0008 ± 0.0000 | +0.12% |
| Del/NCF | ndcg@50 | 0.0011 | 0.0010 ± 0.0000 | 0.0009 ± 0.0000 | -5.42% |
| Del/NCF | precision@10 | 0.0006 | 0.0005 ± 0.0001 | 0.0005 ± 0.0001 | +0.00% |
| Del/NCF | precision@20 | 0.0004 | 0.0004 ± 0.0000 | 0.0004 ± 0.0000 | +0.00% |
| Del/NCF | precision@50 | 0.0003 | 0.0002 ± 0.0000 | 0.0002 ± 0.0000 | -11.11% |
| Del/NCF | hr@10 | 0.0060 | 0.0047 ± 0.0009 | 0.0047 ± 0.0009 | +0.00% |
| Del/NCF | hr@20 | 0.0080 | 0.0080 ± 0.0000 | 0.0080 ± 0.0000 | +0.00% |
| Del/NCF | hr@50 | 0.0140 | 0.0120 ± 0.0000 | 0.0107 ± 0.0009 | -11.11% |
| Bea/MF | recall@10 | 0.0416 | 0.0416 ± 0.0000 | 0.0426 ± 0.0000 | +2.40% |
| Bea/MF | recall@20 | 0.0565 | 0.0568 ± 0.0000 | 0.0568 ± 0.0000 | +0.00% |
| Bea/MF | recall@50 | 0.0909 | 0.0909 ± 0.0000 | 0.0909 ± 0.0000 | +0.00% |
| Bea/MF | ndcg@10 | 0.0292 | 0.0293 ± 0.0000 | 0.0296 ± 0.0000 | +1.26% |
| Bea/MF | ndcg@20 | 0.0339 | 0.0340 ± 0.0000 | 0.0340 ± 0.0000 | +0.06% |
| Bea/MF | ndcg@50 | 0.0423 | 0.0422 ± 0.0000 | 0.0422 ± 0.0000 | +0.01% |
| Bea/MF | precision@10 | 0.0076 | 0.0076 ± 0.0000 | 0.0078 ± 0.0000 | +2.63% |
| Bea/MF | precision@20 | 0.0062 | 0.0064 ± 0.0000 | 0.0064 ± 0.0000 | +0.00% |
| Bea/MF | precision@50 | 0.0044 | 0.0044 ± 0.0000 | 0.0044 ± 0.0000 | +0.00% |
| Bea/MF | hr@10 | 0.0680 | 0.0680 ± 0.0000 | 0.0700 ± 0.0000 | +2.94% |
| Bea/MF | hr@20 | 0.1060 | 0.1080 ± 0.0000 | 0.1080 ± 0.0000 | +0.00% |
| Bea/MF | hr@50 | 0.1560 | 0.1560 ± 0.0000 | 0.1560 ± 0.0000 | +0.00% |
| Bea/LightGCN | recall@10 | 0.0094 | 0.0094 ± 0.0000 | 0.0094 ± 0.0000 | +0.00% |
| Bea/LightGCN | recall@20 | 0.0136 | 0.0136 ± 0.0000 | 0.0136 ± 0.0000 | +0.00% |
| Bea/LightGCN | recall@50 | 0.0395 | 0.0395 ± 0.0000 | 0.0395 ± 0.0000 | +0.00% |
| Bea/LightGCN | ndcg@10 | 0.0068 | 0.0068 ± 0.0000 | 0.0068 ± 0.0000 | -0.04% |
| Bea/LightGCN | ndcg@20 | 0.0081 | 0.0081 ± 0.0000 | 0.0081 ± 0.0000 | -0.04% |
| Bea/LightGCN | ndcg@50 | 0.0142 | 0.0142 ± 0.0000 | 0.0142 ± 0.0000 | -0.03% |
| Bea/LightGCN | precision@10 | 0.0034 | 0.0034 ± 0.0000 | 0.0034 ± 0.0000 | +0.00% |
| Bea/LightGCN | precision@20 | 0.0029 | 0.0029 ± 0.0000 | 0.0029 ± 0.0000 | +0.00% |
| Bea/LightGCN | precision@50 | 0.0026 | 0.0026 ± 0.0000 | 0.0026 ± 0.0000 | +0.00% |
| Bea/LightGCN | hr@10 | 0.0280 | 0.0280 ± 0.0000 | 0.0280 ± 0.0000 | +0.00% |
| Bea/LightGCN | hr@20 | 0.0460 | 0.0460 ± 0.0000 | 0.0460 ± 0.0000 | +0.00% |
| Bea/LightGCN | hr@50 | 0.0920 | 0.0920 ± 0.0000 | 0.0920 ± 0.0000 | +0.00% |
| Bea/NCF | recall@10 | 0.0111 | 0.0096 ± 0.0008 | 0.0086 ± 0.0005 | -10.09% |
| Bea/NCF | recall@20 | 0.0150 | 0.0153 ± 0.0008 | 0.0147 ± 0.0010 | -3.93% |
| Bea/NCF | recall@50 | 0.0393 | 0.0311 ± 0.0011 | 0.0303 ± 0.0011 | -2.42% |
| Bea/NCF | ndcg@10 | 0.0070 | 0.0047 ± 0.0004 | 0.0053 ± 0.0008 | +11.05% |
| Bea/NCF | ndcg@20 | 0.0085 | 0.0065 ± 0.0001 | 0.0073 ± 0.0008 | +12.32% |
| Bea/NCF | ndcg@50 | 0.0137 | 0.0101 ± 0.0001 | 0.0110 ± 0.0008 | +9.21% |
| Bea/NCF | precision@10 | 0.0022 | 0.0018 ± 0.0002 | 0.0015 ± 0.0001 | -18.52% |
| Bea/NCF | precision@20 | 0.0022 | 0.0018 ± 0.0000 | 0.0017 ± 0.0000 | -3.77% |
| Bea/NCF | precision@50 | 0.0016 | 0.0014 ± 0.0000 | 0.0014 ± 0.0000 | +1.94% |
| Bea/NCF | hr@10 | 0.0220 | 0.0180 ± 0.0016 | 0.0147 ± 0.0009 | -18.52% |
| Bea/NCF | hr@20 | 0.0400 | 0.0333 ± 0.0009 | 0.0327 ± 0.0009 | -2.00% |
| Bea/NCF | hr@50 | 0.0740 | 0.0633 ± 0.0009 | 0.0647 ± 0.0025 | +2.11% |

On LightGCN, LFM, Del and Bea barely move from their starting values in either arm (the warm-up checkpoint is saturated), so the agreement between the arms there says little about PACT.

### B.15 SENTRY per cell

| Cell | detection (all features) | false alarm (fresh rounds) | detection, closest pair only | false alarm, closest pair only | detection, norm z-score only |
|---|---|---|---|---|---|
| LFM/MF | 1.000 | 0.035 | 1.000 | 0.040 | 1.000 |
| LFM/LightGCN | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 |
| LFM/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 |
| ML/MF | 1.000 | 0.035 | 1.000 | 0.040 | 1.000 |
| ML/LightGCN | 1.000 | 0.025 | 1.000 | 0.040 | 1.000 |
| ML/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 |
| Del/MF | 1.000 | 0.035 | 1.000 | 0.040 | 1.000 |
| Del/LightGCN | 1.000 | 0.035 | 1.000 | 0.040 | 1.000 |
| Del/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 |
| Bea/MF | 1.000 | 0.035 | 1.000 | 0.040 | 1.000 |
| Bea/LightGCN | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 |
| Bea/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 |


Detector-aware probes on LastFM/MF (per-coordinate standard deviations $\sqrt{1-s^2}\,\sigma_v$ and $s\,\sigma_v$); false-alarm rates of the same detector on fresh honest rounds: 0.04 (closest pair only) and 0.035 (all features).

| separation $s$ | TRIP cosine | flag rate, closest pair only | flag rate, all features |
|---|---|---|---|
| 0.1 | 0.997 | 1.00 | 1.00 |
| 0.3 | 0.975 | 1.00 | 1.00 |
| 0.5 | 0.933 | 1.00 | 1.00 |
| 0.6 | 0.906 | 0.15 | 0.16 |
| 0.707 | 0.875 | 0.03 | 0.04 |

At $s=1/\sqrt2$ the two rows of a pair are independent draws, so no pair-based feature can separate them from honest cold-start items.

### B.16 BPR saturation and embedding norms on the warm-up checkpoints

Share of users whose honest one-epoch update is exactly zero (all BPR gradients saturated), for the 500 targets and for 500 ordinary users (training users 500–999); InvGrad's cosine on the LightGCN targets whose update is not zero; median embedding norms on MF.

| Dataset | LightGCN targets: zero share | LightGCN ordinary: zero share | InvGrad on non-zero LightGCN targets | MF targets: zero share | NCF targets: zero share | MF target median norm | MF ordinary median norm |
|---|---|---|---|---|---|---|---|
| LFM | 0.98 | 0.74 | 0.67 | 0.00 | 0.00 | 2.02 | 0.24 |
| ML | 0.12 | 0.07 | 0.83 | 0.00 | 0.00 | 1.34 | 0.60 |
| Del | 1.00 | 0.99 | — | 0.00 | 0.00 | 1.17 | 0.04 |
| Dou | 0.19 | 0.07 | 0.88 | 0.00 | 0.00 | 0.64 | 0.18 |
| Bea | 0.99 | 0.68 | 0.45 | 0.00 | 0.00 | 1.67 | 0.01 |
| ABk | 0.36 | 0.05 | 0.75 | 0.00 | 0.00 | 0.04 | 0.00 |
| Kin | 1.00 | 0.45 | 0.77 | 0.00 | 0.00 | 0.83 | 0.00 |
| Gow | 0.70 | 0.20 | 0.66 | 0.00 | 0.00 | 0.67 | 0.01 |
| iFa | 1.00 | 0.76 | 0.37 | 0.00 | 0.00 | 0.27 | 0.00 |
| Yelp | 0.60 | 0.05 | 0.74 | 0.00 | 0.00 | 0.82 | 0.02 |

TRIP's probe triples have margin $\mathcal{O}(\varepsilon)$, so their BPR factor is $\tfrac12$ regardless of saturation.

---

## C. Derivations

**LightGCN on a client's star graph.** With two propagation hops on the graph that links user $u$ only to its $M=|\mathcal{I}_u|$
items (each item has degree 1 in this graph), the layer-0 embedding is $\mathbf{u}$, layer 1 is $M^{-1/2}\sum_{j\in\mathcal{I}_u}\mathbf{v}_j$
(normalization $1/\sqrt{\deg u\cdot\deg j}=1/\sqrt{M}$), and layer 2 propagates back to $\mathbf{u}$. LightGCN averages the three layers:
$\tilde{\mathbf{u}}=\tfrac13(2\mathbf{u}+M^{-1/2}\sum_j\mathbf{v}_j)$.

**Isotropic ceiling.** Let $\mathbf{P}$ be the orthogonal projector onto the row space of $\mathbf{A}$ and let the rows of $\mathbf{U}$ be
i.i.d. isotropic with $d$ coordinates. Then $(\mathbf{P}\mathbf{U})_i^\top\mathbf{u}_i\approx P_{ii}\,d$ and
$\|(\mathbf{P}\mathbf{U})_i\|^2\approx\sum_j P_{ij}^2\,d=P_{ii}\,d$ (since $\mathbf{P}^2=\mathbf{P}$), so the cosine tends to
$\sqrt{P_{ii}}=\sqrt{1-(g-1)/N}$ as $d$ grows.

**Adam's first step.** With fresh optimizer state, $m_1=(1-\beta_1)g$ and $v_1=(1-\beta_2)g^2$; bias correction gives $\hat m_1=g$ and
$\hat v_1=g^2$, so the step is $\eta\,g/(|g|+\epsilon)\approx\eta\,\mathrm{sign}(g)$ coordinate-wise. For a sole-writer row this is
$\eta\,\mathrm{sign}(a_{u,j}\mathbf{u}_u)$, the sign pattern of $\pm\mathbf{u}_u$.

**Noise level per measurement.** Noise of standard deviation $\sigma_n$ on each coordinate of the averaged update becomes
$KW\sigma_n$ after undoing the average over the $KW$ participants of a round, and the difference of the two probe rows of a pair
has standard deviation $\sqrt2\,KW\sigma_n$ per entry of $\mathbf{g}_\rho$.
