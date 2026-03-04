# FireAPG v0.18 — Firewall Automatic Policy Generator

Outil **100 % client-side** (fichier HTML autonome, aucun serveur, aucune dépendance externe) pour analyser des logs de trafic réseau et générer automatiquement des règles de firewall optimisées, prêtes à exporter en CLI ou en tableur.

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

Le champ `action` doit être `accept` ou `allow`. Les logs `deny` sont ignorés.  
No-return détecté quand `sentbyte > 0` et `rcvdbyte = 0`.

---

## Object Database (ObjDB)

Permet de résoudre les CIDRs et ports en noms d'objets existants — affichage enrichi dans l'app et réutilisation des noms dans les exports CLI.

### Commandes FortiGate à coller dans l'ObjDB :
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

### Ce qui est parsé — FortiGate :
| Section | Résultat |
|---------|----------|
| `firewall address` / `address6` | Index CIDR → nom (IPv4 et IPv6) |
| `firewall service custom` | Index `proto:port` → nom |
| `system zone` | Interface → nom de zone |
| `system sdwan` members | Interface SDWAN → zone virtuelle (`virtual-wan-link`) |

### Commandes Palo Alto à coller dans l'ObjDB :
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

### Ce qui est parsé — Palo Alto :
| Section | Résultat |
|---------|----------|
| `address` | Index CIDR → nom (IPv4 et IPv6, type `ip-netmask` uniquement) |
| `service` | Index `proto:port` → nom (TCP et UDP) |

> Les objets de type `fqdn`, `ip-range`, `address-group` et `service-group` sont ignorés.  
> Les zones PA ne sont **pas** extraites de l'ObjDB — elles proviennent directement des champs `Source Zone` / `Destination Zone` des logs CSV.

---

## Convention de nommage — services

Priorité appliquée **uniformément** dans toute l'app (affichage, tooltips, exports) :

| Priorité | Source | Exemple |
|----------|--------|---------|
| 1 | Nom dans l'ObjDB chargée | `WebApp-Portal`, `Custom-DNS` |
| 2 | Builtins FortiGate connus | `HTTP`, `HTTPS`, `DNS`, `SSH`, `RDP`, `PING`… |
| 3 | Nom généré standard | `TCP-8080`, `UDP-9999`, `PING`, `IP-47` |

> Sans ObjDB, `tcp-443` est affiché comme `HTTPS (tcp-443)` si le port est connu, sinon `tcp-443` brut.

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

- **↳** FortiGate : zone résolue via ObjDB (ex: `WAN`), ou interface brute si ObjDB absente
- **↳** Palo Alto : zone du log (ex: `Internal`) + interface physique dominante (ex: `ethernet1/8.8`)
- **Adresses** : CIDR brut, ou `NomObjDB (CIDR)` si trouvé dans l'ObjDB
- **Services** : label lisible (`HTTPS (tcp-443)`, `tcp-9180`) ou nom ObjDB
- **fl.** : nombre de flows couverts par ce nœud
- **[42]** : score de permissivité 0–100

### Badges

| Badge | Signification | Tooltip au survol |
|-------|--------------|-------------------|
| `🔴 NR-TCP` | TCP sans réponse | Liste des services TCP concernés |
| `🟠 NR-UDP?` | UDP no-return sur port inhabituel | Liste des services UDP suspects |
| `🟡 NR-UDP` | UDP one-way normal (DNS, NTP, Syslog…) | Liste des services UDP normaux |
| `🔵 SNAT` / `DNAT` / `SNAT+DNAT` | NAT détecté sur au moins un flow | IPs et ports traduits |
| `⊕ Merged (N)` | Nœud issu d'un merge manuel | — |

### Actions sur les nœuds

| Action | Effet |
|--------|-------|
| Clic `▶` / `▼` | Développer / replier les enfants |
| Case à cocher | Sélectionner pour merge ou export partiel |
| `×` sur une destination | Exclure cette IP des exports (sans reconstruire) |
| `⊕ Merge (N)` | Fusionner les nœuds sélectionnés en une règle unique |
| `✕` sur un nœud mergé | Annuler le merge |

### Merge manuel
- Fusionne sources, destinations et services de N règles en une
- Les hôtes `/32` (et `/128` IPv6) sont **toujours conservés distincts** — seuls les réseaux (`/1`–`/31`) peuvent être agrégés en supernet
- Les interfaces sont toutes conservées (multi-intf dans le CLI généré)

---

## Permissivité et ancrage

### Slider (Precise → Broad)
Contrôle le seuil de regroupement des flows. Score 0–100 par nœud basé sur :
- **70 %** largeur des CIDRs sources et destinations
- **30 %** nombre de services distincts

| Score | Label | Comportement |
|-------|-------|-------------|
| 0–10 | Surgical | Règles très précises, peu de regroupement |
| 11–35 | Precise | Regroupement léger |
| 36–65 | Balanced | Compromis règles/granularité |
| 66–80 | Broad | Supernets larges |
| 81–100 | Very Broad | Agrégation maximale |

### ⚡ Auto-Optimize
Trouve automatiquement le seuil minimisant un score combiné règles × permissivité × services.

### Ancrage
- **Auto** : FireAPG choisit le meilleur ancrage selon les patterns comportementaux
- **Fix SRC** : les sources ne sont jamais agrégées (conservées en /32)
- **Fix DST** : les destinations ne sont jamais agrégées
- Les deux peuvent être cochés simultanément

### Filtre No-Return
Bouton dans la barre : exclure les flows TCP NR, ou TCP + UDP suspects, du calcul de l'arbre.

---

## Flow Lens

Explorateur de flows par dimension, en bas à droite de l'écran.

- Navigation par **Source → Destination → Service** (ordre libre, le highlight REI fonctionne quel que soit l'ordre)
- Chaque IP affiche l'interface/zone dominante :
  - FortiGate : `WAN (port1)` si ObjDB chargée, sinon `port1`
  - Palo Alto : `Internal (ethernet1/8.200)` (zone + interface physique, toujours disponible)
- Les services s'affichent avec leur label enrichi (`HTTPS (tcp-443)`)
- **Recherche** : filtre sur la valeur brute ET sur le nom ObjDB
- **Clic au dernier niveau** : met en surbrillance le nœud correspondant dans le REI

---

## Insights (panneau gauche)

Analyse automatique affichée après chargement d'une policy :

| Icône | Condition |
|-------|-----------|
| 🔴 TCP no-return | Flows TCP sans réponse (connexions refusées) |
| 🟠 UDP suspect | Flows UDP sans réponse sur ports non standards |
| 🌍 Destinations étrangères | Pays de destination détectés |
| 🧠 Clusters comportementaux | Sources partageant ≥ 60 % de patterns destination+service |
| 🚫 IPs exclues | IPs individuellement exclues via `×` dans le REI |

---

## Exports

### CLI FortiGate
```
! FireAPG v0.18 — Source policy: Policy-42
! 8 rule(s) — 01/01/2026 — rules disabled

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

### CLI Palo Alto
- Même convention de nommage que FortiGate (`TCP-80`, `UDP-53`, `PING`)
- Création d'objets : `set address "Host-10.0.0.1" ip-netmask 10.0.0.1/32`
- Multi-zone : `from [ Zone1 Zone2 ]`
- Toggle disable → `disabled yes`

### CSV
Une ligne par règle. Colonnes fixes + colonnes optionnelles selon disponibilité :
- Toujours : `Rule#`, `Type`, `No-Return`, `SRC Interface`, `DST Interface`, `Source/Dest CIDR(s)`, `Services`, `Flows`, `Perm.`
- Palo Alto ou FG avec ObjDB zone : colonnes `SRC Zone`, `DST Zone` — pour PA, interface physique en colonne Interface, zone en colonne Zone
- Si ObjDB chargée (adresses) : colonnes `Source Name(s)`, `Destination Name(s)`
- Si ObjDB chargée (services) : colonne `Service Name(s)`

### Excel (.xlsx)
M�me structure que CSV, sur deux feuilles supplémentaires :
- **Raw Flows** : tous les flows dédupliqués avec proto, port, service, interface, pays, bytes
- **No-Return Flows** : sous-ensemble des flows sans réponse

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
| **Règles existantes** | FireAPG génère de nouvelles règles ; il ne détecte pas les doublons avec des règles déjà en place sur le firewall |
| **Merge automatique** | Uniquement manuel (sélection + bouton). Pas de proposition automatique basée sur les clusters |
| **Sessions longue durée** | Les exclusions NR (`×`) et les merges sont en mémoire session uniquement, non sauvegardés |
