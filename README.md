# ADMET-predictor

> ⚠️ **Repozytorium w trakcie finalizacji (stan na 07.06.2026).** Porządkujemy jeszcze
> notebooki i uzupełniamy dokumentację wyników przed prezentacją semestralną. Prosimy
> prowadzących o wstrzymanie się z oceną do finalnego commita — dziękujemy!

Projekt realizowany w ramach kursu **„Uczenie maszynowe w projektowaniu leków" 2025/2026**.
Autorzy: Jakub Cieniuch, Laura Musioł, Ivan Verbovetskyi.

## O projekcie

Profil **ADMET** (Absorption, Distribution, Metabolism, Excretion, Toxicity) opisuje pełny cykl życia cząsteczki w organizmie - od wchłonięcia do krwi i dystrybucji do tkanek, przez przemiany metaboliczne w wątrobie, aż po wydalenie i ocenę bezpieczeństwa. Parametry te decydują jednocześnie o **skuteczności terapeutycznej** leku (czy dotrze do celu w odpowiednim stężeniu) i o **bezpieczeństwie pacjenta**. Wykorzystanie metod komputerowych (*in silico*) do predykcji profilu ADMET na wczesnym etapie badań pozwala szybko odsiać ryzykowne kandydaty, dzięki czemu cały proces tworzenia leku staje się tańszy i krótszy.

Celem projektu jest **porównanie podejścia jednozadaniowego (STL - Single-Task Learning) z podejściem wielozadaniowym (MTL - Multi-Task Learning)** w predykcji 10 endpointów ADMET pochodzących z benchmarku [TDC (Therapeutics Data Commons)](https://tdc.readthedocs.io/). Każdy endpoint trenowany jest:

- na **trzech reprezentacjach molekularnych**: ECFP4 fingerprinty, 10-cechowe deskryptory 2D oraz pretrenowane embeddingi z modelu **MoLFormer**,
- przy użyciu **dwóch rodzin modeli**: klasycznego Random Forest oraz sieci neuronowych w PyTorch (z opcjonalnym połączeniem rezydualnym),
- a w przypadku MTL - w **trzech zestawach tematycznych** (Absorpcja, Eliminacja, Kardiotoksyczność), w których endpointy są pogrupowane według mechanizmów biologicznych.

### Hipotezy badawcze

Na podstawie analizy literatury postawiliśmy następujące hipotezy:

1. Modele MTL osiągają **lepszą średnią jakość predykcji** niż STL - szczególnie dla endpointów z małą liczbą danych treningowych.
2. **Największa poprawa** wyników wystąpi dla najmocniej powiązanych biologicznie endpointów.
3. W przypadku endpointów niepowiązanych pojawi się **transfer negatywny** (MTL pogorszy wyniki względem STL).
4. **Wybór reprezentacji molekularnej** silnie wpływa na skuteczność predykcji - różnice powinny być większe niż między samymi modelami.

**Hipotezy dodatkowe (własne):**

5. **Funkcja ważenia straty** w MTL (sieci neuronowe) wpływa na jakość predykcji - adaptacyjne ważenie niepewnością (*Uncertainty Weighting*, Kendall et al.) powinno radzić sobie lepiej niż prosta suma strat czy uśrednianie, bo zapobiega zdominowaniu gradientu przez zadania o dużej skali błędu.
6. **Wybór rodziny modelu** (Random Forest vs sieć neuronowa) ma znaczenie - sieci powinny lepiej wykorzystać gęste reprezentacje (embeddingi), a RF reprezentacje rzadkie/podstrukturalne (fingerprinty).

> **Wynik dodatkowy - użyteczność MTL przy niedoborze danych:** w eksperymencie kontrolnym ("głód danych"), gdy dla zadania docelowego dostępnych było tylko **10%** zbioru hERG, MTL podniósł AUROC z **0.41 (STL) do 0.84 (MTL)** - najmocniejszy dowód, że MTL pomaga najbardziej tam, gdzie danych jest mało.

### Podejście - dwie fazy

| Faza | Co | Po co |
|---|---|---|
| **I - Baseline STL** | 2 modele (RF, NN) × 3 reprezentacje × 10 endpointów = 60 scenariuszy | wyznaczenie wartości referencyjnej dla każdego endpointu |
| **II - Eksperymenty MTL** | łączenie endpointów w 3 zestawy tematyczne, te same splity train/test co w STL | sprawdzenie, w których konfiguracjach MTL pomaga, a w których szkodzi |

## Endpointy

Lista 10 endpointów dobranych tak, aby reprezentowały zróżnicowane zadania (regresja + klasyfikacja) i zbiory o różnych licznościach.

| # | Endpoint | Zadanie | Co mierzy |
|---|---|:---:|---|
| 1 | Caco2_Wang | regresja | przepuszczalność komórek jelitowych |
| 2 | Lipophilicity_AstraZeneca | regresja | lipofilowość (rozpuszczalność w tłuszczach) |
| 3 | Solubility_AqSolDB | regresja | rozpuszczalność wodna |
| 4 | HIA_Hou | klasyfikacja | wchłanianie z przewodu pokarmowego |
| 5 | Half_Life_Obach | regresja | okres półtrwania leku |
| 6 | Clearance_Hepatocyte_AZ | regresja | szybkość klirensu wątrobowego |
| 7 | CYP3A4_Veith | klasyfikacja | inhibicja CYP3A4 (enzym metabolizujący 50% leków) |
| 8 | VDss_Lombardo | regresja | objętość dystrybucji w tkankach |
| 9 | AMES | klasyfikacja | mutagenność (Ames test) |
| 10 | hERG | klasyfikacja | kardiotoksyczność (blok kanału hERG) |

**Metryki**: regresja → RMSE / MAE / R²; klasyfikacja → Accuracy / F1 / AUROC.

### Zestawy MTL

| Zestaw | Endpointy powiązane | Endpoint kontrolny (niepowiązany) |
|---|---|---|
| **1 - Absorpcja** | Caco-2, HIA, Solubility, Lipophilicity | AMES |
| **2 - Eliminacja** | Half Life, Clearance Hepatocyte, CYP3A4 Inhibition, VDss | AMES |
| **3 - Kardiotoksyczność** | hERG, Lipophilicity, Solubility, VDss | AMES |

W każdym zestawie świadomie dodaliśmy **AMES** jako endpoint biologicznie niepowiązany — pozwala to bezpośrednio zmierzyć transfer negatywny.

## Tech stack

Stos technologiczny pogrupowany według warstw, zgodnie ze schematem projektowym:

| Warstwa | Narzędzia |
|---|---|
| **Środowisko** | Python 3.12, Jupyter / Google Colab (GPU T4) |
| **Cheminformatyka** | [**RDKit**](https://www.rdkit.org/) - parsowanie SMILES, Morgan/ECFP fingerprints, deskryptory 2D, walidacja cząsteczek<br>[**PyTDC**](https://tdc.readthedocs.io/) — pobieranie zbiorów ADMET (`tdc.single_pred.ADME` / `Tox`) |
| **Reprezentacje molekularne** | **ECFP4** - `AllChem.GetMorganFingerprintAsBitVect`, 1024 bity, radius 2<br>**Deskryptory 2D** - 10 cech z RDKit: MW, LogP, HBD, HBA, TPSA, RotatableBonds, AromaticRings, HeavyAtoms, MolMR, FractionCSP3<br>**MoLFormer** - embeddingi z pretrenowanego modelu `ibm/MoLFormer-XL-both-10pct` (HuggingFace Transformers) |
| **Modelowanie - klasyczne** | [**scikit-learn**](https://scikit-learn.org/) - `RandomForestRegressor` / `RandomForestClassifier`, dobór hiperparametrów przez `RandomizedSearchCV` (n_estimators, max_depth, max_features, min_samples_split) |
| **Modelowanie - sieci neuronowe** | [**PyTorch**](https://pytorch.org/) - własny `AdmetEncoder` (Linear + LayerNorm + ReLU + Dropout, opcjonalny residual) z trzema typami głowic: STL_Regressor, STL_Classifier, MTL_Hybrid (`nn.ModuleDict` na endpoint)<br>Trening: optymalizator Adam (lr 1e-3 / 5e-4), 100 epok, MSELoss / BCEWithLogitsLoss, `StandardScaler` na etykietach regresji |
| **Modelowanie - embeddingi** | [**HuggingFace Transformers**](https://huggingface.co/docs/transformers) - załadowanie i inferencja MoLFormer |
| **Analiza danych** | NumPy, Pandas, Matplotlib - manipulacja tabelami, wizualizacja rozkładów endpointów, wykresy diagnostyczne |
| **Serializacja** | `pickle` - zapis splitów train/test do plików `.pkl`, gwarantuje **identyczne** podziały danych dla wszystkich modeli i porównywalność wyników |
| **Metryki** | scikit-learn - `mean_squared_error`, `mean_absolute_error`, `r2_score`, `accuracy_score`, `f1_score`, `roc_auc_score` |

## Architektura sieci neuronowej

Wszystkie modele neuronowe (STL i MTL) dzielą **wspólny szkielet**: enkoder, który mapuje wektor cech molekularnych do 512-wymiarowej przestrzeni ukrytej, oraz jedną lub więcej głowic predykcyjnych.

```
                   SMILES
                     │
                     ▼
        ┌─────────────────────────┐
        │     Featurizer          │   ECFP4 (1024) / Desc (10) / Emb (768)
        └─────────────────────────┘
                     │
                     ▼  wektor cech (input_dim)
        ┌─────────────────────────┐
        │     AdmetEncoder        │
        │  ┌───────────────────┐  │
        │  │ Linear(in→512)    │  │
        │  │ LayerNorm(512)    │  │   blok główny
        │  │ ReLU              │  │
        │  │ Dropout(0.2)      │  │
        │  └───────────────────┘  │
        │            │            │
        │            ▼            │
        │  ┌───────────────────┐  │
        │  │ Linear(512→512)   │  │   warstwa rezydualna (opcja)
        │  └───────────────────┘  │
        │            │            │
        │       x + res(x)        │   skrót rezydualny
        │            │            │
        │           ReLU          │
        └─────────────────────────┘
                     │
                     ▼  cechy 512-d (wspólna przestrzeń reprezentacji)
                     │
        ┌────────────┴────────────┐
        │                         │
   ┌────▼─────┐             ┌─────▼─────┐
   │ STL head │             │ MTL heads │
   │ Linear→1 │             │ ModuleDict│   po jednej głowicy na endpoint
   └──────────┘             └───────────┘
```

### Trzy warianty głowic

| Wariant | Głowica | Wyjście | Kiedy |
|---|---|---|---|
| **STL_Regressor** | `Linear(512, 1)` | wynik liniowy | jeden endpoint regresyjny (np. Caco-2) |
| **STL_Classifier** | `Linear(512, 1)` + sigmoid w eval | prawdopodobieństwo | jeden endpoint klasyfikacyjny (np. HIA) |
| **MTL_Hybrid** | `nn.ModuleDict` - osobna `Linear(512, 1)` per endpoint, oznaczona nazwą zadania | słownik `{task: pred}` | wiele endpointów naraz |

### Połączenie rezydualne - co testujemy

Encoder ma **dwa równoważne przepływy** - porównujemy je jako osobne warianty:

```python
# wariant +Res (z residualem):
out = ReLU( x_main + res_layer(x_main) )

# wariant -Res (bez residualu):
out = ReLU( res_layer(x_main) )
```

Hipoteza: residual stabilizuje gradient i pomaga przy małych zbiorach (mniejsze ryzyko zaniku gradientu w głębszej sieci); przy bogatych embeddingach może być zbędny lub nawet szkodliwy.

### Strategia treningu MTL z nieprzecinającymi się zbiorami

Endpointy ADMET często **nie pokrywają się cząsteczkowo** - ten sam SMILES rzadko występuje we wszystkich pickle'ach naraz. Stosujemy:

1. **Outer join** wszystkich SMILES z wybranych endpointów → jedna macierz `X` (po jednym wierszu na unikalną cząsteczkę).
2. **Słownik etykiet** `y_dict` - dla każdego task'a wektor długości `len(X)` z `NaN` tam, gdzie cząsteczki nie ma w danym endpoincie.
3. **Loss z reduction='none' + maska NaN** - gradient propaguje się tylko przez głowice tych zadań, dla których dana cząsteczka ma etykietę. Wspólny encoder jest aktualizowany przez **wszystkie** dostępne pary (cząsteczka, endpoint).
4. **Test set NIE jest outer-joinowany** - każdy endpoint ewaluujemy osobno na jego pełnym `test_split`, dla porównywalności z STL.

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
├── STL_RF/                             # Single-Task Learning, Random Forest (sklearn)
│   ├── STL_fingerprints_RF.ipynb       # RF na ECFP4
│   ├── STL_Descriptor_RF.ipynb         # RF na deskryptorach 2D
│   ├── STL_embeddings_RF.ipynb         # RF na embeddingach MoLFormer
│   ├── metrics.txt, metrics_ADMET_featurizer.txt
│   └── metryki/                        # metryki per reprezentacja
│
├── MTL_ML/                             # Multi-Task Learning, Random Forest
│   ├── MTL_fingerprints_*_RF.ipynb     # 3 zestawy: absorpcja / eliminacja / kardiotoksyczność
│   ├── MTL_descriptors_*_RF.ipynb
│   ├── MTL_embeddings_absorpcja_RF.ipynb
│   └── metryki/                        # wyniki MTL dla RF
│
├── MTL_NN/                             # Multi-Task Learning, sieci neuronowe (PyTorch)
│   ├── MTL_fingerprints_*.ipynb        # 3 reprezentacje × 3 zestawy + warianty
│   ├── MTL_descriptors_*_NN.ipynb      #   (m.in. testy funkcji ważenia straty:
│   ├── MTL_embeddings_*.ipynb          #    suma / uniform / uncertainty)
│   ├── MTL_loss_weighting_methods.pdf  # opis metod ważenia straty
│   └── metryki/                        # wyniki MTL dla sieci neuronowych
│
├── reports/                            # wygenerowane raporty PDF (porównania)
│   ├── Raport_Porownawczy_STL_MTL7.pdf       # STL vs MTL (RF+desc, NN+emb)
│   ├── Porownanie_Modeli_MTL_RF_vs_NN.pdf    # RF vs NN w trybie MTL
│   ├── Raport_Wynikow_ADMET_STL.pdf          # wyniki bazowe STL
│   ├── Raport_MTL_Znormalizowany_NRMSE9.pdf  # wpływ funkcji straty (NRMSE)
│   └── ... (raporty RF i NN MTL)
│
└── results/                            # zbiorcze metryki wszystkich modeli
    ├── metrics_fingerprints.txt
    ├── metrics_ADMET_featurizer.txt    # = deskryptory
    ├── metrics_MoLFormer_embeddings.txt
    └── visualisation_endpoints.png     # wizualizacja rozkładów / liczności endpointów
```

## Jak uruchomić

1. **Środowisko** - notebooki przygotowane pod Google Colab (mount Google Drive, `accelerator: GPU T4`). Lokalnie wymagają:
   ```bash
   pip install torch rdkit pandas numpy scikit-learn pytdc fuzzywuzzy transformers
   ```
2. **Splity** - pliki `data_splits/*.pkl` należy umieścić w folderze wskazanym przez zmienną `data_folder` (w Colabie: `/content/drive/MyDrive/mldd_data/`). Gwarantuje to identyczny podział train/test dla każdego modelu i porównywalność wyników.
3. **Embeddingi MoLFormer** - wymagają wcześniejszego wygenerowania pliku `{endpoint}_MoLFormer_embeddings.csv` (kolumny `Drug`, `Y`, `emb_0…emb_N`). Pipeline generujący znajduje się w katalogu roboczym (`STL_ML/embeddings_molformer.ipynb`).
4. **Uruchomienie** - każdy notebook iteruje po endpointach i dopisuje metryki do odpowiedniego pliku `metrics_*.txt`.

---

## Wyniki - NN z połączeniem rezydualnym vs. bez (slajd)

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

- **Embeddingi MoLFormer** wygrywają w **8/10** endpointów - pretrenowana reprezentacja transferuje wiedzę chemiczną, której proste FP/deskryptory nie zawierają.
- **Połączenie rezydualne** *nie* poprawia wyników w sposób systematyczny - przy pełnych, gęstych reprezentacjach (embeddingi) wariant **bez** residual jest często lekko lepszy (6/10 endpointów). Przy małych zbiorach regresji (VDss, Half_Life) wariant z residual stabilizuje trening i zapobiega katastrofalnemu spadkowi R² do wartości ujemnych.
- Ujemne R² na **VDss_Lombardo** dla większości wariantów wskazuje, że model przewiduje gorzej niż średnia - endpoint wymaga osobnego potraktowania (mały zbiór, duża wariancja).
- **Fingerprinty ECFP4** pozostają konkurencyjne dla zadań klasyfikacji wymagających wzorców podstrukturalnych (AMES, CYP3A4).
