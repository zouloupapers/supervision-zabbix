# Supervision réseau avec Zabbix et SNMPv3

> **EN** — Network monitoring lab: Zabbix 7.0 LTS (server, web UI, PostgreSQL, agent 2) deployed with Docker Compose, and Cisco IOS devices monitored over SNMPv3 (authPriv, SHA/AES-128, access restricted to the Zabbix server).

Labo de supervision : savoir en temps réel si un équipement ou une interface tombe, suivre la charge CPU et le trafic, et recevoir une alerte. Il complète le labo [lab-reseau-entreprise](https://github.com/zouloupapers/lab-reseau-entreprise).

## Architecture

```
  ┌─────────────────────── Docker Compose ───────────────────────┐
  │                                                              │
  │  zabbix-web (Nginx + PHP) :8080 ──► zabbix-server :10051     │
  │                                        │        │            │
  │                                   PostgreSQL   zabbix-agent2 │
  └────────────────────────────────────────┼─────────────────────┘
                                           │ SNMPv3 (UDP 161)
                          ┌────────────────┼────────────────┐
                       R1-EDGE          R2-CORE            SW1
```

| Service | Image | Rôle |
|---|---|---|
| `postgres` | `postgres:16-alpine` | Base de données de Zabbix |
| `zabbix-server` | `zabbix/zabbix-server-pgsql:alpine-7.0-latest` | Collecte, déclencheurs, alertes |
| `zabbix-web` | `zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest` | Interface web (port 8080) |
| `zabbix-agent` | `zabbix/zabbix-agent2:alpine-7.0-latest` | Supervision du serveur lui-même |

Zabbix 7.0 est une version **LTS** (support long terme).

## Démarrage

Prérequis : Docker et Docker Compose (Docker Desktop sous Windows).

```bash
git clone https://github.com/zouloupapers/supervision-zabbix.git
cd supervision-zabbix
cp .env.example .env        # changer POSTGRES_PASSWORD
docker compose up -d
docker compose ps           # tous les services doivent être "running"
```

Puis ouvrir **http://localhost:8080** et se connecter avec `Admin` / `zabbix`. **Changer ce mot de passe tout de suite** (*User settings → Profile*).

Dernier réglage : dans *Data collection → Hosts → Zabbix server*, remplacer l'interface agent `127.0.0.1` par le nom DNS **`zabbix-agent`** (option *Connect to: DNS*).

## Superviser un équipement Cisco en SNMPv3

1. Sur l'équipement, appliquer [`cisco/snmpv3-cisco.txt`](cisco/snmpv3-cisco.txt) après avoir remplacé l'IP du serveur et les phrases de passe.
2. Dans Zabbix : *Data collection → Hosts → Create host*
   - **Interface SNMP** : IP de gestion de l'équipement, port 161, version **SNMPv3**
   - **Security name** : `zabbix` · **Security level** : `authPriv`
   - **Authentication** : SHA1 + phrase de passe · **Privacy** : AES128 + phrase de passe
   - **Template** : `Cisco IOS by SNMP`
3. Quelques minutes plus tard, *Monitoring → Latest data* affiche CPU, mémoire, état et trafic des interfaces.

Pourquoi SNMPv3 et pas v2c : en v2c la *community* circule en clair sur le réseau. En v3 avec `authPriv`, les échanges sont authentifiés et chiffrés, et l'ACL `SNMP-ZABBIX` empêche toute autre machine d'interroger l'équipement.

## Environnement de test

Packet Tracer est un simulateur : ses équipements ne peuvent pas être interrogés par un vrai serveur Zabbix. Pour ce labo, il faut des équipements joignables depuis la machine Docker : routeurs **Cisco IOSv** dans **GNS3** ou **EVE-NG**, une VM Linux avec `snmpd`, ou le matériel de la salle de TP.

## Vérifications

| Test | Résultat attendu |
|---|---|
| `docker compose ps` | 4 services `running` |
| *Monitoring → Hosts* | Icône SNMP verte pour chaque équipement |
| Couper une interface (`shutdown`) | Problème « Interface down » dans *Monitoring → Problems* |
| `snmpwalk -v3 -l authPriv ...` depuis une autre IP | Pas de réponse (bloqué par l'ACL) |

## Feuille de route

- [ ] Captures d'écran du tableau de bord et d'une alerte (`docs/`)
- [ ] Notifications par e-mail ou Telegram
- [ ] Tableau de bord personnalisé (disponibilité, trafic WAN)
- [ ] Comparaison avec **LibreNMS** (découverte automatique, graphes)
- [ ] Réception des traps SNMP

## Auteur

**Bien De Grace KABITOU ONDONGO** — étudiant en 3ᵉ année de licence Ingénierie des réseaux et systèmes, Université Centrale (Tunis) · CCNA1 & CCNA2.
Je recherche un **stage de fin d'études du 1ᵉʳ février au 31 mars 2027**.

Licence : [MIT](LICENSE)
