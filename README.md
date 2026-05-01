# ADMET-predictor

Predykcja własności **ADMET** (Absorption, Distribution, Metabolism, Excretion, Toxicity) cząsteczek leków. Każdy z 10 endpointów z benchmarku **TDC** (Therapeutics Data Commons) modelowany jest niezależnie (STL — Single-Task Learning) oraz w trybie wielozadaniowym (MTL — Multi-Task Learning), z porównaniem trzech reprezentacji molekularnych (fingerprinty, deskryptory 2D, embeddingi MoLFormer) i dwóch rodzin modeli (Random Forest, sieć neuronowa).

## Endpointy

| # | Endpoint | Zadanie | Metryki |
|---|---|---|---|
| 1 | Caco2_Wang | regresja | RMSE / MAE / R² |
| 2 | Lipophilicity_AstraZeneca | regresja | RMSE / MAE / R² |
| 3 | Solubility_AqSolDB | regresja | RMSE / MAE / R² |
| 4 | HIA_Hou | klasyfikacja | Accuracy / F1 / AUROC |
| 5 | Half_Life_Obach | regresja | RMSE / MAE / R² |
| 6 | Clearance_Hepatocyte_AZ | regresja | RMSE / MAE / R² |
| 7 | CYP3A4_Veith | klasyfikacja | Accuracy / F1 / AUROC |
| 8 | VDss_Lombardo | regresja | RMSE / MAE / R² |
| 9 | AMES | klasyfikacja | Accuracy / F1 / AUROC |
| 10 | hERG | klasyfikacja | Accuracy / F1 / AUROC |

## Tech stack

| Warstwa | Narzędzia |
|---|---|
| Język / środowisko | Python 3.12, Jupyter / Google Colab (GPU T4) |
| Dane | [PyTDC](https://tdc.readthedocs.io/) — `tdc.single_pred.ADME` / `Tox` |
| Reprezentacje molekularne | **RDKit** — ECFP4 (`AllChem.GetMorganFingerprintAsBitVect`, 1024 bit), 10-cechowe deskryptory 2D (MW, LogP, HBD, HBA, TPSA, RotatableBonds, AromaticRings, HeavyAtoms, MolMR, FractionCSP3); **MoLFormer** — embeddingi pretrenowane |
| Modele klasyczne | scikit-learn — `RandomForestRegressor` / `RandomForestClassifier` z `RandomizedSearchCV` |
| Sieci neuronowe | PyTorch — `AdmetEncoder` (Linear → LayerNorm → ReLU → Dropout) z opcjonalnym połączeniem rezydualnym + głowica regresji / klasyfikacji (sigmoid) |
| Trening NN | Adam (lr 1e-3 / 5e-4), 100 epok, MSE / BCELoss, `StandardScaler` na etykietach regresji |
| Metryki | scikit-learn — RMSE, MAE, R² (regresja); Accuracy, F1, AUROC (klasyfikacja) |
| Pomocnicze | pandas, numpy, fuzzywuzzy, pickle (zapis splitów) |

## Struktura repo

```
ADMET-predictor/
├── README.md
├── LICENSE
├── predykcjaADMET_raport.docx.pdf      # raport końcowy z opisem metodologii i wyników
│
├── data_splits/                        # wspólne train/test splity (.pkl) — ten sam podział dla wszystkich modeli
│   ├── AMES_split.pkl
│   ├── CYP3A4_Veith_split.pkl
│   ├── Caco2_Wang_split.pkl
│   ├── Clearance_Hepatocyte_AZ_split.pkl
│   ├── HIA_Hou_split.pkl
│   ├── Half_Life_Obach_split.pkl
│   ├── Lipophilicity_AstraZeneca_split.pkl
│   ├── Solubility_AqSolDB_split.pkl
│   ├── VDss_Lombardo_split.pkl
│   └── hERG_split.pkl
│
├── STL_NN/                             # Single-Task Learning, sieci neuronowe (PyTorch)
│   ├── STL_NN_fingerprints.ipynb       # NN na ECFP4 (1024 bit)
│   ├── STL_NN_descriptors.ipynb        # NN na 10 deskryptorach 2D
│   ├── STL_NN_embeddings.ipynb         # NN na embeddingach MoLFormer
│   ├── metrics_NN_fingerprints.txt
│   ├── metrics_NN_descriptors.txt
│   └── metrics_NN_embeddings.txt
│
├── STL_RF/                             # Single-Task Learning, Random Forest (sklearn) — TODO
│   └── (notebooki dla 3 reprezentacji + plik metryk)
│
├── MTL_ML/                             # Multi-Task Learning — TODO
│   └── (notebooki MTL na grupach endpointów: absorpcja, eliminacja, toksyczność)
│
└── results/                            # zbiorcze metryki wszystkich modeli
    ├── metrics_fingerprints.txt
    ├── metrics_ADMET_featurizer.txt    # = deskryptory
    ├── metrics_MoLFormer_embeddings.txt
    └── visualisation_endpoints.png     # wizualizacja rozkładów / liczności endpointów
```

## Jak uruchomić

1. **Środowisko** — notebooki przygotowane pod Google Colab (mount Google Drive). Lokalnie wymagają:
   ```bash
   pip install torch rdkit pandas numpy scikit-learn pytdc fuzzywuzzy
   ```
2. **Splity** — pliki `data_splits/*.pkl` należy umieścić w folderze wskazanym przez zmienną `data_folder` (w Colabie: `/content/drive/MyDrive/mldd_data/`). Gwarantuje to identyczny podział train/test dla każdego modelu i porównywalność wyników.
3. **Embeddingi MoLFormer** — wymagają wcześniejszego wygenerowania pliku `{endpoint}_MoLFormer_embeddings.csv` (kolumny `Drug`, `Y`, `emb_0…emb_N`). Pipeline generujący znajduje się w katalogu roboczym (`STL_ML/embeddings_molformer.ipynb`).
4. **Uruchomienie** — każdy notebook iteruje po 10 endpointach i dopisuje metryki do odpowiedniego pliku `metrics_*.txt`.

## Architektura sieci neuronowej

```python
class AdmetEncoder(nn.Module):
    # Linear → LayerNorm → ReLU → Dropout, z opcjonalnym połączeniem rezydualnym
    def forward(self, x):
        x = self.main(x)
        return torch.relu(x + self.res_layer(x))   # wariant „with residual connection"
        # return torch.relu(self.res_layer(x))     # wariant „without residual connection"
```

Identyczna szkieletowa architektura używana jest dla wszystkich trzech reprezentacji — różni się tylko `input_dim` (1024 dla FP, 10 dla deskryptorów, wymiar embeddingu MoLFormer dla embeddingów).

---

## Wyniki — NN z połączeniem rezydualnym 

Porównanie tej samej architektury w dwóch wariantach (**+Res** = z połączeniem rezydualnym, **−Res** = bez) na trzech reprezentacjach. Pokazany jest **R²** dla zadań regresji oraz **AUROC** dla klasyfikacji (im wyżej, tym lepiej). Pogrubiona wartość = najlepszy wynik w wierszu.

| Endpoint | Zadanie | Metryka | FP +Res | FP −Res | Desc +Res | Desc −Res | Emb +Res | Emb −Res |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Caco2_Wang | reg | R² | 0.6155 | 0.6047 | 0.6428 | 0.5965 | 0.6442 | **0.6658** |
| Lipophilicity_AstraZeneca | reg | R² | 0.5596 | 0.5734 | 0.4568 | 0.3963 | 0.6323 | **0.6637** |
| Solubility_AqSolDB | reg | R² | 0.7058 | 0.7067 | 0.7319 | 0.7060 | 0.7743 | **0.7762** |
| HIA_Hou | clf | AUROC | 0.8649 | 0.8619 | 0.9300 | 0.9729 | 0.9881 | **0.9897** |
| Half_Life_Obach | reg | R² | **0.4945** | 0.4532 | 0.3206 | 0.2035 | 0.3856 | 0.4116 |
| Clearance_Hepatocyte_AZ | reg | R² | 0.0368 | 0.0161 | 0.1069 | 0.1228 | **0.4474** | 0.3648 |
| CYP3A4_Veith | clf | AUROC | 0.8757 | 0.8780 | 0.8236 | 0.8081 | 0.8788 | **0.8814** |
| VDss_Lombardo | reg | R² | **0.2606** | −1.2274 | −1.6535 | −0.8973 | −0.1870 | 0.0748 |
| AMES | clf | AUROC | **0.8878** | 0.8836 | 0.8085 | 0.7734 | 0.8812 | 0.8814 |
| hERG | clf | AUROC | 0.7948 | 0.7556 | 0.7883 | 0.7797 | 0.8692 | **0.8809** |

### Wnioski

- **Embeddingi MoLFormer** wygrywają w **8/10** endpointów — pretrenowana reprezentacja transferuje wiedzę chemiczną, której proste FP/deskryptory nie zawierają.
- **Połączenie rezydualne** *nie* poprawia wyników w sposób systematyczny — przy pełnych, gęstych reprezentacjach (embeddingi) wariant **bez** residual jest często lekko lepszy (6/10 endpointów). Przy małych zbiorach regresji (VDss, Half_Life) wariant z residual stabilizuje trening i zapobiega katastrofalnemu spadkowi R² do wartości ujemnych.
- Ujemne R² na **VDss_Lombardo** dla większości wariantów wskazuje, że model przewiduje gorzej niż średnia — endpoint wymaga osobnego potraktowania (mały zbiór, duża wariancja).
- **Fingerprinty ECFP4** pozostają konkurencyjne dla zadań klasyfikacji wymagających wzorców podstrukturalnych (AMES, CYP3A4).
