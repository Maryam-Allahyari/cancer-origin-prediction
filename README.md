# Predicting Cancer Origin from Tumour Genomics

[![tests](https://github.com/Maryam-Allahyari/cancer-origin-prediction/actions/workflows/ci.yml/badge.svg)](https://github.com/Maryam-Allahyari/cancer-origin-prediction/actions/workflows/ci.yml)

Can a tumour's genes tell us where a metastatic cancer started?

Cancer treatment is usually chosen by the organ the cancer came from: breast cancer that has spread to the liver is still treated as breast cancer. In about 3–5% of patients, doctors find metastases but can't identify where they started, even after a full work-up. This is called cancer of unknown primary, and it usually has a poor outlook (Pavlidis & Pentheroudakis, 2012). Tumours keep some molecular memory of their tissue of origin, and several groups have built classifiers on this idea from RNA-seq data (Grewal et al., 2019), targeted DNA panels (Penson et al., 2020; Moon et al., 2023) and whole-genome mutation patterns (Jiao et al., 2020).

This project is a smaller, reproducible version of that idea. It uses real data from the Personalized OncoGenomics (POG) program at BC Cancer: gene expression and somatic mutations from 570 patients with advanced cancer (Pleasance et al., 2020). Every patient's cancer type is known, so the question is how well that known origin can be recovered from a metastatic biopsy.

## Key findings

- **A model based on gene expression classifies metastatic biopsies by cancer type with about 0.83 balanced accuracy** across 7 cancer types (chance is about 0.14).
- Mutations alone reach about 0.41. The driver-gene mutation features added nothing on top of expression.
- A regularised logistic regression did better than XGBoost on this small dataset.
- Liver biopsies dominate the main pattern in the expression data, because some normal liver tissue ends up in the sample. The model was not misled by this: liver biopsies were classified as well as others.
- Known tissue markers are among the genes driving predictions, for example GATA3 for breast and NKX2-1 for lung.

## Data

All data come from the POG570 release by BC Cancer's Genome Sciences Centre: <https://www.bcgsc.ca/downloads/POG570/>

| File | Used for |
|---|---|
| `POG570_TPM_expression.txt.gz` | Gene expression (RNA-seq, TPM), the expression features |
| `POG570_small_mutations.txt.gz` | Somatic SNVs and indels from whole-genome sequencing, the mutation features |
| `Table_S1_Demographics.xlsx` | Cancer type (the label), biopsy site and patient details |

**The data are not included in this repository.** The release is for research use only and may not be redistributed. See [docs/data.md](docs/data.md) for file details, cleaning rules and the terms of use.

## Methods

I kept cancer types with at least 15 patients, which left 434 patients in 7 types (BRCA, COLO, LUNG, SARC, PANC, OV, CNS-PNS). Expression values were log2(TPM + 1) transformed, and the 2,000 most variable genes were selected inside each training fold. Mutation features were a mutated / not mutated table for 64 known driver genes (Sondka et al., 2018) plus tumour mutation burden.

I compared logistic regression and XGBoost (Chen & Guestrin, 2016) on three feature sets: expression, mutations, and both. All preprocessing sat inside scikit-learn pipelines (Pedregosa et al., 2011), so nothing was learned from the data being scored. Models were compared with 5-fold cross-validation, repeated five times on 80% of the patients. The final model was chosen beforehand as the simplest one within one standard deviation of the best, and only that model was scored on the remaining 20%. Because the classes are imbalanced, I report balanced accuracy. SHAP values (Lundberg & Lee, 2017) show which genes drive each prediction.

94 of the 434 biopsies were not metastatic. These are often taken from the organ where the cancer started, which makes them easy to classify, so every result is reported for all patients and for metastatic biopsies on their own.

## Results

**Cross-validation** (347 training patients, balanced accuracy, mean ± SD over 5 repeats). These numbers come from the Nextflow pipeline running in Docker. Logistic regression gives identical results on any machine, while XGBoost can differ by about 0.01 between machines because its multithreaded training isn't bit-for-bit reproducible.

| Model | Features | All patients | Metastatic only |
|---|---|---|---|
| Logistic regression | Expression | 0.88 ± 0.01 | 0.83 ± 0.01 |
| Logistic regression | Mutations | 0.46 ± 0.02 | 0.41 ± 0.01 |
| Logistic regression | Both | 0.88 ± 0.01 | 0.83 ± 0.01 |
| XGBoost | Expression | 0.85 ± 0.01 | 0.80 ± 0.03 |
| XGBoost | Mutations | 0.41 ± 0.02 | 0.38 ± 0.02 |
| XGBoost | Both | 0.84 ± 0.01 | 0.79 ± 0.02 |

**Held-out test set**, final model only (logistic regression on expression; 87 patients, 65 metastatic):

| | Balanced accuracy (95% bootstrap CI) |
|---|---|
| All test patients | 0.89 (0.79–0.97) |
| Metastatic test patients | 0.80 (0.73–0.98) |

The test set is small, so its intervals are wide. The cross-validation numbers are the more reliable estimate.

![Confusion matrix of the final model on the test set](docs/figures/confusion_matrix.png)

Most errors are between cancer types that are biologically close. In cross-validation, lung, colorectal and pancreatic cancers (all mostly adenocarcinomas, which grow from gland-forming cells) were sometimes confused with each other. CNS-PNS tumours and sarcomas were also mixed up, which makes sense because the CNS-PNS group includes nerve sheath tumours that are sarcomas themselves.

Adding the 65 mutation columns to the 2,000 expression features changed the model's predictions for only a handful of patients, so on this data the mutations carry little information that expression doesn't already have. For the biopsy-site question, liver-specific genes such as ALB and APOA1 ranked low in the model and made up a median of under 0.5% of each patient's SHAP explanation.

## How to run

You need Python 3.13 and about 300 MB of disk space for the data.

First download the data into `Data/Raw`:

```bash
bash scripts/download_data.sh
```

**With Nextflow and Docker** (Linux or WSL, Java 17+, Nextflow, Docker):

```bash
docker build -t cancer-origin:1.0 .
nextflow run main.nf -profile docker
```

Nextflow runs each step in the container and writes results to `results/`. If a run stops or a setting changes, `-resume` reuses the steps that are already done.

**Without Nextflow:**

```bash
pip install -r requirements.txt
bash scripts/run_local.sh            # clean -> features -> explore -> train -> explain
```

A full run takes 20–40 minutes, mostly for the XGBoost cross-validation.

The tests use small made-up data, so they don't need the download:

```bash
pytest
```

## Limitations

- 434 patients over 7 cancer types, and some groups are small (CNS-PNS has 19 patients). Rarer cancer types had to be left out.
- Some labels group quite different tumours: LUNG includes mesotheliomas, PANC includes neuroendocrine tumours, and CNS-PNS mixes several brain and nerve tumours.
- All samples come from one program at one centre, so results may not carry over to other hospitals or sequencing methods.
- Many patients had been treated before the biopsy, and treatment leaves marks on the tumour genome (Pleasance et al., 2020).
- Good results on cancers of known origin don't show that the model would work on true cancers of unknown primary.

## Next steps

- Better DNA-only features: regional mutation density across the genome and mutation spectra. A first test raised DNA-only balanced accuracy from 0.46 to about 0.61.
- Cleaner labels: separate mesotheliomas and neuroendocrine tumours, and reconsider the mixed CNS-PNS group.
- Survival analysis in R (Kaplan–Meier and Cox regression), with a Quarto report.

## References

- Chen T, Guestrin C. XGBoost: a scalable tree boosting system. *Proceedings of the 22nd ACM SIGKDD International Conference on Knowledge Discovery and Data Mining*. 2016:785–794. doi:10.1145/2939672.2939785
- Grewal JK, Tessier-Cloutier B, Jones M, et al. Application of a neural network whole transcriptome–based pan-cancer method for diagnosis of primary and metastatic cancers. *JAMA Netw Open*. 2019;2(4):e192597. doi:10.1001/jamanetworkopen.2019.2597
- Jiao W, Atwal G, Polak P, et al. A deep learning system accurately classifies primary and metastatic cancers using passenger mutation patterns. *Nat Commun*. 2020;11:728. doi:10.1038/s41467-019-13825-8
- Lundberg SM, Lee SI. A unified approach to interpreting model predictions. *Advances in Neural Information Processing Systems 30 (NIPS 2017)*. 2017:4765–4774.
- Moon I, LoPiccolo J, Baca SC, et al. Machine learning for genetics-based classification and treatment response prediction in cancer of unknown primary. *Nat Med*. 2023;29:2057–2067. doi:10.1038/s41591-023-02482-6
- Pavlidis N, Pentheroudakis G. Cancer of unknown primary site. *Lancet*. 2012;379(9824):1428–1435. doi:10.1016/S0140-6736(11)61178-1
- Pedregosa F, Varoquaux G, Gramfort A, et al. Scikit-learn: machine learning in Python. *J Mach Learn Res*. 2011;12:2825–2830.
- Penson A, Camacho N, Zheng Y, et al. Development of genome-derived tumor type prediction to inform clinical cancer care. *JAMA Oncol*. 2020;6(1):84–91. doi:10.1001/jamaoncol.2019.3985
- Pleasance E, Titmuss E, Williamson L, et al. Pan-cancer analysis of advanced patient tumors reveals interactions between therapy and genomic landscapes. *Nat Cancer*. 2020;1:452–468. doi:10.1038/s43018-020-0050-6
- Sondka Z, Bamford S, Cole CG, et al. The COSMIC Cancer Gene Census: describing genetic dysfunction across all human cancers. *Nat Rev Cancer*. 2018;18:696–705. doi:10.1038/s41568-018-0060-1

## Acknowledgements

This work uses data from the Personalized OncoGenomics (POG) program at BC Cancer. I thank the patients and families who took part, and the POG team for making the data available. The full acknowledgement text requested by the data providers is in `POG570_README.txt`, which comes with the download.
