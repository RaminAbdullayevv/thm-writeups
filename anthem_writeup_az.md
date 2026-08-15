# TryHackMe — Anthem CTF Writeup (Azərbaycanca)

**Platforma:** TryHackMe  
**Otaq:** [Anthem](https://tryhackme.com/room/anthem)  
**Çətinlik:** Asan  
**OS:** Windows  
**Tarix:** 2026  

---

## 📋 Məzmun

1. [Kəşfiyyat (Reconnaissance)](#1-kəşfiyyat)
2. [Veb Tərəfin Araşdırılması](#2-veb-tərəfin-araşdırılması)
3. [İstifadəçi Məlumatlarının Tapılması](#3-istifadəçi-məlumatlarının-tapılması)
4. [RDP ilə Qoşulma](#4-rdp-ilə-qoşulma)
5. [İstifadəçi Flag-ı](#5-istifadəçi-flag-ı)
6. [Privilege Escalation](#6-privilege-escalation)
7. [Administrator Flag-ı](#7-administrator-flag-ı)
8. [Nəticə](#8-nəticə)

---

## 1. Kəşfiyyat

### Nmap Skan

```bash
nmap -sV -sC -p- 10.10.x.x
```

**Açıq portlar:**

| Port | Xidmət | Versiya |
|------|--------|---------|
| 80   | HTTP   | IIS     |
| 3389 | RDP    | Windows |

---

## 2. Veb Tərəfin Araşdırılması

### Gobuster ilə Qovluq Skanı

```bash
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan səhifələr:**

```
/archive        (Status: 301)
/authors        (Status: 200)
/blog           (Status: 200)
/categories     (Status: 200)
/install        (Status: 302) → /umbraco/
/robots.txt     (Status: 200)
/rss            (Status: 200)
/search         (Status: 200)
/sitemap        (Status: 200)
/tags           (Status: 200)
/umbraco        (Status: 200)
```

### robots.txt

```bash
curl http://10.10.x.x/robots.txt
```

`robots.txt` faylında **flag** və gizli yollar tapıldı.

### Umbraco CMS

`/umbraco` — Admin paneli. CMS olaraq **Umbraco** istifadə olunduğu müəyyən edildi.

---

## 3. İstifadəçi Məlumatlarının Tapılması

### Şair Kim?

Blog yazılarını araşdırdıqda bir şeir tapıldı. Şeiri Google-da axtardıqda müəllifin **Solomon Grundy** olduğu məlum oldu.

Admin e-poçtu formatı: `SG@anthem.com`

### Şifrə

Saytın müxtəlif hissələrini araşdırdıqda şifrə açıq şəkildə tapıldı:

```
UmbracoIsTheBest!
```

### Umbraco Admin Girişi

```
URL:      http://10.10.x.x/umbraco
İstifadəçi: SG@anthem.com
Şifrə:    UmbracoIsTheBest!
```

---

## 4. RDP ilə Qoşulma

Kali Linux-dan RDP bağlantısı:

```bash
xfreerdp /u:SG /p:'UmbracoIsTheBest!' /v:10.10.x.x
```

Uğurla Windows masaüstünə daxil olundu.

---

## 5. İstifadəçi Flag-ı

Windows masaüstündə `user.txt` faylı tapıldı.

```
THM{**************}
```

---

## 6. Privilege Escalation

### Gizli Qovluğu Tap

Explorer-də **View → Hidden Items** seçildi.

`C:\backup` qovluğu tapıldı. İçərisində `restore.txt` faylı var idi.

### İcazə Problemi

`restore.txt` faylını açmaq istədikdə:

```
You do not have permission to open this file.
```

### İcazənin Verilməsi

**Metod 1 — GUI:**
1. `restore.txt` → Sağ klik → Properties
2. Security → Edit → Add
3. `SG` yazıb Check Names → OK
4. **Read** işarələ → Apply → OK

**Metod 2 — PowerShell:**

```powershell
icacls C:\backup\restore.txt /grant SG:R
```

### Şifrənin Əldə Edilməsi

`restore.txt` faylını açdıqda **Administrator şifrəsi** tapıldı.

---

## 7. Administrator Flag-ı

Administrator hesabına keçid:

```powershell
runas /user:Administrator cmd
```

və ya yeni RDP sessiyası:

```bash
xfreerdp /u:Administrator /p:'TAPILAN_SIFRE' /v:10.10.x.x
```

`C:\Users\Administrator\Desktop\root.txt` faylında son flag tapıldı:

```
THM{**************}
```

---

## 8. Nəticə

### İstifadə Olunan Alətlər

| Alət       | Məqsəd                        |
|------------|-------------------------------|
| Nmap       | Port skanı                    |
| Gobuster   | Qovluq kəşfi                  |
| xfreerdp   | RDP bağlantısı                |
| curl       | HTTP sorğuları                |
| icacls     | Windows fayl icazələri        |
| PowerShell | Sistem komandaları            |

### Öyrənilən Dərslər

- `robots.txt` həmişə yoxlanılmalıdır
- Gizli fayllar (`Hidden Items`) mühüm məlumat saxlaya bilər
- Windows fayl icazələri privilege escalation üçün istifadə edilə bilər
- Şifrələr heç vaxt açıq mətndə saxlanmamalıdır
- CMS sistemlərinin default path-ları dəyişdirilməlidir

---

*Bu writeup yalnız təhsil məqsədli hazırlanmışdır. TryHackMe platformasındakı CTF laboratoriyasına aiddir.*
