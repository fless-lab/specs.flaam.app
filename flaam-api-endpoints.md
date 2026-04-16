# Flaam — Catalogue complet des endpoints API

> 111 endpoints au total
> Groupés par domaine fonctionnel
> Auth = pas de token requis | Oui = JWT requis | Premium = abonnement requis | Admin = rôle admin

---

## 1. Auth (10 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 1 | POST | `/auth/otp/request` | Demander un OTP (SMS ou WhatsApp) | Non |
| 2 | POST | `/auth/otp/verify` | Vérifier l'OTP, retourne JWT | Non |
| 3 | POST | `/auth/refresh` | Refresh le JWT avec le refresh token | Refresh |
| 4 | POST | `/auth/logout` | Invalider le refresh token | Oui |
| 5 | DELETE | `/auth/account` | Supprimer le compte (soft delete, 3 phases) | Oui |
| 6 | POST | `/auth/email/add` | Ajouter un email de récupération | Oui |
| 7 | POST | `/auth/email/verify` | Vérifier l'email (via token envoyé par mail) | Non |
| 8 | POST | `/auth/recovery/request` | Demander un lien de récupération par email | Non |
| 9 | POST | `/auth/recovery/complete` | Finaliser la récupération (nouveau numéro + OTP) | Non |
| 10 | POST | `/auth/phone/change` | Changer de numéro (vérif ancien + nouveau OTP) | Oui |

---

## 2. MFA (3 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 11 | POST | `/auth/mfa/enable` | Activer le PIN 6 chiffres | Oui |
| 12 | POST | `/auth/mfa/verify` | Vérifier le PIN après l'OTP (si MFA activé) | OTP OK |
| 13 | POST | `/auth/mfa/disable` | Désactiver le MFA (requiert le PIN actuel) | Oui |

---

## 3. Profil (6 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 14 | GET | `/profiles/me` | Mon profil complet | Oui |
| 15 | PUT | `/profiles/me` | Mettre à jour mon profil | Oui |
| 16 | GET | `/profiles/{user_id}` | Voir le profil d'un autre (si autorisé par match) | Oui |
| 17 | POST | `/profiles/me/selfie` | Upload selfie de vérification (liveness) | Oui |
| 18 | GET | `/profiles/me/completeness` | Score de complétion du profil (%) | Oui |
| 19 | PATCH | `/profiles/me/visibility` | Toggle mode pause (visible/invisible) | Oui |

---

## 4. Onboarding (2 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 20 | GET | `/profiles/me/onboarding` | État actuel de l'onboarding (étape + données) | Oui |
| 21 | POST | `/profiles/me/onboarding/skip` | Passer une étape skippable | Oui |

---

## 5. Photos (3 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 22 | POST | `/photos` | Upload une photo (multipart, max 6) | Oui |
| 23 | DELETE | `/photos/{photo_id}` | Supprimer une photo (min 3 obligatoires) | Oui |
| 24 | PATCH | `/photos/reorder` | Réordonner les photos (positions) | Oui |

---

## 6. Quartiers (5 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 25 | GET | `/quartiers?city_id={id}` | Liste des quartiers d'une ville | Oui |
| 26 | POST | `/quartiers/me` | Ajouter un quartier (lives_in/hangs_in/interested_in) | Oui |
| 27 | DELETE | `/quartiers/me/{quartier_id}` | Retirer un quartier | Oui |
| 28 | GET | `/quartiers/me` | Mes quartiers par type | Oui |
| 29 | GET | `/quartiers/{id}/nearby` | Quartiers voisins (graphe de proximité) | Oui |

---

## 7. Spots (7 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 30 | GET | `/spots?city_id={id}&category={cat}&q={search}` | Chercher des spots | Oui |
| 31 | GET | `/spots/{spot_id}` | Détail d'un spot (profils qui fréquentent) | Oui |
| 32 | POST | `/spots/me` | Ajouter un spot à mon profil (max 5 free / 12 premium) | Oui |
| 33 | DELETE | `/spots/me/{spot_id}` | Retirer un spot | Oui |
| 34 | PATCH | `/spots/me/{spot_id}/visibility` | Masquer/afficher un spot sur le profil | Oui |
| 35 | POST | `/spots/me/{spot_id}/checkin` | Check-in à un spot | Oui |
| 36 | GET | `/spots/popular?city_id={id}` | Spots populaires dans une ville | Oui |

---

## 8. Feed & Matching (5 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 37 | GET | `/feed` | Feed du jour (8-12 profils curated) | Oui |
| 38 | GET | `/feed/crossed` | Section "Déjà croisés" (spots en commun) | Oui |
| 39 | POST | `/feed/{profile_id}/like` | Liker un profil (idempotent) | Oui |
| 40 | POST | `/feed/{profile_id}/skip` | Passer un profil (raison optionnelle) | Oui |
| 41 | POST | `/feed/{profile_id}/view` | Logger un temps de vue (signal implicite) | Oui |

---

## 9. Matches (4 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 42 | GET | `/matches` | Mes matchs actifs (triés par dernière activité) | Oui |
| 43 | GET | `/matches/{match_id}` | Détail d'un match (profil + ice-breaker) | Oui |
| 44 | DELETE | `/matches/{match_id}` | Unmatch (irréversible) | Oui |
| 45 | GET | `/matches/likes-received` | Qui m'a liké (liste) | Premium |

---

## 10. Messages (6 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 46 | GET | `/messages/{match_id}?cursor={id}&limit=50` | Historique messages (paginé) | Oui |
| 47 | POST | `/messages/{match_id}` | Envoyer un message texte | Oui |
| 48 | POST | `/messages/{match_id}/voice` | Envoyer un message vocal (multipart) | Oui |
| 49 | POST | `/messages/{match_id}/meetup` | Proposer un date (lieu + créneau) | Oui |
| 50 | PATCH | `/messages/{message_id}/meetup` | Répondre à un meetup (accept/modify/refuse) | Oui |
| 51 | PATCH | `/messages/{match_id}/read` | Marquer les messages comme lus | Oui |

---

## 11. WebSocket (1 endpoint)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 52 | WS | `/ws/chat?token={jwt}` | Connexion WebSocket chat temps réel | JWT |

> Events WS entrants : message, ack, delivered, read, sync_response, pong
> Events WS sortants : message, read, sync, ping

---

## 12. Events (5 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 53 | GET | `/events?city_id={id}&from={date}&to={date}` | Events à venir (filtrés) | Oui |
| 54 | GET | `/events/{event_id}` | Détail d'un event (inscrits, matchs preview) | Oui |
| 55 | POST | `/events/{event_id}/register` | S'inscrire à un event | Oui |
| 56 | DELETE | `/events/{event_id}/register` | Se désinscrire | Oui |
| 57 | GET | `/events/{event_id}/matches-preview` | Nombre de matchs potentiels à l'event | Oui |

---

## 13. Notifications (3 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 58 | GET | `/notifications/preferences` | Mes préférences de notifications | Oui |
| 59 | PUT | `/notifications/preferences` | Mettre à jour les préférences | Oui |
| 60 | POST | `/notifications/fcm-token` | Enregistrer/mettre à jour le token FCM | Oui |

---

## 14. Subscriptions & Paiement (5 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 61 | GET | `/subscriptions/me` | Mon abonnement actuel + paiement pending | Oui |
| 62 | GET | `/subscriptions/plans` | Plans disponibles (prix par ville/devise) | Oui |
| 63 | POST | `/subscriptions/initialize` | Initier un paiement (retourne URL Paystack) | Oui |
| 64 | POST | `/subscriptions/cancel` | Annuler le renouvellement auto | Oui |
| 65 | POST | `/subscriptions/webhook/paystack` | Webhook Paystack (charge.success/failed) | Signature |

---

## 15. Safety & Modération (6 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 66 | POST | `/safety/report` | Signaler un utilisateur (5 raisons) | Oui |
| 67 | POST | `/safety/block` | Bloquer un utilisateur | Oui |
| 68 | DELETE | `/safety/block/{user_id}` | Débloquer | Oui |
| 69 | POST | `/safety/share-date` | Partager un date avec le contact de confiance | Oui |
| 70 | POST | `/safety/emergency` | Bouton d'urgence (envoie GPS + alerte) | Oui |
| 71 | POST | `/safety/timer/cancel` | Désactiver le timer de sécurité | Oui |

---

## 16. Contacts masqués (3 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 72 | POST | `/contacts/blacklist` | Importer contacts à masquer (hashes SHA-256) | Oui |
| 73 | GET | `/contacts/blacklist` | Liste des contacts masqués (hashes) | Oui |
| 74 | DELETE | `/contacts/blacklist/{phone_hash}` | Retirer un contact de la blacklist | Oui |

---

## 17. Villes & Pays (4 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 75 | GET | `/cities?country={code}` | Villes d'un pays (actives + teaser, pas hidden) | Oui |
| 76 | GET | `/cities/countries` | Pays disponibles (qui ont ≥1 ville non-hidden) | Oui |
| 77 | GET | `/cities/{city_id}/launch-status` | Status waitlist (compteur, position, invites) | Oui |
| 78 | POST | `/cities/{city_id}/waitlist/join` | S'inscrire en waitlist pour une ville teaser | Oui |

---

## 18. Behavior & Analytics (2 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 79 | POST | `/behavior/log` | Envoyer des signaux comportementaux en batch | Oui |
| 80 | GET | `/profiles/me/export` | Exporter toutes mes données (RGPD Art. 20) | Oui |

---

## 19. Config (2 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 81 | GET | `/config/version` | Version min/recommandée de l'app | Non |
| 82 | GET | `/config/feature-flags` | Feature flags actives pour cet utilisateur | Oui |

---

## 20. Admin (17 endpoints)

| # | Méthode | Endpoint | Description | Auth |
|---|---------|----------|-------------|------|
| 83 | GET | `/admin/reports?status=pending` | Reports en attente | Admin |
| 84 | PATCH | `/admin/reports/{report_id}` | Traiter un report (action + note) | Admin |
| 85 | POST | `/admin/users/{user_id}/ban` | Bannir (raison obligatoire) | Admin |
| 86 | POST | `/admin/users/{user_id}/unban` | Débannir | Admin |
| 87 | GET | `/admin/stats` | Dashboard statistiques globales + par ville | Admin |
| 88 | POST | `/admin/events` | Créer un event (status=draft) | Admin |
| 89 | PATCH | `/admin/events/{event_id}` | Modifier/publier/annuler un event | Admin |
| 90 | GET | `/admin/events?city_id={id}&status={s}` | Liste events par status | Admin |
| 91 | DELETE | `/admin/events/{event_id}` | Supprimer un event (draft/cancelled only) | Admin |
| 92 | PATCH | `/admin/spots/{spot_id}/verify` | Valider/rejeter un spot proposé | Admin |
| 93 | GET | `/admin/photos/review` | Photos en attente de modération | Admin |
| 94 | PATCH | `/admin/photos/{photo_id}/moderate` | Approuver/rejeter une photo | Admin |
| 95 | GET | `/admin/matching-config` | Tous les poids du matching (groupés) | Admin |
| 96 | PATCH | `/admin/matching-config/{key}` | Modifier un poids | Admin |
| 97 | POST | `/admin/matching/trigger-batch` | Relancer le batch matching manuellement | Admin |
| 98 | GET | `/admin/feature-flags` | Liste des feature flags | Admin |
| 99 | PATCH | `/admin/feature-flags/{key}` | Modifier une flag (rollout %, villes) | Admin |

---

## Résumé par domaine

| Domaine | Endpoints | Auth requise |
|---------|-----------|-------------|
| Auth & MFA | 13 | Mixte |
| Profil & Onboarding | 8 | Oui |
| Photos | 3 | Oui |
| Quartiers | 5 | Oui |
| Spots | 7 | Oui |
| Feed & Matching | 5 | Oui |
| Matches | 4 | Oui/Premium |
| Messages | 6 | Oui |
| WebSocket | 1 | JWT |
| Events | 5 | Oui |
| Notifications | 3 | Oui |
| Subscriptions | 5 | Oui/Signature |
| Safety & Modération | 6 | Oui |
| Contacts masqués | 3 | Oui |
| Villes & Pays | 4 | Oui |
| Behavior & Data export | 2 | Oui |
| Config | 2 | Mixte |
| Admin | 17 | Admin |
| Invitations | 4 | Mixte |
| Admin — Ambassadrices & Waitlist | 5 | Admin |
| **TOTAL** | **108** | — |

---

## 21. Invitations (4 endpoints)

| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 100 | POST | `/invites/generate` | Générer mes codes d'invitation (3 par femme, 50 par ambassadrice) | Oui |
| 101 | GET | `/invites/me` | Liste de mes codes : actifs, utilisés, expirés | Oui |
| 102 | POST | `/invites/validate` | Vérifier un code pendant l'onboarding (avant auth) | Non |
| 103 | POST | `/invites/redeem` | Utiliser un code → skip waitlist + créditer l'inviter | Oui |

## 22. Admin — Ambassadrices & Waitlist (5 endpoints)

| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 104 | POST | `/admin/ambassadors` | Nommer une ambassadrice (active premium lifetime + 50 codes) | Admin |
| 105 | DELETE | `/admin/ambassadors/{user_id}` | Retirer le statut ambassadrice | Admin |
| 106 | GET | `/admin/ambassadors` | Liste ambassadrices + stats (codes utilisés, invités actifs) | Admin |
| 107 | GET | `/admin/waitlist/stats` | Stats waitlist par ville et par genre | Admin |
| 108 | POST | `/admin/waitlist/release` | Forcer la libération manuelle d'un batch | Admin |

---

> **Total mis à jour : 108 endpoints** (99 originaux + 4 invitations + 5 admin ambassadrices/waitlist)

## 23. Event web + ghost users (3 endpoints)

| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 109 | GET | `/web/events/{slug}` | Page web publique de l'event (SSR) | Non |
| 110 | POST | `/auth/event-preregister` | Pré-inscription event (OTP web, crée ghost user) | Non |
| 111 | POST | `/events/{event_id}/checkin` | Check-in QR à l'entrée de l'event | Non |

> **Total mis à jour : 111 endpoints**

## 23. Event pre-registration (4 endpoints)

| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 109 | POST | `/auth/event-preregister` | Pré-inscription event — envoie OTP WhatsApp | Non |
| 110 | POST | `/auth/event-preregister/verify` | Vérifie OTP, crée ghost user, génère QR | Non |
| 111 | GET | `/events/{event_id}/stats` | Stats anonymes pour page web (compteur, quartiers) | Non |
| 112 | POST | `/events/{event_id}/checkin` | Check-in QR à l'entrée de l'event | Staff |

---

> **Total final : 112 endpoints**
