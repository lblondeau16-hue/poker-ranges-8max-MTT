# Hypothèses et limites — Ranges poker 8-max MTT

Ce document détaille, poste par poste, ce qui provient directement de la source
originale, ce qui a été approximé, comment, et avec quel degré de confiance. Il
complète le README (qui en donne la version condensée) et sert de référence pour
toute correction ou extension future du projet.

---

## 1. Source primaire

**Document** : PDF de ranges d'ouverture préflop MTT 8-max ("Make Your Ranges Great",
Pierre Calamusa), couvrant 5 positions (UTG/UTG+1, LJ, HJ, CO, BTN) × 4 profondeurs de
tapis (15-19bb, 20-26bb, 27-40bb, 40bb+), soit 20 grilles de 169 mains.

**Méthode d'extraction** : le PDF ne contient pas de texte structuré exploitable
directement — l'information est encodée dans la couleur de fond de chaque cellule
(rouge = all-in, jaune = raise puis fold, vert = raise puis call/4-bet). L'extraction
s'est faite par :
1. Rasterisation de chaque page en image (`pdftoppm`, 120-150 DPI)
2. Détection automatique des bornes de la grille 13×13 par analyse de la palette de
   couleurs (pixels correspondant aux 4 teintes d'action)
3. Échantillonnage du pixel central de chaque cellule, classification par vote de
   couleur dominante sur un petit patch (pas un pixel isolé, pour tolérer l'anti-aliasing
   et le texte superposé)

**Validation** :
- 0 cellule sur 3 380 n'a atteint le seuil de confiance minimal (aucune classification
  ambiguë)
- Contrôle de cohérence croisée : toute main jouée à une position donnée doit l'être
  aussi dans toute position plus tardive à la même profondeur (monotonie de largeur) —
  une seule exception trouvée (54s, jouée au CO mais pas au BTN à 40bb+), probablement
  une vraie particularité de la chart plutôt qu'une erreur de lecture

**Niveau de confiance** : élevé sur les 5 positions et 4 profondeurs couvertes par le
PDF. C'est la seule partie du dataset directement traçable à une source experte.

---

## 2. Positions approximées

### 2.1 Small Blind (SB)

**Non couverte par le PDF.** Construite par élargissement de la range BTN, en utilisant
un classement de force de main pour sélectionner quelles mains ajouter jusqu'à atteindre
une largeur cible (50-56% selon la profondeur).

**Défaut identifié** : cette largeur cible a été fixée trop haut. Un contrôle SQL a
montré que la SB ressort à ~47% de mains jouées contre ~38% pour le BTN à contexte
équivalent — alors qu'en théorie SB devrait être proche ou légèrement *en dessous* de
BTN, car même en ne faisant face qu'à un seul adversaire (BB), SB reste hors position
pour tout le reste de la main, ce qui limite l'intérêt d'élargir davantage que BTN.

**Limite structurelle non corrigée** : au-delà de la largeur, la SB ne modélise pas la
stratégie de **limp**, qui est une composante réelle et significative du jeu SB en
tournoi (limp pur ou stratégie limp/raise mixte). Le fichier ne traite SB que comme
"raise ou fold", ce qui est une simplification qui pourrait affecter la validité des
conclusions tirées sur cette position spécifiquement.

**Niveau de confiance** : faible. À recalibrer en priorité.

### 2.2 Défense Big Blind (BB_vs_EP, BB_vs_HJ_CO, BB_vs_BTN, BB_vs_SB)

**Non couverte par le PDF.** Construite par bucket d'ouvreur (regroupant les positions
d'ouverture en 4 catégories), avec une largeur de call cible croissante selon l'ouvreur
(plus large contre BTN/SB, plus serrée contre EP), et un sous-ensemble de mains classées
en 3-bet (valeur ou all-in selon la profondeur).

**Défaut identifié** : la largeur du sous-ensemble 3-bet est fixée par la profondeur de
tapis uniquement, **indépendamment du bucket d'ouvreur**. Concrètement, le même ensemble
de mains est classé en 3-bet à une profondeur donnée, que l'ouvreur soit EP ou BTN. Ce
n'est pas réaliste : une stratégie de 3-bet cohérente devrait élargir contre un ouvreur
qui ouvre plus large lui-même (moins de risque, plus de value et de bluffs rentables).
Seule la largeur de **call** varie correctement selon l'ouvreur dans le modèle actuel.

**Niveau de confiance** : moyen sur la largeur globale de défense (la progression
EP → HJ_CO → BTN → SB est cohérente avec la théorie), faible sur la composition
call/3-bet à l'intérieur de cette largeur.

---

## 3. Modificateurs de contexte

Quatre paramètres ont été ajoutés sous forme de multiplicateurs heuristiques appliqués
à la largeur de range de base. Aucun n'est issu d'un solveur ou d'un calcul formel —
ce sont des ajustements directionnels raisonnés, pas des valeurs validées empiriquement.

| Paramètre | Modalités | Logique appliquée |
|---|---|---|
| Agressivité de table | Loose / Normale / Tight | Élargit contre une table tight (vol de blindes plus rentable), resserre contre une table loose (moins de fold equity) — effet pondéré par la position (plus fort en position tardive) |
| Bulle à venir | Oui / Non | Retire les mains marginales et les stack-offs les plus fins — un **proxy** d'ajustement ICM, pas un calcul ICM réel |
| Vitesse de structure | Lente / Normale / Rapide | Élargit légèrement en structure rapide (moins de temps pour jouer post-flop, plus de valeur à l'agressivité préflop) |
| Structure de prix | Flat / Exponentiel | Resserre en structure flat (chaque palier de gain compte, priorité au laddering), élargit en structure exponentielle (l'argent est concentré en haut, priorité à l'accumulation de jetons) |

**Point de vigilance** : ces quatre modificateurs se combinent multiplicativement. Un
scénario cumulant plusieurs modalités extrêmes (ex : Tight + Bulle + Rapide) peut donner
une largeur de range qui n'a jamais été validée dans son ensemble, seulement composant
par composant.

**Niveau de confiance** : faible à moyen. Directionnellement défendables, mais aucune
validation quantitative de l'ampleur des effets ni de leurs interactions.

---

## 4. Rake

**Volontairement absent du modèle.** En tournoi, le rake est prélevé une seule fois sur
le buy-in (ex : 100€ + 10€), pas sur chaque pot joué — contrairement au cash game. Il
n'y a donc pas de mécanisme équivalent au rake cash game à intégrer ici. Cette absence
est un choix, pas un oubli.

---

## 5. Heads-up (2 joueurs)

**Explicitement exclu.** Le cadre conceptuel de tout le modèle (position exprimée en
nombre de joueurs restants à parler, largeur croissante à mesure qu'on approche du
bouton) ne s'applique pas au heads-up, où la dynamique est qualitativement différente
(le joueur au bouton/SB ouvre quasiment 100% des mains). Plutôt que de forcer une
approximation qui serait trompeuse, ce format est simplement hors périmètre.

---

## 6. Anomalies de range détectées

Une analyse SQL a recherché les "trous" de range : une main plus forte (RangForce plus
bas) qui fold, alors qu'une main plus faible est jouée dans le même contexte exact
(position, profondeur, et les 4 modificateurs).

- **442 trous détectés** au total, tous scénarios de contexte confondus
- Un échantillon de **80 cas uniques** (Position × Stack × Main) a été analysé en
  détail, avec un verdict basé sur trois signaux : la source de la donnée (PDF vs
  Approximation), l'écart de force entre les deux mains, et la fréquence d'apparition
  sur les 144 combinaisons de contexte

Résultat de cet échantillon :
- **37 cas** : cas limites dépendant du contexte (l'effet cumulé des modificateurs
  fait basculer une main proche du seuil — pas une incohérence de la range de base)
- **33 cas** : nuances stratégiques probables (mains sourcées du PDF, écart de force
  faible, souvent des suited connectors/gappers ou des mains avec blocker As/Roi —
  cohérent avec des choix fins qu'un classement brut ne capture pas)
- **10 cas** examinés individuellement : 1 confirmé comme défaut de construction
  (SB, 54s, 20-26bb — écart de force de 45 rangs, bien trop important pour être une
  nuance), 1 laissé ouvert sans trancher (BTN, 54s, 40bb+), 8 expliqués par une
  logique stratégique cohérente

**Reste à faire** : ~362 trous n'ont pas encore été examinés individuellement. Le
prochain travail de fiabilisation devrait prioriser ceux issus des positions
approximées (SB, défense BB), qui sont statistiquement plus susceptibles d'être de
vrais défauts que ceux issus du PDF.

---

## 7. Incohérences méthodologiques rencontrées et corrigées

Documentées ici comme rappel de vigilance pour la suite du projet, pas comme défauts
du dataset final :

- Plusieurs premières visualisations agrégeaient les 144 combinaisons de contexte sans
  filtre neutre, produisant des pourcentages supérieurs à 100% (incohérents avec la
  borne théorique de 1326 combos)
- Des graphiques mélangeaient positions d'**ouverture** et de **défense BB** dans une
  même comparaison, alors que ce sont deux gestes stratégiques différents (largeurs non
  comparables directement)
- Une agrégation par position seule (sans neutraliser stack/contexte) a un temps
  masqué la vraie largeur moyenne d'une position

Leçon retenue : tout contrôle qualité sur ce dataset doit systématiquement expliciter
sur quel sous-ensemble de contexte il porte (`TypeAction`, et les 4 modificateurs)
avant toute agrégation.

---

## 8. Tableau de synthèse — niveau de confiance par composant

| Composant | Source | Niveau de confiance |
|---|---|---|
| UTG/UTG+1, LJ, HJ, CO, BTN (ouverture) | PDF Calamusa | Élevé |
| SB (ouverture) | Approximation | Faible — largeur à recalibrer, limp non modélisé |
| Défense BB — largeur globale | Approximation | Moyen |
| Défense BB — répartition call/3-bet | Approximation | Faible — ne varie pas selon l'ouvreur |
| Agressivité de table | Heuristique | Faible à moyen |
| Bulle à venir | Heuristique (proxy ICM) | Faible à moyen |
| Vitesse de structure | Heuristique | Faible |
| Structure de prix | Heuristique | Faible |
| Rake | Non modélisé (choix) | N/A |
| Heads-up | Exclu (choix) | N/A |

---

*Ce document est amené à évoluer à mesure que les points listés dans "Améliorations
prévues" (voir README) sont traités.*
