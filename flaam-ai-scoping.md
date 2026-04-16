# Flaam — AI Scoping

> Document strategique
> Version : avril 2026
> **Principe fondamental : humain au centre, IA autour.**

---

## Philosophie

Flaam n'est pas une "AI dating app" et ne le sera jamais. Les utilisateurs viennent pour rencontrer des humains, pas pour interagir avec un chatbot. L'IA dans Flaam est **invisible pour l'utilisateur** — elle n'existe que pour :

1. Proteger la communaute (moderation, anti-fraude)
2. Reduire la friction UX (normalisation de donnees, suggestions)
3. Ameliorer la qualite des interactions (ice-breakers contextuels)

**Si le marketing dit un jour "Flaam utilise l'IA", c'est un echec.** Le marketing doit dire "Flaam te connecte avec les bonnes personnes". L'IA rend ca vrai silencieusement.

---

## Principe d'architecture

**Une responsabilite = une fonction de service = un point de bascule.**

Chaque usage futur de l'IA doit pouvoir etre ajoute **a l'interieur** d'une fonction existante, pas par une refonte externe. Le code actuel respecte deja ce principe — chaque usage documente ci-dessous a une fonction hook identifiee.

Regle : **quand l'IA arrive, seule l'implementation interne change. Les signatures publiques et les appelants restent identiques.**

---

## Les 5 usages IA strategiques

### 1. Moderation automatique des messages

**Horizon :** 6-12 mois (Phase 1, des 1 000 users actifs)

**Ce que ca fait :**
- Detecte les demandes d'argent ("envoie-moi 5000 FCFA pour mon credit")
- Detecte les tentatives de sortir hors plateforme trop tot ("WhatsApp : +228...")
- Detecte les insultes, harcelement
- Detecte les patterns de scam ("mon oncle general au Nigeria")

**Tech proposee :**
- API externe : Claude Haiku, GPT-4o-mini, ou Gemini Flash (~$0.001 par message)
- Ou modele local fine-tune : DistilBERT multilingue (~200 MB, tourne sur CPU)
- Preferable : API externe au debut (zero infra), migration locale quand les volumes justifient

**Fonction de bascule :** `app/services/moderation_service.py::check_message(content) -> ModerationResult`

**Structure actuelle (rules) :**
```python
def check_message(content: str) -> ModerationResult:
    if _contains_phone_number(content):
        return ModerationResult(flagged=True, reason="phone_exchange_too_early")
    if _contains_money_keywords(content):
        return ModerationResult(flagged=True, reason="money_request")
    if _contains_offensive_words(content):
        return ModerationResult(flagged=True, reason="insult")
    return ModerationResult(flagged=False)
```

**Structure IA future (meme signature) :**
```python
async def check_message(content: str) -> ModerationResult:
    result = await moderation_llm.classify(content)
    return ModerationResult(
        flagged=result.score > 0.7,
        reason=result.category,
        confidence=result.score,
    )
```

**Donnees necessaires :** les messages eux-memes. Deja capture en table `messages` (Session 7).

**Config ENV prevue :** `MESSAGE_MODERATION_MODE` (`rules` | `llm_api` | `llm_local`)

**Cout estime :**
- 500 users actifs, 50 000 messages/mois : ~$50/mois en API externe
- 5 000 users actifs : ~$500/mois → seuil pour migrer vers local
- Modele local : $0/mois (CPU serveur), investissement initial 1-2 semaines de dev

**Valeur :** enorme. C'est ce qui protege les femmes. Sans ca, tu perds tes utilisatrices en 3 mois.

**Priorite :** HAUTE. Premiere IA a implementer.

---

### 2. Extraction et normalisation des spots

**Horizon :** 12-18 mois (Phase 2, des 5 000 users actifs)

**Ce que ca fait :**
- Normalise les noms : "caffe vingt-et-un" → "Cafe 21"
- Detecte les doublons : "Chez Tonton Be" et "Maquis Tonton" = meme spot
- Suggere la categorie automatiquement (cafe, maquis, gym, etc.)
- Pre-remplit les coordonnees depuis Google Places

**Tech proposee :**
- LLM pour normalisation de texte (prompt court, quelques shots)
- Google Places API pour coordonnees et suggestion de categorie
- Matching fuzzy type Levenshtein pour detection doublons

**Fonction de bascule :** `app/services/spot_service.py::search_or_suggest(query, city_id) -> list[SpotCandidate]`

**Structure actuelle (DB search) :**
```python
def search_or_suggest(query, city_id):
    return db.query(Spot).filter(
        Spot.name.ilike(f"%{query}%"),
        Spot.city_id == city_id
    ).limit(10).all()
```

**Structure IA future :**
```python
async def search_or_suggest(query, city_id):
    # 1. Recherche textuelle classique
    direct_matches = await _db_search(query, city_id)
    if direct_matches and len(direct_matches) <= 3:
        return direct_matches
    
    # 2. Normalisation LLM + nouvelle recherche
    normalized = await llm.normalize_spot_name(query)
    fuzzy_matches = await _db_search(normalized, city_id)
    
    # 3. Fallback Google Places si rien trouve
    if not fuzzy_matches:
        external = await google_places.search(query, city_id)
        return [_convert_to_spot_candidate(e) for e in external]
    
    return fuzzy_matches
```

**Donnees necessaires :** les requetes de recherche des users + la base de spots existante. Deja capture.

**Config ENV prevue :** `SPOT_NORMALIZATION_MODE` (`db_only` | `llm_fallback`)

**Cout estime :**
- 10 000 recherches/mois : ~$5-10 (LLM) + ~$20 (Google Places si usage intensif)
- Negligeable par rapport au gain UX

**Valeur :** evite d'avoir 15 versions de "Cafe 21" dans la DB. Ameliore la decouverte de spots pour les nouveaux.

**Priorite :** MOYENNE. Attendre d'avoir des signaux clairs que c'est un pain point.

---

### 3. Coach de profil

**Horizon :** 12-24 mois (Phase 3, des 10 000 users actifs)

**Ce que ca fait :**

Pour les profils faibles (completeness < 0.6), un assistant suggere des ameliorations :
- "Ton prompt 'Un dimanche ideal' est vide. Voici 3 exemples inspires de tes spots et quartiers."
- "Tes photos sont toutes en selfie-chemise. Ajoute une photo en activite (ton spot Salle Olympe par exemple)."
- "Tu as ecrit '42 ans' mais ta date de naissance dit 28 ans. Erreur de frappe ?"

**Regle absolue :** l'IA **suggere**, l'utilisateur **decide**. L'IA ne redige JAMAIS le prompt a sa place. Si l'IA ecrit, tous les profils se ressemblent, la personnalite disparait.

**Tech proposee :**
- LLM avec le profil en contexte + les spots/tags/quartiers
- Generation de 3 suggestions par category (photo, prompt, bio)
- L'utilisateur choisit une suggestion comme point de depart, puis modifie

**Fonction de bascule :** `app/services/profile_coach_service.py::suggest_improvements(user, profile) -> list[Suggestion]`

**Structure v1 future (rules) :**
```python
def suggest_improvements(user, profile) -> list[Suggestion]:
    suggestions = []
    if profile.completeness < 0.5:
        suggestions.append(Suggestion(
            type="photo_variety",
            message="Ajoute une photo en activite, pas juste en selfie.",
            category="photos",
        ))
    if not profile.prompts:
        suggestions.append(Suggestion(
            type="prompt_suggestion",
            message="Un prompt rempli double tes chances de match.",
            category="prompts",
        ))
    return suggestions
```

**Structure v2 future (LLM) :**
```python
async def suggest_improvements(user, profile) -> list[Suggestion]:
    context = _build_profile_context(user, profile)
    llm_response = await coach_llm.analyze(context)
    return [
        Suggestion(
            type=s.type,
            message=s.text,
            example=s.example,  # exemple concret, pas a copier-coller
            category=s.category,
        )
        for s in llm_response.suggestions
    ]
```

**Endpoint associe :** `GET /profiles/me/suggestions` (pas encore cree, prevu Session 10+)

**Donnees necessaires :** le profil complet + les spots/tags/quartiers. Tout est deja en DB.

**Cout estime :**
- 500 users qui consultent/mois x 3 suggestions : ~$5/mois (LLM)
- Negligeable

**Valeur :** ameliore la qualite moyenne des profils masculins, donc l'experience femmes, donc la retention.

**Priorite :** MOYENNE. Seulement si on constate que les hommes ont tendance a bacler leur profil (on le saura avec la telemetrie).

**Attention** : cette feature doit rester **optionnelle**. Un gros bouton "Suggestions pour ameliorer ton profil" dans les settings. Pas une notification agressive.

---

### 4. Ice-breakers contextuels avances

**Horizon :** 18+ mois (Phase 4, des 20 000 users actifs)

**Ce que ca fait :**

Session 6 code des ice-breakers **par templates** avec 7 niveaux de priorite. Exemple de template niveau 1 : "Vous allez tous les deux au Cafe 21."

Plus tard, l'IA genere des ice-breakers **vraiment personnalises** en combinant le contexte complet :

- Template : "Vous allez tous les deux au Cafe 21."
- IA : "Vous allez tous les deux au Cafe 21, et toi tu as ecrit que tu y commandes toujours la meme chose. Qu'est-ce qu'elle prend, elle ?"

**Regle absolue :** l'ice-breaker est une **amorce** que l'utilisateur prolonge. L'IA ne redige jamais le message entier. Si elle redige tout, le match decouvre lors de la rencontre en vrai que son correspondant ne parle pas comme ca, et la confiance s'effondre.

**Tech proposee :**
- LLM avec contexte complet du match (spots, quartiers, tags, prompts, events en commun)
- Prompt engineering pour generer 3 amorces possibles (l'utilisateur choisit ou tape la sienne)
- Max 60 tokens par amorce

**Fonction de bascule :** `app/services/icebreaker_service.py::generate(user_a, user_b, match) -> IceBreaker`

**Structure actuelle (templates, Session 6) :**
```python
def generate(user_a, user_b, match) -> IceBreaker:
    # 1. Extraction du contexte
    context = MatchContext(
        common_spots=_compute_common_spots(user_a, user_b),
        common_quartiers=_compute_common_quartiers(user_a, user_b),
        common_tags=_compute_common_tags(user_a, user_b),
        event_boost=_check_event_boost(user_a, user_b),
        profile_signals=_extract_profile_signals(user_b),
    )
    
    # 2. Selection du niveau (les 7 de la spec)
    level = _select_priority_level(context)
    
    # 3. Rendu
    return _render_template(level, context, user_a.language)
```

**Structure IA future (seule la fonction _render change) :**
```python
async def generate(user_a, user_b, match) -> IceBreaker:
    context = MatchContext(...)  # identique
    level = _select_priority_level(context)  # identique
    
    # Seul ce dernier appel change
    if settings.icebreaker_mode == "llm":
        return await _render_with_llm(level, context, user_a.language)
    return _render_template(level, context, user_a.language)
```

Cette structure (prevue des Session 6) permet la bascule **sans toucher aux etapes 1 et 2**.

**Donnees necessaires :** le profil des deux users + leur historique de spots/events. Tout est deja en DB.

**Config ENV prevue :** `ICEBREAKER_MODE` (`template` | `llm`)

**Cout estime :**
- 1 000 matches/mois x 3 amorces x ~80 tokens : ~$10-20/mois
- Tres rentable si ca passe de 30% de conversations entamees a 50%

**Valeur :** augmente le taux de conversations entamees apres un match. Mesurable via A/B test.

**Priorite :** BASSE. Les templates couvrent 80% du besoin. L'IA apporte un gain marginal.

---

### 5. Detection de patterns comportementaux suspects

**Horizon :** 12-18 mois (Phase 2, des 5 000 users actifs)

**Ce que ca fait :**

Plus avance que la moderation de messages. L'IA analyse l'**ensemble du comportement** d'un user pour detecter :

- Comptes multiples (meme device fingerprint, meme style d'ecriture, memes photos floues)
- Scammer a distance (profils trop beaux, intention "serious" immediate, spots chic, pas de geo-anchor realistique)
- Harceleur qui change de numero (patterns de messages similaires apres ban)
- Faux profils en groupe (plusieurs comptes crees dans la meme fenetre de 24h, meme IP)

**Tech proposee :**
- Embedding comportemental par user (agrege BehaviorLog sur 30 jours)
- Detecteur d'anomalies : IsolationForest ou Autoencoder sur les embeddings
- Matching de style d'ecriture via embeddings de phrases courtes (sentence-transformers)

**Fonction de bascule :** `app/services/abuse_prevention_service.py::compute_risk_score(user, db) -> float`

Fonction deja existante (Session 2). Rule-based aujourd'hui. Enrichie plus tard.

**Structure actuelle :**
```python
def compute_risk_score(user, db) -> float:
    # Regles basees sur AccountHistory
    # (nombre de comptes crees, frequence, device partage, etc.)
    return rule_based_score
```

**Structure IA future :**
```python
async def compute_risk_score(user, db) -> float:
    rule_score = _rule_based_score(user, db)  # existe deja
    
    # Ajout de features comportementales
    behavior_embedding = await embed_user_behavior(user, db)
    pattern_score = await anomaly_detector.score(behavior_embedding)
    
    # Matching de style d'ecriture avec bans recents
    style_match_score = await style_matcher.check(user, db)
    
    # Combinaison ponderee
    return 0.5 * rule_score + 0.3 * pattern_score + 0.2 * style_match_score
```

**Donnees necessaires :**
- BehaviorLog (deja capture)
- AccountHistory (deja capture)
- Messages (capture en Session 7)
- Photos + signatures (capture au MVP)

**Config ENV prevue :** `ANTIFRAUD_MODE` (`rules` | `rules_plus_ml`)

**Cout estime :**
- Modele local sur serveur : $0/mois (CPU)
- Investissement initial : 2-3 semaines de dev + training sur donnees reelles
- Pre-requis : avoir au moins 50 cas de fraude avere pour entrainer

**Valeur :** critique pour maintenir la confiance a moyen terme. Une app de dating sans anti-fraude solide perd ses users a la premiere vague de scammers.

**Priorite :** HAUTE. Mais apres la moderation de messages (Priorite 1), car cette derniere capture deja 80% des cas.

---

## Les usages IA a ne JAMAIS implementer

### ❌ Chatbot qui parle a la place de l'utilisateur

Certaines apps de dating testent ca (un "dating coach" qui repond a votre place). C'est une catastrophe ethique. Le match decouvre lors de la rencontre en vrai que son correspondant ne parle pas comme ca. Confiance detruite, bad press garantie.

### ❌ Scoring esthetique / "beauty AI"

Discriminatoire, biaise sur des datasets non-representatifs (majorite europeenne, jeune, mince). Deployer ca en Afrique de l'Ouest = filtrer contre les morphologies locales. Inacceptable ethiquement et contre la philosophie Flaam.

### ❌ Matching par embeddings de tous les profils

Tentant mais mauvais. Tu perds l'explicabilite ("82% zones communes" devient "distance cosine 0.87" — incomprehensible pour l'utilisateur). Tu crees des bulles de filtre. Tu perds le controle sur l'algo.

L'algo actuel (5 couches avec regles claires + preferences implicites + event boost) est **meilleur** qu'un modele de recommandation black-box. Ne pas changer pour faire "moderne".

### ❌ Generation de photos IA pour les profils

Flaam est une app qui verifie par selfie live. Ajouter des photos IA, c'est contredire le fondement du produit.

### ❌ "Assistant IA premium" qui suggere des conversations en temps reel

Piege commercial. Les gens paient pour des fonctionnalites qui ameliorent leur experience (plus de likes, voir qui m'a like, etc.), pas pour un bot qui leur tient la main pendant les conversations.

---

## Roadmap IA recommandee

| Phase | Seuil user actif | Feature IA | Priorite |
|-------|------------------|-----------|----------|
| Phase 0 | 0-1 000 | Aucune IA. Moderation manuelle admin. | — |
| Phase 1 | 1 000-5 000 | Moderation messages (API externe) | HAUTE |
| Phase 2 | 5 000-20 000 | Detection patterns suspects + extraction spots | HAUTE / MOYENNE |
| Phase 3 | 20 000-50 000 | Coach de profil | MOYENNE |
| Phase 4 | 50 000+ | Ice-breakers IA (A/B test contre templates) | BASSE |

**Regle d'or :** pas d'IA avant d'avoir **au moins 1 000 users actifs**. Avant, pas assez de donnees pour mesurer l'impact, pas assez de volume pour justifier le cout d'integration.

---

## Ce qui est deja prevu dans le code MVP

Le code actuel respecte le principe "une fonction = un point de bascule" :

- `moderation_service.check_message()` — prevu des Session 7
- `spot_service.search_or_suggest()` — fait en Session 4
- `icebreaker_service.generate()` — a faire en Session 6 avec structure 3-etapes
- `abuse_prevention_service.compute_risk_score()` — fait en Session 2
- `profile_coach_service` — a creer en Session 10 avec regles basiques

**Aucun refactor ne sera necessaire pour ajouter de l'IA.** Il suffira de changer l'implementation interne d'une seule fonction par usage.

---

## Donnees deja capturees pour nourrir les modeles futurs

Tout est deja en DB des le MVP :

| Donnee | Table | Usage IA futur |
|--------|-------|----------------|
| Messages | `messages` | Moderation, detection de patterns |
| Signaux comportementaux | `behavior_logs` | Detection de patterns, coach |
| Historique de comptes | `account_histories` | Detection de multi-comptes |
| Device fingerprints | `devices` | Detection de multi-comptes |
| Recherches de spots | `user_spots` + logs | Normalisation automatique |
| Profils complets | `profiles` + `photos` | Coach |
| Matches et interactions | `matches`, `messages` | Ice-breakers contextuels |

**Seule chose a ajouter au modele Message en Session 7 :** un champ `language_detected` (nullable) pour permettre plus tard des modeles de moderation par langue (francais de Lome ≠ francais de Paris ≠ anglais nigerian).

---

## Budget IA annuel projete

| Phase | Cout mensuel | Cout annuel |
|-------|--------------|-------------|
| Phase 1 (moderation API) | $50 | $600 |
| Phase 2 (moderation + antifraud local) | $100 | $1 200 |
| Phase 3 (+ coach + spots) | $200 | $2 400 |
| Phase 4 (+ icebreakers LLM) | $400 | $4 800 |

**Largement finance par le premium a ces seuils d'utilisateurs actifs.** A 5 000 users avec 5% de conversion premium a 2 500 FCFA/mois, ca fait 625 000 FCFA/mois (~$1 000/mois) de revenus recurrents. Le cout IA represente 5-10% des revenus.

---

## Principes ethiques non-negociables

1. **L'IA est invisible.** L'utilisateur ne sait pas qu'elle existe.
2. **L'IA suggere, l'humain decide.** Jamais l'IA n'agit a la place de l'utilisateur.
3. **L'IA ne juge pas l'apparence.** Aucun scoring esthetique, jamais.
4. **L'IA protege, elle ne surveille pas.** Les donnees servent a la moderation, pas a de la vente ou du profilage tiers.
5. **L'IA est remplacable.** Si un modele est biaise ou obsolete, on peut le desactiver via flag ENV sans casser l'app.
6. **L'IA respecte la langue locale.** Les modeles doivent comprendre le francais de l'Afrique de l'Ouest, pas juste le francais standard.
7. **L'IA est frugale.** Pas de GPT-4 quand un modele plus petit fait le travail. Performance > flashy.

---

## Questions ouvertes a trancher plus tard

1. **Entrainement sur donnees utilisateur ?** Au MVP, non. Peut-etre plus tard avec opt-in explicite + anonymisation. RGPD strict.

2. **Partage de modeles avec d'autres apps africaines ?** Possibilite d'open-sourcer un modele de moderation francais-Afrique fine-tune. Communaute tech + bon pour la marque.

3. **Integration avec des APIs externes (OpenAI, Anthropic, etc.) ou modeles locaux ?** Au debut externe pour la vitesse. A 5 000+ users, migration vers local pour le cout et la souverainete des donnees.

4. **Traitement temps-reel vs batch ?** Moderation messages : temps reel obligatoire. Coach de profil : batch (nuit). Ice-breakers : temps reel au moment du match.

---

## Contact et iterations

Ce document evolue avec le projet. A chaque review strategique (tous les 6 mois apres lancement), revalider :
- La roadmap (phases encore pertinentes ?)
- Les seuils utilisateurs (atteints ?)
- Les couts reels vs projetes
- Les nouvelles opportunites IA qui auraient emerge

**Derniere regle :** si une nouvelle idee IA emerge et qu'elle n'entre pas dans les 5 usages strategiques ou qu'elle viole un des 7 principes ethiques, elle est refusee par defaut. La barre est haute parce que Flaam est une app humaine, pas une vitrine tech.
