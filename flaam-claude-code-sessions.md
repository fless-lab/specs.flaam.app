# Flaam Backend — Prompts Claude Code (session par session)

> Comment utiliser ce fichier :
> 1. Ouvre Claude Code dans le dossier du projet `flaam-backend/`
> 2. Copie-colle le prompt de la session en cours
> 3. Laisse Claude Code travailler
> 4. Vérifie que tout compile et que les tests passent
> 5. Commit + push
> 6. Passe à la session suivante
>
> Claude Code peut lire tous les fichiers du projet. Il n'a pas de mémoire
> entre les sessions, mais il voit le code existant. Donc chaque prompt
> lui dit "regarde ce qui existe et continue."

---

## SESSION 1 — Squelette du projet

```
Lis le fichier docs/flaam-backend-spec.md sections 1 à 5 (Stack technique, 
structure projet, modèles SQLAlchemy, config, Docker).

Crée le squelette complet du projet :

1. requirements.txt avec toutes les dépendances (FastAPI, uvicorn, SQLAlchemy, 
   asyncpg, alembic, redis, celery, pydantic-settings, python-jose, passlib, 
   structlog, httpx, python-multipart, moshi)

2. .env.example avec toutes les variables de la spec section 2

3. app/core/config.py — Pydantic Settings qui charge le .env

4. app/db/base.py — Base déclarative SQLAlchemy avec UUIDMixin et TimestampMixin

5. app/db/session.py — AsyncSession factory (async_sessionmaker)

6. TOUS les modèles SQLAlchemy dans app/models/ (un fichier par table) :
   User, Profile, Photo, City, Quartier, QuartierProximity, Spot, UserSpot, 
   UserQuartier, Match, Message, Event, EventRegistration, Report, Block, 
   ContactBlacklist, Subscription, NotificationPreference, BehaviorLog, 
   FeedCache, MatchingConfig, AccountHistory, CityLaunchStatus, Payment
   Les modèles sont dans la spec sections 3-4. Copie-les fidèlement.

7. docker-compose.yml (PostgreSQL 16 + Redis 7 + l'app)

8. Dockerfile (Python 3.12, multi-stage)

9. alembic.ini + alembic/env.py configuré pour async SQLAlchemy

10. app/main.py — FastAPI app avec lifespan (startup: connect DB + Redis)

11. Première migration Alembic (alembic revision --autogenerate)

Vérifie que :
- docker compose build réussit
- docker compose up lance les 3 services
- alembic upgrade head crée toutes les tables
- L'app répond sur http://localhost:8000/docs
```

---

## SESSION 2 — Auth (OTP + JWT)

```
Lis docs/flaam-backend-spec.md sections 6 (Auth endpoints) et 12 
(Security hardening). Lis aussi la mise à jour "Auth sans mot de passe" 
en fin de spec.

Regarde le code existant dans app/ pour comprendre la structure.

Implémente le module Auth complet :

1. app/utils/phone.py — normalisation des numéros (+228 90 12 34 56 → +22890123456),
   hashing SHA-256

2. app/utils/sms.py — SMSDeliveryService :
   MVP : implémente UNIQUEMENT le provider Termii (SMS + WhatsApp).
   OTP = SMS primary (fonctionne sur tout numéro, dual SIM safe).
   Fallback WhatsApp après 30s si SMS non reçu (proposé côté client).
   Messages event (rappels, teasers) = WhatsApp primary, SMS fallback.
   Les classes Twilio/Vonage ne sont PAS implémentées maintenant.
   L'interface abstraite (BaseSMSProvider) est codée pour ajouter
   d'autres providers plus tard sans refactor.
   En mode dev : simule l'envoi (log le code au lieu d'envoyer).
   En mode prod : appelle l'API Termii (https://api.ng.termii.com).

3. app/core/security.py :
   - Génération/vérification JWT (python-jose, HS256, access 15min, refresh 7j)
   - Hashing bcrypt pour le PIN MFA
   - Génération OTP 6 chiffres
   - Stockage OTP dans Redis (TTL 10 min)

4. app/core/dependencies.py :
   - get_db() → AsyncSession
   - get_redis() → aioredis client
   - get_current_user() → dépendance FastAPI qui vérifie le JWT

5. app/schemas/auth.py — Pydantic schemas :
   - OtpRequestBody, OtpResponse
   - OtpVerifyBody, AuthTokenResponse
   - RefreshTokenBody
   - AddEmailBody, VerifyEmailBody
   - RecoveryRequestBody, RecoveryCompleteBody
   - MfaPinBody
   - DeleteAccountBody

6. app/services/auth_service.py — logique métier :
   - request_otp(phone) → génère OTP, stocke dans Redis, envoie SMS
   - verify_otp(phone, code) → vérifie, crée le user si nouveau, retourne JWT
   - refresh_token(refresh) → vérifie et retourne nouveau JWT
   - Intégrer la vérification AccountHistory (anti-abus section 30)

7. app/api/v1/auth.py — routes FastAPI :
   - POST /auth/otp/request
   - POST /auth/otp/verify
   - POST /auth/refresh
   - POST /auth/logout
   - DELETE /auth/account
   - POST /auth/email/add
   - POST /auth/email/verify
   - POST /auth/recovery/request
   - POST /auth/recovery/complete
   - POST /auth/mfa/enable
   - POST /auth/mfa/verify
   - POST /auth/mfa/disable
   - POST /auth/phone/change

8. tests/conftest.py — fixtures :
   - db_session : base de test en mémoire ou conteneur
   - client : httpx.AsyncClient
   - auth_headers : JWT valide pour les tests authentifiés
   - test_user : un utilisateur créé pour les tests

9. tests/test_auth.py — minimum :
   - test_request_otp_success
   - test_verify_otp_success
   - test_verify_otp_invalid_code
   - test_verify_otp_expired
   - test_refresh_token_success
   - test_refresh_token_invalid
   - test_protected_route_without_token

Vérifie que :
- Tous les tests passent (pytest -v)
- Le flow complet fonctionne : request OTP → verify → get token → 
  call protected route → success
- Le rate limiting OTP fonctionne (3 max par 10 min)
```

---

## SESSION 3 — Profil + Photos

```
Lis docs/flaam-backend-spec.md section 5 (endpoints Profiles et Photos).

Regarde le code existant, notamment les modèles User, Profile, Photo et 
l'auth qui est déjà implémentée.

Implémente :

1. app/schemas/profiles.py — tous les schemas Pydantic pour le profil
   (MyProfileResponse, UpdateProfileBody, CompletenessResponse, etc.)

2. app/schemas/photos.py — PhotoResponse, ReorderBody

3. app/services/profile_service.py :
   - get_my_profile(user_id)
   - update_profile(user_id, data)
   - calculate_completeness(user_id) — score % basé sur les champs remplis
   - toggle_visibility(user_id, visible)

4. app/services/photo_service.py :
   - upload_photo(user_id, file, position) — sauvegarde le fichier, 
     génère thumbnail (150px) et medium (600px), extrait dominant_color
   - delete_photo(user_id, photo_id) — vérifie qu'il reste min 3
   - reorder_photos(user_id, new_order)
   - Pour la génération de thumbnails, utilise Pillow

5. app/api/v1/profiles.py :
   - GET /profiles/me
   - PUT /profiles/me
   - GET /profiles/{user_id}
   - POST /profiles/me/selfie
   - GET /profiles/me/completeness
   - PATCH /profiles/me/visibility
   - GET /profiles/me/onboarding
   - POST /profiles/me/onboarding/skip

6. app/api/v1/photos.py :
   - POST /photos (multipart upload)
   - DELETE /photos/{photo_id}
   - PATCH /photos/reorder

7. tests/test_profiles.py et tests/test_photos.py

Vérifie que :
- Upload de photo fonctionne (multipart)
- Les thumbnails sont générées
- La suppression vérifie le minimum de 3 photos
- Le score de complétion se calcule correctement
```

---

## SESSION 4 — Quartiers + Spots + Villes

```
Lis docs/flaam-backend-spec.md sections 6 (endpoints Quartiers et Spots)
et la mise à jour Villes/Pays en fin de spec.

Implémente :

1. app/schemas/quartiers.py, app/schemas/spots.py, app/schemas/cities.py

2. app/services/quartier_service.py :
   - get_quartiers(city_id) → liste des quartiers
   - add_quartier_to_profile(user_id, quartier_id, relation_type)
     relation_type = "lives_in" | "hangs_in" | "interested_in"
     Vérifier les limites : 1 lives_in, 5 hangs_in, 3 interested (free) / 6 (premium)
   - remove_quartier(user_id, quartier_id)
   - get_my_quartiers(user_id) → groupés par type
   - get_nearby_quartiers(quartier_id) → graphe de proximité

3. app/services/spot_service.py :
   - search_spots(city_id, category, query)
   - get_spot_detail(spot_id) → avec les profils qui fréquentent
   - add_spot(user_id, spot_id) — vérifier max 5 free / 12 premium
   - remove_spot(user_id, spot_id)
   - check_in(user_id, spot_id) — mettre à jour le niveau de fidélité
   - get_popular_spots(city_id)
   - Niveaux fidélité : 0 check-in = déclaré, 2 = confirmé, 
     4 = régulier, 6 = habitué

4. app/services/city_service.py :
   - get_cities_by_country(country_code) → villes non-hidden avec status
   - get_available_countries() → pays avec au moins 1 ville non-hidden
   - get_launch_status(city_id) → compteur waitlist
   - join_waitlist(user_id, city_id)

5. app/services/waitlist_service.py (MàJ 7) :
   - process_waitlist_join(user_id, city_id, gender) :
     * Femmes → status "activated" immédiatement, skip la file
     * Hommes → status "waiting" avec position dans la file
   - release_batch(city_id) : libère N hommes si ratio femmes > 40%
   - get_waitlist_position(user_id) : position + total en attente

6. app/services/invite_service.py (MàJ 7) :
   - generate_codes(user_id) : 3 codes par femme, 50 par ambassadrice
   - validate_code(code) : vérif existence + expiration + pas utilisé
   - redeem_code(code, user_id) : skip waitlist + crédite l'inviter
   - Modèle InviteCode (code FLAAM-XXXX, creator, used_by, expires_at)

7. Les routers dans app/api/v1/ :
   - quartiers.py (5 endpoints)
   - spots.py (7 endpoints)
   - cities.py (4 endpoints)
   - invites.py (4 endpoints : generate, me, validate, redeem)
   - waitlist.py (géré via cities.py pour le join)

8. Tests pour chaque module

Seed data : crée une commande ou un script qui insère :
- Le Togo (TG) avec Lomé (launch) + Kara (teaser) + Sokodé (teaser)
- La Côte d'Ivoire (CI) avec Abidjan (launch)
- Les quartiers de Lomé (Tokoin, Bè, Djidjolé, Agoè, etc.)
- Quelques spots exemple (Café 21, Salle Olympe, etc.)

Vérifie que :
- Les limites free/premium sont respectées
- Le check-in incrémente le bon niveau de fidélité
- Le filtre par phase fonctionne (hidden pas retourné)
- La waitlist genrée fonctionne : femme → activée, homme → en attente
- Un code valide skip la waitlist (même pour un homme)
- Un code expiré/utilisé retourne une erreur
```

---

## SESSION 5 — Matching engine (5 couches)

```
Lis docs/flaam-backend-spec.md section 6 EN ENTIER (algorithme de matching,
5 couches, corrections, config dynamique). Lis aussi :
- Section 6.3b (L3 lifestyle scorer) en fin de spec
- Section 6.3c (L4 behavior scorer) en fin de spec
- "Mise à jour 5 — Philosophie de l'algo" en toute fin de spec (feedback
  loop implicite > explicite, weight schedule adaptatif par ancienneté,
  anti-gaming, pas de scoring physique)
- "Mise à jour 6 — Préférences implicites + sanitisation des signaux" :
  content-based profiling depuis les signaux implicites, ajustement ±15%
  de L3, plafonnement log 60s, corroboration obligatoire, return_visit 3.0x
C'est la partie la plus critique du projet. Lis tout complètement avant
de coder.

IMPORTANT — Principes non-négociables :
- L'algo ne score JAMAIS l'apparence physique. Pas d'analyse de photos.
- Les signaux implicites (temps sur profil, photos scrollées, retour sur
  un profil) pèsent 2x plus que les signaux explicites (like/skip).
- Les poids entre couches changent avec l'ancienneté du compte :
  * 0-30j : geo=0.55, lifestyle=0.35, behavior=0.10
  * 30-90j : geo=0.40, lifestyle=0.30, behavior=0.30
  * 90j+ : geo=0.30, lifestyle=0.25, behavior=0.45
- Le "82% zones communes" affiché à l'utilisateur n'est que la composante
  géo. Le ranking réel est multi-dimensionnel et jamais exposé.

Implémente le matching engine dans app/services/matching_engine/ :

1. app/services/matching_engine/pipeline.py :
   - generate_feed(user_id) → orchestre les 5 couches dans l'ordre
   - Retourne 8-12 profils triés par score final

2. app/services/matching_engine/hard_filters.py (L1) :
   - Filtres binaires : genre, âge, ville, bloqué, déjà vu, inactif 7j,
     blacklist contacts, intention compatible, même utilisateur
   - Élimine 70-85% des candidats

3. app/services/matching_engine/geo_scorer.py (L2) :
   - 3 passes : exact match quartier, soft match via graphe proximité, 
     interested match
   - Score quartier + score spots (exact × poids fidélité × fraîcheur)
   - Score final géo normalisé 0-1

4. app/services/matching_engine/lifestyle_scorer.py (L3) :
   - Tags TF-IDF (Jaccard pondéré)
   - Matrice compatibilité intentions (4×4)
   - Similitude rythme (lève-tôt/couche-tard)
   - Bonus langues communes
   - Score final lifestyle normalisé 0-1

5. app/services/matching_engine/behavior_scorer.py (L4) :
   - 4 composantes : response quality, selectivity, richness, depth
   - Multiplicateur 0.6 → 1.4
   - Lire depuis Redis "behavior:{user_id}" avec fallback DB

6. app/services/matching_engine/corrections.py (L5) :
   - Wildcards (10-15% de profils hors zone)
   - Boost nouveaux profils (3x les 10 premiers jours)
   - Garantie de visibilité (chaque profil montré min 3 fois)
   - Shuffle déterministe (même seed = même ordre pour le même jour)

7. app/services/matching_engine/implicit_preferences.py (MàJ 6) :
   - compute_implicit_profile(user_id) : agrège les signaux 30j
   - sanitize_time_signal() : cap log 60s + corroboration obligatoire
   - apply_implicit_adjustment() : bonus/malus ±15% sur L3
   - Cache Redis "implicit_prefs:{user_id}" TTL 25h
   - Content-based PUR, PAS de collaborative filtering

8. app/services/matching_engine/first_impression.py (MàJ 7) :
   - apply_first_impression(user, feed_ids) : re-trie les 3 premiers 
     feeds des nouvelles femmes par behavior_multiplier + completeness
   - Filtre strict : profiles masculins avec completeness ≥ 0.75,
     behavior ≥ 1.0, 3+ photos seulement
   - Asymétrique par design (ne s'applique qu'aux nouvelles femmes)

8. Les poids sont lus depuis la table MatchingConfig (avec cache Redis).
   Implémente le cache : Redis → mémoire → DB

8. Les poids sont lus depuis la table MatchingConfig (avec cache Redis).
   Implémente le cache : Redis → mémoire → DB

9. tests/test_matching.py :
   - test_hard_filters_exclude_blocked
   - test_hard_filters_exclude_same_gender (si préf différente)
   - test_geo_scorer_exact_quartier
   - test_geo_scorer_soft_match_nearby
   - test_geo_scorer_spots_in_common
   - test_lifestyle_scorer_tags
   - test_behavior_multiplier_range
   - test_corrections_wildcard_percentage
   - test_corrections_new_user_boost
   - test_full_pipeline_returns_8_to_12
   - test_sanitize_time_caps_at_60s
   - test_sanitize_time_rejects_without_corroboration
   - test_implicit_profile_empty_when_few_signals
   - test_implicit_adjustment_bounded_15_percent

Important : chaque couche doit être testable indépendamment.
Le pipeline les compose mais chaque scorer prend des inputs et 
retourne des scores, c'est tout.

Vérifie que :
- Le pipeline complet retourne 8-12 profils
- Les filtres L1 éliminent correctement
- Les scores sont normalisés entre 0 et 1
- Le boost nouveau profil fonctionne
- sanitize_time_signal(120, corroborated=True) retourne 1.0 (cap)
- sanitize_time_signal(20, corroborated=False) retourne 0.0 (jeté)
- L'ajustement implicite est borné à ±15%
- Le profil implicite est vide si < 5 signaux
```

---

## SESSION 6 — Feed + Matches endpoints

```
Lis docs/flaam-backend-spec.md section 5 (endpoints Feed et Matches).

Le matching engine est déjà implémenté. Maintenant il faut l'exposer 
via l'API et gérer les matches.

Implémente :

1. app/schemas/feed.py — DailyFeedResponse, LikeResponse, SkipBody

2. app/schemas/matches.py — MatchResponse, MatchDetailResponse

3. app/services/feed_service.py :
   - get_daily_feed(user_id) → utilise le pipeline, cache dans FeedCache/Redis
   - like_profile(user_id, profile_id) → vérifie likes restants (5 free, illimité premium),
     vérifie si match mutuel, crée le Match si oui, génère l'ice-breaker
   - skip_profile(user_id, profile_id, reason)
   - log_view(user_id, profile_id, duration_ms)

4. app/services/icebreaker_service.py :
   - generate_icebreaker(user_a, user_b) → basé sur les spots en commun,
     les tags en commun, les prompts. 7 niveaux de priorité (spec section 14).
   - Templates FR/EN

5. app/api/v1/feed.py :
   - GET /feed
   - GET /feed/crossed
   - POST /feed/{profile_id}/like (idempotent via X-Idempotency-Key)
   - POST /feed/{profile_id}/skip
   - POST /feed/{profile_id}/view

6. app/api/v1/matches.py :
   - GET /matches
   - GET /matches/{match_id}
   - DELETE /matches/{match_id} (unmatch)
   - GET /matches/likes-received (premium only)

7. app/tasks/matching_tasks.py :
   - Celery task generate_all_feeds() — batch nocturne
   - Tourne par timezone (spec section 38)

8. Tests

Vérifie que :
- GET /feed retourne 8-12 profils
- POST like avec idempotency key ne duplique pas
- Le match mutuel crée un Match et génère un ice-breaker
- GET /matches/likes-received retourne 403 pour un user free
```

---

## SESSION 7 — Chat (REST + WebSocket)

```
Lis docs/flaam-backend-spec.md section 5 (endpoints Messages) et 
section 37 du mobile spec (protocole WebSocket).

C'est le deuxième module le plus complexe après le matching.

Implémente :

1. app/schemas/messages.py — tous les schemas

2. app/services/chat_service.py :
   - get_messages(match_id, cursor, limit) → pagination cursor-based
   - send_message(match_id, sender_id, content, client_message_id)
     Déduplication par client_message_id (Redis SET NX)
   - send_voice(match_id, sender_id, file) → sauvegarde le fichier audio
   - propose_meetup(match_id, sender_id, spot_id, date, time)
   - respond_meetup(message_id, action: accept/modify/refuse)
   - mark_read(match_id, user_id, last_read_id)

3. app/api/v1/messages.py (REST) :
   - GET /messages/{match_id}
   - POST /messages/{match_id}
   - POST /messages/{match_id}/voice
   - POST /messages/{match_id}/meetup
   - PATCH /messages/{message_id}/meetup
   - PATCH /messages/{match_id}/read

4. app/ws/chat.py (WebSocket) :
   - Connexion : ws://host/ws/chat?token=JWT
   - Vérification JWT à la connexion
   - Enregistrement dans Redis (ws:online:{user_id})
   - Heartbeat ping/pong toutes les 30s
   - Réception de messages et broadcast au destinataire
   - Sync handler : quand un client envoie {"type": "sync", "last_message_id": "..."},
     retourner tous les messages manqués
   - Read receipts : 4 statuts (sent → delivered → read)
   - Déduplication par client_message_id

5. app/services/moderation_service.py :
   - check_message(content) → détection basique de mots interdits,
     liens externes trop tôt, demandes d'argent (spec section 18)
   - 3 niveaux : temps réel → async → humain

6. Tests :
   - test_send_message_success
   - test_send_message_deduplicate
   - test_pagination_cursor
   - test_meetup_proposal_and_accept
   - test_websocket_connect (utiliser httpx-ws ou starlette testclient)

Vérifie que :
- Un message envoyé en REST apparaît pour le destinataire
- La déduplication client_message_id fonctionne
- Le WebSocket se connecte avec un JWT valide
- Le WebSocket refuse un JWT invalide
```

---

## SESSION 8 — Events + Notifications + Subscriptions

```
Lis docs/flaam-backend-spec.md sections correspondantes dans 
docs/flaam-api-endpoints.md : Events (5 endpoints), Notifications (3), 
Subscriptions (5).

Implémente :

1. app/api/v1/events.py :
   - GET /events (filtré par ville et semaine, seulement status=published/full)
   - GET /events/{event_id}
   - POST /events/{event_id}/register (incrémenter, vérifier capacité, passer en "full")
   - DELETE /events/{event_id}/register (décrémenter, repasser en "published" si place)
   - GET /events/{event_id}/matches-preview (score géo+lifestyle avec les inscrits)
   
   Lis la mise à jour "Events : cycle de vie complet" en fin de spec.
   Les events sont créés par l'admin uniquement (POST /admin/events).
   Les catégories : afterwork, sport, brunch, cultural, networking, workshop, outdoor.
   Les status : draft → published → full → ongoing → completed/cancelled.
   
   Celery tasks pour les events :
   - event_reminder : push 2h avant aux inscrits (toutes les 15 min)
   - event_status_updater : draft→ongoing→completed auto (toutes les 15 min)
   - weekly_event_digest : push le dimanche soir "3 events cette semaine"
   - send_post_event_nudge (MàJ 8) : WhatsApp lendemain matin aux ghost users
     (programmé dynamiquement quand event → completed, PAS dans le beat)

   Event-driven onboarding (MàJ 8) :
   - GET /web/events/{slug} : page web publique SSR (Jinja2 ou templates)
   - POST /auth/event-preregister : crée ghost user + génère QR signé HMAC
   - POST /events/{event_id}/checkin : scan QR → checked_in + spot auto-ajouté
   - Ghost user states : ghost → pre_registered → not_started → in_progress → completed
   - Event boost dans pipeline : co-attendees reçoivent ×1.5 pendant 7j (L5)
   - Ice-breakers contextuels : "Vous étiez tous les deux au {event_name}"

2. app/api/v1/notifications.py :
   - GET /notifications/preferences
   - PUT /notifications/preferences
   - POST /notifications/fcm-token

3. app/services/notification_service.py :
   - send_push(user_id, type, data) → Firebase Cloud Messaging
   - Pour le moment, log au lieu d'envoyer (pas de clé FCM en dev)
   - Templates FR/EN (spec section 26)

4. app/api/v1/subscriptions.py :
   - GET /subscriptions/me
   - GET /subscriptions/plans
   - POST /subscriptions/initialize → retourne l'URL Paystack
   - POST /subscriptions/cancel
   - POST /subscriptions/webhook/paystack → vérifier la signature webhook

5. app/services/payment_service.py :
   - State machine : initialized → pending → success/failed/timeout
   - Vérification signature webhook Paystack (HMAC SHA-512)
   - Activation du premium quand le webhook confirme

6. Tests pour chaque module

Vérifie que :
- L'inscription à un event fonctionne
- Le webhook Paystack avec bonne signature active le premium
- Le webhook avec mauvaise signature retourne 401
- event-preregister crée un ghost user (is_active=false, is_visible=false)
- Le check-in QR avec signature invalide retourne 403
- L'OTP d'un ghost user retourne les données pré-remplies (prefilled)
```

---

## SESSION 9 — Safety + Contacts + Behavior + Config

```
Lis docs/flaam-api-endpoints.md sections Safety (6), Contacts (3), 
Behavior (2), Config (2).

Implémente :

1. app/api/v1/safety.py :
   - POST /safety/report
   - POST /safety/block
   - DELETE /safety/block/{user_id}
   - POST /safety/share-date
   - POST /safety/emergency
   - POST /safety/timer/cancel

2. app/api/v1/contacts.py :
   - POST /contacts/blacklist (reçoit des phone_hash, PAS des numéros en clair)
   - GET /contacts/blacklist
   - DELETE /contacts/blacklist/{phone_hash}

3. app/api/v1/behavior.py :
   - POST /behavior/log (batch d'events comportementaux)

4. app/api/v1/config_api.py :
   - GET /config/version
   - GET /config/feature-flags

5. app/services/scam_detection_service.py (spec section 39) :
   - compute_scam_risk(user_id) → score 0-1
   - 6 signaux comportementaux pondérés

6. app/services/abuse_prevention_service.py (spec section 30) :
   - calculate_restrictions(phone_hash) → matrice de restrictions
   - Vérification à chaque inscription

7. app/core/idempotency.py — middleware X-Idempotency-Key (spec section 34)

8. app/core/rate_limiter.py — sliding window Redis (spec section 15)

9. Tests

Vérifie que :
- Block crée l'entrée ET met à jour AccountHistory.blocked_by_hashes
- Le rate limiter retourne 429 avec le header Retry-After
- L'idempotency key retourne le même résultat sur un retry
```

---

## SESSION 10 — Admin + Safety pipeline + Photo moderation + Premium downgrade

```
Lis docs/flaam-api-endpoints.md section Admin (22 endpoints) et 
docs/flaam-backend-spec.md sections 17-29 (RGPD, monitoring, CI/CD, etc.)
Lis aussi :
- Section 16.1b (Modération photo — 3 modes switchables par ENV)
- MàJ 9 (Genre immutable, premium downgrade, pipeline photo 6 checks, 
  targeted likes, reply reminders)
- docs/flaam-safety-anti-fraud.md (document complet anti-fraude)
- docs/flaam-business-model.md (principes éthiques + pricing)
- docs/flaam-ai-scoping.md (hooks IA futurs)

C'est la session la plus large et la plus critique pour la production.
Prends le temps de tout lire.

Implémente :

=== BLOC A — Admin API (22 endpoints) ===

1. app/api/v1/admin.py — 22 endpoints admin :
   - Reports CRUD (list pending, detail, resolve)
   - Ban/unban user
   - Stats dashboard (users actifs, matches/jour, H/F ratio, churn)
   - Events : POST /admin/events (créer), PATCH /admin/events/{id} 
     (publier/annuler), GET /admin/events (liste par status/ville), 
     DELETE /admin/events/{id}
   - Photos modération (voir Bloc B ci-dessous)
   - Spots validation (approve/reject spots créés par users)
   - Matching config : GET/PATCH /admin/matching-config/{key}
   - Feature flags : GET/PATCH pour flag_* keys
   - Trigger batch matching : POST /admin/matching/trigger-batch
   - Ambassadrices : POST/DELETE/GET /admin/ambassadors
   - Waitlist : GET /admin/waitlist/stats, POST /admin/waitlist/release

2. NOUVEAU — PATCH /admin/users/{user_id}/gender :
   Endpoint pour changement de genre par admin (review humaine).
   Le genre n'est PAS modifiable par l'utilisateur via PUT /profiles/me.
   
   Ce que fait cet endpoint :
   a. Change Profile.gender
   b. INVALIDE le selfie (User.is_selfie_verified = False)
   c. RESET le behavior_multiplier (delete Redis behavior:{user_id})
   d. Notifie l'utilisateur ("Reprends un selfie pour vérifier")
   e. Log l'action avec admin_id + raison + old/new gender
   
   Ajoute aussi dans profile_service.update_profile() :
   if "gender" in data:
       raise AppException(400, "gender_not_modifiable")

3. NOUVEAU — GET /admin/prompts/stats :
   Retourne les prompts les plus likés (compteur prompt_like_count 
   dans le JSONB prompts de Profile). Pour le A/B testing des prompts.
   Trié par like_count desc, top 20.

=== BLOC B — Pipeline modération photos (6 checks) ===

4. app/services/photo_moderation_service.py — dispatcher des 4 modes :
   - Mode "manual" : ne fait rien, photo reste pending
   - Mode "onnx" : enqueue le pipeline complet (6 checks)
   - Mode "external" : appel API Sightengine
   - Mode "off" : auto-approve (tests uniquement)
   Appelé systématiquement après upload_photo dans photo_service.

5. app/services/face_verification_service.py — NOUVEAU :
   - Charge ArcFace ONNX (~30 MB) UNE FOIS au démarrage
   - embed_face(image_path) → np.ndarray 128D ou None
   - verify_photo_against_selfie(user_id, photo_path, db) :
     * Cosine similarity selfie ↔ photo
     * >= 0.7 → OK, 0.5-0.7 → warning, 0.3-0.5 → flag, < 0.3 → flag+admin
   - verify_gender_consistency(user_id, db) :
     * Genre détecté sur selfie vs genre déclaré
     * Mismatch avec confidence > 0.85 → flag review humaine
     * JAMAIS auto-reject (personnes trans/NB légitimes)
   - Config ENV : FACE_VERIFICATION_ENABLED (default false)

6. check_photo_authenticity(file_path) dans photo_moderation_service :
   - Pas d'EXIF → flag
   - Pas de modèle camera → flag
   - Software AI (photoshop, midjourney, stable diffusion) → flag
   - Résolution carrée suspecte (512, 1024, 2048) → flag
   - Risk > 0.5 → manual_review
   - Config ENV : EXIF_CHECK_ENABLED (default true)

7. check_photo_temporal_diversity(photos) :
   - 4+ photos même date EXIF → flag
   - Config : toujours actif quand EXIF_CHECK_ENABLED=true

8. Pipeline complet (mode onnx) — 6 checks dans l'ordre :
   a. EXIF authenticity (< 1ms, pur Python)
   b. NSFW detection (~100ms, ONNX)
   c. Face detection (~50ms, YuNet ONNX)
   d. Selfie ↔ photo comparison (~200ms, ArcFace ONNX)
   e. Genre consistency (inclus InsightFace)
   f. Temporal diversity (< 1ms, EXIF dates)
   
   Decision : approved | manual_review | rejected

9. Endpoints admin modération photos :
   - GET /admin/photos/pending?status=pending|manual_review
   - GET /admin/photos/pending/stats (compteurs par statut)
   - PATCH /admin/photos/{photo_id}/moderation (approve/reject)
   - POST /admin/photos/bulk-approve (liste d'IDs)

10. app/tasks/photo_tasks.py — workers modération :
    a. moderate_photo_onnx(photo_id) : pipeline complet 6 checks
    b. moderate_photo_external(photo_id) : API Sightengine + fallback
    
    Modèles ONNX (pas téléchargés en Session 10, juste le code) :
    - ArcFace ResNet50 (~30 MB) pour face embeddings
    - YuNet (~2 MB) pour face detection
    - NSFW Classifier (~50 MB) pour contenu adulte
    
    2 requirements files :
    - requirements.txt (commun)
    - requirements-moderation.txt (onnxruntime, opencv-python-headless)

=== BLOC C — Premium downgrade (gel doux) ===

11. app/services/subscription_service.py — NOUVEAU ou extension :
    
    async def downgrade_user_limits(user, db):
        """Gel doux : les extras passent is_active_in_matching=False."""
        # Quartiers physiques : garder 3 plus anciens actifs
        # Quartiers interested : garder 3 plus anciens actifs
        # Spots : garder 5 plus anciens actifs
        # User.is_premium = False
    
    async def upgrade_user_limits(user, db):
        """Réactivation instantanée."""
        # Tout repasse à is_active_in_matching=True
        # User.is_premium = True

12. Migration Alembic : ajouter is_active_in_matching (Boolean, default True) 
    sur UserQuartier et UserSpot si pas déjà fait.
    Ajouter like_target_type (String(10), nullable), like_target_id 
    (String(100), nullable), like_comment (String(200), nullable) sur Match.

13. Modifier les scorers matching (L1 hard_filters, L2 geo_scorer) pour 
    filtrer uniquement sur is_active_in_matching=True.

=== BLOC D — Targeted likes + Reply reminders ===

14. Modifier POST /feed/{profile_id}/like pour accepter optionnellement :
    target_type ("photo"|"prompt"|"profile", default "profile")
    target_id (uuid ou index du prompt)
    comment (max 200 chars)
    
    Si target_type != "profile" et comment présent :
    → l'ice-breaker auto-généré est REMPLACÉ par le commentaire
    
    Feature flag : flag_targeted_likes_enabled (default 0.0 = désactivé)
    Quand désactivé, les champs sont ignorés silencieusement.
    
    Tracking A/B : quand un like cible un prompt, incrémenter 
    prompt_like_count dans le JSONB prompts du Profile.

15. app/services/reminder_service.py — NOUVEAU :
    - check_pending_replies(db) : conversations avec message non-répondu > 24h
    - Max 1 reminder par match par 48h
    - Respecte notification_preferences
    - Stub Celery (scheduling Session 11)
    
    Feature flag : flag_reply_reminders_enabled (default 1.0 = activé)
    Template : "Tu n'as pas encore répondu à {name}."

=== BLOC E — RGPD + Middleware + CI/CD ===

16. app/services/gdpr_service.py (spec section 17) :
    - initiate_deletion(user_id) → Phase 1 immédiate
    - Celery stubs pour Phase 2 (J+7) et Phase 3 (J+30)

17. app/core/middleware.py :
    - Security headers (X-Content-Type-Options, X-Frame-Options, etc.)
    - Request ID (X-Request-ID)
    - Structured logging (chaque requête loggée avec durée, status, user_id)

18. .github/workflows/backend.yml — CI pipeline :
    - Lint (ruff)
    - Tests (pytest avec PHOTO_MODERATION_MODE=off)
    - Build Docker image
    
=== BLOC F — Tests (cible 230+) ===

Tests admin :
- test_admin_routes_403_for_normal_user
- test_admin_ban_user
- test_admin_create_event
- test_admin_change_gender_invalidates_selfie
- test_admin_prompts_stats

Tests photo moderation :
- test_mode_off_auto_approves
- test_mode_manual_stays_pending
- test_mode_onnx_dispatches (mock onnxruntime)
- test_exif_check_flags_no_exif
- test_exif_check_flags_square_resolution
- test_face_verification_mismatch_flags (mock ArcFace)
- test_gender_mismatch_flags_for_review (PAS auto-reject)
- test_admin_approve_photo
- test_admin_reject_photo
- test_admin_bulk_approve

Tests premium downgrade :
- test_downgrade_gels_extra_quartiers
- test_downgrade_gels_extra_spots
- test_upgrade_reactivates_all
- test_matching_ignores_inactive_quartiers
- test_matching_ignores_inactive_spots

Tests targeted likes :
- test_like_with_target_photo (flag activé)
- test_like_with_target_ignored_when_flag_disabled
- test_like_with_comment_replaces_icebreaker
- test_prompt_like_count_incremented

Tests reply reminders :
- test_reminder_finds_pending_replies
- test_reminder_respects_48h_cooldown
- test_reminder_respects_disabled_pref

Tests RGPD :
- test_gdpr_phase1_anonymizes
- test_gdpr_deleted_user_blocked_from_login

Vérifie que :
- Toutes les routes admin retournent 403 pour un user normal
- Le dispatcher photo switche entre les 4 modes selon ENV
- Le genre est rejeté dans PUT /profiles/me (400 gender_not_modifiable)
- Le changement admin de genre invalide le selfie
- Le downgrade gèle les extras sans supprimer
- Le CI passe au vert
- TOUS les endpoints sont implémentés (vérifier avec docs/flaam-api-endpoints.md)
```

---

## SESSION 11 — i18n centralisée + Catalogue d'erreurs

```
Session 11 — i18n + FlaamError.

Cette session est un REFACTORING TRANSVERSAL. Tu vas parcourir tout le 
codebase pour centraliser les messages utilisateur en FR/EN.

Lis docs/flaam-backend-spec.md section 21 (error handling).

IMPORTANT : cette session ne cree PAS de nouveaux endpoints. Elle 
ameliore ce qui existe.

Implemente :

1. app/core/i18n.py — Systeme de traduction minimal :

   Dictionnaire MESSAGES avec 45+ cles, chacune ayant "fr" et "en".
   Langue par defaut : FR (marche Togo/CI/Senegal).
   
   def t(key: str, lang: str = "fr", **kwargs) -> str:
       """
       Traduit une cle. Fallback FR si langue inconnue.
       t("otp_sent", "fr", channel="SMS") → "Code envoyé par SMS."
       """
   
   def detect_lang(request: Request) -> str:
       """Accept-Language header → 'fr' ou 'en'. Default 'fr'."""
   
   Cles a couvrir (minimum, tu peux en ajouter si tu en vois dans le code) :
   
   # Auth
   otp_sent, otp_invalid, otp_expired, otp_rate_limited,
   account_creation_blocked, token_expired, token_invalid
   
   # Profile
   gender_not_modifiable, profile_incomplete, selfie_required,
   photo_limit_reached, invalid_display_name
   
   # Feed / Matching
   daily_likes_exhausted, premium_required, no_feed_available,
   already_liked, already_skipped, match_not_found
   
   # Chat
   message_blocked_insult, message_blocked_link, message_flagged_money,
   not_match_participant, match_expired
   
   # Safety
   user_blocked, user_unblocked, report_submitted, 
   emergency_timer_started, emergency_timer_cancelled,
   already_blocked, cannot_block_self
   
   # Subscription
   premium_activated, premium_expired, payment_failed,
   payment_confirmed, plan_not_found, already_premium
   
   # Events
   event_full, event_not_found, event_checkin_success,
   event_checkin_invalid_qr, already_registered
   
   # Likes received (messages affichés dans l'app)
   likes_received_free, likes_received_empty
   
   # RGPD
   account_deleted, export_ready, export_rate_limited
   
   # Notifications push (templates)
   notif_new_match, notif_new_message, notif_reply_reminder,
   notif_event_reminder, notif_daily_feed, notif_selfie_required,
   notif_premium_expired, notif_payment_confirmed,
   notif_safety_alert, notif_match_expiring
   
   # Admin
   admin_required, user_not_found, report_not_found

2. app/core/errors.py — FlaamError + handler global :
   
   class FlaamError(Exception):
       def __init__(self, code: str, status_code: int = 400, 
                    lang: str = "fr", **kwargs):
           self.code = code
           self.status_code = status_code
           self.message = t(code, lang, **kwargs)
   
   Exception handler dans main.py :
   @app.exception_handler(FlaamError)
   async def flaam_error_handler(request, exc):
       return JSONResponse(
           status_code=exc.status_code,
           content={"error": exc.code, "message": exc.message}
       )
   
   L'ancien AppException CONTINUE de fonctionner (pas de breaking change).
   Les nouveaux appels utilisent FlaamError, les anciens restent.

3. Refactoring des services PRINCIPAUX — remplacer les strings hardcodes 
   par t(key, lang).
   
   Services a refactorer en priorite :
   
   a. auth_service.py — messages OTP (otp_sent, otp_invalid, otp_expired,
      otp_rate_limited, account_creation_blocked)
   
   b. feed_service.py — messages likes (daily_likes_exhausted, 
      premium_required, already_liked, already_skipped)
   
   c. chat_service.py — messages moderation (message_blocked_insult, 
      message_blocked_link, not_match_participant)
   
   d. safety_service.py — messages safety (user_blocked, report_submitted,
      emergency_timer_started, cannot_block_self)
   
   e. payment_service.py — messages subscription (premium_activated, 
      premium_expired, payment_failed, plan_not_found)
   
   f. event_service.py — messages events (event_full, event_not_found,
      event_checkin_success, event_checkin_invalid_qr)
   
   g. profile_service.py — messages profil (gender_not_modifiable, 
      photo_limit_reached, selfie_required)
   
   h. gdpr_service.py — messages RGPD (account_deleted)
   
   Pattern :
   AVANT : raise AppException(400, "gender_not_modifiable")
   APRES : raise FlaamError("gender_not_modifiable", 400, lang)
   
   Pour obtenir lang dans les services : ajouter lang: str = "fr" comme 
   parametre aux fonctions de service. Les endpoints passent 
   detect_lang(request).

4. Refactoring notification_service.py — utiliser t() pour les templates :
   
   Remplacer les templates hardcodes par :
   title = t(f"notif_{notif_type}", user_lang, **data)
   
   Ajouter quiet hours : pas de push entre 23h et 7h 
   (timezone via City.timezone). Si hors horaire → queue pour 7h le 
   lendemain.
   
   Ajouter deep linking :
   - new_match → flaam://matches/{match_id}
   - new_message → flaam://chat/{match_id}
   - event_reminder → flaam://events/{event_id}
   - daily_feed → flaam://feed

5. Refactoring likes_received (feed_service.get_likes_received) :
   Remplacer les strings message_fr/message_en par :
   "message": t("likes_received_free", lang, count=total)

6. Tests (cible +12) :
   - test_t_returns_french_by_default
   - test_t_returns_english
   - test_t_fallback_french_on_unknown_lang
   - test_t_formats_kwargs
   - test_t_returns_key_if_missing
   - test_detect_lang_french_default
   - test_detect_lang_english
   - test_flaam_error_returns_json_format
   - test_flaam_error_with_french_message
   - test_flaam_error_with_english_message
   - test_feed_like_exhausted_message_french
   - test_auth_otp_invalid_message_bilingual

Verifie que :
- TOUS les services principaux utilisent t() pour les messages user-facing
- FlaamError handler est monte dans main.py
- AppException continue de fonctionner (backward compat)
- Les notifications utilisent t() avec quiet hours
- Aucune regression (tous les tests existants passent toujours)

Cible : 273 actuels + 12 = 285+ tests.

Avant de coder, presente ton plan. Ne commit rien avant mon OK.
```

---

## SESSION 12 — Cache + Celery beat + Cleanup tasks

```
Session 12 — Cache Redis + Celery beat complet + Cleanup tasks.

Cette session met en place l'infrastructure operationnelle : le cache 
pour la performance et le Celery beat pour l'autonomie de l'app.

Lis docs/flaam-backend-spec.md sections 25 (caching), 38 (timezones).

Implemente :

1. app/core/cache.py — Pattern read-through + stampede prevention :
   
   async def cache_get(key, fallback_fn, ttl, redis):
       """
       Read-through : retourne le cache si present, sinon appelle 
       fallback_fn et cache le resultat.
       
       Stampede prevention : si le cache expire et N requetes arrivent,
       une seule regenere (Redis SET NX lock).
       """
       cached = await redis.get(key)
       if cached:
           return json.loads(cached)
       
       lock_key = f"lock:{key}"
       acquired = await redis.set(lock_key, "1", nx=True, ex=30)
       if not acquired:
           await asyncio.sleep(1)
           cached = await redis.get(key)
           if cached:
               return json.loads(cached)
       
       result = await fallback_fn()
       await redis.set(key, json.dumps(result, default=str), ex=ttl)
       await redis.delete(lock_key)
       return result
   
   async def cache_invalidate(key, redis):
       await redis.delete(key)
   
   async def cache_invalidate_pattern(pattern, redis):
       """Invalide toutes les cles qui matchent (ex: feed:{user_id}:*)"""
       keys = []
       async for key in redis.scan_iter(match=pattern):
           keys.append(key)
       if keys:
           await redis.delete(*keys)

2. Cabler le cache dans les services existants :
   
   a. feed_service.get_daily_feed :
      key = f"feed:{user_id}:{date.isoformat()}"
      TTL = 24h
      Invalide au like/skip (cache_invalidate dans like_profile/skip_profile)
   
   b. config_service.get_config :
      key = "matching_config:{key}"
      TTL = 5min
      Deja cache ? Verifie et ameliore si necessaire.
   
   c. Les autres caches (profile, spots, quartiers) sont optionnels 
      pour le MVP. Ajoute juste les commentaires TODO avec les patterns.

3. app/celeryconfig.py — Beat schedule complet :
   
   Verifie quelles taches existent deja en tant que stubs dans 
   app/tasks/ et lesquelles manquent.
   
   Schedule a implementer :
   
   # Feeds
   generate_all_feeds:
     schedule: crontab(hour=3, minute=0)  # 3h UTC = 3h Lome
   
   # Behavior
   persist_behavior_scores:
     schedule: every 1 hour
   
   # Matching
   release_waitlist_batch:
     schedule: every 6 hours
   
   # Events
   event_reminder:
     schedule: every 15 minutes
   event_status_updater:
     schedule: every 15 minutes
   weekly_event_digest:
     schedule: crontab(day_of_week=0, hour=18, minute=0)  # dimanche 18h UTC
   
   # Subscriptions (NOUVEAU)
   check_expired_subscriptions:
     schedule: every 1 hour
     task: detect Subscription.expires_at < now AND is_active=True
           → appeler downgrade_user_limits() + notif push
   
   # Reminders (NOUVEAU)
   send_reply_reminders:
     schedule: every 4 hours
     task: appeler reminder_service.check_pending_replies()
           envoyer notif push, cooldown 48h, respecter prefs
   
   # Safety (NOUVEAU)
   send_emergency_sms:
     schedule: every 1 minute
     task: scan Redis KEYS safety:timer:* 
           pour chaque cle dont le TTL a expire → envoyer SMS Termii
           au contact de confiance stocke dans la valeur de la cle
           ATTENTION : Redis KEYS est O(n), utiliser SCAN en prod.
           Pour le MVP c'est OK car peu de timers actifs simultanement.
   
   # Analytics
   compute_daily_kpis:
     schedule: crontab(hour=0, minute=30)  # 00h30 UTC
   
   # Cleanup
   purge_expired_matches:
     schedule: every 6 hours
   purge_old_behavior_logs:
     schedule: crontab(day_of_week=1, hour=2, minute=0)  # lundi 2h
   purge_old_feed_caches:
     schedule: every 12 hours
   cleanup_account_histories:
     schedule: crontab(day=1, hour=3, minute=0)  # 1er du mois
   
   # Scam
   compute_scam_risk_batch:
     schedule: every 24 hours

   # Event post-teaser (DYNAMIQUE, pas dans beat)
   # Schedule via .apply_async(eta=...) quand event → completed

4. app/tasks/cleanup_tasks.py — Implementer la logique reelle :
   
   Les taches existent en tant que stubs (log only). Implemente :
   
   a. purge_expired_matches :
      Match.last_message_at < now - 7 jours AND status = "matched"
      → status = "expired"
   
   b. purge_old_behavior_logs :
      BehaviorLog.created_at < now - 90 jours → delete
   
   c. purge_old_feed_caches :
      Redis SCAN feed:* → verifier le TTL, supprimer les orphelins

5. app/tasks/subscription_tasks.py — NOUVEAU :
   
   async def check_expired_subscriptions():
       expired = select(Subscription).where(
           Subscription.expires_at < func.now(),
           Subscription.is_active == True
       )
       for sub in expired:
           sub.is_active = False
           user = await db.get(User, sub.user_id)
           await downgrade_user_limits(user, db)
           await notification_service.send_push(
               user.id, "premium_expired", {},
               t("notif_premium_expired", user_lang)
           )

6. app/tasks/reminder_tasks.py — NOUVEAU :
   
   async def send_reply_reminders():
       # Utilise reminder_service.check_pending_replies() de S9
       # Envoie notification push pour chaque
       # Respecte flag + prefs + cooldown 48h

7. app/tasks/emergency_tasks.py — NOUVEAU :
   
   async def send_emergency_sms():
       # Scan Redis safety:timer:* 
       # Pour chaque timer expire :
       #   - Parse la valeur (contact_phone, contact_name, user_name, 
       #     meeting_place)
       #   - Envoie SMS via sms_service.send_text()
       #   - Supprime la cle Redis
       #   - Log l'envoi

8. Tests (cible +10) :
   - test_cache_get_miss_then_hit
   - test_cache_invalidate_clears
   - test_stampede_prevention (2 appels simultanes → 1 seul fallback_fn)
   - test_feed_cache_invalidated_on_like
   - test_check_expired_subscriptions_downgrades
   - test_send_reply_reminders_respects_cooldown
   - test_purge_expired_matches_changes_status
   - test_purge_old_behavior_logs_deletes
   - test_celery_beat_has_all_tasks (verifie 16+ entries)
   - test_emergency_sms_sends_on_expired_timer (mock SMS)

Cible : 285 + 10 = 295+ tests.

Avant de coder, presente ton plan. Ne commit rien avant mon OK.
```

---

## SESSION 13 — Face verification + Analytics + Export RGPD

```
Session 13 — Face verification pipeline ML + Analytics KPIs + 
Export RGPD.

Lis docs/flaam-safety-anti-fraud.md (pipeline photo, modeles ONNX).
Lis docs/flaam-backend-spec.md sections 17 (RGPD export), 29 (KPIs).

Implemente :

1. app/services/face_verification_service.py :
   
   Le dispatcher photo est pret (S10, photo_moderation_service.py).
   Il manque le service ML qui fait les vrais checks.
   
   Charge le modele ONNX UNIQUEMENT si :
   - FACE_VERIFICATION_ENABLED=true
   - Le fichier existe au path FACE_VERIFICATION_MODEL_PATH
   Si fichier absent → log warning + toutes les fonctions retournent 
   {"status": "skip", "reason": "model_not_loaded"}
   
   class FaceVerificationService:
       def __init__(self):
           self.session = None
           if settings.face_verification_enabled:
               model_path = settings.face_verification_model_path
               if Path(model_path).exists():
                   import onnxruntime as ort
                   self.session = ort.InferenceSession(model_path)
                   log.info("face_model_loaded", path=model_path)
               else:
                   log.warning("face_model_not_found", path=model_path)
       
       def embed_face(self, image_path: str) -> np.ndarray | None:
           """Extrait embedding 128D du visage principal."""
           if self.session is None:
               return None
           # Pre-processing : resize 112x112, normalize [-1,1], RGB
           # Inference ONNX
           # Post-processing : L2-normalize
       
       async def verify_photo_against_selfie(self, user_id, photo_path, db):
           """
           Cosine similarity selfie ↔ photo.
           >= 0.7 → match
           0.5-0.7 → warning (log)
           0.3-0.5 → mismatch (flag_for_review)
           < 0.3 → clear_mismatch (flag + notif admin)
           """
       
       async def verify_gender_consistency(self, user_id, db):
           """
           Genre detecte sur selfie vs genre declare.
           Mismatch confidence > 0.85 → flag review humaine.
           JAMAIS auto-reject (personnes trans/NB).
           """
   
   Singleton : instancier UNE FOIS au demarrage dans main.py ou via 
   un module-level instance.

2. check_photo_authenticity(file_path) dans photo_moderation_service :
   
   Verifie si cette fonction est deja implementee (S10 l'a peut-etre 
   stubbee). Si stubbee, implemente la logique reelle :
   
   from PIL import Image
   
   def check_photo_authenticity(file_path: str) -> dict:
       img = Image.open(file_path)
       flags = []
       
       exif = img._getexif() if hasattr(img, '_getexif') else None
       if exif is None:
           flags.append("no_exif_data")
       else:
           if 272 not in exif:  # Camera Model
               flags.append("no_camera_model")
           if 36867 not in exif:  # DateTimeOriginal
               flags.append("no_date")
           software = str(exif.get(305, "")).lower()
           ai_keywords = ["photoshop", "midjourney", "stable diffusion",
                          "dall-e", "comfyui", "gimp", "canva"]
           if any(k in software for k in ai_keywords):
               flags.append("ai_or_editor_software")
       
       w, h = img.size
       if w == h and w in (256, 512, 768, 1024, 2048):
           flags.append("square_suspicious_resolution")
       
       risk = min(1.0, len(flags) * 0.2)
       return {"flags": flags, "risk": risk}

3. check_photo_temporal_diversity(photos) :
   Si stubbee, implemente : extraire EXIF DateTimeOriginal de chaque 
   photo. Si 4+ photos meme jour → flag risk 0.3.

4. Cabler face_verification dans photo_tasks.moderate_photo_onnx :
   Le pipeline 6 checks dans l'ordre :
   a. EXIF authenticity (toujours)
   b. NSFW detection (si modele dispo)
   c. Face detection (si modele dispo)
   d. Selfie ↔ photo comparison (face_verification_service)
   e. Genre consistency (face_verification_service)
   f. Temporal diversity (toujours)

5. requirements-moderation.txt (NOUVEAU) :
   onnxruntime>=1.18.0
   opencv-python-headless>=4.9.0
   numpy>=1.26.0
   
   PAS installe par defaut. Le Dockerfile ne l'installe que si 
   PHOTO_MODERATION_MODE=onnx.

6. Modele DailyKpi + Analytics :
   
   class DailyKpi(Base, UUIDMixin):
       date: Mapped[date]
       city_id: Mapped[uuid.UUID] = mapped_column(FK cities, nullable=True)
       metric: Mapped[str] = mapped_column(String(50))
       value: Mapped[float]
       __table_args__ = (
           UniqueConstraint("date", "city_id", "metric"),
       )
   
   Migration Alembic pour creer la table.
   
   app/tasks/analytics_tasks.py :
   async def compute_daily_kpis(target_date=None, city_id=None):
       Metriques :
       - signups, signups_completed, daily_active
       - likes, skips, matches, messages
       - premium_count, reports, bans
       Upsert dans DailyKpi (idempotent).

7. app/services/export_service.py — Export RGPD :
   
   async def generate_user_export(user_id, db) -> str:
       """Genere un ZIP avec toutes les donnees de l'utilisateur."""
       # profile.json, photos/, messages.json, matches.json, 
       # spots.json, behavior.json
       # Retourne le chemin du fichier ZIP
   
   Endpoint : GET /profiles/me/export
   Rate limit : 1 export par 24h par user.
   Ajouter dans app/api/v1/profiles.py.

8. Tests (cible +12) :
   - test_face_service_skip_when_model_missing
   - test_face_service_embed_returns_128d (mock onnx session)
   - test_face_mismatch_flags_for_review (mock onnx, similarity 0.4)
   - test_gender_mismatch_flags_not_rejects (mock onnx)
   - test_exif_check_flags_no_exif (photo PIL sans EXIF)
   - test_exif_check_flags_square_resolution
   - test_exif_check_passes_normal_photo
   - test_temporal_diversity_flags_same_day
   - test_compute_daily_kpis_creates_rows
   - test_daily_kpi_upsert_idempotent
   - test_export_generates_zip
   - test_export_rate_limited

Cible : 295 + 12 = 307+ tests.

Avant de coder, presente ton plan. Ne commit rien avant mon OK.
```

---

## SESSION 14 — Stabilisation finale + Fix WS + Backup + Audit

```
Session 14 — Derniere session. Stabilisation, fix, backup, audit complet.

C'est la session de FINITION. Apres ca, le backend est pret pour la 
beta privee a Lome.

Implemente :

1. Fix WebSocket flaky test :
   Le test "Event loop is closed" passe en isolation mais echoue en 
   full suite. Cause : le sync_client (Starlette TestClient) et le 
   async client (httpx) partagent le meme event loop.
   
   Approche recommandee :
   - Isoler le sync_client dans une fixture avec son propre event loop
   - OU utiliser anyio.from_thread dans le sync_client pour eviter le 
     conflit de loop
   - OU separer les tests WS dans un pytest-group avec une loop_scope 
     distincte
   
   Verification : pytest full suite (tous les fichiers) → 0 failures.

2. app/core/logging.py — Structlog config :
   Verifie si structlog est deja configure (S10 middleware).
   Si oui, ameliore. Si non, cree.
   
   - Format JSON pour parsing Loki/ELK
   - Processors : add_log_level, TimeStamper(fmt="iso"), 
     StackInfoRenderer, JSONRenderer
   - JAMAIS logger : JWT, passwords, phones, message content
   - Integration avec RequestIDMiddleware (bind request_id au contexte)

3. scripts/backup.sh :
   #!/bin/bash
   set -euo pipefail
   BACKUP_DIR="/backups"
   DATE=$(date +%Y%m%d_%H%M%S)
   DUMP_FILE="$BACKUP_DIR/flaam_$DATE.dump.gz"
   
   pg_dump --format=custom "$DATABASE_URL" | gzip > "$DUMP_FILE"
   
   # Retention locale 7 jours
   find "$BACKUP_DIR" -name "flaam_*.dump.gz" -mtime +7 -delete
   
   # TODO: upload S3/R2 en prod
   # aws s3 cp "$DUMP_FILE" "s3://flaam-backups/$DUMP_FILE"
   
   echo "Backup OK: $DUMP_FILE"
   
   chmod +x scripts/backup.sh

4. VERIFICATION FINALE — Audit complet :
   
   Ouvre docs/flaam-api-endpoints.md et verifie que CHAQUE endpoint 
   liste est implemente dans le code.
   
   Pour chaque endpoint :
   a. La route existe dans app/api/v1/
   b. Le schema request/response existe dans app/schemas/
   c. La logique metier existe dans app/services/
   d. Au moins 1 test existe dans tests/
   
   Verifie aussi ces elements transversaux :
   - app/core/i18n.py existe avec 45+ cles FR/EN
   - app/core/errors.py existe avec FlaamError + handler
   - app/core/cache.py existe avec read-through + stampede
   - app/core/logging.py existe avec structlog JSON
   - app/core/idempotency.py existe avec le middleware
   - app/core/rate_limiter.py existe avec sliding window Redis
   - app/celeryconfig.py a le beat schedule (16+ taches)
   - app/services/notification_service.py utilise t() + quiet hours
   - app/services/face_verification_service.py existe (skip si pas de modele)
   - scripts/backup.sh existe et est executable
   
   Liste-moi les endpoints manquants s'il y en a.
   Liste-moi les sections de la spec (1-40) qui n'ont pas d'implementation.

5. Tests finaux :
   - test_structlog_json_format
   - test_full_suite_no_flaky (run ALL tests → 0 failures)
   
   Cible finale : 310+ tests, 0 failures, 0 flaky.

6. Lancement des commandes de verification :
   - docker compose exec api pytest --tb=short -q (tous les tests)
   - docker compose exec api ruff check app/ (lint)
   - docker compose up --build (build propre)
   - curl http://localhost:8000/docs (Swagger accessible)
   
   Dis-moi le resultat de chaque commande.

7. Commit final :
   Message : "Session 14: stabilization + WS fix + backup + audit (XXX tests)"
   Tag v1.0.0-beta
   Push.

C'est la derniere session. Apres ca, Flaam backend est pret pour la 
beta privee a Lome.
```