# FireAPG — Cahier des charges · Mise en production SaaS

> **Document de référence** pour l'implémentation de la couche auth, licence et déploiement de FireAPG.
> Rédigé en phase de réflexion — à soumettre tel quel à Claude au moment de l'implémentation.

---

## Contexte

FireAPG est une application d'analyse de logs firewall et de génération automatique de règles (FortiGate et Palo Alto). Elle existe aujourd'hui sous forme d'un fichier HTML unique, 100 % client-side — tout le traitement se fait dans le navigateur, aucune donnée ne transite par un serveur.

L'objectif est de transformer cette app en service commercial tout en **conservant la promesse de confidentialité** : les logs firewall des clients ne doivent jamais quitter leur navigateur ni transiter par le serveur.

---

## Ce qui doit être implémenté

### 1. Authentification utilisateur

Page de login hébergée sur le domaine principal. L'utilisateur s'authentifie avant d'accéder à l'app.

**Fonctionnalités requises :**
- Inscription par email + mot de passe
- Login / logout
- Réinitialisation de mot de passe par email
- Optionnel : 2FA (TOTP)
- Optionnel : login via Google / GitHub (OAuth)

**Solutions candidates — à évaluer au moment de l'implémentation :**

| Option | Type | Notes |
|--------|------|-------|
| **Better Auth** | Bibliothèque JS open source | Recommandé — moderne, léger, bien documenté, intégration simple |
| **Lucia** | Bibliothèque JS open source | Alternative minimaliste, contrôle total, plus de code manuel |
| **Keycloak** | Service open source à héberger | Très complet, adapté si on prévoit une gestion multi-tenant ou enterprise |
| **Authentik** | Service open source à héberger | Bonne alternative à Keycloak, interface admin plus moderne |

> **Note au moment de l'implémentation :** proposer la solution la plus adaptée à l'écosystème JS en 2025+, en tenant compte des évolutions depuis la rédaction de ce document.

---

### 2. Gestion des licences et abonnements

Après authentification, le serveur vérifie si l'utilisateur a un abonnement actif avant de lui donner accès à l'app.

**Modèle de licence envisagé :**
- Abonnement mensuel ou annuel
- Période d'essai gratuite (ex : 14 jours)
- Accès bloqué immédiatement à expiration

**Structure de données minimale :**

```
users
├── id
├── email
├── password_hash
├── created_at
└── subscription_status  (trialing / active / expired / cancelled)

subscriptions
├── id
├── user_id
├── plan          (monthly / annual)
├── started_at
├── expires_at
└── stripe_subscription_id
```

**Paiement :**

| Option | Notes |
|--------|-------|
| **Stripe** | Standard universel, webhooks simples, documentation excellente |
| **LemonSqueezy** | Alternative Stripe, plus simple pour les indépendants, gère la TVA automatiquement |
| **Paddle** | Similaire à LemonSqueezy, bon pour les marchés européens |

> **Note au moment de l'implémentation :** vérifier les frais de commission et la simplicité d'intégration de chaque option à cette date.

---

### 3. Accès à l'app via token volatile (JWT)

Une fois authentifié et la licence vérifiée, le serveur génère un **token JWT signé** avec une durée de vie courte (recommandé : 1 heure). L'app FireAPG est accessible uniquement via une URL contenant ce token.

**Flux :**

```
Login réussi + licence active
        │
        ▼
Serveur génère JWT signé (exp: +1h)
        │
        ▼
Redirect vers /app?token=eyJhbGc...
        │
        ▼
App FireAPG charge, vérifie le token localement
(pas de second appel serveur — vérification cryptographique)
        │
        ├── Token valide ──▶ App démarre normalement
        └── Token invalide/expiré ──▶ Retour vers login
```

**Propriétés du token :**
- Signé avec clé secrète côté serveur (HS256 minimum, RS256 recommandé)
- Contient : user_id, email, expiration, plan
- Usage unique ou lié à la session (à définir)
- Token précédent invalidé à chaque nouvelle connexion

---

### 4. Obfuscation du code source

Le fichier FireAPG.html distribué aux clients doit être obfusqué pour protéger la propriété intellectuelle (algorithme de permissivité, parseurs, logique de génération CLI).

**Outil recommandé :** `javascript-obfuscator` (open source, npm)

**Script de build à créer :**

```bash
# build.sh — à lancer avant chaque déploiement
javascript-obfuscator FireAPG.html \
  --output dist/fireapg.html \
  --compact true \
  --string-array true \
  --rotate-string-array true \
  --self-defending true
```

**Workflow de développement :**
- On travaille toujours sur `FireAPG.html` (source, non versionné publiquement)
- `dist/fireapg.html` est le fichier déployé (jamais modifié à la main)
- Le build est lancé manuellement avant chaque déploiement

---

### 5. Architecture Docker

L'ensemble de l'infrastructure doit être conteneurisé pour simplifier la maintenance et le déploiement sur les serveurs existants.

**Services à conteneuriser :**

```
docker-compose.yml
├── frontend        — Page login + page marketing (Nginx)
├── backend         — API auth + licence + JWT (Node.js)
├── database        — Base de données (SQLite ou PostgreSQL)
└── reverse-proxy   — Entrée HTTPS, routing (Traefik ou Nginx)
```

**Détail des services :**

**frontend** — Nginx servant les fichiers statiques : page marketing, page login, et l'app FireAPG obfusquée. Pas de logique serveur.

**backend** — API Node.js exposant :
- `POST /auth/login` — vérifie credentials, retourne JWT
- `POST /auth/register` — création de compte
- `POST /auth/logout` — invalidation de session
- `GET /auth/me` — vérifie session active
- `POST /webhooks/stripe` — reçoit les événements de paiement
- `GET /licence/check` — vérifie si l'abonnement est actif

**database :**

| Option | Notes |
|--------|-------|
| **SQLite** | Zéro configuration, fichier unique, largement suffisant jusqu'à ~1 000 utilisateurs actifs |
| **PostgreSQL** | Recommandé si on prévoit une croissance significative ou des besoins de requêtes complexes |

> **Note au moment de l'implémentation :** choisir en fonction du volume d'utilisateurs anticipé.

**reverse-proxy :**

| Option | Notes |
|--------|-------|
| **Traefik** | S'intègre nativement avec Docker, gestion automatique des certificats Let's Encrypt, configuration par labels |
| **Nginx Proxy Manager** | Interface graphique, plus simple à configurer manuellement |
| **Caddy** | Très simple, HTTPS automatique, bonne alternative légère |

---

### 6. Structure des URLs

```
fireapg.com/              ← Page marketing
fireapg.com/login         ← Authentification
fireapg.com/register      ← Inscription
fireapg.com/pricing       ← Tarifs + lien d'achat
fireapg.com/dashboard     ← Tableau de bord utilisateur (abonnement, factures)
fireapg.com/app           ← Redirige vers login si pas de session
fireapg.com/app?token=XXX ← App FireAPG (accès après auth)
```

---

## Prérequis techniques

### Serveur existant
- Docker + Docker Compose installés
- Port 80 et 443 ouverts
- Nom de domaine pointant vers le serveur (A record configuré)

### Comptes externes à créer
- **Stripe** (ou alternative) — pour la gestion des paiements
- **Fournisseur email transactionnel** — pour les emails d'inscription et reset password

Options email :

| Option | Notes |
|--------|-------|
| **Resend** | Simple, generous free tier, API moderne |
| **Brevo (ex-Sendinblue)** | Français, RGPD natif, free tier généreux |
| **Postmark** | Excellente délivrabilité, orienté transactionnel |

### Environnement de développement local
- Node.js 18+ installé
- npm ou pnpm
- Docker Desktop (pour tester localement avant déploiement)

---

## Ce qui est hors scope (pour cette phase)

- Interface d'administration (gestion des utilisateurs, stats d'usage) — peut être ajouté en v2
- Sauvegarde de session FireAPG (états, merges, marquages "Traité") — prévu en v2
- Multi-tenancy (équipes, partage de sessions entre collègues) — v3
- API publique — non envisagé

---

## Prompt d'implémentation

> À copier-coller tel quel dans une nouvelle conversation Claude au moment de l'implémentation.

---

```
Je veux implémenter la couche SaaS de FireAPG, mon application d'analyse
de logs firewall et de génération de règles (FortiGate / Palo Alto).

L'app existe aujourd'hui sous forme d'un fichier HTML unique, 100% client-side.
Je veux ajouter : authentification, gestion de licences/abonnements,
accès via token JWT volatile, obfuscation du code, et déploiement Docker.

Contrainte fondamentale : les logs firewall des clients ne doivent JAMAIS
transiter par mon serveur. Tout le traitement reste dans le navigateur.
Le serveur gère uniquement l'auth et la vérification de licence.

Voici le cahier des charges complet : [coller le contenu de ce document]

Avant de commencer à coder, j'aimerais que tu :

1. Proposes la stack technique que tu recommandes aujourd'hui pour ce projet,
   en tenant compte des évolutions depuis la rédaction de ce document (date : mars 2026).
   Pour chaque composant (auth, paiement, base de données, reverse proxy),
   indique si ta recommandation diffère de celle du cahier des charges et pourquoi.

2. Estimes le nombre d'étapes et l'ordre d'implémentation recommandé.

3. Confirmes que tu peux me guider pas à pas, y compris pour les actions
   de configuration externe (DNS, Stripe, email) en me dictant exactement
   quoi faire à chaque étape.

Une fois que tu as répondu à ces trois points et que je valide,
on commence l'implémentation dans l'ordre que tu proposes.

Contexte serveur : j'ai mes propres serveurs, Docker + Docker Compose
sont déjà installés. Je ne suis pas développeur mais je suis à l'aise
avec un terminal et le copier-coller de commandes.
```

---

*Document rédigé en mars 2026 — FireAPG v0.19*
