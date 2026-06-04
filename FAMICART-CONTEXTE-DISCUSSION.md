# FamiCart — Contexte exhaustif de la discussion

> Document de reprise pour continuer le projet avec un autre agent ou une nouvelle session.  
> Synthèse de toutes les idées, décisions et pistes techniques évoquées.

---

## Table des matières

1. [Vision produit](#1-vision-produit)
2. [Personas et cas d'usage](#2-personas-et-cas-dusage)
3. [Modèle Famille / Foyers](#3-modèle-famille--foyers)
4. [Fonctionnalités listes collaboratives](#4-fonctionnalités-listes-collaboratives)
5. [Tickets OCR et exploitation des lignes](#5-tickets-ocr-et-exploitation-des-lignes)
6. [Remboursements et suivi dépenses (fille)](#6-remboursements-et-suivi-dépenses-fille)
7. [Stack technique retenue et préférences](#7-stack-technique-retenue-et-préférences)
8. [Architecture cible](#8-architecture-cible)
9. [Schéma de données (conceptuel)](#9-schéma-de-données-conceptuel)
10. [Sécurité RLS et visibilité](#10-sécurité-rls-et-visibilité)
11. [Backend .NET : quand et pourquoi](#11-backend-net--quand-et-pourquoi)
12. [Stockage fichiers : DigitalOcean Spaces / S3](#12-stockage-fichiers--digitalocean-spaces--s3)
13. [Intégration Alexa / Alexa+](#13-intégration-alexa--alexa)
14. [Nom du projet et repository GitHub](#14-nom-du-projet-et-repository-github)
15. [Cloud Agent Cursor — changer de repository](#15-cloud-agent-cursor--changer-de-repository)
16. [Roadmap proposée par phases](#16-roadmap-proposée-par-phases)
17. [Questions ouvertes à trancher](#17-questions-ouvertes-à-trancher)
18. [Liens et références utiles](#18-liens-et-références-utiles)

---

## 1. Vision produit

**FamiCart** (nom retenu) est une application familiale pour :

- Gérer des **listes de courses partagées** en temps réel (ex. l'épouse prépare la liste, le conjoint fait les courses et coche les articles ; l'autre voit ce qui est coché).
- **Scanner des tickets de caisse** (OCR type Mistral OCR) pour enregistrer les achats, les lignes, les montants.
- Suivre les **dépenses de la fille étudiante** (foyer séparé) et gérer les **remboursements** parents ↔ fille avec preuve ticket.
- Fonctionner en **mobile** (prioritaire) et **web** depuis **une seule codebase** (pas deux apps séparées).

L'OCR est un **accélérateur** ; la confiance repose sur la **validation humaine** post-OCR et le rapprochement explicite liste ↔ ticket.

---

## 2. Personas et cas d'usage

### Composition familiale

| Personne | Rôle métier | Logement principal |
|----------|-------------|-------------------|
| Utilisateur (papa) | Parent | Foyer parents (avec épouse) |
| Épouse | Parent / éditrice de listes | Foyer parents |
| Fille | Étudiante | Foyer Belgique (semaine) ; présente chez parents week-ends / vacances |

### Cas d'usage 1 — Couple (foyer parents)

- L'épouse crée une liste de courses et la partage.
- Le conjoint va en magasin, coche les articles ; elle voit l'avancement en temps réel.
- Après achat : scan du ticket pour archiver et éventuellement rapprocher avec la liste.

### Cas d'usage 2 — Fille (deux foyers)

- En semaine : courses à **son appart en Belgique** (foyer fille).
- Week-ends / vacances : peut faire des courses au **foyer parents**.
- Elle scanne ses tickets ; les **parents** doivent **voir**, **valider** et suivre les **remboursements**.
- Il faut distinguer **pour quel foyer** chaque course / ticket compte (pas seulement « où elle est » géographiquement).

### Cas d'usage 3 — Voix et alertes (souhait)

- S'appuyer sur **Alexa+** pour alimenter la liste (« ajoute du lait ») et remonter des **alertes** (liste mise à jour, ticket à valider, etc.) — voir section Alexa.

---

## 3. Modèle Famille / Foyers

### Hiérarchie à deux niveaux

| Niveau | Nom produit (FR) | Nom technique suggéré | Rôle |
|--------|------------------|----------------------|------|
| **Au-dessus** | **Famille** | `families` | 3 membres, rôles parent/enfant, remboursements, accès transversal aux données des foyers « enfants » selon rôle |
| **En dessous** | **Foyer** / **Lieu** | `homes` | Chaque logement : listes, tickets, habitudes OCR, stats d'achat **par lieu** |

**Ne pas** utiliser le seul mot `household` pour les deux niveaux (ambiguïté EN).

### Foyer fille : double appartenance

- La fille est **membre résidente des deux foyers** :
  - **Foyer parents** (`home_members` : resident ou resident_secondary)
  - **Foyer Belgique** (`home_members` : resident principal en semaine)
- Les parents ont le rôle **`parent`** au niveau **famille** → accès lecture / validation sur le foyer Belgique **sans** y être résidents.

### Question clé : « Pour quel foyer ? »

- Ce n'est **pas** « où êtes-vous ? » (GPS peut tromper) mais **« pour quel foyer cette course / ce ticket compte ? »**
- **Moment recommandé pour demander** : au début d'une **session courses** (« Je fais les courses ») ou à la **création de liste** — **une fois par intention d'achat**, pas à chaque article coché.
- Si la fille ouvre une **liste partagée** créée par les parents sur le foyer parents → le `home_id` de la liste **prime**, pas de question.
- **Scan ticket** : même foyer par défaut que la session / liste, **modifiable** avant validation OCR.
- **Sélecteur de foyer** persistant dans l'UI pour la fille ; défaut réglable (ex. semaine = Belgique).

### Concept `shopping_sessions` (recommandé)

```
shopping_sessions
  id, user_id, home_id, started_at, ended_at, list_id (nullable)

receipts
  home_id NOT NULL
  shopping_session_id (nullable)
  uploaded_by
```

Évite de rattacher un ticket au mauvais foyer le même jour.

---

## 4. Fonctionnalités listes collaboratives

### MVP listes

- Comptes + invitation (email ou lien magique).
- Listes partagées avec **sync temps réel** (cocher / décocher).
- Mode **« je fais les courses »** : UI mobile optimisée (gros boutons, offline souhaité).
- Historique des listes clôturées.

### Collaboration temps réel

- **Optimistic UI** + file d'attente offline (Capacitor SQLite ou Ionic Storage) ; sync au retour réseau.
- Afficher **« coché par X »** ; version / `updated_at` par item pour limiter les conflits.
- Supabase **Realtime** sur `list_items` ; limiter les subscriptions aux listes ouvertes.

### Événements à diffuser

`item_added`, `item_checked`, `item_removed`, `list_closed`, `shopper_joined`.

---

## 5. Tickets OCR et exploitation des lignes

### Pipeline OCR

1. Photo (Capacitor Camera) → compression JPEG conseillée (~1600px large).
2. Upload vers **stockage objet** (Spaces/S3) via **URL présignée** (PUT direct depuis l'app).
3. Job asynchrone : **Mistral OCR** (ou équivalent) via **.NET** ou Edge Function.
4. Parse JSON → tables `receipts` + `receipt_lines`.
5. **Écran de correction obligatoire** avant validation (l'OCR n'est jamais 100 % fiable).
6. Conserver `raw_ocr_json` pour ré-analyse future.
7. Notification « ticket prêt à valider » (push et/ou Alexa Proactive Events).

### Données par ligne ticket

- Libellé brut, quantité, prix unitaire, total ligne.
- Ticket : date, magasin, total TTC.
- Enrichissement progressif : `canonical_name`, catégorie, `household_products` (catalogue foyer).

### Idées de fonctionnalités basées sur l'historique OCR

#### Listes intelligentes (idée initiale utilisateur + extensions)

| Fonctionnalité | Description |
|----------------|-------------|
| **Souvent achetés** | Articles achetés ≥ N fois sur 8 semaines → section « Souvent achetés » à la création de liste |
| **Autocomplétion** | Recherche sur libellés historiques + dernière quantité moyenne |
| **Liste modèle auto** | « Créer à partir des habitudes » (panier moyen, ex. courses du vendredi) |
| **Oublis probables** | Comparer liste en cours aux 2–3 derniers tickets : articles souvent achetés mais absents de la liste |
| **Liste par magasin** | Modèles / habitudes si magasin identifié sur ticket |

#### Rapprochement liste ↔ ticket

| Fonctionnalité | Description |
|----------------|-------------|
| **Coche automatique suggérée** | Fuzzy match (`pg_trgm` ou Levenshtein) ligne ticket ↔ item liste |
| **Écart prévu / réel** | Sur liste pas sur ticket ; sur ticket pas sur liste (impulsif / oubli) |
| **Substitutions mémorisées** | Ex. « lait ½ écrémé » ↔ « LAIT UHT 1L » validé une fois → réutilisé |

#### Historique et favoris

- « Dernière fois que j'ai acheté… » (date, magasin, prix).
- Favoris explicites + **favoris implicites** (fréquence d'achat).
- Timeline des courses (ticket + liste + qui a acheté).

#### Prix et budget

- **Prix de référence** à l'ajout d'un article (dernier prix, min/max 6 mois).
- **Estimation totale** de la liste vs total ticket post-scan.
- Alertes prix aberrant ou doublon ticket.

#### Analytics (phase ultérieure)

- Répartition par catégorie (% alimentation / entretien).
- Évolution panier / semaine ; top articles ; panier moyen par enseigne.
- Saisonnalité simple sur historique.

#### Qualité données

- Table `household_products` : `canonical_name`, alias OCR, catégorie, unité.
- Fusion doublons libellés ; apprentissage des corrections utilisateur.

#### Inventaire léger (optionnel, prudent)

- Consommation estimée pour produits répétitifs (« probablement à racheter ») — **ne pas sur-promettre**.

### Ordre de valeur produit OCR

1. Valider et normaliser les lignes.
2. Souvent achetés + autocomplétion.
3. Rapprochement liste / ticket.
4. Prix de référence + budget.
5. Modèles de liste + oublis.
6. Analytics et inventaire léger.

---

## 6. Remboursements et suivi dépenses (fille)

- Ticket sur **foyer Belgique** + `expense_claim` au niveau **famille**.
- Statuts suggérés : `draft` → `submitted` → `approved` → `rejected` → `reimbursed`.
- Parents voient filtre « Dépenses chez Léa ce mois », claims en attente.
- Règles par foyer possibles : ticket foyer parents ≠ flux remboursement.
- Commentaires sur une ligne ticket pour dialogue parent / fille.
- Export CSV/PDF pour compta familiale (phase ultérieure).

---

## 7. Stack technique retenue et préférences

### Maîtrisé / souhaité par l'utilisateur

| Couche | Technologie |
|--------|-------------|
| **Mobile + Web (une codebase)** | **Angular** + **Ionic** + **Capacitor** |
| **BaaS / données / auth / realtime** | **Supabase** (Postgres, Auth, Realtime, RLS) |
| **Backend métier** | **ASP.NET Core** (.NET) — quand nécessaire |
| **Stockage fichiers** (tickets, pièces jointes) | **DigitalOcean Spaces** ou **AWS S3** (API compatible S3) ; maîtrise utilisateur |
| **OCR** | **Mistral OCR** (style) ou équivalent |
| **Voix / alertes** | **Alexa+** / Skill Alexa (voir section dédiée) |

### Angular / Ionic

- Viser **Angular 17+** (standalone components, signals optionnels).
- **Capacitor** : Camera, Network, Push Notifications, SQLite (offline).
- **PWA** possible pour usage navigateur.
- Structure suggérée :

```
src/app/
  core/           # auth, supabase client, guards
  features/
    lists/
    shopping/     # mode courses
    receipts/
    expenses/
  shared/
```

### Supabase

- `@supabase/supabase-js` dans un service injectable.
- **RLS strict** : clé `anon` côté client uniquement ; **service role** uniquement côté .NET.
- Realtime sur `list_items`.
- Configurer redirect URLs pour magic link + deep links Capacitor (`appUrlOpen`).

### Ce qu'on évite au début

- Backend NestJS séparé si Supabase + .NET suffisent.
- Deux apps web + mobile séparées.
- Microservices OCR.
- Supabase Storage si l'utilisateur préfère Spaces (Postgres Supabase + fichiers Spaces).

---

## 8. Architecture cible

```
┌─────────────────────────────────────────────────────────────┐
│  Angular + Ionic + Capacitor (iOS, Android, Web/PWA)        │
└────────────┬───────────────────────────────┬────────────────┘
             │                               │
             │ CRUD listes, Realtime, Auth     │ Upload photo
             ▼                               ▼
┌────────────────────────┐         ┌─────────────────────────┐
│  Supabase              │         │  ASP.NET Core API       │
│  - Postgres + RLS      │◄────────│  - Presigned URLs S3    │
│  - Auth                │         │  - OCR orchestration    │
│  - Realtime            │         │  - Remboursements       │
│  - (pas les fichiers)  │         │  - Webhook Alexa Skill  │
└────────────────────────┘         │  - Proactive Events     │
             ▲                     └───────────┬─────────────┘
             │                                 │
             │                                 ▼
             │                     ┌─────────────────────────┐
             │                     │  DO Spaces / S3         │
             │                     │  tickets, pièces jointes│
             │                     └───────────┬─────────────┘
             │                                 │
             │                                 ▼
             │                     ┌─────────────────────────┐
             │                     │  Mistral OCR API          │
             └─────────────────────┴─────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Alexa Skill / Alexa+ Actions  →  API .NET  →  Supabase      │
└─────────────────────────────────────────────────────────────┘
```

### Répartition des responsabilités

| Opération | Canal |
|-----------|--------|
| Listes, coches, Realtime | App → **Supabase direct** |
| Upload / lecture ticket | App → **.NET** (presigned) → **Spaces** |
| OCR, lignes, validation | **.NET** → OCR → **Supabase** |
| Remboursements sensibles | App → **.NET** (JWT Supabase validé) |
| Voix Alexa | **Skill** → **.NET** |
| Alertes push riches | **FCM** + Capacitor |
| Alertes Alexa | **Proactive Events** (schémas imposés) |

---

## 9. Schéma de données (conceptuel)

```text
families
  id, name, created_at

family_members
  family_id, user_id
  role: owner | parent | adult | student | child

homes
  id, family_id, name, type: primary | secondary | student

home_members
  home_id, user_id
  role: resident | manager | viewer

profiles
  id (lié auth.users), ...

lists
  id, home_id, created_by, title, status: active | shopping | closed

list_members (optionnel si permissions fines)
  list_id, user_id, role

list_items
  id, list_id, label, quantity?, category?, sort_order
  checked_at, checked_by, version/updated_at
  linked_product_id? (household_products)

shopping_sessions
  id, user_id, home_id, list_id?, started_at, ended_at

receipts
  id, home_id, family_id (ou dérivé), uploaded_by
  storage_provider: 'do_spaces' | 'aws_s3'
  bucket, storage_key, mime_type, byte_size
  ocr_status: pending | processing | done | failed
  raw_ocr_json, merchant, purchased_at, total_ttc
  shopping_session_id?

receipt_lines
  id, receipt_id, description, qty, unit_price, line_total
  match_status: unmatched | matched_list_item | manual
  list_item_id?

household_products
  id, home_id (ou family_id selon choix)
  canonical_name, aliases[], category, default_unit

purchase_stats (vue matérialisée ou table dérivée)
  home_id, product_id, frequency, last_price, ...

expense_claims
  id, family_id, home_id, receipt_id, submitted_by
  amount, currency, status
  approved_by, notes, timestamps
```

**Préfixe clés S3** : `households/{familyId}/homes/{homeId}/receipts/{receiptId}.jpg`

---

## 10. Sécurité RLS et visibilité

### Principes

- Bucket **privé** ; lecture/écriture via **URLs présignées** courtes (PUT upload, GET affichage).
- **Jamais** de secret Spaces / service role Supabase dans l'app mobile.
- Corrélation : statut `pending_upload` jusqu'à confirmation Postgres après PUT.

### Helpers RLS suggérés

- `is_family_member(family_id)`
- `can_access_home(home_id)` :
  - membre `home_members` **OU**
  - `family_members.role IN ('parent', 'owner')` pour la famille du foyer

### Matrice simplifiée

| Donnée | Résidents du foyer | Parents (autre foyer) | Fille sur foyer parents |
|--------|-------------------|------------------------|-------------------------|
| Listes | CRUD / cocher | Lire (+ éditer si manager) | Selon rôle |
| Tickets | CRUD + scan | Lire + valider remboursement | Ses tickets chez elle visibles par parents |
| Stats / suggestions | Par `home_id` | Agrégat famille optionnel | Séparé par foyer |
| Remboursements | — | Valider | Elle soumet |

---

## 11. Backend .NET : quand et pourquoi

### Pas nécessaire au début pour

- MVP listes partagées + Realtime + RLS.

### Utile / recommandé pour

| Besoin | Raison |
|--------|--------|
| **Presigned URLs** Spaces/S3 | Secrets côté serveur |
| **OCR long** (Mistral) | Workers, retries, Hangfire/queues ; dépasse limites Edge Functions |
| **Workflow remboursements** | Règles métier, audit, tests xUnit |
| **Webhooks Storage** → OCR | Déclenchement fiable post-upload |
| **Skill Alexa + Proactive Events** | Endpoints stables, JWT validation |
| **Rapprochement fuzzy avancé** | Logique C# + `pg_trgm` |

### Forme recommandée

- **ASP.NET Core Minimal API** ou controllers fins.
- Supabase reste la DB ; .NET utilise **Npgsql** ou client Supabase **service role** pour écritures post-OCR.
- Valider le **JWT Supabase** sur les routes sensibles.
- **Ne pas** recréer tout le CRUD listes en .NET (perte du Realtime).

### Approche progressive

1. **Supabase-first** (listes).
2. Ajouter **.NET ciblé** (presigned, OCR, remboursements).
3. **Alexa** en phase voix.

---

## 12. Stockage fichiers : DigitalOcean Spaces / S3

### Pourquoi pas Supabase Storage (pour cet utilisateur)

- Maîtrise déjà **AWS S3 / DO Spaces**.
- Coût, lifecycle, backup, CDN sous contrôle.
- Supabase reste vérité **relationnelle** ; fichiers externes.

### Flux upload recommandé (presigned PUT)

1. App → `POST /api/receipts/{id}/upload-url` (.NET).
2. .NET vérifie JWT + droit sur foyer → génère PUT presigned + `storage_key`.
3. App → PUT direct vers Spaces.
4. App → confirme en base (`confirm-upload`).
5. Webhook / event → job OCR .NET.

### Bonnes pratiques

- Compression image côté app avant PUT.
- Lifecycle bucket (archivage / suppression RGPD).
- Stocker `storage_key` en base, pas URL publique permanente.
- Job nettoyage orphelins S3 si Postgres échoue après upload.

### SDK .NET

- `AmazonS3Client` avec `ServiceURL` = endpoint Spaces.
- `GetPreSignedUrl` pour PUT et GET.

---

## 13. Intégration Alexa / Alexa+

### Contexte important : API Listes Alexa dépréciée

- Depuis le **1er juillet 2024**, **List Management REST API** et **List skills** ne sont **plus supportés**.
- **Impossible** de synchroniser l'app avec la liste Courses native d'Alexa.
- La **source de vérité** reste **FamiCart (Supabase)** ; la voix passe par **une skill custom** ou **Alexa+ Actions**.

### Alexa+ — SDK AI-native (early access)

| SDK | Usage |
|-----|--------|
| **Alexa AI Action SDK** | Brancher les **API REST .NET** ; Alexa+ appelle `addListItem`, etc. — **meilleure piste long terme** |
| **Alexa AI Web Action SDK** | Navigation sur site web (moins robuste qu'API) |
| **Alexa AI Multi-Agent SDK** | Agents tiers à côté d'Alexa (partenaires) |

- Demande d'accès : [developer.amazon.com/alexa/alexa-ai](https://developer.amazon.com/en-US/alexa/alexa-ai)
- Les skills **classiques ASK** restent supportées ; nouveaux outils orientés Alexa+.

### Approche recommandée (phases)

**Phase 1 — Skill Alexa classique (aujourd'hui)**

- Intents FR : `AddItem`, `ReadList`, `SetActiveHome`, etc.
- Backend skill : endpoint HTTPS **ASP.NET Core** (ou Lambda).
- **Account Linking** (Amazon ↔ compte FamiCart / Supabase) obligatoire.
- Gestion **foyer actif** pour la fille (slot `HomeName` ou question en session).

**Phase 2 — Proactive Events (alertes)**

- API : [Proactive Events](https://developer.amazon.com/en-US/docs/alexa/smapi/proactive-events-api.html)
- Endpoint Europe : `https://api.eu.amazonalexa.com/v1/proactiveEvents/`
- **Pas de message libre** : schémas imposés ([liste](https://developer.amazon.com/en-US/docs/alexa/smapi/schemas-for-proactive-events.html)) :
  - `AMAZON.MessageAlert.Activated` → « X nouveaux articles sur la liste… », « Ticket à valider… »
  - `AMAZON.Occasion.Updated` → rappels type rendez-vous / courses prévues
  - `AMAZON.OrderStatus.Updated` — utiliser avec prudence (sémantique commande)
- Quotas / 24h ; utilisateur doit activer notifications pour la skill.
- Certification Amazon requise pour production.

**Phase 3 — Alexa AI Action SDK** quand API .NET stable et accès obtenu.

### Complément

- **FCM** + Capacitor Push pour alertes détaillées et deep links.
- Alexa pour mains libres et rappels courts.

### Exemples utterances

- « Alexa, ajoute du lait à FamiCart pour chez papa et maman. »
- « Qu'est-ce qu'il reste sur la liste ? »
- « Je fais les courses pour mon appart. »

### Liens

- [AI-native SDKs for Alexa+](https://developer.amazon.com/en-US/alexa/alexa-ai)
- [Fonctionnalités dépréciées (listes)](https://developer.amazon.com/en-US/docs/alexa/ask-overviews/deprecated-features.html)
- [Proactive Events](https://developer.amazon.com/en-US/docs/alexa/smapi/proactive-events-api.html)

---

## 14. Nom du projet et repository GitHub

### Nom de marque retenu : **FamiCart**

- **Fami** (famille) + **Cart** (panier / courses).
- Court, mémorable, adapté usage familial FR/BE.
- Ne évoque pas OCR/remboursements → à préciser dans README / sous-titre.

### Convention GitHub

| Usage | Nom |
|-------|-----|
| Marque UI | `FamiCart` |
| Repository | `famicart` (minuscules, pas d'espaces) |
| Variante URL lisible | `fami-cart` (optionnel) |

### Monorepo suggéré (futur)

- `famicart` — tout en un (mobile + API)
- ou `famicart-mobile` + `famicart-api` si séparation

### README — ligne suggérée

> Application familiale de listes de courses partagées, multi-foyers, avec scan de tickets (OCR) et suivi des dépenses / remboursements.

### Alternatives évoquées (non retenues)

`foyer-courses`, `family-cart`, `pantry-family`, `multi-home-shopping`, etc.

---

## 15. Cloud Agent Cursor — changer de repository

### Fait important

- Un **Cloud Agent** est **lié à l'URL du repo** à la création.
- **On ne peut pas** changer le repo d'un agent existant de façon fiable.
- Renommer / déplacer un repo peut rendre les anciens agents **lecture seule**.

### Procédure pour travailler sur FamiCart

1. Créer le repo GitHub `famicart`.
2. **GitHub** → Applications → **Cursor** → autoriser le repo.
3. [cursor.com](https://cursor.com) → **Dashboard** → **Cloud Agents** → **nouvel environnement** → sélectionner `famicart`.
4. Configurer secrets, commande `install`, snapshot si besoin.
5. **Nouvel agent** depuis ce repo (branche `main` ou autre).
6. Optionnel : `.cursor/environment.json` + section `AGENTS.md` « Cursor Cloud specific instructions » dans le repo.

### Agent local (Cursor desktop)

- **File → Open Folder** sur le clone `famicart` — suit le workspace ouvert.

### Documentation

- [Cloud Environment Setup](https://cursor.com/docs/cloud-agent/setup)
- [Cloud Agents](https://cursor.com/docs/cloud-agent)

---

## 16. Roadmap proposée par phases

### Sprint 0 — Fondations

- Projet Ionic Angular + Supabase Auth.
- Tables `families`, `homes`, `family_members`, `home_members`, `profiles`.
- RLS de base.

### Sprint 1 — Listes (MVP)

- CRUD listes + items + Realtime coches.
- Mode courses mobile.
- Partage / invitations famille.

### Sprint 2 — Multi-foyer fille

- Sélecteur de foyer + `shopping_sessions`.
- Question « pour quel foyer ? » en début de session.
- Dashboard parents (vue foyer Belgique).

### Sprint 3 — Fichiers

- Upload photo ticket → Spaces (presigned via .NET).
- Archivage sans OCR.

### Sprint 4 — OCR

- Mistral OCR + écran correction + `receipt_lines`.
- `household_products` minimal.

### Sprint 5 — Intelligence listes

- Souvent achetés + autocomplétion.
- Rapprochement liste ↔ ticket.

### Sprint 6 — Remboursements

- `expense_claims` + workflow parent.
- Notifications (FCM puis Alexa).

### Sprint 7 — Voix

- Skill Alexa FR + Account Linking.
- Proactive Events (2–3 types).

### Sprint 8+ — Alexa+ Actions, analytics, export compta.

---

## 17. Questions ouvertes à trancher

1. **Monorepo** (`famicart`) ou repos séparés mobile / API ?
2. **Remboursement** : montant global ticket ou ligne par ligne ?
3. **Offline** : indispensable dès la v1 des courses ?
4. **Langues** : FR seul au début ? NL pour Belgique ?
5. **Auth** : magic link Supabase vs email/password ?
6. **`household_products`** : scope `home_id` vs `family_id` ?
7. **Ticket acheté au foyer parents pour la fille** : `home_id` parents + flag / claim ?
8. **Monétisation** : usage familial gratuit ; plafond OCR plus tard ?
9. **Angular** : standalone + signals dès le départ ?
10. **Certification Alexa** : priorité ou après MVP mobile ?

---

## 18. Liens et références utiles

### Cursor / Cloud Agents

- https://cursor.com/docs/cloud-agent/setup
- https://cursor.com/docs/cloud-agent

### Supabase

- https://supabase.com/docs
- Skills Cursor : Supabase, Postgres best practices (si utilisés)

### Alexa

- https://developer.amazon.com/en-US/alexa/alexa-ai
- https://developer.amazon.com/en-US/docs/alexa/ask-overviews/deprecated-features.html
- https://developer.amazon.com/en-US/docs/alexa/smapi/proactive-events-api.html
- https://developer.amazon.com/en-US/docs/alexa/smapi/schemas-for-proactive-events.html

### DigitalOcean Spaces

- API compatible S3 ; endpoint régional `*.digitaloceanspaces.com`

---

## Annexe — Parcours exemple « semaine vs week-end » (fille)

**Semaine (Belgique)**

1. Foyer actif = Appart Léa (défaut ou choix session).
2. Liste / coches / ticket → `home_id` Belgique.
3. Parents reçoivent notification claim / ticket à valider.

**Week-end (chez parents)**

1. « Je fais les courses » → choisit **Chez papa et maman** — ou ouvre liste partagée des parents (foyer imposé).
2. Ticket rattaché au foyer Parents.
3. Habitudes OCR et suggestions restent **distinctes** par foyer.

---

## Annexe — Endpoints .NET minimaux (ébauche)

| Méthode | Route | Rôle |
|---------|-------|------|
| POST | `/api/receipts/{id}/upload-url` | Presigned PUT |
| GET | `/api/receipts/{id}/download-url` | Presigned GET |
| POST | `/api/receipts/{id}/confirm-upload` | Confirme + enqueue OCR |
| POST | `/api/receipts/{id}/process-ocr` | Worker / interne |
| PATCH | `/api/receipts/{id}/lines` | Correction post-OCR |
| POST | `/api/expenses` | Soumettre claim |
| POST | `/api/expenses/{id}/approve` | Parent |
| POST | `/api/shopping-sessions` | Démarre session + `home_id` |
| * | `/api/alexa/...` | Webhook skill |

Les listes et Realtime restent en **accès direct Supabase** depuis l'app.

---

## Annexe — Intents Alexa (ébauche)

- `AddToListIntent` — item, optional quantity, optional home
- `ReadListIntent` — home / liste active
- `StartShoppingSessionIntent` — home
- `CheckOffItemIntent` — item name
- `LaunchRequest` — bienvenue + rappel foyer actif

---

*Document généré pour reprise de contexte — conversation initiale sur l'idée d'application, architecture Angular/Ionic/Supabase/.NET/Spaces, modèle famille/foyers, OCR, FamiCart, Alexa+, et configuration Cloud Agent Cursor.*
