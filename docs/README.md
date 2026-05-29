# Pentest Active Directory : Référence Exam

Ce GitBook est ta référence complète pour l'épreuve certificative de pentest Active Directory.

## Contexte de l'épreuve

- Connexion **SSH** fournie vers une machine **Kali Linux** déjà positionnée dans le réseau cible
- Objectif : trouver le maximum de **flags / accès** en autonomie
- Environnement cible : réseau Windows avec **Active Directory**

## Comment utiliser ce GitBook pendant l'exam

| Situation | Aller à |
|-----------|---------|
| Démarrage, je viens de me connecter en SSH | [Phase 0 : Setup Kali](phases/00-setup-kali.md) |
| Je veux le déroulé complet étape par étape | [Méthodologie Exam](methodologie/methodology-exam.md) |
| Je suis bloqué, je cherche un scénario précis | [Scénarios types](methodologie/scenarios.md) |
| Je veux une commande précise vite | [Cheatsheet commandes](cheatsheets/commandes.md) |
| Je dois choisir un mode hashcat | [Cheatsheet hashcat](cheatsheets/hashcat.md) |
| Je ne comprends pas un concept | [Fondamentaux](fondamentaux/active-directory.md) |

## Chaîne d'attaque résumée

```
[Phase 0] Setup Kali → variables, outils, dossier de travail
    ↓
[Phase 1] Reconnaissance → Nmap, nslookup, identifier le DC
    ↓
[Phase 2] Accès Initial → Responder, Relay NTLM, MITMv6, Password Spraying
    ↓
[Phase 3] Énumération → nxc, BloodHound, ldapdomaindump, SMB shares
    ↓
[Phase 4] Escalade → Kerberoasting, AS-REP Roasting, ACL Abuse, ADCS
    ↓
[Phase 5] Mouvement Latéral → PtH, PtT, OPtH, psexec/wmiexec/evil-winrm
    ↓
[Phase 6] Compromission → DCSync, Golden Ticket, persistance
```

## Règles d'or pour l'exam

1. **Créer son dossier de travail en premier** : tout logguer
2. **Lancer BloodHound dès le premier compte** : c'est la carte du territoire
3. **Lire le graphe BloodHound avant de choisir une technique**
4. **Épreuve à tiroir** : chaque flag peut cacher des credentials pour la suite
5. **Toujours tester les credentials trouvés sur toutes les machines** avec `nxc smb`
6. **Vérifier SYSVOL et NETLOGON** : mines à flags
