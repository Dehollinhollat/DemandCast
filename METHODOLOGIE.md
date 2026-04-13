# 📊 DemandCast — Méthodologie

> **Auteur :** Déhollin HOLLAT, Chef de Projet Data IA  
> **Date :** Avril 2025  
> **En une phrase :** Un système qui prédit les ventes à 30 jours et envoie automatiquement une alerte par email quand un risque de rupture de stock est détecté.

---

## 📌 Sommaire

1. [Pourquoi ce projet ?](#1-pourquoi-ce-projet-)
2. [Les données utilisées](#2-les-données-utilisées)
3. [Quels produits sont analysés ?](#3-quels-produits-sont-analysés-)
4. [Comment le système prédit-il les ventes ?](#4-comment-le-système-prédit-il-les-ventes-)
5. [Résultats](#5-résultats--qui-gagne-)
6. [Pipeline d'alertes automatiques](#6-pipeline-dalertes-automatiques)
7. [Stack technique](#7-stack-technique)
8. [Limites et perspectives](#8-limites-et-perspectives)

---

## 1. Pourquoi ce projet ?

En grande distribution, manquer de stock coûte cher : ventes perdues, clients mécontents, pénalités fournisseurs.

**DemandCast** permet aux équipes logistiques d'anticiper les problèmes avant qu'ils n'arrivent, grâce à l'intelligence artificielle.

---

## 2. Les données utilisées

Source : **Kaggle — Store Sales Time Series Forecasting (Favorita)**

| Paramètre | Valeur |
|---|---|
| Période couverte | Janvier 2013 → Août 2017 |
| Volume | ~3 millions de lignes de ventes |
| Fréquence | 1 ligne par jour, par magasin, par produit |
| Fichiers | Ventes, prix du pétrole, jours fériés, infos magasins |

---

## 3. Quels produits sont analysés ?

Sur 33 familles disponibles, 5 ont été sélectionnées pour leur diversité de profils :

| Famille | Pourquoi ce choix |
|---|---|
| **GROCERY I** (Épicerie) | La plus vendue — référence du projet |
| **BEVERAGES** (Boissons) | Forte variation selon les saisons |
| **PRODUCE** (Fruits & Légumes) | Ventes très irrégulières |
| **CLEANING** (Entretien) | Comportement stable et régulier |
| **DAIRY** (Produits laitiers) | Achat quotidien prévisible |

---

## 4. Comment le système prédit-il les ventes ?

Deux approches ont été testées et comparées.

### ✅ Approche 1 — Prophet (retenue)

Prophet est un outil open-source créé par **Meta (Facebook)**, conçu pour prévoir des séries temporelles business. Il détecte automatiquement :

- Les **tendances** longue durée (croissance ou déclin des ventes)
- La **saisonnalité hebdomadaire** (le dimanche se vend-il plus que le jeudi ?)
- La **saisonnalité annuelle** (décembre est-il un mois fort ?)
- L'**impact des jours fériés** (les ventes chutent-elles le 1er janvier ?)

### ❌ Approche 2 — LSTM (testée, non retenue)

Le **LSTM** (Long Short-Term Memory) est un type de réseau de neurones artificiels qui apprend des séquences dans le temps — un peu comme un humain qui mémoriserait les ventes des 30 derniers jours pour prédire demain.

> ⚠️ Il nécessite idéalement **5 000+ points** par série pour converger. Avec seulement ~1 700 points disponibles par famille, les résultats sont insuffisants.

---

## 5. Résultats — Qui gagne ?

### 📖 Lexique des indicateurs

| Sigle | Nom complet | Ce que ça mesure |
|---|---|---|
| **MAE** | Mean Absolute Error — Erreur Absolue Moyenne | En moyenne, de combien d'unités le système se trompe-t-il par jour ? |
| **RMSE** | Root Mean Square Error — Racine de l'Erreur Quadratique Moyenne | Comme le MAE, mais pénalise davantage les grosses erreurs |
| **MAPE** | Mean Absolute Percentage Error — Erreur Moyenne en Pourcentage | À quel pourcentage près le système prédit-il ? **Plus c'est bas, mieux c'est.** |

> 💡 **Exemple :** Un MAPE de 8% sur des ventes de 100 000 unités = prévision entre 92 000 et 108 000 unités.

### Prophet — Résultats sur les 5 familles (test : 90 jours)

| Famille | MAE | MAPE | Évaluation |
|---|---|---|---|
| GROCERY I | 21 528 unités | 8,3% | ✅ Excellent |
| BEVERAGES | 19 422 unités | 10,2% | ✅ Bon |
| DAIRY | 4 730 unités | 10,1% | ✅ Bon |
| CLEANING | 8 409 unités | 11,0% | ✅ Bon |
| PRODUCE | 17 405 unités | 14,1% | ⚠️ Correct |

### LSTM — Résultats sur GROCERY I (test : 20%)

| Version | MAPE | Évaluation |
|---|---|---|
| v1 — Baseline (fenêtre 30j) | 62,4% | ❌ |
| v2 — Avec données N-1 | 62,6% | ❌ |
| v3 — Hybride (saisonnalité) | 59,2% | ❌ |

**→ Prophet est retenu pour le déploiement en production.**

---

## 6. Pipeline d'alertes automatiques

```
⏰ Chaque lundi à 7h30
        ↓
🌐 Appel HTTP → FastAPI /forecast
        ↓
🔍 Détection rupture (seuil 75% de la référence)
        ↓
📧 Email HTML automatique → équipe supply chain
```

**Logique d'alerte :**
- La **référence** = moyenne des ventes prévues sur les 30 derniers jours historiques
- Si les ventes prévues tombent **sous 75% de cette référence** → alerte déclenchée
- Un email récapitulatif est envoyé avec le statut, les 7 prochains jours et les jours à risque identifiés

---

## 7. Stack technique

| Rôle | Technologie |
|---|---|
| Langage | Python 3.11 |
| Modèle de prévision | Prophet 1.3.0 (Meta) |
| Deep Learning (comparaison) | TensorFlow 2.21 / Keras |
| Traitement des données | pandas, numpy, scikit-learn |
| API | FastAPI + Uvicorn |
| Automatisation | n8n |
| Tunnel Internet | ngrok |
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

---

*DemandCast · Déhollin HOLLAT, Chef de Projet Data IA · Avril 2025*
