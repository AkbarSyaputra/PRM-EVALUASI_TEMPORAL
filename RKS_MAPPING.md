# Pemetaan RKS → Notebook — Lampiran Laporan Kemajuan PRM-RE

Dokumen ini memetakan setiap tujuan & hipotesis pada **RKS**
(`PRM-RE_Akbar Syaputra_RKS_Evaluasi Temporal Robustness.pdf`) ke bukti di
`notebook/EuRepoC_Temporal_Robustness_Research_SPRINT.ipynb`. Angka diambil dari
output sel eksekusi terakhir (0 error, 108 sel).

## Tujuan RKS (§1.2)

| # | Tujuan (RKS) | Bukti di notebook | Status |
|---|---|---|---|
| T1 | Klasifier multilabel 6 jenis insiden dari informasi awal; metrik Macro/Micro/Hamming | Sel 48 `MODEL_FACTORY` + `perf_df` (Macro/Micro/Weighted/Hamming per model × split) | ✅ Terjawab — 8 model (5 sesuai RKS + 3 ekstensi: XGBoost, CatBoost, Chain_LogReg; TabPFN absen). Amendmen dicatat |
| T2 | Ukur pergeseran JSD + PSI antarperiode | Sel 56 `shift_pivot` (JSD + PSI per fitur); Modul A sel 26: C1 JSD+CI+ambang permutasi, C2 C2ST isi/mask/gabungan, C3 missingness+CI | ✅ Terjawab (Modul A = ekstensi di atas janji) |
| T3 | Analisis degradasi vs pergeseran + kestabilan fitur | Sel 59 robustness; sel 61 dekomposisi S4.3; sel 79/98 permutation importance | ⚠️ Terjawab — **penyimpangan**: SHAP absen → permutation importance (di-assert `EXPECTED_IMPORTANCE_METHOD` sel 22). Catat amendmen |
| T4 | Adaptasi: retrain biasa vs recency-weighting dengan data terbaru terbatas | Sel 72 Modul C: L1 vs L2, evaluasi **remaining-2023** (leak E2 diperbaiki), H8 probe | ✅ Terjawab — **H8 terbukti**: L2−L1 = [+0,038; +0,033; +0,027; +0,025], semua > SESOI 0,02 |
| Temuan 1 | Pergeseran distribusi terukur | Sel 26 (JSD+CI+C2ST) | ✅ |
| Temuan 2 | Besaran degradasi performa | Sel 59, 61 (+ B-1 all-ones) | ✅ — koreksi: "layak" dibaca terhadap all-ones 0,409/0,480/0,468, bukan 0,076 |
| Temuan 3 | Efektivitas adaptasi kronologis | Sel 72 (H8), sel 71 (variabilitas 20 slice) | ✅ |
| Temuan 4 | Pola kesalahan sistematis | Sel 84 (error per label TEST), sel 99 (FINAL) | ✅ |

## Hipotesis RKS (§3.5)

| # | Hipotesis | Kriteria RKS asli | Amendmen (dicatat di laporan) | Verdict (Claim Registry sel 104) |
|---|---|---|---|---|
| **H1** | Performa menurun konsisten seiring jarak temporal | Wilcoxon berpasangan n=3 (2022→2024), p<0,05 | **Unit inferensi → 33 episode rolling-origin (h=1..3)**; block bootstrap **per tahun uji**; SESOI=0.02. Alasan: p-min eksak n=3 = 0,25 (mustahil); RKS §3.4 tahap 6 sudah menjanjikan Walk-Forward | ❌ **Tidak didukung**: slope +0,0153/tahun < SESOI; CI [+0,0062; +0,0246] |
| **H2** | Recency weighting > retrain biasa | selisih signifikan di 2023, arah konsisten di 2024 | evaluasi **remaining-2023** (fix leak E2: baris adaptasi tidak ikut dinilai); tanpa Wilcoxon (n kecil) → ukuran efek + SESOI | ✅ **Terbukti**: L2−L1 = [+0,038; +0,033; +0,027; +0,025] di k = 8/16/32/64 |

## Temuan di luar hipotesis (bukan bagian janji RKS — dilaporkan terpisah)

| Label | Temuan | Angka |
|---|---|---|
| **F-1** | Metrik mentah menyembunyikan degradasi (masking) | kualitas like-for-like +0,0750 di 15/16 pasangan FINAL(2024); komposisi −0,0436; net mentah +0,0314 |
| **B-1** | Baseline all-ones mengalahkan semua model di 2024 | 0,4094 / 0,4802 / 0,4681; 8/8 model < 0,4681 di FINAL(2024) |
| **B-2** | Brier Skill Score | **4/6 label BSS ≤ 0** (model kalah dari prediksi konstan prevalensi TRAIN) di VAL(2022) |
| **H6** | Pemilihan insiden (margin) vs acak | ❌ gagal — delta negatif di semua k |
| **H3/H4** | slice kronologis vs acak; stabilitas importance | ⚠️ sebagian / ❌ (rho 0,786→0,253) |

## Amendmen yang dicatat (bukan pelanggaran senyap)

1. **Statistik H1**: Wilcoxon n=3 → 33 episode rolling + block bootstrap per tahun uji (alasan konstruktif: p-min = 0,25).
2. **Model**: RKS §2.1 = 5 → dijalankan 8 (xgboost/lightgbm/catboost terpasang + Chain_LogReg); TabPFN absen.
3. **SHAP → permutation importance**: `shap` tidak terpasang; notebook meng-assert `EXPECTED_IMPORTANCE_METHOD`.
4. **2024 hanya dilaporkan, tidak dipakai memilih** (RKS §3.1): pemilihan threshold di VAL (sel B4), pemilihan rate Z2 di VAL (sel E1), evaluasi adaptasi di remaining-2023; kemunculan 2024 di B4/B5/E1/Modul A = pelaporan/deskriptif.
5. **Baseline all-ones ditambahkan** (di luar daftar baseline RKS) agar klaim "layak" jujur.
6. **Modul B (Shapley) & D (route) = Lampiran eksploratif**, di luar cakupan RKS — D4 sintetis gagal sebagian, CI Tier-1 mencakup nol, route tak terbentuk.

## Keputusan terkait yang mengikat

- **H1 resmi = rolling (B2)**; "masking komposisi" = temuan F-1, bukan hipotesis (mencegah HARKing).
- Verdict seluruh hipotesis = output sel Claim Registry (sel 104), bukan ketikan README.
- Angka H8/H6/kurva E2 versi lama (sebelum fix leak) ditarik dari seluruh narasi.