# Klasifikasi Multi-Label Insiden Siber di Bawah Pergeseran Distribusi Temporal

Studi berbasis reproduktifitas pada **EuRepoC Global Dataset 1.3** (3.414 insiden siber,
2011–2024). Tugasnya mengklasifikasi `incident_type` secara multi-label (6 label) dari
fitur *early*/*conditionally-available* saja — yaitu informasi yang tersedia pada titik
prediksi insiden, bukan field post-hoc hasil penyelidikan. Pertanyaan utamanya bukan
"seberapa akurat", melainkan **apakah performa yang turun dari waktu ke waktu benar-benar
degradasi model, atau sekadar perubahan bentuk data** (komposisi label dan missingness
yang bergeser menurut periode).

Temuan pokoknya terbalik dari draf sebelumnya: model **memang memburuk** — kualitas
like-for-like rata-rata +0,075 penurunan di FINAL(2024), turun di 15 dari 16 pasangan
model×periode — tetapi **pergeseran komposisi label menutupi sebagian penurunan itu**
(efek komposisi −0,044), sehingga skor mentah hanya turun +0,031. Metrik agregat justru
**menyembunyikan degradasi** (masking effect). Sebagian penurunan juga bersifat mekanis:
fitur yang paling banyak hilang di 2024 dapat dibuang tanpa menjatuhkan skor.

## Isi repo

| path | isi |
|---|---|
| `notebook/EuRepoC_Temporal_Robustness_Research_SPRINT.ipynb` | **notebook utama**: 51 sel kode, seluruh angka di Discussion/Limitations dihasilkan di sini |
| `notebook/SPRINT_REVISI_PENELITIAN.md` | dokumen sprint: daftar perbaikan pra-pemrosesan S1–S6 dan status penerapannya |
| `notebook/dataset/` | `eurepoc_global_dataset_1_3.csv` (satu-satunya input, tanpa state eksternal) |
| `notebook/out/` | artefak keluaran: `missingness_by_period.csv`, `category_vocab.json`, `top_k_regions.json`, `feature_eligibility.json` |
| `archived/` | notebook versi lama (`REVISED`, `SPRINT_OPTIMIZED`) disimpan untuk perbandingan, tidak diubah |

## Menjalankan notebook

```console
pip install -r notebook/requirements.txt
cd notebook
jupyter nbconvert --to notebook --execute --inplace \
    EuRepoC_Temporal_Robustness_Research_SPRINT.ipynb
```

nbconvert mengeksekusi semua sel kode berurutan dalam satu kernel; gagal = sel itu
berhenti dengan traceback di terminal. Bisa juga dibuka manual di Jupyter dan
*Run All* — hasilnya sama karena tidak ada state eksternal.

Berkas data dicari berurutan di `$EUREPOC_WORK_DIR`, `./dataset/`, lalu direktori notebook.
Direktori keluaran juga mengikuti `EUREPOC_WORK_DIR` (default `./out`) — arahkan ke tempat
lain bila ingin artefak tidak menimpa hasil sebelumnya:

```console
EUREPOC_WORK_DIR=/tmp/eurepoc-run jupyter nbconvert --to notebook --execute --inplace \
    notebook/EuRepoC_Temporal_Robustness_Research_SPRINT.ipynb
```

## Desain evaluasi

Split murni kronologis, tidak ada pengacakan. Model dan hyperparameter dipilih **hanya**
pada VAL(2022); FINAL(2024) disentuh sekali, di Bagian 26.

| split | periode | n insiden | peran |
|---|---|---|---|
| TRAIN | ≤ 2021 | 1545 | fitting + vocabulary (`MultiLabelBinarizer`, multi-hot, region tag) |
| VAL | 2022 | 354 | seleksi model dan strategi |
| TEST | 2023 | 723 | walk-forward (≤ 2023), error analysis, stability |
| FINAL | 2024 | 699 | evaluasi sekali, setelah semua keputusan dibekukan |

Preprocessing di-fit di dalam `Pipeline` pada tiap fold training saja. 92 baris tanpa
`start_date` dan baris tanpa target diketahui dibuang; komposisi label baris yang dibuang
dilaporkan terbuka di sel desain split (tidak acak: *Hijacking with Misuse* 68,5% pada
baris yang dibuang vs 52,3% pada baris yang dipakai).

## Status hipotesis

Verdict di bawah dihitung oleh sel notebook dari data, bukan diketik manual, dan
tersimpan sebagai output di dalam notebook itu sendiri.

| hipotesis | verdict | angka kunci |
|---|---|---|
| **H1** — performa turun seiring waktu karena model memburuk | ✅ didukung (dengan koreksi penamaan: metrik mentah MENYEMBUNYIKAN degradasi) | kualitas like-for-like turun di 15/16 pasangan (rata-rata +0,0750 di FINAL(2024)); komposisi menutupi −0,0436 → net mentah hanya +0,0314; monotonisitas mentah gagal (7 dari 8 model naik 2022→2023) |
| **H2** — shift distribusi label berasosiasi dengan penurunan | ✅ didukung, lemah | ρ = 0,615; p-perm = 0,038; CI [0,044; 0,871]; efek minimum terdeteksi ≈ 0,732 > ρ |
| **H3** — adaptasi dengan data terbaru lebih baik, urutan kronologis penting | ⚠️ sebagian | kronologis > slice 2022: ya; kronologis > slice acak: **tidak** (di dalam rentang 20 slice acak) |
| **H4** — peringkat importance stabil sepanjang waktu | ❌ tidak lulus ambang 0,70 | ρ: 0,786 (TRAIN↔VAL) → 0,571 (VAL↔TEST) → 0,253 (TRAIN↔TEST) |

## Batasan lingkungan eksekusi

- **8 model**: `xgboost` 3.2.0, `lightgbm` 4.70, `catboost` 1.2.10 terpasang, sehingga
  `MODEL_FACTORY` berisi `LogReg_OVR`, `RandomForest`, `ExtraTrees`, `HistGB`,
  `XGBoost`, `LightGBM`, `CatBoost`, `Chain_LogReg` (`TabPFN` tidak terpasang,
  diecualikan oleh probe di notebook).
- **Importance = permutation importance (Macro F1 drop)**: `shap` tidak terpasang,
  sehingga fallback permutation dipakai untuk seluruh periode.
- Nilai itu di-**assert** di notebook (sel 16 dan sel 22) terhadap konstanta
  `EXPECTED_N_MODELS` (adaptif: `4 + int(HAS_XGB) + int(HAS_LGBM) + int(HAS_CAT)
  + int(HAS_CHAIN) + int(HAS_TABPFN)`) / `EXPECTED_IMPORTANCE_METHOD`, agar narasi
  tidak pernah melaporkan konfigurasi yang berbeda dari yang benar-benar dijalankan.
- Hasil bergantung pada `RANDOM_SEED = 42` (estimator, split, bootstrap, permutation test).

## Hasil modul RADAR (implementasi proposal v2)

Semua modul dieksekusi penuh dalam notebook (0 error). Angka di bawah dari output sel:

| Modul | Hasil utama |
|---|---|
| A — karakterisasi pergeseran | JSD label-combo + CI + ambang permutasi; C2ST (AUC isi/mask/gabungan); missingness drift + CI; audit Zenodo → `EXTERNAL_VALIDATION` (lihat output sel C4) |
| B — atribusi Shapley | Tier-1: φ_Y=−0,0066, φ_M=−0,0091 (sum=ΔM tepat, err 0); Tier-2: φ_Y=−0,0186, φ_C=−0,0013, φ_M=−0,0032 (err 3.5e-18); CI bootstrap mencakup nol; **suntikan sintetis tidak seluruhnya lulus → status EKSPLORATIF**; headroom +0,0908 |
| C — adaptasi | Z2 (ROD rate 0.5) terbaik tanpa label: 2024 = 0,4654 vs Z0 0,4227; **H8 terbukti**: L2 (recency kontinu) − L1 = [+0,040; +0,035; +0,029; +0,032] semua > SESOI; **H6 gagal**: pemilihan margin tidak mengungguli acak |
| D — RADAR-Route | routing depth-2 **tidak terbentuk** (dominan tunggal Y di semua episode) → hasil negatif Tier-3, dilaporkan apa adanya |
| Statistik | 33 episode rolling-origin (h=1..3); H1 slope +0,0153/tahun, CI [+0,0009; +0,0296] → **False** (slope < SESOI 0,02) meski CI > 0 |

Verdict tiap hipotesis (H1–H8) dicetak oleh sel notebook — bukan diketik di README.
