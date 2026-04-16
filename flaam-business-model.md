# Flaam — Business Model

> Document strategique
> Version : avril 2026
> **Ligne directrice : donner assez pour seduire, pas assez pour remplacer le paiement.**

---

## Philosophie

Flaam est une app de dating **humaine**, pas extractive. La monetisation existe pour :
1. Couvrir les couts serveurs et SMS/WhatsApp
2. Payer le developpement continu
3. Financer l'expansion geographique
4. Recompenser l'effort du fondateur

**Mais elle n'existe pas pour** :
- Extraire le maximum de revenus par utilisateur
- Creer de l'anxiete artificielle pour pousser a l'achat
- Bloquer l'acces aux fonctionnalites essentielles
- Manipuler les comportements via dark patterns

---

## Les 6 principes non-negociables

### 1. Les messages sont gratuits, toujours

Jamais de paywall sur la messagerie. Jamais de limite artificielle de messages. Si deux personnes ont matche, elles peuvent discuter librement, autant qu'elles veulent.

### 2. Un user free doit pouvoir rencontrer quelqu'un

5 likes par jour = environ 1 vraie rencontre en 2-3 semaines (taux de match 20%, taux de reponse 50%, taux de RDV 50%). C'est le minimum fonctionnel. On ne descend jamais en-dessous.

### 3. Zero notification manipulatrice

Les notifications push servent uniquement a :
- Nouveau match
- Nouveau message
- Event dans ta ville
- Rappel safety (timer rendez-vous)

**Jamais pour :** "5 personnes t'ont like, paie pour voir !", "Offre speciale 50% !", "Ta chance de rencontrer l'amour expire !"

### 4. Zero urgence artificielle

Pas de compte a rebours faux. Pas de "Offre valable 10 min !". Pas de "Derniere chance !". Le prix est le prix, tout le temps.

### 5. Maximum 2 niveaux

Free et Premium. C'est tout. Pas de Gold, Platinum, Diamond, VIP. La complexite = manipulation. La simplicite = confiance.

### 6. Le premium accelere, il ne debloque pas

Toutes les fonctionnalites essentielles sont en free. Le premium offre :
- De la **vitesse** (plus de likes, priorite dans le feed)
- Du **confort** (rewind, incognito, quartiers etendus)

**Jamais l'acces** a quelque chose de fondamental (match, conversation, safety).

---

## Structure tarifaire

### Pricing premium

| Plan | Prix | Cible |
|------|------|-------|
| **Hebdomadaire** | 1 500 FCFA (~$2.50) | Entree facile, engagement court |
| **Mensuel** | 5 000 FCFA (~$8.00) | Meilleur rapport qualite/prix (incentive mensuel) |

**Pas de plan annuel au MVP.** Pas d'engagement long. On veut que le user puisse arreter facilement = signe de confiance.

**Mode de paiement principal :** mobile money via Paystack
- Togo : Flooz, T-Money
- Cote d'Ivoire : Orange Money, MTN MoMo, Moov Money, Wave
- Nigeria (expansion) : Verve, USSD, tous les momo

**Commission Paystack :** 1,5% a 3,5% selon le canal. Negligeable.

### Comparaison Free vs Premium

| Fonctionnalite | Free | Premium |
|---------------|------|---------|
| **Likes quotidiens** | 5/jour | Illimites |
| **Quartiers actifs** | 3 | 5 |
| **Quartiers "interested"** | 3 | 6 |
| **Spots tagues** | 5 | 12 |
| **Voir qui t'a like** | Count + 3 aperçus floutes | Liste complete avec photos |
| **Rewind (annuler dernier skip)** | 1/jour | Illimite |
| **Mode incognito** | Non | Oui |
| **Badge premium discret** | Non | Oui (visible par tes matchs uniquement) |
| **Messagerie** | **Illimitee** | Illimitee |
| **Blocker/reporter** | **Gratuit** | Gratuit |
| **Ice-breakers** | **Disponibles** | Disponibles |
| **Events Flaam** | **Acces libre** | Acces libre |
| **Priority de rencontre** (ex super-like) | 500 FCFA/unite | 2 inclus/semaine |

**Note :** les elements en **gras** sont les fonctionnalites essentielles qui restent toujours gratuites.

---

## Endpoint `GET /matches/likes-received` : 2 niveaux

Cet endpoint a ete code en Session 6 en mode "403 free / complet premium". Il doit etre modifie en Session 7 pour implementer le mode 2-tier.

### Mode free

```json
{
  "total_count": 7,
  "preview": [
    {"blurred_photo_url": "...", "first_letter": "A"},
    {"blurred_photo_url": "...", "first_letter": "K"},
    {"blurred_photo_url": "...", "first_letter": "Y"}
  ],
  "message_fr": "Tu as 7 personnes qui t'ont like. L'algorithme va les mettre progressivement dans ton feed si vous vous correspondez. Passe Premium si tu veux les voir maintenant.",
  "message_en": "7 people have liked you. The algorithm will progressively add them to your feed if you match. Go Premium to see them now.",
  "is_premium_user": false
}
```

### Mode premium

```json
{
  "total_count": 7,
  "profiles": [
    { /* FeedProfileItem complet */ }
  ],
  "is_premium_user": true
}
```

### Philosophie

L'utilisateur free **sait qu'il est desirable** (preuve visuelle par les 3 aperçus floutes). Il comprend qu'il **n'est pas oblige de payer** pour rencontrer ces personnes. Il a le **choix** : attendre ou accelerer.

Zero anxiete. Zero manipulation. Juste une proposition claire.

---

## Mecaniques de revenus

### 1. Subscriptions premium (80% des revenus)

Le coeur du modele. Recurrent, previsible.

**Estimations de conversion :**
- Cible : 3-5% des users actifs
- Realistic : 2-3% en Afrique (sensibilite prix)
- Optimisation possible via incentives contextuels (voir section Freebies)

### 2. "Priorite de rencontre" (10% des revenus)

Anciennement "Super-like". Renomme pour rester aligne avec la philosophie.

**Effet :**
- Tu apparais en **premier dans le feed** de la personne pendant 48h
- La personne voit un indicateur discret : "Cette personne s'interesse particulierement a toi"
- Si vous matchez, l'ice-breaker mentionne "votre priorite mutuelle"
- La personne ne sait pas que tu as utilise un "boost"

**Prix :**
- 1 unite = 500 FCFA
- Pack 5 = 2 000 FCFA (20% de reduction)
- Pack 10 = 3 500 FCFA (30% de reduction)

**Limites :**
- Max 2 par semaine par utilisateur (anti-spam)
- **Gratuit** si tu as fait un check-in a un event Flaam dans les 7 derniers jours (boucle IRL -> app)

### 3. Events sponsorises (5-10% des revenus)

**Modele A — MVP :** le lieu paie un forfait fixe pour heberger un Flaam event.
- Afterwork : 15 000 FCFA
- Brunch : 20 000 FCFA
- Event themed : 25 000 - 30 000 FCFA

**Inclus dans le forfait :**
- Page web de pre-inscription sur flaam.app
- QR codes pour check-in
- Promotion dans l'app (feed banner, notifications non-manipulatrices)
- Post-event : teaser WhatsApp aux participants
- Visibilite "Flaam x [Nom du lieu]" pendant 14 jours

**Modele B — Phase 2 (apres 5 000 users actifs) :** revenue share sur les consos
- 5-10% des ventes du soir reverses a Flaam
- Plus scalable, mais necessite integration de tracking

### 4. Boosts de profil ponctuels (2-5% des revenus)

**Effet :** ton profil est promu en haut du feed des users compatibles pendant une duree limitee.

**Prix :**
- 30 min = 1 000 FCFA
- 1 heure = 1 500 FCFA

**Limites :**
- Max 1 boost par 24h
- Fonctionne uniquement pendant les heures actives de ta ville (18h-23h locale)

---

## Freebies et incentives

### Principe : "donner assez pour seduire, pas remplacer le paiement"

Les freebies servent a :
1. **Equilibrer le ratio H/F** (offrir plus aux femmes pour qu'elles restent)
2. **Recompenser les comportements productifs** (events, invitations, profils complets)
3. **Creer de la viralite** (invitations recompensees)
4. **Permettre l'essai** avant l'achat

### Premium offert a l'inscription (one-shot)

| Profil | Premium offert | Justification |
|--------|----------------|---------------|
| Femme inscrite | **5 jours** | Valeur inherente a leur presence, equilibre ratio |
| Homme avec code d'invitation | **3 jours** | Recompense l'effort du reseau social |
| Homme checked_in a un event Flaam | **3 jours** | Recompense l'effort physique de venir |
| Homme sortant de waitlist classique | **2 jours** | Essai minimal pour goûter au premium |
| Ambassadrice (programme) | **Premium a vie** | Status VIP, contribution communautaire |

### Recompenses d'actions (recurrentes)

| Action | Recompense | Limites |
|--------|-----------|---------|
| Completer son profil a 100% | +1 jour | **Une seule fois dans la vie** |
| Inviter un ami qui s'active 7 jours | +2 jours | **Max 2 amis/mois** (= 4 jours max via invitations) |
| Check-in a un event Flaam | +2 jours | 1 recompense par event |

### Eligibilite pour inviter (anti-abuse)

Un inviteur est eligible aux recompenses uniquement si :
- **Compte actif depuis au moins 30 jours**
- **Profile completeness ≥ 0.70**
- **Non banni, sans flag moderation**

Eligibilite de l'invite (pour que l'inviteur gagne ses jours) :
- **S'inscrit via le code d'invitation**
- **Complete son profil dans les 7 jours**
- **Reste actif 7 jours consecutifs** (pas un fake account cree pour farmer)

### Le cap critique : 10 jours/mois

**Maximum absolu de 10 jours de premium gratuits par mois glissant**, tous freebies confondus (inscription + actions recompensees).

Sans ce cap :
- Abus par farming d'invitations
- Le premium gratuit devient la norme
- Revenus instables

Avec ce cap :
- Frustration saine ("j'aimerais 5 jours de plus ce mois-ci")
- Conversion naturelle vers le paiement
- Impossibilite de farmer

### Exemple concret : user typique engage

**Mois 1 — inscription** (user femme) :
- +5 jours premium a l'inscription

Elle utilise ses 5 jours, voit la valeur du premium, et a 2 options :
- Payer 1 500 FCFA/semaine ou 5 000 FCFA/mois
- Gagner plus de jours gratuits via actions

**Mois 2 — user engage** :
- 2 amis invites et actives = 4 jours
- 2 events check-in = 4 jours
- Completeness deja 100% (bonus one-shot consomme au mois 1)
- **Total : 8 jours de premium gratuit (< cap 10)**
- Il/elle peut choisir de payer pour les 22 jours restants ou rester free

**Mois 3 — user super-engage** :
- 3 amis invites (2 seulement recompenses, cap mois) = 4 jours
- 3 events = 6 jours (depasse le cap 10, donc plafonne a 6 jours supplementaires)
- **Total : 10 jours gratuits (cap atteint)**

**Mois 4 et suivants :** le cycle se repete. L'user a experimente le premium, connait sa valeur, et decide consciemment de payer ou pas.

---

## Anti-patterns interdits

Liste exhaustive des mecaniques **jamais** implementees chez Flaam :

### Monetisation

- Paywall sur la messagerie
- Limite artificielle de messages
- Paywall sur le blocker/reporter
- "Voir qui t'a vu" (stalker-friendly)
- Filtres de recherche ethniques ou physiques en premium
- Plus de 3 niveaux d'abonnement premium
- Renouvellement automatique trompeur (doit etre explicite)

### Notifications

- Notifications push pour pousser l'achat ("5 personnes t'ont like, paie !")
- Notifications factices ("Ton match te manque, paie pour le recuperer !")
- Emails marketing agressifs (max 1 email promotionnel par mois)

### Urgence artificielle

- Comptes a rebours faux ("Offre valable 10 min !")
- "Promotions" permanentes ("Derniere chance -50% !")
- Faux scarcity ("Plus que 3 places !")

### Mecaniques psychologiques

- Gamification manipulatrice (streaks quotidiens qui forcent la connexion)
- FOMO artificiel (peur de manquer)
- Comparaison sociale ("Les users premium matchent 5x plus !")
- Recompenses variables type casino

### Page de desabonnement

- Boutons "Annuler" caches derriere 6 clics
- Dark patterns "Etes-vous sur ?" x 3 avec propositions de reduction
- Obligation de contacter le support pour se desabonner

**Regle simple :** la page de desabonnement est accessible en **2 clics** depuis les settings. Zero friction inutile. Confiance > revenus a court terme.

---

## Projections financieres

### Hypotheses de base

- Cout serveur VPS (Hetzner CPX21) : 15-25 €/mois ~ 10 000-15 000 FCFA
- Cout Termii SMS/WhatsApp : 8 000-15 000 FCFA/mois
- Cout IA moderation (des 1 000 users) : ~30 000 FCFA/mois
- **Total couts MVP :** ~35 000 FCFA/mois
- **Break-even premium :** ~10 users payants/mois

### Scenario conservateur — Lome uniquement

| Mois | Users actifs | Premium (4%) | Revenus | Profit net |
|------|--------------|--------------|---------|------------|
| 1-3 | 100-500 | 4-20 | 20-100k FCFA | -30k a 65k |
| 4-6 | 500-1 500 | 20-60 | 100-300k | 65-265k |
| 7-9 | 1 500-3 000 | 60-120 | 300-600k | 265-565k |
| 10-12 | 3 000-5 000 | 120-200 | 600k-1M | 565k-965k |

**Revenu annuel total estime :** 4-5 millions FCFA (~$7 000-8 500)

### Scenario optimiste — Lome + Abidjan

| Mois | Users actifs | Premium (3%) | Revenus | Profit net |
|------|--------------|--------------|---------|------------|
| 1-6 | 0-2 000 | 0-60 | 0-300k | variable |
| 7-12 | 5 000-10 000 | 150-300 | 750k-1.5M | ~1.4M |

**Revenu annuel total estime :** 10-15 millions FCFA (~$16 000-25 000)

### Projection annee 2 (si product-market fit valide)

- 10 000-30 000 users actifs (Lome + Abidjan + Dakar)
- 3% de conversion premium
- 300-900 premium x 5 000 FCFA = **1.5-4.5M FCFA/mois**
- Plus revenus super-likes, events, boosts
- **Total annee 2 : 20-60M FCFA (~$33k-100k)**

### Metriques cles a surveiller

| Metrique | Cible sante | Alerte si |
|----------|-------------|-----------|
| Ratio H/F actif | 60/40 - 50/50 | > 75% H |
| Taux de conversion premium | 3-4% | < 1.5% |
| Churn premium mensuel | < 25% | > 40% |
| NPS (net promoter score) | > 40 | < 20 |
| Temps median avant 1er match | < 7 jours | > 21 jours |
| DAU/MAU (daily active / monthly) | > 30% | < 15% |
| Retention J30 | > 25% | < 10% |

---

## Decisions produit a trancher plus tard

Questions ouvertes qui meriteront une reflexion a des seuils precis.

### Apres 5 000 users actifs

**Q1 : Evenements payants pour users ?**
- Idee : certains events premium (concerts, rooftop exclusifs) sont reserves aux users premium
- Risque : segmentation qui divise la communaute
- Decision a prendre avec des vraies donnees

**Q2 : Plan annuel ?**
- Pour/contre : revenus upfront vs perte de confiance
- Peut-etre avec garantie "rembourse si pas de match en 30 jours" ?

**Q3 : Partenariats marques ?**
- Un brand de boisson parraine-t-il nos events ?
- Comment garder l'independance produit ?

### Apres 20 000 users actifs

**Q4 : Expansion internationale ?**
- Senegal, Cameroun, Benin ?
- Meme business model ou adaptation locale ?

**Q5 : Fonctionnalites premium avancees ?**
- Appels video ?
- Groupes d'amis qui matchent ensemble (double dates) ?
- Tester et abandonner si pas de demande reelle

### Apres 50 000 users actifs

**Q6 : Levee de fonds ou bootstrap continue ?**
- Quel niveau de dilution accepter ?
- Qui sont les "bons" investisseurs pour un produit humain comme Flaam ?

**Q7 : Vente partielle ou totale ?**
- A quel moment envisager une exit ?
- Valeurs a preserver dans toute negociation

---

## Regles d'or pour toute decision future

1. **Test "rencontrer sans payer"** : un user free peut-il realistiquement trouver quelqu'un en 2-3 semaines sans depenser ?

2. **Test "honte a ma mere"** : peux-tu expliquer fierement la feature a quelqu'un que tu respectes ?

3. **Test "concurrent ethique"** : si un concurrent honnete proposait les memes features en plus simple sans cette mecanique discutable, les users partiraient-ils ?

4. **Test "long terme"** : cette decision augmente-t-elle la retention sur 6 mois, ou just les revenus du mois prochain ?

Si une de ces 4 questions a une reponse insatisfaisante, **la feature ne va pas en production**.

---

## Communication externe

### Ce qu'on dit aux users

Dans l'app, la page premium dit simplement :
- "Flaam Premium — Pour rencontrer plus vite ceux qui te plaisent"
- Liste des fonctionnalites
- Prix en FCFA
- Bouton "Passer Premium"
- Lien "Comment ca marche ?" vers une page explicative

Ton : honnete, direct, pas de hype.

### Ce qu'on dit aux partenaires events

"Flaam organise des events qui ramenent des jeunes professionnels qualifies dans ton etablissement. 24-60 personnes par event, toutes verifiees, toutes locales. On te donne de la visibilite dans notre app et on t'amene des clients. Toi tu nous offres le lieu."

Ton : professionnel, win-win, clair.

### Ce qu'on dit aux investisseurs (plus tard)

"Flaam construit un produit dating humain et scalable en Afrique francophone. On a des revenus recurrents, une retention solide, et une expansion geographique naturelle. On ne sera jamais le plus rentable par user, mais on sera le plus aime, le plus durable et le plus mature."

Ton : long-termiste, different, assume.

---

## Iteration de ce document

Ce document est vivant. Il sera revise :
- **Tous les 3 mois** apres lancement (review des chiffres reels vs projections)
- **A chaque seuil** (1k users, 5k, 10k, 20k) pour decider des nouvelles mecaniques
- **A chaque doute ethique** rencontre en production (un dark pattern qui m'est propose, je reviens ici)

La philosophie (section "6 principes non-negociables" et "Anti-patterns interdits") **ne change jamais**. Les chiffres et mecaniques peuvent s'ajuster.

---

## Derniere regle, la plus importante

> **Si tu as un doute sur une decision de monetisation, reviens a ce document.**
>
> Si le doute persiste, ne fais pas la feature.
>
> Flaam peut attendre un mois de plus de revenus. Flaam ne peut pas se permettre de perdre sa valeur ethique.

---

**Signe :** Raouf, fondateur de Flaam
**Date de creation :** avril 2026
**Derniere revision :** avril 2026
