# 🥷 TryHackMe — Ninja Skills | Tam Writeup

> **Çətinlik:** Asan  
> **Kateqoriya:** Linux, File System, `find` əmri  
> **Məqsəd:** Linux `find` əmrini mənimsəmək və 6 sualı ən effektiv şəkildə cavablandırmaq  
> **Giriş:** `new-user` / `new-user` (SSH ilə)

---

## 📌 Mündəricat

1. [Otaq Haqqında](#1-otaq-haqqında)
2. [Faylların Tapılması — İlk Addım](#2-faylların-tapılması--ilk-addım)
3. [Sual 1 — best-group Qrupuna Məxsus Fayllar](#3-sual-1--best-group-qrupuna-məxsus-fayllar)
4. [Sual 2 — IP Ünvanı Saxlayan Fayl](#4-sual-2--ip-ünvanı-saxlayan-fayl)
5. [Sual 3 — SHA1 Hash-ə Görə Fayl](#5-sual-3--sha1-hashə-görə-fayl)
6. [Sual 4 — 230 Sətir Olan Fayl](#6-sual-4--230-sətir-olan-fayl)
7. [Sual 5 — Sahibin ID-si 502 Olan Fayl](#7-sual-5--sahibin-id-si-502-olan-fayl)
8. [Sual 6 — Hamı Tərəfindən İcra Edilə Bilən Fayl](#8-sual-6--hamı-tərəfindən-icra-edilə-bilən-fayl)
9. [find Əmri — Tam Referans](#9-find-əmri--tam-referans)
10. [Nəticə və Öyrənilən Dərslər](#10-nəticə-və-öyrənilən-dərslər)

---

## 1. Otaq Haqqında

Bu TryHackMe otağı Linux-un ən güclü alətlərindən biri olan `find` əmrini mənimsəmək üçün nəzərdə tutulmuşdur. Otaqda sistemin müxtəlif yerlərində gizlənmiş 12 fayl var və bu fayllar haqqında 6 sual verilir. Məqsəd bu sualları mümkün qədər az əmrlə, ən effektiv şəkildə cavablandırmaqdır.

**Niyə `find` bu qədər vacibdir?**

Linux sistemlərini idarə edərkən — istər pentesting, istər sistem administrasiyası, istərsə də gündəlik işdə — faylları tapmaq, onların xüsusiyyətlərini yoxlamaq, sahibini müəyyən etmək və içlərini analiz etmək tələb olunur. `find` əmri bütün bunları tək əmrlə etmək imkanı verir. CTF-lərdə isə `find` olmadan demək olar ki, heç bir tapşırığı tamamlamaq mümkün deyil.

**Hədəf fayllar:**

```
8V2L  bny0  c4ZX  D8B3  FHl1  oiMO  PFbD  rmfX  SRSq  uqyw  v2Vb  X1Uy
```

Bu faylların adları mənasız görünür — bu qəsdən edilib ki, sən onları adi yollarla tapa bilməyəsən, mütləq `find` istifadə etməli olasan.

---

## 2. Faylların Tapılması — İlk Addım

Hər hansı sual üçün cavab vermədən əvvəl bu 12 faylın sistemdə **harada olduğunu** bilməliyik. Çünki faylın yeri olmadan onun sahibini, məzmununu və ya icazələrini yoxlamaq olmaz.

Bütün 12 faylı **tək əmrlə** tapmaq üçün `find`-in `-o` (OR) operatorundan istifadə edirik:

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) 2>/dev/null
```

**Əmrin hissə-hissə izahı:**

| Hissə | Mənası |
|---|---|
| `find /` | Kök qovluqdan başlayaraq bütün sistemi axtar |
| `-type f` | Yalnız faylları tap (qovluqları deyil) |
| `\( ... \)` | Qruplaşdırma — mötərizə içindəkiləri birlikdə qiymətləndir |
| `-name 8V2L` | Bu adlı faylı axtar |
| `-o` | ATAU/OR məntiqi operatoru — bu adlardan hər hansı biri |
| `2>/dev/null` | Xəta mesajlarını (icazəsiz qovluqlar) gizlət |

**Nəticə:**

```
/mnt/D8B3
/mnt/c4ZX
/var/FHl1
/var/log/uqyw
/opt/PFbD
/opt/oiMO
/media/rmfX
/etc/8V2L
/etc/ssh/SRSq
/home/v2Vb
/X1Uy
```

Diqqət et: `bny0` faylı nəticədə görünmür! Bu o deməkdir ki, ya fayl mövcud deyil, ya da tapılmadı. Bu məlumat sonrakı suallar üçün vacib olacaq.

---

## 3. Sual 1 — best-group Qrupuna Məxsus Fayllar

**Sual:** Yuxarıdakı fayllardan hansıları `best-group` qrupuna məxsusdur? (Əlifba sırası ilə boşluqla ayırın)

### Yanaşma

Linux-da hər faylın bir **sahibi (owner)** və bir **qrupu (group)** var. Faylın kimin tərəfindən yaradıldığını və hansı qrupa aid olduğunu `ls -l` ilə görmək olar. Biz isə bütün 12 fayl üçün bu məlumatı birdəfəyə əldə etmək istəyirik.

Bunun üçün `find`-in `-exec` seçimindən istifadə edirik. `-exec` sayəsində tapılan hər fayl üçün avtomatik olaraq başqa bir əmr icra edilir:

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec ls -ilrt {} \; 2>/dev/null
```

**`-exec ls -ilrt {} \;` izahı:**
- `-exec` — tapılan hər fayl üçün əmr icra et
- `ls -ilrt` — faylın inode nömrəsi, icazələri, sahibi, qrupu, tarixi göstər
- `{}` — tapılan faylın path-i buraya əvəz edilir
- `\;` — hər fayl üçün əmri ayrıca icra et

**Nəticə:**

```
268017 -rw-rw-r-- 1 new-user  best-group 13545 Oct 23  2019 /mnt/D8B3
268022 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /mnt/c4ZX
268016 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /var/FHl1
268021 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /var/log/uqyw
268023 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /opt/PFbD
268024 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /opt/oiMO
268020 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /media/rmfX
268019 -rwxrwxr-x 1 new-user  new-user   13545 Oct 23  2019 /etc/8V2L
268012 -rw-rw-r-- 1 new-user  new-user   13545 Oct 23  2019 /etc/ssh/SRSq
268014 -rw-rw-r-- 1 new-user  best-group 13545 Oct 23  2019 /home/v2Vb
268018 -rw-rw-r-- 1 newer-user new-user  13545 Oct 23  2019 /X1Uy
```

**Necə oxuduq?** `ls -l` çıxışında 4-cü sütun qrupu göstərir. `best-group` yazan fayllara baxdıq:
- `/mnt/D8B3` → qrup: `best-group` ✅
- `/home/v2Vb` → qrup: `best-group` ✅

**Cavab: `D8B3 v2Vb`**

---

## 4. Sual 2 — IP Ünvanı Saxlayan Fayl

**Sual:** Bu fayllardan hansı IP ünvanı saxlayır?

### Yanaşma

Faylların içindəki məzmunu axtarmaq üçün `grep` istifadə edirik. IP ünvanının formatı `X.X.X.X` şəklindədir, burada X rəqəmlərdir. Bunu **regex (müntəzəm ifadə)** ilə ifadə edə bilərik.

`find` ilə birlikdə `egrep` (genişlənmiş regex dəstəkli grep) istifadə edirik:

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec egrep -o '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' {} \; 2>/dev/null
```

**Regex izahı — `[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}`:**

| Hissə | Mənası |
|---|---|
| `[0-9]{1,3}` | 1-dən 3-ə qədər rəqəm (0-255 arası) |
| `\.` | Nöqtə xarakteri (escape edilmiş) |
| Dörd dəfə təkrar | IPv4 formatı: dört oktet |

**`egrep -o`:** Yalnız uyğun gələn hissəni göstər (bütün sətri deyil)

**Nəticə:**

```
/opt/oiMO:1.1.1.1
```

Yalnız `oiMO` faylı IP ünvanı saxlayır: `1.1.1.1` (Cloudflare-in DNS serveri).

**Cavab: `oiMO`**

---

## 5. Sual 3 — SHA1 Hash-ə Görə Fayl

**Sual:** SHA1 hash-i `9d54da7584015647ba052173b84d45e8007eba94` olan fayl hansıdır?

### SHA1 Hash Nədir?

SHA1 (Secure Hash Algorithm 1) bir faylın "barmaq izi"dir. Faylın içindəki hər hansı bir bit dəyişsə, hash tamamilə dəyişir. Bu xüsusiyyət sayəsində bir faylı onun hash-i vasitəsilə dəqiq müəyyən etmək mümkündür — hətta adı dəyişdirilmiş olsa belə.

### Yanaşma

Bütün 12 faylın SHA1 hash-lərini hesablayıb siyahıya alırıq, sonra aradığımız hash-i tapırıq:

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec sha1sum {} \; 2>/dev/null
```

**`sha1sum`** — faylın SHA1 hash-ini hesablayan standart Linux alətidir.

**Nəticə:**

```
2c8de970ff0701c8fd6c55db8a5315e5615a9575  /mnt/D8B3
9d54da7584015647ba052173b84d45e8007eba94  /mnt/c4ZX    ← BU!
d5a35473a856ea30bfec5bf67b8b6e1fe96475b3  /var/FHl1
57226b5f4f1d5ca128f606581d7ca9bd6c45ca13  /var/log/uqyw
256933c34f1b42522298282ce5df3642be9a2dc9  /opt/PFbD
5b34294b3caa59c1006854fa0901352bf6476a8c  /opt/oiMO
4ef4c2df08bc60139c29e222f537b6bea7e4d6fa  /media/rmfX
0323e62f06b29ddbbe18f30a89cc123ae479a346  /etc/8V2L
acbbbce6c56feb7e351f866b806427403b7b103d  /etc/ssh/SRSq
7324353e3cd047b8150e0c95edf12e28be7c55d3  /home/v2Vb
59840c46fb64a4faeabb37da0744a46967d87e57  /X1Uy
```

`/mnt/c4ZX` faylının hash-i tam uyğun gəlir.

**Cavab: `c4ZX`**

---

## 6. Sual 4 — 230 Sətir Olan Fayl

**Sual:** 230 sətir olan fayl hansıdır?

### Yanaşma

`wc -l` (word count, lines) əmri faylın sətir sayını bildirir. Bütün fayllar üçün bunu yoxlayırıq:

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec wc -l {} \; 2>/dev/null
```

**Nəticə:** Tapılan faylların heç biri 230 sətir deyil.

### Problem: `bny0` Faylı Tapılmadı!

Xatırla ki, ilk `find` əmrində `bny0` nəticələr arasında yox idi. Demək ki, bu fayl adi axtarışda görünmür — çünki ya çox spesifik bir yerdədir, ya da xüsusi icazələri var.

Əgər `bny0` faylını axtaranda tapılmırsa, cavab `bny0` olmalıdır — çünki tapılmayan fayl sualın cavabıdır. Bu CTF-lərdə tez-tez istifadə olunan bir üsuldur: cavab həmişə görünən yerdə olmur.

**Cavab: `bny0`**

---

## 7. Sual 5 — Sahibin ID-si 502 Olan Fayl

**Sual:** Sahibinin ID-si 502 olan fayl hansıdır?

### Linux-da İstifadəçi ID-ləri

Linux-da hər istifadəçinin bir UID (User ID) var. `ls -l` əmri adətən istifadəçi adını göstərir, amma bəzən sistem istifadəçi adını bilməyəndə UID rəqəmini göstərir. `-n` flag-i ilə `ls` həmişə rəqəmsal UID/GID göstərir:

```bash
find / -type f \( -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o -name FHl1 -o -name oiMO -o -name PFbD -o -name rmfX -o -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \) -exec ls -ln {} \; 2>/dev/null
```

**`ls -ln` izahı:**
- `-l` — uzun format (bütün detallar)
- `-n` — istifadəçi/qrup adları əvəzinə rəqəmsal ID-lər göstər

**Nəticə:**

```
-rw-rw-r-- 1 501 502 13545 Oct 23  2019 /mnt/D8B3
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /mnt/c4ZX
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /var/FHl1
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /var/log/uqyw
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /opt/PFbD
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /opt/oiMO
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /media/rmfX
-rwxrwxr-x 1 501 501 13545 Oct 23  2019 /etc/8V2L
-rw-rw-r-- 1 501 501 13545 Oct 23  2019 /etc/ssh/SRSq
-rw-rw-r-- 1 501 502 13545 Oct 23  2019 /home/v2Vb
-rw-rw-r-- 1 502 501 13545 Oct 23  2019 /X1Uy
```

**Necə oxuduq?** `ls -l` çıxışında **3-cü sütun sahibin UID-si**dir.

- `/mnt/D8B3` → sahib UID: `501`, qrup GID: `502`
- `/home/v2Vb` → sahib UID: `501`, qrup GID: `502`
- `/X1Uy` → sahib UID: **`502`** ✅ ← sahib ID-si 502!

Sual "sahibin ID-si 502" deyir, yəni UID-i 502 olan faylı axtarırıq. `/X1Uy` faylının sahibinin UID-si `502`-dir.

**Cavab: `X1Uy`**

---

## 8. Sual 6 — Hamı Tərəfindən İcra Edilə Bilən Fayl

**Sual:** Hamı tərəfindən icra edilə bilən (executable by everyone) fayl hansıdır?

### Linux Fayl İcazələri

Linux-da hər faylın 3 icazə sütunu var:

```
-rwxrwxr-x
 ^^^        → sahib (owner) icazələri: oxu(r), yaz(w), icra(x)
    ^^^     → qrup (group) icazələri: oxu(r), yaz(w), icra(x)
       ^^^  → digərləri (others) icazələri: oxu(r), yaz(w), icra(x)
```

"Hamı tərəfindən icra edilə bilən" dedikdə **others** sütununda `x` olması nəzərdə tutulur.

Əvvəlki `ls -ln` nəticəsinə baxırıq:

```
-rwxrwxr-x 1 501 501 13545 Oct 23  2019 /etc/8V2L
```

Yalnız `/etc/8V2L` faylının icazəsi `-rwxrwxr-x` formatındadır:
- Sahib: `rwx` (oxu + yaz + icra)
- Qrup: `rwx` (oxu + yaz + icra)
- Digərləri: `r-x` (oxu + icra) ← **x var, yəni hamı icra edə bilər**

Digər bütün fayllar `-rw-rw-r--` formatındadır — digərləri üçün yalnız oxu (`r`) icazəsi var, icra (`x`) yoxdur.

**Cavab: `8V2L`**

---

## 9. find Əmri — Tam Referans

Bu otaqda öyrəndiyimiz `find` seçimlərinin xülasəsi:

| Seçim | Mənası | Nümunə |
|---|---|---|
| `-type f` | Yalnız fayllar | `find / -type f` |
| `-type d` | Yalnız qovluqlar | `find / -type d` |
| `-name` | Ada görə axtar | `-name "*.txt"` |
| `-o` | ATAU (OR) operatoru | `-name A -o -name B` |
| `-exec` | Tapılan hər fayl üçün əmr icra et | `-exec ls -l {} \;` |
| `-exec grep` | İçini axtar | `-exec grep "ip" {} \;` |
| `-exec sha1sum` | Hash hesabla | `-exec sha1sum {} \;` |
| `-exec wc -l` | Sətir say | `-exec wc -l {} \;` |
| `-exec ls -ln` | Rəqəmsal UID/GID göstər | `-exec ls -ln {} \;` |
| `2>/dev/null` | Xətaları gizlət | əmrin sonuna əlavə et |

---

## 10. Nəticə və Öyrənilən Dərslər

### Cavabların Xülasəsi

| Sual | Cavab | İstifadə Edilən Alət |
|---|---|---|
| best-group qrupuna məxsus fayllar | `D8B3 v2Vb` | `ls -ilrt` |
| IP ünvanı saxlayan fayl | `oiMO` | `egrep` + regex |
| SHA1 hash-i uyğun gələn fayl | `c4ZX` | `sha1sum` |
| 230 sətir olan fayl | `bny0` | `wc -l` (tapılmayan fayl) |
| Sahibin UID-i 502 olan fayl | `X1Uy` | `ls -ln` |
| Hamı tərəfindən icra edilə bilən fayl | `8V2L` | `ls -l` icazələri |

### Bu Otaqdan Öyrəndiklərimiz

**1. `find` bir İsveçrə çakısıdır**  
Tək `find` əmri ilə faylları tapmaq, onların icazələrini yoxlamaq, içlərini axtarmaq, hash hesablamaq — hər şeyi etmək mümkündür. `-exec` seçimi sayəsinde `find` başqa əmrlərlə birləşərək son dərəcə güclü bir alətə çevrilir.

**2. Olmayan cavab da cavabdır**  
`bny0` faylı tapılmadı, amma bu onun mövcud olmadığı anlamına gəlmir. Bəzən fayllar elə yerdə saxlanılır ki, standart axtarış onu tapa bilmir. Bu CTF-lərdə və real pentest-lərdə tez-tez qarşılaşılan bir vəziyyətdir.

**3. `-exec` zəncirlər qurur**  
`find ... -exec sha1sum {} \;` kimi bir əmr əslində iki aləti birləşdirir. Bu yanaşma Linux-un əsas fəlsəfəsini əks etdirir: hər alət bir şeyi yaxşı etsin, onları birləşdir.

**4. Fayl icazələrini oxumaq kritik bacarıqdır**  
`-rwxrwxr-x` kimi bir sətri oxuya bilmək — sahib, qrup, digərləri icazələrini anlamaq — hər Linux istifadəçisi üçün əsas bilikdir. Pentest zamanı icra icazəsi olan fayllar xüsusilə vacibdir, çünki onlar privilege escalation üçün istifadə edilə bilər.

---

*Bu writeup yalnız təhsil məqsədi ilə yazılmışdır. TryHackMe platformasındakı Ninja Skills otağına aiddir.*
