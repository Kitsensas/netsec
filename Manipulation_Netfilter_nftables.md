# Laboratoire - Securite des reseaux : Netfilter / nftables

Documentation complete de la manipulation : topologie LAN - WAN - DMZ avec un pare-feu
GNU/Linux configure en nftables, selon une politique de securite en default-deny
(tout est bloque par defaut, on ouvre au cas par cas).

Environnement : VirtualBox, Debian 13 (Trixie).

Principe du document : on part d'un squelette (etape 3), puis on AJOUTE les regles flux
par flux. A chaque etape on recharge avec `sudo nft -f /etc/nftables.conf` et on teste.

Topologie :

                              Internet
                                 |
                       [ Client externe ]   <- WAN (carte Reseau NAT ou Pont)
                                 |
                           enp0s3 (WAN)
                                 |
    [ Client LAN ] -- enp0s8 (LAN) [ FIREWALL ] enp0s9 (DMZ) -- [ Serveur Web/SSH ]
       192.168.1.10              nftables                 172.16.0.10

LAN et DMZ sont sur deux switchs virtuels (reseaux internes) DISTINCTS : tout trafic
entre zones est oblige de passer par le pare-feu. Le WAN donne l'acces Internet.


# 1. Materiel et machines virtuelles

VMs (4) :
- Firewall (routeur / pare-feu) : Debian 13 (netinst), 3 cartes
    Carte 1 = Reseau NAT ou Pont (WAN)        -> enp0s3
    Carte 2 = Reseau interne "lan"            -> enp0s8
    Carte 3 = Reseau interne "dmz"            -> enp0s9
- Serveur DMZ (Web HTTP + SSH) : Debian 13, 1 carte, reseau interne "dmz" -> enp0s3
- Client LAN (poste interne de confiance) : Debian 13, 1 carte, reseau interne "lan"
- Client externe (utilisateur Internet) : 1 carte, Reseau NAT ou Pont

Activer le routage sur le firewall (obligatoire, sinon rien ne traverse) :

    echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ipforward.conf
    sudo sysctl --system
    cat /proc/sys/net/ipv4/ip_forward    # doit afficher 1

Installer les services sur le serveur DMZ AVANT d'appliquer le filtrage :

    sudo apt update
    sudo apt install -y apache2 openssh-server

Pieges :
- ip_forward obligatoire sur le firewall, sinon aucun trafic ne traverse.
- Les noms de reseaux internes ("lan", "dmz") doivent etre identiques sur chaque carte,
  sinon les machines ne sont pas sur le meme switch virtuel.
- /etc/sysctl.conf peut ne pas exister sur une install minimale : utiliser un drop-in
  dans /etc/sysctl.d/ comme ci-dessus.
- Installer apache/ssh sur la DMZ AVANT de fermer le firewall : une fois la politique
  appliquee, la DMZ n'a plus d'acces Internet pour faire apt.


# 2. Adressage IP

WAN :
- Firewall enp0s3 : DHCP (IP "publique" simulee)
- Client externe : DHCP

LAN = 192.168.1.0/24 :
- Firewall enp0s8 : 192.168.1.254/24
- Client LAN : 192.168.1.10/24, passerelle 192.168.1.254, DNS 208.67.222.123

DMZ = 172.16.0.0/16 :
- Firewall enp0s9 : 172.16.0.254/16
- Serveur DMZ : 172.16.0.10/16, passerelle 172.16.0.254, DNS 208.67.222.123

Le DNS 208.67.222.123 est OpenDNS FamilyShield, qui bloque certains sites malicieux
(ex. internetbadguys.com).

Firewall, /etc/network/interfaces :

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
        address 192.168.1.254/24

    auto enp0s9
    iface enp0s9 inet static
        address 172.16.0.254/16

Serveur DMZ, /etc/network/interfaces :

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 172.16.0.10/16
        gateway 172.16.0.254

Client LAN, DNS force directement dans /etc/resolv.conf :

    echo 'nameserver 208.67.222.123' | sudo tee /etc/resolv.conf

Pieges :
- Verifier le mapping carte->interface avec `ip a` (l'ordre enp0s3/8/9 peut varier).
- DNS Debian : la directive dans interfaces est `dns-nameservers` (PLURIEL) et exige le
  paquet resolvconf. Le plus simple = ecrire dans /etc/resolv.conf le mot `nameserver`
  (UN SEUL mot, sans tiret : pas "name-server").
- Coherence de la passerelle DMZ : le serveur DMZ doit avoir gateway = l'IP REELLE du
  firewall cote DMZ (172.16.0.254), sinon le TCP se connecte mais la reponse ne revient
  jamais ("awaiting response..."). Attention .254 != .254.


# 3. Squelette de depart

Creer /etc/nftables.conf avec le socle : politique drop sur les 3 chaines de filtrage,
loopback autorisee, conntrack. La table nat est creee mais vide pour l'instant.

    #!/usr/sbin/nft -f

    flush ruleset

    define WAN = "enp0s3"
    define LAN = "enp0s8"
    define DMZ = "enp0s9"

    table inet filter {
        chain input {
            type filter hook input priority filter; policy drop;
            iif "lo" accept
            ct state established,related accept
            ct state invalid drop
        }
        chain forward {
            type filter hook forward priority filter; policy drop;
            ct state established,related accept
            ct state invalid drop
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
        }
        chain postrouting {
            type nat hook postrouting priority srcnat;
        }
    }

Charger : `sudo nft -f /etc/nftables.conf`   Verifier : `sudo nft list ruleset`

Pieges :
- policy drop UNIQUEMENT sur les 3 chaines de FILTRAGE (input, forward, output). NE PAS
  toucher aux chaines de la table nat : leur policy accept est normale (on ne filtre
  jamais dans nat).
- Travailler depuis la CONSOLE de la VM firewall, pas en SSH : policy drop couperait une
  session SSH tant que la regle SSH n'existe pas.
- Apres ce chargement, presque tout est bloque (default-deny) : c'est voulu, on rouvre
  flux par flux.


# 4. LAN -> WAN (SNAT, DNS, HTTPS, reject HTTP, adresses critiques)

Ajouter dans la chaine postrouting (table ip nat) :

    oifname $WAN ip saddr 192.168.1.0/24 masquerade

Ajouter dans la chaine forward (table inet filter), apres le conntrack :

    # Adresses critiques : aucun acces WAN ni DMZ (une seule regle), placee EN PREMIER
    ip saddr { 192.168.1.50-192.168.1.60, 192.168.1.100, 192.168.1.200 } \
        oifname { $WAN, $DMZ } drop

    # DNS autorise uniquement vers OpenDNS
    iifname $LAN oifname $WAN ip daddr 208.67.222.123 \
        meta l4proto { tcp, udp } th dport 53 accept

    # HTTPS sortant
    iifname $LAN oifname $WAN tcp dport 443 accept

    # HTTP refuse immediatement (reject) au lieu de laisser ramer
    iifname $LAN oifname $WAN tcp dport 80 reject with tcp reset

Recharger, puis tester (voir etape 11).

Pieges :
- reject vs drop : pour le LAN (utilisateurs de confiance) -> reject with tcp reset
  (reponse immediate, le navigateur ne rame pas). Pour WAN -> ANY -> drop (le pare-feu
  reste furtif). C'est la reponse aux deux questions du labo.
- masquerade : sans lui, le retour d'Internet ne sait pas revenir vers 192.168.1.0/24.
- DNS restreint a 208.67.222.123 : une requete vers un autre DNS sera bloquee.
- La regle des adresses critiques doit etre placee AVANT les autorisations, sinon elle
  ne prime pas.


# 5. LAN -> DMZ (ping + HTTP)

Ajouter dans la chaine forward :

    # Ping autorise
    iifname $LAN oifname $DMZ icmp type echo-request accept

    # Acces HTTP au serveur web
    iifname $LAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 80 accept

Pieges :
- Aucun NAT pour ce flux : le firewall route simplement entre deux reseaux internes ;
  le serveur DMZ voit la vraie IP source du client.
- Aucune regle DMZ -> LAN necessaire : les reponses reviennent via ct state
  established,related. La DMZ ne peut donc PAS initier de connexion vers le LAN.
- Si le ping passe mais pas le HTTP ("awaiting response") : passerelle du serveur DMZ
  cassee (voir piege etape 2).


# 6. WAN -> DMZ (DNAT du site web + SSH 61337 -> 22)

Ajouter dans la chaine prerouting (table ip nat) :

    iifname $WAN tcp dport 80    dnat to 172.16.0.10:80
    iifname $WAN tcp dport 61337 dnat to 172.16.0.10:22

Ajouter dans la chaine forward (table inet filter) :

    # destination DEJA traduite par le DNAT -> on filtre sur l'IP interne et le port 22
    iifname $WAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 80 accept
    iifname $WAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 22 accept

Pieges :
- LE piege classique : le DNAT (prerouting) s'execute AVANT la chaine forward. Quand le
  paquet arrive dans forward, sa destination est DEJA traduite. On filtre donc sur
  `ip daddr 172.16.0.10 tcp dport 22` (le port 22 TRADUIT), surtout PAS 61337.
- Aucun SNAT necessaire : le serveur DMZ ayant le firewall comme passerelle, conntrack
  inverse automatiquement le DNAT au retour.
- Tester depuis le client externe via l'IP WAN du firewall, jamais via 172.16.0.10.


# 7. ANY -> Firewall (ping echo-request)

Ajouter dans la chaine input :

    icmp type echo-request accept

Pieges :
- echo-request UNIQUEMENT (precision demandee par le labo).
- L'echo-reply genere par le firewall sort grace a ct state established,related dans
  output : rien a ajouter cote output.
- traceroute UDP (defaut Linux) affiche "* * *" pour le firewall (normal). traceroute -I
  (ICMP) fonctionne : c'est la demonstration de la precision de la regle.


# 8. LAN -> Firewall (SSH pour 2 machines + limite 3/min)

Prerequis : openssh-server installe sur le firewall. Si output est en drop, ouvrir
temporairement la sortie pour faire apt, puis recharger :

    sudo nft insert rule inet filter output accept
    sudo apt update && sudo apt install -y openssh-server
    sudo nft -f /etc/nftables.conf   # le flush ruleset enleve la regle temporaire

Ajouter dans la chaine input :

    iifname $LAN ip saddr { 192.168.1.10, 192.168.1.20 } tcp dport 22 \
        ct state new limit rate 3/minute accept

Pieges :
- Une seule regle pour deux machines = set anonyme { 192.168.1.10, 192.168.1.20 }.
- ct state new + limit rate 3/minute = limite des OUVERTURES de connexion. La rafale
  (burst) par defaut est de 5 : on voit ~5 connexions passer avant blocage. Ajouter
  `burst 3 packets` pour un test strict a 3.
- TESTER DEPUIS LE CLIENT LAN, pas depuis le firewall : une connexion du firewall vers
  sa propre IP passe par lo (capturee par `iif "lo" accept`), donc la limite n'est jamais
  evaluee.


# 9. WAN -> Firewall (logging des tentatives SSH puis fermeture)

Ajouter dans la chaine input :

    iifname $WAN tcp dport 22 ct state new \
        log prefix "SSH-WAN: " reject with tcp reset

Lire les logs (cote firewall) :

    sudo journalctl -k --grep "SSH-WAN"
    sudo journalctl -kf | grep "SSH-WAN"

Pieges :
- ct state new pour ne logger que les nouvelles connexions.
- Avec drop : aucune reponse -> le client retransmet son SYN -> PLUSIEURS logs par
  tentative. Avec reject with tcp reset : fermeture immediate (RST) -> UN seul log par
  tentative. Le flag TCP de ces paquets est SYN.
- Reperer une meme tentative par son port source (SPT) identique sur plusieurs lignes.


# 10. Set blocked_IPs (plages privees interdites sur le WAN)

Ajouter le set DANS la table inet filter (au meme niveau que les chaines) :

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

Ajouter la regle de blocage EN HAUT des chaines input ET forward (juste apres le
conntrack) :

    iifname $WAN ip saddr @blocked_IPs drop

Verifier : `sudo nft list set inet filter blocked_IPs`

Pieges :
- NE JAMAIS bloquer la plage dans laquelle se trouve sa propre interface WAN. A l'ecole
  (WAN en 10.x) -> exclure 10.0.0.0/8. A la maison (WAN en 192.168.68.x) -> exclure
  192.168.0.0/16 a la place, sinon on se coupe Internet et le client externe.
- Appliquer le drop en input ET en forward, place en HAUT des chaines.


# 11. Tests par etape

Connectivite de base, depuis le firewall (avant filtrage) :

    ping 192.168.1.10        # LAN
    ping 172.16.0.10         # DMZ
    ping 1.1.1.1             # Internet

LAN -> WAN, depuis le client LAN :

    nslookup google.com 208.67.222.123           # DNS OpenDNS -> OK
    wget -qO- https://www.google.com | head       # HTTPS -> OK
    nslookup internetbadguys.com 208.67.222.123   # -> 146.112.61.x = bloque par OpenDNS
    wget http://www.enseignement.be               # HTTP -> "Connection refused" immediat

LAN -> DMZ, depuis le client LAN :

    ping 172.16.0.10
    wget -qO- http://172.16.0.10 | head            # page Apache

WAN -> DMZ, depuis le client externe (via l'IP WAN du firewall) :

    wget http://<IP_WAN>                 # site web de la DMZ
    ssh -p 61337 user@<IP_WAN>           # SSH vers le serveur DMZ (pas le firewall)

ANY -> Firewall :

    ping 192.168.1.254                   # LAN
    ping 172.16.0.254                    # DMZ
    ping <IP_WAN>                        # externe
    traceroute -I <IP_firewall>          # ICMP -> OK
    traceroute <IP_firewall>             # UDP -> "* * *" (normal)

LAN -> Firewall, SSH + limite (depuis le client LAN) :

    ssh user@192.168.1.254               # doit se connecter
    for i in $(seq 1 8); do ssh -o BatchMode=yes -o ConnectTimeout=3 user@192.168.1.254 true; echo "--- $i ---"; done

WAN -> Firewall, logging SSH :

    # client externe :
    ssh user@<IP_WAN>                    # "Connection refused" immediat (reject)
    # firewall :
    sudo journalctl -k --grep "SSH-WAN"


# 12. Commandes utiles

    sudo nft -f /etc/nftables.conf                  # charger la config
    sudo nft list ruleset                           # afficher toutes les regles
    sudo nft list table inet filter                 # une table precise
    sudo nft list set inet filter blocked_IPs       # contenu d'un set
    sudo nft insert rule inet filter output accept  # ouvrir temporairement la sortie
    sudo journalctl -k --grep "SSH-WAN"             # logs nftables filtres


# 13. Synthese de la politique appliquee

- LAN -> WAN : HTTPS (443) ; DNS vers OpenDNS uniquement ; HTTP rejete ; adresses
  critiques bloquees.
- LAN -> DMZ : ping ; HTTP (80) vers le serveur web.
- LAN -> Firewall : SSH depuis 192.168.1.10 et 192.168.1.20 seulement, 3 conn./min.
- WAN -> DMZ : HTTP (80) et SSH (61337 -> 22) via DNAT sur l'IP du firewall.
- WAN -> Firewall : ping (echo-request) ; SSH logge puis ferme (reject).
- WAN -> ANY : rien (drop par defaut, furtif).
- DMZ -> LAN / WAN : rien a l'initiative de la DMZ (seules les reponses established
  passent).
- Tous -> Firewall : ping (echo-request) ; bogons WAN bloques via le set.

Politique par defaut sur les 3 chaines de filtrage : drop.


# Recapitulatif des pieges les plus couteux

- policy accept au lieu de policy drop sur les chaines de filtrage.
- /etc/sysctl.conf inexistant -> utiliser un drop-in /etc/sysctl.d/.
- dns-nameserver (singulier) dans interfaces, ou name-server (avec tiret) dans
  resolv.conf -> aucune resolution. Bon mot-cle : nameserver (un seul mot).
- Passerelle du serveur DMZ cassee (169.254.x) -> TCP connecte mais reponse jamais
  recue. Definir gateway 172.16.0.254.
- DNAT : filtrer dans forward sur le port traduit (22) et l'IP interne, pas sur 61337.
- SSH absent du firewall + output drop -> apt impossible : ouvrir la sortie
  temporairement puis recharger.
- Tester la limite SSH depuis le firewall lui-meme (passe par lo) -> tester depuis le
  client LAN.
- Set blocked_IPs : bloquer sa propre plage WAN -> perte d'Internet. Exclure la plage
  du WAN.
