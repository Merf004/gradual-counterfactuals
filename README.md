# Explicabilité contrefactuelle des modèles de régression

Ce dépôt contient le code expérimental du mémoire *"Explicabilité contrefactuelle des
modèles de régression"* (Master 2 Informatique, Université de Yaoundé I). La méthode
proposée génère des contrefactuels pour des modèles de régression en s'appuyant sur des
motifs graduels extraits des données d'entraînement, puis les compare à quatre méthodes
de référence de l'état de l'art (DiCE, MOC, NICE, What-If).

## Sommaire

1. [Structure du dépôt](#structure-du-dépôt)
2. [Prérequis](#prérequis)
3. [Installation](#installation)
4. [Installation R (notebooks de comparaison)](#installation-r-notebooks-de-comparaison)
5. [Ordre d'exécution](#ordre-dexécution)
6. [Détail des notebooks](#détail-des-notebooks)
7. [Paramètres importants à respecter](#paramètres-importants-à-respecter)
8. [Fichiers générés (outputs/)](#fichiers-générés-outputs)
9. [Dépannage](#dépannage)
10. [Reproductibilité](#reproductibilité)

---

## Structure du dépôt

```
gradual-counterfactuals/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── automobile/imports-85.data
│   ├── concrete/concrete_data.csv
│   └── movies/movie_data.csv
├── notebooks/
│   ├── automobile/
│   │   ├── 1_SVM_Model.ipynb
│   │   ├── 2_TG_Patterns.ipynb
│   │   ├── 3_Contrefactuels.ipynb
│   │   ├── 4_DiCE.ipynb        (Python)
│   │   ├── 5_MOC.ipynb         (R)
│   │   ├── 6_NICE.ipynb        (R)
│   │   └── 7_WhatIf.ipynb      (R)
│   ├── concrete/
│   │   ├── 1_SVM_Model.ipynb
│   │   ├── 2_TG_Patterns.ipynb
│   │   └── 3_Contrefactuels.ipynb
│   └── movies/
│       ├── 1_SVM_Model.ipynb
│       ├── 2_TG_Patterns.ipynb
│       └── 3_Contrefactuels.ipynb
└── outputs/
    ├── automobile/   (généré à l'exécution : modèle, splits, motifs, contrefactuels, graphes, métriques)
    ├── concrete/
    └── movies/
```

Les méthodes de comparaison (DiCE, MOC, NICE, What-If) ne sont exécutées que sur le
jeu de données **Automobile**, sur une seule instance, conformément au protocole du
mémoire (section 4.4).

---

## Prérequis

- **Python 3.12** (recommandé — testé avec 3.12.9)
- **R** (≥ 4.0), uniquement nécessaire pour les notebooks de
  comparaison MOC/NICE/What-If (Automobile)
- **Git**
- Environ 500 Mo d'espace disque (jeux de données + modèles entraînés)

---

## Installation

### 1. Cloner le dépôt

```bash
git clone https://github.com/Merf004/gradual-counterfactuals.git
cd gradual-counterfactuals
```

### 2. Créer et activer l'environnement virtuel Python

**Windows (PowerShell) :**
```powershell
py -3.12 -m venv venv
.\venv\Scripts\Activate.ps1
```
Si l'activation échoue avec une erreur de policy d'exécution PowerShell, utilise
`venv\Scripts\activate.bat` dans une invite `cmd` classique à la place.

**macOS / Linux :**
```bash
python3.12 -m venv venv
source venv/bin/activate
```

Une fois activé, ton invite de commande doit afficher `(venv)` au début de la ligne.

### 3. Vérifier que le venv est bien actif

```bash
python -c "import sys; print(sys.executable)"
```
Le chemin affiché doit pointer vers `.../gradual-counterfactuals/venv/...`. Si ce
n'est pas le cas, l'environnement n'est pas correctement activé — recommence l'étape 2
avant de continuer.

### 4. Installer les dépendances

Important : utilise toujours `python -m pip`, jamais `pip` seul (sous Windows,
`pip` peut résoudre vers un autre interpréteur Python que celui du venv actif) :

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### 5. Vérifier l'installation

```bash
python -m pip show pandas numpy scikit-learn mlxtend joblib matplotlib seaborn
```
Chaque bibliothèque doit afficher un numéro de version. Si l'une d'elles manque,
relance l'étape 4.

### 6. Lancer Jupyter

```bash
jupyter notebook
```
ou, si tu utilises VS Code, ouvre simplement les fichiers `.ipynb` avec l'extension
Jupyter installée (sélectionne l'interpréteur `venv` comme kernel).

---

## Installation R (notebooks de comparaison)

Nécessaire uniquement pour `5_MOC.ipynb`, `6_NICE.ipynb`, `7_WhatIf.ipynb`
(Automobile). Dans la console R (RStudio), à exécuter une seule fois :

```r
install.packages("reticulate")
install.packages("data.table")
install.packages("remotes")
remotes::install_cran("iml")
remotes::install_github("dandls/counterfactuals")
```

### Lier R au venv Python du projet

Dans chaque notebook R (`5_MOC.ipynb`, `6_NICE.ipynb`, `7_WhatIf.ipynb`), la
Cellule 1 contient une ligne `use_python(...)` qu'il faut adapter au chemin exact de
ton venv. Pour retrouver ce chemin :

```powershell
# Terminal, venv active
(Get-Command python).Source
```

Colle ce chemin exact dans la Cellule 1, par exemple :
```r
use_python("D:/chemin/vers/gradual-counterfactuals/venv/Scripts/python.exe", required = TRUE)
print(py_config())
```

Le `print(py_config())` doit afficher une ligne `numpy:` et une ligne `joblib:` avec un
chemin et un numéro de version (pas `[NOT FOUND]`). Si ce n'est pas le cas, va
directement à la section Dépannage.

Point critique : `reticulate` fige la configuration Python une seule fois par
session R. Si tu modifies quoi que ce soit dans le venv Python (installation d'un
paquet, correction d'un chemin) après avoir déjà exécuté une cellule R qui touche à
Python, tu dois redémarrer complètement R (Session → Restart R) puis réexécuter
le notebook depuis la Cellule 1.

---

## Ordre d'exécution

Pour chacun des trois jeux de données (Automobile, Concrete, Movies), les
notebooks doivent être exécutés dans cet ordre précis, car chacun dépend des
fichiers produits par le précédent :

```
1_SVM_Model.ipynb      →  entraîne le modèle, sauvegarde X_train/X_test/y_train/y_test
        ↓
2_TG_Patterns.ipynb    →  extrait les motifs graduels (sur X_train uniquement)
        ↓
3_Contrefactuels.ipynb →  génère les contrefactuels, calcule les métriques
```

`1_SVM_Model.ipynb` ne doit être exécuté qu'UNE SEULE FOIS par jeu de données —
c'est lui qui fixe le split train/test (graine aléatoire fixe) que tous les autres
notebooks réutilisent. Ne le relance pas après coup, sous peine de changer le split et
de rendre les motifs/contrefactuels incohérents avec les nouvelles données de test.

### Pour Automobile uniquement, après le notebook 3

```
4_DiCE.ipynb    (Python)
5_MOC.ipynb     (R)
6_NICE.ipynb    (R)
7_WhatIf.ipynb  (R)
```

Ces quatre notebooks peuvent être exécutés dans n'importe quel ordre entre eux, mais
après `3_Contrefactuels.ipynb`, car ils réutilisent le modèle et les splits générés
par `1_SVM_Model.ipynb`, et doivent viser exactement la même instance et la même
cible `y'` que celles choisies dans `3_Contrefactuels.ipynb` (Cellule 6) — voir la
section suivante.

### Récapitulatif complet (les 9 + 4 notebooks)

```
notebooks/automobile/1_SVM_Model.ipynb
notebooks/automobile/2_TG_Patterns.ipynb
notebooks/automobile/3_Contrefactuels.ipynb
notebooks/automobile/4_DiCE.ipynb
notebooks/automobile/5_MOC.ipynb
notebooks/automobile/6_NICE.ipynb
notebooks/automobile/7_WhatIf.ipynb

notebooks/concrete/1_SVM_Model.ipynb
notebooks/concrete/2_TG_Patterns.ipynb
notebooks/concrete/3_Contrefactuels.ipynb

notebooks/movies/1_SVM_Model.ipynb
notebooks/movies/2_TG_Patterns.ipynb
notebooks/movies/3_Contrefactuels.ipynb
```

---

## Détail des notebooks

### `1_SVM_Model.ipynb`
Charge et nettoie les données brutes, effectue le split train/test (80/20, graine
fixe), entraîne un SVR (noyau RBF, hyperparamètres par défaut, avec standardisation
interne des features et de la cible), évalue le modèle en holdout (RMSE, MAE, R²,
MAPE), puis sauvegarde le modèle (`svm_model.pkl`) et les splits
(`X_train.csv`, `X_test.csv`, `y_train.csv`, `y_test.csv`) dans `outputs/<dataset>/`.

### `2_TG_Patterns.ipynb`
Charge `X_train`/`y_train` uniquement, définit et appelle la fonction
`extraire_motifs_graduels(...)` (une seule fonction, appelée deux fois : une pour les
motifs ascendants, une pour les descendants), puis sauvegarde
`motifs_ascendants.csv` et `motifs_descendants.csv`.

### `3_Contrefactuels.ipynb`
Charge le modèle, les motifs, `X_train`/`X_test`. Définit la fonction unique
`generer_contrefactuels(...)` (Phase 3 du mémoire). Génère les contrefactuels pour
une instance choisie (`INDEX_INSTANCE`), affiche/sauvegarde le tableau détaillé et le
graphe de convergence, calcule les métriques (validité, proximité, sparsité,
diversité) sur cette instance, puis évalue la méthode sur l'ensemble des 20 % de
test, en moyennant les métriques.

Cette dernière étape (boucle sur tout le test set) peut prendre plusieurs minutes
selon le jeu de données (les cas où aucun contrefactuel valide n'existe demandent une
recherche exhaustive avant d'abandonner — c'est normal, pas un bug).

### `4_DiCE.ipynb`, `5_MOC.ipynb`, `6_NICE.ipynb`, `7_WhatIf.ipynb`
Reproduisent, chacun avec sa propre méthode de référence, la génération de
contrefactuels et le calcul des métriques sur la même instance et la même cible `y'`
que celle utilisée dans `3_Contrefactuels.ipynb`, afin de produire un tableau
comparatif équivalent au Tableau 4.17 du mémoire.

---

## Paramètres importants à respecter

| Paramètre | Valeur | Où | Remarque |
|---|---|---|---|
| `RANDOM_STATE` | 42 | Notebook 1 (tous datasets) | Fixe le split train/test — ne pas changer entre relances |
| `TEST_SIZE` | 0.2 | Notebook 1 | 80 % train / 20 % test |
| `MIN_SUPPORT` | 0.25 | Notebook 2 | Support minimal Apriori |
| `MIN_SEQ_SIZE` | 2 | Notebook 2 | Taille minimale d'une séquence consécutive |
| `EPSILON` | 1e-5 | Notebook 3 | Seuil de convergence de la dichotomie |
| `MAX_ITER` | 100 | Notebook 3 | Itérations max de la dichotomie |
| `MAX_HOPS` | 4 | Notebook 3 | Nombre max de motifs chaînés |
| `SEUIL_VALIDITE` | 0.01 | Notebook 3 + 4/5/6/7 | Tolérance de validité (mis à jour depuis 0.03) |
| `INDEX_INSTANCE` / `DELTA` (Automobile) | à fixer une fois | Notebook 3, puis reporté à l'identique dans 4/5/6/7 | Doit être identique partout pour une comparaison valide |

Attention : si tu changes `INDEX_INSTANCE` ou `DELTA` dans
`3_Contrefactuels.ipynb` (Automobile), reporte exactement les mêmes valeurs dans les
quatre notebooks de comparaison, sinon le tableau comparatif final n'aura aucun sens
(chaque méthode expliquerait une instance/cible différente).

---

## Fichiers générés (outputs/)

Pour chaque dataset, après exécution complète :

```
outputs/<dataset>/
├── svm_model.pkl                    (notebook 1)
├── X_train.csv, X_test.csv          (notebook 1)
├── y_train.csv, y_test.csv          (notebook 1)
├── metriques_svm.csv                (notebook 1 — RMSE, MAE, R2, MAPE)
├── motifs_ascendants.csv            (notebook 2)
├── motifs_descendants.csv           (notebook 2)
├── contrefactuels_instance_X.csv    (notebook 3 — instance détaillée)
├── graphe_instance_X.png            (notebook 3)
└── metriques_test_set.csv           (notebook 3 — moyenne sur tout le test set)
```

Pour Automobile uniquement, en plus :
```
outputs/automobile/
├── contrefactuels_dice.csv     (notebook 4)
├── contrefactuels_moc.csv      (notebook 5)
├── contrefactuels_nice.csv     (notebook 6)
└── contrefactuels_whatif.csv   (notebook 7)
```

---

## Dépannage

**`ModuleNotFoundError: No module named 'joblib'` dans un notebook R**
→ `reticulate` n'utilise pas le venv du projet. Vérifie le chemin dans
`use_python(..., required = TRUE)`, redémarre complètement R (Session → Restart R),
et réexécute le notebook depuis le début. Vérifie aussi que l'installation a bien été
faite avec `python -m pip install -r requirements.txt` (et pas `pip install` seul,
qui peut résoudre vers un autre Python que celui du venv sous Windows).

**`Required version of NumPy not available` dans un notebook R**
→ Teste d'abord `python -c "import numpy; print(numpy.__version__)"` dans un terminal
(hors R) pour vérifier que numpy fonctionne côté Python pur. Si ça échoue, réinstalle
numpy (`python -m pip install numpy==2.4.4`) ou recrée le venv proprement. Si ça
fonctionne côté terminal mais pas dans R, c'est que la session R a figé une ancienne
configuration : redémarre R et relance depuis la Cellule 1.

**Warnings `UserWarning: X does not have valid feature names` qui noient les logs**
→ Déjà géré dans le code fourni via `warnings.filterwarnings(...)` (Python) et
`py_run_string("import warnings; ...")` (R). Si ça réapparaît, vérifie que cette ligne
est bien présente dans la Cellule 1 de chaque notebook concerné.

**Le notebook 3 (batch sur le test set) semble bloqué / très lent**
→ Normal pour certaines instances où aucun contrefactuel valide n'existe (recherche
exhaustive avant abandon). Vérifie les logs affichés tous les 5–10 instances pour
confirmer que la boucle progresse.

**`pip install -r requirements.txt` échoue avec des erreurs de version Python**
→ Signe que `pip` pointe vers un autre Python que celui du venv actif. Utilise
toujours `python -m pip install ...` plutôt que `pip install ...` seul.

---

## Reproductibilité

- Le split train/test est déterministe (`random_state=42`), fixé une fois pour toutes
  dans `1_SVM_Model.ipynb` de chaque dataset.
- Les motifs graduels sont extraits uniquement sur `X_train`/`y_train`, jamais sur le
  test, pour éviter toute fuite d'information.
- Toutes les versions de bibliothèques utilisées sont fixées dans `requirements.txt`.
- Les notebooks affichent des logs (`print`) à chaque étape importante, pour suivre
  la progression et identifier facilement où se trouve l'exécution en cas de blocage.

---

## Citation

Ce dépôt accompagne le mémoire de Master 2 :

> FEDIM PIOMO Merlin Brice, *Explicabilité contrefactuelle des modèles de régression*,
> Mémoire de Master en Informatique, Université de Yaoundé I, sous la direction du
> Pr Norbert TSOPZE, année académique 2025/2026.