# Despliegue de Active Directory Domain Controller con Samba 4 en Rocky Linux 9 Minimal

Este proyecto documenta la implementación completa de un Controlador de Dominio principal (AD DC) sobre **Rocky Linux 9 Minimal** utilizando **Samba 4**, proporcionando una alternativa económica y robusta a Windows Server para centralizar la autenticación, administración de identidades, directivas de grupo (GPO), servicio DHCP y almacenamiento compartido en entornos corporativos.

## 🔗Link del video en Youtube

https://youtu.be/601xdsOMKBY

---

## 🏛️ Arquitectura del Entorno

* **Servidor (DC1):** Rocky Linux 9 Minimal.


* **Dirección IP:** `192.168.10.130/24` (Interfaz `ens192`).


* **Puerta de Enlace (Gateway):** `192.168.5.2`.


* **Roles:** Active Directory Domain Controller (AD DC), Servidor DNS interno y Servidor DHCP.


* **Dominio / Realm:** `darknet.solutions` / `DARKNET.SOLUTIONS` (NetBIOS: `DARKNET`).


* **Almacenamiento:** Unidad primaria para el sistema operativo y una unidad NVMe secundaria (`/dev/nvme0n2`) montada en `/srv/samba` para perfiles móviles y File Server.




* **Cliente:** Windows 10 Pro / Enterprise unido al dominio y gestionado vía RSAT.



---

## 🏢 Estructura Organizativa y Políticas

El directorio activo se organizó en 5 Unidades Organizativas (OUs), cada una con su respectivo grupo de seguridad y dos usuarios asignados:

| Unidad Organizativa (OU) | Grupo de Seguridad | Usuarios | Restricciones de GPO | Permisos en File Server |
| :--- | :--- | :--- | :--- | :--- |
| **Dirección de Tecnología** | `g_tecnologia` | `lperez`, `crodriguez` | Ninguna (Libre de restricciones) | Acceso total sin veto de archivos |
| **Dirección Administrativa** | `g_administrativa` | `jlopez`, `amendez` | Panel de Control, CMD, RUN bloqueados + Wallpaper corporativo | Solo documentos y texto (Veto de audio, video y ejecutables) |
| **Dirección Legal** | `g_legal` | `mhernandez`, `wbelliard` | Panel de Control, CMD, RUN bloqueados + Wallpaper corporativo | Solo documentos y texto (Veto de audio, video y ejecutables) |
| **Dirección de Comunicaciones** | `g_comunicaciones` | `jpereira`, `hmanuel` | Panel de Control, CMD, RUN bloqueados + Wallpaper corporativo | Documentos, texto, audio y video (Veto de ejecutables) |
| **Dirección de Gestión Humana** | `g_gestionhumana` | `jcaminero`, `ftatis` | Panel de Control, CMD, RUN bloqueados + Wallpaper corporativo | Solo documentos y texto (Veto de audio, video y ejecutables) |

* **Políticas Globales de Cuenta:** Bloqueo de cuenta tras 3 intentos fallidos y complejidad de contraseñas habilitada (mínimo 8 caracteres).


* **Perfiles de Usuario:** Redirección de la carpeta *Documentos* (`Folder Redirection`) hacia el almacenamiento centralizado en `/srv/samba/profiles` con permisos exclusivos para cada usuario.



---

## 🚀 Guía de Despliegue Paso a Paso

### 1. Preparación del Entorno en Rocky Linux

Configuración del nombre de host y resolución estática:

```bash
sudo hostnamectl set-hostname dc1.darknet.solutions
```

Edición de `/etc/hosts` para soporte de Kerberos:

```text
127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain
192.168.10.130 dc1.darknet.solutions dc1
```

Ajuste de SELinux a modo permisivo y apertura de puertos perimetrales en `firewalld`:

```bash
sudo setenforce 0
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config

sudo firewall-cmd --add-service={dns,dhcp,samba} --permanent
sudo firewall-cmd --add-port={88/tcp,88/udp,135/tcp,389/tcp,389/udp,445/tcp,464/tcp,464/udp,636/tcp,3268/tcp,3269/tcp} --permanent
sudo firewall-cmd --reload
```

---

### 2. Instalación y Aprovisionamiento de Samba 4

Habilitación del repositorio de Tranquil IT e instalación de binarios:

```bash
sudo tee /etc/yum.repos.d/tranquil-it-samba.repo <<EOF
[tranquil-it-samba]
name=Samba AD DC - Tranquil IT
baseurl=https://samba.tranquil.it/redhat9/samba-4.23/
gpgcheck=0
gpgkey=https://samba.tranquil.it/tissamba-pubkey.gpg
enabled=1
EOF

sudo dnf clean all
sudo dnf update -y
sudo dnf install epel-release -y
sudo dnf config-manager --set-enabled crb
sudo dnf install samba samba-dc krb5-workstation -y
```

Aprovisionamiento de la base de datos de Active Directory:

```bash
sudo rm -f /etc/samba/smb.conf

sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=DARKNET.SOLUTIONS \
  --domain=DARKNET \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --host-ip=192.168.10.130 \
  --adminpass="D4rkn3t2026"
```

Configuración de Kerberos y resolución interna:

```bash
sudo rm -f /etc/krb5.conf
sudo ln -s /var/lib/samba/private/krb5.conf /etc/krb5.conf

echo -e "search darknet.solutions\nnameserver 127.0.0.1" | sudo tee /etc/resolv.conf

sudo systemctl disable --now smb nmb winbind 2>/dev/null
sudo systemctl mask smb nmb winbind 2>/dev/null
sudo systemctl enable --now samba
```

---

### 3. Configuración del Servidor DHCP

Instalación de `dhcp-server` y definición de la subred del laboratorio en `/etc/dhcp/dhcpd.conf`:

```ini
option domain-name "darknet.solutions";
option domain-name-servers 192.168.10.130;
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.50 192.168.10.150;
    option routers 192.168.10.130;
}
```

Asignar el demonio DHCP a la interfaz de red interna `ens192` mediante `sudo systemctl edit dhcpd` y arrancar el servicio:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd ens192

```

```bash
sudo systemctl start dhcpd
```

---

### 4. Estructuración Interna y Políticas de Contraseña

Creación de Unidades Organizativas (OUs), Grupos de Seguridad y Cuentas de Usuario por CLI:

```bash
# Unidades Organizativas
sudo samba-tool ou create "OU=Tecnologia,DC=darknet,DC=solutions"
sudo samba-tool ou create "OU=Administrativa,DC=darknet,DC=solutions"
sudo samba-tool ou create "OU=Legal,DC=darknet,DC=solutions"
sudo samba-tool ou create "OU=Comunicaciones,DC=darknet,DC=solutions"
sudo samba-tool ou create "OU=GestionHumana,DC=darknet,DC=solutions"

# Grupos de Seguridad
for g in g_tecnologia g_administrativa g_legal g_comunicaciones g_gestionhumana; do
    sudo samba-tool group add $g
done

# Usuarios departamentales
sudo samba-tool user create lperez "2026@D4rkn3t" --userou="OU=Tecnologia"
sudo samba-tool user create crodriguez "2026@D4rkn3t" --userou="OU=Tecnologia"
sudo samba-tool user create jlopez "2026@D4rkn3t" --userou="OU=Administrativa"
sudo samba-tool user create amendez "2026@D4rkn3t" --userou="OU=Administrativa"
sudo samba-tool user create mhernandez "2026@D4rkn3t" --userou="OU=Legal"
sudo samba-tool user create wbelliard "2026@D4rkn3t" --userou="OU=Legal"
sudo samba-tool user create jpereira "2026@D4rkn3t" --userou="OU=Comunicaciones"
sudo samba-tool user create hmanuel "2026@D4rkn3t" --userou="OU=Comunicaciones"
sudo samba-tool user create jcaminero "2026@D4rkn3t" --userou="OU=GestionHumana"
sudo samba-tool user create ftatis "2026@D4rkn3t" --userou="OU=GestionHumana"

# Asignación de Miembros a Grupos
sudo samba-tool group addmembers g_tecnologia lperez,crodriguez
sudo samba-tool group addmembers g_administrativa jlopez,amendez
sudo samba-tool group addmembers g_legal mhernandez,wbelliard
sudo samba-tool group addmembers g_comunicaciones jpereira,hmanuel
sudo samba-tool group addmembers g_gestionhumana jcaminero,ftatis

# Política de contraseñas y bloqueo
sudo samba-tool domain passwordsettings set --complexity=on --min-pwd-length=8
sudo samba-tool domain passwordsettings set --account-lockout-threshold=3

```

---

### 5. Almacenamiento Dedicado (NVMe) y File Server

Formateo y montaje del disco NVMe secundario con persistencia en `/etc/fstab`:

```bash
sudo mkfs.ext4 /dev/nvme0n2
sudo mkdir -p /srv/samba
echo "/dev/nvme0n2 /srv/samba ext4 defaults 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

Creación de directorios, asignación de permisos y activación del Sticky Bit en perfiles:

```bash
sudo mkdir -p /srv/samba/{profiles,restringidos,comunicaciones,tecnologia}
sudo chown -R root:root /srv/samba/
sudo chmod -R 777 /srv/samba/
sudo chmod 1777 /srv/samba/profiles
```

Configuración de recursos compartidos con filtrado departamental (`valid users`) y veto de archivos (`veto files`) en `/etc/samba/smb.conf`:

```ini
[global]
        dns forwarder = 192.168.5.2
        netbios name = DC1
        realm = DARKNET.SOLUTIONS
        server role = active directory domain controller
        workgroup = DARKNET
        idmap_ldb:use rfc2307 = yes
        interfaces = 192.168.10.130 127.0.0.1
        bind interfaces only = yes

[sysvol]
        path = /var/lib/samba/sysvol
        read only = No

[netlogon]
        path = /var/lib/samba/sysvol/darknet.solutions/scripts
        read only = No

[Profiles]
        path = /srv/samba/profiles
        read only = No
        guest ok = No
        browseable = No
        directory mask = 0700
        create mask = 0700

[FormatosRestringidos]
        path = /srv/samba/restringidos
        read only = No
        guest ok = No
        valid users = @g_administrativa @g_legal @g_gestionhumana
        veto files = /*.mp3/*.mp4/*.avi/*.mkv/*.exe/*.msi/*.bat/
        delete veto files = Yes

[Comunicaciones]
        path = /srv/samba/comunicaciones
        read only = No
        guest ok = No
        valid users = @g_comunicaciones
        veto files = /*.exe/*.msi/*.bat/
        delete veto files = Yes

[Tecnologia]
        path = /srv/samba/tecnologia
        read only = No
        guest ok = No
        valid users = @g_tecnologia

```

Reiniciar Samba para aplicar la configuración:

```bash
sudo systemctl restart samba
```

---

### 6. Integración del Cliente Windows 10 y Despliegue de GPOs

1. **Unión al Dominio:** Conectar la máquina cliente al adaptador virtual interno, verificar recepción de IP y DNS por DHCP e ingresar al dominio `darknet.solutions` con la cuenta `Administrator`.


2. **Instalación de RSAT:** Instalar las herramientas opcionales de *Active Directory Domain Services* y *Group Policy Management Tools*.


3. **`GPO_Restricciones_Departamentales`:**
* Prohibir el acceso al Panel de Control y Configuración.


* Impedir el acceso al Símbolo del sistema (CMD).


* Quitar el menú "Ejecutar" (RUN).


* Asignar fondo de pantalla corporativo vía ruta de red.


* Vincular a: `Administrativa`, `Legal`, `Comunicaciones` y `GestionHumana` (excluyendo a `Tecnologia`).




4. **`GPO_Redireccion_Documentos`:**
* Redirección básica de la carpeta *Documentos* apuntando a la ruta raíz `\\192.168.10.130\Profiles` con derechos exclusivos otorgados al usuario.





---

## 🔧 Resolución de Problemas (Troubleshooting)

* **El dominio no resuelve o desaparece tras reiniciar:** NetworkManager puede sobreescribir `/etc/resolv.conf`. Restaurar la línea `nameserver 127.0.0.1` y aplicar el atributo de inmutabilidad:


```bash
sudo chattr +i /etc/resolv.conf

```


* **Fallo de Kerberos / Desfase Horario:** Sincronizar el reloj del sistema mediante Chrony:


```bash
sudo systemctl restart chronyd
sudo chronyc sources -v
```


* **Error "Accessing the resource has been disallowed" / Timeout de SMB:**
* Generar la Zona DNS Inversa y registrar el registro PTR correspondiente para la validación mutua de Kerberos:


```bash
sudo samba-tool dns zonecreate 127.0.0.1 10.168.192.in-addr.arpa
sudo samba-tool dns add 127.0.0.1 10.168.192.in-addr.arpa 130 PTR dc1.darknet.solutions
```


* Enlazar Samba únicamente a la interfaz de la LAN en `smb.conf` (`bind interfaces only = yes`) para prevenir fugas de IPs virtuales (como Docker o NAT externa) en las consultas DNS.


* En el cliente Windows 10, purgar cachés de resolución, sesiones SMB y tickets Kerberos:


```cmd
ipconfig /flushdns
klist purge
net use * /delete /y
```

## 👤 Contactos
https://www.linkedin.com/in/vincent-perez-1295a5341