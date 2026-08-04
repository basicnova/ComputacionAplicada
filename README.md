# ComputacionAplicada
Trabajo práctico integrador grupal

# TP - Configuración de Servidor Debian (TPServer)

Guía de armado y configuración de una máquina virtual Debian con SSH, Apache+PHP, MariaDB, particionado de disco y backups automáticos.

---

## 1) Configuración del entorno

### 1.1 Descarga y armado de la VM
- Los archivos `.rar` de la VM (partidos en varias partes) se descargan de Blackboard.
- **Extraer `part01.rar` con 7-Zip** (con WinRAR suele tirar error de EOF).
- Ejecutar el `.ova` para importar la VM.
- Verificar que la Red esté en modo **Adaptador Puente**, en la placa de red correcta.

### 1.2 Blanquear contraseña de root
1. Al bootear, en el GRUB apretar `e` para editar los parámetros de arranque.
2. Modificar la línea de arranque: cambiar `ro` por `rw` y agregar `init=/bin/bash`.
   - Quedaría algo como: `rw init=/bin/bash`
   - Opcional: sacar `quiet` para ver más logs.
3. `Ctrl+X` para bootear con esos parámetros → entra en una consola limitada.
4. Cambiar la contraseña de root:
   ```bash
   passwd root
   ```
   Nueva contraseña: `palermo`
5. Reiniciar y loguearse normal:
   - Usuario: `root`
   - Contraseña: `palermo`

### 1.3 Configurar hostname
No usar `hostname nombre` (es temporal). Editar el archivo persistente:
```bash
cd /etc/
cp hostname hostname_backup   # backup por las dudas
vim hostname                  # cambiar "debian" por "TPServer"
cat hostname                  # verificar el cambio
```

---

## 2) Servicios

> Las pruebas de conectividad y acceso web se hacen desde la máquina física u otra máquina de la misma red.

### 2.1 Arreglar repositorios (sources.list)
Si `apt-get update` falla (mirror `cdn2-fastly.deb.debian.org` caído), usar los repos oficiales de Debian bullseye:

```
deb http://deb.debian.org/debian bullseye main contrib non-free
deb-src http://deb.debian.org/debian bullseye main contrib non-free

deb http://deb.debian.org/debian-security/ bullseye-security main contrib non-free
deb-src http://deb.debian.org/debian-security/ bullseye-security main contrib non-free

deb http://deb.debian.org/debian bullseye-updates main contrib non-free
deb-src http://deb.debian.org/debian bullseye-updates main contrib non-free
```

```bash
cp sources.list sources.list.backup
vim sources.list
apt-get update
```

### 2.2 SSH (acceso root con clave pública/privada)

Instalar:
```bash
apt-get install openssh-server
systemctl status ssh
```

Configurar acceso root vía SSH, editando `/etc/ssh/sshd_config`:
```bash
cp sshd_config sshd_config_backup
vim sshd_config
```
Descomentar y setear:
```
PermitRootLogin yes
PublicKeyAuthentication yes
```
Reiniciar el servicio:
```bash
/etc/init.d/ssh restart
```

Configurar las claves:
```bash
mkdir /root/.ssh
chmod 700 /root/.ssh
# copiar la clave privada como id_rsa (permiso 600)
# copiar la clave pública como id_rsa.pub / authorized_keys
chmod 600 /root/.ssh/id_rsa
```

Para transferir archivos a la VM: **WinSCP** (se pueden pasar directo a `/root/`).

Ver la IP para conectarse:
```bash
ip a
```

### 2.3 Web: Apache + PHP

```bash
apt-get install apache2
systemctl status apache2
```

Instalar PHP (7.3+) y el módulo de Apache:
```bash
apt install php-common libapache2-mod-php php-cli
```

Verificar versión:
```bash
php -v
```

Reiniciar Apache:
```bash
/etc/init.d/apache2 restart
```

- Los archivos que sirve Apache están en `/var/www/html`.
- Copiar `index.php` y `logo.png` ahí (vía WinSCP a `/root` y después mover).
- **Borrar `index.html`** (hacer backup antes) para que Apache sirva `index.php` directo.

### 2.4 Base de datos: MariaDB

```bash
apt-get install mariadb-server
apt-get install php-mysql   # conector PHP <-> MySQL/MariaDB
```

Importar el script SQL:
```bash
mysql < /root/db.sql
```
> **No copiar `db.sql` a `/var/www/html`**: quedaría accesible públicamente.

Verificar:
```bash
mysql
show databases;
quit;
```

Ver servicios/puertos activos:
```bash
netstat -natupl
```

Ver la página con la base de datos cargada: `http://<ip>/` (ej: `192.168.1.49`)

Logs de Apache:
```bash
cd /var/log/apache2
ls -ltr
cat error.log
cat access.log
```

---

## 3) Configuración de red (IP estática)

Archivo: `/etc/network/interfaces`

```
auto enp0s3
iface enp0s3 inet static
    address 192.168.1.49
    netmask 255.255.255.0
    gateway 192.168.1.1
```

- La IP debe estar en el mismo rango que la máquina física (clase C: `192.168.0.0` – `192.168.255.255`).
- El gateway es el de salida hacia la red física (verificar con `ipconfig` en Windows host, o `ip route show default` en Linux).

Aplicar cambios:
```bash
systemctl restart networking
```

---

## 4) Almacenamiento

### 4.1 Agregar disco de 10 GB
1. Apagar la VM.
2. En VirtualBox: Configuración → Almacenamiento → Añadir disco duro → Crear → asignar tamaño → asignación dinámica/completa.
3. Iniciar la VM y verificar:
   ```bash
   lsblk
   ls /dev
   ```

### 4.2 Particiones (tipo 83)
- `/www_dir`: 3 GB
- `/backup_dir`: 6 GB

(usar `fdisk` sobre el disco nuevo para crear las particiones primarias tipo Linux/83, formatearlas con `mkfs.ext4` o similar).

### 4.3 Apuntar Apache a `/www_dir`
Copiar `index.php` y `logo.png` a `/www_dir`.

Editar `/etc/apache2/sites-available/000-default.conf` y cambiar `DocumentRoot` a `/www_dir` (o la ruta correspondiente, ej. `/root/www_dir`).

> Nota: `/etc/apache2/apache2.conf` es para configuración global (permisos de directorios, etc.), no para el `DocumentRoot` específico del sitio. Si Apache sigue buscando en `/var/www/html`, revisar los permisos/`<Directory>` en `apache2.conf`.

### 4.4 Montaje automático al bootear
Editar `/etc/fstab` para que `/www_dir` y `/backup_dir` se monten automáticamente al iniciar.

Verificar espacio en disco:
```bash
df -h
```

### 4.5 Info de particiones persistente
`/proc/partitions` es efímero (se pierde al apagar la VM) y no se puede escribir dentro de `/proc` directamente. Solución: copiar su contenido a un archivo fuera de `/proc` (ej. `/root/particion`), idealmente automatizado con un script + cron.

---

## 5) Backup automático

### 5.1 Script `backup_full.sh`

```bash
mkdir -p /opt/scripts
```

```bash
#!/bin/bash

# Función de ayuda
mostrar_ayuda() {
    echo "Uso: $0 <origen> <destino>"
    echo
    echo "Este script crea un archivo .tar.gz como backup de un directorio o archivo"
    echo
    echo "Ejemplo:"
    echo "  $0 /var/log /backup_dir"
}

if [ "$1" == "-help" ]; then
    mostrar_ayuda
    exit 0
fi

# Argumentos
ORIGEN="$1"
DESTINO="$2"

# Verificar que origen y destino existan
if [ ! -e "$ORIGEN" ]; then
    echo "Argumento inválido: $ORIGEN"
    exit 1
fi

if [ ! -e "$DESTINO" ]; then
    echo "Argumento inválido: $DESTINO"
    exit 1
fi

# Fecha en formato ANSI
FECHA=$(date +%Y%m%d)
NOMBRE=$(basename "$ORIGEN")
ARCHIVO="${DESTINO}/${NOMBRE}bkp${FECHA}.tar.gz"

tar -czf "$ARCHIVO" "$ORIGEN"

echo "Backup completo: $ARCHIVO"
```

Dar permisos de ejecución:
```bash
chmod +x /opt/scripts/backup_full.sh
```

> Si el script se transfirió desde Windows (WinSCP), puede traer caracteres invisibles (`\r`). Solucionarlo con:
> ```bash
> dos2unix backup_full.sh
> ```

Prueba manual:
```bash
/opt/scripts/./backup_full.sh /var/logs /backup_dir
```

### 5.2 Programar en cron

Editar `/etc/crontab`:
```bash
vim /etc/crontab
```

Tareas requeridas:
- **Todos los días a las 00:00** → backup de `/var/logs`
- **Lunes, miércoles y viernes a las 23:00** → backup de `/www_dir`

Ejemplo de líneas cron:
```
0 0 * * *      root  /opt/scripts/backup_full.sh /var/logs /backup_dir
0 23 * * 1,3,5 root  /opt/scripts/backup_full.sh /www_dir /backup_dir
```

(Crear `/var/logs` si no existe: `mkdir /var/logs`)

---

## 6) Entregables

- GitHub tiene un límite de 25 MB por archivo, así que si el `.tar.gz` de `/var` (o el backup completo) supera ese tamaño, hay que **particionarlo** en partes más chicas para poder subirlo (ej. con `split`).

---

## Notas generales / comandos útiles

| Comando | Uso |
|---|---|
| `systemctl status <servicio>` | Ver estado de un servicio |
| `systemctl restart <servicio>` | Reiniciar un servicio |
| `netstat -natupl` | Ver puertos/servicios en escucha |
| `ip a` | Ver IPs de las interfaces |
| `ip route show default` | Ver el gateway por defecto |
| `lsblk` | Ver discos y particiones |
| `df -h` | Ver espacio en disco (formato legible) |
