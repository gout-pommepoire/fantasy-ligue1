# Fantasy Draft Ligue 1

Outil de draft fantasy inspiré de 38-0, centré sur la Ligue 1 française.  
Le joueur tire des clubs au hasard (toutes saisons confondues), choisit un joueur par tirage, compose une équipe de 11 sur une formation tactique, puis simule une saison complète de Ligue 1.

---

## Structure du projet

```
fantasy_ligue1/
├── fantasy_ligue1.db       Base de données SQLite (source de vérité)
├── generate_html.py        Générateur de l'interface (à relancer après chaque modif DB)
├── draft_ligue1.html       L'outil final — ouvrir dans un navigateur, aucun serveur requis
│
├── migrate_to_multisaison.py   Migration initiale mono→multi-saison (déjà exécuté)
├── import_fc23.py              Import FIFA 23 (saison 2022-23) depuis Kaggle
├── import_fc24.py              Import FC 24  (saison 2023-24) depuis Kaggle
├── create_db.py                Création initiale de la DB + import FC 25 (saison 2024-25)
└── import_kaggle.py            Script générique d'import Kaggle (usage interne)
```

---

## Démarrage rapide

```bash
# 1. Régénérer l'interface après une modif de la DB
python3 generate_html.py

# 2. Ouvrir l'outil
open draft_ligue1.html
```

Aucun serveur, aucune dépendance front. Tout est embarqué dans le HTML.

---

## Base de données

### Schéma multi-saison

Le principe central : **chaque joueur × saison = une carte draftable unique**.  
Dembélé PSG FC 25 et Dembélé PSG FC 24 sont deux cartes différentes avec des stats différentes, liées au même joueur réel via `players.sofifa_id`.

```
players ──────────────── player_versions ──────── player_attributes_v
(identité réelle)         (carte par saison)        (stats détaillées)
sofifa_id (clé SoFIFA)    overall, photo_url…        pac, sho, pas…
                          ├── player_id → players
                          └── season_id → seasons

clubs ────────────────── club_versions ──────── player_club_versions
(club réel)               (club par saison)      (qui joue où)
name, color_primary       color, logo…           player_version_id
                                                 club_version_id
seasons
label: "2024-25"
fifa_version: "FC 25"
```

### Tables principales

| Table | Rôle | Lignes actuelles |
|---|---|---|
| `players` | Identité réelle d'un joueur (sofifa_id) | ~2 900 |
| `seasons` | Catalogue des saisons FIFA | 3 |
| `player_versions` | Snapshot joueur × saison = carte draftable | ~1 670 |
| `player_attributes_v` | 45 stats FIFA par carte | ~1 670 |
| `clubs` | Clubs réels | 27 |
| `club_versions` | Club × saison (couleurs, logo) | 56 |
| `player_club_versions` | Appartenance joueur ↔ club par saison | ~1 670 |
| `positions` | 13 postes (GK, CB, LB…) avec libellé FR | 13 |

### Saisons disponibles

| label | fifa_version | Clubs | Joueurs |
|---|---|---|---|
| 2022-23 | FIFA 23 | 20 | 652 |
| 2023-24 | FC 24 | 18 | 500 |
| 2024-25 | FC 25 | 18 | 517 |

### Requête type — joueurs d'un club une saison donnée

```sql
SELECT pv.id AS draft_id, p.name, pos.label AS poste, pv.overall
FROM player_versions pv
JOIN players p ON pv.player_id = p.id
JOIN player_club_versions pcv ON pcv.player_version_id = pv.id AND pcv.is_main = 1
JOIN club_versions cv ON pcv.club_version_id = cv.id
JOIN clubs c ON cv.club_id = c.id
JOIN seasons s ON pv.season_id = s.id
LEFT JOIN positions pos ON pv.main_position_id = pos.id
WHERE c.name = 'Paris Saint-Germain' AND s.label = '2024-25'
ORDER BY pv.overall DESC;
```

### Requête type — même joueur sur plusieurs saisons

```sql
SELECT pv.id AS draft_id, s.label, c.name AS club, pv.overall
FROM player_versions pv
JOIN players p ON pv.player_id = p.id
JOIN seasons s ON pv.season_id = s.id
JOIN player_club_versions pcv ON pcv.player_version_id = pv.id AND pcv.is_main = 1
JOIN club_versions cv ON pcv.club_version_id = cv.id
JOIN clubs c ON cv.club_id = c.id
WHERE p.sofifa_id = 231443  -- Dembélé
ORDER BY s.year_end;
```

---

## Ajouter une nouvelle saison (ex : FC 26 / 2025-26)

### 1. Télécharger le dataset Kaggle

```python
import kagglehub
path = kagglehub.dataset_download("stefanoleone992/ea-sports-fc-25-complete-player-dataset")
```

### 2. Créer le script d'import (copier import_fc24.py)

Modifier les constantes en tête de fichier :

```python
SEASON_LABEL  = "2025-26"
SEASON_FIFA   = "FC 26"
SEASON_YEAR   = 2026
PHOTO_SUFFIX  = "26_120.png"
CSV_PATH      = "~/.cache/kagglehub/datasets/.../male_players.csv"
```

Filtrer sur `fifa_version == 26` dans la ligne :
```python
ligue1 = df[(df['league_name'].str.contains('Ligue 1', na=False)) &
            (df['fifa_version'] == 26)].copy()
```

Vérifier/compléter `CLUB_NAME_MAP` si de nouveaux clubs sont promus.

### 3. Lancer l'import

```bash
python3 import_fc26.py
```

### 4. Régénérer l'interface

```bash
python3 generate_html.py
```

---

## Mécanique de draft

### Pool et tirage

- Le pool contient **tous les clubs de toutes les saisons** (56 clubs actuellement).
- À chaque lancer de dé, un club est tiré **au hasard** — la saison n'est pas choisie par le joueur.
- Si le club tiré n'a **aucun joueur compatible** avec les postes libres de la formation → reroll automatique gratuit, le club est quand même retiré du pool.
- **11 lancers maximum** pour 11 joueurs.

### Règles de poste

Chaque joueur a un poste principal (`pos`) et des postes alternatifs (`alt`). Un joueur ne peut être placé que dans un slot compatible selon cette table :

| Slot formation | Positions acceptées |
|---|---|
| GK | GK |
| CB | CB |
| LB | LB, LWB |
| RB | RB, RWB |
| CDM | CDM, CM |
| CM | CM, CDM, CAM |
| CAM | CAM, CM, CF |
| LM | LM, LW |
| RM | RM, RW |
| LW | LW, LM |
| RW | RW, RM |
| ST | ST, CF |
| CF / SS | CF, ST, CAM |

### Formations disponibles

| Formation | Slots |
|---|---|
| 4-4-2 | GK · RB CB CB LB · RM CM CM LM · ST ST |
| 4-3-3 | GK · RB CB CB LB · CM CM CM · RW ST LW |
| 4-2-3-1 | GK · RB CB CB LB · CDM CDM · CAM CAM CAM · ST |
| 3-5-2 | GK · CB CB CB · RM CM CDM CM LM · ST ST |
| 3-4-2-1 | GK · CB CB CB · RM CM CM LM · CAM CAM · ST |

La formation est **verrouillée après le premier pick** — elle ne peut plus être changée.

---

## Simulation de saison

### Formule de calcul des ratings d'équipe

Avant chaque match, les stats individuelles des joueurs sont agrégées :

```
GK  = moyenne overall des gardiens
DEF = moyenne overall des défenseurs (CB, LB, RB)
MID = moyenne overall des milieux (CDM, CM, CAM, LM, RM)
FWD = moyenne overall des attaquants (LW, RW, CF, ST)

ATK  = 0.55 × FWD + 0.35 × MID + 0.10 × DEF
DEFR = 0.35 × GK  + 0.45 × DEF + 0.20 × MID
OVR  = (GK + 1.2×DEF + 1.2×MID + FWD) / 4.4
```

Source : formule identique à celle de 38-0 (reverse-engineered).

### Formule de simulation d'un match

```
λ_domicile = max(0.2,  1.3 + (ATK_dom − DEFR_ext) / 10)
λ_extérieur = max(0.2, 1.3 + (ATK_ext − DEFR_dom) / 10)

P_dom = 1 + (random() − 0.5) × 0.30   # facteur aléatoire ±15%
P_ext = 1 + (random() − 0.5) × 0.30

buts_dom = Poisson(max(0.15, λ_dom × P_dom))   # cappé à 9
buts_ext = Poisson(max(0.15, λ_ext × P_ext))
```

Le facteur aléatoire est délibérément plus élevé (×0.30) que l'original de 38-0 (×0.18) pour favoriser les upsets et rendre la simulation plus fun.

### Probabilités d'upset indicatives

| Affiche | Victoire équipe faible |
|---|---|
| PSG (OVR 85) − Clermont (OVR 72) | ~2 % |
| PSG − Monaco (OVR 78) | ~7 % |
| Monaco − Le Havre (OVR 72) | ~16 % |
| Équipe drafted (OVR 76) − PSG | ~4–15 % selon le draft |

### Format saison

- 18 équipes en Ligue 1 2024-25 (17 équipes réelles FC 25 + ton équipe draftée).
- Chaque équipe joue **34 matchs** (aller-retour contre chacune des 17 autres).
- Classement final : points → différence de buts → buts marqués.
- Affichage : classement complet, bilan perso (V/N/D, BP/BC), 34 résultats.

---

## Points d'amélioration identifiés

### Court terme
- [ ] Mise à jour des couleurs clubs manquantes (Angers, Le Havre, Lille OSC) dans `clubs.color_primary`
- [ ] Import FC 26 dès que le dataset Kaggle sera disponible
- [ ] Simulation match par match avec buts en temps réel et commentaires

### Moyen terme
- [ ] Avantage domicile (actuellement non modélisé — coefficient +3 sur ATK_dom à tester)
- [ ] Sauvegarder / partager son équipe draftée (URL encodée ou localStorage)
- [ ] Mode multijoueur : plusieurs joueurs draftent chacun une équipe, saison partagée

### Long terme
- [ ] Données réelles de matchs Ligue 1 pour calibrer les formules
- [ ] Coupes (Coupe de France, Ligue des Champions draft)
- [ ] Historique des drafts avec stats personnelles

---

## Sources de données

| Saison | Dataset Kaggle | Filtre appliqué |
|---|---|---|
| 2022-23 | `stefanoleone992/fifa-23-complete-player-dataset` | `league_name LIKE 'Ligue 1%' AND fifa_version = 23` |
| 2023-24 | `stefanoleone992/ea-sports-fc-24-complete-player-dataset` | `league_name LIKE 'Ligue 1%' AND fifa_version = 24` |
| 2024-25 | Données initiales `create_db.py` (FC 25) | Ligue 1 uniquement |

Les datasets `stefanoleone992` contiennent des données cumulatives depuis FIFA 15 — le filtre `fifa_version == N` est **obligatoire** pour n'extraire que la saison souhaitée.

Les `player_id` dans ces datasets correspondent aux `sofifa_id` dans notre base.

Les photos joueurs sont construites depuis le `sofifa_id` :
```python
s = str(sofifa_id).zfill(6)
photo_url = f"https://cdn.sofifa.net/players/{s[:3]}/{s[3:]}/25_120.png"
```
