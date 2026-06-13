### Single-nucleus RNA-seq analysis of Disease-Associated Microglial Heterogeneity  
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

## Workflow (following single-cell best practices)

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
confirmed microglial identity for **clusters 1 and 19**, correspondent to cell
separation based on markers in Mathy 2019.

### 8. Microglial Subclustering 
Clusters 1 and 19 isolated and reclustered at resolution of 0.3 to resolve 
DAM-like vs homeostatic substates.

### 9. DAM State Scoring
Each microglial nucleus scored continuously against signatures in each microglial substate:

| State | Marker Genes |
|---|---|
| Homeostatic | *P2RY12, CX3CR1, TMEM119, P2RY13* |
| DAM-like | *SPP1, TREM2, APOE, CD9, CD68* |


### 10. Pathology Correlation 
Donor-level pseudobulked DAM scores correlated against Plaque.Stage and 
Tangle.Stage using Spearman's correlation to see whether microglial activation 
correlates to amyloid or tangle pathology more closely.

### 11. Predictive Modelling 
A Multi-Layer Perceptron (MLP) was built using Pytorch to predict the donor's diagnosis (AD vs Control)
based on their pseudobulked microglial profiles. SHAP values were used for interpretation to see what 
genes' expression contributed most to their predicted disease status.

---

## Results

**1. Clustering by cell type**
> 17 clusters were identified and were separate plots were generated for each cell type's markers from
> Mathys 2019 panel.
> 
> <img width="1217" height="1069" alt="image" src="https://github.com/user-attachments/assets/7eb10b10-f1d6-4ee9-98bc-a75f37e84b30" />
> CellTypist was also used to validate the manual annotation, which largely corresponded. However, excitatory
> neurons were not identified in both methods for both AD and control samples. This could be due to snRNA-seq not capturing
> transcripts localised to excitatory neuronal processes rather thatn disease-related depletion.
> 
> <img width="2019" height="450" alt="image" src="https://github.com/user-attachments/assets/ba11c231-7f3e-4492-8ef9-fe260fbf745c" />


**2. Microglial cluster identity and AD enrichment**  
> Clusters 1 and 19 were classified as microglia in the initial clustering.
> Further clustering was done, identifying 4 transcriptionally distinct microglial subclusters.
> 
> <img width="1028" height="445" alt="image" src="https://github.com/user-attachments/assets/2dc2483f-5dbf-4720-9e1f-c04eb66f3628" />
> These subclusters were scored based on the expression of DAM and homeostatic markers based on literature. However, scores
> were relatively diffused and not conclusive enough to assign states to each cluster.
> 
> <img width="4196" height="450" alt="image" src="https://github.com/user-attachments/assets/e826ce16-5efe-4afc-ba2b-94c252af35a4" />
> Differential analysis between AD and control microglia was done to identify what were the genes differentiating the
> 2 substates most instead of literature markers (that did not show large enough distinction). However, in "AD vs rest"
> plot, XIST was one of the top genes, which was unexpected as it is not an AD-related gene marker.
> 
> <img width="1947" height="473" alt="image" src="https://github.com/user-attachments/assets/cc56f844-e53a-4901-837c-55719c1abc73" />
> Since XIST is largely expressed in females, there could be disproportionate samples for male and female donors. This was
> confirmed by a cross-tabulation: 53% of the AD cells were female, but 67% of the control cells were male.
> Thus, differential analysis was done for each sex.
> 
> Female
> 
> <img width="1947" height="473" alt="image" src="https://github.com/user-attachments/assets/dc27ab86-dfff-48c8-801c-a436a8ca1d4c" />
> Male
> <img width="1947" height="473" alt="image" src="https://github.com/user-attachments/assets/4e3dd925-929d-4c11-a8df-e7fe755b022a" />
> The top differentially expressed gene markers were different from the canonical gene markers, which explains the diffuse
> signals in the scoring. (Canonical markers are based on scRNA-seq, so cytoplasmic transcripts might not be captured in
> snRNA-seq). A dotplot was done based on the common differentially expressed markers in both sexes, for both DAM and
> homeostatic - see Worfkflow.
>
> 
> <img width="425" height="301" alt="image" src="https://github.com/user-attachments/assets/c593f286-61bf-4ab9-884c-a5dd5ef03e81" />
> Further confirmation of the proportionof AD and control in each subcluster:
>
> 
> <img width="262" height="115" alt="image" src="https://github.com/user-attachments/assets/618fd627-a572-4de1-bf17-e24908d21dec" />


**3. Pathology correlation — plaque vs tangle**  
> <img width="421" height="141" alt="image" src="https://github.com/user-attachments/assets/29a092e3-97be-42fb-96ba-cfd619eb3dbc" />
> 

**4. Predictive modelling**  

> **Architecture:** 3-layer MLP (32 → 16 → 1), Dropout(0.3), BCEWithLogitsLoss with class imbalance weighting.

> **Performance:** 5-fold stratified cross-validation yielded mean AUC 0.950 ± 0.100. Given n=18, perfect separation in 4 of 5 folds likely reflects overfitting rather than true generalisation.
>  Note: Classification performance is not the primary claim of this analysis. Further validation on larger cohorts is needed.

> **SHAP interpretation:** MEF2C (mean |SHAP| = 1.22) and DPYD (1.01) were the strongest predictors. High MEF2C pushed predictions toward Control — consistent with its 
> role as a homeostatic microglial transcription factor. High DPYD pushed toward AD. Notably, NEAT1 and 
> MALAT1, the top DE genes at the cell level, ranked lower in donor-level SHAP importance, highlighting the difference 
> between statistical differential expression and predictive relevance at the donor level.
> <img width="778" height="450" alt="image" src="https://github.com/user-attachments/assets/68b3cda8-9216-4c80-9276-0369e6b4e106" />
>
> <img width="738" height="450" alt="image" src="https://github.com/user-attachments/assets/a792014c-c4f5-425c-8710-d5d1f613db10" />



---

## Limitations

- **snRNA-seq DAM undercapture:** Cytoplasmic DAM transcripts are systematically 
  underrepresented in nuclear RNA. DAM signal is likely underestimated throughout; 
  findings are interpreted conservatively.

- **Aggressive MT filtering:** The < 5% MT threshold reduced nuclei by ~46%. 
  This may have disproportionately removed stressed but biologically informative 
  cells. Future work could explore relaxed thresholds with more granular QC metrics.
  Note: A relaxed MT threshold of 10% recovered only 254 additional nuclei (0.8% increase),
  confirming that the 5% threshold did not substantially bias cell recovery. The absence of
  canonical DAM markers is likely due to snRNA-seq undercapture of cytoplasmic transcripts
  rather than over-aggressive QC filtering.

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


- Mathys H et al. (2019). Single-cell transcriptomic analysis of Alzheimer's disease. 
  *Nature*, 570, 332–337.  

-  Morabito S, Miyoshi E, Michael N, Shahin S et al. Single-nucleus chromatin accessibility
   and transcriptomic characterization of Alzheimer's disease. Nat Genet 2021 Aug;53(8):1143-1155.
   
- Sun N et al. (2023). Human microglial state dynamics in Alzheimer's disease 
  progression. *Cell*, 186(20), 4386–4403.  
