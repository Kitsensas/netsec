# Laboratoire VPN — OpenVPN & pfSense

Documentation complete de la manipulation :
- VPN site-to-site protege par une PKI
- VPN Remote Access protege par PKI + authentification RADIUS sur Active Directory

Environnement : VirtualBox, pfSense CE 2.8, Debian 13 (sans GUI), Windows Server.
Topologie : un Site A complet (reseau 10.210.0.0/16) relie par tunnel a un Site B
minimal (reseau 10.211.0.0/16) servant a demontrer le tunnel.


# 1. Materiel et machines virtuelles

Site A (5 VMs) :
- pfSense (pare-feu / serveur VPN) : 2 cartes
    Carte 1 = Acces par pont (WAN)
    Carte 2 = Reseau interne "SR1" (LAN)
- Windows Server (AD DS + DNS + NPS/RADIUS) : 1 carte, reseau interne "SR1"
- Routeur Debian : 3 cartes, reseaux internes "SR1", "SR2", "SR3"
- Client Debian 1 : 1 carte, reseau interne "SR2"
- Client Debian 2 : 1 carte, reseau interne "SR3"

Site B minimal (2 VMs) :
- pfSense B : 2 cartes (pont = WAN, reseau interne "SRB1" = LAN)
- 1 hote AVEC navigateur sur "SRB1" (sert a configurer pfSense B et a tester le tunnel)

Specs indicatives :
- pfSense : 2 Go RAM pour l'install, 2 vCPU, disque 20 Go
- Windows Server : 4 Go RAM, disque 40-60 Go
- Debian : 512 Mo a 1 Go

Pieges :
- Clonage de VM : TOUJOURS choisir "Generer de nouvelles adresses MAC pour toutes
  les cartes". Sinon deux cartes ont la meme MAC sur le reseau en pont = conflits,
  DHCP qui deconne, tunnel impossible.
- Install pfSense 2.8 : l'installeur plante avec "panic: Could not malloc ... zfs_zstd"
  si la VM manque de RAM. Solution : mettre 2 Go ET choisir "Auto (UFS)" au lieu de ZFS.
- Reseaux internes : le NOM doit etre tape a l'identique sur chaque carte ("SR1", "SR2"...),
  sinon les machines ne sont pas sur le meme switch virtuel.


# 2. Adressage IP et routage

Site A :
- WAN pfSense : DHCP (IP "publique" simulee, ex. 192.168.68.113)
- SR1 = 10.210.1.0/24 : pfSense LAN 10.210.1.1 / Windows Server 10.210.1.10 / Routeur 10.210.1.2
- SR2 = 10.210.2.0/24 : Routeur 10.210.2.1 / Client 1 10.210.2.10
- SR3 = 10.210.3.0/24 : Routeur 10.210.3.1 / Client 2 10.210.3.10

Site B :
- WAN pfSense B : DHCP (ex. 192.168.68.103)
- LAN = 10.211.1.0/24 : pfSense B 10.211.1.1 / hote 10.211.1.10

Routeur Debian, fichier /etc/network/interfaces :

    auto lo
    iface lo inet loopback

    auto enp0s3            # SR1 (vers pfSense + Windows Server)
    iface enp0s3 inet static
        address 10.210.1.2/24
        gateway 10.210.1.1

    auto enp0s8            # SR2
    iface enp0s8 inet static
        address 10.210.2.1/24

    auto enp0s9            # SR3
    iface enp0s9 inet static
        address 10.210.3.1/24

Activer le routage sur le routeur Debian :

    echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ipforward.conf
    sudo sysctl --system

Client Debian 1 (SR2), /etc/network/interfaces :

    auto enp0s3
    iface enp0s3 inet static
        address 10.210.2.10/24
        gateway 10.210.2.1

(Client Debian 2 = pareil en 10.210.3.10/24, passerelle 10.210.3.1)

Windows Server : IP statique 10.210.1.10/24, passerelle 10.210.1.1, DNS 127.0.0.1.
Route statique pour joindre SR2/SR3 via le routeur (invite cmd admin) :

    route -p add 10.210.0.0 mask 255.255.0.0 10.210.1.2

Pieges :
- Verifier le mapping carte->interface avec `ip a` (l'ordre enp0s3/8/9 peut varier).
- UNE seule ligne `gateway` sur le routeur (sur SR1, vers pfSense). SR2 et SR3 n'en ont
  PAS : le routeur lui-meme est leur passerelle.
- DNS Debian : la directive dans /etc/network/interfaces est `dns-nameservers` (PLURIEL)
  et exige le paquet resolvconf. Le plus simple = ecrire directement dans /etc/resolv.conf :
  `nameserver 8.8.8.8` (UN SEUL mot, sans tiret : pas "name-server").
- Le Windows Server a pfSense comme passerelle. Sans la route statique vers le routeur,
  il ne joint pas SR2/SR3.
- Le DNS du Windows Server doit pointer sur lui-meme (127.0.0.1), sinon l'AD ne marche pas.


# 3. Installation et configuration initiale de pfSense

Install :
- Type de VM : FreeBSD 64-bit. Partitionnement : Auto (UFS).
- Apres reboot, retirer l'ISO.

Console (menu 16 options) :
- Assignation : WAN = em0 (Carte 1, pont), LAN = em1 (Carte 2, SR1).
- Option 2 (Set interface IP) : LAN = 10.210.1.1/24, pas de DHCP, pas d'IPv6, garder HTTPS.
- Le WAN prend son IP en DHCP automatiquement.

Wizard (depuis un navigateur sur SR1, ex. le Windows Server, https://10.210.1.1) :
- Identifiants par defaut : admin / pfsense
- Etape WAN : DECOCHER "Block RFC1918 Private Networks".
- Changer le mot de passe admin.

Pieges :
- admin / pfsense en MINUSCULES, sensible a la casse.
- "Block private networks" sur le WAN : a DECOCHER absolument. Comme les IP "publiques"
  des deux sites sont privees (192.168.68.x), ce blocage empeche le tunnel VPN de
  s'etablir. Verifiable aussi dans Interfaces > WAN.
- Le webGUI n'est joignable que depuis le LAN par defaut. Il faut donc un navigateur
  sur SR1. Si on est bloque dehors : console option 3 = reset admin a pfsense.


# 4. Test de connectivite intra-site (etape cruciale)

Tous les sous-reseaux d'un site doivent se parler AVANT de toucher au VPN.
Depuis le Client Debian 1 (SR2) :

    ping 10.210.2.1     # sa passerelle (routeur)
    ping 10.210.3.10    # Client 2, autre sous-reseau
    ping 10.210.1.10    # Windows Server
    ping 10.210.1.1     # pfSense

Pieges :
- Tester depuis un CLIENT, pas depuis le routeur : le routeur est directement connecte
  a tout, donc ses pings ne prouvent pas que le routage inter-sous-reseaux fonctionne.
- Si ca echoue : verifier ip_forward=1 sur le routeur et la passerelle des clients.


# 5. Routes statiques sur pfSense (indispensable pour le VPN)

pfSense ne connait que SR1. Il faut lui apprendre a joindre SR2/SR3 via le routeur.

System > Routing > onglet Gateways > Add :
- Interface : LAN
- Name : ROUTER_DEBIAN
- Gateway : 10.210.1.2

System > Routing > onglet Static Routes > Add (deux fois) :
- 10.210.2.0/24 via ROUTER_DEBIAN
- 10.210.3.0/24 via ROUTER_DEBIAN

Test : Diagnostics > Ping vers 10.210.2.10 et 10.210.3.10.

Pieges :
- Sans ces routes, le VPN ne pourra jamais livrer de trafic dans SR2/SR3.
- C'est aussi ce qui permet au Windows Server (passerelle = pfSense) de joindre SR2/SR3
  si on n'a pas mis la route statique cote Windows.


# 6. VPN site-to-site avec PKI

Site A = serveur. Site B = client.

## 6.1 Creer la PKI (Site A, System > Cert. Manager)

Onglet CAs > Add :
- VPN-CA, methode "Create an internal Certificate Authority", RSA 2048, SHA256.

Onglet Certificates > Add (certificat SERVEUR) :
- vpn-server, signe par VPN-CA, Certificate Type = "Server Certificate".

Onglet Certificates > Add (certificat CLIENT) :
- site-b-virton, signe par VPN-CA, Common Name = site-b, Type = "User Certificate".

Pieges :
- NE PAS utiliser le certificat "webConfigurator Default" comme cert serveur du VPN.
- Bien mettre Certificate Type = Server pour le serveur, User pour le client.
- Retenir le Common Name du client (site-b) : il resservira pour les Client Specific
  Overrides (etape restriction).

## 6.2 OpenVPN Server (Site A, VPN > OpenVPN > Servers > Add)

- Server Mode : Peer to Peer (SSL/TLS)
- Protocol : UDP on IPv4 / Device : tun / Interface : WAN / Port : 1194
- TLS Configuration : DECOCHER "Use a TLS Key" (plus simple)
- Peer CA : VPN-CA / Server certificate : vpn-server
- IPv4 Tunnel Network : 10.250.0.0/24
- IPv4 Local network(s) : 10.210.0.0/16
- IPv4 Remote network(s) : 10.211.0.0/16

Firewall :
- Firewall > Rules > WAN > Add : Pass, UDP, dest = WAN address, port 1194.
- Firewall > Rules > OpenVPN > Add : Pass, any, any, any.

Pieges :
- Les /16 dans Local/Remote network(s) sont des ROUTES (resume des /24), pas des masques
  d'interface. Les hotes gardent leur /24.
- Oublier la regle sur l'onglet OpenVPN = tunnel "up" mais aucun ping ne passe.
- Si "Use a TLS Key" est coche, il faut l'activer ET partager la meme cle des deux cotes.

## 6.3 Exporter / importer les certificats

Sur Site A (Cert. Manager), exporter :
- le certificat de la CA (VPN-CA.crt)  -- PAS la cle privee de la CA
- le certificat client (site-b-virton.crt)
- la cle client (site-b-virton.key)

Transferer les 3 fichiers vers le Site B (le presse-papier VirtualBox marche rarement ;
utiliser un dossier partage, une cle USB ou un transfert de fichiers).

Sur Site B (Cert. Manager) :
- Onglet CAs > Add > "Import an existing Certificate Authority" : coller VPN-CA.crt,
  laisser la cle privee VIDE.
- Onglet Certificates > Add > "Import an existing Certificate" : coller le cert ET la cle.

Pieges :
- On exporte le CERTIFICAT de la CA, jamais sa cle privee (elle reste sur le serveur).
- Le client a besoin de SON certificat ET de SA cle privee, sinon il ne s'authentifie pas.
- Presse-papier inter-VM souvent casse (Guest Additions requis des deux cotes).

## 6.4 OpenVPN Client (Site B, VPN > OpenVPN > Clients > Add)

- Server Mode : Peer to Peer (SSL/TLS)
- Protocol : UDP on IPv4 / Device : tun / Interface : WAN
- Server host or address : 192.168.68.113 (WAN du Site A) / Port : 1194
- DECOCHER "Use a TLS Key"
- Peer CA : VPN-ca (importee) / Client Certificate : site-b-virton (importe)
- IPv4 Tunnel Network : VIDE
- IPv4 Remote network(s) : 10.210.0.0/16

Firewall Site B :
- Firewall > Rules > OpenVPN > Add : Pass, any, any, any.

Pieges :
- NE PAS oublier de creer le client : sans lui, rien n'initie le tunnel, le serveur reste
  "actif" mais aucun client ne se connecte (erreur frequente).
- Le client initie la connexion (sortie) : pas besoin de regle WAN entrante sur le Site B.
- Protocole, chiffrement, TLS key et certificats doivent correspondre au serveur.

## 6.5 Test du tunnel

- Status > OpenVPN sur les DEUX pfSense : etat = up (vert).
- Depuis l'hote SRB1 (10.211.1.10) : ping 10.210.1.10 ou 10.210.2.10.
- Depuis un client du Site A : ping 10.211.1.10.

Pieges :
- "up" ne veut pas dire que le trafic passe : il faut les regles sur l'onglet OpenVPN
  des DEUX cotes.
- En cas de probleme : Status > System Logs > OpenVPN (client ET serveur) donne la cause
  (cert, port bloque, TLS key qui ne correspond pas, "Block private networks"...).


# 7. Remote Access VPN : Active Directory + RADIUS

## 7.1 Active Directory (Windows Server)

- Installer les roles AD DS et DNS Server.
- Promouvoir en controleur de domaine : "Add a new forest", domaine ex. henallux.local,
  definir le mot de passe DSRM.
- Redemarrage automatique.

Verifier le DNS : Tools > DNS > Forward Lookup Zones doit contenir henallux.local et
_msdcs.henallux.local. Ou en PowerShell : nslookup henallux.local 127.0.0.1.

Pieges :
- Pendant la promotion, la case DNS peut etre grisee si le role DNS etait deja installe :
  c'est normal, le DNS est bien la.
- IP statique a conserver, DNS = 127.0.0.1.

## 7.2 Utilisateur et groupe (AD Users and Computers)

- Creer un utilisateur (ex. prof), mot de passe defini.
- Creer un groupe de securite global : VPN-Users.
- Ajouter l'utilisateur a VPN-Users (onglet Member Of).

Pieges :
- DECOCHER "User must change password at next logon" : sinon l'auth MS-CHAPv2 echoue.
- L'utilisateur DOIT etre membre de VPN-Users (c'est la condition de la Network Policy).
- Mot de passe non expirant = confort pour le labo.

## 7.3 NPS : installation, enregistrement, client RADIUS

- Installer le role "Network Policy and Access Services" (c'est lui qui contient NPS).
- Tools > Network Policy Server.
- Clic droit sur "NPS (Local)" > "Register server in Active Directory".
- RADIUS Clients > New :
    Friendly name : pfSense
    Address : 10.210.1.1   (l'IP LAN de pfSense = sa source quand il interroge NPS)
    Shared Secret : un secret robuste (a reutiliser a l'identique cote pfSense)

Pieges :
- L'adresse du client RADIUS = l'IP LAN de pfSense (10.210.1.1), PAS son WAN.
- "Register server in Active Directory" est obligatoire pour authentifier des comptes AD.
- Le shared secret doit etre strictement identique des deux cotes (casse comprise).

## 7.4 Network Policy (NPS)

Policies > Network Policies > New :
- Nom : VPN Access, type de NAS = Unspecified.
- Condition : Windows Groups = VPN-Users.
- Access Permission : Access granted.
- Authentication Methods : cocher MS-CHAP-v2, NE PAS cocher "Unencrypted (PAP)".
- IMPORTANT : remonter cette policy en position 1 (Processing Order = 1).

Pieges :
- L'ORDRE des policies : NPS a des policies "deny" par defaut. Si la tienne n'est pas en
  premier, une policy deny passe avant et refuse l'acces.
- La methode d'auth doit correspondre a celle de pfSense (MS-CHAPv2).
- La condition de groupe doit correspondre au groupe de l'utilisateur.

## 7.5 Serveur RADIUS dans pfSense

System > User Manager > Authentication Servers > Add :
- Type : RADIUS
- Protocol : MS-CHAPv2   (doit correspondre a la Network Policy NPS)
- Hostname or IP : 10.210.1.10 (le Windows Server)
- Shared Secret : le meme que dans NPS
- Services offered : Authentication / Port : 1812
- Authentication Timeout : 5
- RADIUS NAS IP Attribute : LAN (10.210.1.1)

Pieges :
- Le Protocol doit etre le meme que la policy NPS (ici MS-CHAPv2).
- Mettre un Authentication Timeout (5 s) : sinon, si NPS ne repond pas, la page part en
  erreur "50x" au lieu d'afficher un echec propre.
- Shared secret identique a NPS.

## 7.6 Test de l'authentification RADIUS

Diagnostics > Authentication : Authentication Server = ton serveur RADIUS,
Username = prof, Password = celui de l'utilisateur. Resultat attendu :
"User: prof authenticated successfully".

Pieges (c'est ici que ca coince le plus souvent) :
- Erreur "50x" = TIMEOUT = NPS ne repond pas. Cause n1 : le PARE-FEU WINDOWS bloque le
  port RADIUS. Activer la regle entrante "Serveur NPS (authentification RADIUS - UDP
  entrant)" (UDP 1812/1813), profil Domain. Pour confirmer : desactiver temporairement
  le pare-feu Windows et retester.
- Verifier que le service NPS tourne (services.msc).
- LE bon outil de diagnostic : Event Viewer > Custom Views > Server Roles >
  Network Policy and Access Services :
    Event 6273 = refus, la ligne "Reason" donne la cause exacte.
    Event 6274 = paquet rejete (souvent shared secret qui ne correspond pas).
    Aucun event = la requete n'arrive pas a NPS (pare-feu / service / mauvaise IP).
- Causes frequentes de refus : utilisateur pas dans VPN-Users, policy pas en position 1,
  shared secret different, "must change password at next logon" coche, methode d'auth
  qui ne correspond pas (PAP vs MS-CHAPv2).


# 8. Serveur OpenVPN Remote Access (a finaliser)

VPN > OpenVPN > Servers > Add (ou wizard OpenVPN Remote Access) :
- Server Mode : commencer par "Remote Access (User Auth)" pour valider RADIUS seul,
  puis basculer sur "Remote Access (SSL/TLS + User Auth)" pour ajouter la verif du cert.
- Backend for authentication : le serveur RADIUS NPS-RADIUS.
- Peer CA : VPN-CA / Server certificate : vpn-server.
- IPv4 Tunnel Network : un reseau dedie, ex. 10.251.0.0/24.
- IPv4 Local network(s) : les sous-reseaux a rendre accessibles aux clients distants.

Firewall : regle WAN Pass UDP 1194 (port choisi) + regle Pass sur l'onglet OpenVPN.
Client : utiliser le package "OpenVPN Client Export" pour generer la config du client.

Pieges :
- Tester PAR ETAPES comme le demande le labo : d'abord User Auth seul, puis ajouter le
  certificat (SSL/TLS + User Auth). Sinon on ne sait pas d'ou vient un echec.
- Ne pas cocher l'authentification non chiffree.
- Penser aux deux regles firewall (WAN + OpenVPN), sinon meme symptome que le site-to-site
  (connexion qui ne monte pas ou trafic qui ne passe pas).


# 9. Restriction d'acces (etape 4.6.1)

- Creer un nouveau couple utilisateur + certificat client (nouveau Common Name).
- VPN > OpenVPN > Client Specific Overrides > Add :
    Common Name : le CN exact du certificat de cet utilisateur.
    IPv4 Local Network/s : ne pousser que SR1 et SR2 (10.210.1.0/24, 10.210.2.0/24).

Pieges :
- Le Common Name de l'override doit correspondre EXACTEMENT au CN du certificat client.
- Ne pousser que les sous-reseaux 1 et 2 (pas SR3) pour cet utilisateur restreint.
- L'override s'applique a CET utilisateur uniquement, les autres gardent l'acces general.


# Recapitulatif des pieges les plus couteux

- Clonage sans regenerer les MAC.
- Install pfSense en ZFS sans assez de RAM (panic) -> UFS + 2 Go.
- "Block private networks" laisse coche sur le WAN -> tunnel impossible.
- Oubli des regles sur l'onglet OpenVPN -> tunnel up mais aucun trafic.
- Oubli de creer le CLIENT OpenVPN cote Site B.
- Pare-feu Windows qui bloque UDP 1812 -> erreur 50x au test RADIUS.
- Network Policy NPS pas en position 1 -> refus par une policy deny.
- Utilisateur pas membre du groupe VPN-Users.
- "User must change password at next logon" coche -> auth MS-CHAPv2 qui echoue.
- Protocole RADIUS pfSense != methode NPS.
