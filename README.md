# ADMET-predictor

> ⚠️ **Repozytorium w trakcie finalizacji (stan na 14.06.2026).** Porządkujemy jeszcze
> notebooki i uzupełniamy dokumentację wyników. Prosimy
> prowadzących o wstrzymanie się z oceną do finalnego commita — dziękujemy!

Projekt realizowany w ramach kursu **„Uczenie maszynowe w projektowaniu leków" 2025/2026**.

Autorzy: Jakub Cieniuch, Laura Musioł, Ivan Verbovetskyi

## O projekcie

Profil **ADMET** (Absorption, Distribution, Metabolism, Excretion, Toxicity) opisuje pełny cykl życia cząsteczki w organizmie - od wchłonięcia do krwi i dystrybucji do tkanek, przez przemiany metaboliczne w wątrobie, aż po wydalenie i ocenę bezpieczeństwa. Parametry te decydują jednocześnie o **skuteczności terapeutycznej** leku (czy dotrze do celu w odpowiednim stężeniu) i o **bezpieczeństwie pacjenta**. Wykorzystanie metod komputerowych (*in silico*) do predykcji profilu ADMET  na wczesnym etapie badań pozwala na szybszą eliminację ryzykownych związków. Dzięki temu proces tworzenia leku staje się tańszy i szybszy, chroniąc przed kosztownymi niepowodzeniami w fazie testów klinicznych.

Celem projektu jest **porównanie podejścia jednozadaniowego (STL - Single-Task Learning) z podejściem wielozadaniowym (MTL - Multi-Task Learning)** w predykcji 10 endpointów ADMET pochodzących z benchmarku [TDC (Therapeutics Data Commons)](https://tdc.readthedocs.io/). Każdy endpoint trenowany jest:

- na **trzech reprezentacjach molekularnych**: fingerprinty ECFP4, 10-cechowe deskryptory 2D oraz pretrenowane embeddingi z modelu **MoLFormer**,
- przy użyciu **dwóch modeli**: klasycznego Random Forest oraz sieci neuronowych w PyTorch,
- a w przypadku MTL - w **trzech zestawach tematycznych** (Absorpcja, Eliminacja, Kardiotoksyczność), w których endpointy są pogrupowane według mechanizmów biologicznych.

### Hipotezy badawcze

1. Przewaga uczenia wielozadaniowego (MTL) nad modelami jednostkowymi (Single-Task Learning)
2. Transfer wiedzy między powiązanymi biologicznie parametrami 
3. Wpływ reprezentacji molekularnej na precyzję predykcji

**Hipotezy dodatkowe (własne):**

4. Wpływ funkcji straty na predykcję w sieci neuronowej MTL
5. Wpływ wyboru modelu (Random Forest vs sieć neuronowa) na jakość predykcji

### Dwie fazy projektu

| Faza | Co | Po co |
|---|---|---|
| **I - Baseline STL** | 2 modele (RF, NN) × 3 reprezentacje × 10 endpointów = 60 scenariuszy | wyznaczenie wartości referencyjnej dla każdego endpointu |
| **II - Eksperymenty MTL** | łączenie endpointów w 3 zestawy tematyczne, te same splity train/test co w STL | sprawdzenie, w których konfiguracjach MTL pomaga, a w których szkodzi |

## Wykorzystane endpointy

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

## Tech stack
Cheminformatyka: RDKit, TDC.
Modelowanie: PyTorch, scikit-learn, HuggingFace Transformers.
Analiza Danych: NumPy, Pandas, Matplotlib.

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

1. **Środowisko** - notebooki przygotowane pod Google Colab (mount Google Drive, `accelerator: GPU T4`). W notebookach już znajdują się komórki z komendami instalującymi wszystkie potrzebne biblioteki.
2. **Splity** - folder `data_splits` razem z plikami w nim należy umieścić na Google Dysku. Po uruchomieniu kodu Colab poprosi o dostęp do dysku, na którym będą się znajdować te dane, aby móc wytrenować modele.
3. **Embeddingi MoLFormer** - muszą zostać wygenerowane, ze względu na to, że GitHub nie zezwala na przesyłanie plików większych niż 25MB. Aby je wygenerować, należy stworzyć folder `data_splits` na Google Dysku, odpalić w Colabie skrypt `embeddings_molformer.ipynb`, a następnie nadać mu dostęp do Google Dysku. Embeddingi zostaną wygenerowane, a następnie umieszczone w `data_splits`.
4. **Uruchomienie** - każdy notebook iteruje po endpointach i dopisuje metryki do odpowiedniego pliku `metrics_*.txt`.
5. **Uwagi:**
   1. W przypadku błędów w Colabie rekomendujemy zrobić Restart Session.
   2. Trenowanie modeli Random Forest Single Task Learning może trwać długo ze względu na wykorzystanie sklearn, który wykorzystuje tylko CPU.

## Podsumowanie wyników
Uczenie wielozadaniowe (MTL) nie gwarantuje automatycznej poprawy i w prostych konfiguracjach daje wyniki zbliżone do modeli jednozadaniowych (STL). Jednak przy odpowiednim połączeniu zadań powiązanych biologicznie, MTL poprawia skuteczność predykcji, szczególnie dla małych i trudnych zbiorów danych (np. Half-Life). Przykładowo, włączenie do zestawu powiązanych parametrów eliminacji pozwoliło obniżyć błąd (RMSE) dla predykcji okresu półtrwania aż o 11% w przypadku sieci neuronowych i o 6% dla algorytmu Random Forest. Podobny, bardzo wyraźny zysk zanotowano przy ocenie wchłaniania (połączenie HIA i Caco-2), gdzie jakość klasyfikacji (AUROC) wzrosła o 9,3%. Należy jednak unikać łączenia zbyt wielu zróżnicowanych zadań, co może wprowadzać szum informacyjny i pogarszać wyniki. Doskonale obrazuje to przypadek parametru hERG – dołożenie do jego predykcji aż trzech dodatkowych właściwości sprawiło, że ostateczny wynik spadł poniżej pułapu wyznaczonego przez bazowy model STL (z 0.869 do 0.864).

Kluczowym czynnikiem decydującym o sukcesie jest również ścisłe dopasowanie reprezentacji molekularnej do wykorzystywanego algorytmu. Sieci neuronowe osiągają najwyższą skuteczność przy użyciu gęstych embeddingów, z kolei w przypadku algorytmu Random Forest w trybie MTL najlepiej sprawdzają się stabilne, niskowymiarowe deskryptory. Równie istotny jest dobór funkcji straty w sieciach neuronowych. Ponieważ poszczególne zadania drastycznie różnią się skalą (np. klasyfikacja vs regresja), zastosowanie adaptacyjnego ważenia strat (Uncertainty Weighting) pozwala modelowi automatycznie balansować te różnice.

📌 Szczegółowe wyniki eksperymentów, tabele, wykresy oraz bardziej obszerna analiza wszystkich postawionych hipotez znajdują się w plikach PDF w folderze "reports" oraz w prezentacji podsumowującej projekt.
