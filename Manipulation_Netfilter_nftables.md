# Laboratoire — Sécurité des réseaux : Netfilter / nftables

Récapitulatif complet de la manipulation : mise en place d'une topologie **LAN – WAN – DMZ** avec un pare-feu GNU/Linux configuré avec **nftables**, conformément à une politique de sécurité réseau.

> **Hyperviseur utilisé** : VirtualBox · **OS** : Debian 13 (Trixie)

---

## 1. Objectifs

- Monter une topologie d'entreprise classique (LAN, WAN, DMZ).
- Configurer un Linux en **routeur / pare-feu** avec nftables.
- Mettre en place des règles de **filtrage**, de **NAT** (SNAT + DNAT), de **suivi de connexion** et de **logging**.
- Respecter une politique en **default-deny** (tout est bloqué par défaut, on ouvre au cas par cas).

---

## 2. Topologie

```
                          Internet
                             |
                  [ Client externe ]  <-- WAN (carte en Réseau NAT/pont)
                             |
                       enp0s3 (WAN)
                             |
   [ Client LAN ] --- enp0s8 (LAN) [ FIREWALL ] enp0s9 (DMZ) --- [ Serveur Web/SSH ]
        192.168.1.10            nftables               172.16.0.10
```

- **LAN** et **DMZ** sont sur deux switchs virtuels (réseaux internes) **distincts** : tout trafic entre zones est obligé de passer par le pare-feu.
- Le WAN donne l'accès Internet réel.

---

## 3. Machines virtuelles & cartes réseau

| VM | Rôle | OS | Cartes (VirtualBox) | Interface |
|----|------|----|--------------------|-----------|
| **Firewall** | Routeur / pare-feu | Debian 13 (netinst) | Carte 1 : **Réseau NAT** ou **Pont** (WAN) | `enp0s3` |
| | | | Carte 2 : **Réseau interne** `lan` | `enp0s8` |
| | | | Carte 3 : **Réseau interne** `dmz` | `enp0s9` |
| **Serveur DMZ** | Web (HTTP) + SSH | Debian 13 | 1 carte : **Réseau interne** `dmz` | `enp0s3` |
| **Client LAN** | Poste interne de confiance | Debian 13 | 1 carte : **Réseau interne** `lan` | `enp0s3` |
| **Client externe** | Utilisateur Internet | Au choix | 1 carte : **Réseau NAT** ou **Pont** | — |

**Activation du routage sur le firewall** (obligatoire, sinon rien ne traverse) :

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
cat /proc/sys/net/ipv4/ip_forward    # doit afficher 1
```

**Services à installer sur le serveur DMZ** (avant d'appliquer la politique de filtrage) :

```bash
sudo apt update
sudo apt install -y apache2 openssh-server
```

---

## 4. Adressage IP

| Réseau | Sous-réseau | Machine | Adresse | Passerelle | DNS |
|--------|-------------|---------|---------|------------|-----|
| **WAN** | (DHCP) | Firewall `enp0s3` | DHCP | auto | — |
| | | Client externe | DHCP | auto | — |
| **LAN** | 192.168.1.0/24 | Firewall `enp0s8` | `192.168.1.254/24` | — | — |
| | | Client LAN | `192.168.1.10/24` | `192.168.1.254` | `208.67.222.123` |
| **DMZ** | 172.16.0.0/16 | Firewall `enp0s9` | `172.16.0.245/16` | — | — |
| | | Serveur DMZ | `172.16.0.10/16` | `172.16.0.245` | `208.67.222.123` |

> Le serveur DNS `208.67.222.123` est **OpenDNS FamilyShield**, qui bloque certains sites malicieux (ex : `internetbadguys.com`).

**Exemple — `/etc/network/interfaces` du firewall :**

```
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
    address 192.168.1.254/24

auto enp0s9
iface enp0s9 inet static
    address 172.16.0.245/16
```

**Exemple — `/etc/network/interfaces` du serveur DMZ :**

```
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 172.16.0.10/16
    gateway 172.16.0.245
```

**Client LAN** — DNS forcé directement dans `/etc/resolv.conf` :

```bash
echo 'nameserver 208.67.222.123' | sudo tee /etc/resolv.conf
```

---

## 5. ⚠️ Pièges à éviter (vécu)

Ces points sont la cause des principaux blocages rencontrés. À relire avant de débugger pendant des heures.

| # | Piège | Symptôme | Solution |
|---|-------|----------|----------|
| 1 | **`policy accept` au lieu de `policy drop`** | Le pare-feu laisse tout passer (ex : ping LAN→DMZ marche alors qu'il ne devrait pas) | Mettre `policy drop` sur les 3 chaînes de filtrage (`input`, `forward`, `output`). **Ne pas** toucher aux chaînes `nat` (leur `policy accept` est normal). |
| 2 | **`/etc/sysctl.conf` inexistant** | Pas de fichier à éditer pour le forwarding | Utiliser un drop-in : `/etc/sysctl.d/99-ipforward.conf`. |
| 3 | **`dns-nameserver` (singulier)** dans `interfaces` | DNS non configuré | Le mot-clé est `dns-nameservers` (**pluriel**) **et** nécessite le paquet `resolvconf`. Le plus simple : éditer `/etc/resolv.conf` directement. |
| 4 | **`name-server` (avec tiret)** dans `resolv.conf` | « no servers could be reached », fallback sur `::1` / `127.0.0.1` | Le mot-clé est `nameserver` (**un seul mot, sans tiret**). |
| 5 | **Passerelle du serveur DMZ cassée** (`169.254.x` / pas de `gateway`) | Le TCP se connecte (`connected`) mais la **réponse HTTP ne revient jamais** (« awaiting response… ») | Définir `gateway 172.16.0.245` (= IP réelle du firewall côté DMZ). Cohérence indispensable : `.245` ≠ `.254`. |
| 6 | **DNAT : mauvais critère dans `forward`** | Le DNAT 61337→22 « ne marche pas » | Le DNAT (`prerouting`) s'exécute **avant** `forward` : la destination y est **déjà traduite**. Filtrer sur `ip daddr 172.16.0.10 tcp dport 22` (le port **22**, pas 61337). |
| 7 | **SSH absent du firewall + `output` en drop** | `apt` ne peut rien télécharger | Ouvrir temporairement la sortie : `sudo nft insert rule inet filter output accept`, installer, puis recharger la config (le `flush ruleset` enlève la règle temporaire). |
| 8 | **Tester la limite SSH depuis le firewall lui-même** | La limitation « ne fonctionne pas » | Une connexion vers sa propre IP passe par `lo` (capturée par `iif "lo" accept`). **Tester depuis le client LAN.** |
| 9 | **Set `blocked_IPs` : bloquer sa propre plage WAN** | Perte d'Internet / du client externe | Ne **jamais** bloquer la plage où se trouve le WAN. À l'école (WAN `10.x`) → exclure `10.0.0.0/8`. À la maison (WAN `192.168.68.x`) → exclure `192.168.0.0/16`. |

---

## 6. Concepts clés

- **`reject` vs `drop`** :
  - **LAN → WAN** (utilisateurs de confiance) → `reject with tcp reset` : réponse immédiate (« connection refused »), pas de navigateur qui rame.
  - **WAN → ANY** (Internet hostile) → `drop` : silence radio, le pare-feu reste furtif et ne révèle rien à un attaquant.
- **Suivi de connexion (conntrack)** : `ct state established,related accept` autorise automatiquement le trafic retour d'une session déjà ouverte → pas besoin de règles « entrantes » dangereuses.
- **Logging SSH** : un `drop` provoque des **retransmissions du SYN** → plusieurs logs par tentative. Un `reject with tcp reset` ferme net → **1 seul log** par tentative. Le flag TCP de ces paquets est **`SYN`** (début de la poignée de main TCP).

---

## 7. Fichier `/etc/nftables.conf` complet

> Adapter les noms d'interfaces (`enp0s3/8/9`) à la sortie réelle de `ip a`.
> Charger avec : `sudo nft -f /etc/nftables.conf` · Vérifier avec : `sudo nft list ruleset`

```nft
#!/usr/sbin/nft -f

flush ruleset

# --- Variables (lisibilité) ---
define WAN = "enp0s3"
define LAN = "enp0s8"
define DMZ = "enp0s9"

table inet filter {

    # Plages privées/APIPA interdites depuis le WAN.
    # /!\ On exclut la plage du WAN : 192.168.0.0/16 (WAN en 192.168.68.x).
    #     À l'école (WAN 10.x), on exclurait 10.0.0.0/8 à la place.
    set blocked_IPs {
        type ipv4_addr
        flags interval
        elements = {
            10.0.0.0/8,
            172.16.0.0/12,
            127.0.0.0/8,
            169.254.0.0/16
        }
    }

    chain input {
        type filter hook input priority filter; policy drop;

        # Interface loopback
        iif "lo" accept

        # Connexions déjà établies
        ct state established,related accept
        ct state invalid drop

        # Bogons usurpés arrivant par le WAN
        iifname $WAN ip saddr @blocked_IPs drop

        # ANY -> Firewall : ping autorisé (echo-request uniquement)
        icmp type echo-request accept

        # LAN -> Firewall : SSH pour 2 machines, max 3 nouvelles connexions/minute
        iifname $LAN ip saddr { 192.168.1.10, 192.168.1.20 } tcp dport 22 \
            ct state new limit rate 3/minute accept

        # WAN -> Firewall : logger puis fermer les tentatives SSH depuis Internet
        iifname $WAN tcp dport 22 ct state new \
            log prefix "SSH-WAN: " reject with tcp reset
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        ct state established,related accept
        ct state invalid drop

        # Bogons usurpés arrivant par le WAN
        iifname $WAN ip saddr @blocked_IPs drop

        # Adresses critiques : aucun accès WAN ni DMZ (une seule règle)
        ip saddr { 192.168.1.50-192.168.1.60, 192.168.1.100, 192.168.1.200 } \
            oifname { $WAN, $DMZ } drop

        # --- LAN -> WAN ---
        # DNS autorisé uniquement vers OpenDNS
        iifname $LAN oifname $WAN ip daddr 208.67.222.123 \
            meta l4proto { tcp, udp } th dport 53 accept
        # HTTPS sortant
        iifname $LAN oifname $WAN tcp dport 443 accept
        # HTTP refusé immédiatement (reject) au lieu de laisser ramer
        iifname $LAN oifname $WAN tcp dport 80 reject with tcp reset

        # --- LAN -> DMZ ---
        # Ping autorisé
        iifname $LAN oifname $DMZ icmp type echo-request accept
        # Accès HTTP au serveur web
        iifname $LAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 80 accept

        # --- WAN -> DMZ (destination DÉJÀ traduite par le DNAT) ---
        # HTTP
        iifname $WAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 80 accept
        # SSH (61337 traduit en 22 par le DNAT)
        iifname $WAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 22 accept
    }

    chain output {
        type filter hook output priority filter; policy drop;

        oif "lo" accept
        ct state established,related accept
    }
}

table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat;

        # WAN -> DMZ : DNAT du site web et du SSH
        iifname $WAN tcp dport 80    dnat to 172.16.0.10:80
        iifname $WAN tcp dport 61337 dnat to 172.16.0.10:22
    }

    chain postrouting {
        type nat hook postrouting priority srcnat;

        # LAN -> WAN : SNAT (masquerade)
        oifname $WAN ip saddr 192.168.1.0/24 masquerade
    }
}
```

---

## 8. Tests par étape

### Connectivité de base (avant d'appliquer le filtrage)

```bash
# Depuis le firewall
ping 192.168.1.10        # LAN
ping 172.16.0.10         # DMZ
ping 1.1.1.1             # Internet
```

### LAN → WAN (depuis le client LAN)

```bash
nslookup google.com 208.67.222.123          # DNS OpenDNS -> OK
wget -qO- https://www.google.com | head      # HTTPS -> OK
nslookup internetbadguys.com 208.67.222.123  # -> 146.112.61.x = bloqué par OpenDNS
wget http://www.enseignement.be              # HTTP -> "Connection refused" immédiat (reject)
```

### LAN → DMZ (depuis le client LAN)

```bash
ping 172.16.0.10
wget -qO- http://172.16.0.10 | head           # page Apache
```

### WAN → DMZ (depuis le client externe, via l'IP WAN du firewall)

```bash
wget http://<IP_WAN>                # site web de la DMZ
ssh -p 61337 user@<IP_WAN>          # SSH vers le serveur DMZ (pas le firewall)
```

### ANY → Firewall

```bash
ping 192.168.1.254                  # LAN
ping 172.16.0.245                   # DMZ
ping <IP_WAN>                       # externe
traceroute -I <IP_firewall>         # ICMP -> OK
traceroute <IP_firewall>            # UDP -> "* * *" (normal : seul echo-request est permis)
```

### LAN → Firewall : SSH + limite (depuis le client LAN)

```bash
ssh user@192.168.1.254              # doit se connecter
# Test de la limite (les premières passent puis "Connection timed out") :
for i in $(seq 1 8); do ssh -o BatchMode=yes -o ConnectTimeout=3 user@192.168.1.254 true; echo "--- $i ---"; done
```

### WAN → Firewall : logging SSH (depuis le client externe + firewall)

```bash
# Client externe :
ssh user@<IP_WAN>                   # "Connection refused" immédiat (reject)

# Firewall : ne voir que les logs SSH
sudo journalctl -k --grep "SSH-WAN"
sudo journalctl -kf | grep "SSH-WAN"
```

---

## 9. Commandes utiles

```bash
sudo nft -f /etc/nftables.conf                 # charger la config
sudo nft list ruleset                          # afficher toutes les règles
sudo nft list table inet filter                # une table précise
sudo nft list set inet filter blocked_IPs      # contenu d'un set
sudo nft insert rule inet filter output accept # ouvrir temporairement la sortie (puis recharger)
sudo journalctl -k --grep "SSH-WAN"            # logs nftables filtrés
```

---

## 10. Synthèse de la politique appliquée

| Sens | Autorisé |
|------|----------|
| **LAN → WAN** | HTTPS (443), DNS vers OpenDNS uniquement ; HTTP rejeté ; adresses critiques bloquées |
| **LAN → DMZ** | Ping, HTTP (80) vers le serveur web |
| **LAN → Firewall** | SSH (depuis `192.168.1.10` et `192.168.1.20` seulement, 3 conn./min) |
| **WAN → DMZ** | HTTP (80) et SSH (61337→22) via DNAT sur l'IP du firewall |
| **WAN → Firewall** | Ping (echo-request) ; SSH loggé puis fermé (reject) |
| **WAN → ANY** | Rien (drop par défaut, furtif) |
| **DMZ → LAN / WAN** | Rien à l'initiative de la DMZ (seules les réponses `established` passent) |
| **Tous → Firewall** | Ping (echo-request) ; bogons WAN bloqués via le set |

Politique par défaut sur les 3 chaînes de filtrage : **`drop`**.
