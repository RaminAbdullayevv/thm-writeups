# 🕵️ TryHackMe — OhSINT | Azərbaycan Dilli Writeup

**Çətinlik Səviyyəsi:** Başlanğıc  
**Kateqoriya:** OSINT (Açıq Mənbəli Kəşfiyyat)  
**Platforma:** [TryHackMe](https://tryhackme.com/room/ohsint)

---

## 📌 Giriş

**OSINT (Open-Source Intelligence)** — açıq mənbələrdən məlumat toplama sənətidir. Bu bacarıq kibertəhlükəsizlik sahəsində çox vacibdir, çünki bir insanın ya da sistemin ictimaiyyətə açıq məlumatları potensial zəifliklər barədə bizə çox şey deyə bilər.

Bu writeup-da TryHackMe platformasındakı **OhSINT** otağını addım-addım həll edəcəyik. Başlanğıc üçün əla bir OSINT tapşırığıdır.

---

## 📁 Tapşırığı Başa Düşmək

Otağa daxil olduqdan sonra **Task 1**-dəki mavi "Download Task Files" düyməsinə basıb faylı yükləyirik.

Bizə verilən yeganə fayl:

```
WindowsXP_1551719014755.jpg
```

İlk baxışda bu şəkil tamamilə adi görünür — Windows XP-nin klassik çəmənlik fonu. Heç bir açar söz, heç bir gizli mətn yoxdur. Amma OSINT-də biz bilirik ki, əsl məlumat çox vaxt **səthin altında** gizlənir. Gəlin metadata-ya baxaq!

---

## 🔍 Addım 1 — Şəkilin Metadata-sını Çıxarmaq (ExifTool)

Şəkilin gizli məlumatlarını (GPS koordinatları, müəllif, kamera modeli, tarixlər) əldə etmək üçün **ExifTool**-dan istifadə edirik.

### Quraşdırma:

```bash
sudo apt install exiftool
```

### İstifadə:

```bash
exiftool WindowsXP_1551719014755.jpg
```

### Nəticə — 2 Əsas Məlumat Tapıldı:

| Sahə | Dəyər |
|------|-------|
| GPS Koordinatları | 54°17'41.27"N, 2°15'1.33"W |
| Copyright | **OWoodflint** |

> 💡 **İzah:** ExifTool şəkilin EXIF məlumatlarını oxuyur. Bu məlumatlar kameralar və telefonlar tərəfindən avtomatik olaraq şəkil faylının içinə yazılır. Çoxları bu məlumatların orada olduğundan belə xəbərsizdir!

---

## 🔎 Addım 2 — "OWoodflint" Adını Araşdırmaq

`OWoodflint` adını götürüb Google-da axtarırıq.

**Nəticədə 3 platforma tapıldı:**

- 🐦 **X (köhnə Twitter)** — `@OWoodflint`
- 🐙 **GitHub** — `OWoodflint`
- 📝 **WordPress Blogu**

Bu üç mənbə bizə bütün sualların cavablarını verəcək.

---

## ✅ Sual 1 — İstifadəçinin Avatarı Nədir?

**Platforma:** X (Twitter)  
**Metod:** `@OWoodflint` profilinə baxırıq.

Profil şəklinə baxanda görürük ki, avatar bir **pişikdir (cat)**.

**Cavab:** `cat` 🐱

---

## ✅ Sual 2 — İstifadəçi Haradadır?

**Platforma:** GitHub  
**Metod:** `OWoodflint`-in GitHub profilinə, xüsusilə README faylına baxırıq.

README faylında açıq şəkildə yazılıb ki, istifadəçi **Londondan**dır.

**Cavab:** `London` 🇬🇧

---

## ✅ Sual 3 — Qoşulduğu Wi-Fi Şəbəkəsinin SSID-i Nədir?

Bu sual bir az daha dərin araşdırma tələb edir.

### Addım 3a — BSSID-i Tapmaq (Twitter)

`@OWoodflint`-in X (Twitter) hesabını yoxlayırıq. Orada bir tvit var ki, istifadəçi bir **BSSID** paylaşıb:

```
B4:5D:50:AA:86:41
```

> 💡 **BSSID nədir?** BSSID (Basic Service Set Identifier) — bir Wi-Fi giriş nöqtəsinin unikal MAC ünvanıdır. Hər router-in özünəməxsus bir "barmaq izi" kimidir.

### Addım 3b — SSID-i Tapmaq (Wigle.net)

[wigle.net](https://wigle.net) saytına daxil oluruq. Bu sayt dünya üzrə milyonlarla Wi-Fi şəbəkəsinin BSSID, SSID və coğrafi məkan məlumatlarını saxlayan nəhəng bir verilənlər bazasıdır.

**Qeyd:** Wigle.net indi qeydiyyat tələb edir. Yeni hesablar üçün gündə yalnız **5 ətraflı sorğu** icazəsi verilir. Hər sorğunu düşünərək istifadə edin!

1. Wigle.net-ə qeydiyyatdan keçirik
2. "Advanced Search" bölməsinə gedirik
3. **BSSID** sahəsinə `B4:5D:50:AA:86:41` daxil edirik
4. **Filter** düyməsinə basırıq
5. Nəticədə xəritəni tam uzaqlaşdırırıq
6. London ərazisinə keçirik — **qırmızı halqa** görünür
7. Tədricən yaxınlaşdırırıq — küçə səviyyəsindəki SSID görünür

**Cavab:** `UnileverWiFi` 📡

---

## ✅ Sual 4 — İstifadəçinin E-poçtu Nədir?

**Platforma:** GitHub  
**Metod:** `OWoodflint`-in GitHub README faylına baxırıq.

README-də e-poçt ünvanı açıq şəkildə yazılıb:

**Cavab:** `OWoodflint@gmail.com` 📧

---

## ✅ Sual 5 — E-poçtu Harada Tapdınız?

Əvvəlki suala əsasən cavab aydındır — e-poçtu GitHub README faylında tapdıq.

**Cavab:** `GitHub` 🐙

---

## ✅ Sual 6 — İstifadəçi Tətildə Haradadır?

**Platforma:** WordPress Blogu  
**Metod:** `OWoodflint`-in WordPress blogunu yoxlayırıq.

Blog yazısında istifadəçi tətil yerini açıqca qeyd edib — **New York**.

**Cavab:** `New York` 🗽

---

## ✅ Sual 7 — İstifadəçinin Parolu Nədir?

Bu sonuncu sual ən çətin olanıdır.

**Metod:**
1. X (Twitter) yoxlandı → ipucu yoxdur
2. GitHub yoxlandı → ipucu yoxdur
3. WordPress saytının **mənbə koduna** baxılır (Ctrl+U və ya sağ klik → "Page Source")

Mənbə kodunu diqqətlə araşdıranda gizli bir sətir tapılır — normal baxışda görünmür, amma HTML kodunda aşkar şəkildə yazılıb.

**Tapılan parol:**

```
pennYDr0pper.!
```

**Cavab:** `pennYDr0pper.!` 🔑

---

## 📊 Bütün Cavabların Xülasəsi

| # | Sual | Cavab | Mənbə |
|---|------|-------|-------|
| 1 | İstifadəçinin avatarı nədir? | `cat` | Twitter/X |
| 2 | İstifadəçi harada yaşayır? | `London` | GitHub |
| 3 | Qoşulduğu Wi-Fi-nın SSID-i? | `UnileverWiFi` | Twitter + Wigle.net |
| 4 | İstifadəçinin e-poçtu? | `OWoodflint@gmail.com` | GitHub |
| 5 | E-poçtu harada tapdınız? | `GitHub` | GitHub |
| 6 | Tətildə harada? | `New York` | WordPress |
| 7 | İstifadəçinin parolu? | `pennYDr0pper.!` | WordPress (mənbə kodu) |

---

## 🛠️ İstifadə Edilən Alətlər

| Alət | Məqsəd | Link |
|------|--------|------|
| **ExifTool** | Şəkil metadata-sını çıxarmaq | `sudo apt install exiftool` |
| **Google** | Ad üzrə axtarış aparmaq | google.com |
| **X (Twitter)** | Sosial media profili yoxlamaq | x.com |
| **GitHub** | README və e-poçt tapmaq | github.com |
| **Wigle.net** | BSSID → SSID çevirmək | wigle.net |
| **WordPress** | Blog yazıları + mənbə kodu | wordpress.com |

---

## 🎓 Öyrənilən Dərslər

1. **Metadata təhlükəlidir** — Şəkillər GPS koordinatları, müəllif adı, cihaz məlumatları kimi gizli məlumatlar saxlaya bilər. Şəkil paylaşmazdan əvvəl metadata-nı silin.

2. **Sosial media izi** — İnsanlar həm Twitter, həm GitHub, həm də blog yazılarında öz məlumatlarını şüursuz şəkildə açıqlaya bilər. Bu məlumatların birləşdirilməsi çox güclü profil yaradır.

3. **BSSID = Yer məlumatı** — Bir Wi-Fi nöqtəsinin BSSID-ini bilirsənsə, Wigle.net kimi alətlərlə həmin nöqtənin fiziki yerini tapa bilərsən.

4. **Mənbə kodu gizli məlumat saxlaya bilər** — Veb saytların HTML mənbə kodunda şərhlər (`<!-- -->`) içərisində ya da gizli elementlərdə həssas məlumatlar buraxıla bilər.

5. **OSINT = Parçaları birləşdirmək** — Heç bir məlumat tək başına böyük bir şey ifadə etmir. Amma bir neçə platformadakı məlumatları birləşdirdikdə tam bir şəkil ortaya çıxır.

---

## ⚠️ Hüquqi Xəbərdarlıq

Bu writeup yalnız **TryHackMe platformasındakı nəzərdə tutulmuş CTF tapşırığı** üçün hazırlanmışdır. Burada öyrənilən metodları real insanlara ya da sistemlərə icazəsiz şəkildə tətbiq etmək **qeyri-qanuni** və **etikaya ziddir**. OSINT bacarıqlarını həmişə qanuni çərçivədə və icazə alınaraq istifadə edin.

---

*Writeup hazırladı: Azərbaycan dilli TryHackMe icması üçün*  
*Əsas mənbə: [Sunjid Ahmed Siyem — Medium](https://medium.com/@sunjid-ahmed/ohsint-tryhackme-walkthrough-69870e899fc7)*
