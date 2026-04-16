# Flaam — Algorithme de matching complet

> Document de référence technique
> Version : avril 2026
> 112 endpoints | 5 couches + préférences implicites + event boost

---

## Philosophie fondamentale

L'algorithme de Flaam maximise la probabilité d'une **vraie conversation**, pas l'engagement. Pas de scoring physique. Pas de punition. Pas d'enfermement dans une bulle. Le feedback implicite (temps sur profil, scroll, retours) pèse 2× plus que l'explicite (like/skip). Le "82% zones communes" affiché à l'utilisateur n'est que la composante géo — le vrai ranking est multi-dimensionnel et jamais exposé.

---

## Vue d'ensemble du pipeline

Le batch Celery tourne chaque nuit (par timezone, entre 3h et 5h). Pour chaque utilisateur actif et visible, il exécute le pipeline complet dans l'ordre suivant :

```
Pool actif (~500-2000 profils)
        │
        ▼
   L1 — Filtres durs ──────────────── Élimine 70-85%
        │
        ▼
   L2 — Score géo (0→100) ─────────── Le différenciateur Flaam
        │
        ▼
   L3 — Score lifestyle (0→1.0) ───── Ajusté par les préférences implicites
        │
        ▼
   L4 — Multiplicateur (×0.6→×1.4) ── Qualité des interactions
        │
        ▼
   Combinaison pondérée adaptative ── Poids changent avec l'ancienneté
        │
        ▼
   L5 — Corrections ───────────────── Anti-bulle + boost + visibilité
        │
        ▼
   Event boost (si applicable) ────── +15 pts entre co-participants
        │
        ▼
   First impression (si applicable) ─ Re-tri pour nouvelles femmes
        │
        ▼
   Feed du jour : 8-12 profils
```

Le graphe de proximité des quartiers est chargé une seule fois en mémoire au début du batch (pas recalculé par utilisateur).

---

## L1 — Filtres durs

Une seule requête SQL massive. C'est binaire : tu passes ou tu ne passes pas.

**Les 10 filtres :**

1. **Même ville** — les deux utilisateurs sont dans la même ville active.
2. **Pas moi-même** — exclusion de son propre profil.
3. **Actif et visible** — compte actif, visible, non banni, connecté dans les 7 derniers jours.
4. **Selfie vérifié** — obligatoire pour apparaître dans les feeds.
5. **Genre compatible (bidirectionnel)** — si moi je cherche des femmes ET elle cherche des hommes. Les deux directions doivent matcher.
6. **Âge compatible (bidirectionnel)** — je veux 22-30 ET elle veut 25-35, l'intersection [25-30] doit exister.
7. **Intentions compatibles** — via la matrice 4×4 en version binaire. Serious × friendship = 0.1, donc exclu au niveau filtre.
8. **Pas bloqué (bidirectionnel)** — ni moi ne l'ai bloqué, ni lui ne m'a bloqué.
9. **Pas dans ma blacklist contacts** — vérification par phone_hash SHA-256. Si mon ex est sur l'app, on ne se voit jamais.
10. **Pas déjà vu récemment** — pas skippé dans les 30 derniers jours, pas déjà matché (pending ou actif).

**Résultat :** on passe de 500-2000 à environ 75-300 candidats (15-30% du pool).

---

## L2 — Score géo (0 → 100)

C'est le différenciateur de Flaam. Aucune autre app de dating ne fait du matching par quartiers et spots fréquentés.

Le score a **4 composantes** pondérées :

### Score quartier (3 passes)

**Passe 1 — Exact match.** Quartiers identiques entre les deux utilisateurs. Poids par type de relation :

| Type | Poids |
|------|-------|
| `lives_in` | 2.0× |
| `works_in` | 1.5× |
| `hangs_in` | 1.0× |
| `interested` | 0.8× |

Si on vit tous les deux à Tokoin, c'est le score max.

**Passe 2 — Soft match via proximité.** Quartiers pas identiques mais voisins dans le graphe `QuartierProximity`. Tokoin et Bè ont un `proximity_score = 0.82`, donc si toi tu vis à Tokoin et le candidat à Bè, ça contribue `poids × 0.82`. Seuil minimum : `proximity > 0.40`. En-dessous de 0.65, le score est encore réduit de moitié pour éviter le bruit.

**Passe 3 — Interested match.** Tes quartiers "interested" (ceux cochés dans "D'autres quartiers qui t'intéressent ?") matchent les quartiers physiques du candidat. Résout le cas Agoè ↔ Bè (proximité trop faible pour la passe 2, mais tu as explicitement coché Bè). Invisible sur le profil public. Limité à 3 free / 6 premium.

### Score spots

Nombre de spots en commun, pondéré par catégorie sociale (un maquis en commun vaut plus qu'un coworking). Jaccard pondéré : `Σ social_weight(spot) / max(|mes_spots|, |ses_spots|)`.

### Bonus fidélité

Moyenne géométrique des niveaux de fidélité sur les spots communs. Deux habitués valent plus que deux déclarés.

| Check-ins | Niveau |
|-----------|--------|
| 0 | Déclaré |
| 2 | Confirmé |
| 4 | Régulier |
| 6 | Habitué |

### Fraîcheur

Decay exponentiel sur les check-ins : `score = exp(-0.693 × jours / halflife)`. Un check-in de la semaine dernière pèse beaucoup plus qu'un de il y a 3 mois.

---

## L3 — Score lifestyle (0 → 1.0)

4 composantes :

| Composante | Poids | Méthode |
|------------|-------|---------|
| Tags Jaccard | 50% | `|tags_communs| / |union_tags|` (TF-IDF en V2 pour pondérer par rareté) |
| Matrice intentions | 25% | Matrice de compatibilité 4×4 |
| Rythme de vie | 15% | Même rythme = 1.0, flexible + anything = 0.7, opposés = 0.3 |
| Langues communes | 10% | 1 langue = 0.5, 2+ = 1.0, zéro = 0.0 |

**Matrice d'intentions :**

|  | Serious | Getting to know | Friendship | Open |
|--|---------|-----------------|------------|------|
| **Serious** | 1.0 | 0.5 | 0.1 | 0.7 |
| **Getting to know** | 0.5 | 1.0 | 0.6 | 0.8 |
| **Friendship** | 0.1 | 0.6 | 1.0 | 0.5 |
| **Open** | 0.7 | 0.8 | 0.5 | 1.0 |

### Ajustement par les préférences implicites

Le score L3 est ajusté par un profil de préférences implicites construit à partir des signaux comportementaux des 30 derniers jours (content-based, PAS collaborative filtering).

Si le candidat ressemble aux profils que tu regardes en silence (temps > 8s, photos scrollées, return visits) → bonus. S'il ressemble aux profils que tu rejettes rapidement (skip < 2s, 0 photo) → malus.

Ajustement plafonné à **±15% du score L3**, pondéré par la confiance (0→1, basée sur le nombre de signaux). Moins de 5 signaux = confiance 0 = aucun ajustement.

---

## L4 — Multiplicateur comportemental (×0.6 → ×1.4)

Ce n'est pas un score, c'est un **multiplicateur**. Il récompense les utilisateurs sains et pénalise les spammeurs. Stocké dans Redis (`behavior:{user_id}`), persisté en DB par cron horaire.

4 composantes multipliées entre elles :

| Composante | Mesure | Sweet spot |
|------------|--------|------------|
| Response quality | Taux de réponse aux matchs | Répondre à 50%+ des matchs |
| Selectivity | Ratio likes / profils vus | 20-40% (ni spam ni inactif) |
| Profile richness | Score de complétion | 3+ photos, 1+ prompt, spots tagués |
| Conversation depth | Messages moyens par match | 5+ messages = bon signal |

**Bornes :** le multiplicateur est toujours entre 0.6 et 1.4. Un spammeur qui like tout descend vers 0.6. Un utilisateur exemplaire monte vers 1.4.

---

## Combinaison pondérée adaptative

```
score_final = (geo_w × L2 + lifestyle_w × L3) × L4
```

Les poids ne sont **pas fixes**. Ils varient avec l'ancienneté du compte :

| Ancienneté | Géo (L2) | Lifestyle (L3) | Behavior (L4) |
|------------|----------|----------------|----------------|
| 0-30 jours | 0.55 | 0.35 | 0.10 |
| 30-90 jours | 0.40 | 0.30 | 0.30 |
| 90+ jours | 0.30 | 0.25 | 0.45 |

Un nouveau : la géo domine (on n'a pas de données comportementales). Après 3 mois : le comportement prend le dessus. L'algo s'adapte à toi, pas toi à l'algo.

---

## L5 — Corrections anti-bulle

Après le tri par score, on assemble le feed de 12 profils :

| Slot | Nombre | Sélection |
|------|--------|-----------|
| Top-score | 8 | Les mieux classés par le pipeline |
| Wildcards | 2 | Bon score géo mais lifestyle différent de l'historique de likes |
| Boost nouveaux | 2 | Inscrits < 10 jours, boost dégressif 3×→2×→1.5× |

**Garantie de visibilité :** chaque profil actif est garanti d'apparaître dans le feed d'au moins N personnes par semaine. Pas de profil fantôme.

**Shuffle déterministe :** `SHA-256(user_id + date)` comme seed. Même user, même jour = même ordre. Empêche le refresh-pour-reorder.

---

## Event boost

Les participants d'un même event récent reçoivent un bonus de **+15 points sur le score L2** (géo).

| Période | Boost |
|---------|-------|
| 0-7 jours après l'event | +15 points (complet) |
| 7-14 jours | Decay linéaire vers 0 |
| 14+ jours | 0 |

Conditions : les deux doivent avoir fait le check-in (QR scanné à l'entrée) et complété leur profil.

Un badge "Était au [event]" apparaît dans le feed pendant 14 jours, et l'ice-breaker du match est contextuel : "Vous étiez tous les deux au Café 21. Qu'est-ce que tu en as pensé ?"

---

## First impression feed

Pour les 3 premiers feeds d'une **nouvelle utilisatrice** (femme uniquement), le tri final est remplacé par un tri par qualité masculine :

1. `behavior_multiplier` (les plus sains en premier)
2. `profile_completeness` (les plus complets ensuite)
3. `combined_score` (l'algo normal en dernier recours)

Filtre strict : seuls les profils masculins avec completeness ≥ 0.75, behavior ≥ 1.0, et 3+ photos apparaissent.

Asymétrique par design. C'est le mécanisme qui retient les femmes.

---

## Feedback loop — signaux collectés

### Signaux implicites (envoyés en batch, pèsent 2× plus)

| Signal | Seuil | Poids |
|--------|-------|-------|
| Temps sur profil | > 8 secondes | 2.0× |
| Photos scrollées | ≥ 3 photos | 1.5× |
| Prompt lu (tap "voir plus") | ≥ 1 | 2.0× |
| Return visit (revenu le lendemain) | ≥ 1 | 3.0× |
| Scroll depth | > 80% du profil | 1.5× |

### Signaux explicites

| Signal | Poids |
|--------|-------|
| Like | 1.0× |
| Skip | 1.0× |
| Skip avec raison | 1.5× |
| Quick skip (< 2s, 0 photo) | Signal négatif |

### Sanitisation des signaux (3 protections)

**1. Plafonnement logarithmique.** Cap à 60 secondes. Formule : `signal = log(t/8) / log(60/8)`. Au-delà de 60s, même score qu'à 60s.

**2. Corroboration obligatoire.** Le temps seul ne suffit pas. Il faut au moins 1 signal d'engagement (scroll photo ≥ 1, tap prompt, scroll depth > 30%). Temps sans corroboration = signal jeté (téléphone posé).

**3. Background detection mobile.** `onPause()` coupe le chrono immédiatement. Retour après > 30 secondes = nouveau signal, pas continuation. Empêche le cas "téléphone posé → faux signal 5 minutes".

---

## Anti-gaming

| Protection | Mécanisme |
|------------|-----------|
| Pas de score unique exposé | Le "82% zones communes" n'est que L2 |
| Decay temporel | Inactif = perd en visibilité doucement |
| Poids changeants | Impossible de "calibrer" son comportement |
| Implicite > explicite | Impossible de simuler le temps passé sur un profil |
| Pas de scoring physique | L'algo n'analyse jamais les photos |
| Config dynamique | 67 clés dans MatchingConfig, modifiables via admin API |

---

## Gender balance (intégré dans le matching)

| Stratégie | Description |
|-----------|-------------|
| Feed asymétrique | Les femmes voient les profils triés par behavior_multiplier |
| Quality gate masculin | Pour apparaître dans le feed d'une femme : completeness ≥ 0.70, behavior ≥ 0.9 |
| Smart like weighting | Un like d'un homme sélectif pèse plus dans les notifications |
| Conversation incentive | Bonus +2 profils demain si la femme initie la conversation |
| Monitoring temps réel | Alerte admin si ratio hommes > 75% dans une ville |

---

## Config dynamique

Tous les poids, seuils et paramètres sont dans la table `MatchingConfig` (67 clés). Cachés Redis + mémoire. Modifiables via `PATCH /admin/matching-config/{key}` avec bornes de sécurité et audit trail.

Exemples de clés : `geo_w_quartier`, `geo_w_spot`, `freshness_decay_halflife`, `weight_schedule_geo_0_30`, `new_user_boost_3d`, `wildcard_count`, `first_impression_min_completeness`, `event_boost_points`, `implicit_confidence_threshold`.

On peut tuner l'algo en production sans redéployer.
