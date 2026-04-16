# Flaam — Safety & Anti-Fraud

> Document technique et strategique
> Version : avril 2026
> **Principe : proteger les utilisateurs sans degrader l'experience.**

---

## Vue d'ensemble

Flaam est une app de dating. Les dating apps attirent 3 types de menaces :

1. **Scammers** — faux profils pour extorquer de l'argent
2. **Catfish** — fausse identite pour tromper sur son apparence/genre
3. **Harceleurs** — comportement abusif envers les matchs
4. **Bots** — comptes automatises (ChatGPT pour les messages, photos AI)
5. **Recidivistes** — bannis qui recreent des comptes

Chaque menace a des contre-mesures specifiques, implementees progressivement.

---

## Matrice des protections par session

### Deja implementees (Sessions 1-9)

| Protection | Session | Status | Efficacite estimee |
|---|---|---|---|
| Selfie live obligatoire | S3 | ✅ Fait | 80% des faux classiques |
| AccountHistory anti-recidive | S2 | ✅ Fait | 90% des re-creations |
| Moderation messages (liens, argent, insultes) | S7 | ✅ Fait | 70% des scams en chat |
| Matching anti-spam (behavior_multiplier) | S5 | ✅ Fait | Penalise les like-all |
| Contact blacklist (SHA-256 cote client) | S9 | ✅ En cours | 100% des contacts connus |
| Scam detection 6 signaux | S9 | ✅ En cours | 60% des scammers |
| Block bidirectionnel | S9 | ✅ En cours | Suppression immediate |
| Rate limiting ZSET | S9 | ✅ En cours | Anti-spam API |
| Idempotency middleware | S9 | ✅ En cours | Anti-double-action |
| Timer urgence (stub) | S9 | ✅ En cours | Protection IRL |

### A implementer en Session 10

| Protection | Priorite | Effort |
|---|---|---|
| Moderation photos admin manuelle | HAUTE | 2h |
| Check EXIF authenticite photos | HAUTE | 1h |
| Selfie ↔ photos face comparison (InsightFace ONNX) | HAUTE | 4h |
| Genre selfie vs genre profil | HAUTE | 1h |
| Invalidation selfie si changement genre | HAUTE | 30min |
| NSFW detection (mode onnx) | MOYENNE | 3h |
| Face detection (au moins 1 visage) | MOYENNE | 1h |
| Temporal diversity des photos (EXIF dates) | BASSE | 1h |

### A implementer post-lancement (1000+ users)

| Protection | Horizon | Effort |
|---|---|---|
| Bot behavior detection (regularite reponses) | 3-6 mois | 1 semaine |
| AI image detection (DIRE model ou API) | 6+ mois | 2-3 jours |
| Style d'ecriture matching (sentence embeddings) | 6+ mois | 1 semaine |
| Liveness check avance sur selfie | 3 mois | 2 jours |
| Screen photo detection (moire patterns) | 6+ mois | 3 jours |

---

## Session 10 — Implementation detaillee

### 10.1 Check EXIF authenticite

**Fichier :** integre dans `app/services/photo_moderation_service.py`

**Quand :** appele sur chaque photo uploadee, quel que soit le mode de moderation.

```python
from PIL import Image
from PIL.ExifTags import TAGS

def check_photo_authenticity(file_path: str) -> dict:
    """
    Analyse les metadonnees EXIF pour detecter les photos suspectes.
    
    Signaux :
    - Pas d'EXIF du tout → probablement AI-generated ou screenshot
    - Pas de modele de camera → suspect
    - Resolution carree parfaite (512, 1024, 2048) → AI-generated
    - Ratio de compression JPEG anormal → re-encode ou AI
    
    Retourne un dict avec flags et risk score (0-1).
    NE PAS auto-reject. Flag pour review si risk > 0.5.
    """
    img = Image.open(file_path)
    flags = []
    
    # 1. Absence totale d'EXIF
    exif = img._getexif() if hasattr(img, '_getexif') else None
    if exif is None:
        flags.append("no_exif_data")
    else:
        # 2. Pas de modele de camera
        if 272 not in exif:  # Tag "Model"
            flags.append("no_camera_model")
        # 3. Pas de date de prise de vue
        if 36867 not in exif:  # Tag "DateTimeOriginal"
            flags.append("no_date")
        # 4. Software de retouche (Photoshop, GIMP, etc.)
        software = exif.get(305, "")  # Tag "Software"
        if any(s in str(software).lower() for s in [
            "photoshop", "gimp", "canva", "midjourney", "dall-e",
            "stable diffusion", "comfyui"
        ]):
            flags.append("ai_or_editor_software")
    
    # 5. Resolution carree suspecte (AI-generated = souvent carre)
    w, h = img.size
    if w == h and w in (256, 512, 768, 1024, 2048):
        flags.append("square_suspicious_resolution")
    
    # 6. Ratio trop parfait (exactement 16:9, 4:3 au pixel pres)
    # Les vrais telephones ne font pas des ratios parfaits au pixel pres
    ratio = w / h if h > 0 else 0
    if ratio in (16/9, 9/16, 4/3, 3/4, 1.0):
        flags.append("perfect_ratio")
    
    risk = min(1.0, len(flags) * 0.2)
    return {"flags": flags, "risk": risk}
```

**Mode de gestion :**
- `risk < 0.3` → passe (0-1 flag, probablement OK)
- `risk 0.3-0.6` → log + continue (2-3 flags, surveiller)
- `risk > 0.6` → `moderation_status = "manual_review"` (3+ flags, suspect)

**Config ENV :** `EXIF_CHECK_ENABLED` (default true). Desactivable pour les tests.

### 10.2 Selfie ↔ Photos face comparison

**Fichier :** `app/services/face_verification_service.py`

**Modele :** InsightFace (ArcFace backbone). ONNX, ~30 MB. CPU < 200ms.

**Quand :** appele apres chaque upload de photo ET apres le selfie.

```python
import onnxruntime as ort
import numpy as np

class FaceVerificationService:
    """
    Compare le selfie verifie avec les photos de profil.
    Utilise ArcFace pour generer des embeddings faciaux 128D.
    Cosine similarity pour mesurer la ressemblance.
    
    Seuils :
    - similarity >= 0.7 : meme personne (OK)
    - similarity 0.5-0.7 : probablement meme personne (warning)
    - similarity < 0.5 : probablement PAS meme personne (flag review)
    - similarity < 0.3 : clairement pas meme personne (flag + notif admin)
    """
    
    THRESHOLD_OK = 0.7
    THRESHOLD_WARNING = 0.5
    THRESHOLD_MISMATCH = 0.3
    
    def __init__(self, model_path: str):
        self.session = ort.InferenceSession(model_path)
    
    def embed_face(self, image_path: str) -> np.ndarray | None:
        """
        Extrait l'embedding 128D du visage principal dans l'image.
        Retourne None si aucun visage detecte.
        """
        # 1. Pre-processing : resize 112x112, normalize
        # 2. Inference ONNX
        # 3. Retourne le vecteur L2-normalise
        pass
    
    async def verify_photo_against_selfie(
        self, user_id, photo_path, db
    ) -> dict:
        """
        Compare une photo uploadee avec le selfie verifie du user.
        """
        selfie = await get_verified_selfie(user_id, db)
        if selfie is None:
            return {"status": "no_selfie", "action": "skip"}
        
        selfie_emb = self.embed_face(selfie.original_path)
        photo_emb = self.embed_face(photo_path)
        
        if selfie_emb is None:
            return {"status": "no_face_in_selfie", "action": "flag"}
        if photo_emb is None:
            return {"status": "no_face_in_photo", "action": "flag"}
        
        similarity = float(np.dot(selfie_emb, photo_emb))
        
        if similarity >= self.THRESHOLD_OK:
            return {"status": "match", "similarity": similarity}
        elif similarity >= self.THRESHOLD_WARNING:
            return {"status": "warning", "similarity": similarity,
                    "action": "log"}
        elif similarity >= self.THRESHOLD_MISMATCH:
            return {"status": "mismatch", "similarity": similarity,
                    "action": "flag_for_review"}
        else:
            return {"status": "clear_mismatch", "similarity": similarity,
                    "action": "flag_and_notify_admin"}
```

**Config ENV :** `FACE_VERIFICATION_ENABLED` (default false au MVP, true quand modele installe).
**Modele ONNX :** `FACE_VERIFICATION_MODEL_PATH` (default /models/arcface_r50.onnx).

### 10.3 Genre selfie vs genre profil

**Integre dans FaceVerificationService :**

```python
async def verify_gender_consistency(self, user_id, db) -> dict:
    """
    Compare le genre detecte sur le selfie avec le genre declare 
    dans le profil.
    
    InsightFace inclut un predicteur de genre (male/female + confidence).
    
    Si selfie_gender != profile_gender avec confidence > 0.85 :
    → flag pour review (pas auto-reject, les personnes non-binaires 
      ou trans peuvent legitimement avoir un mismatch).
    """
    selfie = await get_verified_selfie(user_id, db)
    profile = await get_profile(user_id, db)
    
    if not selfie or not profile:
        return {"status": "skip"}
    
    selfie_analysis = self.analyze_face(selfie.original_path)
    
    detected_gender = selfie_analysis["gender"]  # "male" | "female"
    confidence = selfie_analysis["gender_confidence"]  # 0-1
    declared_gender = profile.gender  # "man" | "woman" | "non_binary"
    
    # Mapping
    gender_map = {"man": "male", "woman": "female"}
    expected = gender_map.get(declared_gender)
    
    if expected is None:
        # Non-binaire : pas de verification de genre
        return {"status": "skip_non_binary"}
    
    if detected_gender != expected and confidence > 0.85:
        return {
            "status": "gender_mismatch",
            "detected": detected_gender,
            "declared": declared_gender,
            "confidence": confidence,
            "action": "flag_for_review",
            # PAS auto-reject : personnes trans legit
        }
    
    return {"status": "consistent"}
```

**IMPORTANT — Sensibilite ethique :**
- Ne PAS auto-reject sur un mismatch de genre
- Les personnes trans ou non-binaires peuvent legitimement avoir un 
  mismatch entre apparence et genre declare
- Toujours flag pour review HUMAINE, jamais decision automatique
- L'admin regarde le profil complet avant de decider

### 10.4 Invalidation selfie si changement genre

**Fichier :** modification dans `app/services/profile_service.py`

```python
async def update_profile(user, data, db):
    # ... code existant ...
    
    # Si le genre change APRES verification selfie
    if "gender" in data and data["gender"] != user.profile.gender:
        if user.is_selfie_verified:
            user.is_selfie_verified = False
            # Log pour audit
            log.warning(
                "selfie_invalidated_gender_change",
                user_id=str(user.id),
                old_gender=user.profile.gender,
                new_gender=data["gender"],
            )
            # Flag pour surveillance (changement de genre post-selfie 
            # est un pattern de catfish)
            await flag_for_review(
                user.id, 
                "gender_changed_after_selfie",
                db
            )
    
    # ... suite du code ...
```

**Consequence UX :** l'utilisateur voit "Ton selfie a ete invalide suite au changement de genre. Reprends un selfie pour verifier ton profil." Il doit refaire le flow selfie, ce qui empeche le scenario "photos de ma soeur + changement genre".

### 10.5 Diversite temporelle des photos

```python
def check_photo_temporal_diversity(photos: list) -> dict:
    """
    Verifie que les photos n'ont pas toutes ete prises le meme jour.
    
    Pattern suspect : 6 photos uploadees avec le meme EXIF date
    → possiblement telechargees d'un seul coup depuis un dossier 
    de photos volees.
    
    Pattern normal : photos prises sur differentes semaines/mois.
    """
    dates = []
    for photo in photos:
        exif_date = extract_exif_date(photo.original_path)
        if exif_date:
            dates.append(exif_date)
    
    if len(dates) < 3:
        return {"status": "insufficient_data", "risk": 0.0}
    
    unique_days = len(set(d.date() for d in dates))
    
    if unique_days == 1 and len(dates) >= 4:
        return {
            "status": "all_same_day",
            "risk": 0.3,
            "action": "log",
            "note": "4+ photos meme jour — possible mais surveiller"
        }
    
    if unique_days <= 2 and len(dates) >= 5:
        return {
            "status": "very_low_diversity",
            "risk": 0.4,
            "action": "flag_for_review"
        }
    
    return {"status": "diverse", "risk": 0.0}
```

### 10.6 Combinaison des signaux dans le flow de moderation

**Pipeline complet dans `moderate_photo` (Session 10) :**

```python
async def moderate_photo_pipeline(photo_id: str):
    """
    Pipeline de moderation photo integrant TOUS les checks.
    Ordre :
    1. EXIF authenticity (rapide, pas de ML)
    2. NSFW detection (ONNX, ~100ms)
    3. Face detection (ONNX, ~50ms)
    4. Face verification selfie ↔ photo (ONNX, ~200ms)
    5. Gender consistency (si premier check du user)
    6. Temporal diversity (si 3+ photos)
    
    Chaque check retourne un risk score. On combine :
    - Si un seul check > 0.7 → manual_review
    - Si somme des risks > 1.0 → manual_review
    - Si NSFW > 0.7 → rejected direct
    - Si face_mismatch < 0.3 → flag + notif admin
    - Sinon → approved
    """
    photo = await load_photo(photo_id)
    results = {}
    
    # 1. EXIF (toujours, gratuit)
    results["exif"] = check_photo_authenticity(photo.original_path)
    
    # 2. NSFW (si mode onnx ou external)
    if settings.photo_moderation_mode in ("onnx", "external"):
        results["nsfw"] = await check_nsfw(photo.medium_path)
        if results["nsfw"]["score"] > 0.7:
            await reject_photo(photo, "nsfw")
            return
    
    # 3. Face detection
    if settings.face_verification_enabled:
        results["face"] = await detect_face(photo.medium_path)
        if results["face"]["count"] == 0:
            results["face"]["action"] = "flag_no_face"
    
    # 4. Selfie comparison
    if settings.face_verification_enabled:
        results["selfie_match"] = await face_service.verify_photo_against_selfie(
            photo.user_id, photo.original_path, db
        )
    
    # 5. Gender consistency (premier check du user seulement)
    if settings.face_verification_enabled and not await already_gender_checked(photo.user_id):
        results["gender"] = await face_service.verify_gender_consistency(
            photo.user_id, db
        )
    
    # 6. Temporal diversity
    user_photos = await get_user_photos(photo.user_id, db)
    if len(user_photos) >= 3:
        results["temporal"] = check_photo_temporal_diversity(user_photos)
    
    # Decision
    decision = combine_moderation_results(results)
    await apply_moderation_decision(photo, decision)
```

---

## Protection contre les scenarios specifiques

### Scenario A — Photos de ma soeur + changement de genre

**Comment ca se passe :**
1. Homme cree un compte avec genre "femme"
2. Uploade les photos de sa soeur
3. Prend un selfie de sa soeur (ou de lui-meme)
4. Matche avec des gens en se faisant passer pour elle
5. Plus tard, change le genre en "homme"

**Protections en place :**
1. **Selfie live obligatoire** → il doit avoir acces physique a sa soeur pour le selfie (possible si elle vit avec lui)
2. **Liveness detection** (quand active) → doit etre un vrai visage 3D, pas une photo d'ecran
3. **Face comparison selfie ↔ photos** → si c'est lui qui prend le selfie mais uploade les photos de sa soeur → mismatch detecte
4. **Changement de genre invalide le selfie** → doit reprendre un selfie apres le changement → impossible de matcher entre-temps
5. **Si signale par un match** → scam detection + review admin

**Faille restante :** si la soeur est physiquement presente et accepte de prendre le selfie pour lui. Aucune technologie ne peut empecher ca. C'est un cas de complicite humaine — la seule protection est le signalement par les matchs + la communaute locale (les gens se connaissent a Lome).

### Scenario B — Photos AI-generated (Midjourney, Stable Diffusion)

**Comment ca se passe :**
1. Genere 6 photos d'une femme fictive avec Midjourney
2. Cree un profil avec ces photos
3. Le selfie est le probleme : comment faire ?

**Protections en place :**
1. **Selfie live** → il ne peut pas faire un selfie d'une personne qui n'existe pas
2. **Face comparison** → le selfie (vrai visage) ne matche pas les photos AI (visage fictif) → mismatch
3. **EXIF check** → photos AI n'ont pas d'EXIF = flag
4. **Resolution suspecte** → photos AI souvent en resolution carree parfaite

**Faille restante :** deepfake en temps reel sur le selfie (app qui projette un visage fictif sur la camera). Contre-mesure : liveness detection avancee (clignement, rotation de tete). A implementer quand la menace devient reelle.

### Scenario C — Scammer classique (photos volees Instagram)

**Comment ca se passe :**
1. Telecharge 6 photos d'une personne reelle depuis Instagram
2. Cree un profil
3. Selfie : prend sa propre photo (qui ne correspond pas)

**Protections en place :**
1. **Face comparison selfie ↔ photos** → son selfie ne matche pas les photos volees → flag
2. **EXIF check** → photos Instagram re-telechargees perdent souvent les EXIF = flag
3. **Temporal diversity** → toutes les photos Instagram d'une meme personne sont souvent prises a des dates proches

### Scenario D — Bot ChatGPT qui repond aux messages

**Comment ca se passe :**
1. Cree un vrai profil (photos reelles, selfie valide)
2. Utilise ChatGPT pour ecrire les messages a sa place
3. Matche normalement, mais les conversations sont "trop parfaites"

**Protections en place :**
1. **Comportement suspect** (post-lancement) : regularite des temps de reponse, absence de messages vocaux, pas de variation d'horaire
2. **Mass messaging detection** : meme structure de message a 5+ matchs
3. **Pas de protection technique au MVP** — c'est un cas limite (la personne est reelle, juste assistee par IA)

**Position philosophique Flaam :** si quelqu'un utilise ChatGPT pour ecrire un premier message mais est sincere dans ses intentions, c'est discutable mais pas catastrophique. Le vrai probleme c'est quand TOUT est faux (profil + photos + messages). Les protections ci-dessus couvrent ce cas.

### Scenario E — Recidiviste banni qui recree un compte

**Comment ca se passe :**
1. Banni pour harcelement
2. Achete une nouvelle SIM
3. Recree un compte

**Protections en place :**
1. **AccountHistory par device_fingerprint** → meme telephone = detecte immediatement
2. **AccountHistory par phone_hash** → meme numero = bloque
3. **Face embedding** (post-lancement) → compare le selfie avec les selfies des comptes bannis → detecte meme avec un nouveau numero et un nouveau telephone
4. **IP fingerprinting** (post-lancement) → meme IP recurrente = surveiller

---

## Configuration ENV pour Session 10

```bash
# ── Photo moderation mode (global) ──
PHOTO_MODERATION_MODE=manual   # manual | onnx | external | off

# ── EXIF authenticity check ──
EXIF_CHECK_ENABLED=true        # Toujours active, meme en mode manual

# ── Face verification (InsightFace/ArcFace) ──
FACE_VERIFICATION_ENABLED=false  # Active quand le modele est installe
FACE_VERIFICATION_MODEL_PATH=/models/arcface_r50.onnx
FACE_GENDER_CHECK_ENABLED=false  # Active avec FACE_VERIFICATION

# ── NSFW detection ──
NSFW_MODEL_PATH=/models/nsfw_detector.onnx
NSFW_THRESHOLD_REJECT=0.7
NSFW_THRESHOLD_REVIEW=0.4

# ── Face detection (YuNet) ──
FACE_DETECTION_MODEL_PATH=/models/yunet_face.onnx

# ── Scam detection ──
SCAM_AUTO_BAN_THRESHOLD=0.7
SCAM_FLAG_THRESHOLD=0.4

# ── Bot detection (post-lancement) ──
BOT_DETECTION_ENABLED=false
BOT_DETECTION_MIN_MESSAGES=50  # Minimum de messages avant analyse

# ── AI image detection (post-lancement) ──
AI_IMAGE_DETECTION_ENABLED=false
AI_IMAGE_DETECTION_MODEL_PATH=/models/dire_detector.onnx
```

---

## Modeles ONNX a telecharger

| Modele | Usage | Taille | Source |
|---|---|---|---|
| ArcFace ResNet50 | Face embeddings (selfie ↔ photos) | ~30 MB | InsightFace GitHub |
| YuNet | Face detection (au moins 1 visage) | ~2 MB | OpenCV contrib |
| NSFW Classifier | Detection contenu adulte | ~50 MB | GantMan/nsfw_model |
| DIRE (optionnel) | Detection images AI-generated | ~200 MB | DIRE GitHub |
| GenderAge (inclus InsightFace) | Genre + age apparent | inclus | InsightFace GitHub |

**Total pour Session 10 :** ~82 MB (ArcFace + YuNet + NSFW).
**Total avec AI detection :** ~282 MB.

**Installation :** copie dans un dossier `/models/` monte en volume Docker.

```dockerfile
# Dockerfile addition pour mode onnx
COPY models/ /models/
RUN pip install onnxruntime==1.18.0 opencv-python-headless==4.9.0.80
```

---

## Metriques de safety a surveiller (post-lancement)

| Metrique | Seuil sain | Alerte si |
|---|---|---|
| Taux de photos flaggees / total uploads | < 5% | > 15% |
| Taux de selfie ↔ photo mismatch | < 2% | > 8% |
| Taux de reports par user actif par mois | < 3% | > 10% |
| Taux de bans automatiques / inscriptions | < 1% | > 5% |
| Temps median de resolution review manuelle | < 24h | > 72h |
| Recidivistes detectes / total bans | > 80% | < 50% |

---

## Principes ethiques non-negociables

1. **Jamais auto-reject sur genre mismatch.** Les personnes trans et non-binaires sont bienvenues. Toujours review humaine.

2. **Jamais auto-reject sur absence d'EXIF.** Beaucoup de telephones low-cost africains ne generent pas d'EXIF propres. Flag, pas reject.

3. **La vie privee prime.** Les face embeddings sont stockes de maniere hashee, pas en clair. Supprimees avec le compte (RGPD).

4. **Pas de surveillance de masse.** Les analyses sont faites uniquement sur les photos uploadees par l'utilisateur, pas sur son activite hors-app.

5. **Transparence.** Si un profil est flag, l'utilisateur est informe : "Ton profil est en cours de verification. Ca prend habituellement 24h."

6. **Droit a l'erreur.** Un flag n'est pas un ban. L'admin regarde, decide, et peut debloquer a tout moment.

---

## Iteration de ce document

A revoir :
- **Tous les mois** pendant les 3 premiers mois post-lancement
- **A chaque incident** de securite (faux profil signale, scam detecte)
- **A chaque nouveau modele ML** integre (mettre a jour les seuils)

Les seuils (0.3, 0.5, 0.7 pour face similarity, 0.4, 0.7 pour NSFW) sont des points de depart. Ils seront affines avec des donnees reelles.