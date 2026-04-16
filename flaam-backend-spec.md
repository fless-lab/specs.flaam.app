# Flaam — Backend specification complète

> App de dating hyper-locale pour jeunes professionnels en Afrique de l'Ouest
> Stack : FastAPI · PostgreSQL + PostGIS · Redis · Celery + RabbitMQ · Docker Compose

---

## 1. Structure du projet

```
flaam-backend/
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile
├── .env.example
├── .env
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── versions/
│   └── script.py.mako
├── scripts/
│   ├── seed_cities.py              # Seed villes + quartiers
│   ├── seed_spots.py               # Import spots Google Places
│   ├── seed_quartier_proximity.py  # Générer le graphe de proximité
│   ├── seed_matching_config.py     # Peupler les poids par défaut
│   ├── create_superadmin.py
│   └── run_matching_batch.py       # Lancer le batch manuellement
├── tests/
│   ├── conftest.py
│   ├── factories/              # Factory Boy pour les fixtures
│   │   ├── user_factory.py
│   │   ├── profile_factory.py
│   │   ├── spot_factory.py
│   │   └── match_factory.py
│   ├── unit/
│   │   ├── test_matching_engine.py
│   │   ├── test_geo_score.py
│   │   ├── test_lifestyle_score.py
│   │   ├── test_behavior_multiplier.py
│   │   └── test_feed_generation.py
│   ├── integration/
│   │   ├── test_auth_flow.py
│   │   ├── test_profile_crud.py
│   │   ├── test_matching_flow.py
│   │   ├── test_chat_flow.py
│   │   └── test_payment_webhook.py
│   └── e2e/
│       └── test_full_user_journey.py
├── app/
│   ├── __init__.py
│   ├── main.py                 # Point d'entrée FastAPI
│   ├── config.py               # Settings Pydantic
│   ├── dependencies.py         # Dépendances injectées (DB, Redis, current_user)
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── security.py         # JWT, hashing, token validation
│   │   ├── permissions.py      # Vérification premium, admin, ownership
│   │   ├── exceptions.py       # Exceptions custom + handlers
│   │   ├── rate_limiter.py     # Rate limiting par Redis
│   │   ├── pagination.py       # Cursor-based pagination helper
│   │   └── constants.py        # Enums, constantes globales
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── session.py          # AsyncSession factory
│   │   ├── base.py             # Base declarative SQLAlchemy
│   │   └── redis.py            # Redis connection pool
│   │
│   ├── models/                 # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── profile.py
│   │   ├── photo.py
│   │   ├── quartier.py
│   │   ├── quartier_proximity.py
│   │   ├── city.py
│   │   ├── spot.py
│   │   ├── user_spot.py
│   │   ├── user_quartier.py
│   │   ├── match.py
│   │   ├── message.py
│   │   ├── event.py
│   │   ├── event_registration.py
│   │   ├── report.py
│   │   ├── block.py
│   │   ├── subscription.py
│   │   ├── notification_preference.py
│   │   ├── device.py
│   │   ├── contact_blacklist.py
│   │   ├── feed_cache.py
│   │   ├── matching_config.py
│   │   └── behavior_log.py
│   │
│   ├── schemas/                # Pydantic schemas (request/response)
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── user.py
│   │   ├── profile.py
│   │   ├── photo.py
│   │   ├── spot.py
│   │   ├── quartier.py
│   │   ├── match.py
│   │   ├── message.py
│   │   ├── event.py
│   │   ├── report.py
│   │   ├── feed.py
│   │   ├── notification.py
│   │   ├── subscription.py
│   │   └── common.py           # Pagination, error responses
│   │
│   ├── api/                    # Routes (controllers)
│   │   ├── __init__.py
│   │   ├── router.py           # Router principal qui agrège tout
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── profiles.py
│   │   │   ├── photos.py
│   │   │   ├── spots.py
│   │   │   ├── quartiers.py
│   │   │   ├── feed.py
│   │   │   ├── matches.py
│   │   │   ├── messages.py
│   │   │   ├── events.py
│   │   │   ├── reports.py
│   │   │   ├── notifications.py
│   │   │   ├── subscriptions.py
│   │   │   ├── safety.py
│   │   │   └── admin.py
│   │   └── ws/
│   │       └── chat.py         # WebSocket endpoint
│   │
│   ├── services/               # Business logic
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── profile_service.py
│   │   ├── photo_service.py
│   │   ├── spot_service.py
│   │   ├── config_service.py       # Config dynamique des poids (Redis + DB)
│   │   ├── matching_engine/
│   │   │   ├── __init__.py
│   │   │   ├── pipeline.py     # Orchestrateur L1 → L5
│   │   │   ├── hard_filters.py # L1
│   │   │   ├── geo_scorer.py   # L2
│   │   │   ├── lifestyle_scorer.py  # L3
│   │   │   ├── behavior_scorer.py   # L4
│   │   │   ├── corrections.py  # L5 (wildcards, boost, shuffle)
│   │   │   ├── feedback.py     # Boucle de feedback
│   │   │   └── weights.py      # Gestion des poids adaptatifs
│   │   ├── feed_service.py
│   │   ├── match_service.py
│   │   ├── chat_service.py
│   │   ├── event_service.py
│   │   ├── notification_service.py
│   │   ├── payment_service.py
│   │   ├── moderation_service.py
│   │   ├── safety_service.py
│   │   └── report_service.py
│   │
│   ├── tasks/                  # Celery tasks
│   │   ├── __init__.py
│   │   ├── celery_app.py       # Configuration Celery
│   │   ├── image_tasks.py      # Compression, modération, upload
│   │   ├── matching_tasks.py   # Batch nocturne, recalcul incrémental
│   │   ├── notification_tasks.py
│   │   ├── cleanup_tasks.py    # Expiration matchs, purge données
│   │   ├── moderation_tasks.py
│   │   └── analytics_tasks.py  # Métriques, compteurs
│   │
│   └── utils/
│       ├── __init__.py
│       ├── image_processing.py # Pillow resize, WebP, strip EXIF
│       ├── sms.py              # Client Termii / Twilio
│       ├── storage.py          # Client R2 (S3-compatible)
│       ├── push.py             # Client FCM / APNs
│       ├── payment.py          # Client Paystack / Flutterwave
│       └── geo.py              # Helpers géospatiaux
```

---

## 2. Configuration

### 2.1 Variables d'environnement (.env)

```bash
# ── App ──
APP_NAME=flaam
APP_ENV=production               # development | staging | production
APP_DEBUG=false
APP_HOST=0.0.0.0
APP_PORT=8000
APP_WORKERS=4                    # Uvicorn workers (= nb CPU)
API_V1_PREFIX=/api/v1
CORS_ORIGINS=https://flaam.app    # Comma-separated

# ── Sécurité ──
SECRET_KEY=<random-64-chars>     # Pour signer les JWT
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=30
JWT_ALGORITHM=HS256
OTP_LENGTH=6
OTP_EXPIRE_SECONDS=600           # 10 min
OTP_MAX_ATTEMPTS=3
OTP_COOLDOWN_SECONDS=60          # Entre deux envois

# ── Database ──
DATABASE_URL=postgresql+asyncpg://flaam:password@db:5432/flaam
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=10
DATABASE_ECHO=false              # true en dev pour voir les requêtes SQL

# ── Redis ──
REDIS_URL=redis://redis:6379/0
REDIS_FEED_DB=1                  # DB séparée pour les feeds
REDIS_CACHE_DB=2                 # DB séparée pour le cache

# ── RabbitMQ ──
RABBITMQ_URL=amqp://flaam:password@rabbitmq:5672/flaam
CELERY_BROKER_URL=${RABBITMQ_URL}
CELERY_RESULT_BACKEND=redis://redis:6379/3

# ── Storage (Cloudflare R2) ──
R2_ENDPOINT=https://<account-id>.r2.cloudflarestorage.com
R2_ACCESS_KEY_ID=<key>
R2_SECRET_ACCESS_KEY=<secret>
R2_BUCKET_PHOTOS=flaam-photos
R2_BUCKET_VOICE=flaam-voice
CDN_BASE_URL=https://cdn.flaam.app

# ── SMS (Termii) ──
TERMII_API_KEY=<key>
TERMII_SENDER_ID=Flaam
TERMII_BASE_URL=https://api.ng.termii.com

# ── SMS fallback (Twilio) ──
TWILIO_ACCOUNT_SID=<sid>
TWILIO_AUTH_TOKEN=<token>
TWILIO_PHONE_NUMBER=+1234567890

# ── Payment (Paystack) ──
PAYSTACK_SECRET_KEY=<key>
PAYSTACK_PUBLIC_KEY=<key>
PAYSTACK_WEBHOOK_SECRET=<secret>

# ── Payment fallback (Flutterwave) ──
FLUTTERWAVE_SECRET_KEY=<key>
FLUTTERWAVE_WEBHOOK_SECRET=<secret>

# ── Firebase (Push notifications) ──
FCM_SERVICE_ACCOUNT_JSON=/secrets/fcm-service-account.json

# ── Modération ──
NSFW_MODEL_PATH=/models/nsfw_detector.onnx
FACE_MODEL_PATH=/models/face_detector.onnx
LIVENESS_MODEL_PATH=/models/liveness.onnx

# ── Matching engine ──
MATCHING_BATCH_HOUR=3            # Heure locale du batch nocturne
MATCHING_FEED_SIZE=12            # Profils par feed quotidien
MATCHING_WILDCARD_COUNT=2        # Wildcards par feed
MATCHING_NEW_USER_BOOST_DAYS=10
MATCHING_MATCH_EXPIRE_DAYS=7
MATCHING_SKIP_COOLDOWN_DAYS=30
MATCHING_MIN_WEEKLY_VISIBILITY=15 # Apparitions minimum/semaine

# ── Rate limiting ──
RATE_LIMIT_DEFAULT=60            # req/min
RATE_LIMIT_PREMIUM=120
RATE_LIMIT_OTP=3                 # tentatives/10min
RATE_LIMIT_LIKES_FREE=5          # likes/jour gratuit
RATE_LIMIT_LIKES_PREMIUM=50      # likes/jour premium

# ── Sentry ──
SENTRY_DSN=https://<key>@sentry.io/<project>

# ── Google Places (seed only) ──
GOOGLE_PLACES_API_KEY=<key>
```

### 2.2 config.py (Pydantic Settings)

```python
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    # App
    app_name: str = "flaam"
    app_env: str = "development"
    app_debug: bool = False
    app_host: str = "0.0.0.0"
    app_port: int = 8000
    app_workers: int = 4
    api_v1_prefix: str = "/api/v1"
    cors_origins: str = "http://localhost:3000"

    # Security
    secret_key: str
    jwt_access_token_expire_minutes: int = 15
    jwt_refresh_token_expire_days: int = 30
    jwt_algorithm: str = "HS256"
    otp_length: int = 6
    otp_expire_seconds: int = 600
    otp_max_attempts: int = 3
    otp_cooldown_seconds: int = 60

    # Database
    database_url: str
    database_pool_size: int = 20
    database_max_overflow: int = 10
    database_echo: bool = False

    # Redis
    redis_url: str = "redis://localhost:6379/0"
    redis_feed_db: int = 1
    redis_cache_db: int = 2

    # RabbitMQ / Celery
    celery_broker_url: str
    celery_result_backend: str

    # R2 Storage
    r2_endpoint: str
    r2_access_key_id: str
    r2_secret_access_key: str
    r2_bucket_photos: str = "flaam-photos"
    r2_bucket_voice: str = "flaam-voice"
    cdn_base_url: str

    # SMS
    termii_api_key: str
    termii_sender_id: str = "Flaam"
    termii_base_url: str = "https://api.ng.termii.com"

    # Payment
    paystack_secret_key: str
    paystack_webhook_secret: str

    # FCM
    fcm_service_account_json: str

    # Matching
    matching_batch_hour: int = 3
    matching_feed_size: int = 12
    matching_wildcard_count: int = 2
    matching_new_user_boost_days: int = 10
    matching_match_expire_days: int = 7
    matching_skip_cooldown_days: int = 30
    matching_min_weekly_visibility: int = 15

    # Rate limiting
    rate_limit_default: int = 60
    rate_limit_premium: int = 120
    rate_limit_likes_free: int = 5
    rate_limit_likes_premium: int = 50

    # Sentry
    sentry_dsn: str = ""

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

---

## 3. Database — Modèles SQLAlchemy complets

### 3.1 Base commune

```python
# app/db/base.py
import uuid
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )


class UUIDMixin:
    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
    )
```

### 3.2 City

```python
# app/models/city.py
from sqlalchemy import String, Boolean, Integer
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class City(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "cities"

    name: Mapped[str] = mapped_column(String(100), nullable=False)
    # ex: "Lomé", "Abidjan", "Dakar", "Accra", "Lagos", "Cotonou"

    country_code: Mapped[str] = mapped_column(String(2), nullable=False)
    # ISO 3166-1 alpha-2 : "TG", "CI", "SN", "GH", "NG", "BJ"

    country_name: Mapped[str] = mapped_column(String(100), nullable=False)

    timezone: Mapped[str] = mapped_column(String(50), nullable=False)
    # ex: "Africa/Lome", "Africa/Abidjan"

    currency_code: Mapped[str] = mapped_column(String(3), nullable=False)
    # "XOF" (FCFA zone UEMOA), "GHS" (cedi), "NGN" (naira)

    premium_price_monthly: Mapped[int] = mapped_column(Integer, nullable=False)
    # En plus petite unité de la devise. Ex: 2000 pour 2000 FCFA

    premium_price_weekly: Mapped[int] = mapped_column(Integer, nullable=False)
    # Ex: 800 pour 800 FCFA

    min_weekly_visibility: Mapped[int] = mapped_column(Integer, default=15)
    # Garantie de visibilité adaptée à la taille du pool

    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    # Désactiver une ville sans supprimer les données

    # Relations
    quartiers = relationship("Quartier", back_populates="city", lazy="selectin")
    spots = relationship("Spot", back_populates="city", lazy="selectin")
```

### 3.3 Quartier

```python
# app/models/quartier.py
import uuid
from sqlalchemy import String, ForeignKey, Float
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Quartier(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "quartiers"

    name: Mapped[str] = mapped_column(String(100), nullable=False)
    # ex: "Bè", "Tokoin", "Djidjolé", "Adidogomé"

    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), nullable=False
    )

    # Coordonnées centrales (pour affichage sur la carte)
    latitude: Mapped[float] = mapped_column(Float, nullable=False)
    longitude: Mapped[float] = mapped_column(Float, nullable=False)

    # Relations
    city = relationship("City", back_populates="quartiers")
```

### 3.3b QuartierProximity (graphe de voisinage)

```python
# app/models/quartier_proximity.py
import uuid
from sqlalchemy import Float, ForeignKey, Index, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin


class QuartierProximity(Base, UUIDMixin):
    __tablename__ = "quartier_proximities"

    # Paire de quartiers (toujours dans la même ville)
    quartier_a_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("quartiers.id"), nullable=False
    )
    quartier_b_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("quartiers.id"), nullable=False
    )

    proximity_score: Mapped[float] = mapped_column(Float, nullable=False)
    # 0.0 → 1.0
    # 1.0 = même quartier (identité)
    # 0.7-0.9 = quartiers voisins directs (Bè ↔ Ablogamé)
    # 0.4-0.6 = quartiers proches (Bè ↔ Tokoin)
    # 0.1-0.3 = quartiers éloignés dans la même ville (Agoè ↔ Bè)
    # 0.0 = pas de lien (jamais le cas dans la même ville, on met minimum 0.05)
    #
    # Calcul : 1.0 - (distance_km / max_distance_ville_km)
    # Clampé à [0.05, 1.0]
    # Pré-calculé une fois par ville lors du seed, stocké en DB
    # Recalculé uniquement si on ajoute un nouveau quartier

    distance_km: Mapped[float] = mapped_column(Float, nullable=False)
    # Distance réelle en km entre les centres des deux quartiers
    # Stocké pour debug et recalcul. Haversine formula

    quartier_a = relationship("Quartier", foreign_keys=[quartier_a_id], lazy="selectin")
    quartier_b = relationship("Quartier", foreign_keys=[quartier_b_id], lazy="selectin")

    __table_args__ = (
        # Paire unique et ordonnée : a_id < b_id pour éviter les doublons
        Index("uq_quartier_proximity", "quartier_a_id", "quartier_b_id", unique=True),
        Index("ix_qp_quartier_a", "quartier_a_id"),
        Index("ix_qp_quartier_b", "quartier_b_id"),
        CheckConstraint("proximity_score >= 0 AND proximity_score <= 1",
                        name="ck_proximity_range"),
        CheckConstraint("quartier_a_id < quartier_b_id",
                        name="ck_quartier_order"),
    )
```

**Script de seed pour le graphe de proximité :**

```python
# scripts/seed_quartier_proximity.py
"""
Génère la table quartier_proximities pour une ville.
Pour chaque paire de quartiers, calcule la distance Haversine
entre leurs centres et convertit en proximity_score.

Usage : python scripts/seed_quartier_proximity.py --city-id <uuid>

Formule :
  distance_km = haversine(lat_a, lng_a, lat_b, lng_b)
  max_dist = max distance entre deux quartiers de la ville
  proximity_score = max(0.05, 1.0 - (distance_km / max_dist))
"""
import math

def haversine(lat1, lon1, lat2, lon2) -> float:
    """Distance en km entre deux points GPS."""
    R = 6371  # Rayon de la Terre en km
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = (math.sin(dlat / 2) ** 2
         + math.cos(math.radians(lat1))
         * math.cos(math.radians(lat2))
         * math.sin(dlon / 2) ** 2)
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))


async def seed_proximity_for_city(city_id: str, db_session):
    # 1. Charger tous les quartiers de la ville
    quartiers = await get_quartiers_by_city(city_id, db_session)

    # 2. Calculer toutes les distances paire par paire
    pairs = []
    for i, qa in enumerate(quartiers):
        for qb in quartiers[i + 1:]:
            dist = haversine(qa.latitude, qa.longitude, qb.latitude, qb.longitude)
            pairs.append((qa.id, qb.id, dist))

    # 3. Trouver la distance max de la ville
    max_dist = max(d for _, _, d in pairs) if pairs else 1.0

    # 4. Convertir en proximity_score et insérer
    for qa_id, qb_id, dist in pairs:
        # Ordonner les IDs (contrainte ck_quartier_order)
        a_id, b_id = sorted([qa_id, qb_id])
        score = max(0.05, 1.0 - (dist / max_dist))

        await upsert_proximity(a_id, b_id, score, dist, db_session)

    # Exemple pour Lomé (distances approximatives) :
    # Bè ↔ Tokoin        : 2.1 km → score 0.82
    # Bè ↔ Djidjolé      : 3.5 km → score 0.70
    # Bè ↔ Agbalépédo    : 4.0 km → score 0.66
    # Bè ↔ Agoè          : 9.2 km → score 0.21
    # Tokoin ↔ Agoè      : 7.8 km → score 0.33
    # Agoè ↔ Adidogomé   : 2.5 km → score 0.79
```

### 3.4 User

```python
# app/models/user.py
import uuid
from datetime import datetime
from sqlalchemy import String, Boolean, DateTime, ForeignKey, Index, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class User(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "users"

    # ── Authentification ──
    phone_hash: Mapped[str] = mapped_column(
        String(128), unique=True, nullable=False, index=True
    )
    # SHA-256 du numéro normalisé (+228XXXXXXXX)
    # On ne stocke JAMAIS le numéro en clair

    phone_country_code: Mapped[str] = mapped_column(String(5), nullable=False)
    # "+228", "+225", "+221"... pour le routing SMS

    is_phone_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    is_selfie_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    is_id_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    # 3 niveaux : téléphone → selfie (obligatoire) → pièce d'identité (optionnel)

    # ── Statut ──
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    # False = compte suspendu par la modération

    is_visible: Mapped[bool] = mapped_column(Boolean, default=True)
    # False = mode pause (profil masqué, conversations actives)

    is_premium: Mapped[bool] = mapped_column(Boolean, default=False)

    is_banned: Mapped[bool] = mapped_column(Boolean, default=False)
    ban_reason: Mapped[str | None] = mapped_column(String(500), nullable=True)

    # ── Localisation ──
    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), nullable=False
    )

    # ── Activité ──
    last_active_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    # Mis à jour à chaque requête API authentifiée

    last_feed_generated_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    # ── Langue ──
    language: Mapped[str] = mapped_column(String(5), default="fr")
    # "fr" ou "en"

    # ── Anti-gaming ──
    account_created_count: Mapped[int] = mapped_column(default=1)
    # Incrémenté si même phone_hash revient après suppression
    # Si > 1, pas de boost nouveau profil

    # ── Relations ──
    city = relationship("City", lazy="selectin")
    profile = relationship("Profile", back_populates="user", uselist=False, lazy="selectin")
    photos = relationship("Photo", back_populates="user", lazy="selectin",
                          order_by="Photo.display_order")
    devices = relationship("Device", back_populates="user", lazy="selectin")
    user_quartiers = relationship("UserQuartier", back_populates="user", lazy="selectin")
    user_spots = relationship("UserSpot", back_populates="user", lazy="selectin")
    subscription = relationship("Subscription", back_populates="user",
                                uselist=False, lazy="selectin")
    notification_prefs = relationship("NotificationPreference", back_populates="user",
                                      uselist=False, lazy="selectin")

    __table_args__ = (
        Index("ix_users_city_active", "city_id", "last_active_at"),
        Index("ix_users_city_visible", "city_id", "is_visible", "is_active"),
    )
```

### 3.5 Device

```python
# app/models/device.py
import uuid
from datetime import datetime
from sqlalchemy import String, DateTime, ForeignKey, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Device(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "devices"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    device_fingerprint: Mapped[str] = mapped_column(String(256), nullable=False)
    # Hash de : Android ID + modèle + OS version
    # Pour détecter les comptes multiples sur le même device

    platform: Mapped[str] = mapped_column(String(10), nullable=False)
    # "android" | "ios"

    fcm_token: Mapped[str | None] = mapped_column(String(512), nullable=True)
    # Token Firebase Cloud Messaging pour les push notifications
    # Mis à jour à chaque login

    app_version: Mapped[str | None] = mapped_column(String(20), nullable=True)
    # "1.0.0", "1.2.3" — pour cibler les notifications par version

    os_version: Mapped[str | None] = mapped_column(String(20), nullable=True)

    last_login_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    user = relationship("User", back_populates="devices")
```

### 3.6 Profile

```python
# app/models/profile.py
import uuid
from datetime import date
from sqlalchemy import String, Date, Float, Integer, ForeignKey, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Profile(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "profiles"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"),
        unique=True, nullable=False
    )

    # ── Identité ──
    display_name: Mapped[str] = mapped_column(String(50), nullable=False)
    # Prénom uniquement, jamais de nom de famille

    birth_date: Mapped[date] = mapped_column(Date, nullable=False)
    # 18+ vérifié côté application

    gender: Mapped[str] = mapped_column(String(20), nullable=False)
    # "man" | "woman" | "non_binary"

    seeking_gender: Mapped[str] = mapped_column(String(20), nullable=False)
    # "men" | "women" | "everyone"

    # ── Mode de vie ──
    intention: Mapped[str] = mapped_column(String(30), nullable=False)
    # "serious" | "getting_to_know" | "friendship_first" | "open"

    sector: Mapped[str] = mapped_column(String(30), nullable=False)
    # "tech" | "finance" | "health" | "education" | "commerce" |
    # "creative" | "public_admin" | "student" | "other"

    rhythm: Mapped[str | None] = mapped_column(String(20), nullable=True)
    # "early_bird" | "night_owl" | null (non renseigné)

    # ── Prompts (3 max) ──
    prompts: Mapped[dict] = mapped_column(JSONB, default=list)
    # Format: [
    #   {"question": "Un dimanche parfait c'est...", "answer": "Grasse mat + brunch + plage"},
    #   {"question": "Mon maquis préféré parce que...", "answer": "Le poulet braisé de chez Tonton"},
    #   {"question": "Je cherche quelqu'un qui...", "answer": "Sait rire de tout"}
    # ]
    # Questions choisies parmi une liste prédéfinie (voir constants.py)
    # Max 3 prompts, answer max 200 caractères

    # ── Tags lifestyle (max 8) ──
    tags: Mapped[list] = mapped_column(JSONB, default=list)
    # ["foodie", "sport", "tech", "travel", "music", "reading",
    #  "faith_based", "cinema", "brunch_addict", "yoga", "gaming",
    #  "entrepreneurship", "art", "dancing", "cooking", "photography"]
    # Max 8 tags sélectionnés

    # ── Langues parlées ──
    languages: Mapped[list] = mapped_column(JSONB, default=list)
    # ["fr", "en", "ewe", "mina", "kabiye", "wolof", "twi", "yoruba", "fon", ...]
    # Tags libres, pas de liste fermée

    # ── Préférences de découverte ──
    seeking_age_min: Mapped[int] = mapped_column(Integer, default=18)
    seeking_age_max: Mapped[int] = mapped_column(Integer, default=40)

    # ── Scores calculés (mis à jour par le matching engine) ──
    profile_completeness: Mapped[float] = mapped_column(Float, default=0.0)
    # 0.0 → 1.0. Calculé à chaque update de profil
    # Formule :
    #   has_3_photos * 0.30
    # + has_prompts * 0.20
    # + has_tags * 0.15
    # + has_spots * 0.15
    # + has_verified_selfie * 0.10
    # + has_bio_complete * 0.10

    behavior_multiplier: Mapped[float] = mapped_column(Float, default=1.0)
    # 0.6 → 1.4. Mis à jour en real-time par le matching engine (via Redis)
    # Persisté en DB toutes les heures par un cron

    # ── Relations ──
    user = relationship("User", back_populates="profile")

    __table_args__ = (
        CheckConstraint("seeking_age_min >= 18", name="ck_seeking_age_min"),
        CheckConstraint("seeking_age_max >= seeking_age_min", name="ck_seeking_age_range"),
    )
```

### 3.7 Photo

```python
# app/models/photo.py
import uuid
from sqlalchemy import String, Integer, Boolean, ForeignKey, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Photo(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "photos"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    # ── URLs (générées après upload vers R2) ──
    original_url: Mapped[str] = mapped_column(String(500), nullable=False)
    # https://cdn.flaam.app/photos/{user_id}/{photo_id}_original.webp
    # Max 1200px côté le plus long

    thumbnail_url: Mapped[str] = mapped_column(String(500), nullable=False)
    # https://cdn.flaam.app/photos/{user_id}/{photo_id}_thumb.webp
    # 150x150px carré, crop centre

    medium_url: Mapped[str] = mapped_column(String(500), nullable=False)
    # https://cdn.flaam.app/photos/{user_id}/{photo_id}_medium.webp
    # 600px côté le plus long

    # ── Métadonnées ──
    display_order: Mapped[int] = mapped_column(Integer, nullable=False)
    # 0 = photo principale, 1-5 = secondaires

    is_verified_selfie: Mapped[bool] = mapped_column(Boolean, default=False)
    # True si c'est le selfie de vérification

    content_hash: Mapped[str] = mapped_column(String(64), nullable=False)
    # SHA-256 de l'image originale
    # Pour détecter les photos dupliquées entre profils

    width: Mapped[int] = mapped_column(Integer, nullable=False)
    height: Mapped[int] = mapped_column(Integer, nullable=False)

    file_size_bytes: Mapped[int] = mapped_column(Integer, nullable=False)
    # Taille du fichier compressé WebP

    # ── Modération ──
    moderation_status: Mapped[str] = mapped_column(String(20), default="pending")
    # "pending" | "approved" | "rejected" | "manual_review"

    moderation_score: Mapped[float | None] = mapped_column(nullable=True)
    # Score NSFW 0.0 → 1.0. Si > 0.7 → rejected. Si 0.4-0.7 → manual_review

    rejection_reason: Mapped[str | None] = mapped_column(String(200), nullable=True)
    # "nsfw" | "not_a_person" | "group_photo_only" | "copyrighted" | "other"

    user = relationship("User", back_populates="photos")

    __table_args__ = (
        CheckConstraint("display_order >= 0 AND display_order <= 5",
                        name="ck_photo_order"),
    )
```

### 3.8 Spot

```python
# app/models/spot.py
import uuid
from geoalchemy2 import Geometry
from sqlalchemy import String, Integer, Boolean, Float, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Spot(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "spots"

    name: Mapped[str] = mapped_column(String(200), nullable=False)
    # "Café 21", "Maquis Chez Tonton", "Salle Olympe"

    category: Mapped[str] = mapped_column(String(30), nullable=False)
    # "cafe" | "restaurant" | "gym" | "coworking" | "bar" |
    # "worship" | "market" | "beach" | "park" | "other"

    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), nullable=False
    )

    # ── Géolocalisation ──
    location = mapped_column(
        Geometry(geometry_type="POINT", srid=4326), nullable=False
    )
    # Stocké en SRID 4326 (WGS84, lat/lng standard)

    latitude: Mapped[float] = mapped_column(Float, nullable=False)
    longitude: Mapped[float] = mapped_column(Float, nullable=False)
    # Dupliqué en colonnes simples pour les lectures sans PostGIS

    address: Mapped[str | None] = mapped_column(String(300), nullable=True)

    # ── Métadonnées ──
    google_place_id: Mapped[str | None] = mapped_column(String(200), nullable=True)
    # Référence Google Places si importé. Null si créé par un utilisateur

    total_checkins: Mapped[int] = mapped_column(Integer, default=0)
    # Compteur global (tous utilisateurs). Pour le tri "populaire"

    total_users: Mapped[int] = mapped_column(Integer, default=0)
    # Nombre d'utilisateurs distincts qui ont ce spot

    is_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    # True si validé par la modération (import Google Places = auto-vérifié)

    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    created_by_user_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), nullable=True
    )
    # Null si importé. User ID si proposé par un utilisateur

    # ── Pondération matching ──
    social_weight: Mapped[float] = mapped_column(Float, default=1.0)
    # Poids dans le matching. Maquis/bar = 1.5 (contexte social fort).
    # Coworking = 0.8 (contexte fonctionnel). Configurable par catégorie.
    # Valeurs par défaut :
    # cafe=1.2, restaurant=1.5, gym=1.0, coworking=0.8, bar=1.5,
    # worship=1.0, market=0.7, beach=1.3, park=1.1, other=0.8

    city = relationship("City", back_populates="spots")

    __table_args__ = (
        Index("ix_spots_location", "location", postgresql_using="gist"),
        Index("ix_spots_city_category", "city_id", "category"),
        Index("ix_spots_city_active", "city_id", "is_active"),
    )
```

### 3.9 UserSpot (relation user ↔ spot)

```python
# app/models/user_spot.py
import uuid
from datetime import datetime
from sqlalchemy import Integer, Float, String, DateTime, ForeignKey, func, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class UserSpot(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "user_spots"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    spot_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("spots.id", ondelete="CASCADE"), nullable=False
    )

    # ── Check-ins ──
    checkin_count: Mapped[int] = mapped_column(Integer, default=0)
    last_checkin_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    first_checkin_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    # ── Fidélité (calculée) ──
    fidelity_level: Mapped[str] = mapped_column(String(20), default="declared")
    # "declared"  → 0 check-ins (ajouté manuellement)
    # "confirmed" → 1-2 check-ins
    # "regular"   → 3-5 check-ins
    # "regular_plus" → 6+ check-ins (= "habitué" côté UI)

    fidelity_score: Mapped[float] = mapped_column(Float, default=0.5)
    # Score continu 0.0 → 1.0
    # declared=0.5, confirmed=0.7, regular=0.85, regular_plus=1.0
    # Affecté aussi par la fraîcheur du dernier check-in

    # ── Visibilité ──
    is_visible: Mapped[bool] = mapped_column(default=True)
    # L'utilisateur peut masquer un spot de son profil public
    # Il reste pris en compte pour le matching mais n'est pas affiché

    is_active_in_matching: Mapped[bool] = mapped_column(default=True)
    # Gel doux lors de l'expiration du premium.
    # Quand premium expire : les spots au-delà de la limite free (5)
    # passent à False. Données conservées, juste ignorées par le matching.
    # Réactivation instantanée quand premium se réactive.

    user = relationship("User", back_populates="user_spots")
    spot = relationship("Spot", lazy="selectin")

    __table_args__ = (
        Index("ix_user_spots_user", "user_id"),
        Index("ix_user_spots_spot", "spot_id"),
        Index("uq_user_spot", "user_id", "spot_id", unique=True),
    )
```

### 3.10 UserQuartier

```python
# app/models/user_quartier.py
import uuid
from sqlalchemy import String, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class UserQuartier(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "user_quartiers"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    quartier_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("quartiers.id"), nullable=False
    )

    relation_type: Mapped[str] = mapped_column(String(15), nullable=False)
    # "lives"      → habite dans ce quartier (poids 2.0x dans le matching)
    # "works"      → travaille dans ce quartier (poids 1.5x)
    # "hangs"      → sort dans ce quartier (poids 1.0x)
    # "interested" → veut rencontrer des gens de ce quartier (poids 0.8x)
    #
    # "interested" résout le cas : "Je vis à Agoè mais je préfère
    # rencontrer des gens de Bè." Pas de lien géographique logique,
    # c'est une pure préférence personnelle.
    #
    # Différences avec les autres types :
    # - N'apparaît PAS sur le profil public (invisible pour les matchs)
    # - Agit uniquement comme paramètre de découverte (comme seeking_age)
    # - Limité à 3 en free, 6 en premium (évite que les gens cochent toute la ville)
    # - N'affecte PAS le score géo affiché ("82% zones communes")
    #   mais booste le ranking interne dans le feed

    is_primary: Mapped[bool] = mapped_column(default=False)
    # True si c'est le quartier principal (1 seul "lives" peut être primary)
    # Utilisé pour l'affichage "Ama · Tokoin" sur le profil résumé

    is_active_in_matching: Mapped[bool] = mapped_column(default=True)
    # Gel doux lors de l'expiration du premium.
    # Quand premium expire : les quartiers au-delà de la limite free (3 physiques,
    # 3 interested) passent à False. Ils restent en DB (pas de suppression de données),
    # mais sont ignorés par les scorers L1/L2 du matching.
    # Quand premium se réactive : tout repasse à True instantanément.
    # L'utilisateur peut aussi choisir lesquels garder actifs via l'app.

    user = relationship("User", back_populates="user_quartiers")
    quartier = relationship("Quartier", lazy="selectin")

    __table_args__ = (
        Index("uq_user_quartier", "user_id", "quartier_id", "relation_type", unique=True),
        Index("ix_user_quartiers_quartier", "quartier_id"),
        Index("ix_user_quartiers_user_type", "user_id", "relation_type"),
    )
```

### 3.11 Match

```python
# app/models/match.py
import uuid
from datetime import datetime
from sqlalchemy import String, DateTime, ForeignKey, Index, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Match(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "matches"

    # ── Les deux utilisateurs ──
    user_a_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    user_b_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    # Convention : user_a est celui qui a liké en premier
    # user_b est celui qui a liké en retour (complétant le match)

    # ── Statut ──
    status: Mapped[str] = mapped_column(String(20), default="pending")
    # "pending"    → user_a a liké, en attente du retour de user_b
    # "matched"    → like mutuel, conversation ouverte
    # "expired"    → 7 jours sans message après match
    # "unmatched"  → l'un des deux a unmatch
    # "reported"   → match fermé suite à un signalement

    # ── Engagement ──
    liked_prompt_id: Mapped[str | None] = mapped_column(String(50), nullable=True)
    # Si user_a a liké un prompt spécifique, stocke le prompt question
    # Utilisé pour générer l'ice-breaker

    # ── Dates ──
    matched_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    # Timestamp du like mutuel

    expires_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    # matched_at + 7 jours. Remis à null dès qu'un message est envoyé

    last_message_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    unmatched_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    unmatched_by: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), nullable=True
    )

    # ── Métadonnées matching ──
    geo_score: Mapped[float | None] = mapped_column(nullable=True)
    # Score géo au moment du match (pour analytics)

    lifestyle_score: Mapped[float | None] = mapped_column(nullable=True)

    was_wildcard: Mapped[bool] = mapped_column(default=False)
    # True si ce profil a été proposé comme wildcard

    # ── Relations ──
    user_a = relationship("User", foreign_keys=[user_a_id], lazy="selectin")
    user_b = relationship("User", foreign_keys=[user_b_id], lazy="selectin")
    messages = relationship("Message", back_populates="match",
                            order_by="Message.created_at", lazy="selectin")

    __table_args__ = (
        Index("ix_matches_user_a", "user_a_id", "status"),
        Index("ix_matches_user_b", "user_b_id", "status"),
        Index("ix_matches_expires", "expires_at",
              postgresql_where="status = 'matched' AND expires_at IS NOT NULL"),
        Index("uq_match_pair", "user_a_id", "user_b_id", unique=True),
    )
```

### 3.12 Message

```python
# app/models/message.py
import uuid
from sqlalchemy import String, Integer, Boolean, ForeignKey, Index, Text
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Message(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "messages"

    match_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("matches.id", ondelete="CASCADE"), nullable=False
    )
    sender_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    # ── Contenu ──
    message_type: Mapped[str] = mapped_column(String(20), nullable=False)
    # "text"     → message texte classique
    # "voice"    → message vocal
    # "photo"    → photo partagée dans le chat
    # "meetup"   → message structuré "On se retrouve ?"
    # "system"   → message système (ice-breaker, expiration warning)

    content: Mapped[str | None] = mapped_column(Text, nullable=True)
    # Texte du message. Null pour les vocaux et photos

    media_url: Mapped[str | None] = mapped_column(String(500), nullable=True)
    # URL R2 pour vocaux et photos

    media_duration_seconds: Mapped[int | None] = mapped_column(Integer, nullable=True)
    # Durée du vocal en secondes (max 60)

    # ── Meetup (message structuré) ──
    meetup_data: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    # Format pour message_type="meetup" :
    # {
    #   "spot_id": "uuid",
    #   "spot_name": "Café 21",
    #   "proposed_time": "2026-04-15T18:00:00+00:00",
    #   "status": "proposed" | "accepted" | "modified" | "declined",
    #   "modified_time": null | "2026-04-15T19:00:00+00:00"
    # }

    # ── Statut ──
    is_read: Mapped[bool] = mapped_column(Boolean, default=False)
    read_at: Mapped[str | None] = mapped_column(nullable=True)

    is_flagged: Mapped[bool] = mapped_column(Boolean, default=False)
    # Flaggé par le filtre automatique. Envoyé en modération

    flag_reason: Mapped[str | None] = mapped_column(String(50), nullable=True)
    # "inappropriate" | "spam" | "scam_link" | "money_request"

    match = relationship("Match", back_populates="messages")

    __table_args__ = (
        Index("ix_messages_match", "match_id", "created_at"),
        Index("ix_messages_sender", "sender_id"),
    )
```

### 3.13 Event

```python
# app/models/event.py
import uuid
from datetime import datetime
from sqlalchemy import String, Integer, Boolean, DateTime, ForeignKey, Text, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Event(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "events"

    title: Mapped[str] = mapped_column(String(200), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)

    spot_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("spots.id"), nullable=False
    )
    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), nullable=False
    )

    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    ends_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    category: Mapped[str] = mapped_column(String(30), nullable=False)
    # "afterwork" | "sport" | "brunch" | "meetup" | "expo" | "party" | "other"

    max_attendees: Mapped[int | None] = mapped_column(Integer, nullable=True)
    current_attendees: Mapped[int] = mapped_column(Integer, default=0)

    # ── Création ──
    created_by_user_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), nullable=True
    )
    is_admin_created: Mapped[bool] = mapped_column(Boolean, default=False)
    is_sponsored: Mapped[bool] = mapped_column(Boolean, default=False)
    sponsor_name: Mapped[str | None] = mapped_column(String(100), nullable=True)

    # ── Modération ──
    is_approved: Mapped[bool] = mapped_column(Boolean, default=False)
    # Events créés par les users doivent être approuvés
    # Events admin = auto-approved

    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    spot = relationship("Spot", lazy="selectin")
    registrations = relationship("EventRegistration", back_populates="event", lazy="selectin")

    __table_args__ = (
        Index("ix_events_city_date", "city_id", "starts_at"),
        Index("ix_events_active", "is_active", "is_approved", "starts_at"),
    )
```

### 3.14 EventRegistration

```python
# app/models/event_registration.py
import uuid
from sqlalchemy import ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class EventRegistration(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "event_registrations"

    event_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("events.id", ondelete="CASCADE"), nullable=False
    )
    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    event = relationship("Event", back_populates="registrations")

    __table_args__ = (
        Index("uq_event_registration", "event_id", "user_id", unique=True),
    )
```

### 3.15 Report

```python
# app/models/report.py
import uuid
from sqlalchemy import String, Text, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Report(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "reports"

    reporter_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    reported_user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    reason: Mapped[str] = mapped_column(String(30), nullable=False)
    # "fake_profile" | "inappropriate_behavior" | "harassment" |
    # "scam" | "underage" | "other"

    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    # Détail optionnel fourni par le reporter

    evidence_message_ids: Mapped[list | None] = mapped_column(nullable=True)
    # IDs des messages signalés (si applicable)

    # ── Traitement ──
    status: Mapped[str] = mapped_column(String(20), default="pending")
    # "pending" | "reviewing" | "resolved_action" | "resolved_no_action" | "dismissed"

    resolution_note: Mapped[str | None] = mapped_column(Text, nullable=True)
    resolved_by: Mapped[str | None] = mapped_column(String(100), nullable=True)
    # Email ou ID de l'admin qui a traité

    reporter = relationship("User", foreign_keys=[reporter_id])
    reported_user = relationship("User", foreign_keys=[reported_user_id])

    __table_args__ = (
        Index("ix_reports_status", "status"),
        Index("ix_reports_reported", "reported_user_id"),
    )
```

### 3.16 Block

```python
# app/models/block.py
import uuid
from sqlalchemy import ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base, UUIDMixin, TimestampMixin


class Block(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "blocks"

    blocker_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )
    blocked_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    __table_args__ = (
        Index("uq_block", "blocker_id", "blocked_id", unique=True),
        Index("ix_blocks_blocker", "blocker_id"),
        Index("ix_blocks_blocked", "blocked_id"),
    )
```

### 3.17 ContactBlacklist

```python
# app/models/contact_blacklist.py
import uuid
from sqlalchemy import String, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base, UUIDMixin, TimestampMixin


class ContactBlacklist(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "contact_blacklists"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    phone_hash: Mapped[str] = mapped_column(String(128), nullable=False)
    # Hash du numéro du contact à ne jamais montrer
    # Importé depuis le carnet de contacts du téléphone

    __table_args__ = (
        Index("uq_contact_blacklist", "user_id", "phone_hash", unique=True),
        Index("ix_contact_blacklist_phone", "phone_hash"),
    )
```

### 3.18 Subscription

```python
# app/models/subscription.py
import uuid
from datetime import datetime
from sqlalchemy import String, Integer, DateTime, ForeignKey, Boolean
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Subscription(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "subscriptions"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"),
        unique=True, nullable=False
    )

    plan: Mapped[str] = mapped_column(String(20), nullable=False)
    # "weekly" | "monthly"

    provider: Mapped[str] = mapped_column(String(20), nullable=False)
    # "paystack" | "flutterwave"

    provider_subscription_id: Mapped[str | None] = mapped_column(
        String(200), nullable=True
    )
    # Référence chez le provider pour gérer le renouvellement

    provider_customer_id: Mapped[str | None] = mapped_column(
        String(200), nullable=True
    )

    payment_method: Mapped[str] = mapped_column(String(30), nullable=False)
    # "momo" | "tmoney" | "wave" | "orange_money" | "airtel_money" | "card"

    amount: Mapped[int] = mapped_column(Integer, nullable=False)
    # En plus petite unité de la devise

    currency: Mapped[str] = mapped_column(String(3), nullable=False)
    # "XOF", "GHS", "NGN"

    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)

    is_auto_renew: Mapped[bool] = mapped_column(Boolean, default=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    cancelled_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    user = relationship("User", back_populates="subscription")
```

### 3.18b Payment

```python
# app/models/payment.py
import uuid
from datetime import datetime
from sqlalchemy import String, Integer, ForeignKey, DateTime, JSON, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class Payment(Base, UUIDMixin, TimestampMixin):
    """
    Tracking de chaque tentative de paiement (succès ou échec).
    Une Subscription peut avoir plusieurs Payments dans son historique
    (paiement initial, renouvellements, retries après échec).

    State machine : initialized → pending → success | failed | timeout
    Voir section 36 pour les détails de la state machine.
    """
    __tablename__ = "payments"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), nullable=False, index=True
    )
    subscription_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), ForeignKey("subscriptions.id"), nullable=True
    )

    amount: Mapped[int] = mapped_column(Integer, nullable=False)
    # En plus petite unité de la devise (centimes pour XOF)

    currency: Mapped[str] = mapped_column(String(3), default="XOF", nullable=False)

    provider: Mapped[str] = mapped_column(String(20), nullable=False)
    # "paystack" | "mtn_momo" | "moov_money" | "flooz" | "tmoney" | "wave"

    provider_reference: Mapped[str] = mapped_column(
        String(100), unique=True, nullable=False, index=True
    )
    # Référence unique chez le provider (transaction_id Paystack, etc.)
    # Sert aussi de clé pour matcher les webhooks entrants.

    status: Mapped[str] = mapped_column(
        String(20), default="initialized", nullable=False, index=True
    )
    # "initialized" | "pending" | "success" | "failed" | "timeout" | "refunded"

    idempotency_key: Mapped[str | None] = mapped_column(
        String(64), unique=True, nullable=True
    )
    # Pour éviter les doubles paiements en cas de retry réseau côté mobile.
    # Le client génère un UUID v4 et le passe dans le header Idempotency-Key.

    initialized_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    completed_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    # Set quand status passe à success/failed/timeout/refunded

    webhook_payload: Mapped[dict | None] = mapped_column(JSON, nullable=True)
    # Payload brut du webhook provider, gardé pour audit et debug

    failure_reason: Mapped[str | None] = mapped_column(String(200), nullable=True)
    # "insufficient_funds" | "user_cancelled" | "timeout" | "invalid_pin" | etc.

    retry_count: Mapped[int] = mapped_column(Integer, default=0, nullable=False)
    # Nombre de tentatives de polling du status auprès du provider

    user = relationship("User")
    subscription = relationship("Subscription", backref="payments")
```

### 3.19 NotificationPreference

```python
# app/models/notification_preference.py
import uuid
from sqlalchemy import Boolean, String, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base, UUIDMixin, TimestampMixin


class NotificationPreference(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "notification_preferences"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"),
        unique=True, nullable=False
    )

    # ── Catégories ──
    new_match: Mapped[bool] = mapped_column(Boolean, default=True)
    new_message: Mapped[bool] = mapped_column(Boolean, default=True)
    daily_feed: Mapped[bool] = mapped_column(Boolean, default=True)
    events: Mapped[bool] = mapped_column(Boolean, default=True)
    date_reminder: Mapped[bool] = mapped_column(Boolean, default=True)
    weekly_digest: Mapped[bool] = mapped_column(Boolean, default=True)

    # ── Heure du feed quotidien ──
    daily_feed_hour: Mapped[int] = mapped_column(default=9)
    # Heure locale (0-23). Default 9h du matin

    # ── Mode silencieux ──
    quiet_start_hour: Mapped[int] = mapped_column(default=23)
    quiet_end_hour: Mapped[int] = mapped_column(default=7)
    # Pas de push entre 23h et 7h sauf messages directs

    user = relationship("User", back_populates="notification_prefs")
```

### 3.20 BehaviorLog

```python
# app/models/behavior_log.py
import uuid
from sqlalchemy import String, Float, Integer, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base, UUIDMixin, TimestampMixin


class BehaviorLog(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "behavior_logs"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    event_type: Mapped[str] = mapped_column(String(30), nullable=False)
    # "profile_view"      → a vu un profil
    # "profile_scroll"    → a scrollé jusqu'en bas du profil
    # "like"              → a liké
    # "skip"              → a passé
    # "skip_with_reason"  → a passé avec raison
    # "match"             → match mutuel
    # "message_sent"      → a envoyé un message
    # "message_received"  → a reçu un message
    # "meetup_proposed"   → a proposé un date
    # "meetup_accepted"   → a accepté un date
    # "unmatch"           → a unmatch
    # "report"            → a signalé
    # "checkin"           → check-in à un spot
    # "app_open"          → ouverture de l'app
    # "feed_viewed"       → a consulté son feed du jour

    target_user_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), nullable=True
    )
    # L'autre utilisateur impliqué (null pour checkin, app_open, feed_viewed)

    # ── Données contextuelles ──
    duration_seconds: Mapped[float | None] = mapped_column(Float, nullable=True)
    # Pour profile_view : temps passé sur le profil

    metadata: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    # Données supplémentaires selon le type :
    # profile_view : {"scrolled_full": true, "time_on_prompts": 4.2}
    # skip_with_reason : {"reason": "too_far"}
    # like : {"liked_prompt": "Un dimanche parfait..."}
    # checkin : {"spot_id": "uuid", "spot_name": "Café 21"}

    __table_args__ = (
        Index("ix_behavior_user_type", "user_id", "event_type"),
        Index("ix_behavior_created", "created_at"),
        # Partition par mois recommandée en production
    )
```

### 3.21 FeedCache

```python
# app/models/feed_cache.py
import uuid
from datetime import date
from sqlalchemy import Date, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID, ARRAY
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base, UUIDMixin, TimestampMixin


class FeedCache(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "feed_caches"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False
    )

    feed_date: Mapped[date] = mapped_column(Date, nullable=False)
    # La date pour laquelle ce feed a été généré

    profile_ids: Mapped[list] = mapped_column(ARRAY(UUID(as_uuid=True)), nullable=False)
    # Liste ordonnée des profile_ids dans le feed (après shuffle L5)
    # Taille = matching_feed_size (12 par défaut)

    wildcard_ids: Mapped[list] = mapped_column(ARRAY(UUID(as_uuid=True)), default=list)
    # Sous-ensemble de profile_ids qui sont des wildcards

    new_user_ids: Mapped[list] = mapped_column(ARRAY(UUID(as_uuid=True)), default=list)
    # Sous-ensemble de profile_ids qui sont des nouveaux profils boostés

    __table_args__ = (
        Index("uq_feed_user_date", "user_id", "feed_date", unique=True),
        Index("ix_feed_date", "feed_date"),
    )
```

### 3.22 MatchingConfig (config dynamique des poids)

```python
# app/models/matching_config.py
from sqlalchemy import String, Float, Text, DateTime, func, Index
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base, UUIDMixin, TimestampMixin


class MatchingConfig(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "matching_configs"

    # ── Clé unique ──
    key: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    # Exemples de clés :
    # "geo_w_quartier"           → 0.30
    # "geo_w_spot"               → 0.35
    # "geo_w_fidelity"           → 0.20
    # "geo_w_freshness"          → 0.15
    # "lifestyle_w_tags"         → 0.40
    # "lifestyle_w_intention"    → 0.35
    # "lifestyle_w_rhythm"       → 0.15
    # "lifestyle_w_language"     → 0.10
    # "quartier_w_lives"         → 2.0
    # "quartier_w_works"         → 1.5
    # "quartier_w_hangs"         → 1.0
    # "quartier_w_interested"    → 0.8
    # "behavior_response_min"    → 0.6
    # "behavior_response_max"    → 1.4
    # "behavior_selectivity_min" → 0.7
    # "behavior_selectivity_max" → 1.3
    # "behavior_richness_min"    → 0.8
    # "behavior_richness_max"    → 1.2
    # "behavior_depth_min"       → 0.8
    # "behavior_depth_max"       → 1.3
    # "freshness_halflife_days"  → 14
    # "new_user_boost_d0_3"      → 3.0
    # "new_user_boost_d4_7"      → 2.0
    # "new_user_boost_d8_10"     → 1.5
    # "selectivity_optimal"      → 0.22
    # "selectivity_sigma"        → 0.12
    # "proximity_close_thresh"   → 0.40
    # "proximity_neighbor_thresh"→ 0.65
    # "spot_w_cafe"              → 1.2
    # "spot_w_restaurant"        → 1.5
    # "spot_w_gym"               → 1.0
    # etc.
    # "weight_sched_d0_geo"      → 0.55
    # "weight_sched_d0_life"     → 0.30
    # "weight_sched_d0_behav"    → 0.15
    # "weight_sched_d14_geo"     → 0.50
    # ... etc pour chaque palier

    # ── Valeur ──
    value: Mapped[float] = mapped_column(Float, nullable=False)

    # ── Métadonnées ──
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    # Description humaine de ce que fait ce poids
    # Ex: "Poids du sous-score quartiers dans le score géo global"

    category: Mapped[str] = mapped_column(String(30), nullable=False)
    # "geo" | "lifestyle" | "behavior" | "quartier" | "spot" |
    # "proximity" | "boost" | "selectivity" | "weight_schedule" | "limits"
    # Pour organiser dans l'interface admin

    min_value: Mapped[float | None] = mapped_column(Float, nullable=True)
    max_value: Mapped[float | None] = mapped_column(Float, nullable=True)
    # Bornes de sécurité. L'admin ne peut pas mettre geo_w_quartier à 99.0
    # par erreur. Vérifié côté API.

    updated_by: Mapped[str | None] = mapped_column(String(100), nullable=True)
    # Email ou ID de l'admin qui a modifié en dernier

    __table_args__ = (
        Index("ix_matching_config_category", "category"),
    )
```

### 3.23 Service de configuration dynamique

```python
# app/services/config_service.py
"""
Service qui charge les poids du matching engine depuis :
1. Redis (cache, TTL 5 min) → lecture la plus rapide
2. DB (MatchingConfig) → source de vérité
3. constants.py (DEFAULTS) → fallback si la clé n'existe pas en DB

Flow :
- Au démarrage de l'app ou du batch matching, tous les configs
  sont chargés en mémoire depuis Redis (ou DB si Redis est vide)
- Quand un admin modifie un poids via l'API :
  1. UPDATE en DB
  2. Invalidation du cache Redis (DEL matching_config:*)
  3. Les workers Celery reloadent au prochain cycle
  4. L'API reloade au prochain appel
- Pas besoin de redéployer. Pas besoin de restart.

Pourquoi pas juste des .env ?
- Les .env nécessitent un restart des containers
- Pas de bornes de sécurité (tu peux mettre n'importe quoi)
- Pas d'historique de qui a changé quoi
- Pas de catégorisation pour l'interface admin
- Pas de description humaine
"""
from typing import Optional
from app.core.constants import MATCHING_DEFAULTS


# ── Cache mémoire (process-level) ──
_config_cache: dict[str, float] = {}
_cache_loaded: bool = False


async def get_config(key: str, redis_client=None, db_session=None) -> float:
    """
    Récupère un poids de configuration.
    Ordre de priorité : mémoire → Redis → DB → default.
    """
    global _config_cache, _cache_loaded

    # 1. Cache mémoire (le plus rapide, 0 latence)
    if key in _config_cache:
        return _config_cache[key]

    # 2. Redis (< 1ms)
    if redis_client:
        val = await redis_client.get(f"matching_config:{key}")
        if val is not None:
            _config_cache[key] = float(val)
            return float(val)

    # 3. DB
    if db_session:
        row = await db_session.execute(
            select(MatchingConfig.value).where(MatchingConfig.key == key)
        )
        result = row.scalar_one_or_none()
        if result is not None:
            _config_cache[key] = result
            if redis_client:
                await redis_client.set(
                    f"matching_config:{key}", str(result), ex=300  # TTL 5 min
                )
            return result

    # 4. Default (constants.py)
    default = MATCHING_DEFAULTS.get(key)
    if default is not None:
        _config_cache[key] = default
        return default

    raise KeyError(f"Unknown matching config key: {key}")


async def load_all_configs(redis_client, db_session):
    """
    Charge TOUS les configs en mémoire.
    Appelé une fois au début du batch matching.
    """
    global _config_cache, _cache_loaded

    # Charger tout depuis DB
    rows = await db_session.execute(select(MatchingConfig.key, MatchingConfig.value))
    db_configs = {row.key: row.value for row in rows.fetchall()}

    # Merger avec les defaults (DB a priorité)
    _config_cache = {**MATCHING_DEFAULTS, **db_configs}
    _cache_loaded = True

    # Pousser tout dans Redis (pour les workers qui lisent en dehors du batch)
    pipe = redis_client.pipeline()
    for key, val in _config_cache.items():
        pipe.set(f"matching_config:{key}", str(val), ex=300)
    await pipe.execute()


async def set_config(
    key: str, value: float, updated_by: str, redis_client, db_session
) -> dict:
    """
    Met à jour un poids. Appelé par l'API admin.
    Valide les bornes, update DB, invalide Redis.
    """
    # Vérifier que la clé existe
    row = await db_session.execute(
        select(MatchingConfig).where(MatchingConfig.key == key)
    )
    config = row.scalar_one_or_none()

    if config is None:
        raise KeyError(f"Unknown config key: {key}")

    # Vérifier les bornes de sécurité
    if config.min_value is not None and value < config.min_value:
        raise ValueError(f"{key} minimum is {config.min_value}, got {value}")
    if config.max_value is not None and value > config.max_value:
        raise ValueError(f"{key} maximum is {config.max_value}, got {value}")

    # Sauvegarder l'ancienne valeur pour l'audit log
    old_value = config.value

    # Update DB
    config.value = value
    config.updated_by = updated_by
    await db_session.commit()

    # Invalider le cache Redis
    await redis_client.delete(f"matching_config:{key}")

    # Invalider le cache mémoire
    global _config_cache
    _config_cache.pop(key, None)

    return {
        "key": key,
        "old_value": old_value,
        "new_value": value,
        "updated_by": updated_by,
    }


def get_cached(key: str) -> float:
    """
    Lecture synchrone depuis le cache mémoire.
    À utiliser UNIQUEMENT après un load_all_configs().
    C'est ce que le matching engine utilise dans ses boucles internes
    pour ne pas faire de I/O à chaque candidat.
    """
    if key in _config_cache:
        return _config_cache[key]
    return MATCHING_DEFAULTS[key]
```

### 3.24 Defaults et mapping des clés

```python
# app/core/constants.py (section MATCHING_DEFAULTS)
# Ces valeurs sont les FALLBACKS si la clé n'existe pas en DB.
# En production, toutes ces clés sont dans la table matching_configs
# et modifiables via l'API admin sans redéployer.

MATCHING_DEFAULTS = {
    # ── Geo sub-weights (doivent sommer à 1.0) ──
    "geo_w_quartier": 0.30,
    "geo_w_spot": 0.35,
    "geo_w_fidelity": 0.20,
    "geo_w_freshness": 0.15,

    # ── Lifestyle sub-weights (doivent sommer à 1.0) ──
    "lifestyle_w_tags": 0.40,
    "lifestyle_w_intention": 0.35,
    "lifestyle_w_rhythm": 0.15,
    "lifestyle_w_language": 0.10,

    # ── Quartier relation weights ──
    "quartier_w_lives": 2.0,
    "quartier_w_works": 1.5,
    "quartier_w_hangs": 1.0,
    "quartier_w_interested": 0.8,

    # ── Spot social weights ──
    "spot_w_cafe": 1.2,
    "spot_w_restaurant": 1.5,
    "spot_w_gym": 1.0,
    "spot_w_coworking": 0.8,
    "spot_w_bar": 1.5,
    "spot_w_worship": 1.0,
    "spot_w_market": 0.7,
    "spot_w_beach": 1.3,
    "spot_w_park": 1.1,
    "spot_w_other": 0.8,

    # ── Behavior multiplier bounds ──
    "behavior_response_min": 0.6,
    "behavior_response_max": 1.4,
    "behavior_selectivity_min": 0.7,
    "behavior_selectivity_max": 1.3,
    "behavior_richness_min": 0.8,
    "behavior_richness_max": 1.2,
    "behavior_depth_min": 0.8,
    "behavior_depth_max": 1.3,

    # ── Freshness ──
    "freshness_halflife_days": 14.0,

    # ── New user boost ──
    "new_user_boost_d0_3": 3.0,
    "new_user_boost_d4_7": 2.0,
    "new_user_boost_d8_10": 1.5,

    # ── Selectivity bell curve ──
    "selectivity_optimal_ratio": 0.22,
    "selectivity_sigma": 0.12,

    # ── Proximity thresholds ──
    "proximity_close_threshold": 0.40,
    "proximity_neighbor_threshold": 0.65,

    # ── Weight schedule (poids adaptatifs par ancienneté) ──
    "weight_sched_d0_geo": 0.55,
    "weight_sched_d0_life": 0.30,
    "weight_sched_d0_behav": 0.15,
    "weight_sched_d14_geo": 0.50,
    "weight_sched_d14_life": 0.32,
    "weight_sched_d14_behav": 0.18,
    "weight_sched_d30_geo": 0.45,
    "weight_sched_d30_life": 0.35,
    "weight_sched_d30_behav": 0.20,
    "weight_sched_d60_geo": 0.40,
    "weight_sched_d60_life": 0.32,
    "weight_sched_d60_behav": 0.28,
    "weight_sched_d90_geo": 0.35,
    "weight_sched_d90_life": 0.30,
    "weight_sched_d90_behav": 0.35,

    # ── Feed limits ──
    "feed_size": 12.0,
    "wildcard_count": 2.0,
    "match_expire_days": 7.0,
    "skip_cooldown_days": 30.0,
    "min_weekly_visibility_default": 15.0,

    # ── Quartier limits ──
    "max_quartiers_lives": 2.0,
    "max_quartiers_works": 2.0,
    "max_quartiers_hangs": 4.0,
    "max_quartiers_interested_free": 3.0,
    "max_quartiers_interested_premium": 6.0,

    # ── Likes limits ──
    "daily_likes_free": 5.0,
    "daily_likes_premium": 50.0,

    # ── Intention matrix ──
    "intention_serious_serious": 1.0,
    "intention_serious_getting_to_know": 0.7,
    "intention_serious_friendship_first": 0.2,
    "intention_serious_open": 0.4,
    "intention_getting_to_know_getting_to_know": 1.0,
    "intention_getting_to_know_friendship_first": 0.6,
    "intention_getting_to_know_open": 0.8,
    "intention_friendship_first_friendship_first": 1.0,
    "intention_friendship_first_open": 0.5,
    "intention_open_open": 1.0,

    # ── Moderation thresholds ──
    "nsfw_threshold_reject": 0.7,
    "nsfw_threshold_review": 0.4,
}

# Seed script pour peupler la table matching_configs avec les defaults
# scripts/seed_matching_config.py
#
# Pour chaque clé dans MATCHING_DEFAULTS :
#   INSERT INTO matching_configs (key, value, description, category, min_value, max_value)
#   ON CONFLICT (key) DO NOTHING
#
# Catégories auto-détectées par préfixe :
#   "geo_w_*"         → category="geo"
#   "lifestyle_w_*"   → category="lifestyle"
#   "quartier_w_*"    → category="quartier"
#   "spot_w_*"        → category="spot"
#   "behavior_*"      → category="behavior"
#   "freshness_*"     → category="freshness"
#   "new_user_*"      → category="boost"
#   "selectivity_*"   → category="selectivity"
#   "proximity_*"     → category="proximity"
#   "weight_sched_*"  → category="weight_schedule"
#   "intention_*"     → category="intention"
#   "max_quartiers_*" → category="limits"
#   "daily_likes_*"   → category="limits"
#   "feed_*"          → category="feed"
#   "nsfw_*"          → category="moderation"
#
# Bornes de sécurité par catégorie :
#   weights (geo, lifestyle) : min=0.0, max=1.0
#   quartier/spot weights    : min=0.0, max=5.0
#   behavior bounds          : min=0.1, max=3.0
#   boost multipliers        : min=1.0, max=10.0
#   thresholds               : min=0.0, max=1.0
#   limits                   : min=1.0, max=100.0
```

### 3.25 API admin pour la config

```python
# ── GET /admin/matching-config ──
# Retourne toutes les configs groupées par catégorie
# Response 200
{
    "configs": {
        "geo": [
            {
                "key": "geo_w_quartier",
                "value": 0.30,
                "description": "Poids du sous-score quartiers dans le score géo",
                "min_value": 0.0,
                "max_value": 1.0,
                "updated_at": "2026-04-14T10:00:00Z",
                "updated_by": "admin@flaam.app"
            },
            {
                "key": "geo_w_spot",
                "value": 0.35,
                "description": "Poids du sous-score spots dans le score géo",
                "min_value": 0.0,
                "max_value": 1.0,
                "updated_at": null,
                "updated_by": null
            }
        ],
        "quartier": [...],
        "lifestyle": [...],
        "behavior": [...],
        "boost": [...],
        ...
    },
    "total_keys": 67,
    "last_global_update": "2026-04-14T10:00:00Z"
}

# ── PATCH /admin/matching-config/{key} ──
# Met à jour un poids
# Request
{
    "value": 0.40
}
# Response 200
{
    "key": "geo_w_quartier",
    "old_value": 0.30,
    "new_value": 0.40,
    "updated_by": "admin@flaam.app",
    "cache_invalidated": true,
    "warning": null
}

# Response 200 (avec warning si les sous-poids ne somment plus à 1.0)
{
    "key": "geo_w_quartier",
    "old_value": 0.30,
    "new_value": 0.40,
    "updated_by": "admin@flaam.app",
    "cache_invalidated": true,
    "warning": "geo sub-weights now sum to 1.10 (expected 1.0). Other geo weights: spot=0.35, fidelity=0.20, freshness=0.15"
}

# Response 400
{
    "error": "value_out_of_bounds",
    "message": "geo_w_quartier must be between 0.0 and 1.0, got 5.0"
}

# ── POST /admin/matching-config/reset/{key} ──
# Remet un poids à sa valeur par défaut (MATCHING_DEFAULTS)
# Response 200
{
    "key": "geo_w_quartier",
    "old_value": 0.40,
    "new_value": 0.30,
    "reset_to": "default"
}

# ── POST /admin/matching-config/reset-all ──
# Remet TOUS les poids à leurs valeurs par défaut
# Demande confirmation
# Request
{
    "confirm": true
}
# Response 200
{
    "reset_count": 67,
    "message": "All matching configs reset to defaults"
}

# ── GET /admin/matching-config/audit ──
# Historique des modifications
# Response 200
{
    "changes": [
        {
            "key": "geo_w_quartier",
            "old_value": 0.30,
            "new_value": 0.40,
            "updated_by": "admin@flaam.app",
            "updated_at": "2026-04-14T10:00:00Z"
        },
        ...
    ]
}
```

### Comment le matching engine utilise la config dynamique

```python
# AVANT (hardcoded) :
from app.core.constants import GEO_W_QUARTIER, GEO_W_SPOT

raw = GEO_W_QUARTIER * q_score + GEO_W_SPOT * s_score  # Changement = redeploy

# APRÈS (dynamique) :
from app.services.config_service import get_cached

raw = (
    get_cached("geo_w_quartier") * q_score
    + get_cached("geo_w_spot") * s_score
    + get_cached("geo_w_fidelity") * f_score
    + get_cached("geo_w_freshness") * fr_score
)
# Changement = PATCH /admin/matching-config/geo_w_quartier → pris en compte au prochain batch

# Le flow complet :
# 1. Admin change un poids via l'API
# 2. DB mise à jour
# 3. Redis invalidé
# 4. Au prochain batch nocturne (ou recalcul incrémental) :
#    load_all_configs() recharge tout depuis DB → Redis → mémoire
# 5. Les feeds sont générés avec les nouveaux poids
# 6. Le matin, les utilisateurs voient un feed légèrement différent
#
# Latence du changement : max 24h (prochain batch).
# Si urgent : déclencher manuellement le batch via l'API admin
# POST /admin/matching/trigger-batch
```

---

## 4. Constantes & Enums

```python
# app/core/constants.py
from enum import Enum


# ── Genre ──
class Gender(str, Enum):
    MAN = "man"
    WOMAN = "woman"
    NON_BINARY = "non_binary"


class SeekingGender(str, Enum):
    MEN = "men"
    WOMEN = "women"
    EVERYONE = "everyone"


# ── Intentions ──
class Intention(str, Enum):
    SERIOUS = "serious"
    GETTING_TO_KNOW = "getting_to_know"
    FRIENDSHIP_FIRST = "friendship_first"
    OPEN = "open"


# Matrice de compatibilité des intentions
# intention_a → intention_b → score (0.0 à 1.0)
INTENTION_MATRIX = {
    "serious":          {"serious": 1.0, "getting_to_know": 0.7, "friendship_first": 0.2, "open": 0.4},
    "getting_to_know":  {"serious": 0.7, "getting_to_know": 1.0, "friendship_first": 0.6, "open": 0.8},
    "friendship_first": {"serious": 0.2, "getting_to_know": 0.6, "friendship_first": 1.0, "open": 0.5},
    "open":             {"serious": 0.4, "getting_to_know": 0.8, "friendship_first": 0.5, "open": 1.0},
}


# ── Secteurs ──
class Sector(str, Enum):
    TECH = "tech"
    FINANCE = "finance"
    HEALTH = "health"
    EDUCATION = "education"
    COMMERCE = "commerce"
    CREATIVE = "creative"
    PUBLIC_ADMIN = "public_admin"
    STUDENT = "student"
    OTHER = "other"


# ── Catégories de spots ──
class SpotCategory(str, Enum):
    CAFE = "cafe"
    RESTAURANT = "restaurant"
    GYM = "gym"
    COWORKING = "coworking"
    BAR = "bar"
    WORSHIP = "worship"
    MARKET = "market"
    BEACH = "beach"
    PARK = "park"
    OTHER = "other"


# Poids social par catégorie de spot
SPOT_SOCIAL_WEIGHTS = {
    "cafe": 1.2,
    "restaurant": 1.5,
    "gym": 1.0,
    "coworking": 0.8,
    "bar": 1.5,
    "worship": 1.0,
    "market": 0.7,
    "beach": 1.3,
    "park": 1.1,
    "other": 0.8,
}


# ── Niveaux de fidélité ──
FIDELITY_THRESHOLDS = {
    "declared": (0, 0.5),       # 0 check-ins, score 0.5
    "confirmed": (1, 0.7),      # 1-2 check-ins, score 0.7
    "regular": (3, 0.85),       # 3-5 check-ins, score 0.85
    "regular_plus": (6, 1.0),   # 6+ check-ins, score 1.0
}

FIDELITY_LABELS = {
    "fr": {
        "declared": "Déclaré",
        "confirmed": "Confirmé",
        "regular": "Régulier",
        "regular_plus": "Habitué",
    },
    "en": {
        "declared": "Declared",
        "confirmed": "Confirmed",
        "regular": "Regular",
        "regular_plus": "Regular",
    },
}


# ── Tags lifestyle disponibles ──
AVAILABLE_TAGS = [
    "foodie", "sport", "tech", "travel", "music", "reading",
    "faith_based", "cinema", "brunch_addict", "yoga", "gaming",
    "entrepreneurship", "art", "dancing", "cooking", "photography",
    "fashion", "volunteering", "pets", "nightlife", "outdoor",
    "wellness", "cars", "writing", "diy",
]
MAX_TAGS = 8


# ── Prompts disponibles ──
AVAILABLE_PROMPTS = {
    "fr": [
        "Un dimanche parfait c'est...",
        "Mon maquis préféré parce que...",
        "Je cherche quelqu'un qui...",
        "La chose la plus spontanée que j'ai faite...",
        "Je suis passionné(e) par...",
        "Mon guilty pleasure c'est...",
        "Si je pouvais dîner avec une personne...",
        "Un truc que les gens ne devinent pas sur moi...",
        "Mon quartier est le meilleur parce que...",
        "Le week-end tu me trouves...",
    ],
    "en": [
        "A perfect Sunday is...",
        "My favorite spot because...",
        "I'm looking for someone who...",
        "The most spontaneous thing I've done...",
        "I'm passionate about...",
        "My guilty pleasure is...",
        "If I could have dinner with anyone...",
        "Something people don't guess about me...",
        "My neighborhood is the best because...",
        "On weekends you'll find me...",
    ],
}
MAX_PROMPTS = 3
MAX_PROMPT_ANSWER_LENGTH = 200


# ── Matching engine weights ──
# Poids adaptatifs par ancienneté du compte
WEIGHT_SCHEDULE = {
    # jour → (geo_weight, lifestyle_weight, behavior_weight)
    0:  (0.55, 0.30, 0.15),   # Nouveau
    14: (0.50, 0.32, 0.18),
    30: (0.45, 0.35, 0.20),
    60: (0.40, 0.32, 0.28),
    90: (0.35, 0.30, 0.35),   # Mature
}

# Geo sub-weights
GEO_W_QUARTIER = 0.30
GEO_W_SPOT = 0.35
GEO_W_FIDELITY = 0.20
GEO_W_FRESHNESS = 0.15

# Lifestyle sub-weights
LIFESTYLE_W_TAGS = 0.40
LIFESTYLE_W_INTENTION = 0.35
LIFESTYLE_W_RHYTHM = 0.15
LIFESTYLE_W_LANGUAGE = 0.10

# Quartier relation weights
QUARTIER_RELATION_WEIGHTS = {
    "lives": 2.0,
    "works": 1.5,
    "hangs": 1.0,
    "interested": 0.8,
    # "interested" pèse moins que "hangs" parce que c'est déclaratif
    # sans activité réelle. Mais c'est assez pour remonter des profils
    # de quartiers éloignés (ex: Agoè → Bè) dans le feed.
}

# Limites quartiers par type
MAX_QUARTIERS_LIVES = 2
MAX_QUARTIERS_WORKS = 2
MAX_QUARTIERS_HANGS = 4
MAX_QUARTIERS_INTERESTED_FREE = 3
MAX_QUARTIERS_INTERESTED_PREMIUM = 6

# Quartier proximity thresholds
PROXIMITY_MIN_SCORE = 0.05          # Score minimum (même ville, très éloigné)
PROXIMITY_NEIGHBOR_THRESHOLD = 0.65  # Au-dessus = considéré comme voisin
PROXIMITY_CLOSE_THRESHOLD = 0.40     # Au-dessus = considéré comme proche

# Soft matching : comment la proximité affecte le score géo
# Quand deux utilisateurs n'ont PAS un quartier en commun mais ont
# des quartiers PROCHES, on accorde un score partiel :
#
# Quartier exact en commun → poids full (QUARTIER_RELATION_WEIGHTS)
# Quartiers voisins (proximity > 0.65) → poids × proximity_score
# Quartiers proches (proximity 0.40-0.65) → poids × proximity_score × 0.5
# Quartiers éloignés (proximity < 0.40) → ignoré dans le soft match
#
# Exemple concret :
# User A vit à Bè (weight=2.0), User B vit à Tokoin (proximity=0.82)
# → Score = 2.0 × 0.82 = 1.64 (au lieu de 0.0 sans soft matching)
#
# User A vit à Agoè (weight=2.0), User B vit à Bè (proximity=0.21)
# → Score = 0.0 (proximity < 0.40, ignoré en soft match)
# SAUF si User A a "interested" sur Bè → weight 0.8, exact match → score 0.8
#
# C'est exactement le cas d'usage : la proximité géo ne peut pas
# résoudre Agoè→Bè, mais le type "interested" oui.

# Behavior multiplier bounds
BEHAVIOR_RESPONSE_RANGE = (0.6, 1.4)
BEHAVIOR_SELECTIVITY_RANGE = (0.7, 1.3)
BEHAVIOR_RICHNESS_RANGE = (0.8, 1.2)
BEHAVIOR_DEPTH_RANGE = (0.8, 1.3)

# Freshness decay (jours depuis dernier check-in → multiplicateur)
FRESHNESS_DECAY_HALFLIFE = 14  # Demi-vie en jours
# Score = exp(-0.693 * jours / halflife)
# 1 jour → 0.95, 7 jours → 0.71, 14 jours → 0.50, 30 jours → 0.23

# New user boost schedule
NEW_USER_BOOST = {
    (0, 3): 3.0,     # Jours 0-3 : 3x visibilité
    (4, 7): 2.0,     # Jours 4-7 : 2x
    (8, 10): 1.5,    # Jours 8-10 : 1.5x
}

# Selectivity sweet spot (bell curve center)
SELECTIVITY_OPTIMAL_RATIO = 0.22  # Like 22% des profils = optimal
SELECTIVITY_SIGMA = 0.12          # Écart-type de la cloche


# ── Raisons de skip ──
class SkipReason(str, Enum):
    NOT_MY_TYPE = "not_my_type"
    TOO_FAR = "too_far"
    DIFFERENT_INTENTIONS = "different_intentions"
    NO_REASON = "no_reason"


# ── Raisons de signalement ──
class ReportReason(str, Enum):
    FAKE_PROFILE = "fake_profile"
    INAPPROPRIATE = "inappropriate_behavior"
    HARASSMENT = "harassment"
    SCAM = "scam"
    UNDERAGE = "underage"
    OTHER = "other"


# ── Catégories d'events ──
class EventCategory(str, Enum):
    AFTERWORK = "afterwork"
    SPORT = "sport"
    BRUNCH = "brunch"
    MEETUP = "meetup"
    EXPO = "expo"
    PARTY = "party"
    OTHER = "other"


# ── Message flags ──
SPAM_KEYWORDS = [
    "envoie moi de l'argent", "send money", "western union",
    "momo transfer", "compte bancaire", "bank account",
    "investissement", "crypto", "forex",
]
SCAM_URL_PATTERNS = [
    r"bit\.ly", r"tinyurl", r"t\.co",
    # Plus tout domaine non-whitelisté
]


# ── Modération photo ──
NSFW_THRESHOLD_REJECT = 0.7
NSFW_THRESHOLD_REVIEW = 0.4
```

---

## 5. API endpoints complets

### 5.1 Auth

| Méthode | Route | Description | Auth | Rate limit |
|---------|-------|-------------|------|------------|
| POST | `/auth/otp/request` | Demande un code OTP | Non | 3/10min/numéro |
| POST | `/auth/otp/verify` | Vérifie le code OTP, retourne JWT | Non | 3/10min/numéro |
| POST | `/auth/refresh` | Refresh le token JWT | Refresh token | 10/min |
| POST | `/auth/logout` | Invalide le refresh token | Oui | — |
| DELETE | `/auth/account` | Suppression de compte (soft delete) | Oui | 1/jour |

```python
# ── POST /auth/otp/request ──
# Request
{
    "phone": "+22890123456",
    "device_fingerprint": "sha256:abcdef123456"
}
# Response 200
{
    "message": "OTP sent",
    "expires_in": 600,
    "retry_after": 60
}
# Response 429
{
    "error": "rate_limited",
    "retry_after": 45
}

# ── POST /auth/otp/verify ──
# Request
{
    "phone": "+22890123456",
    "code": "482917",
    "device_fingerprint": "sha256:abcdef123456",
    "platform": "android",
    "app_version": "1.0.0",
    "os_version": "Android 12"
}
# Response 200 (utilisateur existant)
{
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "token_type": "bearer",
    "expires_in": 900,
    "is_new_user": false,
    "user_id": "uuid"
}
# Response 200 (nouvel utilisateur)
{
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "token_type": "bearer",
    "expires_in": 900,
    "is_new_user": true,
    "user_id": "uuid",
    "onboarding_step": "city_selection"
}
# Response 401
{
    "error": "invalid_otp",
    "attempts_remaining": 2
}
```

### 5.2 Profiles

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/profiles/me` | Mon profil complet | Oui |
| PUT | `/profiles/me` | Mettre à jour mon profil | Oui |
| GET | `/profiles/{user_id}` | Voir le profil d'un autre (si autorisé) | Oui |
| POST | `/profiles/me/verify-selfie` | Upload selfie de vérification | Oui |
| GET | `/profiles/me/completeness` | Score de complétion | Oui |
| PATCH | `/profiles/me/visibility` | Toggle visibilité (mode pause) | Oui |

```python
# ── PUT /profiles/me ──
# Request
{
    "display_name": "Ama",
    "birth_date": "2000-03-15",
    # IMPORTANT : "gender" n'est PAS modifiable via cet endpoint.
    # Le genre est déclaré à l'onboarding (étape basic_info) et verrouillé.
    # Si "gender" est envoyé dans le body → 400 gender_not_modifiable.
    # Raison : le genre est un paramètre structurel du matching (waitlist genrée,
    # first impression feed, quality gate masculin). Permettre le changement
    # ouvrirait la porte au catfish (photos de sa sœur + changement genre).
    # Changement légitime : uniquement via admin PATCH /admin/users/{id}/gender
    # avec invalidation du selfie + review humaine.
    #
    # "seeking_gender" EST modifiable librement (c'est une préférence, pas une identité).
    "seeking_gender": "men",
    "intention": "serious",
    "sector": "finance",
    "rhythm": "night_owl",
    "prompts": [
        {
            "question": "Un dimanche parfait c'est...",
            "answer": "Grasse mat + brunch à Tokoin + Netflix"
        },
        {
            "question": "Mon maquis préféré parce que...",
            "answer": "Le poulet braisé de chez Tonton, imbattable"
        }
    ],
    "tags": ["foodie", "brunch_addict", "yoga", "finance", "travel"],
    "languages": ["fr", "ewe", "en"],
    "seeking_age_min": 25,
    "seeking_age_max": 35
}
# Response 200
{
    "id": "uuid",
    "user_id": "uuid",
    "display_name": "Ama",
    "age": 26,
    "gender": "woman",
    "intention": "serious",
    "sector": "finance",
    "rhythm": "night_owl",
    "prompts": [...],
    "tags": [...],
    "languages": [...],
    "photos": [...],
    "quartiers": [...],
    "spots": [...],
    "profile_completeness": 0.85,
    "is_selfie_verified": true,
    "is_id_verified": false,
    "updated_at": "2026-04-14T12:00:00Z"
}
```

### 5.3 Photos

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| POST | `/photos` | Upload une photo | Oui |
| DELETE | `/photos/{photo_id}` | Supprimer une photo | Oui |
| PATCH | `/photos/reorder` | Réordonner les photos | Oui |

```python
# ── POST /photos ──
# Request : multipart/form-data
# - file: image (JPEG, PNG, WebP, max 10MB)
# - display_order: int (0-5)
# Response 201
{
    "id": "uuid",
    "original_url": "https://cdn.flaam.app/photos/uuid/uuid_original.webp",
    "thumbnail_url": "https://cdn.flaam.app/photos/uuid/uuid_thumb.webp",
    "medium_url": "https://cdn.flaam.app/photos/uuid/uuid_medium.webp",
    "display_order": 1,
    "moderation_status": "pending",
    "width": 1200,
    "height": 1600,
    "file_size_bytes": 142000
}
# Response 400
{
    "error": "max_photos_reached",
    "message": "Maximum 6 photos allowed"
}
```

### 5.4 Quartiers

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/quartiers?city_id={id}` | Liste des quartiers d'une ville | Oui |
| POST | `/quartiers/me` | Ajouter un quartier à mon profil | Oui |
| DELETE | `/quartiers/me/{quartier_id}?relation_type={type}` | Retirer un quartier | Oui |
| GET | `/quartiers/me` | Mes quartiers (par type) | Oui |
| GET | `/quartiers/{id}/nearby` | Quartiers voisins (pour suggestions) | Oui |

```python
# ── POST /quartiers/me ──
# Request
{
    "quartier_id": "uuid",
    "relation_type": "interested"
    // "lives" | "works" | "hangs" | "interested"
}
# Response 201
{
    "id": "uuid",
    "quartier": {
        "id": "uuid",
        "name": "Bè",
        "latitude": 6.1601,
        "longitude": 1.2290
    },
    "relation_type": "interested",
    "is_primary": false
}

# Règles de validation (côté service) :
#
# 1. Limites par type :
#    - lives : max 2 (tu peux vivre entre deux quartiers)
#    - works : max 2
#    - hangs : max 4
#    - interested : max 3 (free) / max 6 (premium)
#
# 2. Minimum : au moins 1 quartier "lives" obligatoire
#
# 3. Même ville uniquement : le quartier doit appartenir à la
#    même city_id que l'utilisateur
#
# 4. Pas de doublon type+quartier : tu ne peux pas avoir
#    deux fois "lives" sur Tokoin, mais tu peux avoir
#    "lives" à Tokoin ET "hangs" à Tokoin (ça veut dire
#    tu y vis ET tu y sors)

# Response 400 — limite atteinte
{
    "error": "max_quartiers_reached",
    "message": "Maximum 3 interested quartiers for free users",
    "limit": 3,
    "current": 3,
    "upgrade_to_premium": true
}

# Response 400 — quartier hors de ta ville
{
    "error": "quartier_not_in_city",
    "message": "This quartier belongs to Abidjan, not Lomé"
}

# ── GET /quartiers/me ──
# Response 200
{
    "lives": [
        {"quartier_id": "uuid", "name": "Agoè", "is_primary": true}
    ],
    "works": [
        {"quartier_id": "uuid", "name": "Tokoin", "is_primary": false}
    ],
    "hangs": [
        {"quartier_id": "uuid", "name": "Djidjolé", "is_primary": false},
        {"quartier_id": "uuid", "name": "Hédzranawoé", "is_primary": false}
    ],
    "interested": [
        {"quartier_id": "uuid", "name": "Bè", "is_primary": false},
        {"quartier_id": "uuid", "name": "Nyékonakpoè", "is_primary": false}
    ],
    "limits": {
        "lives": {"current": 1, "max": 2},
        "works": {"current": 1, "max": 2},
        "hangs": {"current": 2, "max": 4},
        "interested": {"current": 2, "max": 3, "max_premium": 6}
    }
}

# ── GET /quartiers/{id}/nearby ──
# Retourne les quartiers voisins triés par proximité
# Utile pour suggérer des quartiers d'intérêt à l'utilisateur
# Response 200
{
    "quartier": {"id": "uuid", "name": "Agoè"},
    "nearby": [
        {"id": "uuid", "name": "Adidogomé", "proximity": 0.79, "distance_km": 2.5},
        {"id": "uuid", "name": "Zanguéra", "proximity": 0.72, "distance_km": 3.3},
        {"id": "uuid", "name": "Totsi", "proximity": 0.65, "distance_km": 4.1}
    ]
}
```

### 5.5 Spots

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/spots?city_id={id}&category={cat}&q={search}` | Chercher des spots | Oui |
| GET | `/spots/{spot_id}` | Détail d'un spot | Oui |
| POST | `/spots/me` | Ajouter un spot à mon profil | Oui |
| DELETE | `/spots/me/{spot_id}` | Retirer un spot | Oui |
| PATCH | `/spots/me/{spot_id}/visibility` | Masquer/afficher un spot | Oui |
| POST | `/spots/me/{spot_id}/checkin` | Check-in à un spot | Oui |
| POST | `/spots/suggest` | Proposer un nouveau spot | Oui |
| GET | `/spots/popular?city_id={id}` | Spots populaires | Oui |

```python
# ── POST /spots/me/{spot_id}/checkin ──
# Request
{
    "latitude": 6.1725,
    "longitude": 1.2137
}
# Response 200
{
    "spot_id": "uuid",
    "spot_name": "Café 21",
    "checkin_count": 4,
    "fidelity_level": "regular",
    "fidelity_label": "Régulier",
    "previous_level": "confirmed",
    "level_upgraded": true
}
# Response 400
{
    "error": "too_far",
    "message": "You must be within 100m of the spot",
    "distance_meters": 340
}
```

### 5.6 Feed & discovery

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/feed` | Mon feed du jour | Oui |
| GET | `/feed/crossed` | Section "Déjà croisés" | Oui |
| POST | `/feed/{profile_id}/like` | Liker un profil | Oui |
| POST | `/feed/{profile_id}/skip` | Passer un profil | Oui |
| POST | `/feed/{profile_id}/view` | Logger une vue (implicite) | Oui |

```python
# ── GET /feed ──
# Response 200
{
    "feed_date": "2026-04-14",
    "profiles": [
        {
            "id": "uuid",
            "user_id": "uuid",
            "display_name": "Ama",
            "age": 26,
            "intention": "serious",
            "sector": "finance",
            "rhythm": "night_owl",
            "photos": [
                {"url": "https://cdn.flaam.app/..._medium.webp", "order": 0},
                {"url": "https://cdn.flaam.app/..._medium.webp", "order": 1}
            ],
            "prompts": [...],
            "tags": ["foodie", "yoga"],
            "tags_in_common": ["foodie", "brunch_addict"],
            "languages": ["fr", "ewe"],
            "quartiers": [
                {"name": "Tokoin", "relation_type": "lives"},
                {"name": "Bè", "relation_type": "hangs"}
            ],
            "spots_in_common": [
                {
                    "spot_id": "uuid",
                    "name": "Café 21",
                    "category": "cafe",
                    "their_fidelity": "regular_plus",
                    "your_fidelity": "regular"
                }
            ],
            "geo_score_display": 82,
            "is_verified": true,
            "is_new_user": false,
            "is_wildcard": false
        }
        // ... 8-12 profils
    ],
    "remaining_likes": 3,
    "is_premium": false,
    "next_refresh_at": "2026-04-15T03:00:00+00:00"
}

# ── POST /feed/{profile_id}/like ──
# Request
{
    "liked_prompt": "Un dimanche parfait c'est..."
    // Optionnel : si l'utilisateur a liké un prompt spécifique
}
# Response 200 (pas encore de match)
{
    "status": "liked",
    "remaining_likes": 2
}
# Response 200 (match !)
{
    "status": "matched",
    "match_id": "uuid",
    "ice_breaker": "Vous allez tous les deux au Café 21 — c'est quoi votre commande préférée ?",
    "remaining_likes": 2
}
# Response 429
{
    "error": "daily_likes_exhausted",
    "resets_at": "2026-04-15T03:00:00+00:00",
    "upgrade_to_premium": true
}

# ── POST /feed/{profile_id}/skip ──
# Request
{
    "reason": "too_far"
    // Optionnel. "not_my_type" | "too_far" | "different_intentions" | "no_reason"
}
# Response 200
{
    "status": "skipped",
    "will_reappear_after": "2026-05-14"
}

# ── POST /feed/{profile_id}/view ──
# Request (envoyé automatiquement par le client)
{
    "duration_seconds": 8.4,
    "scrolled_full": true,
    "prompts_viewed": 2
}
# Response 204 (no content)
```

### 5.7 Matches

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/matches` | Mes matchs actifs | Oui |
| GET | `/matches/{match_id}` | Détail d'un match | Oui |
| DELETE | `/matches/{match_id}` | Unmatch | Oui |
| GET | `/matches/likes-received` | Qui m'a liké (premium) | Premium |

### 5.8 Messages (REST + WebSocket)

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/messages/{match_id}?cursor={id}&limit=50` | Historique des messages | Oui |
| POST | `/messages/{match_id}` | Envoyer un message | Oui |
| POST | `/messages/{match_id}/voice` | Envoyer un vocal | Oui |
| POST | `/messages/{match_id}/meetup` | Proposer un date | Oui |
| PATCH | `/messages/{message_id}/meetup` | Accepter/modifier un date | Oui |
| PATCH | `/messages/{match_id}/read` | Marquer comme lu | Oui |
| WS | `/ws/chat` | WebSocket pour le temps réel | JWT |

```python
# ── WebSocket /ws/chat ──
# Connexion : ws://api.flaam.app/ws/chat?token=JWT_ACCESS_TOKEN

# Message entrant (client → serveur)
{
    "type": "message",
    "match_id": "uuid",
    "content": "Salut ! Tu vas souvent au Café 21 ?",
    "message_type": "text"
}

# Message sortant (serveur → client)
{
    "type": "new_message",
    "match_id": "uuid",
    "message": {
        "id": "uuid",
        "sender_id": "uuid",
        "content": "Salut ! Tu vas souvent au Café 21 ?",
        "message_type": "text",
        "created_at": "2026-04-14T15:30:00Z"
    }
}

# Typing indicator
{"type": "typing_start", "match_id": "uuid"}
{"type": "typing_stop", "match_id": "uuid"}

# Read receipt
{"type": "read", "match_id": "uuid", "last_read_message_id": "uuid"}

# Match event (pushed by server)
{
    "type": "new_match",
    "match": {
        "id": "uuid",
        "user": { /* profil résumé */ },
        "ice_breaker": "...",
        "matched_at": "2026-04-14T15:30:00Z"
    }
}

# Connexion perdue → le client reconnecte automatiquement
# et envoie : {"type": "sync", "last_message_id": "uuid"}
# Le serveur renvoie tous les messages manqués
```

### 5.9 Events

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/events?city_id={id}&from={date}&to={date}` | Events à venir | Oui |
| GET | `/events/{event_id}` | Détail d'un event | Oui |
| POST | `/events/{event_id}/register` | S'inscrire | Oui |
| DELETE | `/events/{event_id}/register` | Se désinscrire | Oui |
| GET | `/events/{event_id}/matches-preview` | Nb de matchs potentiels à l'event | Oui |
| POST | `/events` | Proposer un event (V2) | Oui |

### 5.10 Notifications

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/notifications/preferences` | Mes préférences | Oui |
| PUT | `/notifications/preferences` | Mettre à jour | Oui |

### 5.11 Subscriptions (premium)

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/subscriptions/me` | Mon abonnement | Oui |
| GET | `/subscriptions/plans` | Plans disponibles (prix par ville) | Oui |
| POST | `/subscriptions/initialize` | Initier un paiement | Oui |
| POST | `/subscriptions/webhook/paystack` | Webhook Paystack | Signature |
| POST | `/subscriptions/webhook/flutterwave` | Webhook Flutterwave | Signature |
| POST | `/subscriptions/cancel` | Annuler le renouvellement | Oui |

```python
# ── POST /subscriptions/initialize ──
# Request
{
    "plan": "monthly",
    "payment_method": "momo",
    "provider": "paystack"
}
# Response 200
{
    "authorization_url": "https://paystack.com/pay/xxx",
    "reference": "flaam_sub_uuid",
    "provider": "paystack"
}
# Le client ouvre l'URL dans un WebView
# Paystack gère le flow MoMo/TMoney/carte
# Après paiement, Paystack envoie un webhook à /subscriptions/webhook/paystack

# ── POST /subscriptions/webhook/paystack ──
# Paystack envoie :
{
    "event": "charge.success",
    "data": {
        "reference": "flaam_sub_uuid",
        "amount": 200000,  // 2000 XOF en kobo
        "currency": "XOF",
        "status": "success",
        "customer": { "email": "...", "phone": "+22890123456" }
    }
}
# Le serveur vérifie la signature HMAC
# Active l'abonnement premium
# Met à jour user.is_premium = true
# Envoie une notification push de confirmation
```

### 5.12 Safety

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| POST | `/safety/report` | Signaler un utilisateur | Oui |
| POST | `/safety/block` | Bloquer un utilisateur | Oui |
| DELETE | `/safety/block/{user_id}` | Débloquer | Oui |
| POST | `/safety/share-date` | Partager un date avec un contact | Oui |
| POST | `/safety/emergency` | Bouton d'urgence | Oui |
| POST | `/contacts/blacklist` | Importer contacts à masquer (hashes SHA-256) | Oui |
| GET | `/contacts/blacklist` | Liste des contacts masqués | Oui |
| DELETE | `/contacts/blacklist/{phone_hash}` | Retirer un contact de la blacklist | Oui |

```python
# ── POST /safety/share-date ──
# Request
{
    "match_id": "uuid",
    "contact_phone": "+22891234567",
    "contact_name": "Maman",
    "spot_id": "uuid",
    "planned_time": "2026-04-14T19:00:00+00:00",
    "timer_hours": 3
}
# Response 200
{
    "share_id": "uuid",
    "sms_sent": true,
    "timer_expires_at": "2026-04-14T22:00:00+00:00",
    "message": "SMS sent to Maman. If you don't deactivate the timer in 3h, she will be alerted."
}
# Le SMS envoyé contient :
# "[Flaam] Ama va à un rendez-vous au Café 21 à 19h.
#  En cas de problème, elle sera localisée ici : https://flaam.app/safety/uuid"

# ── POST /safety/emergency ──
# Request
{
    "match_id": "uuid"
}
# Response 200
# Le serveur :
# 1. Récupère la dernière position GPS connue (du check-in ou du share-date)
# 2. Envoie un SMS d'urgence au contact de confiance
# 3. Log l'incident pour la modération
# 4. Flag le match pour review
{
    "emergency_id": "uuid",
    "contact_alerted": true,
    "location_shared": true
}
```

### 5.13 Admin

| Méthode | Route | Description | Auth |
|---------|-------|-------------|------|
| GET | `/admin/reports?status=pending` | Reports en attente | Admin |
| PATCH | `/admin/reports/{report_id}` | Traiter un report | Admin |
| POST | `/admin/users/{user_id}/ban` | Bannir un utilisateur | Admin |
| POST | `/admin/users/{user_id}/unban` | Débannir | Admin |
| GET | `/admin/stats` | Statistiques globales | Admin |
| POST | `/admin/events` | Créer un event (admin) | Admin |
| PATCH | `/admin/spots/{spot_id}/verify` | Valider un spot | Admin |
| GET | `/admin/photos/review` | Photos en attente de modération | Admin |
| PATCH | `/admin/photos/{photo_id}/moderate` | Approuver/rejeter | Admin |
| GET | `/admin/matching-config` | Tous les poids, groupés par catégorie | Admin |
| PATCH | `/admin/matching-config/{key}` | Modifier un poids | Admin |
| POST | `/admin/matching-config/reset/{key}` | Remettre un poids au default | Admin |
| POST | `/admin/matching-config/reset-all` | Reset tous les poids | Admin |
| GET | `/admin/matching-config/audit` | Historique des modifications | Admin |
| POST | `/admin/matching/trigger-batch` | Relancer le batch manuellement | Admin |
| POST | `/admin/matching/trigger-single/{user_id}` | Regénérer le feed d'un user | Admin |

---

## 6. Matching engine — implémentation détaillée

### 6.1 Pipeline orchestrateur

```python
# app/services/matching_engine/pipeline.py
import numpy as np
from datetime import datetime, timedelta
from uuid import UUID

from app.services.matching_engine.hard_filters import apply_hard_filters
from app.services.matching_engine.geo_scorer import compute_geo_scores
from app.services.matching_engine.lifestyle_scorer import compute_lifestyle_scores
from app.services.matching_engine.behavior_scorer import get_behavior_multipliers
from app.services.matching_engine.corrections import (
    inject_wildcards,
    apply_new_user_boost,
    ensure_minimum_visibility,
    shuffle_feed,
)
from app.services.matching_engine.weights import get_adaptive_weights
from app.core.constants import MATCHING_FEED_SIZE, MATCHING_WILDCARD_COUNT


async def generate_feed_for_user(
    user_id: UUID,
    db_session,
    redis_client,
) -> dict:
    """
    Pipeline complet L1 → L5.
    Appelé par le batch nocturne pour chaque utilisateur actif.
    Retourne le feed du jour : liste ordonnée de profile_ids.
    """

    # ── L0 : Récupérer les données de l'utilisateur ──
    user = await get_user_with_full_profile(user_id, db_session)
    if not user or not user.is_active or not user.is_visible:
        return {"profile_ids": [], "wildcards": [], "new_users": []}

    account_age_days = (datetime.utcnow() - user.created_at).days

    # ── L1 : Filtres durs ──
    # Retourne les IDs des candidats qui passent tous les filtres binaires
    candidate_ids = await apply_hard_filters(
        user=user,
        db_session=db_session,
    )
    # Résultat : ~15-30% du pool actif de la ville

    if len(candidate_ids) == 0:
        return {"profile_ids": [], "wildcards": [], "new_users": []}

    # ── L2 : Scores géo ──
    # Charger le graphe de proximité de la ville en cache mémoire
    # (une seule fois par batch, pas à chaque utilisateur)
    from app.services.matching_engine.geo_scorer import load_proximity_cache
    await load_proximity_cache(user.city_id, db_session)

    # Calcule un score 0-100 pour chaque candidat
    # Intègre : exact match quartiers + soft match proximité +
    # "interested" quartiers + spots communs + fidélité + fraîcheur
    geo_scores = await compute_geo_scores(
        user=user,
        candidate_ids=candidate_ids,
        db_session=db_session,
    )
    # Résultat : {candidate_id: float_score, ...}

    # ── L3 : Scores lifestyle ──
    lifestyle_scores = await compute_lifestyle_scores(
        user=user,
        candidate_ids=candidate_ids,
        db_session=db_session,
    )

    # ── L4 : Multiplicateurs comportementaux ──
    behavior_multipliers = await get_behavior_multipliers(
        candidate_ids=candidate_ids,
        redis_client=redis_client,
    )

    # ── Combinaison pondérée ──
    geo_w, life_w, behav_w = get_adaptive_weights(account_age_days)

    final_scores = {}
    for cid in candidate_ids:
        base_score = (
            geo_w * geo_scores.get(cid, 0)
            + life_w * lifestyle_scores.get(cid, 0)
        )
        multiplier = behavior_multipliers.get(cid, 1.0)
        final_scores[cid] = base_score * multiplier

    # Trier par score décroissant
    sorted_candidates = sorted(
        final_scores.items(), key=lambda x: x[1], reverse=True
    )

    # ── L5 : Corrections ──
    # Prendre les N meilleurs
    feed_size = MATCHING_FEED_SIZE  # 12
    wildcard_count = MATCHING_WILDCARD_COUNT  # 2

    top_n = feed_size - wildcard_count - 2  # 8 slots pour le top
    top_profiles = [cid for cid, _ in sorted_candidates[:top_n]]

    # Wildcards : scores géo élevés mais lifestyle différent
    wildcards = await inject_wildcards(
        user=user,
        top_profiles=top_profiles,
        all_candidates=sorted_candidates,
        geo_scores=geo_scores,
        lifestyle_scores=lifestyle_scores,
        count=wildcard_count,
        db_session=db_session,
    )

    # Boost nouveaux profils
    new_users = await apply_new_user_boost(
        user=user,
        remaining_candidates=[
            cid for cid, _ in sorted_candidates
            if cid not in top_profiles and cid not in wildcards
        ],
        max_count=2,
        db_session=db_session,
    )

    # Assembler le feed
    feed_ids = top_profiles + wildcards + new_users

    # Garantie de visibilité minimum
    feed_ids = await ensure_minimum_visibility(
        feed_ids=feed_ids,
        user=user,
        db_session=db_session,
        redis_client=redis_client,
    )

    # Shuffle déterministe
    feed_ids = shuffle_feed(
        feed_ids=feed_ids,
        user_id=user_id,
        date=datetime.utcnow().date(),
    )

    return {
        "profile_ids": feed_ids,
        "wildcards": wildcards,
        "new_users": new_users,
    }
```

### 6.2 L1 — Filtres durs

```python
# app/services/matching_engine/hard_filters.py
from datetime import datetime, timedelta
from sqlalchemy import select, and_, not_, exists
from app.models import User, Profile, Block, Match, ContactBlacklist, FeedCache
from app.core.constants import MATCHING_SKIP_COOLDOWN_DAYS


async def apply_hard_filters(user, db_session) -> list[UUID]:
    """
    Retourne les IDs des candidats valides.
    Requête SQL unique avec tous les filtres.
    """
    now = datetime.utcnow()
    skip_cutoff = now - timedelta(days=MATCHING_SKIP_COOLDOWN_DAYS)
    active_cutoff = now - timedelta(days=7)

    profile = user.profile

    stmt = (
        select(User.id)
        .join(Profile, Profile.user_id == User.id)
        .where(
            and_(
                # Même ville
                User.city_id == user.city_id,

                # Pas moi
                User.id != user.id,

                # Actif et visible
                User.is_active == True,
                User.is_visible == True,
                User.is_banned == False,
                User.last_active_at >= active_cutoff,

                # Selfie vérifié obligatoire
                User.is_selfie_verified == True,

                # Genre compatible (bidirectionnel)
                _gender_match(profile, Profile),

                # Âge compatible (bidirectionnel)
                _age_match(profile, Profile),

                # Intentions compatibles
                _intention_compatible(profile.intention, Profile.intention),

                # Pas bloqué (dans les deux sens)
                not_(exists(
                    select(Block.id).where(
                        and_(Block.blocker_id == user.id, Block.blocked_id == User.id)
                    )
                )),
                not_(exists(
                    select(Block.id).where(
                        and_(Block.blocker_id == User.id, Block.blocked_id == user.id)
                    )
                )),

                # Pas dans la blacklist de contacts
                not_(exists(
                    select(ContactBlacklist.id).where(
                        and_(
                            ContactBlacklist.user_id == user.id,
                            ContactBlacklist.phone_hash == User.phone_hash,
                        )
                    )
                )),

                # Pas déjà montré et skippé dans les 30 derniers jours
                not_(exists(
                    select(Match.id).where(
                        and_(
                            Match.user_a_id == user.id,
                            Match.user_b_id == User.id,
                            Match.status == "skipped",
                            Match.created_at >= skip_cutoff,
                        )
                    )
                )),

                # Pas déjà matché (actif ou expiré)
                not_(exists(
                    select(Match.id).where(
                        and_(
                            (
                                (Match.user_a_id == user.id) & (Match.user_b_id == User.id)
                                | (Match.user_a_id == User.id) & (Match.user_b_id == user.id)
                            ),
                            Match.status.in_(["pending", "matched"]),
                        )
                    )
                )),
            )
        )
    )

    result = await db_session.execute(stmt)
    return [row[0] for row in result.fetchall()]
```

### 6.3 L2 — Score géo (détaillé, avec proximité + interested)

```python
# app/services/matching_engine/geo_scorer.py
import math
from collections import defaultdict
from uuid import UUID

from app.core.constants import (
    GEO_W_QUARTIER, GEO_W_SPOT, GEO_W_FIDELITY, GEO_W_FRESHNESS,
    QUARTIER_RELATION_WEIGHTS, FRESHNESS_DECAY_HALFLIFE,
    SPOT_SOCIAL_WEIGHTS, PROXIMITY_CLOSE_THRESHOLD,
)


# ── Cache en mémoire du graphe de proximité ──
# Chargé au démarrage du matching batch, pas à chaque calcul
# Format : {(quartier_a_id, quartier_b_id): proximity_score}
_proximity_cache: dict[tuple[UUID, UUID], float] = {}


async def load_proximity_cache(city_id: UUID, db_session):
    """
    Charge le graphe de proximité d'une ville en mémoire.
    Appelé une fois au début du batch nocturne.
    """
    global _proximity_cache
    rows = await db_session.execute(
        select(QuartierProximity.quartier_a_id,
               QuartierProximity.quartier_b_id,
               QuartierProximity.proximity_score)
        .join(Quartier, Quartier.id == QuartierProximity.quartier_a_id)
        .where(Quartier.city_id == city_id)
    )
    _proximity_cache = {}
    for a_id, b_id, score in rows.fetchall():
        _proximity_cache[(a_id, b_id)] = score
        _proximity_cache[(b_id, a_id)] = score  # Symétrique


def get_proximity(q1: UUID, q2: UUID) -> float:
    """Retourne le score de proximité entre deux quartiers. 1.0 si identiques."""
    if q1 == q2:
        return 1.0
    return _proximity_cache.get((q1, q2), 0.0)


async def compute_geo_scores(user, candidate_ids, db_session) -> dict:
    """
    Pour chaque candidat, calcule un score géo 0 → 100.
    Intègre :
    - Exact match sur les quartiers (Jaccard pondéré classique)
    - Soft match via le graphe de proximité (quartiers voisins)
    - Le type "interested" (préférence personnelle cross-ville)
    - Spots communs pondérés par catégorie sociale
    - Bonus fidélité sur les spots communs
    - Fraîcheur des check-ins
    """
    # Charger les données de l'utilisateur
    # Format : {quartier_id: relation_type}
    user_quartiers = {
        uq.quartier_id: uq.relation_type
        for uq in user.user_quartiers
    }
    user_spots = {
        us.spot_id: us
        for us in user.user_spots
    }

    # Séparer les quartiers "interested" des quartiers physiques
    user_physical_quartiers = {
        qid: rtype for qid, rtype in user_quartiers.items()
        if rtype != "interested"
    }
    user_interested_quartiers = {
        qid: rtype for qid, rtype in user_quartiers.items()
        if rtype == "interested"
    }

    # Charger en batch les données des candidats
    candidates_quartiers = await load_candidates_quartiers(candidate_ids, db_session)
    candidates_spots = await load_candidates_spots(candidate_ids, db_session)

    scores = {}
    for cid in candidate_ids:
        cq = candidates_quartiers.get(cid, {})
        cs = candidates_spots.get(cid, {})

        # Séparer les quartiers physiques du candidat aussi
        cq_physical = {qid: rt for qid, rt in cq.items() if rt != "interested"}

        q_score = _quartier_score_with_proximity(
            user_physical=user_physical_quartiers,
            user_interested=user_interested_quartiers,
            candidate_physical=cq_physical,
        )
        s_score = _spot_overlap(user_spots, cs)
        f_score = _fidelity_bonus(user_spots, cs)
        fr_score = _freshness_score(user_spots, cs)

        raw = (
            GEO_W_QUARTIER * q_score
            + GEO_W_SPOT * s_score
            + GEO_W_FIDELITY * f_score
            + GEO_W_FRESHNESS * fr_score
        )
        scores[cid] = min(100.0, raw * 100)

    return scores


def _quartier_score_with_proximity(
    user_physical: dict,
    user_interested: dict,
    candidate_physical: dict,
) -> float:
    """
    Score quartier en 3 passes :

    Passe 1 — Exact match (Jaccard pondéré classique)
    Quartiers identiques entre les deux users.
    Ex: tous les deux à Tokoin → full weight.

    Passe 2 — Soft match via proximité
    Quartiers pas identiques mais voisins dans le graphe.
    Ex: user à Bè, candidat à Tokoin (proximity=0.82)
    → contribution = weight × proximity_score
    On ne compte que si proximity > PROXIMITY_CLOSE_THRESHOLD (0.40)
    pour éviter le bruit des quartiers éloignés.

    Passe 3 — Interested match
    Le type "interested" de l'utilisateur matche un quartier physique
    du candidat. C'est un exact match mais avec le poids réduit (0.8x).
    Résout le cas Agoè → Bè où la proximité est trop faible.
    """
    if not candidate_physical:
        return 0.0
    if not user_physical and not user_interested:
        return 0.0

    total_score = 0.0
    matched_candidate_quartiers = set()  # Pour ne pas double-compter

    # ── Passe 1 : Exact match ──
    exact_common = set(user_physical.keys()) & set(candidate_physical.keys())
    for qid in exact_common:
        w_user = QUARTIER_RELATION_WEIGHTS[user_physical[qid]]
        w_cand = QUARTIER_RELATION_WEIGHTS[candidate_physical[qid]]
        total_score += max(w_user, w_cand)
        matched_candidate_quartiers.add(qid)

    # ── Passe 2 : Soft match via proximité ──
    for user_qid, user_rtype in user_physical.items():
        if user_qid in exact_common:
            continue  # Déjà compté en exact
        for cand_qid, cand_rtype in candidate_physical.items():
            if cand_qid in matched_candidate_quartiers:
                continue  # Déjà matché

            proximity = get_proximity(user_qid, cand_qid)
            if proximity < PROXIMITY_CLOSE_THRESHOLD:
                continue  # Trop loin, on ignore

            w_user = QUARTIER_RELATION_WEIGHTS[user_rtype]
            w_cand = QUARTIER_RELATION_WEIGHTS[cand_rtype]
            contribution = max(w_user, w_cand) * proximity

            # Si les quartiers sont proches mais pas voisins (0.40-0.65),
            # on réduit encore de moitié pour ne pas surpondérer
            if proximity < 0.65:
                contribution *= 0.5

            total_score += contribution
            matched_candidate_quartiers.add(cand_qid)
            break  # Un seul soft match par quartier user

    # ── Passe 3 : Interested match ──
    for int_qid in user_interested:
        if int_qid in matched_candidate_quartiers:
            continue  # Déjà matché par une autre passe
        if int_qid in candidate_physical:
            # Le user est "interested" par un quartier OÙ le candidat est physiquement
            w_cand = QUARTIER_RELATION_WEIGHTS[candidate_physical[int_qid]]
            w_interested = QUARTIER_RELATION_WEIGHTS["interested"]
            # On prend le min car c'est "interested" qui limite
            total_score += min(w_interested, w_cand)
            matched_candidate_quartiers.add(int_qid)

    # ── Normalisation ──
    # Diviser par le nombre total de quartiers dans l'union
    all_quartiers = (
        set(user_physical.keys())
        | set(user_interested.keys())
        | set(candidate_physical.keys())
    )
    if not all_quartiers:
        return 0.0

    return min(1.0, total_score / len(all_quartiers))


def _spot_overlap(user_spots: dict, candidate_spots: dict) -> float:
    """
    Nombre de spots communs, pondéré par le social_weight de la catégorie.
    Un maquis en commun vaut plus qu'un coworking en commun.
    """
    if not user_spots or not candidate_spots:
        return 0.0

    common_spot_ids = set(user_spots.keys()) & set(candidate_spots.keys())
    if not common_spot_ids:
        return 0.0

    weighted_sum = 0.0
    for sid in common_spot_ids:
        spot = user_spots[sid].spot
        social_w = SPOT_SOCIAL_WEIGHTS.get(spot.category, 0.8)
        weighted_sum += social_w

    max_possible = max(len(user_spots), len(candidate_spots))
    return min(1.0, weighted_sum / max_possible)


def _fidelity_bonus(user_spots: dict, candidate_spots: dict) -> float:
    """
    Bonus basé sur les niveaux de fidélité des spots communs.
    Deux habitués = multiplicateur élevé, deux déclarés = multiplicateur neutre.
    Moyenne géométrique pour récompenser les paires élevées.
    """
    common = set(user_spots.keys()) & set(candidate_spots.keys())
    if not common:
        return 0.0

    bonus_sum = 0.0
    for sid in common:
        my_score = user_spots[sid].fidelity_score
        their_score = candidate_spots[sid].fidelity_score
        bonus_sum += math.sqrt(my_score * their_score)

    return min(1.0, bonus_sum / len(common))


def _freshness_score(user_spots: dict, candidate_spots: dict) -> float:
    """
    Score basé sur la fraîcheur des check-ins sur les spots communs.
    Decay exponentiel : score = exp(-0.693 * jours / halflife)
    Un check-in récent = signal fort. Un vieux tag = signal faible.
    """
    common = set(user_spots.keys()) & set(candidate_spots.keys())
    if not common:
        return 0.5  # Score neutre si pas de spots communs

    from datetime import datetime, timezone
    now = datetime.now(timezone.utc)
    freshness_sum = 0.0

    for sid in common:
        their_last = candidate_spots[sid].last_checkin_at
        if their_last is None:
            freshness_sum += 0.3  # Déclaré sans check-in
        else:
            days = (now - their_last).days
            decay = math.exp(-0.693 * days / FRESHNESS_DECAY_HALFLIFE)
            freshness_sum += decay

    return min(1.0, freshness_sum / len(common))
```

### 6.4 L5 — Corrections

```python
# app/services/matching_engine/corrections.py
import hashlib
import struct
from datetime import datetime, timedelta
from app.core.constants import NEW_USER_BOOST


async def inject_wildcards(
    user, top_profiles, all_candidates, geo_scores, lifestyle_scores, count, db_session
) -> list:
    """
    Sélectionne des profils avec un score géo élevé mais un profil lifestyle
    différent de l'historique de likes de l'utilisateur.
    """
    # Charger l'historique de likes récent (30 derniers jours)
    liked_profile_ids = await get_recent_liked_profiles(user.id, db_session, days=30)
    if not liked_profile_ids:
        # Pas d'historique → pas de wildcards, juste prendre les suivants
        remaining = [cid for cid, _ in all_candidates if cid not in top_profiles]
        return remaining[:count]

    # Calculer le "profil type" liké : moyenne des tags, secteurs, etc.
    liked_profile_vector = await compute_liked_profile_vector(liked_profile_ids, db_session)

    # Chercher des candidats avec :
    # 1. geo_score > médiane (ancrage géo fort)
    # 2. lifestyle_score < percentile 30 par rapport au profil type liké
    # = des gens proches géographiquement mais différents en style de vie
    geo_median = sorted(geo_scores.values())[len(geo_scores) // 2] if geo_scores else 0

    wildcard_pool = []
    for cid, _ in all_candidates:
        if cid in top_profiles:
            continue
        if geo_scores.get(cid, 0) < geo_median:
            continue
        # Calculer la distance au "profil type"
        distance = await compute_profile_distance(cid, liked_profile_vector, db_session)
        if distance > 0.5:  # Suffisamment différent
            wildcard_pool.append((cid, distance))

    # Trier par distance décroissante (les plus différents en premier)
    wildcard_pool.sort(key=lambda x: x[1], reverse=True)
    return [cid for cid, _ in wildcard_pool[:count]]


async def apply_new_user_boost(user, remaining_candidates, max_count, db_session) -> list:
    """
    Sélectionne des nouveaux inscrits (< 10 jours) parmi les candidats restants.
    """
    now = datetime.utcnow()
    new_users = []

    for cid in remaining_candidates:
        candidate = await get_user_basic(cid, db_session)
        if not candidate:
            continue

        age_days = (now - candidate.created_at).days

        # Vérifier qu'il n'a pas déjà reset son compte
        if candidate.account_created_count > 1:
            continue

        for (day_min, day_max), boost_factor in NEW_USER_BOOST.items():
            if day_min <= age_days <= day_max:
                new_users.append(cid)
                break

        if len(new_users) >= max_count:
            break

    return new_users


def shuffle_feed(feed_ids: list, user_id, date) -> list:
    """
    Shuffle déterministe : même résultat pour le même user le même jour.
    Empêche le refresh-pour-reorder.
    """
    seed_str = f"{user_id}:{date.isoformat()}"
    seed_bytes = hashlib.sha256(seed_str.encode()).digest()
    seed_int = struct.unpack("<I", seed_bytes[:4])[0]

    import random
    rng = random.Random(seed_int)
    shuffled = feed_ids.copy()
    rng.shuffle(shuffled)
    return shuffled
```

---

## 7. Celery tasks

```python
# app/tasks/celery_app.py
from celery import Celery
from celery.schedules import crontab
from app.config import get_settings

settings = get_settings()

celery_app = Celery(
    "flaam",
    broker=settings.celery_broker_url,
    backend=settings.celery_result_backend,
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_routes={
        "app.tasks.image_tasks.*": {"queue": "images"},
        "app.tasks.matching_tasks.*": {"queue": "matching"},
        "app.tasks.notification_tasks.*": {"queue": "notifications"},
        "app.tasks.moderation_tasks.*": {"queue": "moderation"},
        "app.tasks.cleanup_tasks.*": {"queue": "cleanup"},
    },
    beat_schedule={
        # ── Matching batch : tous les jours à 3h UTC ──
        "generate-daily-feeds": {
            "task": "app.tasks.matching_tasks.generate_all_feeds",
            "schedule": crontab(hour=3, minute=0),
        },
        # ── Expiration des matchs sans message : tous les jours à 4h ──
        "expire-stale-matches": {
            "task": "app.tasks.cleanup_tasks.expire_stale_matches",
            "schedule": crontab(hour=4, minute=0),
        },
        # ── Persistance des behavior_multipliers Redis → DB : toutes les heures ──
        "persist-behavior-scores": {
            "task": "app.tasks.matching_tasks.persist_behavior_scores",
            "schedule": crontab(minute=0),  # Toutes les heures pile
        },
        # ── Digest hebdomadaire : dimanche 18h UTC ──
        "weekly-digest": {
            "task": "app.tasks.notification_tasks.send_weekly_digest",
            "schedule": crontab(hour=18, minute=0, day_of_week="sunday"),
        },
        # ── Nettoyage photos orphelines : tous les jours à 5h ──
        "cleanup-orphan-photos": {
            "task": "app.tasks.cleanup_tasks.cleanup_orphan_photos",
            "schedule": crontab(hour=5, minute=0),
        },
        # ── Purge behavior_logs > 90 jours : chaque lundi à 6h ──
        "purge-old-behavior-logs": {
            "task": "app.tasks.cleanup_tasks.purge_old_behavior_logs",
            "schedule": crontab(hour=6, minute=0, day_of_week="monday"),
        },
        # ── Recalcul des stats spots (total_checkins, total_users) : chaque nuit ──
        "refresh-spot-stats": {
            "task": "app.tasks.analytics_tasks.refresh_spot_stats",
            "schedule": crontab(hour=2, minute=30),
        },
        # ── Vérification des abonnements expirés : toutes les heures ──
        "check-expired-subscriptions": {
            "task": "app.tasks.cleanup_tasks.check_expired_subscriptions",
            "schedule": crontab(minute=15),
        },
    },
)
```

```python
# app/tasks/image_tasks.py
from app.tasks.celery_app import celery_app


@celery_app.task(bind=True, max_retries=3, default_retry_delay=30)
def process_photo_upload(self, user_id: str, photo_id: str, file_path: str):
    """
    Pipeline de traitement d'une photo uploadée :
    1. Resize → 3 tailles (original 1200px, medium 600px, thumb 150px)
    2. Convertir en WebP (qualité 82%)
    3. Strip EXIF (vie privée)
    4. Calculer le hash SHA-256 (déduplication)
    5. Upload vers R2 (3 fichiers)
    6. Lancer la modération NSFW
    7. Mettre à jour la DB (URLs, dimensions, statut)
    """
    ...


@celery_app.task(bind=True, max_retries=2, default_retry_delay=10)
def moderate_photo(self, photo_id: str):
    """
    Modération automatique :
    1. NSFW detection (modèle ONNX)
    2. Face detection (au moins 1 visage visible)
    3. Si score NSFW > 0.7 → rejected
    4. Si score NSFW 0.4-0.7 → manual_review
    5. Si pas de visage détecté → manual_review
    6. Sinon → approved
    """
    ...


@celery_app.task
def verify_selfie(user_id: str, selfie_path: str, pose_requested: str):
    """
    Vérification selfie :
    1. Face detection
    2. Liveness check (le modèle vérifie que la pose correspond)
    3. Comparaison avec les photos de profil (même personne ?)
    4. Si tout OK → user.is_selfie_verified = True
    """
    ...
```

### 16.1b Modération photo — 3 modes switchables par ENV

La modération photo tourne dans un des 3 modes, pilotables par la variable d'environnement `PHOTO_MODERATION_MODE` :

| Mode | Valeur ENV | Usage | Coût |
|------|-----------|-------|------|
| `manual` | `PHOTO_MODERATION_MODE=manual` | MVP, volumes < 100 photos/jour | 0 (temps admin) |
| `onnx` | `PHOTO_MODERATION_MODE=onnx` | Production, volumes > 100/jour | 0 (CPU serveur) |
| `external` | `PHOTO_MODERATION_MODE=external` | Scale, volumes > 1000/jour | ~$0.001/photo (Sightengine) |
| `off` | `PHOTO_MODERATION_MODE=off` | Tests/dev uniquement | — |

**Comportement commun aux 3 modes :**
- Toute photo uploadée passe par `moderation_status="pending"` à la création
- La photo est **visible quand même** pendant qu'elle est pending (on ne bloque pas l'UX)
- Si rejected, la photo est masquée du profil mais gardée en DB 7 jours (appel possible)
- L'admin peut toujours override manuellement via `PATCH /admin/photos/{id}/moderation`

**Mode `manual` :**
- Pas de worker Celery, pas de call API
- Les photos restent `pending` jusqu'à action admin
- Admin voit la queue via `GET /admin/photos/pending` et approve/reject à la main
- Utilisé pour les premiers mois du lancement (MVP Lomé)

**Mode `onnx` :**
- Worker Celery `moderate_photo_onnx` charge 2 modèles en mémoire au démarrage :
  - `NSFW_MODEL_PATH` : modèle de détection NSFW (NSFW_detector.onnx, ~50 MB)
  - `FACE_MODEL_PATH` : face detection (YuNet ou UltraFace, ~2 MB)
- Traite la queue en async
- Seuils dans les constantes : `NSFW_THRESHOLD_REJECT=0.7`, `NSFW_THRESHOLD_REVIEW=0.4`
- Si NSFW > 0.7 → `rejected`, raison "nsfw"
- Si NSFW 0.4-0.7 → `manual_review` (fallback admin)
- Si pas de visage détecté → `manual_review`, raison "no_face"
- Sinon → `approved`

**Mode `external` :**
- Worker Celery `moderate_photo_external` appelle l'API Sightengine ou Google Vision
- Même logique de seuils que `onnx`
- Retries automatiques si l'API est down (max 3, backoff exponentiel)
- Si > 3 échecs consécutifs : fallback vers `manual_review`

**Mode `off` :**
- Toute photo est auto-approuvée (`moderation_status="approved"`)
- Uniquement pour les tests automatisés et le dev local sans modèles

**Config .env pour les 3 modes :**

```bash
# Mode de modération (manual | onnx | external | off)
PHOTO_MODERATION_MODE=manual

# Mode onnx : chemins vers les modèles ONNX
NSFW_MODEL_PATH=/models/nsfw_detector.onnx
FACE_MODEL_PATH=/models/yunet_face.onnx

# Mode external : credentials API
SIGHTENGINE_USER=xxx
SIGHTENGINE_SECRET=xxx
SIGHTENGINE_MODELS=nudity-2.0,face-attributes

# Seuils communs aux modes onnx et external
NSFW_THRESHOLD_REJECT=0.7
NSFW_THRESHOLD_REVIEW=0.4
```

**Stratégie de migration :**
- Lancement Lomé : `manual` pendant 2-3 mois
- 500+ inscriptions/mois : migration vers `onnx` (télécharger les modèles dans l'image Docker)
- 2000+ inscriptions/mois : évaluer `external` si le CPU serveur sature

**Code du dispatcher :**

```python
# app/services/photo_moderation_service.py
from app.core.config import get_settings

settings = get_settings()


async def moderate_photo(photo_id: str) -> None:
    """
    Dispatcher qui appelle la bonne implémentation selon PHOTO_MODERATION_MODE.
    """
    mode = settings.photo_moderation_mode
    
    if mode == "off":
        await _auto_approve(photo_id)
    elif mode == "manual":
        # Rien à faire, la photo reste pending jusqu'à action admin
        return
    elif mode == "onnx":
        from app.tasks.photo_tasks import moderate_photo_onnx
        moderate_photo_onnx.delay(photo_id)
    elif mode == "external":
        from app.tasks.photo_tasks import moderate_photo_external
        moderate_photo_external.delay(photo_id)
    else:
        raise ValueError(f"Unknown PHOTO_MODERATION_MODE: {mode}")
```

Le même dispatcher est utilisé pour `verify_selfie` avec les mêmes modes.


```python
# app/tasks/matching_tasks.py
from app.tasks.celery_app import celery_app


@celery_app.task
def generate_all_feeds():
    """
    Batch nocturne : génère le feed du lendemain pour tous les utilisateurs actifs.

    Pipeline :
    1. Récupérer toutes les villes actives
    2. Pour chaque ville :
       a. Charger le graphe de proximité en cache mémoire (1 fois par ville)
       b. Récupérer les users actifs de la ville (last_active < 7j)
       c. Pour chaque user, lancer generate_feed_for_user
       d. Stocker le résultat dans Redis (feed:{user_id}, TTL 24h) et FeedCache (DB)
    3. Logger les métriques : nb users traités, temps total, score moyen

    Le traitement par ville permet de :
    - Charger le graphe de proximité une seule fois par ville
    - Paralléliser par ville si nécessaire (chaque ville = un sub-task Celery)
    - Limiter la mémoire (un seul graphe en cache à la fois)

    Temps estimé :
    - Lomé (~2000 users actifs) : ~3 min
    - Abidjan (~10000 users actifs) : ~15 min
    - Toutes les villes : < 30 min (séquentiel) ou < 10 min (parallèle)
    """
    ...


@celery_app.task
def generate_single_feed(user_id: str):
    """
    Génère le feed pour un seul utilisateur.
    Appelé par le batch nocturne ou en recalcul incrémental mid-day.

    Recalcul incrémental déclenché quand :
    - Un nouveau match modifie le pool (un candidat précédemment affiché est maintenant matché)
    - L'utilisateur change ses quartiers ou ses préférences
    - L'utilisateur ajoute un quartier "interested" (nouveau signal fort)
    - Un premium active le Mode Voyageur (change de ville temporairement)
    """
    ...


@celery_app.task
def update_behavior_on_action(user_id: str, action_type: str, metadata: dict):
    """
    Appelé à chaque action utilisateur (like, skip, message, etc.)
    Met à jour le behavior_multiplier dans Redis en temps réel.

    Pipeline :
    1. Log l'action dans behavior_logs (DB, async via INSERT)
    2. Recalcule les 4 composantes du multiplier :
       - response_quality : ratio réponses / matchs reçus (30 derniers jours)
       - selectivity_index : ratio likes / profils vus (bell curve)
       - profile_richness : score de complétion du profil
       - conversation_depth : nb conversations > 5 messages / total matchs
    3. Multiplie les 4 composantes → behavior_multiplier (0.6 → 1.4)
    4. Stocke dans Redis key "behavior:{user_id}" (pas de TTL, persisté par cron)
    5. Si le nouveau multiplier diffère de > 0.1 de l'ancien :
       déclencher un recalcul incrémental du feed

    Actions trackées :
    - profile_view → durée + scroll complet
    - like → prompt liké
    - skip → raison optionnelle
    - message_sent → longueur + type
    - meetup_proposed → signal très positif
    - unmatch → signal négatif
    - report → signal négatif fort
    - checkin → spot_id (met aussi à jour UserSpot.fidelity)
    """
    ...


@celery_app.task
def persist_behavior_scores():
    """
    Cron toutes les heures : persiste les scores Redis → DB.
    Pour survivre à un restart Redis.

    Pipeline :
    1. SCAN Redis pour toutes les clés "behavior:*"
    2. Pour chaque clé, lire le multiplier
    3. UPDATE profiles SET behavior_multiplier = X WHERE user_id = Y
    4. Batch de 500 UPDATEs par transaction
    """
    ...
```

---

## 8. Docker Compose

```yaml
# docker-compose.yml
services:
  api:
    build: .
    command: >
      uvicorn app.main:app
      --host 0.0.0.0
      --port 8000
      --workers 4
      --loop uvloop
      --http httptools
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    volumes:
      - ./models:/models:ro
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2"

  celery-worker-images:
    build: .
    command: celery -A app.tasks.celery_app worker -Q images -c 2 --loglevel=info
    env_file: .env
    depends_on: [db, redis, rabbitmq]
    volumes:
      - ./models:/models:ro
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G

  celery-worker-matching:
    build: .
    command: celery -A app.tasks.celery_app worker -Q matching -c 2 --loglevel=info
    env_file: .env
    depends_on: [db, redis, rabbitmq]
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 2G

  celery-worker-notifications:
    build: .
    command: celery -A app.tasks.celery_app worker -Q notifications -c 2 --loglevel=info
    env_file: .env
    depends_on: [db, redis, rabbitmq]
    restart: unless-stopped

  celery-worker-general:
    build: .
    command: celery -A app.tasks.celery_app worker -Q moderation,cleanup -c 1 --loglevel=info
    env_file: .env
    depends_on: [db, redis, rabbitmq]
    volumes:
      - ./models:/models:ro
    restart: unless-stopped

  celery-beat:
    build: .
    command: celery -A app.tasks.celery_app beat --loglevel=info
    env_file: .env
    depends_on: [rabbitmq]
    restart: unless-stopped

  db:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_USER: flaam
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: flaam
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U flaam"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 4G

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: flaam
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
      RABBITMQ_DEFAULT_VHOST: flaam
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmqctl", "status"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on: [api]
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
```

---

## 9. Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    libgeos-dev \
    libproj-dev \
    gdal-bin \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 10. Point d'entrée FastAPI

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import sentry_sdk

from app.config import get_settings
from app.api.router import api_router
from app.api.ws.chat import ws_router
from app.db.session import engine
from app.db.redis import redis_pool
from app.core.exceptions import register_exception_handlers


settings = get_settings()

if settings.sentry_dsn:
    sentry_sdk.init(dsn=settings.sentry_dsn, traces_sample_rate=0.1)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await redis_pool.initialize()
    yield
    # Shutdown
    await engine.dispose()
    await redis_pool.close()


app = FastAPI(
    title="Flaam API",
    version="1.0.0",
    docs_url="/docs" if settings.app_debug else None,
    redoc_url=None,
    lifespan=lifespan,
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins.split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routes
app.include_router(api_router, prefix=settings.api_v1_prefix)
app.include_router(ws_router)

# Exception handlers
register_exception_handlers(app)


@app.get("/health")
async def health():
    return {"status": "ok", "version": "1.0.0"}
```

---

## 11. Résumé des index de la base de données

| Table | Index | Type | Raison |
|-------|-------|------|--------|
| users | ix_users_city_active | B-tree (city_id, last_active_at) | L1 : filtre principal |
| users | ix_users_city_visible | B-tree (city_id, is_visible, is_active) | L1 : filtre visibilité |
| users | phone_hash | UNIQUE | Login |
| profiles | profiles_user_id_key | UNIQUE | 1:1 user→profile |
| profiles | GIN(tags) | GIN | L3 : recherche tags |
| photos | ix_photos_user | B-tree (user_id) | Chargement profil |
| spots | ix_spots_location | GIST | Check-in : nearest spot |
| spots | ix_spots_city_category | B-tree | Recherche spots |
| user_spots | uq_user_spot | UNIQUE (user_id, spot_id) | Pas de doublons |
| user_spots | ix_user_spots_spot | B-tree (spot_id) | L2 : spots communs |
| user_quartiers | uq_user_quartier | UNIQUE (user_id, quartier_id, type) | Pas de doublons |
| user_quartiers | ix_user_quartiers_user_type | B-tree (user_id, relation_type) | Filtrer par type (interested vs physical) |
| quartier_proximities | uq_quartier_proximity | UNIQUE (quartier_a_id, quartier_b_id) | Pas de doublons, paire ordonnée |
| quartier_proximities | ix_qp_quartier_a | B-tree (quartier_a_id) | Lookup rapide voisins d'un quartier |
| quartier_proximities | ix_qp_quartier_b | B-tree (quartier_b_id) | Lookup rapide voisins d'un quartier |
| matches | uq_match_pair | UNIQUE (user_a_id, user_b_id) | 1 match par paire |
| matches | ix_matches_expires | B-tree partiel (expires_at) WHERE status='matched' | Cron expiration |
| messages | ix_messages_match | B-tree (match_id, created_at) | Chargement historique |
| blocks | uq_block | UNIQUE (blocker_id, blocked_id) | Pas de doublons |
| behavior_logs | ix_behavior_user_type | B-tree (user_id, event_type) | Agrégation |
| behavior_logs | ix_behavior_created | B-tree (created_at) | Purge |
| feed_caches | uq_feed_user_date | UNIQUE (user_id, feed_date) | 1 feed/jour/user |
| events | ix_events_city_date | B-tree (city_id, starts_at) | Feed events |
| reports | ix_reports_status | B-tree (status) | Admin : pending reports |
| contact_blacklists | ix_contact_blacklist_phone | B-tree (phone_hash) | L1 : filtre contacts |

---

## 12. Résumé des flux de données critiques

### 12.1 Flux : check-in à un spot

```
Client                API                     Celery              Redis           DB
  │                    │                        │                   │              │
  ├─ POST /spots/me/   │                        │                   │              │
  │  {spot_id}/checkin │                        │                   │              │
  │  {lat, lng}        │                        │                   │              │
  │ ──────────────────>│                        │                   │              │
  │                    ├─ Vérifie distance      │                   │              │
  │                    │  (< 100m du spot)      │                   │              │
  │                    ├─ UPDATE user_spots     │                   │              │
  │                    │  SET checkin_count++   │                   │              │
  │                    │  + recalcul fidelity   │                   │──────────────>│
  │                    ├─ UPDATE spots          │                   │              │
  │                    │  SET total_checkins++  │                   │──────────────>│
  │                    ├─ INSERT behavior_log   │                   │              │
  │                    │  (type=checkin)        │                   │──────────────>│
  │                    ├─ Dispatch task ────────>│                   │              │
  │                    │  update_behavior       │                   │              │
  │                    │                        ├─ Recalcul         │              │
  │                    │                        │  multiplier       │              │
  │                    │                        ├─ SET behavior:uid─>│              │
  │<────── 200 ────────│                        │                   │              │
  │  {fidelity_level,  │                        │                   │              │
  │   level_upgraded}  │                        │                   │              │
```

### 12.2 Flux : like → match → ice-breaker

```
Client A              API                     Redis           DB              Client B
  │                    │                        │              │                │
  ├─ POST /feed/       │                        │              │                │
  │  {profile_id}/like │                        │              │                │
  │ ──────────────────>│                        │              │                │
  │                    ├─ Vérifie likes restants │              │                │
  │                    │  (GET rate:likes:uid)──>│              │                │
  │                    ├─ Check si B a déjà     │              │                │
  │                    │  liké A (SELECT match) │──────────────>│                │
  │                    │                        │              │                │
  │                    │  ┌─ SI pas de like ──────────────────────────────────┐  │
  │                    │  │  INSERT match       │              │              │  │
  │                    │  │  (status=pending)   │──────────────>│              │  │
  │                    │  │  DECR rate:likes:uid>│              │              │  │
  │<── 200 {liked} ────│  │                     │              │              │  │
  │                    │  └──────────────────────────────────────────────────┘  │
  │                    │                        │              │                │
  │                    │  ┌─ SI B avait liké A ────────────────────────────────┐│
  │                    │  │  UPDATE match       │              │                ││
  │                    │  │  SET status=matched │──────────────>│                ││
  │                    │  │  + matched_at=now   │              │                ││
  │                    │  │  + expires_at=+7j   │              │                ││
  │                    │  │                     │              │                ││
  │                    │  │  Génère ice-breaker │              │                ││
  │                    │  │  basé sur spots     │              │                ││
  │                    │  │  communs ou prompt   │              │                ││
  │                    │  │  liké               │              │                ││
  │                    │  │                     │              │                ││
  │                    │  │  INSERT message     │              │                ││
  │                    │  │  (type=system,      │──────────────>│                ││
  │                    │  │   ice-breaker)      │              │                ││
  │                    │  │                     │              │                ││
  │                    │  │  PUBLISH ws:user_b  │              │                ││
  │                    │  │  {type:new_match}──>│─── push ─────────────────────>││
  │                    │  │                     │              │                ││
  │                    │  │  Push notification  │              │                ││
  │                    │  │  → Celery task ─────────────────── FCM ──────────>  ││
  │<── 200 {matched,   │  │                     │              │                ││
  │   ice_breaker} ────│  └──────────────────────────────────────────────────┘│
  │                    │                        │              │                │
```

### 12.3 Flux : batch nocturne de génération des feeds

```
Celery Beat (3h UTC)
  │
  ├─ Trigger: generate_all_feeds
  │
  ├─ Pour chaque ville active :
  │   │
  │   ├─ Charger graphe de proximité en mémoire (QuartierProximity)
  │   │   → ~50 quartiers par ville = ~1225 paires = ~10KB en RAM
  │   │
  │   ├─ SELECT users actifs de la ville (last_active < 7j, is_visible=true)
  │   │
  │   ├─ Pour chaque user (parallélisable par chunk de 100) :
  │   │   │
  │   │   ├─ L1 : SQL filtres durs → pool de candidats (~15-30% du total)
  │   │   ├─ L2 : Geo score (exact + proximity + interested + spots)
  │   │   ├─ L3 : Lifestyle score (tags TF-IDF + intention matrice)
  │   │   ├─ L4 : Behavior multiplier (depuis Redis)
  │   │   ├─ Combinaison pondérée (poids adaptatifs par ancienneté)
  │   │   ├─ L5 : Wildcards + boost nouveaux + garantie visibilité + shuffle
  │   │   │
  │   │   ├─ SET Redis feed:{user_id} → liste ordonnée de profile_ids (TTL 24h)
  │   │   ├─ UPSERT DB FeedCache → backup persistant
  │   │   └─ UPDATE User.last_feed_generated_at
  │   │
  │   └─ Log métriques ville : nb users, temps, score moyen, % wildcards
  │
  └─ Log métriques globales + alerte Telegram si erreur
```

### 12.4 Flux : quartier "interested" → impact sur le feed

```
Exemple concret : Raouf vit à Agoè, ajoute Bè en "interested"

AVANT (sans interested) :
  Raouf (Agoè) ↔ Ama (Bè) :
    quartier_overlap = 0.0 (aucun quartier commun)
    proximity(Agoè, Bè) = 0.21 (< seuil 0.40, ignoré en soft match)
    → geo_score très bas → Ama n'apparaît quasi jamais dans le feed de Raouf

APRÈS (avec interested sur Bè) :
  Raouf (Agoè + interested:Bè) ↔ Ama (Bè) :
    Passe 1 : exact match → 0 (pas de quartier physique commun)
    Passe 2 : soft match → 0 (proximity 0.21 < seuil 0.40)
    Passe 3 : interested match → Bè est dans interested de Raouf
              ET Bè est un quartier physique d'Ama (lives)
              → contribution = min(0.8, 2.0) = 0.8
    Total quartier_score = 0.8 / 3 quartiers union = 0.27
    → geo_score monte significativement → Ama remonte dans le feed

  Et si en plus Raouf fréquente le Café 21 à Bè (spot commun) :
    spot_overlap s'ajoute → geo_score encore plus fort
    → Ama apparaît dans les top 8 du feed
```


---

# PARTIE 2 — Sections critiques, production, et avancées

---

## 13. Onboarding state machine (CRITIQUE)

```python
# app/core/onboarding.py
"""
Machine à états de l'onboarding. Chaque nouvel utilisateur passe par
ces étapes dans l'ordre. Certaines sont bloquantes (impossible de
continuer sans les compléter), d'autres sont skippables.

L'état courant est stocké dans Redis (onboarding:{user_id}) et persisté
en DB dans users.onboarding_step. Le client reçoit l'état à chaque login.
"""
from enum import Enum


class OnboardingStep(str, Enum):
    # ── Phase 1 : Identité (toutes bloquantes) ──
    CITY_SELECTION = "city_selection"
    # Écran : carte ou liste des villes disponibles
    # Bloquant : OUI — on ne peut rien faire sans ville
    # Validation : city_id doit exister et être active
    # Résultat : users.city_id est set

    PHONE_VERIFIED = "phone_verified"
    # Auto-complété si l'utilisateur arrive ici (OTP déjà vérifié au login)
    # Skip automatique

    BASIC_INFO = "basic_info"
    # Écran : prénom, date de naissance, genre, genre recherché
    # Bloquant : OUI
    # Validation : 18+, prénom 2-50 chars, genre valide
    # Résultat : profile créé avec les champs de base

    SELFIE_VERIFICATION = "selfie_verification"
    # Écran : caméra frontale, pose aléatoire demandée
    # Bloquant : OUI — zéro tolérance, c'est la feature confiance #1
    # Validation : face détectée + liveness check + pose correcte
    # Retry : 3 tentatives max, puis cooldown 1h
    # Résultat : users.is_selfie_verified = true

    # ── Phase 2 : Profil (semi-bloquantes) ──
    PHOTOS = "photos"
    # Écran : upload 3-6 photos
    # Bloquant : OUI — minimum 3 photos obligatoire
    # Validation : modération auto, au moins 1 visage visible
    # Async : la modération tourne en background, mais l'utilisateur
    #         peut continuer. Les photos "pending" sont affichées avec
    #         un badge "en cours de vérification"

    QUARTIERS = "quartiers"
    # Écran : carte des quartiers de la ville, sélection multiple
    # Bloquant : OUI — minimum 1 quartier "lives"
    # UX : pré-sélection GPS si permission accordée
    # Résultat : user_quartiers créés

    INTENTION = "intention"
    # Écran : choix parmi 4 options (sérieux, apprendre, amitié, ouvert)
    # Bloquant : OUI
    # UX : gros boutons, pas de texte à écrire

    SECTOR = "sector"
    # Écran : choix du secteur d'activité
    # Bloquant : OUI
    # UX : grille d'icônes

    # ── Phase 3 : Enrichissement (skippables) ──
    PROMPTS = "prompts"
    # Écran : choisir 1-3 prompts et y répondre
    # Bloquant : NON — skippable mais le profil sera moins visible
    # UX : "Skip for now" en petit en bas

    TAGS = "tags"
    # Écran : grille de tags lifestyle, max 8
    # Bloquant : NON — skippable
    # UX : tags populaires mis en avant

    SPOTS = "spots"
    # Écran : recherche de spots, ajout à son profil
    # Bloquant : NON — skippable
    # UX : "Spots populaires près de toi" en suggestions

    NOTIFICATION_PERMISSION = "notification_permission"
    # Écran natif OS : autoriser les notifications
    # Bloquant : NON — skippable mais on insiste
    # UX : écran custom avant le prompt OS qui explique pourquoi

    # ── Terminé ──
    COMPLETED = "completed"
    # L'utilisateur accède au feed pour la première fois
    # Trigger : génération immédiate d'un feed (pas attendre le batch)


# Ordre des étapes
ONBOARDING_FLOW = [
    OnboardingStep.CITY_SELECTION,
    OnboardingStep.PHONE_VERIFIED,
    OnboardingStep.BASIC_INFO,
    OnboardingStep.SELFIE_VERIFICATION,
    OnboardingStep.PHOTOS,
    OnboardingStep.QUARTIERS,
    OnboardingStep.INTENTION,
    OnboardingStep.SECTOR,
    OnboardingStep.PROMPTS,
    OnboardingStep.TAGS,
    OnboardingStep.SPOTS,
    OnboardingStep.NOTIFICATION_PERMISSION,
    OnboardingStep.COMPLETED,
]

# Étapes qu'on peut skip
SKIPPABLE_STEPS = {
    OnboardingStep.PROMPTS,
    OnboardingStep.TAGS,
    OnboardingStep.SPOTS,
    OnboardingStep.NOTIFICATION_PERMISSION,
}

# Score de complétion par étape (pour inciter à revenir)
STEP_COMPLETENESS_WEIGHT = {
    "city_selection": 0.0,     # Prérequis, pas un "plus"
    "basic_info": 0.0,         # Prérequis
    "selfie_verification": 0.10,
    "photos": 0.30,            # 3 photos = 0.30, chaque photo en plus = +0.05
    "quartiers": 0.15,
    "intention": 0.0,          # Prérequis
    "sector": 0.0,             # Prérequis
    "prompts": 0.20,           # Gros bonus car très différenciant
    "tags": 0.15,
    "spots": 0.10,
}
```

```python
# ── GET /profiles/me/onboarding ──
# Response 200
{
    "current_step": "photos",
    "steps": [
        {"step": "city_selection", "status": "completed"},
        {"step": "phone_verified", "status": "completed"},
        {"step": "basic_info", "status": "completed"},
        {"step": "selfie_verification", "status": "completed"},
        {"step": "photos", "status": "in_progress", "detail": {"count": 1, "min": 3}},
        {"step": "quartiers", "status": "pending"},
        {"step": "intention", "status": "pending"},
        {"step": "sector", "status": "pending"},
        {"step": "prompts", "status": "pending", "skippable": true},
        {"step": "tags", "status": "pending", "skippable": true},
        {"step": "spots", "status": "pending", "skippable": true},
        {"step": "notification_permission", "status": "pending", "skippable": true}
    ],
    "progress_percent": 33,
    "profile_completeness": 0.40
}

# ── POST /profiles/me/onboarding/skip ──
# Skip l'étape courante (si skippable)
# Request
{
    "step": "prompts"
}
# Response 200
{
    "skipped": "prompts",
    "next_step": "tags",
    "warning": "Your profile will be less visible without prompts. You can add them later."
}
# Response 400
{
    "error": "step_not_skippable",
    "message": "photos is required and cannot be skipped"
}
```

---

## 14. Ice-breaker generation (CRITIQUE)

```python
# app/services/icebreaker_service.py
"""
Génère un ice-breaker contextuel quand deux personnes matchent.
L'ice-breaker est le premier message (type="system") dans la conversation.

Hiérarchie de priorité (on prend le premier qui matche) :
1. Prompt liké → question directe sur la réponse
2. Spot commun de haut niveau (régulier+) → question sur le lieu
3. Tag commun rare → question sur le centre d'intérêt
4. Quartier commun → question sur le quartier
5. Fallback générique (jamais vide)
"""
import random

# ── Templates par type (FR / EN) ──
TEMPLATES = {
    "fr": {
        "prompt_liked": [
            "💬 {liker} a aimé ta réponse à « {question} ». Raconte-lui en plus !",
            "💬 Ta réponse à « {question} » a attiré l'attention de {liker}. À vous de jouer !",
        ],
        "spot_common_high": [
            "📍 Vous êtes tous les deux des habitués de {spot}. C'est quoi votre commande préférée ?",
            "📍 {spot} c'est votre QG à tous les deux ! Vous vous êtes peut-être déjà croisés ?",
            "📍 Deux réguliers de {spot} qui se matchent — c'est quoi votre meilleur souvenir là-bas ?",
        ],
        "spot_common_low": [
            "📍 Vous fréquentez tous les deux {spot}. C'est quoi qui vous y attire ?",
            "📍 {spot} en commun ! Vous y allez plutôt le matin ou le soir ?",
        ],
        "tag_common_rare": [
            "🎯 Vous êtes tous les deux passionnés de {tag} — c'est pas si courant ! Racontez-vous ça.",
            "🎯 {tag} en commun ! C'est quoi qui vous a lancé là-dedans ?",
        ],
        "tag_common_normal": [
            "🎯 Vous partagez le goût pour {tag}. C'est quoi votre truc préféré dans ce domaine ?",
        ],
        "quartier_common": [
            "🏘️ Voisins de {quartier} ! C'est quoi votre endroit secret dans le coin ?",
            "🏘️ Vous êtes du côté de {quartier} tous les deux. Le meilleur spot du quartier ?",
        ],
        "fallback": [
            "👋 Vous avez matché ! Dites-vous bonjour et découvrez ce que vous avez en commun.",
            "👋 Premier pas fait ! Qui commence la conversation ?",
            "👋 Match ! Qu'est-ce qui vous a attiré dans le profil de l'autre ?",
        ],
    },
    "en": {
        "prompt_liked": [
            "💬 {liker} liked your answer to '{question}'. Tell them more!",
            "💬 Your answer to '{question}' caught {liker}'s eye. Your move!",
        ],
        "spot_common_high": [
            "📍 You're both regulars at {spot}. What's your go-to order?",
            "📍 {spot} is your shared spot! Have you maybe crossed paths before?",
        ],
        "spot_common_low": [
            "📍 You both go to {spot}. What draws you there?",
        ],
        "tag_common_rare": [
            "🎯 You're both into {tag} — that's rare! Tell each other about it.",
        ],
        "tag_common_normal": [
            "🎯 You share a love for {tag}. What's your favorite thing about it?",
        ],
        "quartier_common": [
            "🏘️ Neighbors in {quartier}! What's your secret spot in the area?",
        ],
        "fallback": [
            "👋 You matched! Say hi and find out what you have in common.",
            "👋 Match! What caught your eye about each other's profile?",
        ],
    },
}

# Tags considérés "rares" (< 5% d'utilisation dans la ville)
# Calculé dynamiquement par le batch analytics
# Fallback hardcoded pour le MVP :
RARE_TAGS_DEFAULT = {
    "bonsai", "writing", "diy", "volunteering", "cars",
    "photography", "art", "wellness",
}


async def generate_icebreaker(
    match, user_a, user_b, db_session
) -> str:
    """
    Génère l'ice-breaker pour un match.
    Retourne le texte du message.
    """
    lang = user_b.language  # Langue de celui qui reçoit le message

    # 1. Prompt liké ?
    if match.liked_prompt_id:
        prompt = _find_prompt(user_b.profile.prompts, match.liked_prompt_id)
        if prompt:
            template = random.choice(TEMPLATES[lang]["prompt_liked"])
            return template.format(
                liker=user_a.profile.display_name,
                question=prompt["question"],
            )

    # 2. Spot commun haut niveau ?
    common_spots = await _get_common_spots(user_a.id, user_b.id, db_session)
    high_spots = [s for s in common_spots if s["max_fidelity"] >= "regular"]
    if high_spots:
        spot = random.choice(high_spots)
        template = random.choice(TEMPLATES[lang]["spot_common_high"])
        return template.format(spot=spot["name"])

    # 3. Spot commun bas niveau ?
    if common_spots:
        spot = random.choice(common_spots)
        template = random.choice(TEMPLATES[lang]["spot_common_low"])
        return template.format(spot=spot["name"])

    # 4. Tag commun rare ?
    common_tags = set(user_a.profile.tags) & set(user_b.profile.tags)
    rare_common = common_tags & RARE_TAGS_DEFAULT
    if rare_common:
        tag = random.choice(list(rare_common))
        tag_label = TAG_LABELS[lang].get(tag, tag)
        template = random.choice(TEMPLATES[lang]["tag_common_rare"])
        return template.format(tag=tag_label)

    # 5. Tag commun normal ?
    if common_tags:
        tag = random.choice(list(common_tags))
        tag_label = TAG_LABELS[lang].get(tag, tag)
        template = random.choice(TEMPLATES[lang]["tag_common_normal"])
        return template.format(tag=tag_label)

    # 6. Quartier commun ?
    common_q = await _get_common_quartiers(user_a.id, user_b.id, db_session)
    if common_q:
        quartier = random.choice(common_q)
        template = random.choice(TEMPLATES[lang]["quartier_common"])
        return template.format(quartier=quartier["name"])

    # 7. Fallback
    return random.choice(TEMPLATES[lang]["fallback"])
```

---

## 15. Rate limiting (CRITIQUE)

```python
# app/core/rate_limiter.py
"""
Rate limiting basé sur Redis sliding window log.
Chaque requête ajoute un timestamp dans une sorted set Redis.
On compte les timestamps dans la fenêtre glissante.

Pourquoi sliding window et pas fixed window ?
- Fixed window : 60 req/min → tu peux faire 60 req à 0:59 + 60 à 1:00 = 120 en 2s
- Sliding window : on regarde les 60 dernières secondes à chaque requête

Headers retournés sur chaque réponse :
- X-RateLimit-Limit: 60
- X-RateLimit-Remaining: 42
- X-RateLimit-Reset: 1713100000 (timestamp Unix de la fin de la fenêtre)
- Retry-After: 18 (secondes, uniquement sur 429)
"""
import time
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware
from app.db.redis import get_redis


class RateLimitMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        redis = await get_redis()

        # Identifier l'appelant
        user = getattr(request.state, "user", None)
        if user:
            identifier = f"user:{user.id}"
            is_premium = user.is_premium
            is_admin = getattr(user, "is_admin", False)
        else:
            identifier = f"ip:{request.client.host}"
            is_premium = False
            is_admin = False

        # Admin bypass
        if is_admin:
            return await call_next(request)

        # Déterminer la limite selon le contexte
        limit_config = self._get_limit(request.url.path, is_premium)
        limit = limit_config["limit"]
        window = limit_config["window"]

        # Redis key
        key = f"rate:{identifier}:{request.url.path}"

        # Sliding window
        now = time.time()
        window_start = now - window

        pipe = redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)  # Purger les vieux
        pipe.zadd(key, {str(now): now})               # Ajouter le courant
        pipe.zcard(key)                                # Compter
        pipe.expire(key, window + 1)                   # TTL de sécurité
        _, _, count, _ = await pipe.execute()

        # Headers
        remaining = max(0, limit - count)
        reset_at = int(now + window)

        if count > limit:
            raise HTTPException(
                status_code=429,
                detail={
                    "error": "rate_limited",
                    "limit": limit,
                    "window_seconds": window,
                    "retry_after": int(window - (now - window_start)),
                },
                headers={
                    "X-RateLimit-Limit": str(limit),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(reset_at),
                    "Retry-After": str(int(window - (now - window_start))),
                },
            )

        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(limit)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"] = str(reset_at)
        return response

    def _get_limit(self, path: str, is_premium: bool) -> dict:
        """Retourne {limit, window} selon l'endpoint."""
        # Limites spécifiques par endpoint
        ENDPOINT_LIMITS = {
            "/auth/otp/request": {"limit": 3, "window": 600},     # 3 / 10min
            "/auth/otp/verify":  {"limit": 5, "window": 600},     # 5 / 10min
            "/photos":           {"limit": 10, "window": 60},     # 10 / min
            "/safety/emergency": {"limit": 5, "window": 3600},    # 5 / heure
        }

        for pattern, config in ENDPOINT_LIMITS.items():
            if path.endswith(pattern):
                return config

        # Limites globales
        if is_premium:
            return {"limit": 120, "window": 60}   # 120 req/min
        return {"limit": 60, "window": 60}         # 60 req/min


# ── Rate limiting spécifique aux likes ──
# Géré dans le service, pas dans le middleware, car c'est une limite
# business (X likes/jour) pas une limite technique (req/min)

async def check_daily_likes(user_id, is_premium, redis) -> dict:
    """
    Vérifie si l'utilisateur a encore des likes aujourd'hui.
    Retourne {allowed: bool, remaining: int, resets_at: str}
    """
    from datetime import datetime, timezone
    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    key = f"likes:{user_id}:{today}"

    count = await redis.get(key)
    count = int(count) if count else 0

    limit = 50 if is_premium else 5  # Depuis la config dynamique en prod

    if count >= limit:
        # Calculer le reset (prochain batch à 3h UTC)
        return {
            "allowed": False,
            "remaining": 0,
            "resets_at": f"{today}T03:00:00Z",
        }

    return {
        "allowed": True,
        "remaining": limit - count,
        "resets_at": f"{today}T03:00:00Z",
    }


async def increment_daily_likes(user_id, redis):
    """Incrémente le compteur de likes du jour."""
    from datetime import datetime, timezone
    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    key = f"likes:{user_id}:{today}"
    await redis.incr(key)
    await redis.expire(key, 86400 + 3600)  # 25h TTL (marge)
```

---

## 16. Security hardening (CRITIQUE)

```python
# app/core/security.py
"""
Sécurité complète du backend.
"""
import hashlib
import hmac
import re
from datetime import datetime, timedelta, timezone
from uuid import UUID

import jwt
from passlib.context import CryptContext

from app.config import get_settings

settings = get_settings()


# ── Phone hashing ──
# On ne stocke JAMAIS le numéro de téléphone en clair.
# SHA-256 avec un sel statique (pas besoin de bcrypt ici car on ne
# compare jamais en temps constant — on fait juste des lookups par hash)

def hash_phone(phone: str) -> str:
    """Normalise et hash un numéro de téléphone."""
    # Normaliser : retirer espaces, tirets, parenthèses
    normalized = re.sub(r"[\s\-\(\)]", "", phone)
    # Vérifier le format : doit commencer par +
    if not normalized.startswith("+"):
        raise ValueError("Phone must start with country code (+228...)")
    # Vérifier longueur (8-15 chiffres après le +)
    digits = normalized[1:]
    if not digits.isdigit() or not (8 <= len(digits) <= 15):
        raise ValueError("Invalid phone number format")
    # Hash avec sel
    salted = f"flaam:phone:{normalized}:{settings.secret_key[:16]}"
    return hashlib.sha256(salted.encode()).hexdigest()


# ── JWT ──

def create_access_token(user_id: UUID, is_admin: bool = False) -> str:
    expire = datetime.now(timezone.utc) + timedelta(
        minutes=settings.jwt_access_token_expire_minutes
    )
    payload = {
        "sub": str(user_id),
        "exp": expire,
        "type": "access",
        "admin": is_admin,
        "iat": datetime.now(timezone.utc),
    }
    return jwt.encode(payload, settings.secret_key, algorithm=settings.jwt_algorithm)


def create_refresh_token(user_id: UUID) -> str:
    expire = datetime.now(timezone.utc) + timedelta(
        days=settings.jwt_refresh_token_expire_days
    )
    payload = {
        "sub": str(user_id),
        "exp": expire,
        "type": "refresh",
        "iat": datetime.now(timezone.utc),
    }
    return jwt.encode(payload, settings.secret_key, algorithm=settings.jwt_algorithm)


def decode_token(token: str) -> dict:
    """Décode et valide un JWT. Raise jwt.InvalidTokenError si invalide."""
    return jwt.decode(
        token, settings.secret_key, algorithms=[settings.jwt_algorithm]
    )


# ── Webhook signature verification ──

def verify_paystack_signature(payload: bytes, signature: str) -> bool:
    """Vérifie la signature HMAC-SHA512 de Paystack."""
    expected = hmac.new(
        settings.paystack_webhook_secret.encode(),
        payload,
        hashlib.sha512,
    ).hexdigest()
    return hmac.compare_digest(expected, signature)


# ── Input validation ──

def sanitize_text(text: str, max_length: int = 500) -> str:
    """Nettoie un texte utilisateur."""
    # Strip whitespace
    text = text.strip()
    # Limiter la longueur
    text = text[:max_length]
    # Retirer les caractères de contrôle (sauf newlines)
    text = re.sub(r"[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]", "", text)
    # Pas de HTML
    text = re.sub(r"<[^>]+>", "", text)
    return text


def validate_display_name(name: str) -> str:
    """Valide un prénom."""
    name = sanitize_text(name, max_length=50)
    if len(name) < 2:
        raise ValueError("Name must be at least 2 characters")
    if re.search(r"[0-9@#$%^&*()_+=\[\]{};:\"\\|<>?/~`]", name):
        raise ValueError("Name contains invalid characters")
    return name
```

```python
# app/core/security_middleware.py
"""
Headers de sécurité ajoutés à chaque réponse.
"""
from starlette.middleware.base import BaseHTTPMiddleware


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)

        # HSTS : forcer HTTPS pendant 1 an
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains"
        )
        # Pas d'iframe
        response.headers["X-Frame-Options"] = "DENY"
        # Pas de sniffing MIME
        response.headers["X-Content-Type-Options"] = "nosniff"
        # XSS protection
        response.headers["X-XSS-Protection"] = "1; mode=block"
        # Referrer policy
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        # Permissions policy (pas de caméra/micro/géoloc côté API)
        response.headers["Permissions-Policy"] = (
            "camera=(), microphone=(), geolocation=()"
        )

        return response
```

---

## 17. RGPD — Pipeline de suppression complète (CRITIQUE)

```python
# app/services/gdpr_service.py
"""
Pipeline de suppression de compte conforme RGPD.
Appelé quand un utilisateur fait DELETE /auth/account.

Principe : suppression progressive en 3 phases.
Phase 1 (immédiate) : anonymisation visible
Phase 2 (J+7) : suppression des contenus
Phase 3 (J+30) : hard delete de toute trace
"""


async def initiate_account_deletion(user_id: UUID, db_session, redis):
    """Phase 1 — Anonymisation immédiate."""

    # 1. Profil invisible
    user.is_active = False
    user.is_visible = False

    # 2. Anonymiser le profil
    profile.display_name = "Utilisateur supprimé"
    profile.prompts = []
    profile.tags = []
    profile.languages = []
    profile.bio = None

    # 3. Retirer les photos du CDN (cache invalidation)
    for photo in user.photos:
        await schedule_photo_deletion(photo, delay_days=0)
        # Les URLs CDN retournent 404 immédiatement
        # (Cloudflare purge cache par URL)

    # 4. Fermer tous les matchs actifs
    # Les conversations restent visibles pour l'autre personne
    # avec "Utilisateur supprimé" comme nom
    await close_all_matches(user_id, db_session)

    # 5. Supprimer le feed Redis
    await redis.delete(f"feed:{user_id}")
    await redis.delete(f"behavior:{user_id}")

    # 6. Révoquer tous les tokens
    await redis.delete(f"session:{user_id}:*")

    # 7. Planifier Phase 2 et Phase 3 via Celery
    schedule_phase2_deletion.apply_async(
        args=[str(user_id)], countdown=7 * 86400  # J+7
    )
    schedule_phase3_deletion.apply_async(
        args=[str(user_id)], countdown=30 * 86400  # J+30
    )

    # 8. Logger l'événement (pour audit, sans données personnelles)
    await log_gdpr_event(user_id, "deletion_initiated")

    return {"status": "deletion_initiated", "hard_delete_at": "+30 days"}


# ── Phase 2 : J+7 — Suppression des contenus ──

@celery_app.task
def schedule_phase2_deletion(user_id: str):
    """
    Supprime les contenus générés par l'utilisateur.
    Les données structurelles restent pour l'intégrité référentielle.
    """
    # Messages envoyés → contenu remplacé par "[message supprimé]"
    # (on garde la structure pour que les conversations de l'autre
    # personne restent compréhensibles)

    # Vocaux R2 → supprimés
    # Photos chat → supprimées

    # Behavior logs → anonymisés (user_id remplacé par hash)
    # (gardés pour les stats agrégées de l'algo)

    # Reports où l'utilisateur est reporter → anonymisés
    # Reports où l'utilisateur est reported → gardés (sécurité)


# ── Phase 3 : J+30 — Hard delete ──

@celery_app.task
def schedule_phase3_deletion(user_id: str):
    """
    Suppression définitive de toute trace.
    """
    # DELETE FROM users WHERE id = user_id CASCADE
    # Cascade supprime : profile, photos, user_spots, user_quartiers,
    # devices, subscription, notification_preferences,
    # contact_blacklists, feed_caches

    # Matches : on garde la structure mais on anonymise
    # (l'autre personne voit toujours "ancien match")

    # Photos R2 : suppression des fichiers physiques
    # (normalement déjà fait en Phase 1, mais double vérification)

    # Redis : purge de toutes les clés contenant user_id

    # phone_hash : supprimé → le numéro peut se réinscrire
    # MAIS on garde un flag dans une table séparée :
    # deleted_accounts (phone_hash, deleted_at, reason)
    # Pour détecter les suppressions/recréations fréquentes


# ── Export des données (RGPD Article 20) ──

async def export_user_data(user_id: UUID, db_session) -> dict:
    """
    GET /profiles/me/export
    Retourne toutes les données personnelles au format JSON.
    """
    return {
        "account": {
            "created_at": "...",
            "city": "Lomé",
            "language": "fr",
            "is_premium": False,
        },
        "profile": {
            "display_name": "Ama",
            "birth_date": "2000-03-15",
            "gender": "woman",
            "intention": "serious",
            "sector": "finance",
            "prompts": [...],
            "tags": [...],
            "languages": [...],
        },
        "photos": [
            {"url": "...", "uploaded_at": "...", "order": 0},
        ],
        "quartiers": [
            {"name": "Tokoin", "type": "lives"},
        ],
        "spots": [
            {"name": "Café 21", "checkins": 4, "fidelity": "regular"},
        ],
        "matches": [
            {"matched_with": "anonymized_id", "matched_at": "...", "status": "expired"},
        ],
        "messages_sent": 47,  # Compteur, pas le contenu (vie privée de l'autre)
        "events_attended": 2,
        "reports_filed": 0,
        "exported_at": "2026-04-14T12:00:00Z",
    }
```

---

## 18. Modération des messages — Pipeline complet (CRITIQUE)

```python
# app/services/moderation_service.py
"""
Pipeline de modération des messages en 3 niveaux :
1. Filtre temps réel (avant envoi)
2. Analyse async (après envoi)
3. Review humaine (escalation)
"""

# ── Niveau 1 : Filtre temps réel (synchrone, < 5ms) ──

def filter_message_realtime(content: str, sender_id: UUID) -> dict:
    """
    Appelé AVANT d'envoyer le message.
    Retourne {allowed: bool, reason: str, action: str}
    """
    content_lower = content.lower().strip()

    # 1a. Liens suspects
    url_pattern = re.compile(r"https?://\S+|www\.\S+|bit\.ly/\S+|tinyurl\.\S+")
    urls = url_pattern.findall(content_lower)
    if urls:
        # Whitelist : flaam.app, google maps, instagram (partage de profil)
        ALLOWED_DOMAINS = {"flaam.app", "maps.google.com", "goo.gl/maps", "instagram.com"}
        for url in urls:
            domain = extract_domain(url)
            if domain not in ALLOWED_DOMAINS:
                return {
                    "allowed": False,
                    "reason": "suspicious_link",
                    "action": "block",
                    "user_message": "Links are not allowed in messages for your safety.",
                }

    # 1b. Demandes d'argent (keywords)
    MONEY_KEYWORDS = [
        "envoie moi", "send me", "western union", "moneygram",
        "mon compte", "my account", "transfert", "transfer",
        "momo", "tmoney", "wave", "orange money",
        "j'ai besoin d'argent", "i need money", "aide moi financ",
        "numéro de compte", "account number", "iban",
    ]
    for keyword in MONEY_KEYWORDS:
        if keyword in content_lower:
            return {
                "allowed": True,  # On laisse passer mais on flag
                "reason": "potential_scam",
                "action": "flag_for_review",
                "user_message": None,
            }

    # 1c. Numéros de téléphone (encourager à rester in-app)
    phone_pattern = re.compile(r"\+?\d[\d\s\-]{7,14}\d")
    if phone_pattern.search(content):
        return {
            "allowed": True,  # On laisse passer (choix personnel)
            "reason": "phone_shared",
            "action": "log",  # Juste logger, pas bloquer
            "user_message": None,
        }

    # 1d. Tout OK
    return {"allowed": True, "reason": None, "action": "none"}


# ── Niveau 2 : Analyse async (Celery task, après envoi) ──

@celery_app.task
def analyze_message_async(message_id: str, sender_id: str, content: str):
    """
    Analyse approfondie après envoi. Ne bloque pas l'utilisateur.
    """
    # 2a. Détection de messages copié-collés
    # Hash du contenu (premiers 100 chars, lowercase, stripped)
    content_hash = hashlib.md5(content[:100].lower().strip().encode()).hexdigest()
    # Compter combien de matchs différents ont reçu ce même hash
    similar_count = count_similar_messages(sender_id, content_hash, days=7)
    if similar_count >= 3:
        # Même message envoyé à 3+ matchs = spam
        flag_message(message_id, "copypaste_spam")
        # Réduire le behavior_multiplier
        reduce_behavior_score(sender_id, "spam_detected", penalty=0.1)

    # 2b. Détection de patterns de scam
    # Progression rapide vers échange d'argent
    conversation_age = get_conversation_age(message_id)
    if conversation_age < timedelta(hours=2):
        if any(kw in content.lower() for kw in MONEY_KEYWORDS):
            flag_message(message_id, "early_money_request")
            # Alerte modération prioritaire

    # 2c. Contenu sexuel non-sollicité
    # Heuristique simple (pas de ML, trop lourd pour le MVP)
    SEXUAL_KEYWORDS = [...]  # Liste configurable
    if any(kw in content.lower() for kw in SEXUAL_KEYWORDS):
        # Vérifier si c'est le premier message de la conversation
        if is_first_or_second_message(message_id):
            flag_message(message_id, "unsolicited_sexual")


# ── Niveau 3 : Review humaine (queue admin) ──

# Les messages flaggés apparaissent dans GET /admin/moderation/messages
# L'admin peut :
# - Dismiss (faux positif)
# - Warn user (envoyer un avertissement)
# - Mute user (interdire d'envoyer des messages pendant X heures)
# - Ban user (si récidive)
#
# Escalation automatique :
# - 3 flags en 7 jours → avertissement auto
# - 5 flags en 30 jours → mute 24h auto
# - 2 mutes → ban temporaire 7 jours
# - 1 ban temporaire + nouveau flag → ban permanent

ESCALATION_RULES = {
    "flags_to_warning": 3,           # Nombre de flags avant warning auto
    "flags_to_mute": 5,              # Nombre de flags avant mute auto
    "mute_duration_hours": 24,
    "mutes_to_temp_ban": 2,
    "temp_ban_duration_days": 7,
    "temp_bans_to_permanent": 1,
}
```

---

## 19. Monitoring & alerting (PRODUCTION)

```yaml
# monitoring/prometheus.yml
# Métriques exposées par FastAPI via prometheus-fastapi-instrumentator

scrape_configs:
  - job_name: 'flaam-api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: /metrics
    scrape_interval: 15s

  - job_name: 'celery'
    static_configs:
      - targets: ['celery-exporter:9808']
    scrape_interval: 30s

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    scrape_interval: 30s

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
    scrape_interval: 15s
```

```python
# app/core/metrics.py
"""
Métriques custom Prometheus pour le business Flaam.
En plus des métriques HTTP standard (latence, status codes, etc.)
fournies par prometheus-fastapi-instrumentator.
"""
from prometheus_client import Counter, Histogram, Gauge

# ── Matching ──
feed_generation_duration = Histogram(
    "flaam_feed_generation_seconds",
    "Time to generate a single user's daily feed",
    buckets=[0.1, 0.5, 1, 2, 5, 10, 30],
)
feed_generation_total = Counter(
    "flaam_feeds_generated_total",
    "Total feeds generated",
    ["city"],
)
feed_avg_score = Gauge(
    "flaam_feed_avg_geo_score",
    "Average geo score in generated feeds",
    ["city"],
)

# ── User actions ──
likes_total = Counter("flaam_likes_total", "Total likes", ["premium"])
matches_total = Counter("flaam_matches_total", "Total matches")
messages_total = Counter("flaam_messages_total", "Total messages", ["type"])
checkins_total = Counter("flaam_checkins_total", "Total check-ins")
signups_total = Counter("flaam_signups_total", "Total signups", ["city"])
deletions_total = Counter("flaam_deletions_total", "Total account deletions")

# ── Moderation ──
flagged_messages = Counter("flaam_flagged_messages_total", "Flagged messages", ["reason"])
reports_total = Counter("flaam_reports_total", "Reports filed", ["reason"])
bans_total = Counter("flaam_bans_total", "Users banned")

# ── Infrastructure ──
websocket_connections = Gauge("flaam_ws_connections", "Active WebSocket connections")
celery_queue_size = Gauge("flaam_celery_queue_size", "Celery queue size", ["queue"])
```

```python
# Alertes Telegram (ou Slack) — envoyées par Alertmanager

ALERT_RULES = {
    # ── Critique (réveille-toi) ──
    "api_down": {
        "condition": "up{job='flaam-api'} == 0 for 2m",
        "severity": "critical",
        "message": "🔴 API Flaam is DOWN",
    },
    "high_error_rate": {
        "condition": "rate(http_requests_total{status=~'5..'}[5m]) > 0.1",
        "severity": "critical",
        "message": "🔴 Error rate > 10% on API",
    },
    "db_connections_exhausted": {
        "condition": "pg_stat_activity_count > 18",  # Pool size = 20
        "severity": "critical",
        "message": "🔴 DB connections near limit (18/20)",
    },

    # ── Warning (regarde quand tu peux) ──
    "high_latency": {
        "condition": "histogram_quantile(0.95, http_request_duration_seconds) > 2",
        "severity": "warning",
        "message": "🟡 P95 latency > 2s",
    },
    "celery_queue_backlog": {
        "condition": "flaam_celery_queue_size{queue='matching'} > 1000",
        "severity": "warning",
        "message": "🟡 Matching queue backlog > 1000 tasks",
    },
    "redis_memory_high": {
        "condition": "redis_memory_used_bytes / redis_memory_max_bytes > 0.85",
        "severity": "warning",
        "message": "🟡 Redis memory > 85%",
    },
    "disk_space_low": {
        "condition": "node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.15",
        "severity": "warning",
        "message": "🟡 Disk space < 15% remaining",
    },
    "feed_generation_slow": {
        "condition": "flaam_feed_generation_seconds{quantile='0.95'} > 10",
        "severity": "warning",
        "message": "🟡 Feed generation P95 > 10s",
    },
}
```

---

## 20. Logging structuré (PRODUCTION)

```python
# app/core/logging.py
"""
Logging JSON structuré via structlog.
Chaque ligne de log est un objet JSON parsable.
Centralisé via Loki + Grafana.
"""
import structlog

def setup_logging():
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    )

# ── Quoi logger ──

# TOUJOURS logger (INFO) :
# - Chaque requête API : method, path, status, duration_ms, user_id
# - Chaque match créé : user_a_id, user_b_id, geo_score, was_wildcard
# - Chaque inscription : user_id, city, platform
# - Chaque paiement : user_id, amount, provider, status
# - Chaque action de modération : admin_id, action, target_user_id
# - Chaque batch matching : city, users_processed, duration_s, errors

# LOGGER EN WARNING :
# - Rate limit hit : ip/user_id, endpoint, count
# - Message flaggé : message_id, reason, sender_id
# - OTP échoué : phone_hash (jamais le numéro), attempts
# - Photo rejetée : photo_id, reason, score

# LOGGER EN ERROR :
# - Exceptions non gérées (auto via Sentry aussi)
# - Échec paiement webhook : provider, reference, error
# - Échec envoi SMS : provider, error
# - Échec push notification : fcm_error, user_id

# JAMAIS LOGGER :
# - Contenu des messages (vie privée)
# - Numéros de téléphone en clair
# - Tokens JWT
# - Mots de passe / codes OTP
# - Photos (URLs ok, contenu jamais)

# Exemple de ligne de log :
# {
#   "timestamp": "2026-04-14T15:30:00Z",
#   "level": "info",
#   "event": "api_request",
#   "method": "POST",
#   "path": "/api/v1/feed/uuid/like",
#   "status": 200,
#   "duration_ms": 45,
#   "user_id": "uuid",
#   "ip": "102.xxx.xxx.xxx"
# }

# ── Rétention ──
# - Loki : 30 jours de logs chauds
# - Après 30 jours : archivé compressé sur R2 (90 jours)
# - Après 90 jours : supprimé
```

---

## 21. Error handling — Catalogue complet (PRODUCTION)

```python
# app/core/exceptions.py

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse


class FlaamException(Exception):
    """Base exception pour toutes les erreurs business Flaam."""
    def __init__(self, code: str, message: str, status_code: int = 400, detail: dict = None):
        self.code = code
        self.message = message
        self.status_code = status_code
        self.detail = detail or {}


def register_exception_handlers(app: FastAPI):
    @app.exception_handler(FlaamException)
    async def flaam_handler(request: Request, exc: FlaamException):
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": exc.code,
                "message": exc.message,
                **exc.detail,
            },
        )

    @app.exception_handler(HTTPException)
    async def http_handler(request: Request, exc: HTTPException):
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": "http_error", "message": str(exc.detail)},
        )

    @app.exception_handler(Exception)
    async def generic_handler(request: Request, exc: Exception):
        # Logger + Sentry
        import sentry_sdk
        sentry_sdk.capture_exception(exc)
        return JSONResponse(
            status_code=500,
            content={"error": "internal_error", "message": "Something went wrong"},
        )


# ── Catalogue d'erreurs ──
# Chaque erreur a un code unique, un message, et un status HTTP.
# Le client peut mapper les codes pour afficher des messages localisés.

ERROR_CATALOGUE = {
    # Auth
    "invalid_otp":              (401, "Invalid or expired OTP code"),
    "otp_max_attempts":         (429, "Too many OTP attempts. Try again in {retry_after}s"),
    "otp_cooldown":             (429, "Please wait {retry_after}s before requesting a new OTP"),
    "token_expired":            (401, "Access token has expired"),
    "token_invalid":            (401, "Invalid token"),
    "refresh_token_invalid":    (401, "Invalid refresh token"),

    # Profile
    "profile_not_found":        (404, "Profile not found"),
    "profile_incomplete":       (400, "Profile is incomplete. Complete onboarding first"),
    "name_invalid":             (400, "Display name contains invalid characters"),
    "age_under_18":             (400, "You must be 18 or older"),
    "max_photos_reached":       (400, "Maximum 6 photos allowed"),
    "min_photos_required":      (400, "At least 3 photos are required"),
    "photo_moderation_failed":  (400, "Photo was rejected: {reason}"),
    "selfie_failed":            (400, "Selfie verification failed: {reason}"),

    # Quartiers
    "max_quartiers_reached":    (400, "Maximum {max} {type} quartiers reached"),
    "quartier_not_in_city":     (400, "This quartier is not in your city"),
    "min_quartier_required":    (400, "At least 1 'lives' quartier is required"),
    "step_not_skippable":       (400, "{step} is required and cannot be skipped"),

    # Spots
    "spot_not_found":           (404, "Spot not found"),
    "checkin_too_far":          (400, "You must be within 100m of the spot"),
    "spot_already_added":       (409, "This spot is already in your profile"),

    # Feed / Matching
    "daily_likes_exhausted":    (429, "No likes remaining today. Resets at {resets_at}"),
    "already_liked":            (409, "You already liked this profile"),
    "profile_not_in_feed":      (400, "This profile is not in your current feed"),
    "self_like":                (400, "You cannot like yourself"),

    # Matches
    "match_not_found":          (404, "Match not found"),
    "match_expired":            (410, "This match has expired"),
    "not_matched":              (403, "You are not matched with this user"),

    # Messages
    "message_too_long":         (400, "Message exceeds 2000 characters"),
    "voice_too_long":           (400, "Voice message exceeds 60 seconds"),
    "message_blocked":          (400, "Message blocked: {reason}"),
    "user_muted":               (403, "You are temporarily muted until {until}"),

    # Events
    "event_full":               (400, "This event is full"),
    "event_past":               (400, "This event has already passed"),
    "already_registered":       (409, "You are already registered for this event"),

    # Payment
    "payment_failed":           (400, "Payment failed: {reason}"),
    "already_premium":          (409, "You already have an active premium subscription"),
    "invalid_webhook_signature":(400, "Invalid webhook signature"),

    # Safety
    "user_blocked":             (400, "You have blocked this user"),
    "user_banned":              (403, "Your account has been suspended: {reason}"),
    "rate_limited":             (429, "Too many requests. Retry in {retry_after}s"),

    # Admin
    "admin_required":           (403, "Admin access required"),
    "config_out_of_bounds":     (400, "{key} must be between {min} and {max}"),
}
```

---

## 22. Backup & disaster recovery (PRODUCTION)

```yaml
# scripts/backup/backup.sh
# Lancé par cron du host (pas Docker) toutes les 6 heures

#!/bin/bash
set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
R2_BUCKET="flaam-backups"

# ── PostgreSQL ──
echo "[backup] Starting PostgreSQL dump..."
docker exec flaam-db pg_dump -U flaam -Fc flaam > "${BACKUP_DIR}/db_${TIMESTAMP}.dump"
# Format custom (-Fc) : compressé, supporte pg_restore sélectif
# Taille estimée : ~50MB pour 100K users

# Chiffrer avant upload
gpg --symmetric --cipher-algo AES256 \
    --passphrase-file /secrets/backup-passphrase \
    "${BACKUP_DIR}/db_${TIMESTAMP}.dump"

# Upload vers R2
aws s3 cp "${BACKUP_DIR}/db_${TIMESTAMP}.dump.gpg" \
    "s3://${R2_BUCKET}/postgres/db_${TIMESTAMP}.dump.gpg" \
    --endpoint-url "${R2_ENDPOINT}"

# Nettoyer le local (garder 24h)
find "${BACKUP_DIR}" -name "db_*.dump*" -mtime +1 -delete

# ── Redis ──
echo "[backup] Starting Redis backup..."
docker exec flaam-redis redis-cli BGSAVE
sleep 5
docker cp flaam-redis:/data/dump.rdb "${BACKUP_DIR}/redis_${TIMESTAMP}.rdb"
aws s3 cp "${BACKUP_DIR}/redis_${TIMESTAMP}.rdb" \
    "s3://${R2_BUCKET}/redis/redis_${TIMESTAMP}.rdb" \
    --endpoint-url "${R2_ENDPOINT}"

echo "[backup] Done."
```

```python
# Politique de rétention des backups :
BACKUP_RETENTION = {
    "postgresql": {
        "frequency": "every 6 hours",
        "retention_hot": "7 days",     # Sur R2, accès immédiat
        "retention_cold": "90 days",   # Sur R2 Infrequent Access
        "encryption": "AES-256 GPG",
    },
    "redis": {
        "frequency": "every 6 hours",
        "retention": "3 days",         # Redis est reconstituable depuis PostgreSQL
        "note": "Le cache se reconstruit automatiquement au prochain batch",
    },
    "r2_photos": {
        "backup": "R2 built-in redundancy (3 copies)",
        "note": "Pas de backup supplémentaire, R2 est déjà répliqué",
    },
}

# RTO (Recovery Time Objective) : 1 heure
# RPO (Recovery Point Objective) : 6 heures (fréquence des backups)
#
# Procédure de restore :
# 1. Provisionner un nouveau VPS (si le serveur est mort)
# 2. Installer Docker Compose
# 3. Télécharger le dernier backup PostgreSQL depuis R2
# 4. gpg --decrypt db_latest.dump.gpg > db_latest.dump
# 5. docker exec -i flaam-db pg_restore -U flaam -d flaam < db_latest.dump
# 6. docker-compose up -d
# 7. Le batch matching regénère les feeds automatiquement
# 8. Redis se remplit naturellement au fil des requêtes
# Temps total estimé : 30-45 min
```

---

## 23. Alembic migrations (PRODUCTION)

```python
# Conventions de migration Alembic

# ── Naming convention ──
# Fichier : {revision_id}_{descriptive_name}.py
# Ex : a1b2c3d4_add_quartier_proximity_table.py
# Ex : e5f6g7h8_add_interested_relation_type.py

# ── Règles ──
# 1. JAMAIS de migration destructive sans vérification
#    - DROP TABLE → toujours vérifier que la table est vide
#    - DROP COLUMN → toujours faire en 2 étapes :
#      a. Deploy code qui n'utilise plus la colonne
#      b. Migration qui drop la colonne
#
# 2. Zero-downtime migrations :
#    - ALTER TABLE ADD COLUMN → OK (pas de lock)
#    - ALTER TABLE ADD COLUMN NOT NULL DEFAULT → OK en PG 11+
#    - CREATE INDEX → TOUJOURS "CONCURRENTLY" (pas de lock)
#    - ALTER TABLE ALTER COLUMN TYPE → DANGEREUX (full table rewrite)
#      → Préférer : add new column, backfill, switch, drop old
#
# 3. Chaque migration a un upgrade() ET un downgrade()
#    - Le downgrade doit être testé
#    - Si le downgrade est impossible (données perdues), le documenter
#
# 4. Backfills :
#    - Jamais dans la migration elle-même (trop lent, bloque le deploy)
#    - Faire un script séparé dans scripts/backfill_xxx.py
#    - Exécuter après le deploy, en background

# ── Exemple de migration type ──
"""add quartier proximity table

Revision ID: a1b2c3d4
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import UUID


def upgrade():
    op.create_table(
        "quartier_proximities",
        sa.Column("id", UUID(as_uuid=True), primary_key=True),
        sa.Column("quartier_a_id", UUID(as_uuid=True),
                  sa.ForeignKey("quartiers.id"), nullable=False),
        sa.Column("quartier_b_id", UUID(as_uuid=True),
                  sa.ForeignKey("quartiers.id"), nullable=False),
        sa.Column("proximity_score", sa.Float, nullable=False),
        sa.Column("distance_km", sa.Float, nullable=False),
        sa.CheckConstraint("proximity_score >= 0 AND proximity_score <= 1"),
        sa.CheckConstraint("quartier_a_id < quartier_b_id"),
    )
    op.create_index(
        "uq_quartier_proximity", "quartier_proximities",
        ["quartier_a_id", "quartier_b_id"], unique=True,
    )
    # Index créé CONCURRENTLY en production :
    # op.execute("CREATE INDEX CONCURRENTLY ix_qp_a ON quartier_proximities (quartier_a_id)")


def downgrade():
    op.drop_table("quartier_proximities")
```

---

## 24. CI/CD pipeline (PRODUCTION)

```yaml
# .github/workflows/deploy.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: flaam_test
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-test.txt
      - name: Run linters
        run: |
          ruff check .
          mypy app/ --ignore-missing-imports
      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/flaam_test
          REDIS_URL: redis://localhost:6379/0
          SECRET_KEY: test-secret-key-not-for-production
        run: pytest tests/ -v --cov=app --cov-report=term --cov-fail-under=80
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t flaam-api:${{ github.sha }} .
      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u flaam --password-stdin ghcr.io
          docker tag flaam-api:${{ github.sha }} ghcr.io/flaam/api:${{ github.sha }}
          docker tag flaam-api:${{ github.sha }} ghcr.io/flaam/api:latest
          docker push ghcr.io/flaam/api:${{ github.sha }}
          docker push ghcr.io/flaam/api:latest
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/flaam
            docker pull ghcr.io/flaam/api:latest
            # Rolling restart (zero-downtime)
            docker-compose up -d --no-deps --build api
            docker-compose up -d --no-deps --build celery-worker-images
            docker-compose up -d --no-deps --build celery-worker-matching
            docker-compose up -d --no-deps --build celery-worker-notifications
            docker-compose up -d --no-deps --build celery-worker-general
            docker-compose up -d --no-deps --build celery-beat
            # Health check
            sleep 10
            curl -f http://localhost:8000/health || exit 1
            echo "Deploy successful"
      - name: Notify Telegram
        if: always()
        run: |
          STATUS="${{ job.status }}"
          curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
            -d "chat_id=${{ secrets.TELEGRAM_CHAT_ID }}" \
            -d "text=🚀 Deploy ${STATUS}: flaam-api@${{ github.sha }}"
```

---

## 25. Caching strategy (PRODUCTION)

```python
# app/core/caching.py
"""
Stratégie de cache Redis complète.
Chaque type de donnée a un TTL, une stratégie d'invalidation,
et une clé de fallback.
"""

CACHE_STRATEGY = {
    # ── Feeds (cœur de l'app) ──
    "feed:{user_id}": {
        "type": "LIST of profile_ids",
        "ttl": 86400,  # 24h
        "invalidation": "Batch nocturne remplace. Recalcul incrémental si action forte.",
        "fallback": "Générer à la volée (lent, ~2s)",
        "warmup": "Batch nocturne pré-remplit pour tous les users actifs",
    },

    # ── Behavior scores ──
    "behavior:{user_id}": {
        "type": "HASH {response_quality, selectivity, richness, depth, multiplier}",
        "ttl": None,  # Pas de TTL, persisté en DB toutes les heures
        "invalidation": "Mis à jour en temps réel à chaque action",
        "fallback": "Lire depuis profiles.behavior_multiplier (DB)",
    },

    # ── Sessions ──
    "session:{token_jti}": {
        "type": "STRING user_id",
        "ttl": 900,   # 15min (access token)
        "invalidation": "Supprimé au logout",
        "note": "On stocke le JTI, pas le token entier",
    },
    "refresh:{token_jti}": {
        "type": "STRING user_id",
        "ttl": 2592000,  # 30 jours
        "invalidation": "Supprimé au logout ou rotation",
    },

    # ── Rate limiting ──
    "rate:{identifier}:{path}": {
        "type": "SORTED SET de timestamps",
        "ttl": 120,  # 2min (marge sur la fenêtre de 60s)
        "invalidation": "Auto-expire",
    },
    "likes:{user_id}:{date}": {
        "type": "STRING counter",
        "ttl": 90000,  # 25h
        "invalidation": "Auto-expire",
    },

    # ── OTP ──
    "otp:{phone_hash}": {
        "type": "HASH {code, attempts, created_at}",
        "ttl": 600,  # 10min
        "invalidation": "Supprimé après vérification réussie",
    },

    # ── Chat (messages récents) ──
    "recent_msgs:{match_id}": {
        "type": "LIST de messages sérialisés JSON",
        "ttl": 604800,  # 7 jours
        "invalidation": "LPUSH à chaque nouveau message, LTRIM à 50",
        "fallback": "SELECT FROM messages ORDER BY created_at DESC LIMIT 50",
    },

    # ── Matching config ──
    "matching_config:{key}": {
        "type": "STRING float value",
        "ttl": 300,  # 5min
        "invalidation": "DEL sur modification admin",
        "fallback": "DB → constants.py defaults",
    },

    # ── Blacklist ──
    "blacklist:{phone_hash}": {
        "type": "STRING 1",
        "ttl": None,
        "invalidation": "Manuel (admin unban)",
    },

    # ── Profil résumé (pour le feed, évite un JOIN lourd) ──
    "profile_summary:{user_id}": {
        "type": "HASH {name, age, intention, sector, photo_url, ...}",
        "ttl": 3600,  # 1h
        "invalidation": "DEL quand le profil est modifié",
        "fallback": "JOIN users + profiles + photos LIMIT 1",
    },

    # ── Event counters ──
    "event_attendees:{event_id}": {
        "type": "STRING counter",
        "ttl": None,
        "invalidation": "INCR/DECR sur inscription/désinscription",
        "fallback": "COUNT(*) FROM event_registrations",
    },
}

# ── Cache stampede prevention ──
# Quand le cache expire et 100 requêtes arrivent en même temps,
# toutes vont frapper la DB. Solution : lock distribué.
#
# Algorithme :
# 1. Cache miss → tenter SET lock:{key} NX EX 5 (lock 5s)
# 2. Si lock acquis → requête DB → SET cache → DEL lock
# 3. Si lock pas acquis → attendre 100ms → retry depuis le cache
# 4. Après 3 retries → requête DB directement (safety net)
```

---

## 26. Notification templates (PRODUCTION)

```python
# app/services/notification_templates.py

PUSH_TEMPLATES = {
    "new_match": {
        "fr": {
            "title": "Nouveau match ! 💫",
            "body": "Quelqu'un de {quartier} s'intéresse à toi",
            # On ne révèle pas le nom → incite à ouvrir l'app
        },
        "en": {
            "title": "New match! 💫",
            "body": "Someone from {quartier} is interested in you",
        },
        "data": {
            "type": "new_match",
            "match_id": "{match_id}",
            "deep_link": "flaam://matches/{match_id}",
        },
    },
    "new_message": {
        "fr": {
            "title": "{sender_name}",
            "body": "{preview}",  # Premiers 80 chars du message
        },
        "en": {
            "title": "{sender_name}",
            "body": "{preview}",
        },
        "data": {
            "type": "new_message",
            "match_id": "{match_id}",
            "deep_link": "flaam://chat/{match_id}",
        },
    },
    "daily_feed": {
        "fr": {
            "title": "Tes profils du jour sont prêts",
            "body": "{count} personnes à découvrir dans ton coin",
        },
        "en": {
            "title": "Your daily profiles are ready",
            "body": "{count} people to discover near you",
        },
        "data": {
            "type": "daily_feed",
            "deep_link": "flaam://feed",
        },
    },
    "match_expiring": {
        "fr": {
            "title": "Match qui expire bientôt ⏳",
            "body": "Ta conversation avec {name} expire dans 24h. Dis quelque chose !",
        },
        "en": {
            "title": "Match expiring soon ⏳",
            "body": "Your conversation with {name} expires in 24h. Say something!",
        },
        "data": {
            "type": "match_expiring",
            "match_id": "{match_id}",
            "deep_link": "flaam://chat/{match_id}",
        },
    },
    "event_reminder": {
        "fr": {
            "title": "Event dans 2h",
            "body": "{event_title} à {spot_name}. {attendees} personnes inscrites",
        },
        "en": {
            "title": "Event in 2h",
            "body": "{event_title} at {spot_name}. {attendees} people going",
        },
        "data": {
            "type": "event_reminder",
            "event_id": "{event_id}",
            "deep_link": "flaam://events/{event_id}",
        },
    },
    "date_reminder": {
        "fr": {
            "title": "Date dans 2h 📍",
            "body": "Rendez-vous avec {name} à {spot_name}",
        },
        "en": {
            "title": "Date in 2h 📍",
            "body": "Meeting {name} at {spot_name}",
        },
        "data": {
            "type": "date_reminder",
            "match_id": "{match_id}",
            "deep_link": "flaam://chat/{match_id}",
        },
    },
    "safety_timer_warning": {
        "fr": {
            "title": "Timer de sécurité ⚠️",
            "body": "N'oublie pas de désactiver ton timer. Il expire dans 30 min.",
        },
        "en": {
            "title": "Safety timer ⚠️",
            "body": "Don't forget to deactivate your timer. It expires in 30 min.",
        },
        "data": {
            "type": "safety_timer",
            "deep_link": "flaam://safety",
        },
    },
    "premium_expired": {
        "fr": {
            "title": "Ton abonnement a expiré",
            "body": "Renouvelle pour garder tes avantages premium",
        },
        "en": {
            "title": "Your subscription has expired",
            "body": "Renew to keep your premium benefits",
        },
        "data": {
            "type": "premium_expired",
            "deep_link": "flaam://settings/subscription",
        },
    },
}

# ── Deep linking scheme ──
# flaam://feed                     → Onglet feed
# flaam://matches/{match_id}       → Détail du match
# flaam://chat/{match_id}          → Conversation
# flaam://events/{event_id}        → Détail event
# flaam://spots                    → Onglet spots
# flaam://safety                   → Écran sécurité
# flaam://settings/subscription    → Paramètres abo
# flaam://profile/edit             → Édition profil
```

---

## 27. App versioning & force update (PRODUCTION)

```python
# app/api/v1/app_version.py

# ── GET /app/version-check ──
# Appelé par le client à chaque démarrage de l'app
# Request headers : X-App-Version: 1.2.3, X-Platform: android
# Response 200
{
    "minimum_version": "1.0.0",
    "recommended_version": "1.3.0",
    "current_latest": "1.3.2",
    "force_update": false,
    "update_message": null,
    "store_url": "https://play.google.com/store/apps/details?id=app.flaam"
}

# Response 200 (force update)
{
    "minimum_version": "1.2.0",
    "recommended_version": "1.3.0",
    "current_latest": "1.3.2",
    "force_update": true,
    "update_message": {
        "fr": "Une mise à jour importante est disponible. Mets à jour pour continuer.",
        "en": "An important update is available. Update to continue."
    },
    "store_url": "https://play.google.com/store/apps/details?id=app.flaam"
}

# Stocké en DB (table simple) :
# app_versions :
#   platform    | min_version | recommended_version | latest_version | store_url
#   "android"   | "1.0.0"     | "1.3.0"            | "1.3.2"        | "https://..."
#   "ios"       | "1.0.0"     | "1.2.0"            | "1.2.0"        | "https://..."

# Politique de force update :
# - Changement de schéma API breaking → force update
# - Faille de sécurité corrigée → force update
# - Nouvelles features → soft update (recommended, pas forcé)
```

---

## 28. Feature flags (NICE-TO-HAVE)

```python
# app/models/feature_flag.py

class FeatureFlag(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "feature_flags"

    key: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    # Ex: "events_user_creation", "voice_messages", "mode_voyageur",
    #     "premium_boost", "interested_quartiers"

    is_enabled: Mapped[bool] = mapped_column(default=False)
    # Global on/off

    rollout_percentage: Mapped[int] = mapped_column(default=0)
    # 0-100. Si enabled=true et rollout=50, seuls 50% des users voient la feature
    # Déterministe : hash(user_id + flag_key) % 100 < rollout_percentage

    city_ids: Mapped[list | None] = mapped_column(JSONB, nullable=True)
    # Si non-null, la feature n'est active que dans ces villes
    # Ex: ["uuid-lome", "uuid-abidjan"] pour un rollout progressif par ville

    premium_only: Mapped[bool] = mapped_column(default=False)
    # Si true, seuls les premium y ont accès

    description: Mapped[str | None] = mapped_column(Text, nullable=True)


# ── Usage dans le code ──
async def is_feature_enabled(flag_key: str, user) -> bool:
    flag = await get_flag(flag_key)  # Cache Redis, TTL 1min
    if not flag or not flag.is_enabled:
        return False
    if flag.premium_only and not user.is_premium:
        return False
    if flag.city_ids and str(user.city_id) not in flag.city_ids:
        return False
    if flag.rollout_percentage < 100:
        hash_val = int(hashlib.md5(f"{user.id}{flag_key}".encode()).hexdigest(), 16)
        if hash_val % 100 >= flag.rollout_percentage:
            return False
    return True

# ── API admin ──
# GET /admin/feature-flags → liste toutes les flags
# PATCH /admin/feature-flags/{key} → modifier (enable, rollout %, villes)
# POST /admin/feature-flags → créer une nouvelle flag
```

---

## 29. Analytics & KPIs (NICE-TO-HAVE)

```python
# app/services/analytics_service.py
"""
KPIs business calculés quotidiennement par un Celery task.
Stockés dans une table analytics_daily pour le dashboard admin.
"""

DAILY_KPIS = {
    # ── Acquisition ──
    "signups": "COUNT new users today",
    "signups_completed_onboarding": "COUNT users who reached COMPLETED today",
    "onboarding_drop_off_step": "Most common step where users abandon",

    # ── Engagement ──
    "dau": "COUNT users who opened the app today",
    "wau": "COUNT users active in the last 7 days",
    "mau": "COUNT users active in the last 30 days",
    "avg_session_duration_seconds": "Average time in app per session",
    "avg_profiles_viewed_per_user": "Profiles viewed / active users",
    "avg_likes_per_user": "Likes given / active users",

    # ── Matching ──
    "likes_given": "Total likes today",
    "matches_created": "Total new matches today",
    "match_rate": "matches / likes (devrait être 15-30%)",
    "wildcard_like_rate": "likes on wildcards / total wildcard views",

    # ── Conversation ──
    "conversations_started": "Matches where first message was sent today",
    "conversation_rate": "conversations_started / matches (cible: > 60%)",
    "avg_messages_per_conversation": "Messages échangés avant expiration ou unmatch",
    "meetup_proposals": "Nombre de 'On se retrouve ?' envoyés",
    "meetup_acceptance_rate": "Acceptés / proposés",

    # ── Rétention ──
    "d1_retention": "% users who return day after signup",
    "d7_retention": "% users who return 7 days after signup",
    "d30_retention": "% users who return 30 days after signup",
    "churn_rate": "% users inactive > 14 days / total users",

    # ── Monétisation ──
    "new_subscribers": "New premium subscriptions today",
    "mrr": "Monthly Recurring Revenue (somme des abonnements actifs)",
    "conversion_rate": "premium users / total active users",
    "arpu": "Revenue / active users",

    # ── Sécurité ──
    "reports_filed": "Total reports today",
    "users_banned": "Users banned today",
    "messages_flagged": "Messages auto-flagged today",
    "false_positive_rate": "Flagged messages dismissed by admin / total flagged",

    # ── Par ville ──
    # Tous les KPIs ci-dessus sont aussi calculés par ville
    # pour identifier les marchés qui performent et ceux qui stagnent
}
```

---

## 30. Anti-abus — Contrôle du cycle création/suppression/recréation (CRITIQUE)

### 30.1 Le problème

Quelqu'un peut abuser du cycle de vie du compte :
- **Boost farming** : créer → profiter du boost nouveau profil 10 jours → supprimer → recréer → nouveau boost
- **Ban evasion** : se faire bannir → supprimer → recréer avec le même numéro
- **Like reset** : épuiser ses 5 likes gratuits → supprimer → recréer → 5 nouveaux likes
- **Block evasion** : harcelé quelqu'un → bloqué → supprimer → recréer → réapparaître dans le feed de la victime
- **Report washing** : accumuler des signalements → supprimer avant ban → recréer propre

La difficulté c'est que le RGPD impose la suppression des données personnelles, mais pour la sécurité des autres utilisateurs on DOIT se souvenir que ce numéro/device a existé. La solution légale : on ne garde que des **identifiants hachés** et des **métadonnées de sécurité**, pas de données personnelles.

### 30.2 Modèle AccountHistory

```python
# app/models/account_history.py
"""
Table de mémoire anti-abus.
Survit à la suppression du compte (RGPD compatible).

Base légale RGPD : intérêt légitime (Article 6.1.f) — la prévention
de la fraude et la protection des autres utilisateurs justifie la
rétention de données minimales de sécurité après suppression.

On ne garde AUCUNE donnée personnelle identifiable ici :
- Pas de nom, pas de photo, pas de profil
- Pas de numéro en clair
- Juste des hashes et des compteurs
"""
import uuid
from datetime import datetime
from sqlalchemy import String, Integer, Float, Boolean, DateTime, Text, Index, func
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base, UUIDMixin


class AccountHistory(Base, UUIDMixin):
    __tablename__ = "account_histories"

    # ── Identifiants hachés (jamais le clair) ──
    phone_hash: Mapped[str] = mapped_column(String(128), nullable=False, index=True)
    # SHA-256 du numéro normalisé. Identique au hash dans users.phone_hash
    # Permet de retrouver l'historique quand le même numéro se réinscrit

    device_fingerprints: Mapped[list] = mapped_column(JSONB, default=list)
    # Liste des hashes de devices associés à ce numéro au fil du temps
    # Ex: ["sha256:abc123", "sha256:def456"]
    # Un device qui apparaît sur plusieurs phone_hash = red flag

    # ── Compteurs de vie ──
    total_accounts_created: Mapped[int] = mapped_column(Integer, default=1)
    # Nombre total de comptes créés avec ce numéro (inclut le courant)

    total_accounts_deleted: Mapped[int] = mapped_column(Integer, default=0)
    # Nombre de suppressions volontaires

    total_bans: Mapped[int] = mapped_column(Integer, default=0)
    # Nombre de fois que ce numéro a été banni

    # ── Timestamps ──
    first_account_created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    last_account_created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    last_account_deleted_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    last_ban_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    # ── Raison du dernier départ ──
    last_departure_reason: Mapped[str | None] = mapped_column(String(30), nullable=True)
    # "user_deleted"     → suppression volontaire
    # "banned_spam"      → banni pour spam
    # "banned_harassment" → banni pour harcèlement
    # "banned_scam"      → banni pour arnaque
    # "banned_fake"      → banni pour faux profil
    # "banned_underage"  → banni car mineur
    # "banned_other"     → banni autre raison

    # ── Scores de risque (calculés) ──
    risk_score: Mapped[float] = mapped_column(Float, default=0.0)
    # 0.0 = clean, 1.0 = très suspect
    # Calculé à partir de l'historique (voir formule ci-dessous)

    # ── Restrictions appliquées ──
    # (calculées dynamiquement, mais persistées pour audit)
    current_restriction: Mapped[str] = mapped_column(String(30), default="none")
    # "none"              → aucune restriction
    # "no_boost"          → pas de boost nouveau profil
    # "reduced_likes"     → likes quotidiens réduits (3 au lieu de 5)
    # "probation"         → boost interdit + visibilité réduite + modération renforcée
    # "cooldown_active"   → impossible de créer un compte (trop tôt)
    # "permanent_ban"     → impossible de créer un compte (jamais)

    restriction_expires_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    # Null si permanent ou pas de restriction

    # ── Blocks survivants ──
    # Quand quelqu'un bloque un utilisateur et que celui-ci supprime son compte,
    # le block doit survivre. On stocke les phone_hash des personnes qui
    # ont bloqué ce numéro.
    blocked_by_hashes: Mapped[list] = mapped_column(JSONB, default=list)
    # Ex: ["sha256:aaa", "sha256:bbb"]
    # À la recréation du compte, ces hashes sont comparés avec les users actifs
    # et les blocks sont automatiquement restaurés

    # ── Notes admin ──
    admin_notes: Mapped[str | None] = mapped_column(Text, nullable=True)

    __table_args__ = (
        Index("ix_ah_phone", "phone_hash"),
        Index("ix_ah_risk", "risk_score"),
        # Index GIN sur device_fingerprints pour chercher un device
        Index("ix_ah_devices", "device_fingerprints", postgresql_using="gin"),
    )
```

### 30.3 Matrice de restrictions par fenêtre temporelle

```python
# app/services/abuse_prevention_service.py
"""
Quand un utilisateur essaie de recréer un compte, on calcule
les restrictions à appliquer basées sur son historique.
"""
from datetime import datetime, timedelta, timezone


def calculate_restrictions(history: AccountHistory) -> dict:
    """
    Détermine les restrictions pour un compte qui revient.
    Appelé lors de POST /auth/otp/verify quand is_new_user=true
    mais qu'un AccountHistory existe pour ce phone_hash.

    Retourne : {
        "allowed": bool,           # Peut créer un compte ?
        "restriction": str,        # Niveau de restriction
        "reason": str,             # Explication
        "new_user_boost": bool,    # A droit au boost ?
        "daily_likes_override": int | None,  # Override du nombre de likes
        "restriction_expires_at": datetime | None,
        "risk_score": float,
    }
    """
    now = datetime.now(timezone.utc)

    # ── CAS 1 : Ban permanent ──
    # Si l'utilisateur a été banni pour des raisons graves
    # (harcèlement, scam, mineur), c'est permanent. Point.
    PERMANENT_BAN_REASONS = {"banned_harassment", "banned_scam", "banned_underage"}
    if history.last_departure_reason in PERMANENT_BAN_REASONS:
        return {
            "allowed": False,
            "restriction": "permanent_ban",
            "reason": f"Account permanently banned: {history.last_departure_reason}",
            "new_user_boost": False,
            "daily_likes_override": None,
            "restriction_expires_at": None,
            "risk_score": 1.0,
        }

    # ── CAS 2 : Banni pour spam/fake (ban temporaire, mais récidive = permanent) ──
    if history.total_bans >= 2:
        return {
            "allowed": False,
            "restriction": "permanent_ban",
            "reason": f"Multiple bans ({history.total_bans}). Permanent.",
            "new_user_boost": False,
            "daily_likes_override": None,
            "restriction_expires_at": None,
            "risk_score": 1.0,
        }

    if history.total_bans == 1 and history.last_departure_reason in {"banned_spam", "banned_fake"}:
        # 1 ban pour spam/fake → peut revenir mais en probation
        days_since_ban = (now - history.last_ban_at).days if history.last_ban_at else 0

        if days_since_ban < 30:
            return {
                "allowed": False,
                "restriction": "cooldown_active",
                "reason": f"Banned {days_since_ban} days ago. Must wait 30 days.",
                "new_user_boost": False,
                "daily_likes_override": None,
                "restriction_expires_at": history.last_ban_at + timedelta(days=30),
                "risk_score": 0.8,
            }
        else:
            # 30+ jours après le ban → probation
            return {
                "allowed": True,
                "restriction": "probation",
                "reason": "Returning after ban. Probation period.",
                "new_user_boost": False,
                "daily_likes_override": 3,  # 3 likes/jour au lieu de 5
                "restriction_expires_at": now + timedelta(days=60),
                "risk_score": 0.6,
            }

    # ── CAS 3 : Suppression volontaire (pas de ban) ──
    # C'est ici que la fenêtre temporelle entre en jeu
    if history.last_account_deleted_at is None:
        # Jamais supprimé → premier compte, pas de restriction
        return _no_restriction()

    days_since_deletion = (now - history.last_account_deleted_at).days
    total_cycles = history.total_accounts_deleted  # Nb de fois supprimé

    # ── Moins de 1 heure → clairement du gaming ──
    hours_since = (now - history.last_account_deleted_at).total_seconds() / 3600
    if hours_since < 1:
        return {
            "allowed": False,
            "restriction": "cooldown_active",
            "reason": "Account deleted less than 1 hour ago.",
            "new_user_boost": False,
            "daily_likes_override": None,
            "restriction_expires_at": history.last_account_deleted_at + timedelta(hours=24),
            "risk_score": 0.7,
        }

    # ── Moins de 24 heures → probablement du gaming ──
    if days_since_deletion < 1:
        return {
            "allowed": True,  # On laisse revenir mais sans avantages
            "restriction": "no_boost",
            "reason": "Account deleted less than 24h ago. No new user boost.",
            "new_user_boost": False,
            "daily_likes_override": 3,
            "restriction_expires_at": None,
            "risk_score": 0.5,
        }

    # ── 1 à 7 jours → suspect ──
    if days_since_deletion < 7:
        return {
            "allowed": True,
            "restriction": "no_boost",
            "reason": "Account deleted less than 7 days ago.",
            "new_user_boost": False,
            "daily_likes_override": None,  # Likes normaux
            "restriction_expires_at": None,
            "risk_score": 0.3,
        }

    # ── 7 à 30 jours → léger doute ──
    if days_since_deletion < 30:
        if total_cycles >= 3:
            # 3+ suppressions en tout → pattern suspect même si 7+ jours
            return {
                "allowed": True,
                "restriction": "no_boost",
                "reason": f"{total_cycles} account cycles detected.",
                "new_user_boost": False,
                "daily_likes_override": None,
                "restriction_expires_at": None,
                "risk_score": 0.4,
            }
        return {
            "allowed": True,
            "restriction": "no_boost",
            "reason": "Account deleted less than 30 days ago.",
            "new_user_boost": False,
            "daily_likes_override": None,
            "restriction_expires_at": None,
            "risk_score": 0.2,
        }

    # ── 30 jours à 6 mois → OK mais pas de boost si beaucoup de cycles ──
    if days_since_deletion < 180:
        if total_cycles >= 3:
            return {
                "allowed": True,
                "restriction": "no_boost",
                "reason": f"{total_cycles} account cycles. No boost despite time gap.",
                "new_user_boost": False,
                "daily_likes_override": None,
                "restriction_expires_at": None,
                "risk_score": 0.2,
            }
        # 1-2 cycles, 30+ jours → retour légitime probable
        return {
            "allowed": True,
            "restriction": "reduced_boost",
            "reason": "Returning user. Reduced new user boost.",
            "new_user_boost": True,  # Boost accordé mais réduit
            "boost_multiplier": 0.5, # 50% du boost normal
            "daily_likes_override": None,
            "restriction_expires_at": None,
            "risk_score": 0.1,
        }

    # ── 6 mois à 1 an → retour légitime ──
    if days_since_deletion < 365:
        if total_cycles >= 5:
            # Même après 6 mois, 5+ cycles c'est un pattern
            return {
                "allowed": True,
                "restriction": "no_boost",
                "reason": f"{total_cycles} total cycles. Excessive.",
                "new_user_boost": False,
                "daily_likes_override": None,
                "restriction_expires_at": None,
                "risk_score": 0.15,
            }
        return _clean_return(risk=0.05)

    # ── Plus d'1 an → ardoise quasi propre ──
    if total_cycles >= 5:
        # Même après 1 an, 5+ cycles → pas de boost par précaution
        return {
            "allowed": True,
            "restriction": "no_boost",
            "reason": f"{total_cycles} total cycles over lifetime.",
            "new_user_boost": False,
            "daily_likes_override": None,
            "restriction_expires_at": None,
            "risk_score": 0.1,
        }
    # Vraiment propre
    return _clean_return(risk=0.0)


def _no_restriction():
    return {
        "allowed": True,
        "restriction": "none",
        "reason": None,
        "new_user_boost": True,
        "daily_likes_override": None,
        "restriction_expires_at": None,
        "risk_score": 0.0,
    }


def _clean_return(risk: float):
    return {
        "allowed": True,
        "restriction": "none",
        "reason": "Returning user. Clean history.",
        "new_user_boost": True,
        "daily_likes_override": None,
        "restriction_expires_at": None,
        "risk_score": risk,
    }
```

### 30.4 Tableau récapitulatif

```
┌─────────────────────┬─────────────────┬───────────┬──────────┬────────────────┐
│ Fenêtre temporelle  │ Peut recréer ?  │ Boost ?   │ Likes    │ Risk score     │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ < 1 heure           │ ❌ NON          │ —         │ —        │ 0.70           │
│                     │ Cooldown 24h    │           │          │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ 1h → 24h            │ ✅ OUI          │ ❌ Non    │ 3/jour   │ 0.50           │
│                     │                 │           │ (réduit) │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ 1 → 7 jours         │ ✅ OUI          │ ❌ Non    │ Normal   │ 0.30           │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ 7 → 30 jours        │ ✅ OUI          │ ❌ Non    │ Normal   │ 0.20           │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ 30 jours → 6 mois   │ ✅ OUI          │ ⚡ 50%   │ Normal   │ 0.10           │
│ (1-2 cycles)        │                 │ du boost  │          │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ 6 mois → 1 an       │ ✅ OUI          │ ✅ Full   │ Normal   │ 0.05           │
│ (1-4 cycles)        │                 │           │          │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ > 1 an              │ ✅ OUI          │ ✅ Full   │ Normal   │ 0.00           │
│ (< 5 cycles)        │                 │           │          │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ Banni 1x (spam)     │ ❌ 30j cooldown │ ❌ Non    │ 3/jour   │ 0.80 → 0.60   │
│ puis après 30j      │ puis probation  │           │ 60 jours │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ Banni 2x+           │ ❌ PERMANENT    │ —         │ —        │ 1.00           │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ Banni harcèlement / │ ❌ PERMANENT    │ —         │ —        │ 1.00           │
│ scam / mineur       │                 │           │          │                │
├─────────────────────┼─────────────────┼───────────┼──────────┼────────────────┤
│ 5+ cycles (tout     │ ✅ OUI          │ ❌ Non    │ Normal   │ 0.10-0.40      │
│ temps confondu)     │ mais jamais     │ (jamais)  │          │ (selon timing) │
│                     │ de boost        │           │          │                │
└─────────────────────┴─────────────────┴───────────┴──────────┴────────────────┘
```

### 30.5 Intégration dans le flow d'inscription

```python
# app/services/auth_service.py (extrait)

async def handle_new_signup(phone_hash: str, device_fp: str, db_session, redis):
    """
    Appelé quand POST /auth/otp/verify réussit et que le phone_hash
    n'a pas de compte actif associé.
    """

    # 1. Chercher dans l'historique
    history = await db_session.execute(
        select(AccountHistory).where(AccountHistory.phone_hash == phone_hash)
    )
    history = history.scalar_one_or_none()

    # 2. Chercher aussi par device fingerprint (SIM changée mais même device)
    if history is None:
        history = await find_history_by_device(device_fp, db_session)
        # Si on trouve un historique via device mais pas via phone,
        # c'est potentiellement quelqu'un qui a changé de SIM pour contourner.
        # On lie les deux : ajouter le nouveau phone_hash à cet historique.

    # 3. Calculer les restrictions
    if history:
        restrictions = calculate_restrictions(history)

        if not restrictions["allowed"]:
            # Compte interdit de recréation
            return {
                "error": "account_creation_blocked",
                "reason": restrictions["reason"],
                "restriction": restrictions["restriction"],
                "expires_at": restrictions.get("restriction_expires_at"),
            }

        # Mettre à jour l'historique
        history.total_accounts_created += 1
        history.last_account_created_at = datetime.now(timezone.utc)
        if device_fp not in history.device_fingerprints:
            history.device_fingerprints.append(device_fp)
        history.risk_score = restrictions["risk_score"]
        history.current_restriction = restrictions["restriction"]
        history.restriction_expires_at = restrictions.get("restriction_expires_at")

    else:
        # Premier compte, créer l'historique
        history = AccountHistory(
            phone_hash=phone_hash,
            device_fingerprints=[device_fp],
            total_accounts_created=1,
        )
        db_session.add(history)
        restrictions = _no_restriction()

    # 4. Créer le compte utilisateur
    user = User(
        phone_hash=phone_hash,
        is_phone_verified=True,
        account_created_count=history.total_accounts_created,
        # ... autres champs
    )
    db_session.add(user)

    # 5. Restaurer les blocks survivants
    if history and history.blocked_by_hashes:
        await restore_blocks(user.id, history.blocked_by_hashes, db_session)
        # Pour chaque hash dans blocked_by_hashes :
        # - Trouver le user actif avec ce phone_hash
        # - Créer un Block(blocker=found_user, blocked=new_user)
        # L'utilisateur revenu ne peut pas voir les gens qui l'avaient bloqué

    await db_session.commit()

    return {
        "user": user,
        "restrictions": restrictions,
        "is_returning_user": history.total_accounts_created > 1,
    }
```

### 30.6 Protection par device fingerprint (changement de SIM)

```python
# app/services/abuse_prevention_service.py (suite)

async def find_history_by_device(device_fp: str, db_session) -> AccountHistory | None:
    """
    Cherche un historique par device fingerprint.
    Cas d'usage : l'utilisateur change de SIM pour contourner le phone_hash.

    Le device fingerprint est un hash de :
    - Android ID (persistant par device, reset si factory reset)
    - Modèle du device (ex: "Tecno Spark 10")
    - Résolution écran
    NOT included : IMEI (nécessite permission spéciale sur Android 10+)
    """
    result = await db_session.execute(
        select(AccountHistory).where(
            AccountHistory.device_fingerprints.contains([device_fp])
        )
    )
    return result.scalar_one_or_none()


async def detect_multi_account(phone_hash: str, device_fp: str, db_session) -> dict:
    """
    Détecte si un device est associé à plusieurs comptes.
    Appelé à chaque login.

    Scénarios :
    - 1 device, 1 phone → normal
    - 1 device, 2+ phones → suspect (multi-compte)
    - 2+ devices, 1 phone → normal (changement de téléphone)
    """
    # Compter les phone_hash distincts liés à ce device
    result = await db_session.execute(
        select(func.count(AccountHistory.id)).where(
            AccountHistory.device_fingerprints.contains([device_fp]),
            AccountHistory.phone_hash != phone_hash,
        )
    )
    other_accounts = result.scalar()

    if other_accounts == 0:
        return {"multi_account": False}

    if other_accounts == 1:
        # 2 comptes sur le même device : peut-être légitime (conjoint)
        # mais on flag
        return {
            "multi_account": True,
            "severity": "low",
            "other_accounts_count": 1,
            "action": "log",  # Juste logger
        }

    # 3+ comptes sur le même device → clairement suspect
    return {
        "multi_account": True,
        "severity": "high",
        "other_accounts_count": other_accounts,
        "action": "flag_for_review",
        # Alerter la modération
    }
```

### 30.7 Mise à jour de la suppression RGPD

```python
# Quand un compte est supprimé (Phase 1 du RGPD flow), on met à jour l'historique :

async def update_history_on_deletion(user, reason: str, db_session):
    """
    Appelé dans initiate_account_deletion() AVANT de supprimer les données.
    """
    history = await get_or_create_history(user.phone_hash, db_session)

    history.total_accounts_deleted += 1
    history.last_account_deleted_at = datetime.now(timezone.utc)
    history.last_departure_reason = reason  # "user_deleted" ou "banned_*"

    if reason.startswith("banned_"):
        history.total_bans += 1
        history.last_ban_at = datetime.now(timezone.utc)

    # Sauvegarder les blocks que d'autres personnes ont sur cet utilisateur
    # Pour les restaurer à la recréation
    blocks = await db_session.execute(
        select(Block.blocker_id).where(Block.blocked_id == user.id)
    )
    blocker_ids = [row[0] for row in blocks.fetchall()]

    # Convertir en phone_hash (car les IDs seront supprimés en Phase 3)
    for blocker_id in blocker_ids:
        blocker = await db_session.get(User, blocker_id)
        if blocker and blocker.phone_hash not in history.blocked_by_hashes:
            history.blocked_by_hashes.append(blocker.phone_hash)

    # Recalculer le risk score
    history.risk_score = _compute_risk_score(history)

    await db_session.commit()


def _compute_risk_score(history: AccountHistory) -> float:
    """
    Score de risque 0.0 → 1.0 basé sur l'historique complet.
    """
    score = 0.0

    # Bans
    score += min(0.5, history.total_bans * 0.25)

    # Cycles fréquents
    if history.total_accounts_created >= 5:
        score += 0.2
    elif history.total_accounts_created >= 3:
        score += 0.1

    # Vitesse de recréation (si on a les données)
    if history.last_account_deleted_at and history.last_account_created_at:
        gap = (history.last_account_created_at - history.last_account_deleted_at)
        if gap.total_seconds() < 3600:  # < 1h entre suppression et recréation
            score += 0.3
        elif gap.days < 7:
            score += 0.1

    # Multi-device
    if len(history.device_fingerprints) >= 3:
        score += 0.1

    return min(1.0, score)
```

### 30.8 Rétention des AccountHistory

```python
# L'AccountHistory est la SEULE table qui survit indéfiniment
# après suppression d'un compte. Justification RGPD :
#
# - Ne contient AUCUNE donnée personnelle identifiable
#   (pas de nom, photo, profil, numéro en clair)
# - Contient uniquement des hashes et des compteurs de sécurité
# - Base légale : intérêt légitime (Article 6.1.f)
#   "La prévention de la fraude et la protection de la sécurité des
#    autres utilisateurs constituent un intérêt légitime justifiant
#    le traitement minimal de données post-suppression."
#
# Politique de rétention de l'AccountHistory :
# - Comptes clean (risk_score=0, 0 bans, ≤2 cycles) : supprimé après 2 ans
# - Comptes à risque (risk_score > 0.3 ou bans > 0) : conservé 5 ans
# - Bans permanents (harcèlement, scam, mineur) : conservé indéfiniment
#
# Cron mensuel : purge_old_account_histories

HISTORY_RETENTION = {
    "clean": {"max_age_years": 2},
    "risky": {"max_age_years": 5},
    "permanent_ban": {"max_age_years": None},  # Jamais supprimé
}
```


---

## 31. Cold start & stratégie de lancement par ville (CRITIQUE)

```python
# app/services/city_launch_service.py
"""
Le cold start est le problème #1 des apps de dating.
87% des apps échouent principalement à cause de ça.

Principe : ne JAMAIS lancer une ville avec 0 utilisateurs.
Lancer = avoir atteint un seuil minimum AVANT d'activer le matching.
"""

# ── Seuils de viabilité par taille de ville ──
CITY_LAUNCH_THRESHOLDS = {
    # Pour qu'un utilisateur ouvre l'app et trouve au moins 8-12 profils
    # dans son feed, il faut un pool minimum après filtres L1 (~20% passent).
    # Donc pour 12 profils dans le feed → ~60 candidats post-filtre → ~300 inscrits.
    # Avec du buffer pour les inactifs : 500 minimum.

    "minimum_users_to_activate": 500,
    # En dessous : l'app est en mode "waitlist" dans cette ville

    "healthy_pool": 2000,
    # Au-dessus : l'algo fonctionne bien, on peut réduire le marketing

    "critical_gender_ratio": 0.30,
    # Minimum 30% de chaque genre. En dessous = expérience dégradée

    "minimum_female_users": 150,
    # Seuil absolu de femmes inscrites avant activation
    # (les femmes sont le côté rare du marché, elles dictent la viabilité)
}


# ── 4 phases de lancement d'une ville ──

class CityLaunchPhase:
    """
    Phase 1 : TEASER (pré-lancement)
    ─────────────────────────────────
    - La ville apparaît dans l'app mais est "coming soon"
    - Les utilisateurs peuvent s'inscrire sur une waitlist
    - Ils remplissent leur profil complet (onboarding normal)
    - Ils voient un compteur : "247/500 inscrits. Plus que 253 !"
    - Gamification : "Invite 3 amis et passe en priorité à l'ouverture"
    - PAS de feed, PAS de matching, PAS de like
    - Objectif : atteindre le seuil de 500 avant d'activer

    Pourquoi ça marche :
    - Les gens investissent (profil rempli) → coût de sortie
    - L'effet FOMO ("je veux être dans les premiers")
    - Le compteur crée une dynamique collective
    - À l'ouverture, 500 personnes ont un profil complet = feed riche jour 1

    Phase 2 : LAUNCH (activation)
    ──────────────────────────────
    - Seuil atteint → batch nocturne génère les premiers feeds
    - Notification push à tous les inscrits : "Flaam est LIVE à Lomé !"
    - Les "early adopters" qui ont invité des amis reçoivent
      1 semaine de premium gratuit
    - Boost agressif des nouveaux profils (5x au lieu de 3x la première semaine)
    - Feed étendu à 15-20 profils/jour (au lieu de 12) pendant les 2 premières semaines
      pour compenser le pool encore petit

    Phase 3 : GROWTH (croissance)
    ─────────────────────────────
    - Pool > 500, < 2000
    - Marketing organique : events locaux dans l'app pour créer du buzz
    - Partenariats cafés/restos locaux : "Inscris-toi sur Flaam et
      obtiens 10% au Café 21" → ramène les femmes (qui sont le côté rare)
    - Monitoring quotidien du gender ratio
    - Si ratio hommes > 75% : réduire la visibilité de l'app dans les
      canaux masculins, augmenter le marketing ciblé femmes

    Phase 4 : STABLE (maturité)
    ───────────────────────────
    - Pool > 2000, gender ratio > 35/65
    - Algo fonctionne normalement
    - Feed revient à 12 profils/jour
    - Boost nouveaux profils revient à la normale (3x)
    - Prêt à devenir "hub" pour aider le lancement de villes voisines
    """
    TEASER = "teaser"
    LAUNCH = "launch"
    GROWTH = "growth"
    STABLE = "stable"


# ── API : ce que voit l'utilisateur d'une ville en teaser ──

# GET /feed dans une ville en phase TEASER :
TEASER_RESPONSE = {
    "city_status": "teaser",
    "total_registered": 247,
    "threshold": 500,
    "remaining": 253,
    "your_position": 89,  # Ta position dans la waitlist
    "estimated_launch": "2-3 semaines",
    "profiles": [],  # Pas de feed
    "invite_bonus": {
        "invited_count": 2,
        "needed_for_premium": 3,
        "reward": "1 week free premium at launch",
    },
    "message": "Lomé n'est pas encore ouvert. Invite tes amis pour accélérer !",
}
```

```python
# ── Modèle DB pour le suivi du lancement ──

class CityLaunchStatus(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "city_launch_statuses"

    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), unique=True, nullable=False
    )
    phase: Mapped[str] = mapped_column(String(20), default="teaser")
    # "teaser" | "launch" | "growth" | "stable"

    total_registered: Mapped[int] = mapped_column(Integer, default=0)
    male_registered: Mapped[int] = mapped_column(Integer, default=0)
    female_registered: Mapped[int] = mapped_column(Integer, default=0)
    # Mis à jour par trigger ou cron

    launched_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    # Quand la ville est passée de teaser à launch

    # Waitlist
    waitlist_invites_total: Mapped[int] = mapped_column(Integer, default=0)
    # Nombre total d'invitations envoyées (viralité)
```

---

## 32. Gestion du déséquilibre hommes/femmes (CRITIQUE)

```python
# app/services/matching_engine/gender_balance.py
"""
Le déséquilibre H/F est le problème structurel #1 de toutes les apps de dating.
Typiquement 70% hommes / 30% femmes. Ça crée :
- Hommes : frustration, 0 matchs, like tout le monde en désespoir, churn
- Femmes : submergées, fatigue, 200 likes non lus, churn aussi

Flaam a un avantage structurel : le feed limité à 12 profils/jour.
Ça protège déjà les femmes de la surcharge. Mais il faut aller plus loin.
"""

# ── Principe : ne pas trahir, équilibrer silencieusement ──
# On ne pénalise AUCUN genre. On ne cache pas de profils.
# On ajuste la DISTRIBUTION pour que l'expérience soit viable des deux côtés.

GENDER_BALANCE_STRATEGIES = {

    "asymmetric_feed_size": {
        # Quand le ratio est déséquilibré, le feed du côté rare est plus petit
        # et celui du côté abondant aussi. Mais le côté rare voit des profils
        # plus qualifiés (meilleurs scores).
        #
        # Exemple avec 70% hommes / 30% femmes :
        # - Feed d'une femme : 12 profils (normal), mais tous triés par
        #   behavior_multiplier (les hommes qui répondent, qui sont sélectifs,
        #   qui ont des bonnes conversations sont priorisés)
        # - Feed d'un homme : 10 profils (légèrement réduit), car le pool
        #   de femmes est plus petit, mais chaque profil montré est une femme
        #   active récemment
        #
        # Résultat : les hommes voient moins de profils mais des profils
        # qui ont plus de chances de répondre. Les femmes voient les
        # meilleurs profils masculins. Win-win.
        "enabled": True,
    },

    "quality_gate_on_majority": {
        # Le côté abondant (hommes) doit avoir un profil plus complet
        # pour apparaître dans les feeds du côté rare (femmes).
        #
        # Concrètement : pour apparaître dans le feed d'une femme,
        # un homme doit avoir :
        # - completeness >= 0.70 (3+ photos, au moins 1 prompt, spots tagués)
        # - behavior_multiplier >= 0.9 (pas un spammeur de likes)
        # - is_selfie_verified = true (déjà obligatoire mais renforcé ici)
        #
        # Ce gate N'EXISTE PAS dans l'autre sens (pas de gate pour les femmes
        # apparaissant dans les feeds des hommes). C'est asymétrique par design.
        #
        # L'homme ne sait pas que ce gate existe. Il voit juste que
        # compléter son profil lui donne plus de matchs.
        "enabled": True,
        "min_completeness_for_majority": 0.70,
        "min_behavior_for_majority": 0.9,
    },

    "smart_like_weighting": {
        # Un like d'un homme sélectif (selectivity_index élevé) pèse plus
        # dans les notifications de la femme qu'un like d'un homme qui
        # like tout le monde.
        #
        # La femme voit "3 personnes s'intéressent à toi" mais ces 3
        # sont triées par la qualité du like, pas par ordre chronologique.
        # Le gars qui a scrollé tout son profil et liké un prompt spécifique
        # apparaît en premier. Le gars qui a liké en 0.5 seconde apparaît dernier.
        "enabled": True,
    },

    "conversation_starter_incentive": {
        # Le côté rare (femmes) est encouragé à initier la conversation.
        # Pas un Bumble (femme DOIT parler en premier), mais un nudge :
        # "Tu as 3 matchs qui attendent. Envoie le premier message
        # et débloque un profil bonus demain."
        #
        # Ça aide les hommes (qui reçoivent un message, yay) et
        # les femmes (qui se sentent en contrôle).
        "enabled": True,
        "bonus_profiles_for_initiator": 2,
    },

    "real_time_ratio_monitoring": {
        # Dashboard admin avec alertes :
        # - Ratio H/F par ville, mis à jour quotidiennement
        # - Alerte si ratio > 75/25 : "Abidjan gender ratio critical: 78% male"
        # - Action recommandée : augmenter marketing ciblé femmes,
        #   partenariats avec communities féminines (salons de beauté,
        #   groupes WhatsApp, événements networking femmes)
        "alert_threshold": 0.75,
    },
}
```

---

## 33. Profils fantômes & anti-ghosting conversation (CRITIQUE)

```python
# app/services/ghost_detection_service.py
"""
Problème 1 : Profils fantômes
Des gens s'inscrivent et ne reviennent jamais. Leur profil reste
dans le pool, pollue les feeds, génère des matchs sans réponse.

Problème 2 : Ghosting post-match
Deux personnes matchent, l'une écrit, l'autre ne répond jamais.
Pire expérience possible. L'expiration à 7j est réactive, pas proactive.
"""

# ── Profils fantômes : détection et gestion ──

GHOST_PROFILE_RULES = {
    "inactive_threshold_days": 7,
    # Pas connecté depuis 7 jours → exclu des feeds (déjà en L1)
    # Mais on va plus loin :

    "zombie_threshold_days": 14,
    # 14 jours sans connexion → profil automatiquement mis en mode pause
    # (is_visible = false). Pas supprimé, juste caché.
    # Si l'utilisateur revient → réactivation automatique au login

    "dormant_threshold_days": 30,
    # 30 jours → notification push : "Tu nous manques ! Tes matchs t'attendent."
    # Si pas de retour après la notification → reste en pause

    "cleanup_threshold_days": 90,
    # 90 jours sans connexion → archivage
    # Le profil est retiré des compteurs de la ville
    # Les matchs en cours sont expirés
    # Si l'utilisateur revient → réactivation complète mais considéré
    # comme "returning user" (pas de boost)

    "never_active_threshold_hours": 48,
    # Inscrit mais jamais ouvert l'app après l'onboarding → 48h
    # Notification : "Tu as créé ton profil mais pas encore
    # découvert qui est autour de toi !"
    # Si pas de retour → ne JAMAIS montrer dans les feeds des autres
    # (profil complet mais intention = 0)
}

# ── Anti-ghosting post-match ──

ANTI_GHOSTING_RULES = {
    "first_message_nudge": {
        # 4h après le match sans message de personne :
        # Push aux deux : "Vous avez matché il y a 4h ! Qui brise la glace ?"
        "delay_hours": 4,
        "enabled": True,
    },

    "unanswered_message_nudge": {
        # A a envoyé un message, B n'a pas répondu après 24h :
        # Push à B : "Tu as un message non lu de {name}"
        # (une seule relance, pas de spam)
        "delay_hours": 24,
        "max_nudges": 1,
        "enabled": True,
    },

    "expiration_warning": {
        # 24h avant l'expiration du match (J+6 sur 7) :
        # Push aux deux : "Votre match expire demain. Dernière chance !"
        "hours_before_expiry": 24,
        "enabled": True,
    },

    "graceful_exit": {
        # Au lieu de juste "expirer" silencieusement, proposer :
        # "Ce match n'a pas donné suite. Pas grave !"
        # + bouton "Passer au suivant" (qui libère le slot du feed)
        # Ça normalise le fait de ne pas matcher avec tout le monde
        "enabled": True,
    },

    "response_rate_visibility": {
        # Sur le profil (visible uniquement par la personne elle-même) :
        # "Tu as répondu à 4/7 matchs cette semaine"
        # Pas visible par les autres (pas de shaming public)
        # Juste un nudge pour encourager à répondre
        "enabled": True,
    },

    "behavior_penalty": {
        # Les gens qui ne répondent JAMAIS à leurs matchs
        # voient leur behavior_multiplier baisser
        # → moins visibles dans les feeds des autres
        # C'est déjà dans L4 mais on le renforce :
        # 0 réponses sur les 5 derniers matchs = multiplier × 0.7
        "zero_response_threshold": 5,
        "penalty_factor": 0.7,
    },
}
```

---

## 34. Résilience réseau & mode offline-first (CRITIQUE)

```python
# Architecture côté client (Kotlin/Android) — spec pour le dev mobile

"""
En Afrique de l'Ouest, le réseau est :
- 2G/3G majoritaire (pas 4G partout)
- Coupures fréquentes (5-15 par jour)
- Latence haute (200-500ms)
- Data cher (500 FCFA/semaine pour certains forfaits)

L'app DOIT fonctionner en mode dégradé. Principe : offline-first.
Tout ce qui peut être lu depuis le cache local l'est.
Toute action est d'abord enregistrée localement puis sync.
"""

OFFLINE_FIRST_ARCHITECTURE = {

    "local_cache_room_db": {
        # Room DB locale stocke :
        # - Feed du jour (profils complets + photos thumbnail)
        # - Conversations actives (derniers 50 messages par match)
        # - Profil de l'utilisateur
        # - Quartiers et spots
        # - Matches actifs
        #
        # Taille estimée : 5-15 MB (largement dans les capacités)
        # Refresh : au login + pull-to-refresh + sync en background
    },

    "action_queue": {
        # Toutes les actions utilisateur sont enregistrées dans une queue locale
        # AVANT d'être envoyées au serveur. Format :
        # {action_id, type, payload, created_at, synced: false}
        #
        # Types d'actions queueables :
        # - like / skip (avec raison)
        # - message texte
        # - read receipt
        # - check-in
        # - profile update
        #
        # PAS queueable (besoin du serveur) :
        # - upload photo (trop lourd)
        # - paiement (besoin du provider)
        # - inscription (besoin OTP)
        #
        # Quand le réseau revient :
        # 1. Trier la queue par created_at (FIFO)
        # 2. Envoyer chaque action séquentiellement
        # 3. En cas d'erreur 409 (conflit) : ignorer (état déjà changé)
        # 4. Marquer comme synced: true
        # 5. Retry automatique toutes les 30s si le réseau est dispo
    },

    "idempotency": {
        # CHAQUE requête API a un header X-Idempotency-Key
        # Clé = hash(action_id) généré côté client
        # Le serveur stocke les clés traitées dans Redis (TTL 24h)
        # Si une requête arrive avec une clé déjà traitée → 200 avec le
        # résultat original (pas de double like, double message, etc.)
        #
        # Implémentation serveur :
        # 1. Vérifier Redis GET idempotency:{key}
        # 2. Si existe → retourner le résultat stocké
        # 3. Si n'existe pas → traiter la requête
        # 4. SET idempotency:{key} {response_json} EX 86400
    },

    "progressive_image_loading": {
        # Les photos sont le poste data #1. Stratégie :
        # 1. Feed : charger UNIQUEMENT les thumbnails (150px, ~5-10KB chacune)
        # 2. L'utilisateur tape sur un profil → charger le medium (600px, ~30-50KB)
        # 3. Jamais charger l'original en-app (réservé au CDN)
        #
        # Placeholder : rectangle flou couleur dominante (calculé côté serveur,
        # stocké comme champ dominant_color dans le modèle Photo)
        #
        # Mode data saver (activable dans les paramètres) :
        # - Charger uniquement la première photo de chaque profil
        # - Les autres photos se chargent au scroll
        # - Les messages vocaux affichent "Télécharger (34 KB)" au lieu d'auto-play
    },

    "network_state_detection": {
        # Le client détecte 3 états :
        # "online" → tout normal
        # "slow" → RTT > 2s ou throughput < 50KB/s → mode dégradé
        #   (pas de chargement d'images en background, queue plus agressive)
        # "offline" → pas de réseau → tout depuis le cache local
        #
        # Indicateur visuel discret : petit point dans le header
        # Vert = online, Jaune = slow, Rouge = offline
        # Pas de popup bloquant "Pas de connexion" (frustrant)
    },
}
```

```python
# ── Côté serveur : middleware idempotency ──

# app/core/idempotency.py
class IdempotencyMiddleware(BaseHTTPMiddleware):
    """
    Vérifie X-Idempotency-Key sur les requêtes mutatives (POST, PUT, PATCH, DELETE).
    Si la clé a déjà été traitée, retourne le résultat original.
    """
    MUTABLE_METHODS = {"POST", "PUT", "PATCH", "DELETE"}

    async def dispatch(self, request, call_next):
        if request.method not in self.MUTABLE_METHODS:
            return await call_next(request)

        key = request.headers.get("X-Idempotency-Key")
        if not key:
            return await call_next(request)

        redis = await get_redis()
        cached = await redis.get(f"idempotency:{key}")

        if cached:
            # Déjà traité → retourner le résultat original
            import json
            data = json.loads(cached)
            return JSONResponse(
                status_code=data["status"],
                content=data["body"],
            )

        # Traiter la requête
        response = await call_next(request)

        # Stocker le résultat (seulement pour les succès 2xx)
        if 200 <= response.status_code < 300:
            body = b""
            async for chunk in response.body_iterator:
                body += chunk
            import json
            await redis.set(
                f"idempotency:{key}",
                json.dumps({"status": response.status_code, "body": json.loads(body)}),
                ex=86400,  # 24h
            )
            return Response(
                content=body,
                status_code=response.status_code,
                headers=dict(response.headers),
            )

        return response
```

---

## 35. Fiabilité SMS & fallbacks OTP (CRITIQUE)

```python
# app/utils/sms.py
"""
Les SMS en Afrique de l'Ouest ne sont pas fiables à 100%.
Problèmes courants :
- Opérateur qui bloque les SMS transactionnels (Togocel parfois)
- Délai de livraison 1-10 minutes
- Numéros virtuels/MVNO mal routés
- Plafond de SMS/jour dépassé chez le provider

Solution : multi-provider avec fallback + canal alternatif.

MVP : Termii UNIQUEMENT. C'est le seul provider implémenté au lancement.
- OTP : SMS en PRIMARY (fiable sur n'importe quel numéro, pas besoin de WhatsApp).
  Raison : dual SIM courant au Togo (Togocel + Moov), WhatsApp souvent sur
  un seul des deux numéros. Un OTP WhatsApp qui ne livre pas = utilisateur perdu.
- OTP fallback : WhatsApp (si SMS échoue après 30s, ET que le numéro a WhatsApp).
  L'app propose "Renvoyer par WhatsApp ?" après 30s sans réception SMS.
- Messages event (rappels, teasers) : WhatsApp en primary (pas critique, plus riche,
  4x moins cher). SMS fallback si WhatsApp échoue.
- L'interface abstraite SMSDeliveryService est codée dès le début
  pour ajouter Twilio/Vonage plus tard sans refactor
- Mais les classes Twilio/Vonage ne sont PAS implémentées au MVP
- Flash call : désactivé (V2)

Tarifs Termii réels (avril 2026) :
- SMS : $0.0250/transaction (~15 FCFA)
- WhatsApp : $0.0060/transaction (~3.6 FCFA)
- Voice : $0.0102/min (~6 FCFA)
- Email : $0.001/transaction (~0.6 FCFA)
- Plan Starter : $0/mois (sandbox inclus pour le dev)

Coût estimé premier mois (500 inscriptions) :
- 500 OTP SMS × $0.025 = $12.50 (~7 500 FCFA)
- 200 messages event WhatsApp × $0.006 = $1.20 (~720 FCFA)
- Total : ~$14/mois (~8 400 FCFA). Négligeable.
"""

class SMSDeliveryService:
    """
    Cascade de 3 providers + 1 canal alternatif.
    Chaque provider est tenté avec un timeout.
    Si échec, on passe au suivant.
    """

    PROVIDERS = [
        {
            "name": "termii",
            "priority": 1,
            "timeout_seconds": 10,
            "countries": ["TG", "CI", "SN", "BJ", "NG", "GH"],
            # Spécialisé Afrique, meilleur tarif, meilleure délivrabilité locale
        },
        {
            "name": "twilio",
            "priority": 2,
            "timeout_seconds": 15,
            "countries": ["*"],  # Global fallback
            # Plus cher mais très fiable
        },
        {
            "name": "vonage",
            "priority": 3,
            "timeout_seconds": 15,
            "countries": ["*"],
            # Dernier recours SMS
        },
    ]

    ALTERNATIVE_CHANNELS = {
        "whatsapp_otp": {
            # Si les 3 SMS échouent → proposer l'OTP via WhatsApp
            # WhatsApp a une API Business pour envoyer des OTP
            # Avantage : quasi 100% de délivrabilité en Afrique de l'Ouest
            #            (tout le monde a WhatsApp)
            # Coût : ~0.05 USD par message (comparable au SMS)
            # Implémentation : WhatsApp Business API via 360dialog ou Meta directement
            "enabled": True,
            "provider": "meta_whatsapp_business",
        },
        "flash_call": {
            # Alternative gratuite : appel flash
            # L'app fait un appel qui raccroche immédiatement,
            # le numéro appelant contient le code OTP dans les derniers chiffres
            # Ex : appel depuis +228 90 XX 48 29 17 → code = 482917
            # Avantage : gratuit, ne dépend pas du SMS
            # Inconvénient : certains Android bloquent la détection des appels
            "enabled": False,  # V2
        },
    }

    async def send_otp(self, phone: str, code: str) -> dict:
        """
        Tente d'envoyer l'OTP par SMS via la cascade de providers.
        Si tous échouent, propose WhatsApp.
        """
        errors = []

        for provider in self.PROVIDERS:
            try:
                result = await self._send_via(provider["name"], phone, code,
                                               timeout=provider["timeout_seconds"])
                if result["delivered"]:
                    return {
                        "channel": "sms",
                        "provider": provider["name"],
                        "status": "sent",
                        "message_id": result["message_id"],
                    }
                errors.append({"provider": provider["name"], "error": result["error"]})
            except asyncio.TimeoutError:
                errors.append({"provider": provider["name"], "error": "timeout"})
            except Exception as e:
                errors.append({"provider": provider["name"], "error": str(e)})

        # Tous les SMS ont échoué → tenter WhatsApp
        if self.ALTERNATIVE_CHANNELS["whatsapp_otp"]["enabled"]:
            try:
                result = await self._send_whatsapp_otp(phone, code)
                return {
                    "channel": "whatsapp",
                    "provider": "meta",
                    "status": "sent",
                    "message_id": result["message_id"],
                }
            except Exception as e:
                errors.append({"provider": "whatsapp", "error": str(e)})

        # Tout a échoué
        return {
            "channel": None,
            "status": "failed",
            "errors": errors,
            "fallback_available": False,
            # Le client affiche : "Impossible d'envoyer le code.
            # Vérifie ton numéro et réessaie dans 1 minute."
        }


# ── Réponse API quand le SMS échoue ──
# POST /auth/otp/request
# Response 200 (envoyé par WhatsApp au lieu de SMS)
{
    "message": "OTP sent via WhatsApp",
    "channel": "whatsapp",
    "expires_in": 600,
    "retry_after": 60,
}
# Le client affiche : "Code envoyé sur WhatsApp" avec l'icône WhatsApp
# au lieu de l'icône SMS
```

---

## 36. Gestion des échecs de paiement mobile money (CRITIQUE)

```python
# app/services/payment_service.py
"""
Les paiements mobile money en Afrique de l'Ouest échouent SOUVENT :
- Timeout réseau pendant la transaction
- Solde insuffisant détecté après initiation
- Confirmation USSD non validée par l'utilisateur
- Webhook Paystack qui arrive en retard (5-30 min)
- Double débit si retry trop rapide

Principe : ne JAMAIS activer le premium sur la foi du client.
Attendre le webhook. Gérer le délai avec grâce.
"""

PAYMENT_STATES = {
    "initialized":   "Transaction initiée, en attente de paiement",
    "pending":       "Paiement soumis, en attente de confirmation provider",
    "processing":    "Provider confirme le traitement",
    "success":       "Paiement confirmé → premium activé",
    "failed":        "Paiement échoué définitivement",
    "timeout":       "Pas de réponse après 30 min → marqué failed",
    "refunded":      "Remboursé (double débit, erreur, etc.)",
}

class PaymentStateMachine:

    async def initialize_payment(self, user_id, plan, method, provider) -> dict:
        """
        Étape 1 : Créer la transaction côté serveur.
        """
        # Vérifier que l'utilisateur n'a pas déjà un paiement pending
        existing = await get_pending_payment(user_id)
        if existing:
            age_minutes = (now - existing.created_at).total_seconds() / 60
            if age_minutes < 30:
                return {
                    "error": "payment_already_pending",
                    "message": "You have a pending payment. Please wait or try again in "
                               f"{int(30 - age_minutes)} minutes.",
                    "existing_reference": existing.reference,
                }
            else:
                # Timeout l'ancien
                existing.status = "timeout"
                await db_session.commit()

        # Générer une référence unique (idempotency)
        reference = f"flaam_{user_id}_{uuid4().hex[:8]}"

        # Créer la transaction locale
        payment = Payment(
            user_id=user_id,
            reference=reference,
            amount=plan_amount,
            currency=currency,
            provider=provider,
            payment_method=method,
            status="initialized",
        )
        db_session.add(payment)
        await db_session.commit()

        # Initier chez Paystack
        paystack_response = await paystack_client.initialize(
            reference=reference,
            amount=plan_amount * 100,  # Paystack veut des kobo/centimes
            currency=currency,
            email=f"{user_id}@flaam.app",  # Email factice (Paystack le requiert)
            channels=[method],  # ["mobile_money"]
        )

        return {
            "reference": reference,
            "authorization_url": paystack_response["authorization_url"],
            "status": "initialized",
            "expires_in_minutes": 30,
        }

    async def handle_webhook(self, event: str, data: dict) -> dict:
        """
        Étape 2 : Webhook Paystack confirme le paiement.
        Peut arriver 1 seconde ou 30 minutes après l'initiation.
        """
        reference = data["reference"]
        payment = await get_payment_by_reference(reference)

        if not payment:
            return {"error": "unknown_reference"}

        if payment.status == "success":
            # Idempotent : déjà traité (webhook reçu en double)
            return {"status": "already_processed"}

        if event == "charge.success":
            # Vérifier le montant (sécurité)
            if data["amount"] != payment.amount * 100:
                await flag_suspicious_payment(payment)
                return {"error": "amount_mismatch"}

            payment.status = "success"
            payment.provider_reference = data.get("id")
            payment.confirmed_at = datetime.now(timezone.utc)

            # ACTIVER LE PREMIUM
            await activate_premium(payment.user_id, payment.plan)

            # Notification push
            await send_push(payment.user_id, "premium_activated")

        elif event == "charge.failed":
            payment.status = "failed"
            payment.failure_reason = data.get("gateway_response", "unknown")

            # Notification push
            await send_push(payment.user_id, "payment_failed",
                          reason=payment.failure_reason)

        await db_session.commit()
        return {"status": "processed"}


# ── Ce que voit l'utilisateur pendant l'attente ──

# GET /subscriptions/me (pendant que le paiement est pending)
PENDING_PAYMENT_RESPONSE = {
    "subscription": None,  # Pas encore premium
    "pending_payment": {
        "reference": "flaam_uuid_abc12345",
        "status": "pending",
        "initiated_at": "2026-04-14T15:00:00Z",
        "message": "Paiement en cours de vérification. Ça peut prendre jusqu'à 30 minutes avec mobile money.",
        "auto_timeout_at": "2026-04-14T15:30:00Z",
    },
}
# Le client affiche un écran d'attente avec un spinner
# + "Ton paiement est en cours. Tu peux fermer l'app,
#    on t'enverra une notification dès que c'est confirmé."
```

---

## 37. WebSocket — Protocole de reconnexion & sync (CRITIQUE)

```python
# app/api/ws/chat.py (protocole complet)
"""
Le WebSocket se déconnecte CONSTAMMENT sur réseau mobile africain.
On doit gérer :
1. Reconnexion automatique avec backoff exponentiel
2. Sync des messages manqués
3. Détection des doublons
4. Indicateurs de statut fiables (envoyé/reçu/lu)
"""

WEBSOCKET_PROTOCOL = {

    "connection": {
        # Client → ws://api.flaam.app/ws/chat?token=JWT
        # Le serveur vérifie le JWT et enregistre la connexion
        # Redis SET ws:online:{user_id} {server_instance_id}
        # Heartbeat : client envoie {"type": "ping"} toutes les 30s
        # Serveur répond {"type": "pong"}
        # Si pas de ping pendant 60s → serveur ferme la connexion
        # → Redis DEL ws:online:{user_id}
    },

    "reconnection_client_side": {
        # Quand le WS se ferme (réseau coupé, serveur restart, timeout) :
        # 1. Attendre 1s → tenter reconnexion
        # 2. Si échoue → attendre 2s → retry
        # 3. Si échoue → 4s → 8s → 16s → max 30s
        # 4. Backoff exponentiel avec jitter aléatoire (±500ms)
        # 5. Quand le réseau revient (détecté par NetworkCallback Android) → retry immédiat
        # 6. Pendant la déconnexion : queue locale des messages à envoyer
        "initial_delay_ms": 1000,
        "max_delay_ms": 30000,
        "backoff_factor": 2,
        "jitter_ms": 500,
        "max_retries": None,  # Infini, on arrête jamais de retry
    },

    "sync_on_reconnect": {
        # Dès que le WS est rétabli, le client envoie :
        # {"type": "sync", "last_message_id": "uuid-du-dernier-message-reçu"}
        #
        # Le serveur répond avec tous les messages manqués :
        # {"type": "sync_response", "messages": [...], "missed_count": 3}
        #
        # Le client intègre les messages dans la conversation locale
        # en évitant les doublons (par message_id unique)
    },

    "message_delivery_status": {
        # Chaque message passe par 4 statuts :
        #
        # "sending"  → le message est dans la queue locale (pas encore envoyé)
        #              UI : icône horloge ⏱
        #
        # "sent"     → le serveur a reçu et stocké le message
        #              Le serveur répond : {"type": "ack", "message_id": "uuid",
        #                                   "status": "sent", "server_timestamp": "..."}
        #              UI : icône check simple ✓
        #
        # "delivered" → le message est arrivé sur le device du destinataire
        #               Le serveur envoie quand le destinataire reçoit via WS
        #               ou quand il pull les messages manqués
        #               UI : double check ✓✓
        #
        # "read"     → le destinataire a ouvert la conversation
        #              Le client du destinataire envoie :
        #              {"type": "read", "match_id": "uuid", "last_read_id": "uuid"}
        #              UI : double check bleu ✓✓
        #
        # Si le destinataire est offline :
        # - Le message est stocké en DB et Redis
        # - Une push notification est envoyée
        # - Le statut reste "sent" jusqu'au retour du destinataire
        # - Au retour → sync → "delivered"
    },

    "duplicate_prevention": {
        # Le client génère un client_message_id (UUID v4) pour chaque message
        # Ce client_message_id est envoyé avec le message
        # Le serveur vérifie en Redis : SET message_dedup:{client_message_id} NX EX 86400
        # Si la clé existe déjà → message ignoré (doublon de retry)
        # Retourne quand même un ACK (le client doit savoir que c'est OK)
    },

    "offline_message_flow": {
        # Client A envoie un message à Client B (offline) :
        # 1. A → WS → Serveur : message reçu et stocké
        # 2. Serveur → A : ACK "sent" ✓
        # 3. Serveur vérifie Redis ws:online:{B} → absent → B est offline
        # 4. Serveur → FCM push à B : "Nouveau message de A"
        # 5. Serveur stocke en Redis recent_msgs:{match_id} (LPUSH)
        # 6. B ouvre l'app (1 heure plus tard)
        # 7. B → WS connect → {"type": "sync", "last_message_id": "..."}
        # 8. Serveur → B : {"type": "sync_response", "messages": [{...}]}
        # 9. Serveur → A : {"type": "delivered", "message_id": "uuid"} ✓✓
        # 10. B ouvre la conversation
        # 11. B → WS : {"type": "read", "match_id": "...", "last_read_id": "..."}
        # 12. Serveur → A : {"type": "read", "message_id": "uuid"} ✓✓ bleu
    },
}
```

---

## 38. Timezones (PRODUCTION)

```python
# app/core/timezone.py
"""
Flaam est multi-ville, multi-timezone.
Lomé = UTC+0, Lagos = UTC+1, Abidjan = UTC+0, Accra = UTC+0.

Règle d'or : TOUT est stocké en UTC en DB et Redis.
La conversion en heure locale se fait au dernier moment (notification, API response).
"""

# ── Batch nocturne par timezone ──
# Le batch de 3h UTC n'est pas bon pour Lagos (= 4h, OK) mais si on
# ajoute Nairobi un jour (UTC+3), 3h UTC = 6h du matin = trop tard.
# Solution : le batch tourne par ville, à l'heure locale configurée.

BATCH_SCHEDULE = {
    # Le Celery Beat est en UTC. On calcule l'heure UTC du batch
    # pour chaque ville en fonction de son timezone.
    # Heure locale cible : 3h du matin (activité minimale)

    # Lomé (UTC+0)    → batch à 03:00 UTC
    # Abidjan (UTC+0) → batch à 03:00 UTC (même heure, mutualisé)
    # Accra (UTC+0)   → batch à 03:00 UTC
    # Lagos (UTC+1)   → batch à 02:00 UTC (= 3h locale)
    # Dakar (UTC+0)   → batch à 03:00 UTC
    # Nairobi (UTC+3) → batch à 00:00 UTC (= 3h locale)

    # Implémentation : une tâche Celery Beat par timezone distincte
    # "generate_feeds_utc0": crontab(hour=3, minute=0),
    # "generate_feeds_utc1": crontab(hour=2, minute=0),
    # Chaque tâche charge les villes de sa timezone et génère les feeds
}

# ── Notifications à l'heure locale ──
# La notification "Tes profils du jour" respecte l'heure choisie
# par l'utilisateur (default 9h) dans SON timezone.
# Le serveur calcule : user_preferred_hour - city_utc_offset = UTC hour
# Et schedule la notification à cette heure UTC.

# ── Heures dans les réponses API ──
# Toutes les dates retournées par l'API sont en UTC (ISO 8601 avec Z).
# Le client convertit en heure locale en utilisant le timezone du device.
# On NE fait PAS la conversion côté serveur (le client sait mieux).
# Exception : les notifications push contiennent l'heure formatée
# en heure locale pour que la preview soit lisible.
```

---

## 39. Détection de romance scam / brouteurs (CRITIQUE)

```python
# app/services/scam_detection_service.py
"""
L'Afrique de l'Ouest est connue pour les romance scams (broutage).
Pattern classique :
1. Profil attrayant (souvent vrai visage, pas de catfish)
2. Conversation chaleureuse pendant 1-3 semaines
3. Création d'un lien émotionnel
4. Prétexte : urgence médicale, voyage, problème familial
5. Demande d'argent

La vérification selfie NE PROTÈGE PAS contre ça car les scammers
utilisent leur propre visage. Il faut de la détection comportementale.
"""

SCAM_DETECTION_SIGNALS = {

    "conversation_velocity": {
        # Signal : messages très longs, très émotionnels, très tôt
        # Les scammers écrivent des pavés romantiques dès le premier message
        # pour créer un attachement rapide
        #
        # Heuristique :
        # avg_message_length > 200 chars dans les 10 premiers messages
        # ET contient des mots émotionnels ("love", "amour", "heart", "miss you")
        # → flag "fast_emotional_escalation"
        "weight": 0.3,
    },

    "money_mention_timing": {
        # Signal : mention d'argent dans les 7 premiers jours de conversation
        # Les vrais utilisateurs ne parlent pas d'argent si tôt
        #
        # Mots clés surveillés (multi-langue) :
        # FR: "envoie", "transfert", "momo", "urgence", "hôpital", "accident",
        #     "aide moi", "prête moi", "western union"
        # EN: "send me", "transfer", "urgent", "hospital", "help me",
        #     "lend me", "emergency"
        # Ewe/Mina: adaptations locales
        #
        # Si détecté dans les 7 premiers jours → flag fort
        "weight": 0.5,
    },

    "one_sided_conversation": {
        # Signal : le scammer écrit beaucoup, la victime écrit peu
        # Ratio messages envoyés / reçus > 3:1 sur les 20 derniers messages
        # Les scammers bombardent pour maintenir l'engagement
        "weight": 0.2,
    },

    "multiple_concurrent_deep_conversations": {
        # Signal : la même personne a 5+ conversations actives simultanément
        # avec des messages longs et émotionnels
        # Un vrai utilisateur peut chatter avec 2-3 matchs mais pas 10
        # avec la même intensité
        #
        # Heuristique :
        # count(conversations avec >20 messages cette semaine) > 5
        # → flag "mass_romance"
        "weight": 0.4,
    },

    "profile_too_perfect": {
        # Signal : photos très professionnelles (studio lighting, poses)
        # sur un profil avec peu de contenu organique
        # Les scammers utilisent des photos très attrayantes
        #
        # Pas de ML pour ça en MVP, mais :
        # - Reverse image search sur les photos (Google Vision API, ponctuel)
        # - Si une photo apparaît ailleurs sur internet → flag
        "weight": 0.3,
    },

    "external_link_sharing": {
        # Signal : partager des liens vers WhatsApp, Telegram, etc. tôt
        # Les scammers veulent quitter l'app rapidement pour éviter
        # la modération et la traçabilité
        #
        # Mots : "mon whatsapp", "écris moi sur", "telegram", "+228"
        # dans les 5 premiers messages → flag
        "weight": 0.3,
    },
}


async def compute_scam_risk(user_id: UUID, db_session) -> float:
    """
    Calcule un score de risque de scam 0.0 → 1.0.
    Appelé par un cron quotidien sur tous les utilisateurs actifs.
    Si score > 0.7 → review manuelle obligatoire.
    Si score > 0.9 → suspension automatique.
    """
    score = 0.0

    # Vérifier chaque signal
    for signal_name, config in SCAM_DETECTION_SIGNALS.items():
        detected = await check_signal(signal_name, user_id, db_session)
        if detected:
            score += config["weight"]

    # Facteurs atténuants
    # - Profil vérifié pièce d'identité : -0.2
    # - Compte > 3 mois sans flag : -0.1
    # - A eu des dates confirmés (meetup accepted) : -0.3

    return min(1.0, max(0.0, score))
```

---

## 40. Google Play Store compliance (PRODUCTION)

```python
# docs/play_store_compliance.py
"""
Google Play a des règles strictes pour les apps de dating.
Non-compliance = rejection ou retrait. Checklist obligatoire.
"""

PLAY_STORE_REQUIREMENTS = {

    "age_verification": {
        # Google exige une vérification d'âge pour les apps de dating
        # Notre implémentation : date de naissance obligatoire à l'inscription
        # + déclaration que l'utilisateur a 18+
        # Google n'exige pas de vérification documentaire mais on l'offre
        # comme option (badge vérifié)
        "status": "covered",
        "implementation": "birth_date in Profile + 18+ check",
    },

    "content_rating": {
        # L'app doit être classée "Mature 17+" ou équivalent
        # Car c'est une app de dating
        "status": "to_configure",
        "rating": "Mature 17+",
        "questionnaire": "Fill out the IARC rating questionnaire on Play Console",
    },

    "user_generated_content_policy": {
        # Google exige :
        # 1. Modération du contenu (photos, messages) → couvert
        # 2. Mécanisme de signalement → couvert
        # 3. Blocage des utilisateurs → couvert
        # 4. Mention dans les CGU que le contenu inapproprié est interdit
        "status": "covered",
    },

    "privacy_policy": {
        # Obligatoire. Doit couvrir :
        # - Quelles données sont collectées
        # - Comment elles sont utilisées
        # - Avec qui elles sont partagées
        # - Comment les supprimer (RGPD)
        # - Politique de cookies (si web)
        # URL publique requise dans le listing Play Store
        "status": "to_write",
        "url": "https://flaam.app/privacy",
    },

    "data_safety_section": {
        # Formulaire Play Console qui déclare :
        "data_collected": {
            "phone_number": "Required for account creation, not shared",
            "location": "Optional, used for check-in, not tracked continuously",
            "photos": "Required for profile, stored on our servers",
            "messages": "Required for chat, stored encrypted",
            "device_id": "Used for security and fraud prevention",
        },
        "data_shared": "None shared with third parties",
        "data_deletion": "Users can delete all data from Settings",
        "encryption_in_transit": True,
    },

    "no_deceptive_practices": {
        # Google interdit :
        # - Fake profiles / bot accounts → on n'en a pas
        # - Misleading subscription practices → prix clairs, annulation facile
        # - Forced in-app purchases → l'app est utilisable en gratuit
        "status": "covered",
    },

    "subscription_guidelines": {
        # Google exige :
        # - Prix clairement affiché avant achat
        # - Politique d'annulation claire
        # - Pas de dark patterns (pas de "X" caché pour refuser)
        # - Respecter la politique de remboursement Play Store
        #
        # Note : si on utilise le billing Google Play, Google prend 15-30%.
        # Avec Paystack (mobile money), on bypass ça.
        # MAIS Google pourrait exiger le Play Billing pour les achats in-app.
        # Vérifier la politique actuelle au moment du submit.
        "status": "to_verify",
        "risk": "Google might require Play Billing for subscriptions",
        "mitigation": "Paystack handles mobile money outside Play Billing. "
                      "Google's policy allows alternative billing in some regions.",
    },

    "target_audience": {
        # Ne PAS inclure les mineurs dans l'audience cible
        # Configurer dans Play Console : "Not designed for children"
        "designed_for_children": False,
        "min_age": 18,
    },

    "app_permissions": {
        # Déclarer uniquement les permissions nécessaires :
        "INTERNET": "Required",
        "ACCESS_FINE_LOCATION": "Optional, for check-in",
        "CAMERA": "Required, for selfie verification",
        "READ_CONTACTS": "Optional, for contact blacklist",
        "POST_NOTIFICATIONS": "Required, for push notifications",
        # NE PAS demander : BACKGROUND_LOCATION, READ_SMS, RECORD_AUDIO
        # sauf si justifié (les vocaux utilisent MediaRecorder, pas RECORD_AUDIO direct)
    },
}
```

---

# MISE À JOUR — Villes/Pays + Auth sans mot de passe + Email

> À intégrer dans flaam-backend-spec.md et flaam-mobile-spec.md

---

## BACKEND — Mise à jour du modèle City

```python
# app/models/city.py (MIS À JOUR)
"""
Chaque ville a un status qui détermine ce que l'utilisateur voit.
L'écran de sélection de ville affiche TOUTES les villes du pays
sauf celles en status "hidden".
"""

class CityPhase(str, Enum):
    HIDDEN = "hidden"     # Pas affichée du tout (pas dans les plans immédiats)
    TEASER = "teaser"     # Affichée grisée, waitlist active, compteur visible
    LAUNCH = "launch"     # Active, matching activé, boost agressif
    GROWTH = "growth"     # Active, marketing en cours
    STABLE = "stable"     # Active, maturité atteinte


class City(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "cities"

    name: Mapped[str] = mapped_column(String(100), nullable=False)
    country_code: Mapped[str] = mapped_column(String(2), nullable=False)
    # ISO 3166-1 alpha-2 : "TG" = Togo, "CI" = Côte d'Ivoire, etc.

    country_name: Mapped[str] = mapped_column(String(100), nullable=False)
    # "Togo", "Côte d'Ivoire" — pour affichage

    country_flag: Mapped[str] = mapped_column(String(10), nullable=False)
    # Emoji drapeau : "🇹🇬", "🇨🇮"

    phone_prefix: Mapped[str] = mapped_column(String(5), nullable=False)
    # "+228", "+225" — pour auto-détection du pays

    timezone: Mapped[str] = mapped_column(String(50), nullable=False)
    # "Africa/Lome", "Africa/Abidjan"

    phase: Mapped[str] = mapped_column(String(20), default="hidden")
    # hidden | teaser | launch | growth | stable

    display_order: Mapped[int] = mapped_column(Integer, default=0)
    # Ordre d'affichage dans le picker (villes actives en premier)

    # ── Waitlist (pour les villes en teaser) ──
    waitlist_threshold: Mapped[int] = mapped_column(Integer, default=500)
    # Nombre d'inscrits nécessaires pour passer en launch


# ── API : liste des villes par pays ──

# GET /cities?country=TG
# Retourne TOUTES les villes du Togo SAUF celles en status "hidden"
# Triées par display_order (actives d'abord, teaser ensuite)

# Response :
{
    "country_code": "TG",
    "country_name": "Togo",
    "country_flag": "🇹🇬",
    "cities": [
        {
            "id": "uuid-lome",
            "name": "Lomé",
            "phase": "launch",        # ← active, sélectionnable
            "selectable": True,
        },
        {
            "id": "uuid-kara",
            "name": "Kara",
            "phase": "teaser",        # ← grisée, waitlist
            "selectable": False,
            "waitlist": {
                "total_registered": 127,
                "threshold": 500,
                "remaining": 373,
            },
        },
        {
            "id": "uuid-sokode",
            "name": "Sokodé",
            "phase": "teaser",
            "selectable": False,
            "waitlist": {
                "total_registered": 43,
                "threshold": 500,
                "remaining": 457,
            },
        },
    ]
}


# GET /cities (sans filtre pays)
# Retourne TOUS les pays qui ont au moins 1 ville non-hidden
# Utilisé quand quelqu'un tape "Pas au Togo ?" pour changer de pays

# Response :
{
    "countries": [
        {
            "country_code": "TG",
            "country_name": "Togo",
            "country_flag": "🇹🇬",
            "phone_prefix": "+228",
            "active_cities_count": 1,   # Villes en launch/growth/stable
            "teaser_cities_count": 2,   # Villes en teaser
        },
        {
            "country_code": "CI",
            "country_name": "Côte d'Ivoire",
            "country_flag": "🇨🇮",
            "phone_prefix": "+225",
            "active_cities_count": 1,
            "teaser_cities_count": 1,
        },
        # ...
    ]
}


# POST /cities/{cityId}/waitlist/join
# Quand quelqu'un sélectionne une ville en teaser
# L'utilisateur est inscrit en waitlist pour cette ville
# Il peut continuer l'onboarding normalement avec cette ville
# mais son feed sera vide (mode waitlist) tant que la ville n'est pas active
```

---

## BACKEND — Auth sans mot de passe + Email recovery

```python
# app/models/user.py (CHAMPS AJOUTÉS)
"""
Modèle d'authentification : PASSWORDLESS (comme WhatsApp)
- Auth principale : OTP par SMS/WhatsApp
- Pas de mot de passe. Jamais.
- Email optionnel mais fortement encouragé (recovery + notifications)
- MFA optionnel : PIN à 6 chiffres

Pourquoi pas de mot de passe :
1. Cible Afrique de l'Ouest → beaucoup d'utilisateurs pas habitués aux mots de passe
2. Moins de friction à l'inscription (pas de "8 caractères dont 1 majuscule...")
3. Pas de "mot de passe oublié" (le problème #1 du support)
4. Le téléphone EST le facteur d'authentification (comme WhatsApp, Telegram)
"""

class User(Base, UUIDMixin, TimestampMixin):
    # ... (champs existants)

    # ── Email (optionnel mais encouragé) ──
    email: Mapped[str | None] = mapped_column(String(255), nullable=True, unique=True)
    is_email_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    email_verified_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    # ── MFA optionnel ──
    mfa_enabled: Mapped[bool] = mapped_column(Boolean, default=False)
    mfa_pin_hash: Mapped[str | None] = mapped_column(String(128), nullable=True)
    # PIN à 6 chiffres, hashé bcrypt. Demandé à chaque login si activé.
    # PAS un mot de passe — c'est un PIN court pour empêcher quelqu'un
    # qui a accès au téléphone de se connecter sans le code.

    # ── Recovery ──
    recovery_email: Mapped[str | None] = mapped_column(String(255), nullable=True)
    # Peut être différent de l'email principal
    # Utilisé UNIQUEMENT pour la récupération de compte


# ── Scénarios d'authentification ──

AUTH_SCENARIOS = {

    "inscription_normale": {
        # 1. Entrer le numéro
        # 2. Recevoir l'OTP (SMS ou WhatsApp)
        # 3. Vérifier l'OTP
        # 4. Compte créé → JWT émis
        # 5. Plus tard dans l'onboarding ou les settings :
        #    "Ajoute ton email pour sécuriser ton compte" (encouragé mais pas bloquant)
    },

    "connexion_normale": {
        # 1. Entrer le numéro
        # 2. OTP
        # 3. Si MFA activé → demander le PIN à 6 chiffres
        # 4. JWT émis
    },

    "changement_de_telephone": {
        # L'utilisateur a un nouveau numéro mais le même téléphone (ou un nouveau)
        #
        # Option A : depuis l'app (connecté)
        # Settings > Compte > Changer de numéro
        # 1. Vérifier l'ancien numéro (OTP)
        # 2. Entrer le nouveau numéro
        # 3. Vérifier le nouveau numéro (OTP)
        # 4. Numéro mis à jour
        #
        # Option B : perdu l'accès au téléphone
        # → voir "recovery" ci-dessous
    },

    "recovery_numero_perdu": {
        # L'utilisateur a perdu son téléphone ET changé de numéro.
        # Il ne peut plus recevoir d'OTP sur l'ancien numéro.
        #
        # PRÉREQUIS : avoir ajouté un email de récupération
        #
        # Flow :
        # 1. Écran login → "J'ai perdu mon numéro"
        # 2. Entrer l'email de récupération
        # 3. Recevoir un lien de récupération par email (expire en 1h)
        # 4. Cliquer le lien → ouvre l'app
        # 5. Entrer le nouveau numéro
        # 6. Vérifier le nouveau numéro (OTP)
        # 7. Ancien numéro remplacé, JWT émis
        #
        # SANS email de récupération :
        # → Support manuel (support@flaam.app)
        # → Vérification d'identité (selfie + pièce d'identité)
        # → Délai 24-48h
    },

    "mfa_pin": {
        # Optionnel. Activable dans Settings > Sécurité.
        # PIN à 6 chiffres demandé à chaque nouveau login.
        # Pas demandé si le JWT est encore valide (l'app est déjà ouverte).
        #
        # Cas d'usage : empêcher un collègue/ami qui a accès au téléphone
        # de se connecter à Flaam avec un OTP intercepté.
        #
        # PIN oublié → recovery via email
    },
}


# ── API endpoints ──

# POST /auth/email/add
# { "email": "raouf@example.com" }
# → Envoie un email de vérification avec un lien
# → Le lien contient un token (expire en 24h)

# POST /auth/email/verify
# { "token": "abc123..." }
# → Marque l'email comme vérifié

# POST /auth/recovery/request
# { "email": "raouf@example.com" }
# → Envoie un lien de récupération (expire en 1h)
# → Retourne 200 même si l'email n'existe pas (sécurité)

# POST /auth/recovery/confirm
# { "recovery_token": "abc123...", "new_phone": "+22890123456" }
# → Déclenche un OTP vers le nouveau numéro

# POST /auth/recovery/complete
# { "recovery_token": "abc123...", "otp": "482917" }
# → Ancien numéro remplacé, JWT émis

# POST /auth/mfa/enable
# { "pin": "142857" }
# → Hash le PIN, active le MFA

# POST /auth/mfa/verify
# { "pin": "142857" }
# → Appelé après l'OTP si MFA est activé
# → Si OK → JWT émis

# POST /auth/mfa/disable
# { "pin": "142857" }  (doit fournir le PIN actuel pour désactiver)


# ── Utilisation de l'email (au-delà de la recovery) ──

EMAIL_USES = {
    "account_recovery": True,       # Toujours
    "payment_receipts": True,       # Reçu après chaque paiement premium
    "security_alerts": True,        # "Nouvelle connexion depuis un device inconnu"
    "weekly_digest": "opt_in",      # "Tu as 3 nouveaux likes cette semaine"
    "event_invitations": "opt_in",  # "Afterwork vendredi à Tokoin"
    "marketing": "opt_in",          # "Nouvelle feature : messages vocaux !"
    # Les opt_in sont gérés dans NotificationPreference
    # L'utilisateur choisit dans Settings > Notifications > Email
}
```

---

## MOBILE — Mise à jour écran Ville

```kotlin
/**
 * L'écran ville apparaît TOUJOURS dans l'onboarding.
 * Il montre toutes les villes du pays (détecté via l'indicatif).
 *
 * 3 états visuels :
 * - Active (vert) → tappable, continue l'onboarding normalement
 * - Teaser (grisé + badge "Bientôt") → tappable, inscrit en waitlist
 * - Hidden → pas affichée
 */

@HiltViewModel
class CitySelectionViewModel @Inject constructor(
    private val cityApi: CityApi,
    private val userPrefs: UserPrefsDataStore,
) : ViewModel() {

    data class CityUiState(
        val countryCode: String = "TG",
        val countryName: String = "Togo",
        val countryFlag: String = "🇹🇬",
        val cities: List<CityUi> = emptyList(),
        val selectedCityId: String? = null,
        val isLoading: Boolean = true,
        val showCountryPicker: Boolean = false,
    )

    data class CityUi(
        val id: String,
        val name: String,
        val phase: String,          // "launch", "growth", "stable", "teaser"
        val selectable: Boolean,    // true si active
        val waitlistCount: Int? = null,
        val waitlistThreshold: Int? = null,
    )

    private val _state = MutableStateFlow(CityUiState())
    val state = _state.asStateFlow()

    fun detectCountryFromPhone(phonePrefix: String) {
        // "+228" → "TG", "+225" → "CI", etc.
        val countryCode = PhonePrefixMap.getCountry(phonePrefix)
        loadCities(countryCode)
    }

    private fun loadCities(countryCode: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val response = cityApi.getCitiesByCountry(countryCode)
                if (response.isSuccessful) {
                    val body = response.body()!!
                    _state.update { it.copy(
                        countryCode = body.countryCode,
                        countryName = body.countryName,
                        countryFlag = body.countryFlag,
                        cities = body.cities.map { c -> CityUi(
                            id = c.id,
                            name = c.name,
                            phase = c.phase,
                            selectable = c.selectable,
                            waitlistCount = c.waitlist?.totalRegistered,
                            waitlistThreshold = c.waitlist?.threshold,
                        )},
                        isLoading = false,
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false) }
            }
        }
    }

    fun selectCity(cityId: String) {
        val city = _state.value.cities.find { it.id == cityId } ?: return

        if (city.selectable) {
            // Ville active → continuer l'onboarding
            _state.update { it.copy(selectedCityId = cityId) }
            viewModelScope.launch { userPrefs.saveCityId(cityId) }
        } else {
            // Ville en teaser → inscrire en waitlist
            viewModelScope.launch {
                try {
                    cityApi.joinWaitlist(cityId)
                    _state.update { it.copy(selectedCityId = cityId) }
                    userPrefs.saveCityId(cityId)
                    // L'utilisateur continue l'onboarding normalement
                    // mais son feed sera en mode waitlist
                } catch (_: Exception) { }
            }
        }
    }

    fun changeCountry() {
        _state.update { it.copy(showCountryPicker = true) }
    }
}
```

---

## MOBILE — Mise à jour onboarding : ajout de l'écran Email

```kotlin
/**
 * Après l'onboarding principal (profil complet), on propose
 * d'ajouter un email. C'est OPTIONNEL mais fortement encouragé.
 *
 * Cet écran apparaît dans l'écran "C'est prêt !" (écran 14)
 * comme un encart, PAS comme un écran bloquant séparé.
 *
 * Le texte dit :
 * "Ajoute ton email pour sécuriser ton compte.
 *  Si tu perds ton numéro, c'est le seul moyen de récupérer ton compte."
 */

// Modifié dans OnboardingStep : PAS de nouvelle étape
// L'email est proposé dans l'écran Done, pas un écran séparé

// Settings > Sécurité ajoute ces options :
SECURITY_SETTINGS = {
    "email_recovery": {
        // "Email de récupération" → tap → écran pour ajouter/changer l'email
        // Si pas encore ajouté : badge orange "Recommandé"
        // Si ajouté et vérifié : badge vert "Actif"
    },
    "mfa_pin": {
        // "Code PIN" → tap → activer/désactiver le PIN 6 chiffres
        // Si activé : toggle vert
        // Si désactivé : texte "Ajoute un PIN pour protéger ton compte"
    },
    "change_phone": {
        // "Changer de numéro" → tap → flow vérification ancien + nouveau
    },
}


// ── Écran de login : cas "numéro perdu" ──

// Sur l'écran Phone (écran 02), ajouter un lien discret en bas :
// "J'ai perdu mon numéro" → ouvre un écran :
//
// 1. "Entre l'email associé à ton compte Flaam"
// 2. Si l'email existe → "Un lien de récupération a été envoyé"
// 3. Si l'email n'existe pas → même message (sécurité)
//    + "Si tu n'as pas d'email associé, contacte support@flaam.app"
// 4. L'utilisateur clique le lien dans son email
// 5. Deep link : flaam://recovery?token=abc123
// 6. L'app s'ouvre sur l'écran "Nouveau numéro"
// 7. Il entre son nouveau numéro → OTP → connecté
```

---

## MOBILE — Mise à jour Retrofit : nouveaux endpoints

```kotlin
// Ajoutés à AuthApi

interface AuthApi {
    // ... (existants)

    // ── Email ──
    @POST("auth/email/add")
    suspend fun addEmail(@Body body: AddEmailBody): Response<Unit>
    // { "email": "raouf@example.com" }

    @POST("auth/email/verify")
    suspend fun verifyEmail(@Body body: VerifyEmailBody): Response<Unit>
    // { "token": "abc123..." }

    // ── Recovery ──
    @POST("auth/recovery/request")
    suspend fun requestRecovery(@Body body: RecoveryRequestBody): Response<Unit>
    // { "email": "raouf@example.com" }

    @POST("auth/recovery/confirm")
    suspend fun confirmRecovery(@Body body: RecoveryConfirmBody): Response<Unit>
    // { "recovery_token": "abc123...", "new_phone": "+22890123456" }

    @POST("auth/recovery/complete")
    suspend fun completeRecovery(@Body body: RecoveryCompleteBody): Response<AuthTokenResponse>
    // { "recovery_token": "abc123...", "otp": "482917" }

    // ── MFA ──
    @POST("auth/mfa/enable")
    suspend fun enableMfa(@Body body: MfaPinBody): Response<Unit>
    // { "pin": "142857" }

    @POST("auth/mfa/verify")
    suspend fun verifyMfa(@Body body: MfaPinBody): Response<AuthTokenResponse>
    // { "pin": "142857" }

    @POST("auth/mfa/disable")
    suspend fun disableMfa(@Body body: MfaPinBody): Response<Unit>

    // ── Phone change ──
    @POST("auth/phone/change/verify-old")
    suspend fun verifyOldPhone(@Body body: OtpVerifyBody): Response<PhoneChangeTokenResponse>

    @POST("auth/phone/change/set-new")
    suspend fun setNewPhone(@Body body: SetNewPhoneBody): Response<AuthTokenResponse>
    // { "change_token": "...", "new_phone": "+228...", "otp": "..." }
}


// Ajouté à CityApi

interface CityApi {
    // ... (existants)

    @GET("cities")
    suspend fun getCitiesByCountry(
        @Query("country") countryCode: String,
    ): Response<CitiesByCountryResponse>

    @GET("cities/countries")
    suspend fun getAvailableCountries(): Response<CountriesResponse>

    @POST("cities/{cityId}/waitlist/join")
    suspend fun joinWaitlist(
        @Path("cityId") cityId: String,
    ): Response<WaitlistJoinResponse>
}
```


---

# MISE À JOUR 2 — Email hors onboarding + Limites spots

---

## CORRECTION : Email retiré de l'onboarding

```python
# AVANT (FAUX) :
# L'email était proposé dans l'écran "C'est prêt !" (écran 14)
#
# APRÈS (CORRECT) :
# L'email n'apparaît NULLE PART dans l'onboarding.
# Il vit uniquement dans Settings > Sécurité > Email de récupération
#
# L'onboarding reste :
# Phone → OTP → Ville → Infos → Selfie → Photos → Quartiers →
# Intention → Secteur → Prompts → Tags → Spots → C'est prêt
#
# 0 mention d'email. 0 mention de PIN. 0 friction supplémentaire.
#
# Dans Settings > Sécurité, l'utilisateur voit :
# - "Email de récupération" avec badge orange "Recommandé" si pas encore ajouté
# - "Code PIN" avec toggle désactivé par défaut
# - "Changer de numéro"
# - "Vérification selfie" (statut)
#
# On peut envoyer une notification push 7 jours après l'inscription :
# "Sécurise ton compte ! Ajoute un email de récupération au cas où tu perds ton numéro."
# Ça pousse vers Settings > Sécurité sans polluer l'onboarding.

SECURITY_SETTINGS_SCREEN = {
    "sections": [
        {
            "title": "Protection du compte",
            "items": [
                {
                    "label": "Email de récupération",
                    "type": "navigation",
                    "badge": "Recommandé",          # Orange si pas encore ajouté
                    "badge_type": "warning",
                    "value_if_set": "r***f@example.com",  # Masqué
                    "value_if_not_set": "Non configuré",
                    "description": "Si tu perds ton numéro, c'est le seul moyen de récupérer ton compte.",
                },
                {
                    "label": "Code PIN",
                    "type": "toggle",
                    "default": False,
                    "description": "Un code à 6 chiffres demandé à chaque connexion.",
                },
                {
                    "label": "Changer de numéro",
                    "type": "navigation",
                    "value": "+228 90 ** 56",
                },
            ],
        },
        {
            "title": "Vérification",
            "items": [
                {
                    "label": "Selfie vérifié",
                    "type": "status",
                    "value": "Vérifié",             # Vert
                    "badge_type": "success",
                },
            ],
        },
        {
            "title": "Contact de confiance",
            "items": [
                {
                    "label": "Contact pour les dates",
                    "type": "navigation",
                    "value_if_set": "Maman",
                    "value_if_not_set": "Non configuré",
                    "description": "Reçoit un SMS quand tu partages un date + alerte si le timer expire.",
                },
            ],
        },
        {
            "title": "Données",
            "items": [
                {
                    "label": "Contacts masqués",
                    "type": "navigation",
                    "value": "3 contacts",
                },
                {
                    "label": "Exporter mes données",
                    "type": "navigation",
                },
                {
                    "label": "Supprimer mon compte",
                    "type": "navigation",
                    "destructive": True,
                },
            ],
        },
    ],
}
```

---

## AJOUT : Limites des spots

```python
# app/constants.py (AJOUT)

SPOT_LIMITS = {
    "free": {
        "max_spots": 5,
        # L'utilisateur free peut déclarer jusqu'à 5 spots sur son profil
        # Au-delà → message "Passe à Premium pour ajouter plus de spots"
    },
    "premium": {
        "max_spots": 12,
        # Premium peut avoir jusqu'à 12 spots
        # Suffisant pour couvrir ses habitudes sans diluer le signal
    },

    # Pourquoi limiter :
    # 1. Signal géo : si quelqu'un a 50 spots, le score géo "spots en commun"
    #    perd sa valeur (tout le monde a des spots en commun avec lui)
    # 2. Levier premium : "ajoute plus de spots" est un upsell naturel
    # 3. Anti-gaming : empêche de tagger tous les spots de la ville pour
    #    maximiser artificiellement la visibilité
}

# Le check-in n'est pas limité — tu peux faire autant de check-ins
# que tu veux dans tes spots existants. C'est le NOMBRE de spots
# sur ton profil qui est limité, pas l'activité.


# ── Backend : vérification à l'ajout d'un spot ──

async def add_spot_to_profile(user_id, spot_id, db_session):
    user = await get_user(user_id)
    current_spots = await count_user_spots(user_id, db_session)

    max_spots = SPOT_LIMITS["premium"]["max_spots"] if user.is_premium \
        else SPOT_LIMITS["free"]["max_spots"]

    if current_spots >= max_spots:
        if user.is_premium:
            raise SpotLimitReached("Tu as atteint la limite de 12 spots.")
        else:
            raise SpotLimitReached(
                "Tu as atteint la limite de 5 spots. "
                "Passe à Premium pour en ajouter jusqu'à 12."
            )

    # Ajouter le spot
    # ...


# ── Mobile : ce que voit l'utilisateur ──

# Écran Spots (tab) :
# En bas de la liste des spots, le compteur :
# "3 / 5 spots" (free) ou "7 / 12 spots" (premium)
#
# Quand le max est atteint et qu'il tape "+ Ajouter" :
# Free → bottom sheet upsell premium
# Premium → message "Tu as atteint la limite de 12 spots.
#           Retire un spot existant pour en ajouter un nouveau."

# Écran Onboarding spots (écran 13) :
# Même limite appliquée. Le compteur est visible.
# Mais pendant l'onboarding on affiche pas l'upsell premium
# (trop tôt, l'utilisateur ne comprend pas encore la valeur).
```


---

# MISE À JOUR 3 — Events : cycle de vie complet

---

## Qui crée les events ?

```python
# ── V1 (MVP) : Admin uniquement ──
#
# Les events sont du contenu ÉDITORIAL. Pas du user-generated content.
# L'équipe Flaam crée les events en partenariat avec des lieux locaux.
#
# Processus humain :
# 1. L'équipe Flaam contacte un lieu (Café 21, Salle Olympe, Resto Le Palmier...)
# 2. On se met d'accord : date, heure, capacité max, description
# 3. L'admin crée l'event via POST /admin/events
# 4. L'event apparaît dans le feed Events de la ville
# 5. Les utilisateurs s'inscrivent
# 6. L'équipe Flaam peut participer sur place (community management)
#
# Pourquoi pas user-generated en V1 :
# - Risque de spam (faux events pour attirer des gens)
# - Risque de sécurité (events dans des lieux privés)
# - Pas assez d'utilisateurs pour que les events user-generated soient viables
# - La curation crée une meilleure expérience (qualité > quantité)
#
#
# ── V2 : Partenaires vérifiés ──
#
# Les lieux partenaires (vérifiés par l'équipe) peuvent proposer des events.
# Flow : le partenaire propose → admin review → publié ou rejeté.
# Les utilisateurs normaux ne peuvent PAS créer d'events.
#
#
# ── V3 (futur lointain) : Utilisateurs ──
#
# Les utilisateurs premium pourraient proposer des "mini-events"
# (ex: "Je vais courir sur la plage samedi à 7h, qui vient ?")
# Mais c'est très loin. Pas dans le scope actuel.


# ── Modèle Event (COMPLÉTÉ) ──

class EventStatus(str, Enum):
    DRAFT = "draft"           # Créé par l'admin, pas encore publié
    PUBLISHED = "published"   # Visible dans l'app
    FULL = "full"             # Capacité max atteinte, inscription fermée
    ONGOING = "ongoing"       # En cours (le jour J, pendant les heures)
    COMPLETED = "completed"   # Terminé
    CANCELLED = "cancelled"   # Annulé par l'admin


class EventCategory(str, Enum):
    AFTERWORK = "afterwork"       # Soirée décontractée, bar/café
    SPORT = "sport"               # Running, yoga, foot, gym
    BRUNCH = "brunch"             # Brunch du dimanche
    CULTURAL = "cultural"         # Expo, ciné, concert
    NETWORKING = "networking"     # Rencontres pro
    WORKSHOP = "workshop"         # Atelier (cuisine, art, danse)
    OUTDOOR = "outdoor"           # Plage, rando, pique-nique


class Event(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "events"

    # ── Infos de base ──
    title: Mapped[str] = mapped_column(String(200), nullable=False)
    description: Mapped[str] = mapped_column(Text, nullable=False)
    category: Mapped[str] = mapped_column(String(30), nullable=False)
    # afterwork | sport | brunch | cultural | networking | workshop | outdoor

    # ── Lieu ──
    city_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("cities.id"))
    spot_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("spots.id"), nullable=True)
    # Si l'event est dans un spot connu de l'app → lien direct
    # Sinon → adresse libre ci-dessous
    venue_name: Mapped[str] = mapped_column(String(200), nullable=False)
    venue_address: Mapped[str | None] = mapped_column(String(500), nullable=True)

    # ── Date et heure ──
    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    ends_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)

    # ── Capacité ──
    max_capacity: Mapped[int | None] = mapped_column(Integer, nullable=True)
    # Null = pas de limite (rare, déconseillé)
    current_registrations: Mapped[int] = mapped_column(Integer, default=0)

    # ── Status ──
    status: Mapped[str] = mapped_column(String(20), default="draft")
    # draft → published → full/ongoing → completed/cancelled

    # ── Visuel ──
    cover_image_url: Mapped[str | None] = mapped_column(String(500), nullable=True)
    # Image de couverture (uploadée par l'admin)
    cover_color: Mapped[str] = mapped_column(String(7), default="#D85A30")
    # Couleur de fallback si pas d'image (gradient dans l'UI)

    # ── Sponsoring ──
    is_sponsored: Mapped[bool] = mapped_column(Boolean, default=False)
    # Les events sponsorisés ont un badge "Sponsorisé" dans l'UI
    # Le sponsor paie pour la visibilité (pas implémenté en V1, juste le flag)
    sponsor_name: Mapped[str | None] = mapped_column(String(200), nullable=True)

    # ── Admin ──
    created_by_admin_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    # L'admin qui a créé l'event

    # ── Prix ──
    is_free: Mapped[bool] = mapped_column(Boolean, default=True)
    price_amount: Mapped[int | None] = mapped_column(Integer, nullable=True)
    # En FCFA. Null si gratuit.
    price_currency: Mapped[str] = mapped_column(String(3), default="XOF")
    # La plupart des events V1 seront gratuits. Le prix c'est pour V2
    # quand les partenaires organisent des events payants.


# ── Cycle de vie d'un event ──

EVENT_LIFECYCLE = {
    "creation": {
        # L'admin crée l'event (POST /admin/events)
        # Status = "draft"
        # L'event n'est PAS visible dans l'app
    },
    "publication": {
        # L'admin publie l'event (PATCH /admin/events/{id} status=published)
        # Status = "draft" → "published"
        # L'event apparaît dans le feed Events de la ville
        # Notification push aux utilisateurs de la ville :
        # "Nouvel event : {title} ce {jour} !"
    },
    "inscription": {
        # Les utilisateurs s'inscrivent (POST /events/{id}/register)
        # current_registrations += 1
        # Si current_registrations >= max_capacity → status = "full"
        # L'utilisateur reçoit une confirmation
    },
    "desinscription": {
        # DELETE /events/{id}/register
        # current_registrations -= 1
        # Si status était "full" et qu'une place se libère → "published"
    },
    "rappel": {
        # Celery task 2h avant starts_at :
        # Push aux inscrits : "{title} dans 2h ! Rendez-vous à {venue_name}"
    },
    "en_cours": {
        # Celery task à starts_at :
        # Status = "published/full" → "ongoing"
        # (aucun changement visible pour l'utilisateur, c'est pour les stats)
    },
    "termine": {
        # Celery task à ends_at :
        # Status = "ongoing" → "completed"
        # L'event disparaît du feed Events (mais reste dans l'historique)
    },
    "annulation": {
        # L'admin annule (PATCH /admin/events/{id} status=cancelled)
        # Push aux inscrits : "{title} est annulé. Désolé !"
        # L'event disparaît du feed
    },
}


# ── Matches-preview : le truc qui fait cliquer ──

# GET /events/{event_id}/matches-preview
#
# Retourne le nombre de matchs POTENTIELS parmi les inscrits.
# C'est le feature qui différencie Flaam des apps d'events classiques.
#
# Logique :
# 1. Prendre la liste des inscrits à l'event
# 2. Pour chaque inscrit, calculer le score géo + lifestyle avec le user courant
# 3. Compter ceux qui auraient un score > 0.5
# 4. Retourner le nombre (pas les profils — ça c'est du teasing)
#
# Response :
# { "potential_matches": 3, "total_attendees": 12 }
#
# L'UI affiche : "3 matchs potentiels parmi les 12 inscrits"
# C'est le CTA le plus puissant pour pousser l'inscription.
#
# Note de privacy : on ne révèle PAS qui sont ces matchs potentiels.
# L'utilisateur doit s'inscrire et venir pour le découvrir.


# ── Admin API pour les events ──

# POST /admin/events
# Body :
{
    "title": "Afterwork Tokoin",
    "description": "Venez comme vous êtes pour un afterwork décontracté...",
    "category": "afterwork",
    "city_id": "uuid-lome",
    "spot_id": "uuid-cafe-21",           # Optionnel, si c'est un spot connu
    "venue_name": "Café 21",
    "venue_address": "Rue du Commerce, Tokoin",
    "starts_at": "2026-04-18T18:00:00+00:00",
    "ends_at": "2026-04-18T21:00:00+00:00",
    "max_capacity": 30,
    "is_free": True,
    "is_sponsored": False,
    "cover_color": "#E8784A",             # Gradient terracotta
}
# Response : 201 Created avec l'event en status "draft"

# PATCH /admin/events/{event_id}
# Pour publier, annuler, ou modifier un event
{
    "status": "published"
}

# GET /admin/events?city_id={id}&status=draft
# Liste les events par statut (pour le dashboard admin)

# DELETE /admin/events/{event_id}
# Supprimer un event (uniquement si draft ou cancelled)


# ── Celery tasks pour les events ──

EVENT_TASKS = {
    "event_reminder": {
        # Tourne toutes les 15 minutes
        # Cherche les events qui commencent dans les 2 prochaines heures
        # Envoie une push aux inscrits qui n'ont pas encore reçu le rappel
        "schedule": "*/15 * * * *",
    },
    "event_status_updater": {
        # Tourne toutes les 15 minutes
        # Met à jour les status :
        # - published/full → ongoing (si starts_at est passé)
        # - ongoing → completed (si ends_at est passé)
        "schedule": "*/15 * * * *",
    },
    "weekly_event_digest": {
        # Tourne le dimanche soir (par timezone)
        # Push aux utilisateurs : "Cette semaine : 3 events à Lomé"
        # Seulement si au moins 1 event est publié pour la semaine
        "schedule": "dimanche 20h (heure locale)",
    },
}


# ── Ce qui est prévu pour V2 (PAS dans le scope actuel) ──

V2_EVENT_FEATURES = {
    "partner_proposed_events": {
        # Les spots partenaires (Café 21, Salle Olympe) ont un compte "partenaire"
        # Ils proposent des events via POST /partner/events
        # L'admin review et approve/reject
    },
    "paid_events": {
        # Certains events ont un prix (ex: atelier cuisine 5000 FCFA)
        # L'inscription déclenche un paiement mobile money
        # Le partenaire reçoit le paiement moins la commission Flaam (15%)
    },
    "recurring_events": {
        # "Running plage tous les samedis à 7h"
        # L'admin crée un template et les occurrences sont auto-générées
    },
    "event_photos": {
        # Après l'event, l'admin upload des photos
        # Les inscrits reçoivent une notification : "Les photos de l'Afterwork sont là !"
        # Ça crée du contenu et de l'engagement post-event
    },
}
```


---

# MISE À JOUR 4 — Algo L3+L4 manquants + Mobile events

---

## BACKEND — Section 6.3b : L3 — Score lifestyle (MANQUAIT)

```python
# app/services/matching_engine/lifestyle_scorer.py
"""
L3 : Score de compatibilité lifestyle.

Composantes :
1. Tags TF-IDF (Jaccard pondéré)
2. Matrice compatibilité intentions (4×4)
3. Similitude rythme de vie
4. Bonus langues communes

Score final normalisé entre 0.0 et 1.0.
Poids configurables via MatchingConfig.
"""

from app.core.constants import INTENTION_COMPATIBILITY_MATRIX


async def compute_lifestyle_scores(
    user,
    candidate_profiles: list,
    config: dict,
) -> dict[str, float]:
    """
    Retourne {candidate_id: lifestyle_score} pour chaque candidat.
    """
    scores = {}

    for candidate in candidate_profiles:
        score = 0.0

        # ── 1. Tags Jaccard pondéré (50% du score lifestyle) ──
        # Jaccard = |A ∩ B| / |A ∪ B|
        # Pondéré : les tags rares valent plus que les tags communs
        # Un tag que 5% des users ont vaut plus qu'un tag que 80% ont
        user_tags = set(user.profile.tags or [])
        candidate_tags = set(candidate.tags or [])

        if user_tags and candidate_tags:
            intersection = user_tags & candidate_tags
            union = user_tags | candidate_tags
            if union:
                # Jaccard simple
                jaccard = len(intersection) / len(union)
                # Pondération par rareté (TF-IDF simplifié)
                # Les tags rares en commun comptent plus
                # tag_weight = 1 / log(1 + frequency_in_city)
                # Pour le MVP : Jaccard simple suffit
                score += jaccard * config.get("lifestyle_w_tags", 0.50)

        # ── 2. Matrice intentions (25% du score lifestyle) ──
        # 4 intentions : serious, getting_to_know, friendship, open
        # Matrice de compatibilité :
        #                serious  getting_to_know  friendship  open
        # serious          1.0        0.5            0.1       0.7
        # getting_to_know  0.5        1.0            0.6       0.8
        # friendship       0.1        0.6            1.0       0.5
        # open             0.7        0.8            0.5       1.0
        user_intention = user.profile.intention
        candidate_intention = candidate.intention

        intention_score = INTENTION_COMPATIBILITY_MATRIX.get(
            user_intention, {}
        ).get(candidate_intention, 0.5)

        score += intention_score * config.get("lifestyle_w_intention", 0.25)

        # ── 3. Rythme de vie (15% du score lifestyle) ──
        # "early_bird" vs "night_owl" vs "flexible"
        # Même rythme = 1.0, flexible + anything = 0.7, opposés = 0.3
        user_rhythm = user.profile.rhythm or "flexible"
        candidate_rhythm = candidate.rhythm or "flexible"

        if user_rhythm == candidate_rhythm:
            rhythm_score = 1.0
        elif "flexible" in (user_rhythm, candidate_rhythm):
            rhythm_score = 0.7
        else:
            rhythm_score = 0.3  # early_bird vs night_owl

        score += rhythm_score * config.get("lifestyle_w_rhythm", 0.15)

        # ── 4. Langues communes (10% du score lifestyle) ──
        # Bonus si on partage au moins une langue
        # Plus de langues en commun = meilleur score
        user_langs = set(user.profile.languages or [])
        candidate_langs = set(candidate.languages or [])

        if user_langs and candidate_langs:
            common_langs = user_langs & candidate_langs
            if common_langs:
                # Au moins 1 langue commune → score proportionnel
                lang_score = min(1.0, len(common_langs) / 2)
                # 1 langue = 0.5, 2+ langues = 1.0
            else:
                lang_score = 0.0  # Pas de langue commune = problème
        else:
            lang_score = 0.5  # Données manquantes → neutre

        score += lang_score * config.get("lifestyle_w_languages", 0.10)

        # ── Normalisation ──
        scores[str(candidate.user_id)] = min(1.0, max(0.0, score))

    return scores


# ── Constante : matrice d'intentions ──

INTENTION_COMPATIBILITY_MATRIX = {
    "serious": {
        "serious": 1.0,
        "getting_to_know": 0.5,
        "friendship": 0.1,
        "open": 0.7,
    },
    "getting_to_know": {
        "serious": 0.5,
        "getting_to_know": 1.0,
        "friendship": 0.6,
        "open": 0.8,
    },
    "friendship": {
        "serious": 0.1,
        "getting_to_know": 0.6,
        "friendship": 1.0,
        "open": 0.5,
    },
    "open": {
        "serious": 0.7,
        "getting_to_know": 0.8,
        "friendship": 0.5,
        "open": 1.0,
    },
}
```

---

## BACKEND — Section 6.3c : L4 — Multiplicateur comportemental (MANQUAIT)

```python
# app/services/matching_engine/behavior_scorer.py
"""
L4 : Multiplicateur comportemental.

Pas un "score" mais un MULTIPLICATEUR appliqué au score final.
Récompense les utilisateurs qui utilisent l'app de manière saine.
Pénalise ceux qui spamment ou ne répondent jamais.

4 composantes, chacune produit un facteur entre min et max.
Le multiplicateur final = produit des 4 facteurs.
Résultat typique : 0.6 → 1.4 (borné par config).

Stocké dans Redis (behavior:{user_id}) pour accès rapide.
Persisté en DB (profiles.behavior_multiplier) par cron horaire.
"""


async def get_behavior_multipliers(
    candidate_ids: list[str],
    redis_client,
    db_session,
) -> dict[str, float]:
    """
    Retourne {candidate_id: multiplier} pour chaque candidat.
    Source : Redis d'abord, fallback DB.
    """
    multipliers = {}

    # Batch Redis GET
    pipe = redis_client.pipeline()
    for cid in candidate_ids:
        pipe.get(f"behavior:{cid}")
    results = await pipe.execute()

    for cid, cached in zip(candidate_ids, results):
        if cached is not None:
            multipliers[cid] = float(cached)
        else:
            # Fallback DB
            profile = await get_profile(cid, db_session)
            multipliers[cid] = profile.behavior_multiplier if profile else 1.0

    return multipliers


async def update_behavior_on_action(
    user_id: str,
    action_type: str,
    metadata: dict,
    redis_client,
    db_session,
    config: dict,
):
    """
    Appelé à chaque action utilisateur (like, skip, message envoyé,
    message reçu, match, etc.) pour mettre à jour le behavior score.

    Les 4 composantes :

    1. RESPONSE QUALITY (répond-il à ses matchs ?)
       - Compteur : messages_received / messages_sent_first
       - Bon = répond vite et régulièrement
       - Mauvais = ne répond jamais, ghost ses matchs
       - Range : config["behavior_response_min"] (0.6) → config["behavior_response_max"] (1.4)

    2. SELECTIVITY (est-il sélectif dans ses likes ?)
       - Compteur : likes / total_profiles_seen
       - Bon = like 20-40% des profils (sélectif mais actif)
       - Mauvais = like 100% (spam) ou 0% (inactif)
       - Range : config["behavior_selectivity_min"] (0.7) → config["behavior_selectivity_max"] (1.3)

    3. PROFILE RICHNESS (profil complet ?)
       - Score de complétion : photos, prompts, tags, spots
       - Bon = profil complet avec 3+ prompts, 5+ spots
       - Mauvais = profil minimal (3 photos, 0 prompts, 0 spots)
       - Range : config["behavior_richness_min"] (0.8) → config["behavior_richness_max"] (1.2)

    4. CONVERSATION DEPTH (qualité des conversations ?)
       - Moyenne : messages par match actif
       - Bon = conversations longues avec échanges réels
       - Mauvais = "salut" et plus rien
       - Range : config["behavior_depth_min"] (0.8) → config["behavior_depth_max"] (1.3)
    """

    # Lire les compteurs actuels depuis Redis
    stats_key = f"behavior_stats:{user_id}"
    stats = await redis_client.hgetall(stats_key)

    # Mettre à jour selon l'action
    if action_type == "like":
        await redis_client.hincrby(stats_key, "total_likes", 1)
    elif action_type == "skip":
        await redis_client.hincrby(stats_key, "total_skips", 1)
    elif action_type == "message_sent":
        await redis_client.hincrby(stats_key, "messages_sent", 1)
    elif action_type == "message_received":
        await redis_client.hincrby(stats_key, "messages_received", 1)
    elif action_type == "match_response":
        await redis_client.hincrby(stats_key, "matches_responded", 1)
    elif action_type == "profile_viewed":
        await redis_client.hincrby(stats_key, "profiles_viewed", 1)

    # Recalculer les 4 composantes
    total_likes = int(stats.get("total_likes", 0)) + (1 if action_type == "like" else 0)
    total_profiles = int(stats.get("profiles_viewed", 0)) + (1 if action_type == "profile_viewed" else 0)
    messages_sent = int(stats.get("messages_sent", 0))
    messages_received = int(stats.get("messages_received", 0))
    matches_responded = int(stats.get("matches_responded", 0))
    total_matches = int(stats.get("total_matches", 0))

    # 1. Response quality
    if total_matches > 0:
        response_rate = matches_responded / total_matches
        response_factor = lerp(
            config["behavior_response_min"],
            config["behavior_response_max"],
            min(1.0, response_rate),
        )
    else:
        response_factor = 1.0  # Pas encore de matchs → neutre

    # 2. Selectivity
    if total_profiles > 10:  # Minimum 10 profils vus pour évaluer
        like_rate = total_likes / total_profiles
        # Sweet spot = 20-40%. Trop haut ou trop bas = pénalité
        if 0.15 <= like_rate <= 0.45:
            selectivity_factor = config["behavior_selectivity_max"]
        elif like_rate > 0.80:  # Like tout = spam
            selectivity_factor = config["behavior_selectivity_min"]
        elif like_rate < 0.05:  # Like personne = inactif
            selectivity_factor = config["behavior_selectivity_min"]
        else:
            selectivity_factor = 1.0
    else:
        selectivity_factor = 1.0

    # 3. Richness (lire depuis le profil en DB, c'est statique)
    profile = await get_profile(user_id, db_session)
    completeness = calculate_completeness(profile)
    richness_factor = lerp(
        config["behavior_richness_min"],
        config["behavior_richness_max"],
        completeness,
    )

    # 4. Conversation depth
    if messages_sent > 0 and total_matches > 0:
        avg_messages_per_match = messages_sent / max(1, total_matches)
        # 5+ messages par match = bon, 1 message = mauvais
        depth_factor = lerp(
            config["behavior_depth_min"],
            config["behavior_depth_max"],
            min(1.0, avg_messages_per_match / 5),
        )
    else:
        depth_factor = 1.0

    # Multiplicateur final = produit des 4 facteurs
    multiplier = response_factor * selectivity_factor * richness_factor * depth_factor

    # Borner entre 0.6 et 1.4 (sécurité)
    multiplier = max(0.6, min(1.4, multiplier))

    # Stocker dans Redis
    await redis_client.set(f"behavior:{user_id}", str(round(multiplier, 3)))

    return multiplier


def lerp(min_val: float, max_val: float, t: float) -> float:
    """Interpolation linéaire entre min et max."""
    return min_val + (max_val - min_val) * max(0.0, min(1.0, t))
```

---

## MOBILE — Mise à jour Events

```kotlin
/**
 * Le mobile doit connaître les catégories et status d'events
 * pour l'affichage correct dans l'UI.
 */

// ── Enums côté client ──

enum class EventCategory(val labelFr: String, val labelEn: String) {
    AFTERWORK("Afterwork", "Afterwork"),
    SPORT("Sport", "Sport"),
    BRUNCH("Brunch", "Brunch"),
    CULTURAL("Culture", "Cultural"),
    NETWORKING("Networking", "Networking"),
    WORKSHOP("Atelier", "Workshop"),
    OUTDOOR("Plein air", "Outdoor"),
}

enum class EventStatus {
    PUBLISHED,   // Visible, inscription possible
    FULL,        // Visible, inscription fermée (capacité max)
    ONGOING,     // En cours (le jour J)
    COMPLETED,   // Terminé (disparaît du feed)
    CANCELLED,   // Annulé
}


// ── EventsViewModel (complété) ──

@HiltViewModel
class EventsViewModel @Inject constructor(
    private val eventApi: EventApi,
    private val userPrefs: UserPrefsDataStore,
) : ViewModel() {

    data class EventsUiState(
        val events: List<EventUi> = emptyList(),
        val selectedCategory: EventCategory? = null, // null = "Tout"
        val isLoading: Boolean = true,
        val error: String? = null,
    )

    data class EventUi(
        val id: String,
        val title: String,
        val description: String,
        val category: EventCategory,
        val venueName: String,
        val startsAt: Instant,
        val endsAt: Instant,
        val maxCapacity: Int?,
        val currentRegistrations: Int,
        val isFull: Boolean,
        val isSponsored: Boolean,
        val sponsorName: String?,
        val coverImageUrl: String?,
        val coverColor: String,
        val potentialMatches: Int?, // null si pas encore chargé
        val isRegistered: Boolean,
        val dayLabel: String, // "Vendredi", "Samedi", "Dimanche"
        val timeLabel: String, // "18h — 21h"
    )

    private val _state = MutableStateFlow(EventsUiState())
    val state = _state.asStateFlow()

    init { loadEvents() }

    private fun loadEvents() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val cityId = userPrefs.getCityId()
                val response = eventApi.getEvents(cityId = cityId)
                if (response.isSuccessful) {
                    _state.update { it.copy(
                        events = response.body()!!.events.map { e -> e.toUi() },
                        isLoading = false,
                    )}
                }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.toFlaamError().toUserMessage()) }
            }
        }
    }

    fun filterByCategory(category: EventCategory?) {
        _state.update { it.copy(selectedCategory = category) }
    }

    fun register(eventId: String) {
        viewModelScope.launch {
            try {
                val response = eventApi.registerEvent(eventId)
                if (response.isSuccessful) {
                    _state.update { state ->
                        state.copy(events = state.events.map { e ->
                            if (e.id == eventId) e.copy(
                                isRegistered = true,
                                currentRegistrations = e.currentRegistrations + 1,
                                isFull = e.maxCapacity != null &&
                                         e.currentRegistrations + 1 >= e.maxCapacity,
                            ) else e
                        })
                    }
                }
            } catch (_: Exception) { }
        }
    }

    fun unregister(eventId: String) {
        viewModelScope.launch {
            try {
                eventApi.unregisterEvent(eventId)
                _state.update { state ->
                    state.copy(events = state.events.map { e ->
                        if (e.id == eventId) e.copy(
                            isRegistered = false,
                            currentRegistrations = e.currentRegistrations - 1,
                            isFull = false,
                        ) else e
                    })
                }
            } catch (_: Exception) { }
        }
    }
}
```

---

# MISE À JOUR 5 — Philosophie de l'algo + feedback loop

---

## Philosophie fondamentale du matching

```python
"""
CE QUE L'ALGO N'EST PAS :

1. PAS de scoring physique. L'algo n'analyse JAMAIS les photos pour
   déterminer l'attractivité. C'est ce que fait Tinder avec son ancien
   ELO et c'est toxique. Flaam matche sur le mode de vie, les lieux,
   le comportement — jamais sur l'apparence.

2. PAS de punition. Un score qui baisse c'est une dé-priorisation douce,
   jamais un mur. Et il remonte dès que le comportement change.

3. PAS d'enfermement. Les wildcards (L5) et l'équité de visibilité
   garantissent que l'expérience ne devient jamais prédictible.

4. PAS un score qu'on optimise. L'objectif c'est de maximiser la
   probabilité d'une VRAIE CONVERSATION, pas de maximiser les swipes
   ou le temps passé dans l'app. On optimise le RÉSULTAT (sortir de
   l'app pour aller au date), pas l'ENGAGEMENT (rester dans l'app).

CE QUE L'UTILISATEUR VOIT :
- "82% zones communes" = c'est JUSTE la composante géo (L2)
- Le vrai ranking est multi-dimensionnel et jamais réductible à un chiffre
- Le score affiché est un teaser, pas le classement réel

ANTI-GAMING — pourquoi personne ne peut reverse-engineer l'algo :
1. Pas de score unique exposé
2. Decay temporel (inactif = perd en visibilité doucement)
3. Les poids entre les couches changent avec la maturité de l'utilisateur
4. Le feedback implicite pèse plus que l'explicite (impossible à simuler)
"""


# ── Feedback loop : implicite > explicite ──

FEEDBACK_CONFIG = {
    # Les signaux IMPLICITES sont plus honnêtes qu'un like/skip binaire.
    # Personne ne peut les simuler consciemment.
    
    "implicit_signals": {
        "time_on_profile": {
            "description": "Secondes passées sur un profil",
            "weight": 2.0,  # 2x le poids d'un signal explicite
            "threshold": 8,  # > 8 secondes = intérêt réel
        },
        "photos_scrolled": {
            "description": "Nombre de photos regardées (1-6)",
            "weight": 1.5,
            "threshold": 3,  # 3+ photos = intérêt
        },
        "prompts_read": {
            "description": "A tapé 'voir plus' sur un prompt",
            "weight": 2.0,
            "threshold": 1,  # 1 prompt lu = signal fort
        },
        "return_visit": {
            "description": "Est revenu voir le même profil le lendemain",
            "weight": 3.0,  # Le signal le plus fort
            "threshold": 1,
        },
        "scroll_depth": {
            "description": "A scrollé jusqu'au bas du profil",
            "weight": 1.5,
            "threshold": 0.8,  # 80% du profil vu
        },
    },
    
    "explicit_signals": {
        "like": {
            "weight": 1.0,  # Poids de base
        },
        "skip": {
            "weight": 1.0,
            # Le skip avec raison pèse plus que le skip sans raison
            "with_reason_multiplier": 1.5,
        },
    },
    
    # Les signaux implicites sont envoyés en batch via POST /behavior/log
    # (mobile spec section 23)
    # Les signaux explicites sont envoyés immédiatement (POST /feed/{id}/like)
}


# ── Weight schedule adaptatif ──

WEIGHT_SCHEDULE = {
    # Les pondérations entre les couches ne sont PAS fixes.
    # Elles évoluent avec la maturité de l'utilisateur sur l'app.
    
    # Jour → (geo_weight, lifestyle_weight, behavior_weight)
    # Le total fait toujours 1.0 (normalisé)
    
    "new_user_0_30_days": {
        "geo_weight": 0.55,        # La géo pèse lourd au début
        "lifestyle_weight": 0.35,  # Le lifestyle complète
        "behavior_weight": 0.10,   # Pas assez de data pour juger
        "rationale": "Nouveau = on se fie à la géo et aux déclarations",
    },
    
    "established_30_90_days": {
        "geo_weight": 0.40,
        "lifestyle_weight": 0.30,
        "behavior_weight": 0.30,   # Le comportement commence à compter
        "rationale": "On a assez de data pour évaluer le comportement",
    },
    
    "mature_90_plus_days": {
        "geo_weight": 0.30,
        "lifestyle_weight": 0.25,
        "behavior_weight": 0.45,   # Le comportement domine
        "rationale": "L'algo s'adapte à toi, pas toi à l'algo",
    },
    
    # Implémentation : le pipeline.py lit l'âge du compte et
    # sélectionne la ligne correspondante pour pondérer les scores.
    # Stocké dans matching_configs avec les clés :
    # "weight_schedule_geo_0_30", "weight_schedule_lifestyle_0_30", etc.
}
```


---

# MISE À JOUR 6 — Préférences implicites + sanitisation des signaux

---

## Préférences implicites (content-based, PAS collaborative filtering)

```python
# app/services/matching_engine/implicit_preferences.py
"""
Construit un "profil type implicite" à partir des signaux comportementaux.
Ce profil ajuste le score L3 (lifestyle) pour personnaliser le feed.

PAS du collaborative filtering ("les gens comme toi aiment aussi...").
C'est du content-based pur : on regarde ce que TU as regardé en silence,
et on en déduit ce que tu préfères.

Pourquoi pas du collaborative filtering :
1. Pas assez d'utilisateurs au lancement (cold start)
2. Boîte noire impossible à debug
3. Crée des bulles plus fortes que le content-based
"""

import math
from datetime import datetime, timedelta
from collections import defaultdict


# ── Sanitisation des signaux (CRITIQUE) ──

SIGNAL_CONFIG = {
    "time_on_profile": {
        # Plafonnement logarithmique : au-delà de 60s = bruit
        # Formule : signal = min(1.0, log(t/8) / log(60/8))
        # 8s → 0.0 | 15s → 0.31 | 30s → 0.66 | 60s → 1.0 | 300s → 1.0
        "min_threshold": 8,    # Moins de 8s = pas un signal
        "cap_seconds": 60,     # Au-delà = plafonné
        "requires_corroboration": True,
        # Le temps SEUL ne suffit pas. Il faut au moins 1 signal parmi :
        # - photos_scrolled >= 1
        # - prompt_expanded >= 1
        # - scroll_depth > 0.30
        # Sans corroboration = signal JETÉ (téléphone posé)
    },
    "photos_scrolled": {
        "min_threshold": 1,    # Au moins 1 photo après la première
        "strong_threshold": 3, # 3+ photos = signal fort
        "weight": 1.5,
    },
    "prompt_expanded": {
        "min_threshold": 1,
        "weight": 2.0,         # Lire un prompt = intérêt profond
    },
    "return_visit": {
        "weight": 3.0,         # Le signal le plus fort
        # Retour = a quitté le profil, vu d'autres profils, et est REVENU
        # Impossible à simuler consciemment
    },
    "scroll_depth": {
        "min_threshold": 0.30, # 30% du profil = engagement minimum
        "strong_threshold": 0.80,
        "weight": 1.5,
    },
    "quick_skip": {
        # Signal NÉGATIF : skip en < 2 secondes sans aucune interaction
        "max_seconds": 2,
        "weight": -1.0,       # Rejet implicite
    },
}


def sanitize_time_signal(time_seconds: float, has_corroboration: bool) -> float:
    """
    Transforme le temps brut en signal nettoyé.
    
    Protections contre les faux positifs :
    1. Cap à 60s (au-delà = bruit)
    2. Courbe logarithmique (8s→15s significatif, 120s→300s pas du tout)
    3. Corroboration obligatoire (temps seul ≠ intérêt)
    
    Le cas "téléphone posé" :
    - L'utilisateur ouvre un profil, pose son téléphone, part
    - Le mobile coupe le chrono à onPause() (app en background)
    - Si l'utilisateur revient après > 30s, c'est un NOUVEAU signal
    - Même si le chrono n'est pas coupé côté mobile (bug), 
      le cap à 60s + la corroboration éliminent le faux signal
    """
    if time_seconds < SIGNAL_CONFIG["time_on_profile"]["min_threshold"]:
        return 0.0
    
    if not has_corroboration:
        return 0.0  # Temps sans interaction = téléphone posé
    
    cap = SIGNAL_CONFIG["time_on_profile"]["cap_seconds"]
    capped = min(time_seconds, cap)
    
    # Courbe log : signal = log(t/8) / log(60/8)
    min_t = SIGNAL_CONFIG["time_on_profile"]["min_threshold"]
    signal = math.log(capped / min_t) / math.log(cap / min_t)
    
    return min(1.0, max(0.0, signal))


def has_corroboration(behavior_events: list) -> bool:
    """
    Vérifie qu'il y a au moins 1 signal d'engagement au-delà du temps.
    """
    for event in behavior_events:
        if event["type"] == "photo_scrolled":
            return True
        if event["type"] == "prompt_expanded":
            return True
        if event["type"] == "scroll_depth" and event["data"].get("depth", 0) > 0.30:
            return True
    return False


# ── Construction du profil implicite ──

async def compute_implicit_profile(
    user_id: str,
    db_session,
    redis_client,
) -> dict:
    """
    Construit le profil de préférences implicites.
    Fenêtre : 30 derniers jours.
    
    Algorithme :
    1. Charger tous les behavior_logs de l'utilisateur (30j)
    2. Grouper par profile_id vu
    3. Pour chaque profil vu, calculer un "engagement_score" composite
    4. Filtrer : garder uniquement engagement_score > seuil
    5. Pour chaque profil engagé, extraire ses features (tags, secteur, quartier)
    6. Pondérer les features par l'engagement_score
    7. Normaliser en distribution de probabilité
    8. Même chose pour les signaux négatifs (quick_skip)
    
    Retourne :
    {
        "preferred_tags": {"foodie": 0.8, "couche-tard": 0.6, ...},
        "preferred_sectors": {"creative": 0.7, "tech": 0.3, ...},
        "preferred_quartiers": {"id-be": 0.5, ...},
        "rejected_tags": {"sportif": 0.4, ...},
        "rejected_sectors": {"finance": 0.3, ...},
        "signal_count": 47,
        "confidence": 0.72,
    }
    """
    from app.models import BehaviorLog, Profile
    
    cutoff = datetime.utcnow() - timedelta(days=30)
    
    # 1. Charger les logs
    logs = await db_session.execute(
        select(BehaviorLog)
        .where(BehaviorLog.user_id == user_id)
        .where(BehaviorLog.created_at >= cutoff)
        .order_by(BehaviorLog.created_at)
    )
    logs = logs.scalars().all()
    
    if len(logs) < 5:
        # Pas assez de données → profil vide, confidence = 0
        return {
            "preferred_tags": {},
            "preferred_sectors": {},
            "preferred_quartiers": {},
            "rejected_tags": {},
            "rejected_sectors": {},
            "signal_count": len(logs),
            "confidence": 0.0,
        }
    
    # 2. Grouper par profile_id
    by_profile = defaultdict(list)
    for log in logs:
        by_profile[log.target_profile_id].append(log)
    
    # 3. Calculer engagement_score par profil
    positive_profiles = {}  # {profile_id: engagement_score}
    negative_profiles = []  # [profile_id, ...]
    
    for profile_id, events in by_profile.items():
        corroborated = has_corroboration(
            [{"type": e.event_type, "data": e.event_data} for e in events]
        )
        
        # Chercher le temps passé
        time_events = [e for e in events if e.event_type == "profile_view_duration"]
        total_time = sum(e.event_data.get("duration_ms", 0) / 1000 for e in time_events)
        time_signal = sanitize_time_signal(total_time, corroborated)
        
        # Chercher les autres signaux
        photo_count = sum(1 for e in events if e.event_type == "photo_scrolled")
        prompt_count = sum(1 for e in events if e.event_type == "prompt_expanded")
        return_count = sum(1 for e in events if e.event_type == "return_visit")
        
        # Quick skip ?
        skip_events = [e for e in events if e.event_type == "profile_action" 
                       and e.event_data.get("action") == "skip"]
        is_quick_skip = (
            len(skip_events) > 0 
            and total_time < 2.0 
            and photo_count == 0
        )
        
        if is_quick_skip:
            negative_profiles.append(profile_id)
            continue
        
        # Score d'engagement composite
        engagement = (
            time_signal * 2.0
            + min(photo_count, 5) * 0.3
            + prompt_count * 0.5
            + return_count * 1.0  # Le plus fort
        )
        
        if engagement > 0.5:  # Seuil minimum d'intérêt
            positive_profiles[profile_id] = engagement
    
    # 4-6. Extraire les features et pondérer
    preferred_tags = defaultdict(float)
    preferred_sectors = defaultdict(float)
    preferred_quartiers = defaultdict(float)
    rejected_tags = defaultdict(float)
    rejected_sectors = defaultdict(float)
    
    # Profiles positifs
    for pid, score in positive_profiles.items():
        profile = await db_session.get(Profile, pid)
        if not profile:
            continue
        for tag in (profile.tags or []):
            preferred_tags[tag] += score
        if profile.sector:
            preferred_sectors[profile.sector] += score
        for uq in profile.user_quartiers:
            preferred_quartiers[str(uq.quartier_id)] += score * 0.5
    
    # Profiles négatifs
    for pid in negative_profiles:
        profile = await db_session.get(Profile, pid)
        if not profile:
            continue
        for tag in (profile.tags or []):
            rejected_tags[tag] += 1.0
        if profile.sector:
            rejected_sectors[profile.sector] += 1.0
    
    # 7. Normaliser
    def normalize(d):
        if not d:
            return {}
        max_val = max(d.values())
        if max_val == 0:
            return {}
        return {k: round(v / max_val, 2) for k, v in d.items()}
    
    # 8. Confiance basée sur le nombre de signaux
    total_signals = len(positive_profiles) + len(negative_profiles)
    confidence = min(1.0, total_signals / 50)  # 50+ signaux = confiance max
    
    return {
        "preferred_tags": normalize(dict(preferred_tags)),
        "preferred_sectors": normalize(dict(preferred_sectors)),
        "preferred_quartiers": normalize(dict(preferred_quartiers)),
        "rejected_tags": normalize(dict(rejected_tags)),
        "rejected_sectors": normalize(dict(rejected_sectors)),
        "signal_count": total_signals,
        "confidence": round(confidence, 2),
    }


# ── Intégration dans L3 (lifestyle_scorer.py) ──

async def apply_implicit_adjustment(
    base_score: float,
    candidate_profile,
    implicit_profile: dict,
) -> float:
    """
    Ajuste le score L3 d'un candidat en fonction du profil implicite.
    
    Le bonus/malus est plafonné à ±15% du score L3.
    Multiplié par la confiance (0→1) pour éviter le bruit quand
    on n'a pas assez de données.
    
    Exemple concret :
    - L'utilisateur passe du temps sur les profils "foodie + couche-tard"
    - Un candidat a les tags "foodie + couche-tard" → bonus +12%
    - Un autre candidat a les tags "sportif + lève-tôt" (rejeté implicitement) → malus -8%
    """
    if implicit_profile["confidence"] < 0.3:
        return base_score  # Pas assez de données, on ne touche à rien
    
    candidate_tags = set(candidate_profile.tags or [])
    candidate_sector = candidate_profile.sector or ""
    
    # Calcul du bonus (similarité avec les prefs positives)
    bonus = 0.0
    pref_tags = implicit_profile["preferred_tags"]
    for tag in candidate_tags:
        bonus += pref_tags.get(tag, 0)
    if candidate_sector:
        bonus += implicit_profile["preferred_sectors"].get(candidate_sector, 0)
    
    # Calcul du malus (similarité avec les rejets)
    malus = 0.0
    rej_tags = implicit_profile["rejected_tags"]
    for tag in candidate_tags:
        malus += rej_tags.get(tag, 0)
    if candidate_sector:
        malus += implicit_profile["rejected_sectors"].get(candidate_sector, 0)
    
    # Normaliser
    if candidate_tags:
        bonus /= len(candidate_tags)
        malus /= len(candidate_tags)
    
    # Ajustement plafonné à ±15%, pondéré par la confiance
    adjustment = (bonus - malus) * implicit_profile["confidence"] * 0.15
    adjustment = max(-0.15, min(0.15, adjustment))
    
    return max(0.0, min(1.0, base_score + adjustment))


# ── Cache Redis ──

# Le profil implicite est recalculé 1x par batch nocturne (pas à chaque requête)
# Stocké dans Redis : implicit_prefs:{user_id}
# TTL : 25h (se renouvelle chaque nuit)
# Fallback : si absent, compute_implicit_profile() est appelé à la volée
#            mais avec un TTL court de 6h pour éviter la surcharge
```

---

## Mobile — Sanitisation du BehaviorTracker (CRITIQUE)

```kotlin
/**
 * MISE À JOUR du BehaviorTracker (section 23 mobile spec).
 * 
 * Protection contre les faux signaux :
 * 1. Le chrono s'arrête à onPause() (app en background)
 * 2. Si l'utilisateur revient après > 30s, c'est un NOUVEAU signal
 * 3. Le temps est envoyé avec un flag "corroborated" : 
 *    true si l'utilisateur a touché l'écran (scroll, tap, photo)
 *    false si le temps a juste passé sans interaction
 * 4. Le serveur JETTE les signaux temps-seul sans corroboration
 */

@Singleton
class BehaviorTracker @Inject constructor() {

    private var currentProfileId: String? = null
    private var profileViewStart: Long = 0
    private var hasInteracted: Boolean = false  // NOUVEAU : corroboration
    private var lastPauseTime: Long = 0

    // Appelé quand l'utilisateur voit un nouveau profil
    fun onProfileViewed(profileId: String) {
        finalizeCurrentProfile()
        currentProfileId = profileId
        profileViewStart = SystemClock.elapsedRealtime()
        hasInteracted = false  // Reset la corroboration
    }

    // Appelé quand l'utilisateur scrolle une photo
    fun onPhotoScrolled(profileId: String, photoIndex: Int) {
        hasInteracted = true  // Corroboration !
        // ... (existant)
    }

    // Appelé quand l'utilisateur tap "voir plus" sur un prompt
    fun onPromptExpanded(profileId: String, promptId: String) {
        hasInteracted = true  // Corroboration !
        // ... (existant)
    }

    // Appelé quand l'utilisateur scrolle le profil
    fun onScrollDepthChanged(profileId: String, depth: Float) {
        if (depth > 0.30f) {
            hasInteracted = true  // Corroboration !
        }
        // ... log le scroll_depth
    }

    // ── Lifecycle hooks (NOUVEAU) ──

    // Appelé par le ViewModel quand l'Activity passe en onPause
    fun onAppPaused() {
        lastPauseTime = SystemClock.elapsedRealtime()
        finalizeCurrentProfile()  // Coupe le chrono immédiatement
    }

    // Appelé par le ViewModel quand l'Activity reprend (onResume)
    fun onAppResumed() {
        val pauseDuration = SystemClock.elapsedRealtime() - lastPauseTime
        if (pauseDuration > 30_000) {
            // Plus de 30s d'absence → c'est un nouveau contexte
            // Ne pas reprendre l'ancien chrono
            currentProfileId = null
        }
        // Si < 30s → on reprend normalement (l'utilisateur a juste
        // répondu à une notification ou vérifié l'heure)
    }

    private fun finalizeCurrentProfile() {
        currentProfileId?.let { id ->
            val duration = SystemClock.elapsedRealtime() - profileViewStart
            val durationSeconds = duration / 1000f

            // Cap à 60 secondes côté client aussi
            val cappedDuration = minOf(durationSeconds, 60f)

            events.add(BehaviorEvent(
                type = "profile_view_duration",
                profileId = id,
                data = mapOf(
                    "duration_ms" to duration,
                    "duration_capped_s" to cappedDuration,
                    "corroborated" to hasInteracted,  // CRITIQUE
                ),
                timestamp = System.currentTimeMillis(),
            ))
        }
        currentProfileId = null
        hasInteracted = false
    }
}
```


---

# MISE À JOUR 7 — Stratégie d'acquisition féminine + système d'invitations

---

## Contexte

La section 31 (cold start) et la section 32 (gender balance) posent les principes.
Cette mise à jour ajoute les mécanismes CONCRETS pour attirer et retenir les
femmes au lancement : waitlist genrée, système d'invitations, programme
ambassadrices, et first-impression feed.

---

## 1. Waitlist genrée — les femmes passent devant

```python
# ── Modification du modèle WaitlistEntry ──

class WaitlistEntry(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "waitlist_entries"

    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), nullable=False
    )
    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), unique=True, nullable=False
    )
    phone_number: Mapped[str] = mapped_column(String(20), nullable=False)
    gender: Mapped[str] = mapped_column(String(10), nullable=False)
    # "male" | "female" | "other"

    position: Mapped[int] = mapped_column(Integer, nullable=False)
    # Position dans la file. Les femmes reçoivent position=0 (accès immédiat).

    status: Mapped[str] = mapped_column(String(20), default="waiting")
    # "waiting" | "invited" | "activated" | "expired"

    invited_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    activated_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    invite_code_used: Mapped[str | None] = mapped_column(String(12), nullable=True)
    # Si l'utilisateur est arrivé via un code d'invitation

    # ── Index ──
    __table_args__ = (
        Index("ix_waitlist_city_status", "city_id", "status"),
        Index("ix_waitlist_city_gender", "city_id", "gender"),
        Index("ix_waitlist_city_position", "city_id", "position"),
    )


# ── Logique de la waitlist genrée ──

WAITLIST_CONFIG = {
    "female_skip_waitlist": True,
    # Les femmes inscrites sautent la waitlist et accèdent directement.
    # Elles n'attendent jamais. Leur status passe directement à "activated".

    "male_batch_release_size": 50,
    # On libère les hommes par batch de 50, contrôlé par le ratio.

    "male_release_condition": {
        "min_female_ratio": 0.40,
        # On ne libère un batch d'hommes que si le ratio femmes est > 40%.
        # Si le ratio tombe en dessous, on arrête de libérer.
        # Ça garantit que les femmes ne sont jamais submergées.

        "min_females_absolute": 30,
        # Minimum absolu de femmes actives avant de libérer le premier batch.
    },

    "waitlist_display": {
        # Ce que les hommes voient en attendant :
        "show_position": True,
        "show_total_waiting": True,          # "327 personnes attendent"
        "show_estimated_time": True,          # "Environ 3-5 jours"
        "show_invite_shortcut": True,         # "Invite 3 amis et passe devant"
        # Ce que les FEMMES voient :
        # Rien. Elles ne passent jamais par la waitlist.
    },

    "priority_boost_for_inviters": {
        "invites_needed": 3,
        "boost": "top_10_percent",
        # Un homme qui invite 3 personnes (vérifiées) passe dans le top 10%
        # de la waitlist. Ça crée de la viralité masculine.
    },
}


async def process_waitlist_join(user_id: str, city_id: str, gender: str, db_session):
    """
    Appelé après onboarding complet. Place l'utilisateur dans la waitlist
    OU l'active immédiatement selon le genre.
    """
    if gender == "female" and WAITLIST_CONFIG["female_skip_waitlist"]:
        # Les femmes passent directement
        entry = WaitlistEntry(
            city_id=city_id,
            user_id=user_id,
            phone_number=user.phone_number,
            gender=gender,
            position=0,
            status="activated",
            activated_at=datetime.utcnow(),
        )
        db_session.add(entry)
        # Activer immédiatement dans le matching pool
        await activate_user_in_city(user_id, city_id, db_session)
        return {"status": "activated", "message": "Bienvenue sur Flaam !"}

    else:
        # Hommes et autres → waitlist avec position
        last_position = await db_session.scalar(
            select(func.max(WaitlistEntry.position))
            .where(WaitlistEntry.city_id == city_id)
        ) or 0

        entry = WaitlistEntry(
            city_id=city_id,
            user_id=user_id,
            phone_number=user.phone_number,
            gender=gender,
            position=last_position + 1,
            status="waiting",
        )
        db_session.add(entry)

        total_waiting = await db_session.scalar(
            select(func.count(WaitlistEntry.id))
            .where(WaitlistEntry.city_id == city_id)
            .where(WaitlistEntry.status == "waiting")
        )

        return {
            "status": "waiting",
            "position": last_position + 1,
            "total_waiting": total_waiting,
            "message": f"Tu es #{last_position + 1} sur la liste d'attente.",
            "tip": "Invite 3 amis pour passer devant !",
        }
```

---

## 2. Système d'invitations — chaque femme amène ses copines

```python
# ── Modèle InviteCode ──

class InviteCode(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "invite_codes"

    code: Mapped[str] = mapped_column(
        String(12), unique=True, nullable=False, index=True
    )
    # Format : "FLAAM-XXXX" (8 chars alphanumériques après le préfixe)
    # Généré : secrets.token_urlsafe(6)[:8].upper() → "FLAAM-A3K9M2P1"

    creator_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), nullable=False
    )
    city_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("cities.id"), nullable=False
    )

    type: Mapped[str] = mapped_column(String(20), default="standard")
    # "standard" = invitation normale (3 par femme)
    # "ambassador" = invitation ambassadrice (illimité)
    # "event" = invitation liée à un event Flaam

    used_by_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), nullable=True
    )
    used_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    expires_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False
    )
    # Standard : 30 jours. Ambassador : 90 jours.

    is_active: Mapped[bool] = mapped_column(Boolean, default=True)


# ── Constantes ──

INVITE_CONFIG = {
    "codes_per_female_user": 3,
    # Chaque femme inscrite reçoit 3 codes à partager.
    # Ses amies utilisent le code → accès immédiat (même si homme).

    "codes_per_ambassador": 50,
    # Les ambassadrices ont 50 codes (renouvelables).

    "invite_code_benefits_for_invitee": {
        "skip_waitlist": True,
        # Toute personne invitée (même homme) saute la waitlist.
        # Ça donne une RAISON aux hommes de demander un code à une femme.
        # "Demande un code à une amie sur Flaam" → viralité cross-genre.
    },

    "invite_code_benefits_for_inviter": {
        "per_invite_used": {
            "bonus_likes": 2,  # +2 likes offerts par invitation utilisée
        },
        "at_3_invites_used": {
            "premium_days": 7,  # 1 semaine premium gratuit
        },
        "at_5_invites_used": {
            "premium_days": 30,  # 1 mois premium gratuit
        },
    },
}


# ── Endpoints ──

# POST /invites/generate
# → Génère les codes d'invitation de l'utilisateur
# → Retourne la liste des codes (actifs + utilisés)
# → Erreur si déjà au max (sauf ambassadrices)

# GET /invites/me
# → Liste mes codes : code, status (active/used/expired), used_by (prénom)

# POST /invites/validate
# → Body : { "code": "FLAAM-A3K9M2P1" }
# → Vérifie : existe, actif, pas expiré, pas déjà utilisé
# → Retourne : { "valid": true, "city": "Lomé", "creator_name": "Ama" }

# POST /invites/redeem
# → Appelé pendant l'onboarding si l'utilisateur a un code
# → Marque le code comme utilisé, skip la waitlist, crédite l'inviter


async def generate_invite_codes(user_id: str, db_session) -> list[dict]:
    """
    Génère les codes d'invitation pour un utilisateur.
    Appelé automatiquement après activation du compte.
    """
    user = await db_session.get(User, user_id)

    if user.gender == "female":
        max_codes = INVITE_CONFIG["codes_per_female_user"]
    elif user.is_ambassador:
        max_codes = INVITE_CONFIG["codes_per_ambassador"]
    else:
        max_codes = 0  # Les hommes non-ambassadeurs n'ont pas de codes
        # (sauf s'ils deviennent premium — à évaluer en V2)

    existing = await db_session.scalar(
        select(func.count(InviteCode.id))
        .where(InviteCode.creator_id == user_id)
        .where(InviteCode.is_active == True)
    )

    to_generate = max_codes - existing
    if to_generate <= 0:
        return []

    codes = []
    for _ in range(to_generate):
        code = f"FLAAM-{secrets.token_urlsafe(6)[:8].upper()}"
        invite = InviteCode(
            code=code,
            creator_id=user_id,
            city_id=user.city_id,
            type="ambassador" if user.is_ambassador else "standard",
            expires_at=datetime.utcnow() + timedelta(
                days=90 if user.is_ambassador else 30
            ),
        )
        db_session.add(invite)
        codes.append(code)

    return codes
```

---

## 3. Programme ambassadrices

```python
# ── Champ ajouté au modèle User ──
# is_ambassador: Mapped[bool] = mapped_column(Boolean, default=False)
# ambassador_since: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)

# ── Champ ajouté au modèle Subscription ──
# is_lifetime: Mapped[bool] = mapped_column(Boolean, default=False)
# granted_reason: Mapped[str | None] = mapped_column(String(50), nullable=True)
# "ambassador" | "contest_winner" | "admin_grant" | etc.


AMBASSADOR_CONFIG = {
    "description": """
    Les ambassadrices sont 10-15 femmes influentes dans la scène locale.
    Pas des influenceuses Instagram — des femmes que leur cercle respecte.
    Coiffeuses connues, DRH, organisatrices d'events, blogueuses food.

    Elles reçoivent :
    - Early access (avant le lancement public)
    - Premium à vie (is_lifetime=True)
    - 50 codes d'invitation (renouvelables)
    - Badge spécial sur leur profil (visible mais discret)
    - Accès au salon ambassadrices (groupe WhatsApp privé avec l'équipe)

    En échange :
    - Elles utilisent l'app normalement (pas de contenu sponsorisé)
    - Elles en parlent à leur cercle (organique, pas forcé)
    - Elles remontent les bugs et les retours terrain
    - Elles participent aux events Flaam

    IMPORTANT : l'ambassadrice n'est PAS une influenceuse payée.
    C'est une early adopter avec un réseau. La différence est fondamentale.
    """,

    "max_per_city": 20,
    "invite_codes": 50,
    "premium_type": "lifetime",
    "profile_badge": "ambassador",  # Petit badge 🔥 discret
    "badge_visible_to": "matched_only",
    # Le badge n'est visible que par les gens avec qui elle a matché.
    # Pas visible dans le feed (pour ne pas biaiser les likes).
}


# ── Endpoints admin ──

# POST /admin/ambassadors
# Body : { "user_id": "...", "city_id": "..." }
# → Active le statut ambassadrice
# → Crée le premium lifetime
# → Génère les 50 codes d'invitation
# → Retourne le profil + les codes

# DELETE /admin/ambassadors/{user_id}
# → Retire le statut (mais le premium lifetime reste — c'est une promesse)

# GET /admin/ambassadors?city_id={id}
# → Liste des ambassadrices par ville avec stats :
#   codes utilisés, invités actifs, dernière connexion
```

---

## 4. First-impression feed — les nouvelles femmes voient le meilleur

```python
# app/services/matching_engine/first_impression.py
"""
Quand une femme s'inscrit et ouvre son premier feed, elle doit voir
les MEILLEURS profils masculins. Pas les plus populaires — les plus
sains (bon behavior score, profil complet, conversations de qualité).

Si sa première impression c'est 12 profils avec 1 photo et "Salut ça va",
elle désinstalle. Game over.

Ce module intervient APRÈS le pipeline normal (L1→L5) pour re-trier
les résultats des 3 premiers feeds d'une nouvelle utilisatrice.
"""

FIRST_IMPRESSION_CONFIG = {
    "active_for_first_n_feeds": 3,
    # Les 3 premiers jours de feed sont curatés.
    # Ensuite l'algo normal reprend.

    "applies_to": "new_female_users",
    # Asymétrique par design (cf. section 32 gender balance).
    # Les nouveaux hommes reçoivent le feed normal.

    "sorting_override": {
        # Pendant les 3 premiers feeds, le tri final n'est plus
        # par score combiné (L2+L3+L4) mais par QUALITÉ du profil.
        "primary_sort": "behavior_multiplier",    # Les plus sains en premier
        "secondary_sort": "profile_completeness",  # Les plus complets ensuite
        "tertiary_sort": "combined_score",          # L'algo normal en dernier
    },

    "hard_minimum_for_candidates": {
        "profile_completeness": 0.75,  # 3+ photos, 1+ prompt, spots tagués
        "behavior_multiplier": 1.0,     # Au-dessus de la moyenne
        "photos_count": 3,              # Minimum 3 photos
    },

    "rationale": """
    C'est injuste pour les hommes avec un profil incomplet — ils ne seront
    pas montrés aux nouvelles femmes pendant 3 jours. Mais c'est LE mécanisme
    qui retient les femmes. Un homme avec un profil incomplet ne perd rien :
    il apparaît quand même dans les feeds des femmes établies et dans les
    feeds des autres hommes (si bi/queer). Il gagne juste à compléter son profil.
    """,
}


async def apply_first_impression(
    user,
    feed_ids: list[str],
    db_session,
) -> list[str]:
    """
    Re-trie le feed pour les nouvelles utilisatrices.
    Appelé dans pipeline.py APRÈS L5 (corrections).
    """
    # Vérifier si applicable
    if user.gender != "female":
        return feed_ids

    feeds_seen = await count_feeds_generated(user.id, db_session)
    if feeds_seen >= FIRST_IMPRESSION_CONFIG["active_for_first_n_feeds"]:
        return feed_ids  # Plus applicable, retour à la normale

    # Charger les données de qualité des candidats
    candidates_quality = []
    for cid in feed_ids:
        profile = await get_profile_with_behavior(cid, db_session)
        if not profile:
            continue

        # Appliquer les minimums stricts
        mins = FIRST_IMPRESSION_CONFIG["hard_minimum_for_candidates"]
        if (
            profile.completeness >= mins["profile_completeness"]
            and profile.behavior_multiplier >= mins["behavior_multiplier"]
            and len(profile.photos) >= mins["photos_count"]
        ):
            candidates_quality.append((cid, profile))

    # Si on n'a pas assez de profils qualité, compléter avec le feed normal
    if len(candidates_quality) < 8:
        remaining = [cid for cid in feed_ids
                     if cid not in [c[0] for c in candidates_quality]]
        candidates_quality.extend(
            [(cid, await get_profile_with_behavior(cid, db_session))
             for cid in remaining[:12 - len(candidates_quality)]]
        )

    # Trier par qualité
    candidates_quality.sort(
        key=lambda x: (
            x[1].behavior_multiplier if x[1] else 0,
            x[1].completeness if x[1] else 0,
        ),
        reverse=True,
    )

    return [cid for cid, _ in candidates_quality[:12]]
```

---

## 5. Celery task — libération automatique de la waitlist

```python
# app/tasks/waitlist_tasks.py

@celery_app.task(name="release_waitlist_batch")
async def release_waitlist_batch():
    """
    Tourne toutes les 6 heures.
    Vérifie le ratio H/F dans chaque ville en phase LAUNCH ou GROWTH.
    Si le ratio femmes est > 40%, libère un batch d'hommes de la waitlist.

    Flow :
    1. Pour chaque ville active
    2. Calculer le ratio actuel
    3. Si ratio femmes > seuil → libérer N hommes (par ordre de position)
    4. Envoyer une notification push à chaque homme libéré
    5. Logger les métriques pour le dashboard admin
    """
    async with get_db_session() as db:
        cities = await db.execute(
            select(CityLaunchStatus)
            .where(CityLaunchStatus.phase.in_(["launch", "growth"]))
        )

        for city_status in cities.scalars().all():
            # Compter les actifs par genre
            stats = await db.execute(
                select(
                    User.gender,
                    func.count(User.id),
                )
                .where(User.city_id == city_status.city_id)
                .where(User.is_active == True)
                .where(User.is_visible == True)
                .group_by(User.gender)
            )
            gender_counts = dict(stats.fetchall())
            total = sum(gender_counts.values())
            female_count = gender_counts.get("female", 0)

            if total == 0:
                continue

            female_ratio = female_count / total

            # Vérifier les conditions de libération
            config = WAITLIST_CONFIG["male_release_condition"]
            if (
                female_ratio >= config["min_female_ratio"]
                and female_count >= config["min_females_absolute"]
            ):
                batch_size = WAITLIST_CONFIG["male_batch_release_size"]

                # Sélectionner les hommes en attente par position
                waiting_males = await db.execute(
                    select(WaitlistEntry)
                    .where(WaitlistEntry.city_id == city_status.city_id)
                    .where(WaitlistEntry.status == "waiting")
                    .where(WaitlistEntry.gender == "male")
                    .order_by(WaitlistEntry.position)
                    .limit(batch_size)
                )

                released = 0
                for entry in waiting_males.scalars().all():
                    entry.status = "invited"
                    entry.invited_at = datetime.utcnow()
                    await activate_user_in_city(
                        entry.user_id, city_status.city_id, db
                    )
                    # Notification push
                    await send_notification(
                        user_id=entry.user_id,
                        type="waitlist_activated",
                        title="C'est ton tour !",
                        body="Flaam est maintenant actif pour toi. Découvre qui fréquente les mêmes endroits.",
                    )
                    released += 1

                # Logger
                logger.info(
                    "waitlist_batch_released",
                    city_id=str(city_status.city_id),
                    female_ratio=round(female_ratio, 2),
                    released=released,
                    remaining_waiting=await db.scalar(
                        select(func.count(WaitlistEntry.id))
                        .where(WaitlistEntry.city_id == city_status.city_id)
                        .where(WaitlistEntry.status == "waiting")
                    ),
                )
            else:
                logger.info(
                    "waitlist_batch_skipped",
                    city_id=str(city_status.city_id),
                    female_ratio=round(female_ratio, 2),
                    reason="ratio_too_low" if female_ratio < config["min_female_ratio"]
                           else "not_enough_females",
                )


# ── Celery beat schedule ──
# Ajouter à la section 7 (celery tasks) :
# "release-waitlist-batch": {
#     "task": "release_waitlist_batch",
#     "schedule": crontab(minute=0, hour="*/6"),  # Toutes les 6h
# },
```

---

## 6. Nouveaux endpoints (ajoutés au catalogue)

```
| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 100 | POST | /invites/generate | Générer mes codes d'invitation | Oui |
| 101 | GET | /invites/me | Liste de mes codes (actifs + utilisés) | Oui |
| 102 | POST | /invites/validate | Vérifier un code (onboarding) | Non |
| 103 | POST | /invites/redeem | Utiliser un code (onboarding) | Oui |
| 104 | POST | /admin/ambassadors | Nommer une ambassadrice | Admin |
| 105 | DELETE | /admin/ambassadors/{user_id} | Retirer le statut | Admin |
| 106 | GET | /admin/ambassadors | Liste + stats par ville | Admin |
| 107 | GET | /admin/waitlist/stats | Stats waitlist par ville/genre | Admin |
| 108 | POST | /admin/waitlist/release | Forcer une libération manuelle | Admin |

Total endpoints : 99 + 9 = 108
```


---

# MISE À JOUR 8 — Event-driven onboarding (ghost users) + page web pré-inscription

---

## Contexte

Flaam a 3 portes d'entrée. Les deux premières (classique + invitation) sont
déjà dans la spec. Cette mise à jour ajoute la troisième : l'inscription
via un event Flaam, avec une page web légère qui crée un "ghost user"
AVANT que la personne n'ait l'app.

IMPORTANT : cette porte est un COMPLÉMENT, pas un remplacement.
Les portes 1 (classique) et 2 (invitation) restent inchangées.
L'event-driven onboarding est un canal d'acquisition supplémentaire.

---

## 1. Ghost user — nouveau statut utilisateur

```python
# ── Modification du modèle User ──
# Ajout d'un statut pour les ghost users

class OnboardingStatus(str, Enum):
    """
    5 statuts possibles. Les 3 premiers existent déjà.
    Les 2 nouveaux (ghost, pre_registered) ne concernent que la porte 3.
    
    Porte 1 (classique)   : not_started → in_progress → completed
    Porte 2 (invitation)  : not_started → in_progress → completed
    Porte 3 (event)       : ghost → pre_registered → in_progress → completed
    """
    GHOST = "ghost"
    # Créé via la page web d'un event. A un numéro vérifié et un prénom.
    # Pas de profil, pas de photos, pas de selfie.
    # N'apparaît dans AUCUN feed. Invisible dans le matching.
    # N'a PAS de JWT (pas d'app). Identifié par son numéro de téléphone.

    PRE_REGISTERED = "pre_registered"
    # A fait le check-in à un event (QR scanné).
    # Toujours ghost côté app — mais on sait qu'il était physiquement là.
    # Recevra le message WhatsApp "télécharge Flaam pour voir qui y était".

    NOT_STARTED = "not_started"
    # Statut par défaut quand l'app est installée et OTP vérifié.
    # C'est là que les portes 1, 2 et 3 convergent.
    # Pour la porte 3 : on détecte le ghost et on pré-remplit.

    IN_PROGRESS = "in_progress"
    # L'onboarding a commencé mais n'est pas terminé.

    COMPLETED = "completed"
    # Profil complet, prêt pour le matching.


# Ajouts au modèle User :
# onboarding_status: Mapped[str] = mapped_column(String(20), default="not_started")
# source_channel: Mapped[str | None] = mapped_column(String(20), nullable=True)
# # "organic" | "invite" | "event" — pour analytics
# source_event_id: Mapped[uuid.UUID | None] = mapped_column(
#     UUID(as_uuid=True), ForeignKey("events.id"), nullable=True
# )
# # L'event qui a amené cet utilisateur (porte 3 only)
```

---

## 2. Page web événement — endpoint public de pré-inscription

```python
# ── IMPORTANT ──
# Cette page web est DISTINCTE de l'app mobile.
# C'est une page statique ou SSR servie par le backend (FastAPI + Jinja2)
# ou un mini-site séparé (Next.js, etc.).
# L'utilisateur n'a PAS besoin de télécharger l'app à ce stade.

# ── Endpoints ──

# GET /web/events/{event_slug}
# → Page web publique de l'event (SSR ou statique)
# → Affiche : nom, date, lieu, description, nombre d'inscrits
# → Formulaire de pré-inscription : numéro + prénom
# → PAS de JWT en retour. Juste un message de confirmation.

# POST /auth/event-preregister
# → Body : { "phone": "+22890123456", "event_id": "uuid", "first_name": "Ama" }
# → Envoie un OTP WhatsApp (même flow que l'auth classique)
# → Si OTP valide : crée un ghost user OU retrouve un user existant
# → Associe l'utilisateur à l'event (EventRegistration)
# → Retourne : { "status": "registered", "qr_code_url": "...", "event_name": "..." }
# → Le QR code encode : event_id + user_id (signé HMAC, non falsifiable)


async def event_preregister(
    phone: str,
    event_id: str,
    first_name: str,
    db_session,
) -> dict:
    """
    Pré-inscription à un event via la page web.
    
    3 cas possibles :
    
    1. Nouveau numéro → crée un ghost user + EventRegistration
    2. Ghost user existant (déjà pré-inscrit à un autre event) → ajoute EventRegistration
    3. Utilisateur complet existant (a déjà l'app) → ajoute EventRegistration
       (pas de modification du profil, juste l'inscription à l'event)
    """
    # Vérifier que l'event existe et est ouvert
    event = await db_session.get(Event, event_id)
    if not event or event.status != "published":
        raise EventNotFoundError()
    if event.current_attendees >= event.max_attendees:
        raise EventFullError()

    # Chercher l'utilisateur par numéro
    normalized_phone = normalize_phone(phone)
    user = await get_user_by_phone(normalized_phone, db_session)

    if user is None:
        # Cas 1 : nouveau → créer ghost user
        user = User(
            phone_number=normalized_phone,
            phone_hash=hash_phone(normalized_phone),
            first_name=first_name,
            onboarding_status=OnboardingStatus.GHOST,
            source_channel="event",
            source_event_id=event_id,
            is_active=False,   # Ghost = pas actif dans le matching
            is_visible=False,  # Ghost = invisible
        )
        db_session.add(user)
        await db_session.flush()  # Pour avoir user.id

    elif user.onboarding_status == OnboardingStatus.COMPLETED:
        # Cas 3 : utilisateur complet → juste l'inscrire à l'event
        pass  # Ne touche pas au profil

    # Créer l'inscription à l'event
    registration = EventRegistration(
        event_id=event_id,
        user_id=user.id,
        status="registered",  # Pas encore "checked_in"
        registered_via="web",  # vs "app" pour les inscriptions in-app
    )
    db_session.add(registration)

    # Générer le QR code
    qr_payload = generate_signed_qr(event_id, user.id)

    return {
        "status": "registered",
        "qr_code_url": f"https://api.flaam.app/qr/{qr_payload}",
        "event_name": event.title,
        "event_date": event.starts_at.isoformat(),
        "message": f"Tu es inscrit(e) à {event.title} ! "
                   f"Présente ce QR code à l'entrée.",
    }
```

---

## 3. Check-in à l'event

```python
# POST /events/{event_id}/checkin
# → Body : { "qr_payload": "signed_string" }
# → Vérifie la signature HMAC du QR
# → Passe l'EventRegistration de "registered" à "checked_in"
# → Si ghost user : passe onboarding_status à "pre_registered"
# → Ajoute automatiquement le spot de l'event aux spots de l'utilisateur
# → Retourne : { "status": "checked_in", "attendees_count": 24 }

async def event_checkin(event_id: str, qr_payload: str, db_session) -> dict:
    # Vérifier et décoder le QR
    payload = verify_signed_qr(qr_payload)
    if payload["event_id"] != event_id:
        raise InvalidQRError()

    user_id = payload["user_id"]
    user = await db_session.get(User, user_id)

    # Mettre à jour l'inscription
    registration = await get_event_registration(event_id, user_id, db_session)
    if not registration:
        raise NotRegisteredError()
    registration.status = "checked_in"
    registration.checked_in_at = datetime.utcnow()

    # Si ghost → pre_registered
    if user.onboarding_status == OnboardingStatus.GHOST:
        user.onboarding_status = OnboardingStatus.PRE_REGISTERED

    # Ajouter le spot de l'event automatiquement
    event = await db_session.get(Event, event_id)
    if event.spot_id:
        await add_user_spot(user_id, event.spot_id, source="event_checkin", db_session=db_session)

    attendees = await count_checked_in(event_id, db_session)

    return {
        "status": "checked_in",
        "attendees_count": attendees,
    }
```

---

## 4. Détection du ghost user dans l'app

```python
# ── Modification de l'endpoint POST /auth/otp/verify ──
# Quand l'OTP est vérifié et qu'on retrouve un ghost user,
# on retourne des informations supplémentaires pour l'app.

# Dans la réponse OTP existante, ajouter :

OTP_VERIFY_RESPONSE_WITH_GHOST = {
    "access_token": "jwt...",
    "refresh_token": "jwt...",
    "user": {
        "id": "uuid",
        "onboarding_status": "not_started",  # Ghost → promu à not_started
        "source_channel": "event",
        # ↑ L'app sait que c'est un event user

        # Données pré-remplies (porte 3 seulement)
        "prefilled": {
            "first_name": "Ama",
            "source_event": {
                "id": "uuid",
                "title": "Afterwork Café 21",
                "spot_id": "uuid",
                "spot_name": "Café 21",
                "date": "2026-04-11",
                "attendees_count": 24,
            },
        },
    },
}

# Pour les portes 1 et 2, "prefilled" est null ou absent.
# L'app utilise "prefilled" pour :
# 1. Skip l'écran prénom (déjà rempli)
# 2. Pré-cocher le spot de l'event dans l'écran spots
# 3. Afficher le message contextuel : "Complete ton profil pour
#    voir qui était au Café 21 vendredi."
```

---

## 5. Event attendee boost dans le matching

```python
# app/services/matching_engine/event_boost.py
"""
Quand deux personnes étaient au même event et ont complété leur profil,
elles reçoivent un boost temporaire dans le feed de l'autre.

Ce boost fonctionne comme le new user boost (L5) mais est lié à l'event.

Durée : 7 jours après l'event.
Multiplicateur : 1.5x sur le score final.
Conditions :
  - Les deux personnes étaient checked_in au même event
  - Les deux ont un profil complet (onboarding_status = completed)
  - L'event date de moins de 7 jours

Le badge "Était au [event_name]" est visible sur le profil dans le feed.
"""

EVENT_BOOST_CONFIG = {
    "enabled": True,
    "duration_days": 7,
    "score_multiplier": 1.5,
    "badge_visible": True,
    "badge_text_template": "Était au {event_name}",

    # L'event boost ne REMPLACE PAS le matching normal.
    # Le pipeline L1→L5 tourne normalement.
    # L'event boost est un multiplicateur POST-pipeline,
    # appliqué dans corrections.py après le shuffle.
    # Ça veut dire que si L1 exclut quelqu'un (genre incompatible,
    # bloqué, etc.), l'event boost ne le fera PAS réapparaître.
}


async def get_event_boost_candidates(
    user_id: str,
    feed_candidate_ids: list[str],
    db_session,
) -> dict[str, float]:
    """
    Retourne {candidate_id: boost_multiplier} pour les candidats
    qui étaient au même event récent que l'utilisateur.
    
    Appelé dans pipeline.py après L5.
    """
    cutoff = datetime.utcnow() - timedelta(days=EVENT_BOOST_CONFIG["duration_days"])

    # Trouver les events récents de l'utilisateur
    my_events = await db_session.execute(
        select(EventRegistration.event_id)
        .where(EventRegistration.user_id == user_id)
        .where(EventRegistration.status == "checked_in")
        .join(Event, Event.id == EventRegistration.event_id)
        .where(Event.starts_at >= cutoff)
    )
    my_event_ids = {row[0] for row in my_events.fetchall()}

    if not my_event_ids:
        return {}

    # Trouver les candidats qui étaient aux mêmes events
    co_attendees = await db_session.execute(
        select(EventRegistration.user_id, EventRegistration.event_id)
        .where(EventRegistration.event_id.in_(my_event_ids))
        .where(EventRegistration.status == "checked_in")
        .where(EventRegistration.user_id.in_(feed_candidate_ids))
        .where(EventRegistration.user_id != user_id)
    )

    boosts = {}
    for candidate_id, event_id in co_attendees.fetchall():
        # Un candidat peut être à plusieurs events communs → on prend le max
        boosts[str(candidate_id)] = EVENT_BOOST_CONFIG["score_multiplier"]

    return boosts
```

---

## 6. Ice-breaker contextuel pour les event matches

```python
# Ajout dans la table ice_breakers (section 14 de la spec)

EVENT_ICE_BREAKERS = [
    {
        "trigger": "same_event",
        "level": 1,  # Remplace l'ice-breaker par défaut
        "templates": [
            "Vous étiez tous les deux à {event_name}. C'était comment pour toi ?",
            "Tiens, tu étais aussi au {event_name} ! T'as aimé ?",
            "{event_name} — qu'est-ce qui t'a le plus marqué ?",
            "On était au même endroit {days_ago} ! Comment c'était de ton côté ?",
        ],
        "context_required": ["event_name", "days_ago"],
    },
]
# Si les deux matchés étaient au même event, l'ice-breaker contextuel
# est utilisé à la place de l'ice-breaker générique.
# Ça donne un sujet de conversation immédiat et naturel.
```

---

## 7. WhatsApp post-event (Celery task)

```python
# app/tasks/event_tasks.py

@celery_app.task(name="send_post_event_nudge")
async def send_post_event_nudge(event_id: str):
    """
    Envoyé le lendemain matin (9h) aux participants qui n'ont PAS encore
    complété leur profil.
    
    Programmé automatiquement quand un event passe en status "completed".
    
    Message WhatsApp :
    "Tu étais au [event_name] hier ! 24 personnes y étaient aussi.
    5 fréquentent les mêmes quartiers que toi.
    Télécharge Flaam pour les découvrir → [lien Play Store]"
    
    Ce message est envoyé UNIQUEMENT aux ghost users et pre_registered users.
    Les utilisateurs qui ont déjà l'app reçoivent une notification push classique.
    """
    async with get_db_session() as db:
        # Ghost/pre_registered → WhatsApp
        ghosts = await db.execute(
            select(User)
            .join(EventRegistration, EventRegistration.user_id == User.id)
            .where(EventRegistration.event_id == event_id)
            .where(EventRegistration.status == "checked_in")
            .where(User.onboarding_status.in_(["ghost", "pre_registered"]))
        )
        for user in ghosts.scalars().all():
            await send_whatsapp(
                phone=user.phone_number,
                template="post_event_nudge",
                params={
                    "event_name": event.title,
                    "attendees_count": attendees_count,
                    "play_store_link": "https://play.google.com/store/apps/details?id=app.flaam",
                },
            )

        # Utilisateurs complets → notification push
        complete_users = await db.execute(
            select(User)
            .join(EventRegistration, EventRegistration.user_id == User.id)
            .where(EventRegistration.event_id == event_id)
            .where(EventRegistration.status == "checked_in")
            .where(User.onboarding_status == "completed")
        )
        for user in complete_users.scalars().all():
            await send_notification(
                user_id=user.id,
                type="event_reveal",
                title=f"{event.title} — qui y était ?",
                body=f"{attendees_count} personnes étaient là. "
                     f"Ouvre ton feed pour les découvrir.",
            )


# ── Celery beat ──
# Cette task n'est PAS dans le beat schedule (pas périodique).
# Elle est programmée dynamiquement quand un event passe à "completed" :
# send_post_event_nudge.apply_async(
#     args=[event_id],
#     eta=event.ends_at + timedelta(hours=10),  # Lendemain 9h environ
# )
```

---

## 8. Nouveaux endpoints (ajoutés au catalogue)

```
| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 109 | GET | /web/events/{slug} | Page web publique de l'event (SSR) | Non |
| 110 | POST | /auth/event-preregister | Pré-inscription event (crée ghost user) | Non |
| 111 | POST | /events/{event_id}/checkin | Check-in QR à l'entrée | Non |

Total endpoints : 108 + 3 = 111
```

---

## 9. Modèle EventRegistration — ajouts

```python
# Ajouts au modèle EventRegistration existant (section Events) :

# registered_via: Mapped[str] = mapped_column(String(10), default="app")
# # "app" = inscription depuis l'app (porte 1/2)
# # "web" = inscription depuis la page web (porte 3)
#
# checked_in_at: Mapped[datetime | None] = mapped_column(
#     DateTime(timezone=True), nullable=True
# )
# # Timestamp du check-in physique
#
# qr_code_hash: Mapped[str | None] = mapped_column(String(64), nullable=True)
# # Hash du QR code généré pour cette inscription
```

---

## Résumé — les 3 portes convergent

```
PORTE 1 (classique)           PORTE 2 (invitation)          PORTE 3 (event)
─────────────────             ──────────────────            ───────────────
Play Store                    Play Store                    Page web event
     ↓                             ↓                            ↓
OTP dans l'app                OTP + code FLAAM-XXXX         OTP web (ghost)
     ↓                             ↓                            ↓
                                                            Event + check-in
                                                                 ↓
                                                            Télécharge l'app
                                                                 ↓
              ╔════════════════════════════════════════╗
              ║   ONBOARDING COMMUN (même écrans)      ║
              ║                                        ║
              ║   Selfie → Photos → Quartiers →        ║
              ║   Spots → Prompts → Intention          ║
              ║                                        ║
              ║   Pré-rempli (porte 3) :               ║
              ║   • Prénom déjà saisi                   ║
              ║   • Spot event pré-coché                ║
              ║   • Message contextuel                  ║
              ╚════════════════════════════════════════╝
                               ↓
              Activation (femme=direct, homme=waitlist
              sauf si code d'invitation ou event)
```


---

# MISE À JOUR 8 — Porte 3 : flow event → app + ghost users

---

## Contexte : les 3 portes d'entrée (indépendantes)

```python
"""
Un utilisateur peut arriver dans Flaam par 3 chemins différents.
Chaque porte est indépendante — aucune n'est un pré-requis.

PORTE 1 — Classique (app directe)
  Télécharge l'app → OTP → onboarding complet → waitlist ou activation
  Le flow standard. Fonctionne sans event ni invitation.

PORTE 2 — Invitation (code d'une amie)
  Reçoit un code FLAAM-XXXX → télécharge l'app → OTP → saisit le code
  → skip waitlist → onboarding complet → activation
  Cf. MàJ 7.

PORTE 3 — Event (pré-inscription web → event → app)
  Page web event → numéro + OTP + prénom (30s, pas d'app)
  → ghost user créé dans Flaam
  → QR check-in à l'event
  → WhatsApp teaser après l'event
  → télécharge l'app → numéro reconnu → onboarding accéléré
  → activation avec boost event
  
  C'est la porte la plus puissante pour l'Afrique de l'Ouest
  parce que le dating y est offline-first, pas online-first.
"""
```

---

## 1. Ghost user — le chaînon manquant

```python
# ── Modification du modèle User ──

# Nouveau champ :
# onboarding_source: Mapped[str] = mapped_column(String(20), default="classic")
# "classic" | "invite" | "event"
# Indique par quelle porte l'utilisateur est arrivé.

# event_preregister_id: Mapped[uuid.UUID | None] = mapped_column(
#     UUID(as_uuid=True), ForeignKey("event_preregistrations.id"), nullable=True
# )

# ── Nouveaux statuts d'onboarding ──

ONBOARDING_STATES = {
    "ghost": {
        # Créé via la page web d'un event (porte 3).
        # A un user_id, un numéro vérifié, un prénom.
        # ZÉRO profil. N'apparaît dans AUCUN feed.
        # N'a PAS l'app installée.
        # Peut être associé à un event (event_preregister).
        "has_app": False,
        "visible_in_feeds": False,
        "can_be_matched": False,
    },
    "pre_registered": {
        # Le ghost user a fait le check-in à l'event (QR scanné).
        # Toujours pas d'app. Mais confirmé présent.
        "has_app": False,
        "visible_in_feeds": False,
        "can_be_matched": False,
    },
    "in_progress": {
        # A téléchargé l'app. Onboarding en cours.
        # Le numéro a été reconnu → ghost user converti.
        # En train de remplir profil, photos, spots, etc.
        "has_app": True,
        "visible_in_feeds": False,
        "can_be_matched": False,
    },
    "completed": {
        # Onboarding terminé. Profil complet.
        # En attente d'activation (waitlist hommes) ou déjà actif.
        "has_app": True,
        "visible_in_feeds": True,  # Si activé
        "can_be_matched": True,    # Si activé
    },
}


# ── Modèle EventPreregistration ──

class EventPreregistration(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "event_preregistrations"

    event_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("events.id"), nullable=False
    )
    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id"), nullable=False
    )
    
    # QR code unique pour le check-in
    qr_code: Mapped[str] = mapped_column(
        String(64), unique=True, nullable=False, index=True
    )
    # Format : "EVT-{event_short_id}-{random_8_chars}"
    
    status: Mapped[str] = mapped_column(String(20), default="registered")
    # "registered" → pré-inscrit via web
    # "checked_in" → QR scanné à l'entrée
    # "converted" → a téléchargé l'app et complété l'onboarding
    # "expired" → event passé, jamais converti (cleanup après 30j)
    
    checked_in_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    converted_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    
    # Tags auto-suggérés basés sur la catégorie de l'event
    suggested_tags: Mapped[list | None] = mapped_column(JSON, nullable=True)
    # Ex: event afterwork → ["afterwork", "sortir", "networking"]

    __table_args__ = (
        Index("ix_event_prereg_event", "event_id"),
        Index("ix_event_prereg_user", "user_id"),
        UniqueConstraint("event_id", "user_id", name="uq_event_prereg_user"),
    )
```

---

## 2. Page web event — pré-inscription légère (porte 3)

```python
# ── Endpoints publics (pas de JWT, pas d'app) ──

# La page web est servie en statique : flaam.app/events/{event_slug}
# Elle affiche :
# - Nom de l'event, date, lieu, catégorie
# - Compteur anonyme : "24 inscrits. 8 fréquentent Tokoin."
#   (count de event_preregistrations + agrégation quartiers anonymisée)
# - Bouton "Réserve ta place (30 secondes)"
# - Champ numéro → OTP → prénom → terminé

# POST /auth/event-preregister
# Body : { "phone_number": "+228 90 12 34 56", "event_id": "..." }
# → Envoie un OTP WhatsApp (même système que l'auth classique)
# → Retourne : { "otp_sent": true, "event_name": "Afterwork Café 21" }

# POST /auth/event-preregister/verify
# Body : { "phone_number": "...", "otp": "123456", "first_name": "Ama", "event_id": "..." }
# → Vérifie l'OTP
# → SI le numéro existe déjà (user Flaam actif) :
#   → Associe l'event à l'utilisateur existant
#   → Retourne : { "status": "existing_user", "message": "Tu es déjà sur Flaam !" }
# → SINON :
#   → Crée un ghost user (onboarding_status=ghost, onboarding_source=event)
#   → Crée un EventPreregistration (status=registered, qr_code généré)
#   → Calcule les suggested_tags depuis la catégorie de l'event
#   → Retourne : { "status": "registered", "qr_code": "EVT-AF21-A3K9M2P1",
#                   "message": "C'est noté ! Tu recevras un rappel vendredi." }
# → PAS de JWT en retour (la personne n'a pas l'app)


EVENT_CATEGORY_TO_TAGS = {
    "afterwork": ["afterwork", "sortir", "networking"],
    "sport": ["sport", "fitness", "outdoor"],
    "brunch": ["food", "brunch", "chill"],
    "cultural": ["culture", "art", "sortir"],
    "networking": ["networking", "pro", "business"],
    "workshop": ["apprendre", "workshop", "créatif"],
    "outdoor": ["outdoor", "nature", "aventure"],
}
# Ces tags sont PRÉ-COCHÉS dans l'onboarding (pas auto-assignés).
# L'utilisateur peut les décocher.


# GET /events/{event_id}/stats (public, pas de JWT)
# → Retourne les stats anonymes pour la page web :
# { "registered_count": 24,
#   "quartier_breakdown": {"Tokoin": 8, "Bè": 5, "Djidjolé": 3, "Autres": 8},
#   "event_name": "Afterwork Café 21",
#   "event_date": "2026-04-18T20:00:00",
#   "spots_left": 36 }  # capacity - registered_count
# IMPORTANT : pas de noms, pas de profils, pas de photos. Juste des compteurs.
```

---

## 3. Check-in à l'event (QR code)

```python
# POST /events/{event_id}/checkin
# Body : { "qr_code": "EVT-AF21-A3K9M2P1" }
# Auth : JWT (organisateur/staff) OU clé API event
# → Vérifie que le QR existe et correspond à cet event
# → Passe le status de "registered" à "checked_in"
# → Ajoute automatiquement le spot de l'event aux spots du ghost user
#   (sera pré-coché quand il fera l'onboarding dans l'app)
# → Retourne : { "status": "checked_in", "first_name": "Ama",
#                 "checked_in_count": 15 }  # pour le staff

# Le QR peut aussi être scanné par l'utilisateur lui-même
# s'il a déjà l'app (utilisateur existant qui s'est pré-inscrit).
# Dans ce cas : le check-in est immédiat via l'app.
```

---

## 4. Conversion ghost → app (le moment clé)

```python
# ── WhatsApp teaser post-event ──

# Celery task : envoyé 2h après la fin de l'event aux checked_in non convertis
# Message : 
# "Tu as croisé {checked_in_count} personnes au {event_name} ce soir.
#  {converted_count} ont déjà complété leur profil Flaam.
#  Télécharge l'app pour les découvrir : https://flaam.app/dl"

# ── Détection du ghost user dans l'app ──

# Quand un utilisateur fait l'OTP classique (porte 1) et que son numéro
# correspond à un ghost user, l'app détecte ça côté serveur :

# POST /auth/otp/verify retourne un champ supplémentaire :
# {
#   "token": "...",
#   "is_ghost_conversion": true,
#   "ghost_data": {
#     "first_name": "Ama",
#     "onboarding_source": "event",
#     "event_name": "Afterwork Café 21",
#     "event_spot_id": "uuid-cafe-21",
#     "suggested_tags": ["afterwork", "sortir", "networking"],
#     "attendees_completed": 7,  # combien ont fini l'onboarding
#   }
# }

# L'app utilise ghost_data pour :
# 1. Pré-remplir le prénom (skip l'écran prénom)
# 2. Pré-cocher le spot de l'event dans l'écran spots
# 3. Pré-cocher les suggested_tags dans l'écran tags
# 4. Afficher un message motivant :
#    "7 personnes du Café 21 ont déjà complété leur profil.
#     Complete le tien pour les découvrir demain."
# 5. Après complétion : EventPreregistration.status → "converted"
```

---

## 5. Boost event dans le matching

```python
# app/services/matching_engine/event_boost.py
"""
Les participants d'un même event récent reçoivent un boost temporaire
dans le feed les uns des autres.

Ce n'est PAS un filtre (les non-participants apparaissent aussi).
C'est un bonus de +15 points sur le score L2 (géo) entre deux
personnes qui étaient au même event dans les 7 derniers jours.

Après 7 jours, le boost disparaît progressivement (decay linéaire).
Après 14 jours, il est à zéro.
"""

EVENT_BOOST_CONFIG = {
    "boost_points": 15,          # Ajouté au score L2 (sur 100)
    "full_boost_days": 7,        # Boost complet pendant 7 jours
    "decay_days": 7,             # Puis decay linéaire sur 7 jours
    "total_duration_days": 14,   # Après 14 jours = 0
    
    "requires_checkin": True,
    # Le boost ne s'applique qu'aux checked_in, pas aux simple registered.
    # Tu dois avoir été physiquement présent.
    
    "requires_completed_profile": True,
    # Les ghost users non convertis ne reçoivent pas le boost
    # (ils ne sont pas dans le matching de toute façon).
}


async def compute_event_boost(
    user_id: str,
    candidate_id: str,
    db_session,
) -> float:
    """
    Retourne le boost event entre deux utilisateurs.
    0 si aucun event en commun récent.
    """
    now = datetime.utcnow()
    cutoff = now - timedelta(days=EVENT_BOOST_CONFIG["total_duration_days"])
    
    # Trouver les events en commun où les deux étaient checked_in
    common_events = await db_session.execute(
        select(EventPreregistration.event_id, Event.ended_at)
        .join(Event, Event.id == EventPreregistration.event_id)
        .where(
            EventPreregistration.user_id == user_id,
            EventPreregistration.status.in_(["checked_in", "converted"]),
            Event.ended_at >= cutoff,
            exists(
                select(EventPreregistration.id).where(
                    EventPreregistration.event_id == EventPreregistration.event_id,
                    EventPreregistration.user_id == candidate_id,
                    EventPreregistration.status.in_(["checked_in", "converted"]),
                )
            ),
        )
    )
    
    max_boost = 0.0
    for _, ended_at in common_events.fetchall():
        days_since = (now - ended_at).days
        if days_since <= EVENT_BOOST_CONFIG["full_boost_days"]:
            boost = EVENT_BOOST_CONFIG["boost_points"]
        elif days_since <= EVENT_BOOST_CONFIG["total_duration_days"]:
            # Decay linéaire
            remaining = EVENT_BOOST_CONFIG["total_duration_days"] - days_since
            decay_window = EVENT_BOOST_CONFIG["decay_days"]
            boost = EVENT_BOOST_CONFIG["boost_points"] * (remaining / decay_window)
        else:
            boost = 0
        max_boost = max(max_boost, boost)
    
    return max_boost

# Intégration dans pipeline.py :
# Après le calcul de geo_scores, ajouter le boost event :
# for cid in candidate_ids:
#     event_bonus = await compute_event_boost(user_id, cid, db_session)
#     geo_scores[cid] = min(100.0, geo_scores.get(cid, 0) + event_bonus)
```

---

## 6. Badge event + ice-breaker contextuel

```python
# ── Badge "Était au [event]" dans le feed ──

# Quand le feed est généré et que deux utilisateurs partagent un event
# récent (< 14 jours), le profil dans le feed affiche un badge :
# { "event_badge": { "name": "Afterwork Café 21", "days_ago": 3 } }
# Visible pendant 14 jours, ensuite le badge disparaît.

# ── Ice-breaker contextuel ──

# Quand deux participants du même event matchent, l'ice-breaker
# générique est remplacé par un ice-breaker contextuel :

EVENT_ICEBREAKERS = [
    "Vous étiez tous les deux au {event_name}. Qu'est-ce que tu en as pensé ?",
    "Vous avez fréquenté le même event : {event_name}. C'était comment pour toi ?",
    "Le {event_name}, c'était bien ? Vous y étiez tous les deux !",
]
# Sélection aléatoire, injecté dans le match au lieu du ice-breaker classique.
```

---

## 7. Nouveaux endpoints

```
| # | Méthode | Path | Description | Auth |
|---|---------|------|-------------|------|
| 109 | POST | /auth/event-preregister | Pré-inscription event (envoie OTP) | Non |
| 110 | POST | /auth/event-preregister/verify | Vérifie OTP + crée ghost user | Non |
| 111 | GET | /events/{event_id}/stats | Stats anonymes pour page web | Non |
| 112 | POST | /events/{event_id}/checkin | Check-in QR à l'entrée | Staff |

Total endpoints : 108 + 4 = 112
```

---

## 8. Celery tasks ajoutées

```python
# event_post_teaser : 2h après la fin de chaque event
# → Envoie le WhatsApp teaser aux checked_in non convertis
# → "Tu as croisé X personnes. Y ont déjà leur profil. Télécharge Flaam."

# event_prereg_cleanup : hebdomadaire
# → Les ghost users non convertis après 30 jours → status "expired"
# → Les EventPreregistration expirées → soft delete
# → PAS de suppression du ghost user (il pourrait s'inscrire plus tard
#   par la porte 1 ou 2, le numéro sera reconnu)
```

---

# Mise à jour 9 — Genre immutable, premium downgrade, safety pipeline photos

## Contexte

Décisions produit prises lors de la conception du business model et du système anti-fraude. Ces règles affectent Sessions 10 et 11.

## 1. Genre non modifiable par l'utilisateur

**Règle :** le champ `gender` (man/woman/non_binary) est déclaré à l'onboarding (étape basic_info) et **verrouillé**. L'utilisateur ne peut PAS le modifier via `PUT /profiles/me`.

**Raison :** le genre est un paramètre structurel du matching (waitlist genrée, first impression feed, quality gate masculin). Permettre le changement ouvrirait la porte au catfish (photos de sa sœur + changement genre).

**Le champ `seeking_gender` (qui tu cherches) reste librement modifiable** — c'est une préférence, pas une identité.

**Changement légitime :** uniquement via admin :

```python
# Session 10 — endpoint admin
# PATCH /admin/users/{user_id}/gender
async def admin_change_gender(user_id, new_gender, reason, admin_user, db):
    """
    Changement de genre par un admin après review humaine.
    Conséquences automatiques :
    1. Genre mis à jour
    2. Selfie INVALIDÉ (nouveau selfie requis)
    3. Behavior multiplier RESET à 1.0
    4. Notification à l'utilisateur
    5. Log audit trail
    """
    user = await db.get(User, user_id)
    old_gender = user.profile.gender
    user.profile.gender = new_gender
    user.is_selfie_verified = False
    # Reset behavior (nouveau genre = nouveau contexte)
    await redis.delete(f"behavior:{user_id}")
    
    log.warning(
        "admin_gender_change",
        user_id=str(user_id),
        old_gender=old_gender,
        new_gender=new_gender,
        reason=reason,
        admin_id=str(admin_user.id),
    )
    
    await notification_service.send_push(
        user_id, "profile_update_required",
        {"message_fr": "Ton profil a été mis à jour. Reprends un selfie."}
    )
```

**Dans `profile_service.update_profile()` :**

```python
if "gender" in data:
    raise AppException(
        status.HTTP_400_BAD_REQUEST,
        "gender_not_modifiable",
        "Le genre ne peut pas être modifié. Contacte le support.",
    )
```

## 2. Premium downgrade — Gel doux

Quand un abonnement premium expire, les données au-delà des limites free ne sont **PAS supprimées**. Elles sont **gelées** (ignorées par le matching) et restent en DB pour réactivation instantanée.

### Limites free vs premium

| Ressource | Free | Premium |
|-----------|------|---------|
| Quartiers physiques (lives/works/hangs) | 3 | 5 |
| Quartiers interested | 3 | 6 |
| Spots | 5 | 12 |
| Likes/jour | 5 | 10 (configurable à 15 via admin) |

### Logique de downgrade

```python
# app/services/subscription_service.py

async def downgrade_user_limits(user: User, db: AsyncSession) -> None:
    """
    Gel doux des extras premium. Appelé quand Subscription.expires_at < now.
    Les données restent en DB, juste is_active_in_matching = False.
    """
    # Quartiers physiques : garder les 3 plus anciens
    quartiers = sorted(user.user_quartiers, key=lambda q: q.created_at)
    physical = [q for q in quartiers if q.relation_type != "interested"]
    for i, q in enumerate(physical):
        q.is_active_in_matching = (i < 3)
    
    # Quartiers interested : garder les 3 plus anciens
    interested = [q for q in quartiers if q.relation_type == "interested"]
    for i, q in enumerate(interested):
        q.is_active_in_matching = (i < 3)
    
    # Spots : garder les 5 plus anciens
    spots = sorted(user.user_spots, key=lambda s: s.created_at)
    for i, s in enumerate(spots):
        s.is_active_in_matching = (i < 5)
    
    # Premium flags
    user.is_premium = False
    
    await db.commit()
    
    log.info("premium_downgraded", user_id=str(user.id))


async def upgrade_user_limits(user: User, db: AsyncSession) -> None:
    """Réactive tout quand premium se réactive."""
    for q in user.user_quartiers:
        q.is_active_in_matching = True
    for s in user.user_spots:
        s.is_active_in_matching = True
    user.is_premium = True
    await db.commit()
```

### Comportement côté mobile

Quand premium expire :
- Extras grisés dans l'app avec badge "Premium"
- Notification push : "Ton premium a expiré. Tes quartiers et spots extras sont en pause."
- Écran avec 2 choix : "Réactiver tout (Premium)" ou "Choisir lesquels garder"
- Si l'utilisateur ne choisit pas en 48h → fallback : les 3/5 plus anciens restent actifs
- Likes reviennent à 5/jour immédiatement
- Likes-received revient au mode free (count + preview floutées)
- Mode incognito désactivé immédiatement

### Celery task (Session 11)

```python
# app/tasks/subscription_tasks.py

async def check_expired_subscriptions():
    """
    Tourne toutes les heures. Détecte les subscriptions expirées
    non encore traitées et déclenche le downgrade.
    """
    expired = await db.execute(
        select(Subscription)
        .where(Subscription.expires_at < func.now())
        .where(Subscription.is_active == True)
    )
    for sub in expired.scalars():
        sub.is_active = False
        user = await db.get(User, sub.user_id)
        await downgrade_user_limits(user, db)
        await notification_service.send_push(
            user.id, "premium_expired",
            {"message_fr": "Ton premium a expiré. Tes extras sont en pause."}
        )
```

## 3. Pipeline de modération photos — Session 10

### Ordre du pipeline (6 checks)

```
Photo uploadée
    │
    ▼
1. EXIF check (< 1ms, pur Python)
    │ → Flag si no_exif + square_resolution + ai_software
    ▼
2. NSFW detection (~100ms, ONNX)
    │ → Rejected si score > 0.7
    │ → Manual_review si score 0.4-0.7
    ▼
3. Face detection (~50ms, YuNet ONNX)
    │ → Manual_review si 0 visage détecté
    ▼
4. Selfie ↔ Photo face comparison (~200ms, ArcFace ONNX)
    │ → Flag si similarity < 0.5
    │ → Flag + notif admin si similarity < 0.3
    ▼
5. Genre consistency (inclus dans InsightFace)
    │ → Flag pour review humaine si mismatch (PAS auto-reject — trans/NB)
    ▼
6. Diversité temporelle (< 1ms, EXIF dates)
    │ → Flag si 4+ photos même date EXIF
    ▼
Decision : approved | manual_review | rejected
```

**Temps total : ~350ms par photo sur CPU.** Acceptable pour du Celery async.

### Modèles ONNX nécessaires

| Modèle | Usage | Taille | Source |
|--------|-------|--------|--------|
| ArcFace ResNet50 | Face embeddings (selfie ↔ photos) | ~30 MB | InsightFace GitHub |
| YuNet | Face detection | ~2 MB | OpenCV contrib |
| NSFW Classifier | Contenu adulte | ~50 MB | GantMan/nsfw_model |
| **Total** | — | **~82 MB** | — |

### Configuration ENV

```bash
PHOTO_MODERATION_MODE=manual        # manual | onnx | external | off
EXIF_CHECK_ENABLED=true             # Toujours, même en mode manual
FACE_VERIFICATION_ENABLED=false     # Active quand modèles installés
FACE_VERIFICATION_MODEL_PATH=/models/arcface_r50.onnx
FACE_GENDER_CHECK_ENABLED=false     # Active avec FACE_VERIFICATION
NSFW_MODEL_PATH=/models/nsfw_detector.onnx
FACE_DETECTION_MODEL_PATH=/models/yunet_face.onnx
NSFW_THRESHOLD_REJECT=0.7
NSFW_THRESHOLD_REVIEW=0.4
```

## 4. Targeted likes (feature flag, désactivé au MVP)

Extension de `POST /feed/{profile_id}/like` pour accepter optionnellement :

```python
# Body enrichi (champs optionnels)
{
    "target_type": "photo" | "prompt" | "profile",  # default "profile"
    "target_id": "uuid-de-la-photo-ou-index-du-prompt",
    "comment": "Ton maquis préféré c'est aussi le mien !",  # max 200 chars
}
```

Champs ajoutés sur Match :
- `like_target_type: String(10) | None`
- `like_target_id: String(100) | None`
- `like_comment: String(200) | None`

Quand `target_type != "profile"` et un `comment` est fourni :
- L'ice-breaker auto-généré est **remplacé** par le commentaire de l'utilisateur
- Plus humain, plus engageant

**Feature flag :** `flag_targeted_likes_enabled` (default 0.0 = désactivé au MVP).
Quand désactivé, les champs target_type/target_id/comment sont ignorés.

### A/B testing des prompts (tracking passif)

Quand un like cible un prompt spécifique, le compteur `prompt_like_count` est incrémenté dans le JSONB prompts du Profile. Permet de mesurer quels prompts attirent le plus de likes. Pas d'endpoint dédié — l'admin le consultera via `GET /admin/prompts/stats` (Session 10).

## 5. Reply reminders (feature flag, activé au MVP)

```python
# app/services/reminder_service.py

async def check_pending_replies(db: AsyncSession) -> list[dict]:
    """
    Trouve les conversations avec un message non-répondu depuis > 24h.
    Max 1 reminder par match par 48h (pas de spam).
    Respecte notification_preferences (flag reply_reminders).
    
    Stub Celery — scheduling en Session 11.
    """
    ...
```

**Feature flag :** `flag_reply_reminders_enabled` (default 1.0 = **activé**).
C'est la seule feature activée par défaut car elle réduit le ghosting.

**Template notification :**
- FR : "Tu n'as pas encore répondu à {name}. Elle attend ta réponse."
- EN : "You haven't replied to {name} yet. She's waiting for you."