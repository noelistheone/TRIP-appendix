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

Sections B.17–B.23 report follow-up experiments run after the main sweep. Recovery is scored as above (cosine with the true embedding; null as defined at the top); each section states its design first, and where a direct comparison exists the value under the original protocol is repeated in parentheses.

### B.17 Transcript binding with a TB-compliant NCF attacker (C3c dropped)

Design. Table B.12 shows that TB rejects every attack round on NCF because the MLP scaling of capability C3c moves tensor norms outside the norm-ratio band. The twelve NCF defense runs (TW+BC with $t=4$ and $t=16$, full PACT with $t=16$; $W=21$) were repeated with TB enforced and an attacker that forgoes C3c (MLP scaling factor 1 instead of the $\gamma$ of Section A.3); everything else is unchanged. The comparison column is the original run of the same configuration with C3c and without TB (the "without TB" column of Table B.12). Null and the anonymity ceiling of the realized blocks are those of Table B.11.

| Configuration | Cell | TRIP: TB enforced, no C3c | TRIP: C3c, no TB | Null | ceiling | rounds aborted by TB |
|---|---|---|---|---|---|---|
| TW+BC, $t=4$ | LFM/NCF | -0.002 | 0.463 | 0.079 | 0.495 | 0/150 |
| TW+BC, $t=4$ | ML/NCF | 0.003 | 0.478 | 0.038 | 0.496 | 0/150 |
| TW+BC, $t=4$ | Del/NCF | 0.001 | 0.486 | 0.038 | 0.495 | 0/150 |
| TW+BC, $t=4$ | Bea/NCF | 0.007 | 0.488 | 0.110 | 0.505 | 0/150 |
| TW+BC, $t=16$ | LFM/NCF | -0.006 | 0.230 | 0.079 | 0.255 | 0/150 |
| TW+BC, $t=16$ | ML/NCF | 0.005 | 0.229 | 0.038 | 0.249 | 0/150 |
| TW+BC, $t=16$ | Del/NCF | -0.002 | 0.242 | 0.038 | 0.247 | 0/150 |
| TW+BC, $t=16$ | Bea/NCF | 0.003 | 0.255 | 0.110 | 0.267 | 0/150 |
| full PACT, $t=16$ | LFM/NCF | 0.004 | 0.001 | 0.079 | 0.246 | 0/150 |
| full PACT, $t=16$ | ML/NCF | -0.006 | -0.002 | 0.038 | 0.255 | 0/150 |
| full PACT, $t=16$ | Del/NCF | -0.003 | -0.006 | 0.038 | 0.247 | 0/150 |
| full PACT, $t=16$ | Bea/NCF | 0.002 | 0.000 | 0.110 | 0.264 | 0/150 |

With C3c dropped, TB aborts 0/1800 rounds in total and TRIP's cosine lies within ±0.007 of zero on all cells, against 0.229–0.488 for the same TW+BC configurations with C3c and no TB.

### B.18 Uniform warm-up: no target oversampling, no weight decay, without (E2) and with (E2b) per-batch L2

Design. The warm-up of Section A.2 oversamples the targets' BPR triples and applies coupled weight decay $10^{-4}$ to MF. Two variants repeat the four-dataset protocol on all three backbones from new checkpoints. E2 trains the targets like every other user (oversampling factor 1), uses no weight decay on any backbone, and lengthens the Amazon-Beauty warm-up from 30 to 100 rounds of 3 local epochs so that its model still trains. E2b adds the per-batch L2 term of the public LightGCN/NGCF implementations, $\tfrac{\lambda}{2}\,(\|\mathbf{u}\|^2+\|\mathbf{v}^+\|^2+\|\mathbf{v}^-\|^2)/B$ over the rows of the current batch of size $B$, with $\lambda=10^{-4}$. For each variant the runs are: TRIP at $W=20$ and $W=21$, the per-client baselines with the budgets of Section A.4, TW+BC $t=16$ and full PACT with TB's checks enforced, the utility comparison of Section A.6, and the saturation statistics of Table B.16. Values in parentheses are the results under the original protocol (the runs the paper's main comparison reads).

Recovery under E2:

| Cell | TRIP $W=20$ | TRIP $W=21$ | InvGrad | RAIFLE | DLG | LtI | Null |
|---|---|---|---|---|---|---|---|
| LFM/MF | 0.981 (0.983) | 1.000 (1.000) | 0.884 (0.999) | 0.962 (1.000) | 0.890 (1.000) | 0.163 (0.226) | 0.181 (0.240) |
| LFM/LightGCN | 0.945 (0.965) | 0.963 (0.984) | 0.643 (0.014) | 0.778 (0.675) | 0.815 (0.102) | 0.140 (0.131) | 0.159 (0.151) |
| LFM/NCF | 0.947 (0.929) | 0.965 (0.946) | 0.439 (0.148) | 0.870 (0.268) | 0.946 (0.234) | -0.012 (0.043) | 0.038 (0.079) |
| ML/MF | 0.982 (0.984) | 1.000 (1.000) | 0.983 (0.997) | 0.994 (1.000) | 0.988 (1.000) | 0.355 (0.685) | 0.362 (0.716) |
| ML/LightGCN | 0.893 (0.936) | 0.910 (0.955) | 0.911 (0.743) | 0.985 (0.978) | 0.979 (0.875) | 0.248 (0.295) | 0.235 (0.294) |
| ML/NCF | 0.955 (0.961) | 0.974 (0.981) | 0.091 (0.140) | 0.203 (0.229) | 0.146 (0.225) | -0.015 (-0.012) | 0.038 (0.038) |
| Del/MF | 0.980 (0.892) | 1.000 (1.000) | 0.095 (0.999) | 0.458 (1.000) | 0.115 (1.000) | 0.010 (0.034) | 0.058 (0.095) |
| Del/LightGCN | 0.967 (0.967) | 0.987 (0.986) | 0.006 (-0.003) | 0.383 (0.373) | 0.129 (0.080) | 0.047 (0.020) | 0.086 (0.072) |
| Del/NCF | 0.960 (0.965) | 0.978 (0.984) | 0.446 (0.058) | 0.851 (0.116) | 0.968 (0.139) | -0.012 (-0.011) | 0.038 (0.038) |
| Bea/MF | 0.983 (0.978) | 1.000 (1.000) | 0.276 (0.999) | 0.856 (1.000) | 0.320 (1.000) | 0.061 (0.081) | 0.145 (0.208) |
| Bea/LightGCN | 0.959 (0.956) | 0.976 (0.975) | 0.151 (-0.003) | 0.606 (0.514) | 0.278 (0.062) | 0.123 (0.013) | 0.191 (0.117) |
| Bea/NCF | 0.940 (0.950) | 0.958 (0.968) | 0.021 (0.073) | 0.065 (0.268) | 0.064 (0.210) | -0.006 (0.079) | 0.038 (0.110) |

Recovery under E2b:

| Cell | TRIP $W=20$ | TRIP $W=21$ | InvGrad | RAIFLE | DLG | LtI | Null |
|---|---|---|---|---|---|---|---|
| LFM/MF | 0.981 (0.983) | 1.000 (1.000) | 0.990 (0.999) | 0.997 (1.000) | 0.998 (1.000) | 0.126 (0.226) | 0.148 (0.240) |
| LFM/LightGCN | 0.930 (0.965) | 0.949 (0.984) | 0.820 (0.014) | 0.866 (0.675) | 0.982 (0.102) | 0.090 (0.131) | 0.113 (0.151) |
| LFM/NCF | 0.947 (0.929) | 0.965 (0.946) | 0.438 (0.148) | 0.870 (0.268) | 0.947 (0.234) | -0.012 (0.043) | 0.038 (0.079) |
| ML/MF | 0.982 (0.984) | 1.000 (1.000) | 0.993 (0.997) | 0.999 (1.000) | 0.997 (1.000) | 0.343 (0.685) | 0.352 (0.716) |
| ML/LightGCN | 0.884 (0.936) | 0.901 (0.955) | 0.921 (0.743) | 0.990 (0.978) | 0.986 (0.875) | 0.233 (0.295) | 0.224 (0.294) |
| ML/NCF | 0.955 (0.961) | 0.974 (0.981) | 0.089 (0.140) | 0.203 (0.229) | 0.147 (0.225) | -0.013 (-0.012) | 0.038 (0.038) |
| Del/MF | 0.982 (0.892) | 1.000 (1.000) | 0.992 (0.999) | 0.994 (1.000) | 0.995 (1.000) | 0.058 (0.034) | 0.114 (0.095) |
| Del/LightGCN | 0.950 (0.967) | 0.968 (0.986) | 0.770 (-0.003) | 0.428 (0.373) | 0.679 (0.080) | 0.069 (0.020) | 0.113 (0.072) |
| Del/NCF | 0.960 (0.965) | 0.978 (0.984) | 0.445 (0.058) | 0.851 (0.116) | 0.967 (0.139) | -0.012 (-0.011) | 0.038 (0.038) |
| Bea/MF | 0.983 (0.978) | 1.000 (1.000) | 0.990 (0.999) | 0.956 (1.000) | 0.997 (1.000) | 0.056 (0.081) | 0.151 (0.208) |
| Bea/LightGCN | 0.961 (0.956) | 0.979 (0.975) | 0.851 (-0.003) | 0.827 (0.514) | 0.967 (0.062) | 0.063 (0.013) | 0.148 (0.117) |
| Bea/NCF | 0.940 (0.950) | 0.958 (0.968) | 0.022 (0.073) | 0.069 (0.268) | 0.066 (0.210) | -0.013 (0.079) | 0.038 (0.110) |


PACT under the two warm-ups (TB enforced; the ceiling is that of the realized blocks over the run's own true embeddings, unstratified, $t=16$):

| Variant | Cell | TW+BC $t=16$ | ceiling | gap | Null | best constant | TW+BC: rounds aborted by TB | full PACT | full PACT: rounds aborted by TB |
|---|---|---|---|---|---|---|---|---|---|
| E2 | LFM/MF | 0.299 | 0.300 | -0.001 | 0.181 | 0.181 | 0/150 | 0.004 | 0/150 |
| E2 | LFM/LightGCN | 0.280 | 0.288 | -0.008 | 0.159 | 0.159 | 0/150 | -0.000 | 0/150 |
| E2 | LFM/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2 | ML/MF | 0.430 | 0.432 | -0.002 | 0.362 | 0.363 | 0/150 | 0.004 | 0/150 |
| E2 | ML/LightGCN | 0.308 | 0.336 | -0.028 | 0.235 | 0.236 | 0/150 | -0.006 | 0/150 |
| E2 | ML/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2 | Del/MF | 0.244 | 0.245 | -0.001 | 0.058 | 0.058 | 0/150 | -0.003 | 0/150 |
| E2 | Del/LightGCN | 0.249 | 0.254 | -0.005 | 0.086 | 0.086 | 0/150 | -0.006 | 0/150 |
| E2 | Del/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2 | Bea/MF | 0.288 | 0.289 | -0.001 | 0.145 | 0.145 | 0/150 | 0.004 | 0/150 |
| E2 | Bea/LightGCN | 0.297 | 0.312 | -0.015 | 0.191 | 0.191 | 0/150 | -0.001 | 0/150 |
| E2 | Bea/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2b | LFM/MF | 0.282 | 0.282 | -0.000 | 0.148 | 0.148 | 0/150 | 0.005 | 0/150 |
| E2b | LFM/LightGCN | 0.257 | 0.269 | -0.012 | 0.113 | 0.113 | 0/150 | 0.000 | 0/150 |
| E2b | LFM/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2b | ML/MF | 0.422 | 0.424 | -0.002 | 0.352 | 0.353 | 0/150 | 0.005 | 0/150 |
| E2b | ML/LightGCN | 0.298 | 0.330 | -0.032 | 0.224 | 0.225 | 0/150 | -0.009 | 0/150 |
| E2b | ML/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2b | Del/MF | 0.266 | 0.266 | -0.000 | 0.114 | 0.114 | 0/150 | 0.007 | 0/150 |
| E2b | Del/LightGCN | 0.261 | 0.269 | -0.009 | 0.113 | 0.117 | 0/150 | 0.005 | 0/150 |
| E2b | Del/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |
| E2b | Bea/MF | 0.288 | 0.290 | -0.001 | 0.151 | 0.151 | 0/150 | 0.002 | 0/150 |
| E2b | Bea/LightGCN | 0.274 | 0.285 | -0.011 | 0.148 | 0.148 | 0/150 | -0.002 | 0/150 |
| E2b | Bea/NCF | 0.000 | 0.247 | -0.247 | 0.038 | 0.039 | 150/150 | 0.000 | 150/150 |


Saturation and norms (as in Table B.16: share of users whose honest one-epoch update is exactly zero, for the 500 targets and for 500 ordinary users; median target embedding norm):

| Cell | original: targets zero share | original: ordinary zero share | original: target median norm | E2: targets zero share | E2: ordinary zero share | E2: target median norm | E2b: targets zero share | E2b: ordinary zero share | E2b: target median norm |
|---|---|---|---|---|---|---|---|---|---|
| LFM/MF | 0.00 | 0.00 | 2.02 | 0.09 | 0.10 | 6.58 | 0.00 | 0.00 | 5.40 |
| LFM/LightGCN | 0.98 | 0.74 | 11.84 | 0.12 | 0.12 | 4.93 | 0.00 | 0.00 | 3.16 |
| LFM/NCF | 0.00 | 0.00 | 0.84 | 0.00 | 0.00 | 0.79 | 0.00 | 0.00 | 0.79 |
| ML/MF | 0.00 | 0.00 | 1.34 | 0.00 | 0.01 | 6.65 | 0.00 | 0.00 | 6.36 |
| ML/LightGCN | 0.12 | 0.07 | 10.04 | 0.01 | 0.00 | 5.91 | 0.00 | 0.00 | 5.39 |
| ML/NCF | 0.00 | 0.00 | 0.82 | 0.00 | 0.00 | 0.79 | 0.00 | 0.00 | 0.79 |
| Del/MF | 0.00 | 0.04 | 1.17 | 0.98 | 0.99 | 9.99 | 0.00 | 0.00 | 3.75 |
| Del/LightGCN | 1.00 | 0.99 | 6.30 | 0.99 | 1.00 | 5.43 | 0.00 | 0.00 | 0.60 |
| Del/NCF | 0.00 | 0.00 | 0.80 | 0.00 | 0.00 | 0.79 | 0.00 | 0.00 | 0.79 |
| Bea/MF | 0.00 | 0.16 | 1.67 | 0.75 | 0.74 | 6.55 | 0.00 | 0.00 | 4.60 |
| Bea/LightGCN | 0.99 | 0.68 | 6.32 | 0.77 | 0.75 | 5.33 | 0.00 | 0.00 | 2.82 |
| Bea/NCF | 0.00 | 0.00 | 0.84 | 0.00 | 0.00 | 0.79 | 0.00 | 0.00 | 0.79 |


Utility (protocol of Section A.6: 200 rounds, cohorts of 128, mean over 3 seeds; "honest change" is the honest arm's end value relative to the starting checkpoint):

| Variant | Cell | Recall@20 at start | honest | PACT | honest change | PACT vs. honest | NDCG@20: honest change | NDCG@20: PACT vs. honest |
|---|---|---|---|---|---|---|---|---|
| E2 | LFM/MF | 0.1759 | 0.1766 | 0.1773 | +0.36% | +0.38% | +0.38% | +0.14% |
| E2 | LFM/LightGCN | 0.1878 | 0.1885 | 0.1886 | +0.38% | +0.06% | +0.55% | -0.05% |
| E2 | LFM/NCF | 0.0033 | 0.0033 | 0.0033 | +0.00% | +0.00% | +0.00% | +0.33% |
| E2 | ML/MF | 0.2352 | 0.2385 | 0.2383 | +1.40% | -0.08% | +1.62% | +0.22% |
| E2 | ML/LightGCN | 0.2401 | 0.2429 | 0.2442 | +1.16% | +0.54% | +1.32% | +0.15% |
| E2 | ML/NCF | 0.1584 | 0.1568 | 0.1574 | -1.01% | +0.37% | -1.07% | +0.20% |
| E2 | Del/MF | 0.2994 | 0.2995 | 0.2996 | +0.05% | +0.04% | +0.06% | +0.01% |
| E2 | Del/LightGCN | 0.3632 | 0.3633 | 0.3631 | +0.02% | -0.04% | +0.02% | -0.02% |
| E2 | Del/NCF | 0.0003 | 0.0003 | 0.0003 | +0.00% | +0.00% | +0.00% | +0.00% |
| E2 | Bea/MF | 0.0153 | 0.0153 | 0.0153 | +0.00% | +0.00% | -0.02% | +0.07% |
| E2 | Bea/LightGCN | 0.0179 | 0.0179 | 0.0179 | +0.00% | +0.00% | -0.00% | -0.00% |
| E2 | Bea/NCF | 0.0176 | 0.0182 | 0.0176 | +3.79% | -3.65% | +1.85% | -1.80% |
| E2b | LFM/MF | 0.1854 | 0.1875 | 0.1882 | +1.12% | +0.39% | +0.51% | +0.21% |
| E2b | LFM/LightGCN | 0.2030 | 0.2041 | 0.2037 | +0.58% | -0.22% | +0.82% | -0.10% |
| E2b | LFM/NCF | 0.0033 | 0.0033 | 0.0033 | +0.00% | +0.00% | +0.00% | +0.33% |
| E2b | ML/MF | 0.2385 | 0.2435 | 0.2451 | +2.09% | +0.66% | +2.65% | +0.70% |
| E2b | ML/LightGCN | 0.2454 | 0.2499 | 0.2504 | +1.82% | +0.18% | +1.57% | +0.06% |
| E2b | ML/NCF | 0.1582 | 0.1567 | 0.1574 | -0.95% | +0.42% | -1.04% | +0.21% |
| E2b | Del/MF | 0.3869 | 0.3869 | 0.3871 | +0.01% | +0.06% | +0.08% | +0.01% |
| E2b | Del/LightGCN | 0.3959 | 0.3958 | 0.3959 | -0.03% | +0.02% | -0.00% | -0.03% |
| E2b | Del/NCF | 0.0003 | 0.0003 | 0.0003 | +0.00% | +0.00% | +0.00% | +0.00% |
| E2b | Bea/MF | 0.0262 | 0.0262 | 0.0262 | +0.00% | -0.16% | +0.27% | -0.69% |
| E2b | Bea/LightGCN | 0.0272 | 0.0272 | 0.0273 | +0.13% | +0.31% | +0.16% | -0.06% |
| E2b | Bea/NCF | 0.0176 | 0.0182 | 0.0176 | +3.79% | -3.65% | +1.85% | -1.80% |


The largest target zero-update share over the twelve cells is 1.00 under the original protocol, 0.99 under E2 and 0.00 under E2b. On MF under E2b, TRIP at $W=21$ reaches 1.000 and InvGrad, RAIFLE and DLG reach 0.956–0.999. On LightGCN under E2b the strongest per-client baseline is above TRIP at $W=21$ on 2 of the four datasets (LFM: DLG 0.982 vs. TRIP 0.949; ML: RAIFLE 0.990 vs. TRIP 0.901; Del: InvGrad 0.770 vs. TRIP 0.968; Bea: DLG 0.967 vs. TRIP 0.979). Under both variants TW+BC $t=16$ stays at or below the ceiling of its own blocks (gap -0.032 to -0.000 over the 16 runs in which TB aborted no round) and full PACT has $|\cos|\le0.009$; on NCF the attacker keeps C3c, so TB rejects every round in 16 of 16 runs (compare B.17). Over the 200 utility rounds the honest arm changes Recall@20 by -1.01% to +3.79% (E2) and -0.95% to +3.79% (E2b); PACT vs. honest lies within ±3.65% on Recall@20 and ±1.80% on NDCG@20.

### B.19 Share of the random initialization in the true embeddings, and recovery of the learned part

Design. Every user row is initialized as $0.1\cdot\mathcal{N}(\mathbf{0},\mathbf{I}_d)$ from a generator seeded by the user id, so the start vector $\mathbf{u}_0$ of each target can be rebuilt exactly. For every cell of the main comparison ($W=21$ TRIP runs and the per-client baselines, all ten datasets) this section reports, averaged over the evaluated targets: the cosine between $\mathbf{u}_0$ and the true embedding (what knowing the initialization alone gives), the median displacement $\|\mathbf{u}-\mathbf{u}_0\|/\|\mathbf{u}_0\|$ (how far the warm-up moved the row), the full-vector cosine of the estimate, and the cosine of the *learned part*: estimate and true embedding are both projected onto the complement of $\mathbf{u}_0$ (the component orthogonal to the start vector) before the cosine is taken. The baseline rows score InvGrad, RAIFLE, DLG and LtI on the same learned-part measure. The last table repeats the measure for TRIP under the E2b warm-up of Table B.18.

Per backbone, mean over the ten datasets (displacement: range of the per-dataset medians; TRIP learned part: mean and range):

| Backbone | cosine of $\mathbf{u}_0$ with $\mathbf{u}$ | displacement | TRIP, full vector | TRIP, learned part | InvGrad, learned part | RAIFLE, learned part | DLG, learned part | LtI, learned part |
|---|---|---|---|---|---|---|---|---|
| MF | 0.052 | 1.00–2.60 | 0.999 | 0.999 (0.995–1.000) | 0.998 | 1.000 | 1.000 | 0.411 |
| LightGCN | 0.265 | 4.07–20.40 | 0.967 | 0.966 (0.949–0.985) | 0.257 | 0.605 | 0.355 | 0.212 |
| NCF | 0.966 | 0.03–0.47 | 0.972 | 0.576 (0.190–0.910) | 0.067 | 0.209 | 0.157 | 0.251 |


Per cell:

| Cell | cosine of $\mathbf{u}_0$ with $\mathbf{u}$ | displacement (median) | TRIP, full vector | TRIP, learned part | InvGrad, learned part | RAIFLE, learned part | DLG, learned part | LtI, learned part |
|---|---|---|---|---|---|---|---|---|
| LFM/MF | 0.101 | 2.60 | 1.000 | 1.000 | 0.999 | 1.000 | 1.000 | 0.226 |
| ML/MF | 0.060 | 1.97 | 1.000 | 1.000 | 0.997 | 1.000 | 1.000 | 0.684 |
| Del/MF | 0.068 | 1.71 | 1.000 | 1.000 | 0.999 | 1.000 | 1.000 | 0.035 |
| Dou/MF | 0.012 | 1.28 | 1.000 | 1.000 | 0.996 | 1.000 | 1.000 | 0.988 |
| Bea/MF | 0.126 | 2.24 | 1.000 | 1.000 | 0.999 | 1.000 | 1.000 | 0.083 |
| ABk/MF | 0.003 | 1.00 | 0.997 | 0.997 | 0.999 | 1.000 | 1.000 | 0.503 |
| Kin/MF | 0.026 | 1.44 | 1.000 | 1.000 | 0.998 | 1.000 | 1.000 | 0.409 |
| Gow/MF | 0.018 | 1.30 | 1.000 | 1.000 | 0.998 | 1.000 | 1.000 | 0.541 |
| iFa/MF | 0.092 | 1.02 | 0.995 | 0.995 | 0.999 | 1.000 | 1.000 | 0.044 |
| Yelp/MF | 0.012 | 1.45 | 1.000 | 1.000 | 0.998 | 1.000 | 1.000 | 0.601 |
| LFM/LightGCN | 0.278 | 14.58 | 0.984 | 0.983 | 0.016 | 0.673 | 0.099 | 0.134 |
| ML/LightGCN | 0.128 | 12.95 | 0.955 | 0.954 | 0.742 | 0.978 | 0.876 | 0.295 |
| Del/LightGCN | 0.366 | 7.63 | 0.986 | 0.985 | 0.001 | 0.367 | 0.079 | 0.020 |
| Dou/LightGCN | 0.095 | 20.40 | 0.958 | 0.958 | 0.739 | 0.831 | 0.845 | 0.609 |
| Bea/LightGCN | 0.365 | 7.62 | 0.975 | 0.973 | -0.001 | 0.511 | 0.063 | 0.015 |
| ABk/LightGCN | 0.156 | 14.67 | 0.950 | 0.950 | 0.515 | 0.633 | 0.635 | 0.370 |
| Kin/LightGCN | 0.348 | 8.12 | 0.965 | 0.963 | 0.003 | 0.555 | 0.060 | 0.027 |
| Gow/LightGCN | 0.235 | 13.42 | 0.966 | 0.965 | 0.223 | 0.598 | 0.357 | 0.283 |
| iFa/LightGCN | 0.482 | 4.07 | 0.947 | 0.949 | -0.001 | 0.215 | 0.066 | -0.010 |
| Yelp/LightGCN | 0.196 | 16.43 | 0.984 | 0.983 | 0.335 | 0.686 | 0.474 | 0.373 |
| LFM/NCF | 0.941 | 0.35 | 0.946 | 0.699 | 0.106 | 0.236 | 0.203 | 0.132 |
| ML/NCF | 0.963 | 0.27 | 0.981 | 0.776 | 0.105 | 0.213 | 0.175 | -0.025 |
| Del/NCF | 0.999 | 0.05 | 0.984 | 0.293 | 0.043 | 0.091 | 0.133 | -0.009 |
| Dou/NCF | 0.911 | 0.47 | 0.987 | 0.910 | 0.068 | 0.159 | 0.080 | 0.374 |
| Bea/NCF | 0.967 | 0.25 | 0.968 | 0.693 | 0.039 | 0.309 | 0.249 | 0.282 |
| ABk/NCF | 0.998 | 0.03 | 0.975 | 0.205 | 0.027 | 0.132 | 0.088 | 0.114 |
| Kin/NCF | 0.958 | 0.27 | 0.951 | 0.645 | 0.069 | 0.373 | 0.179 | 0.567 |
| Gow/NCF | 0.981 | 0.17 | 0.962 | 0.508 | 0.034 | 0.203 | 0.194 | 0.319 |
| iFa/NCF | 0.999 | 0.03 | 0.980 | 0.190 | 0.017 | 0.124 | 0.072 | 0.071 |
| Yelp/NCF | 0.946 | 0.34 | 0.983 | 0.837 | 0.157 | 0.250 | 0.197 | 0.682 |


TRIP ($W=21$) under the E2b warm-up:

| Cell | cosine of $\mathbf{u}_0$ with $\mathbf{u}$ | displacement (median) | TRIP, full vector | TRIP, learned part |
|---|---|---|---|---|
| LFM/MF | 0.408 | 6.39 | 1.000 | 1.000 |
| LFM/LightGCN | 0.233 | 3.86 | 0.949 | 0.948 |
| LFM/NCF | 1.000 | 0.00 | 0.965 | 0.007 |
| ML/MF | 0.252 | 7.88 | 1.000 | 1.000 |
| ML/LightGCN | 0.144 | 6.69 | 0.901 | 0.900 |
| ML/NCF | 1.000 | 0.01 | 0.974 | 0.094 |
| Del/MF | 0.458 | 4.37 | 1.000 | 1.000 |
| Del/LightGCN | 0.129 | 1.17 | 0.968 | 0.967 |
| Del/NCF | 1.000 | 0.00 | 0.978 | 0.019 |
| Bea/MF | 0.233 | 5.71 | 1.000 | 1.000 |
| Bea/LightGCN | 0.083 | 3.57 | 0.979 | 0.979 |
| Bea/NCF | 1.000 | 0.00 | 0.958 | 0.010 |


On NCF the initialization alone has cosine 0.966 with the true embedding and the median displacement is 0.03–0.47 of the initial norm; TRIP's learned-part cosine is 0.576 (0.190–0.910) and exceeds that of all four baselines on 10 of 10 datasets. On MF and LightGCN the full-vector and learned-part cosines of TRIP differ by at most 0.002. Under the E2b warm-up the NCF displacement is 0.00–0.01.

### B.20 Per-client baselines at their published budgets (first 100 targets)

Design. Section A.4 caps the baselines below their published budgets. Here InvGrad, RAIFLE and DLG were re-run on LightGCN and NCF for all ten datasets with larger budgets: InvGrad 4800 steps with the published schedule (learning rate ×0.1 at 3/8, 5/8 and 7/8 of the budget), RAIFLE 3 restarts of 1500 steps, DLG with an L-BFGS cap of 2000 iterations per restart; same checkpoints and observed rounds, only the first 100 targets. The comparison is per target: the same targets are taken from the $W=21$ TRIP run and from the original-budget baseline runs of the main comparison. For NCF the learned-part cosine of Table B.19 is given as well. The last table gives convergence diagnostics recorded during the E3 runs: where in the budget InvGrad's best iterate occurred, and how many L-BFGS iterations DLG used.

LightGCN (cosine on the same targets; original budget → E3 budget):

| Dataset | targets | TRIP $W=21$ | InvGrad (300 → 4800 steps) | RAIFLE (300 → 1500 steps per restart) | DLG (cap 500 → 2000) |
|---|---|---|---|---|---|
| LFM | 100 | 0.985 | 0.040 → 0.054 | 0.746 → 0.817 | 0.119 → 0.116 |
| ML | 100 | 0.957 | 0.772 → 0.881 | 0.985 → 0.992 | 0.945 → 0.945 |
| Del | 100 | 0.981 | 0.010 → 0.013 | 0.323 → 0.517 | 0.042 → 0.046 |
| Dou | 100 | 0.959 | 0.678 → 0.720 | 0.835 → 0.870 | 0.798 → 0.802 |
| Bea | 100 | 0.973 | -0.004 → 0.004 | 0.554 → 0.705 | 0.056 → 0.056 |
| ABk | 100 | 0.953 | 0.455 → 0.545 | 0.690 → 0.769 | 0.604 → 0.606 |
| Kin | 100 | 0.968 | -0.005 → -0.003 | 0.621 → 0.737 | 0.038 → 0.038 |
| Gow | 100 | 0.964 | 0.235 → 0.308 | 0.684 → 0.693 | 0.377 → 0.378 |
| iFa | 100 | 0.946 | -0.028 → -0.028 | 0.335 → 0.298 | 0.055 → 0.055 |
| Yelp | 100 | 0.979 | 0.355 → 0.408 | 0.681 → 0.784 | 0.495 → 0.486 |

NCF, full vector:

| Dataset | targets | TRIP $W=21$ | InvGrad (100 → 4800 steps) | RAIFLE (100 → 1500 steps per restart) | DLG (cap 100 → 2000) |
|---|---|---|---|---|---|
| LFM | 100 | 0.946 | 0.154 → 0.124 | 0.257 → 0.219 | 0.228 → 0.227 |
| ML | 100 | 0.980 | 0.157 → 0.149 | 0.279 → 0.247 | 0.276 → 0.277 |
| Del | 100 | 0.982 | 0.068 → 0.035 | 0.117 → 0.052 | 0.147 → 0.146 |
| Dou | 100 | 0.987 | 0.091 → 0.084 | 0.248 → 0.203 | 0.122 → 0.131 |
| Bea | 100 | 0.970 | 0.084 → 0.085 | 0.262 → 0.206 | 0.185 → 0.185 |
| ABk | 100 | 0.975 | 0.066 → 0.066 | 0.120 → 0.091 | 0.153 → 0.150 |
| Kin | 100 | 0.952 | 0.095 → 0.085 | 0.226 → 0.209 | 0.127 → 0.124 |
| Gow | 100 | 0.962 | 0.086 → 0.079 | 0.271 → 0.218 | 0.247 → 0.243 |
| iFa | 100 | 0.981 | 0.036 → 0.021 | 0.079 → 0.055 | 0.075 → 0.075 |
| Yelp | 100 | 0.983 | 0.155 → 0.141 | 0.252 → 0.210 | 0.179 → 0.183 |

NCF, learned part:

| Dataset | targets | TRIP $W=21$ | InvGrad (100 → 4800 steps) | RAIFLE (100 → 1500 steps per restart) | DLG (cap 100 → 2000) |
|---|---|---|---|---|---|
| LFM | 100 | 0.695 | 0.092 → 0.082 | 0.240 → 0.177 | 0.189 → 0.194 |
| ML | 100 | 0.797 | 0.103 → 0.104 | 0.228 → 0.201 | 0.179 → 0.181 |
| Del | 100 | 0.281 | 0.042 → 0.025 | 0.086 → 0.041 | 0.157 → 0.156 |
| Dou | 100 | 0.896 | 0.054 → 0.036 | 0.163 → 0.130 | 0.094 → 0.103 |
| Bea | 100 | 0.704 | 0.045 → 0.037 | 0.292 → 0.214 | 0.230 → 0.230 |
| ABk | 100 | 0.199 | 0.021 → 0.021 | 0.122 → 0.084 | 0.090 → 0.090 |
| Kin | 100 | 0.613 | 0.064 → 0.061 | 0.349 → 0.293 | 0.183 → 0.185 |
| Gow | 100 | 0.519 | 0.022 → 0.026 | 0.213 → 0.150 | 0.208 → 0.202 |
| iFa | 100 | 0.168 | 0.008 → -0.003 | 0.122 → 0.049 | 0.083 → 0.083 |
| Yelp | 100 | 0.824 | 0.139 → 0.127 | 0.238 → 0.206 | 0.191 → 0.194 |


Convergence diagnostics:

| Cell | InvGrad budget (steps) | position of the best iterate (mean fraction of the budget) | share of targets whose best iterate lies in the last 10% | DLG cap (iterations) | L-BFGS iterations (mean) | share of targets hitting the cap |
|---|---|---|---|---|---|---|
| LFM/LightGCN | 4800 | 0.18 | 0.17 | 2000 | 25.3 | 0.00 |
| ML/LightGCN | 4800 | 0.97 | 0.95 | 2000 | 114.6 | 0.00 |
| Del/LightGCN | 4800 | 0.03 | 0.03 | 2000 | 3.7 | 0.00 |
| Dou/LightGCN | 4800 | 0.81 | 0.66 | 2000 | 70.1 | 0.00 |
| Bea/LightGCN | 4800 | 0.05 | 0.04 | 2000 | 8.8 | 0.00 |
| ABk/LightGCN | 4800 | 0.77 | 0.63 | 2000 | 62.4 | 0.00 |
| Kin/LightGCN | 4800 | 0.01 | 0.00 | 2000 | 2.0 | 0.00 |
| Gow/LightGCN | 4800 | 0.54 | 0.45 | 2000 | 60.3 | 0.00 |
| iFa/LightGCN | 4800 | 0.00 | 0.00 | 2000 | 0.5 | 0.00 |
| Yelp/LightGCN | 4800 | 0.60 | 0.48 | 2000 | 66.3 | 0.00 |
| LFM/NCF | 4800 | 0.29 | 0.09 | 2000 | 20.1 | 0.00 |
| ML/NCF | 4800 | 0.20 | 0.00 | 2000 | 21.0 | 0.00 |
| Del/NCF | 4800 | 0.16 | 0.00 | 2000 | 6.4 | 0.00 |
| Dou/NCF | 4800 | 0.10 | 0.01 | 2000 | 12.2 | 0.00 |
| Bea/NCF | 4800 | 0.35 | 0.09 | 2000 | 16.4 | 0.00 |
| ABk/NCF | 4800 | 0.00 | 0.00 | 2000 | 7.0 | 0.00 |
| Kin/NCF | 4800 | 0.14 | 0.06 | 2000 | 9.3 | 0.00 |
| Gow/NCF | 4800 | 0.06 | 0.01 | 2000 | 12.3 | 0.00 |
| iFa/NCF | 4800 | 0.09 | 0.02 | 2000 | 7.8 | 0.00 |
| Yelp/NCF | 4800 | 0.05 | 0.00 | 2000 | 8.7 | 0.00 |


On LightGCN the larger budgets change RAIFLE by -0.037 to +0.194 (largest gain on Del), InvGrad by +0.000 to +0.109 and DLG by -0.009 to +0.004. On the same targets TRIP at $W=21$ is above the E3-budget InvGrad, RAIFLE and DLG on 10/10, 9/10 and 10/10 LightGCN datasets, and on 10/10, 10/10 and 10/10 NCF datasets (learned part: 10/10, 10/10, 10/10). The share of targets on which DLG hits its cap is at most 0.00.

### B.21 Utility from a half-trained checkpoint

Design. In Tables B.14 and B.18 the honest arm barely moves on Delicious and Amazon-Beauty. To test whether this is because the warm-up checkpoint is already converged, the E2b warm-up was repeated with its number of rounds halved on the four small datasets (LFM 400→200, ML 400→200, Del 600→300, Bea 100→50), for MF and LightGCN (the backbones of the utility comparison). Each cell first trains and caches the half-trained checkpoint (the TRIP run on it is reported as well), then runs the utility comparison of Section A.6 (200 rounds, cohorts of 128, seeds 42, 43, 44) from that checkpoint; the rows labelled "full warm-up" are the E2b utility runs of Table B.18 for the same cells. "2 s.e." is twice the standard error of the difference between the arms' Recall@20 means over the three seeds.

| Cell | warm-up | TRIP $W=20$ on this checkpoint | Recall@20 at start | honest | PACT | honest change | PACT vs. honest | PACT − honest | 2 s.e. | NDCG@20: honest change | NDCG@20: PACT vs. honest |
|---|---|---|---|---|---|---|---|---|---|---|---|
| LFM/MF | E2b, full warm-up | 0.981 | 0.1854 | 0.1875 | 0.1882 | +1.12% | +0.39% | +0.0007 | 0.0020 | +0.51% | +0.21% |
| LFM/MF | E2b, half warm-up | 0.981 | 0.1932 | 0.1935 | 0.1932 | +0.12% | -0.15% | -0.0003 | 0.0009 | +0.16% | -0.09% |
| LFM/LightGCN | E2b, full warm-up | 0.930 | 0.2030 | 0.2041 | 0.2037 | +0.58% | -0.22% | -0.0004 | 0.0010 | +0.82% | -0.10% |
| LFM/LightGCN | E2b, half warm-up | 0.925 | 0.2211 | 0.2214 | 0.2216 | +0.16% | +0.06% | +0.0001 | 0.0010 | +0.02% | -0.01% |
| ML/MF | E2b, full warm-up | 0.982 | 0.2385 | 0.2435 | 0.2451 | +2.09% | +0.66% | +0.0016 | 0.0008 | +2.65% | +0.70% |
| ML/MF | E2b, half warm-up | 0.982 | 0.2617 | 0.2668 | 0.2680 | +1.92% | +0.45% | +0.0012 | 0.0007 | +2.52% | +0.84% |
| ML/LightGCN | E2b, full warm-up | 0.884 | 0.2454 | 0.2499 | 0.2504 | +1.82% | +0.18% | +0.0004 | 0.0005 | +1.57% | +0.06% |
| ML/LightGCN | E2b, half warm-up | 0.881 | 0.2560 | 0.2588 | 0.2593 | +1.09% | +0.17% | +0.0005 | 0.0009 | +0.85% | +0.11% |
| Del/MF | E2b, full warm-up | 0.982 | 0.3869 | 0.3869 | 0.3871 | +0.01% | +0.06% | +0.0002 | 0.0002 | +0.08% | +0.01% |
| Del/MF | E2b, half warm-up | 0.982 | 0.3962 | 0.3963 | 0.3961 | +0.04% | -0.04% | -0.0002 | 0.0003 | -0.11% | -0.05% |
| Del/LightGCN | E2b, full warm-up | 0.950 | 0.3959 | 0.3958 | 0.3959 | -0.03% | +0.02% | +0.0001 | 0.0004 | -0.00% | -0.03% |
| Del/LightGCN | E2b, half warm-up | 0.951 | 0.4049 | 0.4046 | 0.4049 | -0.08% | +0.08% | +0.0003 | 0.0001 | -0.06% | +0.00% |
| Bea/MF | E2b, full warm-up | 0.983 | 0.0262 | 0.0262 | 0.0262 | +0.00% | -0.16% | -0.0000 | 0.0001 | +0.27% | -0.69% |
| Bea/MF | E2b, half warm-up | 0.982 | 0.0137 | 0.0137 | 0.0137 | +0.00% | +0.00% | +0.0000 | 0.0000 | +0.86% | +0.05% |
| Bea/LightGCN | E2b, full warm-up | 0.961 | 0.0272 | 0.0272 | 0.0273 | +0.13% | +0.31% | +0.0001 | 0.0001 | +0.16% | -0.06% |
| Bea/LightGCN | E2b, half warm-up | 0.958 | 0.0319 | 0.0319 | 0.0319 | +0.00% | +0.00% | +0.0000 | 0.0000 | +0.02% | +0.04% |


From the half-trained checkpoints the honest arm changes Recall@20 by -0.08% to +1.92%, against -0.03% to +2.09% from the full E2b checkpoints; the half-trained checkpoint starts higher on 7 of 8 cells. Across both starting points and both metrics, PACT vs. honest lies within ±0.84%.

### B.22 Detector-aware probes on all SENTRY cells, NCF SENTRY re-runs, and aggregate-only baselines on four datasets

Design. Three extensions of Tables B.9 and B.15. (i) The detector-aware probes of Table B.15 (per-coordinate standard deviations $\sqrt{1-s^2}\,\sigma_v$ for the pair centre and $s\,\sigma_v$ for the half-difference) were run on all twelve cells of the four-dataset protocol for $s\in\{0.1, 0.3, 0.5, 0.6, 0.707\}$; the flag rate is the share of the 40 attack rounds that SENTRY flags, and the false-alarm rate is that of the same detector on fresh honest rounds. (ii) SENTRY was re-fitted on the NCF cells with the protocol of Section A.5 (60+60 honest rounds for fitting and calibration, $\alpha=0.05$, 200 fresh honest rounds, 40 attack rounds), this time recording per-round flag rates. (iii) The aggregate-only baselines of Table B.9 (each per-client solver given the averaged update of all 500 targets and every victim's interaction list) were run on Delicious and Amazon-Beauty in addition to LastFM and MovieLens.

TRIP cosine under detector-aware probes:

| Cell | Null | $s=0.1$ | $s=0.3$ | $s=0.5$ | $s=0.6$ | $s=0.707$ |
|---|---|---|---|---|---|---|
| LFM/MF | 0.240 | 0.997 | 0.975 | 0.933 | 0.906 | 0.875 |
| LFM/LightGCN | 0.151 | 0.472 | 0.359 | 0.341 | 0.336 | 0.333 |
| LFM/NCF | 0.079 | 0.648 | 0.661 | 0.688 | 0.718 | 0.758 |
| ML/MF | 0.716 | 0.989 | 0.901 | 0.817 | 0.784 | 0.753 |
| ML/LightGCN | 0.294 | 0.309 | 0.243 | 0.233 | 0.230 | 0.229 |
| ML/NCF | 0.038 | 0.880 | 0.886 | 0.901 | 0.912 | 0.926 |
| Del/MF | 0.095 | 0.999 | 0.991 | 0.982 | 0.976 | 0.971 |
| Del/LightGCN | 0.072 | 0.727 | 0.478 | 0.416 | 0.401 | 0.390 |
| Del/NCF | 0.038 | 0.983 | 0.983 | 0.983 | 0.983 | 0.983 |
| Bea/MF | 0.208 | 0.998 | 0.982 | 0.954 | 0.937 | 0.917 |
| Bea/LightGCN | 0.117 | 0.561 | 0.399 | 0.368 | 0.361 | 0.355 |
| Bea/NCF | 0.110 | 0.793 | 0.806 | 0.830 | 0.851 | 0.879 |

SENTRY flag rates on the same runs:

| Cell | false alarm, all features | false alarm, closest pair only | flag rate, all features, $s=0.1$ | flag rate, all features, $s=0.3$ | flag rate, all features, $s=0.5$ | flag rate, all features, $s=0.6$ | flag rate, all features, $s=0.707$ | flag rate, closest pair only, $s=0.707$ |
|---|---|---|---|---|---|---|---|---|
| LFM/MF | 0.035 | 0.04 | 1.00 | 1.00 | 1.00 | 0.16 | 0.04 | 0.03 |
| LFM/LightGCN | 0.030 | 0.04 | 1.00 | 1.00 | 0.98 | 0.12 | 0.04 | 0.03 |
| LFM/NCF | 0.030 | 0.04 | 1.00 | 1.00 | 0.99 | 0.13 | 0.04 | 0.03 |
| ML/MF | 0.035 | 0.04 | 1.00 | 1.00 | 1.00 | 0.15 | 0.04 | 0.03 |
| ML/LightGCN | 0.025 | 0.04 | 1.00 | 1.00 | 0.99 | 0.13 | 0.04 | 0.03 |
| ML/NCF | 0.030 | 0.04 | 1.00 | 1.00 | 0.99 | 0.13 | 0.04 | 0.03 |
| Del/MF | 0.035 | 0.04 | 1.00 | 1.00 | 1.00 | 0.16 | 0.04 | 0.03 |
| Del/LightGCN | 0.035 | 0.04 | 1.00 | 1.00 | 1.00 | 0.16 | 0.04 | 0.03 |
| Del/NCF | 0.030 | 0.04 | 1.00 | 1.00 | 0.99 | 0.13 | 0.04 | 0.03 |
| Bea/MF | 0.035 | 0.04 | 1.00 | 1.00 | 1.00 | 0.16 | 0.04 | 0.03 |
| Bea/LightGCN | 0.030 | 0.04 | 1.00 | 1.00 | 0.99 | 0.13 | 0.04 | 0.03 |
| Bea/NCF | 0.030 | 0.04 | 1.00 | 1.00 | 0.99 | 0.13 | 0.04 | 0.03 |


SENTRY re-run on NCF (scaled probes: probe scale in units of $\sigma_v$):

| Cell | detection (all features) | false alarm (fresh rounds) | detection, closest pair only | false alarm, closest pair only | detection, norm z-score only | scaled probes detected | scale range | per-round flag rate, scaled probes (min–max over scales) |
|---|---|---|---|---|---|---|---|---|
| LFM/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 | 8/8 | 0.001–50 | 1.000 |
| ML/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 | 8/8 | 0.001–50 | 1.000 |
| Del/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 | 8/8 | 0.001–50 | 1.000 |
| Bea/NCF | 1.000 | 0.030 | 1.000 | 0.040 | 1.000 | 8/8 | 0.001–50 | 1.000 |


Per-client baselines given only the 500-target aggregate (as Table B.9):

| Cell | InvGrad | RAIFLE | LtI | TRIP (sum, $W=21$) | Null |
|---|---|---|---|---|---|
| LFM/MF | 0.964 | 0.940 | 0.225 | 1.000 | 0.240 |
| LFM/LightGCN | -0.036 | -0.156 | 0.103 | 0.982 | 0.151 |
| LFM/NCF | 0.081 | 0.249 | 0.043 | 0.946 | 0.079 |
| ML/MF | 0.920 | 0.877 | 0.685 | 1.000 | 0.716 |
| ML/LightGCN | 0.111 | -0.186 | 0.325 | 0.960 | 0.294 |
| ML/NCF | 0.042 | 0.145 | -0.017 | 0.981 | 0.038 |
| Del/MF | 0.986 | 0.989 | 0.034 | 1.000 | 0.095 |
| Del/LightGCN | -0.006 | -0.052 | 0.020 | 0.986 | 0.072 |
| Del/NCF | 0.047 | 0.057 | -0.012 | 0.984 | 0.038 |
| Bea/MF | 0.972 | 0.926 | 0.081 | 1.000 | 0.208 |
| Bea/LightGCN | -0.029 | -0.088 | 0.013 | 0.975 | 0.117 |
| Bea/NCF | 0.058 | 0.249 | 0.079 | 0.968 | 0.110 |


At $s=1/\sqrt2$ the flag rate with all features is 0.04 across the twelve cells, against false-alarm rates of 0.025–0.035 on fresh honest rounds; TRIP's cosine there is 0.753–0.971 on MF, 0.229–0.390 on LightGCN and 0.758–0.983 on NCF, and lies below the null on 1 cell(s): ML/LightGCN. Given only the aggregate, InvGrad reaches 0.920–0.986 and RAIFLE 0.877–0.989 on MF; on LightGCN and NCF neither exceeds 0.249.

### B.23 Clean timing: one job per GPU on an idle machine, all methods including DLG

Design. The timings of Table B.10 come from the original sweep, in which several jobs shared the GPUs, and DLG was not timed. Here TRIP, InvGrad, RAIFLE and DLG were re-run on all thirty cells with one job per GPU (NVIDIA RTX 3090) and nothing else running on the machine, with the same checkpoints, rounds, budgets and solvers as the runs the paper reports; only the timings are taken from these runs. End-to-end time is preparation plus solve (TRIP: its $T=150$ rounds of simulated client training plus the closed-form solve; baselines: the observed round plus the per-client optimization), and the speedup is the baseline's end-to-end time divided by TRIP's. The right column repeats the same statistics for the original sweep.

| Quantity | clean re-run | original sweep |
|---|---|---|
| TRIP server solve, maximum over the 30 cells (s) | 0.40 | 0.54 |
| InvGrad solve, minimum over the 30 cells (s) | 151 | 148 |
| RAIFLE solve, minimum (s) | 384 | 381 |
| DLG solve, minimum (s) | 14.0 | 1.6* |
| solve-time ratio: InvGrad minimum / TRIP maximum | 373 | 276 |
| end-to-end speedup vs. InvGrad and RAIFLE, min–max | 0.23–15.2 | 0.22–16.0 |
| cells with end-to-end speedup < 1 (InvGrad and RAIFLE) | 8/60 | 8/60 |
| end-to-end speedup vs. DLG, min–max | 0.04–10.4 | 0.02–2.1* |
| cells with end-to-end speedup < 1 (DLG) | 14/30 | 27/30* |

\* The original sweep's DLG timings come from an earlier version of the DLG solver whose stopping condition ended L-BFGS after very few iterations; the DLG cosines reported in the paper come from re-runs with the corrected solver. Those timings are not comparable and are listed only for completeness.


Per backbone (solve times in seconds):

| Backbone | TRIP solve, max | InvGrad solve, mean | RAIFLE solve, mean | DLG solve, mean | end-to-end speedup vs. InvGrad | vs. RAIFLE | vs. DLG | cells with speedup < 1 |
|---|---|---|---|---|---|---|---|---|
| MF | 0.40 | 173 | 433 | 20 | 0.25–5.98 | 0.60–15.21 | 0.04–0.96 | Kin (InvGrad), iFa (InvGrad), iFa (RAIFLE), LFM (DLG), ML (DLG), Del (DLG), Dou (DLG), Bea (DLG), ABk (DLG), Kin (DLG), Gow (DLG), iFa (DLG), Yelp (DLG) |
| LightGCN | 0.39 | 208 | 540 | 182 | 0.31–5.78 | 0.76–15.02 | 0.04–10.36 | iFa (InvGrad), iFa (RAIFLE), Bea (DLG), Kin (DLG), iFa (DLG) |
| NCF | 0.35 | 166 | 467 | 328 | 0.23–2.42 | 0.60–6.87 | 0.31–5.75 | Kin (InvGrad), iFa (InvGrad), iFa (RAIFLE), iFa (DLG) |


Per cell:

| Cell | TRIP rounds (s) | TRIP solve (s) | InvGrad solve (s) | RAIFLE solve (s) | DLG solve (s) | speedup vs. InvGrad | speedup vs. RAIFLE | speedup vs. DLG | DLG L-BFGS iterations (mean) |
|---|---|---|---|---|---|---|---|---|---|
| LFM/MF | 28 | 0.02 | 152 | 384 | 26 | 5.45 | 13.74 | 0.96 | 11.2 |
| ML/MF | 28 | 0.02 | 168 | 428 | 23 | 5.98 | 15.21 | 0.85 | 9.5 |
| Del/MF | 32 | 0.40 | 172 | 424 | 14 | 5.53 | 13.41 | 0.55 | 5.8 |
| Dou/MF | 56 | 0.29 | 185 | 460 | 15 | 3.35 | 8.25 | 0.30 | 6.1 |
| Bea/MF | 76 | 0.07 | 159 | 391 | 32 | 2.10 | 5.15 | 0.44 | 12.5 |
| ABk/MF | 157 | 0.02 | 215 | 549 | 15 | 1.42 | 3.55 | 0.14 | 4.6 |
| Kin/MF | 187 | 0.29 | 166 | 413 | 22 | 0.92 | 2.24 | 0.14 | 8.1 |
| Gow/MF | 95 | 0.23 | 171 | 434 | 16 | 1.83 | 4.59 | 0.20 | 6.3 |
| iFa/MF | 707 | 0.05 | 169 | 414 | 15 | 0.25 | 0.60 | 0.04 | 6.1 |
| Yelp/MF | 101 | 0.04 | 169 | 429 | 18 | 1.71 | 4.28 | 0.21 | 7.1 |
| LFM/LightGCN | 38 | 0.24 | 196 | 508 | 119 | 5.22 | 13.48 | 3.16 | 15.8 |
| ML/LightGCN | 35 | 0.18 | 205 | 533 | 367 | 5.78 | 15.02 | 10.36 | 92.6 |
| Del/LightGCN | 42 | 0.12 | 207 | 533 | 55 | 4.99 | 12.63 | 1.37 | 6.0 |
| Dou/LightGCN | 64 | 0.39 | 210 | 564 | 297 | 3.32 | 8.84 | 4.65 | 71.2 |
| Bea/LightGCN | 85 | 0.28 | 189 | 489 | 26 | 2.25 | 5.79 | 0.32 | 3.0 |
| ABk/LightGCN | 168 | 0.02 | 252 | 660 | 319 | 1.55 | 3.98 | 1.95 | 61.9 |
| Kin/LightGCN | 192 | 0.30 | 201 | 525 | 28 | 1.08 | 2.76 | 0.18 | 3.1 |
| Gow/LightGCN | 105 | 0.01 | 208 | 537 | 269 | 2.03 | 5.16 | 2.61 | 51.4 |
| iFa/LightGCN | 699 | 0.30 | 203 | 519 | 14 | 0.31 | 0.76 | 0.04 | 1.5 |
| Yelp/LightGCN | 109 | 0.04 | 204 | 530 | 326 | 1.90 | 4.89 | 3.02 | 61.0 |
| LFM/NCF | 70 | 0.35 | 155 | 454 | 406 | 2.22 | 6.44 | 5.75 | 19.4 |
| ML/NCF | 68 | 0.03 | 164 | 468 | 376 | 2.41 | 6.87 | 5.52 | 18.5 |
| Del/NCF | 71 | 0.26 | 166 | 450 | 268 | 2.42 | 6.44 | 3.85 | 6.4 |
| Dou/NCF | 95 | 0.04 | 169 | 490 | 352 | 1.80 | 5.16 | 3.72 | 12.1 |
| Bea/NCF | 118 | 0.07 | 151 | 427 | 355 | 1.30 | 3.65 | 3.04 | 16.9 |
| ABk/NCF | 195 | 0.02 | 206 | 559 | 320 | 1.10 | 2.91 | 1.68 | 6.9 |
| Kin/NCF | 226 | 0.16 | 161 | 447 | 296 | 0.74 | 2.01 | 1.34 | 9.0 |
| Gow/NCF | 138 | 0.01 | 164 | 470 | 355 | 1.22 | 3.44 | 2.61 | 11.6 |
| iFa/NCF | 743 | 0.06 | 160 | 437 | 223 | 0.23 | 0.60 | 0.31 | 8.1 |
| Yelp/NCF | 144 | 0.02 | 164 | 465 | 333 | 1.17 | 3.27 | 2.34 | 8.4 |


Against DLG, TRIP's end-to-end time is longer (speedup below 1) on 10/10 MF cells and 4/20 LightGCN and NCF cells; against InvGrad and RAIFLE the count is 8/60 in the clean re-run and 8/60 in the original sweep.

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
