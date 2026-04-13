# Méthodologie — DemandCast

**Auteur :** Déhollin HOLLAT, Chef de Projet Data IA  
**Date :** Avril 2025  
**Objectif :** Système de prévision de la demande retail avec alertes automatiques de rupture de stock

---

## 1. Contexte et problématique

La rupture de stock est l'un des principaux risques opérationnels en grande distribution.
DemandCast répond à cette problématique en combinant un moteur de forecasting ML
et un pipeline d'alertes automatiques, permettant aux équipes supply chain d'anticiper
les tensions sur les stocks avant qu'elles ne se produisent.

**Question métier :** Comment prévoir les ventes à 30 jours par famille de produits
et déclencher automatiquement une alerte quand un risque de rupture est détecté ?

---

## 2. Dataset

| Paramètre | Valeur |
|---|---|
| Source | Kaggle — Store Sales Time Series Forecasting (Favorita) |
| Volume brut | ~3 millions de lignes |
| Période | Janvier 2013 — Août 2017 |
| Granularité | Journalière, par magasin et par famille de produits |
| Fichiers utilisés | train.csv, oil.csv, holidays_events.csv, stores.csv |

---

## 3. Nettoyage et préparation des données

### 3.1 Sélection des familles
Sur les 33 familles de produits disponibles, 5 ont été sélectionnées pour leur
représentativité et la diversité de leurs profils de ventes :

- **GROCERY I** — épicerie, famille la plus volumineuse
- **BEVERAGES** — boissons, forte saisonnalité
- **PRODUCE** — fruits et légumes, volatilité naturelle marquée
- **CLEANING** — produits d'entretien, comportement stable
- **DAIRY** — produits laitiers, achat régulier

### 3.2 Agrégation
Les ventes ont été agrégées par date et par famille (somme sur tous les magasins),
produisant une série temporelle par famille de 1 684 points.

### 3.3 Traitement des valeurs manquantes
- **Prix du pétrole (oil)** : 3.53% de valeurs manquantes (weekends et jours fériés
  non cotés) → interpolation linéaire + ffill/bfill pour les extrémités de série
- **Ventes nulles** : jours de fermeture collective identifiés sur toutes les familles
  → remplacement par ffill (propagation de la dernière valeur connue) pour maintenir
  la continuité de la série temporelle

### 3.4 Features enrichies
- `est_ferie` : variable binaire — jours fériés nationaux équatoriens
- `day_of_week` : jour de la semaine (saisonnalité hebdomadaire)
- `month` : mois de l'année (saisonnalité annuelle)

---

## 4. Modélisation

### 4.1 Prophet (Meta)

**Choix justifié :** Prophet est conçu pour les séries temporelles business avec
saisonnalité marquée et données manquantes. Il décompose automatiquement la prévision
en trois composantes interprétables.

**Paramètres retenus :**
- `yearly_seasonality=True` — saisonnalité annuelle (Noël, rentrée)
- `weekly_seasonality=True` — saisonnalité hebdomadaire (dimanche fort)
- `changepoint_prior_scale=0.05` — flexibilité modérée de la tendance
- Jours fériés équatoriens intégrés nativement (`add_country_holidays('EC')`)

**Composantes identifiées sur GROCERY I :**
- **Trend** : croissance quasi-linéaire de 150k à 260k unités (2013-2017)
- **Weekly** : dimanche +60k, jeudi -40k
- **Yearly** : pic de décembre (+80k), creux estival (juillet-août)
- **Holidays** : impact asymétrique fort (+100k veilles de fêtes, -200k fermetures)

### 4.2 LSTM — 3 variantes

Trois architectures ont été comparées pour évaluer l'apport des features saisonnières :

| Variante | Features | Architecture |
|---|---|---|
| v1 — Baseline | Ventes J-30 à J-1 | 2 couches LSTM (64→32) + Dropout 0.2 |
| v2 — N-1 | v1 + ventes même période N-1 | Identique, 3 features en entrée |
| v3 — Hybride | v2 + mois + jour de semaine | Identique, 5 features en entrée |

**Paramètres communs :**
- Fenêtre glissante : 30 jours
- Optimizer : Adam
- Loss : MSE
- Early stopping : patience=5 sur val_loss
- Normalisation : MinMaxScaler (0-1)

---

## 5. Résultats et comparaison

### 5.1 Métriques Prophet — toutes familles (test : 90 jours)

| Famille | MAE | RMSE | MAPE |
|---|---|---|---|
| GROCERY I | 21 528 | 29 293 | **8.29%** |
| BEVERAGES | 19 422 | 24 907 | 10.24% |
| DAIRY | 4 730 | 5 669 | 10.12% |
| CLEANING | 8 409 | 12 341 | 10.98% |
| PRODUCE | 17 405 | 21 373 | 14.08% |

### 5.2 Métriques LSTM — GROCERY I (test : 20%)

| Modèle | MAE | RMSE | MAPE |
|---|---|---|---|
| LSTM v1 (fenêtre) | 44 748 | 60 530 | 62.37% |
| LSTM v2 (N-1) | 44 743 | 60 393 | 62.61% |
| LSTM v3 (hybride) | 45 537 | 64 509 | 59.16% |

### 5.3 Analyse comparative

**Prophet surpasse largement le LSTM** sur ce dataset (8.29% vs 59-62% de MAPE).

Ce résultat s'explique par deux facteurs structurels :
1. **Volume insuffisant** : ~1 700 points par famille — le LSTM nécessite idéalement
   5 000+ points pour converger correctement
2. **Granularité journalière** : le LSTM exprime sa puissance sur des séries à
   granularité fine (horaire, par magasin) où les patterns non-linéaires sont plus riches

**Conclusion :** Prophet est retenu pour le déploiement en production.
Le LSTM serait à revisiter avec des données agrégées par magasin.

---

## 6. Pipeline d'alertes automatiques

### 6.1 Architecture
```
Schedule Trigger (lundi 7h30)
↓
HTTP Request → FastAPI /forecast
↓
Code Node (détection rupture)
↓
Gmail (email HTML hebdomadaire)
```

### 6.2 Logique d'alerte

- **Référence** : moyenne des ventes prévues sur les 30 derniers jours historiques
- **Seuil** : si ventes prévues < 75% de la référence → alerte déclenchée
- **Email** : statut global + tableau des 7 prochains jours + jours à risque identifiés

### 6.3 Fréquence

Déclenchement hebdomadaire le lundi à 7h30 — aligné sur le rythme de planification
des commandes fournisseurs des équipes supply chain.

---

## 7. Stack technique

| Composant | Technologie |
|---|---|
| Langage | Python 3.11 |
| Forecasting | Prophet 1.3.0 (Meta) |
| Deep Learning | TensorFlow 2.21 / Keras |
| Data | pandas, numpy, scikit-learn |
| API | FastAPI + Uvicorn |
| Automatisation | n8n + ngrok |
| Notifications | Gmail (HTML) |
| Environnement | venv Python 3.11 (Windows) |
| Versioning | GitHub |

---

## 8. Limites et perspectives

**Limites identifiées :**
- Modèle entraîné sur une seule famille (GROCERY I) pour l'API — à étendre aux 5
- Données historiques jusqu'en 2017 — modèle à réentraîner sur données récentes
- LSTM sous-performant sur cette granularité — à explorer avec données par magasin

**Perspectives d'amélioration :**
- Intégrer les 5 familles dans l'API avec endpoint par famille
- Ajouter un dashboard Power BI connecté à l'API pour visualisation temps réel
- Explorer XGBoost comme alternative intermédiaire entre Prophet et LSTM
- Mettre en place un réentraînement automatique mensuel du modèle
