# Fondamentaux Wi-Fi

## Les normes 802.11

Le Wi-Fi transmet des données via des ondes radio selon les normes **IEEE 802.11**. Chaque génération apporte une fréquence et un débit plus élevés. Les vitesses ci-dessous sont théoriques.

| Norme | Nom commercial | Vitesse max | Fréquence | Année |
|---|---|---|---|---|
| 802.11b | Wi-Fi 1 | 11 Mbit/s | 2,4 GHz | 1999 |
| 802.11g | Wi-Fi 3 | 54 Mbit/s | 2,4 GHz | 2003 |
| 802.11n | Wi-Fi 4 | 600 Mbit/s | 2,4 + 5 GHz | 2009 |
| 802.11ac | Wi-Fi 5 | 1,3 Gbit/s | 5 GHz | 2013 |
| 802.11ax | Wi-Fi 6 | 10,5 Gbit/s | 1 à 7 GHz | 2021 |
| 802.11be | Wi-Fi 7 | 40 Gbit/s | 2,4 + 5 + 6 GHz | 2024 |

> Plus la fréquence est haute (5 GHz), plus le débit est élevé mais moins ça traverse les obstacles.

---

## Les 4 modes réseau Wi-Fi

### Infrastructure

Un **point d'accès (AP)** central, tous les appareils passent par lui. C'est le mode classique (box, réseau d'entreprise). Les appareils ne communiquent pas directement entre eux : tout transite par l'AP. Tous partagent le même **SSID**.

### Ad-Hoc / Wi-Fi Direct

Connexion directe entre deux appareils, sans AP. Pratique pour un lien temporaire (AirDrop, partage de connexion).

### Pont (Bridge)

Un AP dans chaque bâtiment, les deux communiquent par Wi-Fi et forment un seul réseau logique. Les antennes sont souvent directionnelles. La connexion opère en couche 2 : les machines des deux bâtiments sont sur le même LAN.

```
Bâtiment A                          Bâtiment B
[Switch] <-> [AP Bridge]  ))))  (((( [AP Bridge] <-> [Switch]
```

### Répéteur

Reçoit le signal Wi-Fi et le réémet. Attention : le débit est divisé approximativement **par deux** car l'antenne reçoit et émet sur le même canal simultanément.

---

## Le problème fondamental côté sécurité

Les ondes radio ne s'arrêtent pas aux murs. Quiconque dispose d'une antenne dans le périmètre peut intercepter les trames. C'est pourquoi le chiffrement est indispensable.

---

## WEP : le protocole cassé

### Comment ça fonctionne

WEP chiffre via **XOR + RC4**. Pour chaque trame :

1. **Seed** = IV (3 octets, aléatoire) + clé secrète (5 ou 13 octets)
2. **Keystream** : la seed passe dans RC4, qui génère un flux pseudo-aléatoire
3. **ICV** : un CRC-32 du message est calculé (intégrité)
4. **Trame envoyée** = `(Message + ICV) XOR Keystream`, précédé de l'IV **en clair**

```
┌──────────────┬─────────────────────────────────┐
│  IV en clair │    Message chiffré + ICV         │
└──────────────┴─────────────────────────────────┘
      ↑
  Tout le monde peut le lire
```

### Les failles

| Problème | Conséquence |
|---|---|
| IV de 24 bits (16M de valeurs) | Se répète sur un réseau actif en quelques heures |
| IV envoyé en clair | L'attaquant sait exactement quand deux trames partagent le même IV |
| Clé statique et partagée | Une fuite compromet tout le réseau |
| CRC-32 non cryptographique | Un attaquant peut modifier un message chiffré et recalculer un ICV valide |

Avec **aircrack-ng**, ~50 000 trames suffisent pour retrouver la clé. Sur un réseau actif, ça prend moins de 10 minutes. WEP est abandonné depuis 2004.

---

## WPA et WPA2 : les correctifs

WPA (2003) est un patch d'urgence compatible avec l'ancien matériel. WPA2 (2004) est la solution complète.

### Ce que WPA corrige

| Problème WEP | Correction WPA |
|---|---|
| RC4 avec clé fixe | **TKIP** : clé de chiffrement différente pour chaque trame |
| IV de 24 bits | IV passe à **48 bits** (281 000 milliards de valeurs) |
| CRC-32 falsifiable | **MIC (Michael)** : intégrité cryptographique + compteur anti-rejeu |

WPA2 remplace TKIP par **AES + CCMP** : le standard bancaire et gouvernemental.

| | WPA | WPA2 |
|---|---|---|
| Chiffrement | TKIP (RC4) | AES |
| Protocole | TKIP | CCMP |

### Les deux modes WPA2

**Personal (PSK)** : un seul mot de passe partagé. Simple, usage domestique.

**Enterprise** : chaque utilisateur a ses propres identifiants, gérés par un serveur **RADIUS** via le protocole **802.1X / EAP**.

### Transformation du mot de passe en clé (PMK)

WPA2 ne transmet jamais le mot de passe directement. Il le transforme en **PMK (Pairwise Master Key)** de 256 bits :

```
Mot de passe "Wifimaison84"
        +
SSID "Livebox-4F2A"  (sel)
        +
4096 répétitions HMAC-SHA1
        ↓
PMK = clé 256 bits
```

Le SSID sert de sel pour que deux réseaux avec le même mot de passe aient des PMK différentes, ce qui empêche les tables précalculées.

### La vraie faille WPA2 : le mot de passe faible

WPA2 est solide. Le maillon faible, c'est le choix du mot de passe.

**Handshake (4-way handshake)** : lors de la connexion, un échange en 4 messages contient un hash du mot de passe. Un attaquant peut le capturer passivement ou en forçant une déconnexion.

```
Client                               AP
  |--- "Salut, je veux me connecter" →|
  |← "Prouve que tu connais la clé" --|
  |--- [Hash(mot de passe)] --------->|
  |←-- "OK, bienvenue" ---------------|
              ↑
       L'attaquant capture ça
```

Ensuite il teste des mots de passe en local via dictionnaire. Pas besoin de toucher le réseau.

---

## WPS : la fausse bonne idée

**WPS (Wi-Fi Protected Setup)**, créé en 2007 pour simplifier la configuration, introduit plusieurs vulnérabilités critiques.

### Les 3 méthodes

- **PIN** : code 8 chiffres affiché sous le routeur, tapé sur l'appareil à connecter
- **PBC** : appui simultané sur le bouton WPS de la box et de l'appareil
- **NFC** : approcher physiquement l'appareil du routeur

### Faille PIN : 11 000 combinaisons au lieu de 100 millions

Le routeur vérifie les deux moitiés du PIN séparément et indique si la première moitié est correcte avant que la seconde soit soumise. De plus, le 8e chiffre est une somme de contrôle calculée automatiquement.

```
100 000 000 combinaisons théoriques
        ↓
10 000 (1ère moitié) + 1 000 (2ème moitié, 7e chiffre libre)
        ↓
≈ 11 000 combinaisons à tester
```

Avec un outil automatisé, quelques heures suffisent.

### Attaque Pixie Dust : quelques secondes

Sur certains chipsets, les valeurs **ES1** et **ES2** (nombres aléatoires secrets de la négociation WPS) sont prévisibles ou nulles :

```
Routeur vulnérable :
ES1 = 00000000000000000000000000000000
ES2 = 00000000000000000000000000000000
```

En interceptant quelques valeurs publiques de la négociation (PKE, PKR, E-Hash1, E-Hash2), on recalcule le PIN hors ligne en millisecondes.

### Faille PBC : fenêtre de 2 minutes

Après appui sur le bouton, n'importe quel appareil peut s'associer pendant 2 minutes. Un script qui surveille en boucle peut s'authentifier avant l'appareil légitime.

**Contre-mesure** : désactiver WPS (surtout la méthode PIN) dans l'interface d'administration du routeur.

---

## Récapitulatif des protocoles

| Protocole | Chiffrement | Point faible principal | Statut |
|---|---|---|---|
| WEP | RC4 + XOR | IV 24 bits, clé statique | Abandonné 2004 |
| WPA | TKIP | TKIP faible, mot de passe | À éviter |
| WPA2-PSK | AES + CCMP | Mot de passe faible | Standard actuel |
| WPA2-Enterprise | AES + CCMP + EAP | Config EAP laxiste, certificat non vérifié | Solide si bien configuré |
| WPS PIN | N/A | 11 000 combinaisons, Pixie Dust | Désactiver |
| WPA3-SAE | AES + CCMP | Brute force en ligne si mdp faible | Meilleur standard |
