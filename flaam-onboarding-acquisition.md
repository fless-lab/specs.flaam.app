# Flaam — Onboarding & stratégie d'acquisition

> Document de référence produit & technique
> Version : avril 2026
> Marché : jeunes professionnels, Afrique de l'Ouest, lancement Lomé

---

## Les 3 portes d'entrée

Un utilisateur arrive dans Flaam par l'un de ces trois chemins. Chaque porte est **indépendante** — aucune n'est un pré-requis pour les autres.

```
PORTE 1                    PORTE 2                    PORTE 3
Classique                  Invitation                 Event
│                          │                          │
│ Télécharge l'app         │ Reçoit un code           │ Page web event
│ directement              │ FLAAM-XXXX               │ (pas d'app)
│                          │                          │
▼                          ▼                          ▼
OTP SMS ──────────────────── OTP SMS ──────────────────── OTP SMS (web)
│                          │                          │
│                          │ Saisit le code           │ Ghost user créé
│                          │ → skip waitlist          │ (numéro + prénom)
│                          │                          │
│                          │                          │ QR check-in event
│                          │                          │
│                          │                          │ WhatsApp teaser
│                          │                          │ "Télécharge Flaam"
│                          │                          │
▼                          ▼                          ▼
     ┌──────── ONBOARDING COMMUN ─────────┐
     │ Prénom · Genre · Âge · Intention   │
     │ Selfie vérifié · 3+ photos         │
     │ Quartiers · Spots · 1+ prompt      │
     └────────────────────────────────────┘
                     │
                     ▼
              ┌─── ACTIVATION ───┐
              │                  │
          Femme              Homme
           │                    │
      Immédiate          Waitlist (sauf
                          code ou event)
```

---

## Porte 1 — Classique (app directe)

Le flow standard. L'utilisateur télécharge l'app depuis le Play Store, sans event ni invitation.

### Flow technique complet

**Étape 1 — OTP (passwordless).** L'utilisateur entre son numéro de téléphone. L'app envoie une requête `POST /auth/otp/request`. Le serveur génère un code OTP 6 chiffres, le stocke dans Redis (TTL 10 minutes), et l'envoie par **SMS** via Termii. Si le SMS n'est pas reçu après 30 secondes, l'app propose "Renvoyer par WhatsApp ?". Pas de mot de passe. Jamais.

**Étape 2 — Code d'invitation (optionnel).** "Tu as un code ?" → champ texte FLAAM-XXXX. Si valide : skip la waitlist (même pour un homme). Si pas de code : "Je n'ai pas de code" → continue normalement. Les femmes passent dans tous les cas, code ou pas.

**Étape 3 — Pays + ville.** Le pays est auto-détecté via le préfixe téléphonique (+228 = Togo). L'écran ville est toujours affiché, même s'il n'y a qu'une seule ville active. Les villes en phase teaser montrent un compteur waitlist avec des silhouettes grisées. Les villes en phase hidden ne sont pas affichées du tout.

**Étape 4 — Profil de base.** Prénom (pas de nom de famille), date de naissance, genre, genre recherché, intention (serious / getting_to_know / friendship / open).

**Étape 5 — Selfie de vérification.** CameraX + ML Kit liveness detection. Obligatoire. Le selfie est utilisé uniquement pour la vérification d'identité — il n'est pas stocké dans les photos de profil. L'utilisateur ne peut pas uploader une photo de sa galerie : c'est un selfie live uniquement.

**Étape 6 — Photos (min 3, max 6).** Upload depuis la galerie. Compression côté client avant upload (qualité 80%, max 2048px de large). Modération asynchrone côté serveur (score + review si nécessaire).

**Étape 7 — Quartiers.** "Où tu vis ?", "Où tu travailles ?", "Où tu traînes ?" Chaque réponse associe l'utilisateur à un quartier avec un type de relation (`lives_in`, `works_in`, `hangs_in`). Maximum 3 quartiers en free, 5 en premium. Option "D'autres quartiers qui t'intéressent ?" → type `interested`, max 3 free / 6 premium. Ces quartiers ne sont pas visibles sur le profil public.

**Étape 8 — Spots.** Les spots fréquentés : maquis, cafés, salles de sport, coworkings, marchés. Recherche par nom ou parcours sur la carte. Maximum 5 spots en free, 12 en premium. Chaque spot ajouté commence au niveau "Déclaré" (0 check-in).

**Étape 9 — Prompts (min 1).** L'utilisateur choisit parmi 20+ prompts pré-écrits et écrit sa réponse. Exemples : "Un dimanche idéal à Lomé c'est...", "Le maquis que je recommande toujours...", "Mon quartier préféré parce que...", "Ce week-end tu me trouves à..."

**Étape 10 — Activation.** Femme → activée immédiatement, premier feed généré au prochain batch nocturne. Homme avec code d'invitation valide → activé. Homme sans code → écran waitlist avec position + compteur + "Invite 3 amis pour passer devant".

### Infrastructure SMS/OTP

| Usage | Canal primary | Fallback | Coût |
|-------|--------------|----------|------|
| OTP (auth) | SMS via Termii | WhatsApp après 30s | $0.025 (~15 FCFA) |
| Rappels event | WhatsApp via Termii | SMS | $0.006 (~3.6 FCFA) |
| Teasers post-event | WhatsApp via Termii | SMS | $0.006 (~3.6 FCFA) |

L'OTP est envoyé par SMS en priorité (pas WhatsApp) parce que le SMS arrive sur n'importe quel numéro de téléphone, même sans WhatsApp. Avec le dual SIM courant au Togo (Togocel + Moov), beaucoup de gens ont WhatsApp sur un numéro et s'inscrivent avec l'autre. Un OTP WhatsApp qui ne livre pas = utilisateur perdu.

**Provider :** Termii uniquement au MVP. Plan Starter à $0/mois (sandbox inclus pour le dev). Les classes Twilio/Vonage ne sont pas implémentées — seule l'interface abstraite `BaseSMSProvider` est codée pour ajouter d'autres providers plus tard.

**Coût estimé premier mois (500 inscriptions) :** 500 OTP SMS × $0.025 = $12.50. Plus 200 messages event WhatsApp × $0.006 = $1.20. **Total : ~$14/mois (~8 400 FCFA).**

---

## Porte 2 — Invitation (code d'une amie)

Le système d'invitations crée de la viralité cross-genre et contrôle la qualité du pool.

### Qui reçoit des codes

| Type d'utilisateur | Codes reçus | Expiration |
|--------------------|-------------|------------|
| Femme inscrite | 3 codes | 30 jours |
| Ambassadrice | 50 codes (renouvelables) | 90 jours |
| Homme non-ambassadeur | 0 codes | — |

Les hommes n'ont pas de codes à distribuer. C'est intentionnel : ça crée une dynamique où les hommes demandent un code à une amie sur Flaam, ce qui augmente la viralité cross-genre.

### Format du code

`FLAAM-XXXX` où XXXX est 8 caractères alphanumériques aléatoires. Généré via `secrets.token_urlsafe(6)[:8].upper()`.

### Bénéfices

**Pour la personne invitée :** skip la waitlist (même si c'est un homme). C'est LA raison pour laquelle les gens veulent un code.

**Pour l'inviteuse :**

| Invitations utilisées | Récompense |
|----------------------|------------|
| 1 | +2 likes offerts |
| 3 | 1 semaine premium gratuit |
| 5 | 1 mois premium gratuit |

### Flow technique

1. Après activation, `POST /invites/generate` crée les codes.
2. L'utilisatrice partage un code via Android ShareSheet (WhatsApp, SMS, copier).
3. Le destinataire télécharge l'app, entre le code à l'étape 2 de l'onboarding.
4. `POST /invites/validate` vérifie le code (existe, actif, pas expiré, pas utilisé).
5. `POST /invites/redeem` marque le code comme utilisé, skip la waitlist, crédite l'inviteuse.

### Endpoints

| Méthode | Path | Auth |
|---------|------|------|
| POST | `/invites/generate` | Oui |
| GET | `/invites/me` | Oui |
| POST | `/invites/validate` | Non (onboarding) |
| POST | `/invites/redeem` | Oui |

---

## Porte 3 — Event (la plus puissante)

L'event n'est pas un canal de marketing pour l'app. L'event EST l'app. Les gens viennent pour la soirée, et ils se retrouvent dans Flaam sans l'avoir cherché.

### Le concept : le ghost user

Quand quelqu'un se pré-inscrit à un event via la page web (pas l'app), il est créé comme un **ghost user** dans Flaam. Il a un `user_id`, un numéro vérifié, un prénom — mais zéro profil. Il n'apparaît dans aucun feed. Il ne peut rien faire sauf être associé à un event.

### Flow complet

#### Avant l'event

**1. Découverte.** Post Instagram/WhatsApp : "Afterwork Flaam au Café 21 — vendredi 20h". Lien vers `flaam.app/events/afterwork-cafe21`.

**2. Page web de l'event.** Hébergée sur flaam.app. Affiche le nom de l'event, la date, le lieu, la catégorie. Un compteur anonyme : "24 inscrits. 8 fréquentent Tokoin." (agrégation anonyme, pas de noms ni photos). Bouton "Réserve ta place (30 secondes)".

**3. Pré-inscription légère (30 secondes).** Numéro de téléphone → OTP SMS → prénom. C'est tout. Pas de selfie, pas de profil, pas d'app à télécharger. En coulisses : `POST /auth/event-preregister` → OTP → `POST /auth/event-preregister/verify` → ghost user créé + QR code généré.

**4. Confirmation.** WhatsApp 2h avant l'event : "C'est ce soir ! Présente ce QR code à l'entrée." Le QR code est lié au `user_id` du ghost user.

#### Pendant l'event

**5. Check-in à l'entrée.** Scan du QR code (`POST /events/{event_id}/checkin`). Le status passe de `registered` à `checked_in`. Le spot de l'event est automatiquement ajouté aux spots du ghost user.

**6. Pendant la soirée.** Les gens se croisent, se parlent, socialisent. Flaam n'intervient pas. L'event est une vraie soirée, pas une expérience d'app.

#### Après l'event

**7. Le teaser.** Celery task 2h après la fin de l'event. WhatsApp envoyé aux checked_in non convertis : "Tu as croisé 24 personnes au Café 21 ce soir. 7 ont déjà complété leur profil Flaam. Télécharge l'app pour les découvrir demain."

**8. Téléchargement de l'app.** Le lien WhatsApp ouvre le Play Store. L'app détecte le numéro de téléphone déjà enregistré (ghost user). Pas besoin de re-saisir le numéro — juste OTP pour confirmer. Le champ `is_ghost_conversion = true` dans la réponse OTP déclenche le mode event.

**9. Onboarding accéléré.** Le prénom est pré-rempli (skip l'écran). Le spot de l'event est pré-coché. Les tags sont pré-suggérés depuis la catégorie de l'event (afterwork → "afterwork", "sortir", "networking" — pré-cochés mais décochables). Message motivant : "7 personnes du Café 21 ont déjà complété leur profil. Complete le tien pour les découvrir demain."

**10. Feed du lendemain.** Le batch nocturne génère le feed. Les participants de l'event qui ont complété leur profil apparaissent avec un badge "Était au Café 21". Boost temporaire de +15 points sur le score géo entre co-participants pendant 7 jours. Ice-breaker contextuel : "Vous étiez tous les deux au Café 21. Qu'est-ce que tu en as pensé ?"

### Mapping des tags par catégorie d'event

| Catégorie | Tags pré-suggérés |
|-----------|-------------------|
| afterwork | afterwork, sortir, networking |
| sport | sport, fitness, outdoor |
| brunch | food, brunch, chill |
| cultural | culture, art, sortir |
| networking | networking, pro, business |
| workshop | apprendre, workshop, créatif |
| outdoor | outdoor, nature, aventure |

### Endpoints spécifiques

| Méthode | Path | Auth | Description |
|---------|------|------|-------------|
| POST | `/auth/event-preregister` | Non | Envoie OTP pour pré-inscription web |
| POST | `/auth/event-preregister/verify` | Non | Vérifie OTP, crée ghost user, génère QR |
| GET | `/events/{event_id}/stats` | Non | Compteur anonyme pour la page web |
| POST | `/events/{event_id}/checkin` | Staff | Scan QR → checked_in |

### Statuts du ghost user

```
ghost ──► pre_registered ──► in_progress ──► completed ──► activated
  │            │
  │            └── QR scanné à l'event
  │
  └── Pré-inscription web (numéro + prénom uniquement)
```

- `ghost` : créé via la page web. Pas d'app.
- `pre_registered` : check-in fait à l'event. Toujours pas d'app.
- `in_progress` : a téléchargé l'app, onboarding en cours.
- `completed` : profil complet, en attente d'activation.
- `activated` : dans le matching pool.

---

## Waitlist genrée

La waitlist est le mécanisme qui contrôle le ratio hommes/femmes.

### Règles

| Genre | Comportement |
|-------|-------------|
| Femme | **Skip automatique.** Jamais en waitlist. Activée immédiatement. |
| Homme avec code | Skip la waitlist. Activé immédiatement. |
| Homme sans code (event checked_in) | Skip la waitlist. Activé immédiatement. |
| Homme sans code | En waitlist. Position affichée. |

### Libération automatique

Une Celery task tourne toutes les 6 heures. Pour chaque ville en phase LAUNCH ou GROWTH :

1. Calcule le ratio femmes/total parmi les utilisateurs actifs.
2. Si le ratio femmes est **> 40%** ET qu'il y a au moins **30 femmes actives** → libère un batch de 50 hommes (par ordre de position dans la file).
3. Chaque homme libéré reçoit une notification push : "C'est ton tour ! Flaam est maintenant actif pour toi."
4. Si le ratio est trop bas → aucun homme n'est libéré. La file attend.

### Ce que l'homme en waitlist voit

- Sa position : "#89 sur 327"
- Une barre de progression
- "Invite 3 amis et passe dans le top 10%"
- Un champ "Tu as un code ?" pour saisir un code d'invitation
- Le message : "On ouvre Flaam par petits groupes pour garantir une bonne expérience."

### Ce que la femme voit

Rien. Elle ne passe jamais par la waitlist. L'activation est transparente.

---

## Programme ambassadrices

Les ambassadrices sont 10-15 femmes influentes dans la scène locale. Pas des influenceuses Instagram — des femmes que leur cercle respecte. Coiffeuses connues, DRH, organisatrices d'events, blogueuses food.

### Ce qu'elles reçoivent

- Early access (avant le lancement public)
- Premium à vie
- 50 codes d'invitation (renouvelables)
- Badge discret sur leur profil (visible uniquement par les matchs)
- Accès au salon ambassadrices (groupe WhatsApp privé avec l'équipe)

### Ce qu'elles font

Elles utilisent l'app normalement et en parlent à leur cercle. Pas de contenu sponsorisé, pas de posts forcés. C'est organique.

### Endpoints admin

| Méthode | Path | Description |
|---------|------|-------------|
| POST | `/admin/ambassadors` | Nommer une ambassadrice |
| DELETE | `/admin/ambassadors/{user_id}` | Retirer le statut |
| GET | `/admin/ambassadors` | Liste + stats par ville |

---

## Plan de lancement (Lomé)

| Semaine | Action | Cible |
|---------|--------|-------|
| S-8 | Identifier 15 ambassadrices, leur donner l'app | Femmes |
| S-6 | Ouvrir la waitlist hommes, afficher le compteur | Hommes |
| S-4 | Premier event Flaam (brunch, 60 personnes) | Mixte |
| S-3 | Ouvrir l'inscription femmes (invite-only, 3 invites chacune) | Femmes |
| S-2 | Deuxième event (afterwork) | Mixte |
| S-1 | Ouvrir 50% de la waitlist hommes | Hommes |
| S0 | Lancement public, troisième event | Tous |

L'objectif est d'arriver au lancement public avec un ratio 60/40 femmes/hommes. Il s'inversera naturellement vers 40/60 en quelques semaines, ce qui est le ratio sain.

### Budget pour les 3 premiers events

| Poste | Coût |
|-------|------|
| Lieu | Gratuit (le maquis gagne sur les consos) |
| Branding (kakemono + QR) | ~15 000 FCFA |
| Premier verre offert aux femmes (15 × 1 500) | ~22 500 FCFA |
| OTP + messages WhatsApp | ~8 400 FCFA/mois |
| **Total** | **~45 000 FCFA** (~$75) |

---

## 4 phases de lancement d'une ville

| Phase | Nom | Seuil | Comportement |
|-------|-----|-------|-------------|
| 1 | TEASER | 0-500 inscrits | Ville visible mais grisée. Waitlist active. Compteur "247/500". Pas de feed. |
| 2 | LAUNCH | 500+ inscrits, 150+ femmes | Feed activé. Boost agressif nouveaux profils (5× au lieu de 3×). Feed étendu à 15-20 profils/jour. |
| 3 | GROWTH | 500-2000 | Marketing organique. Events locaux. Monitoring ratio H/F. |
| 4 | STABLE | 2000+ | Algo normal. Feed revient à 12 profils/jour. Prêt à bootstrapper les villes voisines. |

La transition entre phases est gérée par l'admin dans la DB. Pas de transition automatique — c'est une décision humaine basée sur la qualité du pool, pas juste les chiffres.

---

## Modèle de données clé

### WaitlistEntry

| Champ | Type | Description |
|-------|------|-------------|
| city_id | UUID | Ville concernée |
| user_id | UUID | Utilisateur en attente |
| gender | string | "male" / "female" / "other" |
| position | int | Position dans la file (0 = activé immédiatement) |
| status | string | "waiting" / "invited" / "activated" / "expired" |
| invite_code_used | string? | Code utilisé pour skip |

### InviteCode

| Champ | Type | Description |
|-------|------|-------------|
| code | string | "FLAAM-XXXX" (unique) |
| creator_id | UUID | Qui a créé le code |
| type | string | "standard" (3/femme) / "ambassador" (50) |
| used_by_id | UUID? | Qui l'a utilisé |
| expires_at | datetime | 30j (standard) / 90j (ambassador) |

### EventPreregistration

| Champ | Type | Description |
|-------|------|-------------|
| event_id | UUID | L'event concerné |
| user_id | UUID | Le ghost user |
| qr_code | string | "EVT-{short}-{random}" (unique) |
| status | string | "registered" / "checked_in" / "converted" / "expired" |
| suggested_tags | JSON | Tags pré-suggérés depuis la catégorie |

---

## Résumé des endpoints d'acquisition

| # | Méthode | Path | Auth | Porte |
|---|---------|------|------|-------|
| 100 | POST | `/invites/generate` | Oui | 2 |
| 101 | GET | `/invites/me` | Oui | 2 |
| 102 | POST | `/invites/validate` | Non | 2 |
| 103 | POST | `/invites/redeem` | Oui | 2 |
| 104 | POST | `/admin/ambassadors` | Admin | 2 |
| 105 | DELETE | `/admin/ambassadors/{user_id}` | Admin | 2 |
| 106 | GET | `/admin/ambassadors` | Admin | 2 |
| 107 | GET | `/admin/waitlist/stats` | Admin | — |
| 108 | POST | `/admin/waitlist/release` | Admin | — |
| 109 | POST | `/auth/event-preregister` | Non | 3 |
| 110 | POST | `/auth/event-preregister/verify` | Non | 3 |
| 111 | GET | `/events/{event_id}/stats` | Non | 3 |
| 112 | POST | `/events/{event_id}/checkin` | Staff | 3 |
