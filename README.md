# FireAPG v0.16 — Automatic Policy Generator

FireAPG est un outil **100% client-side** (HTML/JS, aucun serveur) qui analyse des logs de trafic réseau FortiGate ou Palo Alto et génère automatiquement des règles de firewall optimisées, prêtes à exporter en CLI.

---

## Fonctionnement général

1. **Importer un fichier de logs** (FortiGate syslog ou Palo Alto CSV)
2. **Sélectionner une policy** (règle source) dans le sélecteur à gauche
3. **Régler le seuil de permissivité** (slider) — plus il est haut, moins de règles générées mais plus larges
4. **Consulter l'arbre de règles** (Rule Expansion Interface) — visualiser, ajuster, merger
5. **Exporter en CLI** — FortiGate ou Palo Alto, copier-coller directement dans le firewall

---

## Sources de logs supportées

| Format | Détection | Champs requis |
|--------|-----------|---------------|
| FortiGate syslog | Automatique (champ `type=traffic`) | `srcip`, `dstip`, `proto`, `dstport`, `action`, `policyid`, `sentbyte`, `rcvdbyte` |
| Palo Alto CSV | Automatique (en-têtes CSV) | `Source address`, `Destination address`, `IP Protocol`, `Destination Port`, `Action`, `Rule`, `Bytes Sent`, `Bytes Received` |

### NAT (FortiGate)
Les champs `trandisp`, `transip`, `transport`, `tranip`, `tranport` sont parsés automatiquement. Les flux NAT reçoivent un badge `🔵 SNAT`, `🔵 DNAT` ou `🔵 SNAT+DNAT` dans l'arbre.

### NAT (Palo Alto)
Colonnes `NAT Type`, `NAT Source IP`, `NAT Source Port`, `NAT Destination IP`, `NAT Destination Port` si présentes dans l'export CSV.

---

## Object Database (ObjDB)

Permet à FireAPG d'utiliser les noms d'objets existants plutôt que d'en créer de nouveaux lors de l'export.

### Commandes FortiGate à exécuter et coller dans l'ObjDB :

```
show firewall address
show firewall address6
show firewall service custom
show system zone
conf system sdwan
show full-configuration members
end
```

> Le prompt CLI (`FW-Home #`) est toléré — FireAPG parse les blocs même avec les lignes de prompt intercalaires.

### Contenu parsé :
- **Objets adresse IPv4/IPv6** → résolution CIDR → nom dans les exports et l'affichage
- **Objets service** → résolution proto:port → nom dans les exports et l'affichage
- **Zones** (system zone + sdwan members) → interface → nom de zone affiché dans le REI et le Flow Lens

### Convention de nommage (export CLI) :
| Priorité | Source | Exemple |
|----------|--------|---------|
| 1 | ObjDB chargée | `MyWebServer`, `Custom-HTTP-8080` |
| 2 | Builtin FG/PA | `HTTP`, `HTTPS`, `PING`, `SSH`, `RDP`… |
| 3 | Généré | `TCP-8080`, `UDP-9999`, `Host-10.0.0.1`, `Net-10.0.0.0_24` |

---

## Rule Expansion Interface (REI)

L'arbre de règles au centre. Chaque nœud représente une règle candidate.

### Badges d'état :

| Badge | Signification | Tooltip |
|-------|--------------|---------|
| `🔴 NR-TCP` | TCP sans réponse (connexion refusée/bloquée) | Liste des services TCP concernés |
| `🟠 NR-UDP?` | UDP sans réponse sur port inhabituel | Liste des services UDP suspects |
| `🟡 NR-UDP` | UDP one-way normal (DNS, NTP, Syslog…) | Liste des services UDP normaux |
| `🔵 SNAT` / `DNAT` / `SNAT+DNAT` | Flow avec NAT détecté | IPs et ports traduits |
| `⊕ Merged (N)` | Nœud résultant d'un merge manuel | Cliquer ✕ pour annuler |

### Affichage dans le nœud :
- **`SRC` ↳ zone/intf** → adresses sources avec interface/zone si ObjDB chargée
- **`DST` ↳ zone/intf** → adresses destinations
- **`SVC`** → services, format `NomObjDB (tcp-80)` si connu, sinon `HTTP (tcp-80)` si builtin, sinon `tcp-80`

### Actions sur les nœuds :
- **Clic sur le toggle** `▶/▼` → développer/replier
- **Case à cocher** → sélectionner pour export ou merge
- **`×` sur une destination NR** → exclure cette IP de l'export (sans reconstruire l'arbre)
- **`⊕ Merge (N)`** → fusionner N nœuds sélectionnés en une seule règle agrégée (avec undo)

---

## Flow Lens

Explorateur de flux interactif en bas à droite. Permet de naviguer par dimension (Source → Destination → Service) pour comprendre qui parle à qui.

- **Recherche** : filtre sur valeur brute ET nom ObjDB
- **Badge NR** visible sur chaque entrée concernée
- **Interface/Zone** affichée si ObjDB chargée : `virtual-wan-link (wan)`
- **Clic au dernier niveau** : met en évidence le nœud correspondant dans le REI (fonctionne quel que soit l'ordre de navigation Source/Destination/Service)

---

## Export CLI

### FortiGate
- Objets adresse créés si absents de l'ObjDB
- Objets service créés si absents (TCP-80, UDP-53, IP-47…)
- Politique : `set srcintf`, `set dstintf`, `set srcaddr`, `set dstaddr`, `set service`
- Multi-interface après merge : `set srcintf "Vlan10" "Vlan20"`
- Toggle `🔴 Add disabled` → ajoute `set status disable` sur toutes les règles

### Palo Alto
- Même convention de nommage que FortiGate (`TCP-80`, `UDP-53`, `PING`…)
- Multi-interface : `from [ Zone1 Zone2 ]`
- Toggle disable → `disabled yes`

### Préfixe de règle
Personnalisable (`OPT` par défaut) → génère `OPT-ID42-001`, `OPT-ID42-002`…

---

## Insights (panneau gauche)

Analyse automatique des anomalies :
- Flux TCP no-return (connexions refusées)
- Flux UDP suspects
- Destinations vers des pays étrangers
- Clusters comportementaux (sources avec patterns similaires)

---

## Limites et points non finalisés

### Limites connues

| Point | État |
|-------|------|
| **IPv6** | Parsé et affiché, mais agrégation CIDR IPv6 est basique (pas de supernet calculation avancée) |
| **NAT Palo Alto** | Colonnes NAT non standardisées selon les versions PAN-OS — peut ne pas parser selon le format d'export |
| **Merge automatique** | Pas implémenté — uniquement merge manuel avec sélection checkbox |
| **Règles existantes** | FireAPG génère de nouvelles règles, il ne détecte pas les doublons avec des règles déjà en place |
| **Groupes d'objets** | L'ObjDB ne parse pas les `address-group` / `service-group` — seulement les objets individuels |
| **FG : adresses FQDN/geography** | Les objets FQDN et geography dans l'ObjDB ne sont pas résolus (marqués comme indeterminate) |
| **PA : multi-device** | Un avertissement est affiché si les logs viennent de plusieurs devices, mais les règles ne sont pas séparées par device |
| **Logs PA syslog (CEF)** | Seul le format CSV export est supporté pour Palo Alto. Le format syslog CEF n'est pas parsé |
| **Durée de session** | Le champ `duration` est parsé mais non utilisé dans la logique de permissivité |
| **Export Excel** | Généré correctement mais les filtres Excel automatiques ne sont pas posés |

### Éléments partiellement implémentés

- **Toggle "Obj Names"** : était un on/off pour l'affichage ObjDB — maintenant les noms s'affichent toujours si disponibles. Le toggle reste visible mais ne contrôle plus l'affichage des noms (peut être refondu ou supprimé)
- **Analyse comportementale** : les clusters sont détectés (badge Insights) mais ne sont pas utilisés pour proposer des merges automatiques
- **No-return exclusion** : l'exclusion `×` d'une IP NR est visuelle uniquement — elle persiste dans la session mais n'est pas sauvegardée

---

## Données et confidentialité

FireAPG fonctionne **entièrement dans le navigateur**. Aucune donnée n'est envoyée à un serveur. Aucune dépendance externe (CDN, API). Le fichier HTML est auto-contenu et peut être utilisé hors-ligne.

