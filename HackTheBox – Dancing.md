
## 1. Descoberta de la màquina

- **ICMP/Ping**:
    
    - Comanda: `ping 10.129.166.154`
        
    - Informació obtinguda:
        
        - **TTL = 127** → pista sobre sistema operatiu (Windows típic TTL inicial 128).
            
        - **Latència** → informació sobre la distància o presència de VPN/NAT.
            
    - Ús: Confirmar host actiu i obtenir indicis del SO.
        
- **Traceroute** (opcional): verificar salts i confirmar TTL efectiu per fingerprinting de la xarxa.
    

---

## 2. Enumeració de serveis

- **SMB (Server Message Block)**:
    
    - Comanda: `smbclient -L //10.129.166.154 -N`
        
    - Obté llista de comparticions disponibles.
        
    - Sortida típica:
        
        `ADMIN$          Disk      Remote Admin C$              Disk      Default share IPC$            IPC       Remote IPC WorkShares      Disk`
        
    - Notes:
        
        - Comparticions administratives (`ADMIN$`, `C$`) → accés restringit.
            
        - `IPC$` → named pipes, usat per RPC/gestió remota.
            
        - Compartició normal accessible (`WorkShares`) → potencial vector d’informació pública o amb credencials en blanc.
            
- **SMB Versions**:
    
    - Missatge `SMB1 disabled` → el servidor usa SMB2/SMB3, millor seguretat, però encara permet accés a comparticions legals.
        
    - Permet enumeració de carpetes i fitxers sense habilitar SMBv1.
        

---

## 3. Accés a comparticions

- **Prompt smbclient**: `smb: \>`
    
- Comandes útils:
    
    - `ls` → llista fitxers/directoris remots.
        
    - `cd <carpeta>` → canviar directori remot.
        
    - `get <fitxer>` → descarregar fitxer individual.
        
    - `mget *` → descarregar múltiples fitxers.
        
    - `lcd <carpeta_local>` → canviar directori local on es guardaran fitxers.
        
    - `recurse ON` i `prompt OFF` → descarregar carpetes recursivament sense preguntar.
        
- Exemple de carpetes dins de `WorkShares`: `Amy.J`, `James.P`.
    

---

## 4. Fitxers i informació

- Continguts de fitxers poden donar **pistes sobre serveis i vulnerabilitats**:
    
    - Exemples: notes sobre Apache, FTP, WinRM.
        
    - Indicacions sobre serveis actius i configuracions a reforçar (ex. WinRM habilitat en Windows → administració remota possible).
        

---

## 5. Vulnerabilitats i vectors principals

- **SMB**:
    
    - Accés a comparticions sense contrasenya (`-N`) → configuració insegura habitual.
        
    - Carpeta `WorkShares` accessible → vector per enumeració i extracció de fitxers sensibles.
        
- **WinRM**:
    
    - Presència indicada en fitxers de notes → possible target de gestió remota en entorns Windows.
        
    - Protocol per executar comandes remotament, habitual en proves de pentesting Windows.
        
- **FTP / Apache (Linux)**:
    
    - Referenciat en fitxers de notes → mostra la importància de revisar serveis exposats.
        
    - Pot ser vector d’exfiltració o prova de configuració insegura en entorns combinats Windows/Linux.
        

---

## 6. Bones pràctiques de pentesting amb SMB

- Sempre enumerar comparticions amb: `smbclient -L //IP -N`
    
- Comprovar serveis amb nmap:
    
    `nmap -p 139,445 --script smb-enum-shares,smb-os-discovery IP`
    
- Analitzar fitxers amb cura, buscar credencials, notes o configuracions insegures.
    
- Recordar que comparticions administratives (`C$`, `ADMIN$`) normalment requereixen credencials vàlides.
    

---

## 7. Resum tècnic del flux

1. **Descoberta host** → ping / TTL → pista SO.
    
2. **Enumeració SMB** → `smbclient -L` → llista comparticions.
    
3. **Accés compartició pública** → `WorkShares` → llistar carpetes, fitxers.
    
4. **Descarregar fitxers** → `get` / `mget` → analitzar informació.
    
5. **Interpretar fitxers** → pistes sobre serveis actius (WinRM, FTP, Apache).

[[Pentesting]]
