### Single-nucleus RNA-seq analysis of disease-associated microglial heterogeneity  
### Human prefrontal cortex — GSE174367

---

## Motivation

Microglia, the brain's resident immune cells, undergo profound transcriptional 
remodelling in Alzheimer's disease (AD). Disease-associated microglial (DAM) states,
characterised by upregulation of *TREM2*, *APOE*, and *SPP1* alongside loss of 
homeostatic markers *P2RY12* and *CX3CR1*, were first described in mouse models 
(Keren-Shaul et al. 2017) and have since been partially recapitulated in human 
postmortem tissue (Yun et al 2021; Sun et al. 2023).

Hence, I aim to find out at what stage of the tangle burden and plaque burden, associated 
with Alzhiemer's disease, affect the transition of homeostatic microglia to DAMs. This was done by 
first identifying transcriptionally distinct subclusters of microglia, followed by pseudobulk analysis.

This project uses publicly available human snRNA-seq data from GSE174367 (Morabito et al. 2021)
to characterise microglial transcriptional substates in the prefrontal cortex and correlate their
abundance with the neuropathology of the respective donors.

---

## Questions guiding the analysis

1. What transcriptionally distinct microglial substates exist in the prefrontal 
   cortex of AD and control donors?

2. Are disease-associated microglial substates enriched in AD donors relative 
   to controls?

3. Does DAM-like microglial abundance correlate more strongly with Plaque.Stage 
   or Tangle.Stage, and what does this suggest about the sequence of microglial 
   activation?

4. Can a donor's pseudobulked microglial transcriptional profile predict their 
   disease status, and which genes drive this prediction?

---

## Dataset

| Property | Detail |
|---|---|
| Accession | GSE174367 |
| Tissue | Prefrontal cortex (postmortem) |
| Donors | 11 AD, 7 control |
| Total nuclei (post-QC) | 31,178 |
| Platform | 10x Genomics Chromium (GPL24676) |
| Metadata | Diagnosis, Plaque.Stage, Tangle.Stage, Age, Sex, PMI, Batch, RIN |

---

## Workflow (following SC best practices)

### 1. Loading the data 
Raw 10x Genomics h5 matrix from NCBI GEO loaded via `sc.read_10x_h5` and aligned to donor 
metadata via barcode index. Duplicate variable names resolved with 
`var_names_make_unique`.

### 2. Quality Control
Cells filtered on three criteria:
- Genes detected per cell: 200–2,500
- Mitochondrial gene content: < 5%
- Genes expressed in fewer than 3 cells removed

This reduced the dataset from 61,472 to 31,178 nuclei. 

### 3. Normalisation and Feature Selection
Counts normalised to 10,000 per cell and log-transformed. Top 2,000 highly 
variable genes selected using Seurat v3 variance-stabilising transformation 
applied to raw counts (stored in `layers["counts"]`).

### 4. Confounder Regression and Scaling
Total counts and mitochondrial percentage regressed out prior to scaling. 
Rationale: QC filtering removes overtly dead cells, but residual stress-related 
transcriptional variation in surviving nuclei would confound clustering if left 
uncorrected. Features scaled to zero mean and unit variance; outliers capped at 
10 standard deviations.

### 5. Dimensionality Reduction
PCA on scaled HVGs. Neighbourhood graph computed on top 30 PCs (k=15 neighbours). 
UMAP initialised from PAGA graph positions for a biologically-informed embedding 
layout.

### 6. Clustering
Leiden clustering at resolution 0.7, producing 17 clusters. Initial marker gene 
inspection using the Mathys 2019 panel (*NRGN*, *GAD1*, *AQP4*, *MBP*, *VCAN*, 
*CSF1R*, *CD74*, *FLT1*) confirmed major cell type separation on the UMAP.

### 7. Cell Type Annotation
CellTypist with the `Adult_Human_PrefrontalCortex.pkl` reference model, using 
majority voting over Leiden cluster boundaries. A tissue-matched PFC model was 
chosen over generic brain models to improve annotation specificity. CellTypist 
confirmed microglial identity for **clusters 4 and 5**, which also showed the 
highest AD enrichment (67% and 71% AD respectively).

### 8. Microglial Subclustering 
Clusters 4 and 5 isolated and reclustered at resolution of 0.5 to resolve 
DAM-like vs homeostatic substates.

### 9. DAM State Scoring
Each microglial nucleus scored continuously against human-validated signatures:

| State | Marker Genes |
|---|---|
| Homeostatic | *P2RY12, CX3CR1, TMEM119, SALL1, CSF1R* |
| DAM-like | *TREM2, APOE, SPP1, CD74* |

Downregulation of homeostatic markers in DAM-scored cells used as confirmatory 
evidence, not primary annotation.

### 10. Pathology Correlation 
Donor-level pseudobulked DAM scores correlated against Plaque.Stage and 
Tangle.Stage using Spearman's correlation to see whether microglial activation 
correlates to amyloid or tangle pathology more closely.

### 11. Predictive Modelling 
Multilayer perceptron (PyTorch) trained on pseudobulked microglial profiles 
(donor × gene matrix) to predict AD diagnosis. 5-fold stratified cross-validation 
used given the small cohort (n=18). To interpret the model's decision-making, SHAP
(SHapley Additive exPlanations) values were computed to rank feature importance and 
reveal the direction of effect that specific genes had on predicting AD status for 
each individual donor.
---

## Results


**Finding structure (to be filled in):**

**1. Microglial cluster identity and AD enrichment**  
> [Observation: which clusters, what proportion AD]  
> [Biological interpretation: what this suggests about microglial activation in AD PFC]  
> [Hypothesis: what the next analysis should test]

**2. DAM-like vs homeostatic substate resolution**  
> [Observation: how many substates identified, which marker genes define them]  
> [Biological interpretation: do substates match published human DAM signatures]  
> [Hypothesis: ]

**3. Pathology correlation — plaque vs tangle**  
> [Observation: correlation coefficients, which pathology stage tracks DAM score more closely]  
> [Biological interpretation: what this implies about when microglia activate]  
> [Hypothesis: ]

**4. Predictive modelling**  
> [Observation: AUC, cross-validation performance]  
> [Biological interpretation: which SHAP-ranked genes overlap with DAM signatures]  
> [Hypothesis: ]

---

## Limitations

- **snRNA-seq DAM undercapture:** Cytoplasmic DAM transcripts are systematically 
  underrepresented in nuclear RNA. DAM signal is likely underestimated throughout; 
  findings are interpreted conservatively.

- **Aggressive MT filtering:** The < 5% MT threshold reduced nuclei by ~46%. 
  This may have disproportionately removed stressed but biologically informative 
  cells. Future work could explore relaxed thresholds with more granular QC metrics.

- **Single brain region:** Only prefrontal cortex profiled. Microglial heterogeneity 
  across regions with differential amyloid and tangle burden (e.g. entorhinal cortex, 
  hippocampus) is not captured.

- **Small donor cohort:** n=18 (11 AD, 7 control). ML findings are 
  hypothesis-generating only and should not be interpreted as statistically 
  conclusive.

- **Ordinal pathology scores:** Plaque.Stage and Tangle.Stage are ordinal 
  neuropathological grades, not continuous quantitative measures of pathology burden.

- **No excitatory neuron cluster recovered:** . Absent in both AD and control samples, suggesting technical rather than 
  disease-related depletion. Diffuse low-level NRGN signal across clusters is consistent with ambient RNA 
  contamination rather than a true neuronal population.
  
- **MT filtering sensitivity check:** A relaxed MT threshold of 10% recovered only 254 additional nuclei (0.8% increase),
  confirming that the 5% threshold did not substantially bias cell recovery. The absence of canonical DAM markers is
  attributable to snRNA-seq undercapture of cytoplasmic transcripts rather than
  over-aggressive QC filtering.


## Repository Structure

```
alzheimer-microglia-snrnaseq/
├── README.md
├── environment.yml
├── .gitignore
├── data/
│   └── data_manifest.md          # describes files and accession; raw data not stored
├── figures/
│   ├── umap_all_cells.png
│   ├── umap_leiden_clusters.png
│   ├── umap_celltypist.png
│   ├── umap_microglia_substates.png
│   ├── dam_score_by_diagnosis.png
│   ├── dam_score_by_plaque_stage.png
│   ├── dam_score_by_tangle_stage.png
│   └── shap_summary.png
├── notebooks/
│   ├── 01_qc_preprocessing.ipynb
│   ├── 02_clustering_annotation.ipynb
│   ├── 03_microglia_subclustering.ipynb
│   ├── 04_dam_scoring_pathology.ipynb
│   └── 05_mlp_shap.ipynb
└── src/
    ├── preprocessing.py
    ├── scoring.py
    └── model.py
```

---

## References

- Keren-Shaul H et al. (2017). A Unique Microglia Type Associated with Restricting 
  Development of Alzheimer's Disease. *Cell*, 169(7), 1276–1290.  
  *(Foundational DAM concept — mouse model)*

- Mathys H et al. (2019). Single-cell transcriptomic analysis of Alzheimer's disease. 
  *Nature*, 570, 332–337.  
  *(ROSMAP single-cell atlas; marker gene reference)*

- 
- Sun N et al. (2023). Human microglial state dynamics in Alzheimer's disease 
  progression. *Cell*, 186(20), 4386–4403.  
