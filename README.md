# 🛒 DemandCast — Moteur de prévision de la demande retail

> Prévoir les ventes à 30 jours par famille de produits et déclencher automatiquement des alertes email en cas de risque de rupture de stock.

![Python](https://img.shields.io/badge/Python-3.11-blue)
![Prophet](https://img.shields.io/badge/Prophet-1.3.0-orange)
![TensorFlow](https://img.shields.io/badge/TensorFlow-2.21-red)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-green)
![n8n](https://img.shields.io/badge/Automatisation-n8n-purple)

---

## 🎯 Objectif

En grande distribution, anticiper les ruptures de stock permet d'éviter des pertes
de chiffre d'affaires et des pénalités fournisseurs.

**DemandCast** combine un modèle de forecasting ML et un pipeline d'alertes automatiques
pour permettre aux équipes supply chain d'agir avant que le problème ne survienne.

---

## 🏗️ Architecture du projet
```demandcast/
│
├── data/
│   ├── raw/                  # Données brutes Kaggle
│   └── processed/            # Données nettoyées (ventes_clean.csv)
│
├── notebooks/
│   ├── 01_exploration.ipynb  # Analyse exploratoire
│   ├── 02_cleaning.ipynb     # Nettoyage et préparation
│   ├── 03_prophet.ipynb      # Modélisation Prophet × 5 familles
│   ├── 04_lstm.ipynb         # LSTM — 3 variantes comparées
│   └── 05_comparison.ipynb   # Comparaison et conclusion
│
├── api/
│   └── main.py               # API FastAPI — endpoint /forecast
│
├── models/
│   └── prophet_grocery.pkl   # Modèle Prophet sérialisé
│
├── exports/                  # Graphes PNG + CSV métriques
├── METHODOLOGIE.md           # Documentation complète du projet
└── README.md
```

---

## 📊 Dataset

**Source :** [Store Sales — Time Series Forecasting](https://www.kaggle.com/competitions/store-sales-time-series-forecasting) (Kaggle / Favorita)

| Paramètre | Valeur |
|---|---|
| Volume | ~3 millions de lignes |
| Période | Janvier 2013 — Août 2017 |
| Granularité | Journalière par magasin et famille de produits |
| Familles analysées | GROCERY I · BEVERAGES · PRODUCE · CLEANING · DAIRY |

---

## 🤖 Modèles

### Prophet (Meta) — Modèle retenu ✅

| Famille | MAPE | Évaluation |
|---|---|---|
| GROCERY I | 8.29% | ✅ Excellent |
| BEVERAGES | 10.24% | ✅ Bon |
| DAIRY | 10.12% | ✅ Bon |
| CLEANING | 10.98% | ✅ Bon |
| PRODUCE | 14.08% | ⚠️ Correct |

### LSTM (TensorFlow) — Comparaison ❌

Trois variantes testées sur GROCERY I :

| Variante | Features | MAPE |
|---|---|---|
| v1 — Baseline | Fenêtre glissante 30j | 62.37% |
| v2 — N-1 | v1 + ventes même période N-1 | 62.61% |
| v3 — Hybride | v2 + mois + jour semaine | 59.16% |

> **Conclusion :** Prophet surpasse le LSTM sur ce dataset (~1 700 points/famille).
> Le LSTM nécessite davantage de volume et une granularité plus fine pour exprimer sa puissance.

---

## 🔔 Pipeline d'alertes automatiques
```
⏰ Chaque lundi à 7h30 (n8n Schedule)
↓
🌐 HTTP Request → FastAPI /forecast?jours=30
↓
🔍 Code Node — détection rupture (seuil : 75% de la référence)
↓
📧 Email HTML automatique → équipe supply chain
```

**Logique d'alerte :** si les ventes prévues tombent sous 75% de la moyenne
des 30 derniers jours → alerte déclenchée avec liste des jours à risque.

---

## 🚀 Installation et lancement

### Prérequis
- Python 3.11
- Compte ngrok (pour exposer l'API)
- Compte n8n (pour les alertes)

### Installation

```bash
# Crée et active le venv
python -m venv C:\venv_demandcast
C:\venv_demandcast\Scripts\activate

# Installe les dépendances
pip install prophet tensorflow fastapi uvicorn scikit-learn pandas matplotlib seaborn pyngrok joblib
```

### Lancement de l'API

```bash
uvicorn api.main:app --reload --port 8000
```

API disponible sur : `http://localhost:8000/docs`

### Endpoint principal

```bash
GET /forecast?jours=30
```

**Réponse exemple :**
```json
{
  "famille": "GROCERY I",
  "nb_jours": 30,
  "alerte": false,
  "jours_a_risque": [],
  "previsions": [...]
}
```

---

## 🛠️ Stack technique

| Rôle | Technologie |
|---|---|
| Forecasting | Prophet 1.3.0 (Meta) |
| Deep Learning | TensorFlow 2.21 / Keras |
| Data | pandas · numpy · scikit-learn |
| API | FastAPI + Uvicorn |
| Automatisation | n8n |
| Tunnel | ngrok |
| Notifications | Gmail HTML |
| Environnement | Python 3.11 venv |

---

## 📁 Exports disponibles

| Fichier | Contenu |
|---|---|
| `metriques_prophet.csv` | MAE / RMSE / MAPE Prophet × 5 familles |
| `metriques_lstm.csv` | MAE / RMSE / MAPE LSTM × 3 variantes |
| `comparaison_modeles.csv` | Tableau comparatif complet |
| `conclusion_modeles.txt` | Analyse et recommandation rédigée |
| `04_prophet_*.png` | Graphes de prévisions par famille |
| `05_prophet_composantes_*.png` | Décomposition Prophet GROCERY I |
| `09_comparaison_mape.png` | Graphe comparatif MAPE |

---

## 📖 Documentation

Pour une explication complète des choix méthodologiques, des résultats
et des perspectives d'amélioration :

👉 [METHODOLOGIE.md](./METHODOLOGIE.md)

---

## 👤 Auteur

**Déhollin HOLLAT** — Chef de Projet Data IA  
MBA Big Data & IA  

---

*Projet réalisé dans le cadre d'un portfolio Data/AI — Avril 2025*
