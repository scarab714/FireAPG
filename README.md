# FireAPG v0.19 — Firewall Automatic Policy Generator

Outil **100 % client-side** — fichier HTML autonome, aucun serveur, aucune donnée transmise — pour analyser des logs de trafic réseau et générer automatiquement des règles de firewall optimisées, prêtes à exporter en CLI ou en tableur.

---

## Vue d'ensemble

FireAPG répond à une question concrète : *"Quelles règles firewall dois-je créer pour couvrir exactement le trafic observé dans mes logs ?"*

L'approche est la suivante :

1. On **observe** ce qui passe vraiment (les logs de trafic autorisé)
2. On **regroupe** les flows qui peuvent partager une même règle
3. On **ajuste** le niveau de précision souhaité
4. On **exporte** les règles prêtes à coller dans la CLI du firewall

---

## Workflow

```
Import logs  →  (charger ObjDB)  →  Sélectionner policy  →  Ajuster seuil  →  Exporter
```

1. **Importer** un fichier de logs (FortiGate syslog ou Palo Alto CSV)
2. **Charger l'Object Database** (optionnel mais recommandé) — sortie CLI du firewall source
3. **Sélectionner une policy** dans le sélecteur gauche
4. **Ajuster le seuil de permissivité** et le mode d'ancrage
5. **Consulter les insights**, affiner dans l'arbre ou le Flow Lens
6. **Exporter** en CLI FortiGate, CLI Palo Alto, CSV ou Excel

---

## Formats de logs supportés

| Format | Détection | Champs requis |
|--------|-----------|---------------|
| **FortiGate syslog** | Automatique (`type=traffic`) | `srcip` `dstip` `proto` `dstport` `action` `policyid` `sentbyte` `rcvdbyte` |
| **Palo Alto CSV** | Automatique (en-têtes CSV) | `Source address` `Destination address` `IP Protocol` `Destination Port` `Action` `Rule` `Bytes Sent` `Bytes Received` |

Seuls les logs `accept` / `allow` sont conservés. Les `deny` sont ignorés.

**Détection no-return :** un flow est marqué "sans réponse" quand `sentbyte > 0` et `rcvdbyte = 0` — cela signifie que des données ont été envoyées mais qu'aucun octet de données applicatives n'est revenu. Cela peut indiquer une connexion refusée, un timeout, ou une asymétrie de routage.

---

## Comment fonctionne l'optimisation

C'est le cœur de l'outil. Voici la logique, étape par étape.

### 1 — Regroupement des flows par interface

Tous les flows d'une policy sont d'abord séparés par paire d'interfaces (`srcintf → dstintf`). Chaque paire constitue un groupe indépendant : on ne mélangera jamais un flux venant de `WAN` avec un flux venant de `LAN` dans la même règle, même s'ils vont au même endroit.

### 2 — Construction d'un arbre de règles candidates

Pour chaque groupe d'interfaces, FireAPG construit un **arbre hiérarchique** :

- Au niveau racine : une règle large qui couvre tous les flows du groupe (sources et destinations agrégées en supernet minimal)
- Au niveau intermédiaire : des sous-groupes basés sur des clusters comportementaux de sources
- Au niveau feuille : une règle précise par paire `(source /32, destination /32)` — soit la représentation la plus granulaire possible

```
[Niveau racine]  10.0.0.0/24 → 192.168.0.0/23  (HTTPS, DNS, TCP-8080)
    [Niveau mid]  10.0.0.1/32 → 192.168.0.0/23  (HTTPS, DNS)
        [Feuille]  10.0.0.1/32 → 192.168.0.5/32  (HTTPS)
        [Feuille]  10.0.0.1/32 → 192.168.0.12/32 (HTTPS, DNS)
    [Niveau mid]  10.0.0.2/32 → 192.168.1.0/24  (TCP-8080)
        [Feuille]  10.0.0.2/32 → 192.168.1.1/32  (TCP-8080)
```

### 3 — Score de permissivité par nœud (0–100)

Chaque nœud de l'arbre reçoit un **score de permissivité** qui mesure à quel point la règle qu'il représente est large (permissive) ou précise (chirurgicale). Ce score combine deux dimensions :

**Composante CIDR (70 % du score)**

On mesure la largeur des adresses sources et destinations. Une règle avec `10.0.0.0/8` est bien plus permissive qu'une règle avec `10.0.0.1/32`. Le score de largeur est calculé comme suit :

- Un `/32` (un seul hôte) donne un score de 0 — très précis
- Un `/0` (tout Internet) donnerait 100 — totalement permissif
- Entre les deux, la progression est linéaire par rapport au nombre de bits masqués

On fait la moyenne entre la largeur source et la largeur destination, en donnant légèrement plus de poids aux destinations (qui sont généralement les serveurs que l'on contrôle).

**Composante services (30 % du score)**

On mesure la diversité des services couverts. Une règle avec 10 services distincts est plus permissive qu'une règle avec 1 seul port. Le score monte progressivement avec le nombre de services :

- 1 service : score proche de 0
- 5+ services différents : score élevé

**Score final = 70% × score_CIDR + 30% × score_services**

### 4 — Le seuil de permissivité (le slider)

Le slider contrôle **quels nœuds de l'arbre sont affichés actifs**. Concrètement :

- Chaque nœud a un score. Le slider définit un seuil S (0 à 100).
- Un nœud est **actif** (sélectionné pour export) si son score ≤ S ET si ses enfants ne sont pas tous actifs.
- En d'autres termes : on descend dans l'arbre jusqu'au niveau où les règles passent sous le seuil.

**Exemple concret :**

| Seuil | Résultat |
|-------|----------|
| **0 (Surgical)** | Uniquement les feuilles `/32 → /32` — une règle par paire d'hôtes, très précis, beaucoup de règles |
| **50 (Balanced)** | Les nœuds intermédiaires — groupes cohérents, quelques supernets, nombre de règles réduit |
| **100 (Very Broad)** | Uniquement la racine — une seule règle large par groupe d'interfaces, très peu de règles mais très permissive |

L'idée est de trouver le seuil qui produit le **moins de règles possibles** tout en restant **suffisamment précis** pour ne pas ouvrir plus que nécessaire.

### 5 — Clustering comportemental des sources

Pour construire les nœuds intermédiaires, FireAPG utilise une heuristique comportementale : deux sources sont considérées comme "semblables" si elles parlent aux **mêmes destinations et aux mêmes services** dans une proportion suffisante (seuil : ≥ 60 % de chevauchement). Elles peuvent alors partager un nœud commun avec un supernet source.

Cela évite d'agréger des sources qui n'ont rien en commun comportementalement, même si leurs adresses IP sont proches numériquement.

### 6 — Agrégation CIDR (supernets)

Quand plusieurs sources (ou destinations) sont regroupées dans un nœud, FireAPG calcule le **supernet minimal** qui les couvre toutes. Par exemple, `10.0.0.1` et `10.0.0.2` sont couverts par `10.0.0.0/30`.

Règle importante : les adresses **hôtes `/32`** (et `/128` en IPv6) ne sont jamais absorbées dans un supernet automatiquement — elles restent toujours distinctes. Seuls les réseaux (`/1` à `/31`) peuvent être aggrégés. Cela évite d'ouvrir accidentellement l'accès à des hôtes non observés dans les logs.

### 7 — Auto-Optimize

Le bouton ⚡ **Auto-Optimize** teste automatiquement tous les seuils possibles (0 à 100) et cherche le point où la courbe `nombre de règles × permissivité moyenne × diversité de services` est minimale. Il positionne le slider sur ce point. C'est un bon point de départ mais l'analyse humaine reste indispensable pour valider le résultat.

---

## Ce que les règles générées NE font PAS

Il est important de comprendre les limites de l'approche :

- FireAPG génère des règles basées sur **ce qui a été observé** — le trafic qui a déjà passé le firewall. Il ne préjuge pas de ce qui devrait ou ne devrait pas passer.
- Les règles ne **remplacent pas** une analyse de sécurité. Elles constituent une **base de travail** à valider par un opérateur.
- Un flow présent dans les logs n'est pas nécessairement légitime — c'est à l'ingénieur de décider.
- FireAPG ne détecte pas les **doublons** avec des règles déjà en place sur le firewall.

---

## Object Database (ObjDB)

Permet de résoudre les CIDRs et ports en noms d'objets existants — affichage enrichi dans l'app et réutilisation des noms dans les exports CLI. Sans ObjDB, les règles fonctionnent mais utilisent des noms générés (`Host-10.0.0.1`, `TCP-8080`).

### Commandes FortiGate à coller dans l'ObjDB

```
show firewall address
show firewall address6
show firewall service custom
show system zone
config system sdwan
show full-configuration members
end
```

> Le prompt CLI (`FW-Home # show …`) est toléré, FireAPG l'ignore automatiquement.

### Ce qui est parsé — FortiGate

| Section | Résultat |
|---------|----------|
| `firewall address` / `address6` | Index CIDR → nom (IPv4 et IPv6) |
| `firewall service custom` | Index `proto:port` → nom |
| `system zone` | Interface → nom de zone |
| `system sdwan` members | Interface SDWAN → zone virtuelle (`virtual-wan-link`) |

### Commandes Palo Alto à coller dans l'ObjDB

```
configure
show address
show service
```

> Deux formats de sortie sont acceptés automatiquement :
> - **Format bloc** (sortie en mode `configure`) : `address { NomObjet { ip-netmask X; } }`
> - **Format `set`** (sortie opérationnelle) : `set address "NomObjet" ip-netmask X`
>
> Les deux peuvent être collés ensemble dans le même champ. FireAPG détecte le format automatiquement.

### Ce qui est parsé — Palo Alto

| Section | Résultat |
|---------|----------|
| `address` | Index CIDR → nom (IPv4 et IPv6, type `ip-netmask` uniquement) |
| `service` | Index `proto:port` → nom (TCP et UDP) |

> Les objets de type `fqdn`, `ip-range`, `address-group` et `service-group` sont ignorés.
> Les zones PA proviennent directement des champs `Source Zone` / `Destination Zone` des logs CSV — pas de l'ObjDB.

---

## Convention de nommage — services

Priorité appliquée uniformément dans toute l'app (affichage, tooltips, exports) :

| Priorité | Source | Exemple |
|----------|--------|---------|
| 1 | Nom dans l'ObjDB chargée | `WebApp-Portal`, `Custom-DNS` |
| 2 | Builtins FortiGate connus | `HTTP`, `HTTPS`, `DNS`, `SSH`, `RDP`, `PING`… |
| 3 | Nom généré standard | `TCP-8080`, `UDP-9999`, `PING`, `IP-47` |

### Convention de nommage — adresses IP

| Priorité | Source | Exemple |
|----------|--------|---------|
| 1 | Nom dans l'ObjDB | `SRV-WebProd`, `Net-DMZ` |
| 2 | Généré | `Host-10.0.0.1`, `Net-10.0.0.0_24` |

---

## Interface — Rule Expansion Interface (REI)

Arbre central de règles candidates. Chaque nœud représente une règle agrégée.

### Lecture d'un nœud

```
⊕ Merged (3)  🔴 NR-TCP  🔵 SNAT
SRC ↳ virtual-wan-link          10.0.0.1/32   10.0.0.2/32
DST ↳ Internal (ethernet1/8.8)  192.168.1.0/24
SVC  HTTPS (tcp-443), DNS (udp-53), tcp-9180
                                              1 234 fl.  [42]
```

- **↳** FortiGate : zone résolue via ObjDB (ex : `WAN`), ou interface brute si ObjDB absente
- **↳** Palo Alto : zone du log (ex : `Internal`) + interface physique dominante (ex : `ethernet1/8.8`)
- **Adresses** : CIDR brut, ou `NomObjDB (CIDR)` si trouvé dans l'ObjDB
- **Services** : label lisible (`HTTPS (tcp-443)`) ou nom ObjDB
- **fl.** : nombre de flows couverts par ce nœud
- **[42]** : score de permissivité 0–100

### Badges

| Badge | Signification | Tooltip au survol |
|-------|--------------|-------------------|
| `🔴 NR-TCP` | TCP sans réponse (connexion refusée ou timeout) | Liste des services TCP concernés |
| `🟠 NR-UDP?` | UDP no-return sur port inhabituel | Liste des services UDP suspects |
| `🟡 NR-UDP` | UDP one-way normal (DNS, NTP, Syslog…) | Liste des services UDP normaux |
| `🔵 SNAT` / `DNAT` / `SNAT+DNAT` | NAT détecté sur au moins un flow | IPs et ports traduits |
| `⊕ Merged (N)` | Nœud issu d'un merge manuel | — |
| `✓ Traité` | Règle marquée comme traitée | Note associée si présente |
| `⚠️ Sans destination` | Toutes les destinations ont été exclues via `×` | — |
| `⚠️ Sans source` | Toutes les sources ont été exclues | — |

### Actions sur les nœuds

| Action | Effet |
|--------|-------|
| Clic `▶` / `▼` | Développer / replier les enfants |
| Case à cocher | Sélectionner pour merge, traitement ou export partiel |
| `×` sur une destination | Exclure cette IP des exports (sans reconstruire l'arbre) |
| `⊕ Merge (N)` | Fusionner les nœuds sélectionnés en une règle unique |
| `✕` sur un nœud mergé | Annuler le merge |
| `✓ Traité (N)` | Marquer les nœuds sélectionnés comme traités (popup note) |
| `✓ Traité` (badge) | Cliquer sur le badge pour démarquer |
| `↩ Rétablir` | Réactiver les exclusions d'un nœud sans source/destination |

### Merge manuel

- Fusionne sources, destinations et services de N règles en une seule
- Les hôtes `/32` (et `/128` IPv6) sont **toujours conservés distincts** — seuls les réseaux peuvent être agrégés en supernet
- Les interfaces sont toutes conservées (multi-intf dans le CLI généré)

### Règles sans source ou destination valide

Si toutes les destinations d'une règle ont été exclues via `×` (ou toutes les sources), la règle est signalée en orange `⚠️ Sans destination`. Elle n'apparaît pas dans les exports. Le bouton `↩ Rétablir` réactive les exclusions de ce nœud.

---

## Marquage "Traité"

Permet de suivre l'avancement de la revue des règles générées.

- Sélectionner une ou plusieurs règles → bouton `✓ Traité (N)` apparaît dans la barre
- Une popup optionnelle permet d'ajouter une note (ex : `"Validé — monitoring interne"`, `"À bloquer — flux non autorisé"`)
- Les règles traitées s'affichent en vert atténué avec un badge `✓ Traité`
- Le compteur `✓ X / Y traités` apparaît dans la barre du haut
- Les règles traitées sont **exclues des exports par défaut** — toggle `✓ Inclure traités` dans chaque onglet d'export pour les réintégrer

Le marquage est basé sur le **contenu de la règle** (sources + destinations + services) et non sur un identifiant technique. Il survit donc aux changements de seuil de permissivité.

---

## Permissivité et ancrage

### Slider (Precise → Broad)

Positionne le seuil d'activation des nœuds dans l'arbre. Un nœud est actif si son score de permissivité est inférieur ou égal au seuil. Voir la section "Comment fonctionne l'optimisation" pour le détail du calcul.

| Score | Label | Comportement |
|-------|-------|-------------|
| 0–10 | Surgical | Règles très précises, peu de regroupement |
| 11–35 | Precise | Regroupement léger |
| 36–65 | Balanced | Compromis règles / granularité |
| 66–80 | Broad | Supernets larges |
| 81–100 | Very Broad | Agrégation maximale |

### ⚡ Auto-Optimize

Teste tous les seuils de 0 à 100 et sélectionne automatiquement celui qui minimise un score combiné `nombre de règles × permissivité moyenne × diversité de services`. Bon point de départ pour une première analyse.

### Ancrage

Contrôle quelle dimension (source ou destination) peut être agrégée en supernet.

- **Auto** : FireAPG choisit le meilleur ancrage selon les patterns comportementaux des flows
- **Fix SRC** : les sources ne sont jamais agrégées — conservées en `/32` — utile quand les sources sont les clients que l'on veut contrôler finement
- **Fix DST** : les destinations ne sont jamais agrégées — utile quand les destinations sont des serveurs connus et précis
- Les deux peuvent être cochés simultanément (aucune agrégation)

### Filtre No-Return

Boutons dans la barre principale : exclure les flows TCP no-return, ou TCP + UDP suspects, du calcul de l'arbre. Les flows `🟡 NR-UDP` (ports one-way connus : DNS, NTP, Syslog…) ne sont jamais exclus.

---

## Flow Lens

Explorateur de flows par dimension, accessible via le bouton `🔍 Flow Lens`.

- Navigation libre par **Source → Destination → Service** (dans n'importe quel ordre)
- Chaque IP affiche l'interface et la zone dominantes :
  - FortiGate : `WAN (port1)` si ObjDB chargée, sinon `port1`
  - Palo Alto : `Internal (ethernet1/8.200)` (zone + interface physique, toujours disponible)
- Les services s'affichent avec leur label enrichi (`HTTPS (tcp-443)`)
- **Recherche** : filtre sur la valeur brute et sur le nom ObjDB
- **Clic au dernier niveau** : met en surbrillance le nœud correspondant dans le REI

---

## Insights (panneau gauche)

Analyse automatique affichée après chargement d'une policy.

| Icône | Condition |
|-------|-----------|
| 🔴 TCP no-return | Flows TCP sans réponse (connexions refusées ou timeout) |
| 🟠 UDP suspect | Flows UDP sans réponse sur ports non standards |
| 🌍 Destinations étrangères | Pays de destination détectés |
| 🧠 Clusters comportementaux | Sources partageant ≥ 60 % de patterns destination+service |
| 🚫 IPs exclues | IPs individuellement exclues via `×` dans le REI |

---

## Exports

### CLI FortiGate

```
! ==============================================================
! FireAPG v0.19 — Source policy: Policy-42
! 8 rule(s) — 01/01/2026
! ==============================================================

config firewall address
edit "Host-10.0.0.5"
  set subnet 10.0.0.5 255.255.255.255
next
end

config firewall service custom
edit "TCP-9180"
  set protocol TCP
  set tcp-portrange 9180
next
end

config firewall policy
edit 0
  set name "OPT-ID42-001"
  set srcintf "virtual-wan-link"
  set dstintf "dmz"
  set srcaddr "Host-10.0.0.5"
  set dstaddr "Net-192.168.1.0_24"
  set service "HTTPS" "TCP-9180"
  set action accept
  set status disable
next
end
```

- Objets adresse IPv4 et IPv6 créés si absents de l'ObjDB
- Objets service créés si absents (`TCP-9180`, `UDP-51822`…)
- Multi-interface : `set srcintf "Vlan10" "Vlan20"`
- Préfixe de règle personnalisable (défaut : `OPT`)
- Toggle `🔴 Add disabled` → toutes les règles générées avec `set status disable`
- Toggle `✓ Inclure traités` → les règles marquées traitées sont commentées avec `! [TRAITÉ] Note: "..."`

### CLI Palo Alto

- Même convention de nommage que FortiGate (`TCP-80`, `UDP-53`, `PING`)
- Création d'objets : `set address "Host-10.0.0.1" ip-netmask 10.0.0.1/32`
- Multi-zone : `from [ Zone1 Zone2 ]`
- Toggle disable → `disabled yes`
- Toggle `✓ Inclure traités` → règles commentées avec `# [TRAITÉ] Note: "..."`

### CSV

Une ligne par règle. Colonnes fixes + colonnes optionnelles selon disponibilité :

- **Toujours présentes :** `Rule#`, `Statut`, `Note`, `Type`, `No-Return`, `SRC Interface`, `DST Interface`, `Source CIDR(s)`, `Destination CIDR(s)`, `Services`, `Flows`, `Perm.`, `Level`, `Source`
- **Si zones disponibles** (Palo Alto ou FG avec ObjDB zone) : `SRC Zone`, `DST Zone`
- **Si ObjDB adresses chargée** : `Source Name(s)`, `Destination Name(s)`
- **Si ObjDB services chargée** : `Service Name(s)`

La colonne `Statut` vaut `TRAITÉ` pour les règles marquées (si le toggle est activé). La colonne `Note` contient la note saisie lors du marquage.

### Excel (.xlsx)

Même structure que CSV sur la feuille principale `FireAPG Rules`, plus deux feuilles supplémentaires :

- **Raw Flows** : tous les flows dédupliqués avec proto, port, service, interface, pays, bytes
- **No-Return Flows** : sous-ensemble des flows sans réponse

Les lignes traitées apparaissent en gris avec texte barré (si le toggle est activé).

---

## Thème

Le bouton `🌓` en haut à droite permet de cycler entre trois modes :

| Icône | Mode |
|-------|------|
| `🌓` | Auto — suit le thème système (clair le jour, sombre la nuit) |
| `🌙` | Sombre forcé |
| `☀` | Clair forcé |

---

## Données et confidentialité

FireAPG fonctionne **entièrement dans le navigateur**. Aucune donnée n'est transmise à un serveur. Le fichier HTML est autonome et utilisable hors-ligne.

---

## Limites connues

| Point | État |
|-------|------|
| **IPv6** | Parsé et affiché. Agrégation CIDR IPv6 basique (supernet simple, pas de résumé hiérarchique avancé) |
| **NAT Palo Alto** | Colonnes NAT non standardisées selon les versions PAN-OS — peut ne pas parser selon l'export |
| **Groupes d'objets** | L'ObjDB ne parse pas les `address-group` ni les `service-group` — objets individuels uniquement |
| **FQDN / Geography / IP-range** | Ces types d'objets ne sont pas résolus (FG FQDN/geography, PA fqdn/ip-range) |
| **Logs PA syslog/CEF** | Seul le format CSV export Palo Alto est supporté |
| **Multi-device PA** | Un avertissement est affiché si les logs proviennent de plusieurs devices ; les règles ne sont pas séparées par device |
| **Règles existantes** | FireAPG génère de nouvelles règles — il ne détecte pas les doublons avec des règles déjà en place |
| **Merge automatique** | Uniquement manuel (sélection + bouton). Pas de proposition automatique basée sur les clusters |
| **Persistance de session** | Les marquages "Traité", exclusions `×` et merges sont en mémoire session uniquement — perdus à la fermeture du fichier |
