# poker-ranges-8max-MTT
Ranges poker 8max MTT : extraction d'un PDF de ranges MTT par pixel-parsing, complété par des positions approximées et documentées (SB, défense BB). Modélisation avec hypothèses explicites, contrôles qualité SQL (CTE, window functions) pour arbitrer bug vs choix stratégique, dashboard Power BI interactif (matrice 13×13).

## Contexte

Ce projet part d'un PDF de ranges d'ouverture préflop en tournoi (8-max, 4 profondeurs
de tapis, 5 positions) et le transforme en un jeu de données exploitable :
extraction automatisée du PDF, complétion des positions manquantes par approximation
documentée, ajout de paramètres contextuels (agressivité de la table, bulle à venir, vitesse
de blindes, structure de payout), puis exploration en Python, contrôles qualité en SQL,
et restitution interactive dans Power BI.

L'objectif n'est pas de produire une stratégie poker parfaite, mais de démontrer un
pipeline de données complet — de la donnée brute non structurée jusqu'à la détection
argumentée d'anomalies.

## Ce que ce projet démontre

- **Extraction de données non structurées** : lecture pixel par pixel d'un PDF (couleurs
  de cellules) pour reconstituer 20 grilles de 169 mains, avec validation croisée
  (0 cellule ambiguë sur 3 380, cohérence de monotonie entre positions)
- **Modélisation de données avec hypothèses explicites** : approximation de positions
  et de paramètres non couverts par la source, documentée plutôt que masquée
- **SQL avancé** (DuckDB) : CTE, fonctions fenêtrées (`OVER PARTITION BY`), agrégations
  conditionnelles, détection d'anomalies
- **Analyse exploratoire Python** : pandas, matplotlib/seaborn, visualisations facettées
- **Esprit critique sur son propre modèle** : arbitrage argumenté entre défaut de
  construction et nuance stratégique légitime, plutôt qu'une confiance aveugle dans
  les chiffres produits
- **Restitution BI** : dashboard Power BI interactif (matrice 13×13 façon range chart,
  slicers dynamiques)

## Pipeline

```
PDF (Pierre Calamusa, ranges 8-max)
   │  extraction pixel par pixel (pdftoppm + classification RGB)
   ▼
CSV structuré (Position × Stack × Contexte × Main → Action)
   │  approximations documentées (SB, défense BB, modificateurs de contexte)
   ▼
Analyse exploratoire (Jupyter / pandas / matplotlib / seaborn)
   │
   ▼
Contrôles qualité et cohérence poker (SQL / DuckDB)
   │
   ▼
Dashboard interactif (Power BI — matrice 13×13, slicers)
```

## Aperçu

PDF : 
<img width="910" height="657" alt="image" src="https://github.com/user-attachments/assets/746a49a4-0349-4e55-92e6-10a361228d24" />

CSV structuré :
<img width="1862" height="405" alt="image" src="https://github.com/user-attachments/assets/0ca6c050-083c-4098-ac9d-405b6b6f5bc8" />

Analyse exploratoire :
<img width="1102" height="758" alt="image" src="https://github.com/user-attachments/assets/9fc68d3b-8fca-44fd-b247-03f2c981bfb2" />

Contrôles qualité et cohérence poker : 
<img width="1105" height="708" alt="image" src="https://github.com/user-attachments/assets/275ce6ed-db61-4a86-8e43-2059033a3f48" />

Power BI : 
<img width="1287" height="727" alt="image" src="https://github.com/user-attachments/assets/c49d6235-f4bc-4efd-82d5-da8e30cd6378" />
<img width="1290" height="727" alt="image" src="https://github.com/user-attachments/assets/7e66e3f5-4b86-45e4-9c08-ca4ad215f5fa" />

## Méthodologie et hypothèses

- **Positions sourcées directement du PDF** : UTG/UTG+1, LJ, HJ, CO, BTN — aux 4
  profondeurs de tapis (15-19bb, 20-26bb, 27-40bb, 40bb+)
- **SB** : approximée par élargissement de la range BTN selon un classement de force
  de main, faute de source directe — ne modélise pas le limp, une composante réelle
  du jeu SB en tournoi
- **Défense BB** (vs EP / HJ-CO / BTN / SB) : approximée par une largeur de call cible
  par bucket d'ouvreur, avec un sous-ensemble de mains en 3-bet
- **Modificateurs de contexte** (agressivité de table, bulle à venir, vitesse de
  structure, format de payout) : multiplicateurs heuristiques appliqués à la largeur
  de range, pas des valeurs issues d'un solveur ou d'un calcul ICM réel
- **Rake** : volontairement absent — non pertinent en tournoi (prélevé sur le buy-in,
  pas sur les pots)
- **Heads-up (2 joueurs)** : explicitement exclu du modèle, dynamique trop différente

## Limites connues

Ce projet documente volontairement ses propres défauts plutôt que de les masquer :

1. **SB approximée trop large** (~47% vs ~38% pour BTN à contexte équivalent) — détecté
   via un contrôle SQL de cohérence, confirme que la calibration initiale était trop
   généreuse et ne tient pas compte du désavantage positionnel post-flop.
2. **Le 3-bet de la BB ne varie pas selon la position de l'ouvreur** — à stack donné,
   le sous-ensemble de mains 3-bet est identique contre un open EP ou un open BTN,
   ce qui n'est pas réaliste (on devrait 3-bet plus large contre une range d'ouverture
   plus large).
3. **Modificateurs de contexte non validés par solveur** — l'effet de la bulle est un
   proxy ICM simplifié (retrait des mains marginales), pas un calcul ICM réel.
4. **442 "trous" de range détectés** (une main forte fold alors qu'une main plus faible
   est jouée dans le même contexte) — un échantillon de 80 a été analysé en détail :
   la majorité s'explique par des nuances stratégiques légitimes (blockers, jouabilité
   post-flop), mais au moins un cas confirme un vrai défaut de construction (SB, 54s,
   20-26bb). Le reste des cas n'a pas encore été examiné.
5. **Incohérences méthodologiques corrigées en cours de route** — certaines analyses
   initiales mélangeaient positions d'ouverture et de défense BB, ou agrégeaient
   plusieurs scénarios de contexte sans filtre neutre, produisant des pourcentages
   incohérents (>100%). Corrigé, mais laissé comme rappel que la QA doit être
   systématique dès la première visualisation.

## Améliorations prévues

- Recalibrer la largeur de base de la SB
- Faire varier la largeur de 3-bet de la BB selon la range de l'ouvreur
- Examiner les ~362 trous de range restants
- Remplacer les modificateurs heuristiques par des données solveur si accès possible
- Modéliser une stratégie de limp pour la SB
- Publier un lien Power BI Service public

## Structure du dépôt

```
poker-ranges-analysis/
README.md
V2_ranges_mtt_8max_enrichi.csv : trop large pour upload github
RangesPierreCalamusa_FR.pdf : trop large pour upload github
Analyse CSV ranges 8max MTT.ipynb
Ranges_MTT_8max.pbix
limites_et_hypotheses.md
```

## Stack technique

Python (pandas, matplotlib, seaborn) · SQL (DuckDB) · Power BI 
