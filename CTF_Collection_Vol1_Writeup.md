# TryHackMe — CTF Collection Vol.1 Write-up

**Otaq:** [CTF collection Vol.1](https://tryhackme.com/room/ctfcollectionvol1)  
**Çətinlik:** Easy  
**Mövzular:** Encoding, Steganography, Cryptography, OSINT, Network Analysis  
**Ümumi tapşırıq sayı:** 20 (+ 1 giriş tapşırığı)

---

## Giriş

Bu otaq CTF bacarıqlarını inkişaf etdirmək üçün DesKel tərəfindən hazırlanmış yeni başlayanlar üçün əla bir kolleksiyadır. Bütün flag-lər `THM{flag}` formatındadır (ayrıca qeyd edilmədikcə). Sakit olun və flag-ləri tutun!

---

## Task 1 — Author note

Giriş tapşırığıdır. Sadəcə "Complete" düyməsini basın.

---

## Task 2 — What does the base said?

**Mövzu:** Base64 Encoding

**İzah:**  
Verilmiş sətrin görünüşünə baxın — sonda `==` işarəsi var. Bu, **Base64** şifrələməsinin əlamətidir. Base64, məlumatları ASCII simvollarına çevirən kodlaşdırma üsuludur.

**İstifadə ediləcək alət:** `base64` (terminal) və ya CyberChef

**Qayda:**
1. Verilmiş mətni götürün
2. Terminalda `echo "..." | base64 -d` əmrini işlədin
3. Nəticə birbaşa flag-dir

**Əlavə məlumat:** Base64 əlifbası A-Z, a-z, 0-9, +, / simvollarından ibarətdir. Sonda `=` işarəsi padding deməkdir.

---

## Task 3 — Meta meta

**Mövzu:** Metadata / EXIF məlumatları

**İzah:**  
Şəkil faylları içərisində gizli metadata saxlayır — müəllif adı, GPS koordinatları, kamera modeli, proqram adı və s. Bu tapşırıqda flag şəklin metadata-sında gizlədilmişdir.

**İstifadə ediləcək alət:** `exiftool`

**Qayda:**
1. Şəkil faylını yükləyin
2. `exiftool fayl.jpg` əmrini işlədin
3. Çıxan məlumatlar arasında flag-i axtarın (Owner, Comment, Description kimi sahələrə diqqət edin)

**Əlavə məlumat:** `exiftool` demək olar ki, bütün fayl formatlarının metadata-sını oxuya bilər.

---

## Task 4 — Mon, are we going to be okay?

**Mövzu:** Steganography — Steghide

**İzah:**  
Steganografiya məlumatı başqa bir məlumatın içinə gizlətmək sənətidir. Bu tapşırıqda flag şəkil faylının içinə `steghide` aləti ilə gizlədilmişdir.

**İstifadə ediləcək alət:** `steghide`

**Qayda:**
1. Şəkil faylını yükləyin
2. `steghide extract -sf fayl.jpg` əmrini işlədin
3. Parol soruşulursa, boş buraxıb Enter basın (bəzən parol yoxdur)
4. Çıxarılan faylı oxuyun

**Əlavə məlumat:** Steghide yalnız JPEG, BMP, WAV, AU formatlarını dəstəkləyir.

---

## Task 5 — Erm……Magick

**Mövzu:** Gizli mətn — HTML / Vizual aldatma

**İzah:**  
Bu tapşırıqda flag göz önündədir, sadəcə görünmür. Mətn ağ rənglə yazılıb (ağ fon üzərində), yaxud HTML kodunda gizlədilmişdir.

**Qayda:**
1. Səhifəni açın
2. Bütün mətni seçin (Ctrl+A) — gizli mətn görünə bilər
3. Yaxud sağ klik → "View Page Source" ilə HTML koduna baxın
4. Flag orada tapılacaq

**Əlavə məlumat:** CTF-lərdə mətn rəngini fon rəngi ilə eyni etmək çox geniş yayılmış gizlətmə üsuludur.

---

## Task 6 — QRrrrr

**Mövzu:** QR Kod

**İzah:**  
Sadəcə verilmiş QR kodu oxumaq lazımdır.

**İstifadə ediləcək alətlər:**
- Telefon kamerası / QR oxuyucu proqram
- Terminal: `zbarimg qr.png`
- Online: [zxing.org](https://zxing.org/w/decode.jspx)

**Qayda:**
1. QR kodu yükləyin
2. Yuxarıdakı alətlərdən biri ilə oxuyun
3. Nəticə flag-dir

---

## Task 7 — Reverse it or read it?

**Mövzu:** Strings / Fayl analizi

**İzah:**  
Verilmiş faylın içindəki mətnləri oxumaq lazımdır. İkili faylların içindəki oxunaqlı mətn parçalarını `strings` aləti tapır.

**Qayda:**
1. Faylı yükləyin
2. `strings fayl` əmrini işlədin
3. Çıxan mətnlər arasında `THM{` formatını axtarın

**Alternativ:** Fayl bir skript ola bilər — o zaman sadəcə `cat fayl` ilə oxuyun.

---

## Task 8 — Another decoding stuff

**Mövzu:** Base58 Encoding

**İzah:**  
Bu dəfə Base64 yox, **Base58** kodlaşdırması istifadə edilmişdir. Base58, Bitcoin ünvanlarında istifadə olunan, qarışıq görünən simvolları (0, O, l, I) çıxarılmış Base64 variantıdır.

**İstifadə ediləcək alət:** CyberChef və ya Python

**Qayda:**
1. Verilmiş mətni götürün
2. CyberChef-də "From Base58" recipe-sini istifadə edin
3. Yaxud terminalda: `python3 -c "import base58; print(base58.b58decode('...'))"`

---

## Task 9 — Left or right?

**Mövzu:** Sezar / ROT Şifrəsi

**İzah:**  
Klassik yer dəyişdirmə şifrəsidir. Hər hərf əlifbada müəyyən sayda sola/sağa sürüşdürülür. Ən məşhur variant ROT13-dür (13 mövqe).

**İstifadə ediləcək alət:** CyberChef — "ROT13" və ya "Caesar Cipher Decode"

**Qayda:**
1. Şifrəli mətni götürün
2. CyberChef-də ROT13 cəhd edin
3. İşləməsə, 1-dən 25-ə qədər bütün shift-ləri cəhd edin
4. `THM{` ilə başlayan nəticə düzgün cavabdır

---

## Task 10 — Make a comment

**Mövzu:** HTML Şərhi (Comment)

**İzah:**  
Veb səhifələrin HTML kodunda `<!-- şərh -->` formatında gizli qeydlər ola bilər. Bu şərhlər brauzer tərəfindən göstərilmir.

**Qayda:**
1. Tapşırıq səhifəsini açın
2. Sağ klik → "View Page Source"
3. `<!--` işarəsini axtarın
4. Flag orada gizlədilmişdir

---

## Task 11 — Can you fix it?

**Mövzu:** PNG Başlıq Düzəltməsi

**İzah:**  
PNG faylının başlığı (magic bytes) pozulmuşdur, ona görə açılmır. PNG faylları `\x89PNG` magic bytes ilə başlamalıdır.

**Qayda:**
1. Faylı hex redaktoru ilə açın (`hexedit` və ya `xxd`)
2. Faylın əvvəlini yoxlayın — ilk 8 bayt düzgün PNG imzası olmalıdır: `89 50 4E 47 0D 0A 1A 0A`
3. Yanlış baytları düzəldin
4. Faylı yadda saxlayın və açın — şəkildə flag görünəcək

---

## Task 12 — Read it

**Mövzu:** OSINT — Reddit

**İzah:**  
Flag internet üzərindəki açıq bir mənbədə — Reddit-də — paylaşılmışdır. OSINT (Open Source Intelligence) açıq mənbələrdən məlumat toplamaq deməkdir.

**Qayda:**
1. Reddit-ə gedin: [reddit.com/r/tryhackme](https://reddit.com/r/tryhackme)
2. DesKel-in postlarını axtarın
3. CTF Collection Vol.1 ilə bağlı postda flag tapılacaq

---

## Task 13 — Spin my head

**Mövzu:** Brainfuck

**İzah:**  
Brainfuck, `+`, `-`, `>`, `<`, `[`, `]`, `.`, `,` simvollarından ibarət ekzotik bir proqramlaşdırma dilidir. Görünüşü çaşdırıcı olsa da, online interpreter ilə asanlıqla işlədilir.

**İstifadə ediləcək alət:** [dcode.fr/brainfuck-language](https://www.dcode.fr/brainfuck-language) və ya Python interpreter

**Qayda:**
1. Brainfuck kodunu götürün
2. Online interpreter-ə yapışdırın
3. "Run" düyməsini basın — nəticə flag-dir

---

## Task 14 — An exclusive!

**Mövzu:** XOR Şifrəsi

**İzah:**  
XOR (Exclusive OR) məntiqi əməliyyatıdır. İki hex dəyərin XOR-u flag-i verir. Əgər `S1 XOR S2 = flag` isə, `flag XOR S2 = S1` da doğrudur.

**Qayda:**
1. Verilmiş iki hex sətri götürün
2. Python ilə: `hex(int(s1, 16) ^ int(s2, 16))`
3. Nəticəni hex-dən ASCII-yə çevirin

---

## Task 15 — Binary walk

**Mövzu:** Steganography — Binwalk

**İzah:**  
Binwalk fayl içinə gömülmüş başqa faylları aşkar edir. Bir JPEG şəklinin içinə ZIP arxivi gizlədilə bilər.

**Qayda:**
1. Faylı yükləyin
2. `binwalk fayl.jpg` əmri ilə içini yoxlayın
3. Gizli fayl tapılarsa, `unzip fayl.jpg` ilə çıxarın
4. Çıxarılan faylı oxuyun

**Qeyd:** `binwalk -e` root icazəsi tələb edə bilər — belə halda birbaşa `unzip` istifadə edin.

---

## Task 16 — Darkness

**Mövzu:** Steganography — Stegsolve / Rəng kanalları

**İzah:**  
Şəkildə flag LSB (Least Significant Bit) steganografiyası ilə gizlədilmişdir. Stegsolve müxtəlif rəng kanallarını ayrıca göstərir.

**İstifadə ediləcək alətlər:**
- `stegsolve` (Java): `java -jar stegsolve.jar`
- `zsteg fayl.png`

**Qayda:**
1. Şəkili Stegsolve-da açın
2. Sol/sağ oxlarla rəng kanalları arasında keçid edin
3. Bir kanalda gizli mətn və ya flag görünəcək

---

## Task 17 — A sounding QR

**Mövzu:** Audio Steganography — Spektrogram

**İzah:**  
Səs faylının spektrogramında (vizual təsviri) flag yazılmışdır. Bu texnikada məlumat səs dalğalarının görünüşünə kodlanır.

**İstifadə ediləcək alətlər:**
- Audacity: Track → Spectrogram
- Online: [dcode.fr/spectral-analysis](https://www.dcode.fr/spectral-analysis)
- Terminal: `sox audio.mp3 -n spectrogram -o spec.png`

**Qayda:**
1. Səs faylını yükləyin
2. Spektrogram rejimini açın
3. Şəkildə flag-i oxuyun

---

## Task 18 — Dig up the past

**Mövzu:** OSINT — Wayback Machine

**İzah:**  
[web.archive.org](https://web.archive.org) — internetin "tarix kitabı"dır. Silinmiş və ya dəyişdirilmiş saytların köhnə versiyalarını saxlayır.

**Qayda:**
1. [web.archive.org](https://web.archive.org) saytına gedin
2. Tapşırıqda verilmiş URL-i axtarın
3. Tapşırıqda göstərilən tarixdəki arxiv versiyasını açın
4. Həmin versiyada flag tapılacaq

---

## Task 19 — Uncrackable!

**Mövzu:** Vigenere Şifrəsi

**İzah:**  
Vigenere şifrəsi açar söz istifadə edən polyalphabetic şifrədir. Açar sözü bilmədən açmaq çətindir, amma flag formatı (`TRYHACKME{...}`) bilinərsə, known-plaintext attack ilə açar tapıla bilər.

**Qayda:**
1. Şifrəli mətni götürün
2. Flag formatı `TRYHACKME{...}` olduğunu bilin
3. Şifrəli mətnin ilk 9 simvolu ilə `TRYHACKME` arasındakı fərqi hesablayın → açar tapılır
4. Tapılmış açarla bütün mətni deşifrə edin

**Python ilə:**
```python
cipher = "ŞIFRELI_METN"
known = "TRYHACKME"
key = ""
for i in range(len(known)):
    k = (ord(cipher[i]) - ord(known[i])) % 26
    key += chr(k + ord('A'))
# Sonra bu açarla tam mətni açın
```

---

## Task 20 — Small bases

**Mövzu:** Decimal → Hex → ASCII

**İzah:**  
Böyük onluq (decimal) ədədi ardıcıl olaraq əvvəlcə hex-ə, sonra ASCII-yə çevirmək lazımdır.

**Qayda:**
1. Verilmiş böyük ədədi götürün
2. Hex-ə çevirin: `hex(ədəd)`
3. Hex-dən ASCII-yə: `bytes.fromhex(hex_dəyər).decode()`

**Python ilə:**
```python
n = VERİLMİŞ_ƏDƏD
print(bytes.fromhex(hex(n)[2:]).decode())
```

---

## Task 21 — Read the packet

**Mövzu:** Network Analysis — Wireshark

**İzah:**  
`.pcap` / `.pcapng` faylları şəbəkə paketlərini saxlayır. Wireshark bu paketləri analiz etməyə imkan verir.

**Qayda:**
1. Faylı Wireshark-da açın
2. Filter çubuğuna `http contains "flag"` yazın
3. Tapılan paketi seçin
4. Sağ klik → **Follow → HTTP Stream** seçin
5. Flag orada görünəcək

**Alternativ filter-lər:**
```
http
tcp contains "flag"
http.request
```

---

## Nəticə

CTF Collection Vol.1 aşağıdakı mövzuları əhatə edir:

| Mövzu | Task-lar |
|---|---|
| Encoding (Base64, Base58) | 2, 8 |
| Metadata / EXIF | 3 |
| Steganography | 4, 15, 16, 17 |
| Kriptografiya (Sezar, Vigenere, XOR) | 9, 14, 19 |
| HTML / Veb | 5, 10 |
| QR Kod | 6, 17 |
| OSINT | 12, 18 |
| Fayl analizi | 7, 11 |
| Brainfuck | 13 |
| Riyaziyyat (Decimal/Hex) | 20 |
| Şəbəkə analizi | 21 |

Bu otağı tamamladıqdan sonra **CTF Collection Vol.2**-yə keçməyi tövsiyə edirəm.

---

*Write-up hazırlandı: CTF öyrənmə məqsədilə. Flag-lər qəsdən yazılmamışdır.*
