# Instalación  manual de los binarios de FLEXlm en Debian bookworm

### Instalación de paquetes y configuración del entorno

```
sudo apt update
sudo apt upgrade
sudo apt install libc6-i386
sudo apt install binutils
sudo apt install lsb-release
sudo apt install tree
```

La instalación se realiza manualmente en el directorio * /opt * son binarios de terceros y sin dependencias del sistema. Se elige este directorio porque en la actualizaciones  evita tocarlo. Crear el usuario y grupo flexlm de sistema

```
useradd flexlm -b /opt/flexlm -c "Usuario flexlm" -s /sbin/nologin --system
grep flexlm /etc/group
id flexlm
````

Creamos la carpeta /opt/flexlm/bin para almacenar los binarios de felxlm y /opt/flexlm/lic par almacenar el fichero de licencia
```
cd /opt/flexlm
sudo mkdir bin
sudo mkdir lic
sudo ln -s /usr/tmp tmp #enlace simbolico
sudo chmod 1777 /usr/tmp #por defecto tiene esos permirsos el directorio /tmp
```
Creamos la carpeta para almacenar los logs en  /var/log/flexlm. Se asigna dueño y permisos.
```
sudo mkdir /var/log/flexlm
sudo chmod 0744 /var/log/flexlm
sudo chown flexlm -R /var/log/flexlm
```
### Binarios FLEXlm de Matlab
Descargo flexlm_matlab_R2024a.zip  del centro de descargas de MathWork
Copio los archivos //R2024a/etc/glnxa64/ a la carpeta /opt/flexlm/bin
```
sudo cp  /*/R2024a/etc/glnxa64/* /opt/flexlm/bin
```
### Obtención y configuración fichero licencia

La licencia para el servidor se consigue en MathWorks - Centro de licencias. Se utiliza como ID de host la mac de interfaz de red. Se copia el fichero *.lic al directorio /opt/flexlm/lic
Obtenemos el hostid para la licencia.
```
./lmhostid
lmhostid - Copyright (c) 1989-2021 Flexera. All Rights Reserved.
The FlexNet host ID of this machine is ""######### ########""
Only use ONE from the list of hostids.
Se copia el fichero *.lic -- el fichero lo he obtenido de MathWorks - Centro de licencias – a la carpeta  /opt/flexlm/lic
Editamos el fichero *.lic
```
Se copia el fichero *.lic -- el fichero lo he obtenido de MathWorks - Centro de licencias – a la carpeta  /opt/flexlm/lic
Editamos el fichero *.lic
```
GNU nano 7.2                                          licencia.lic *
# BEGIN--------------BEGIN--------------BEGIN
# MathWorks license passcode file.
# LicenseNo: ######   HostID: #######
#
# R2024a
#
SERVER matlabSERVER ####### 27000
DAEMON MLM /opt/flexlm/bin/MLM port=27001
```
Creamos un enlace simbólico licencia.lic apuntando al fichero *.lic 
``
sudo ln -s /opt/flexlm/lic/licence_2024a.lic licencia.lic
```
Asignamos dueños y permisos a la carpeta /opt/flexlm
```
sudo chown -R flexlm /opt/flexlm
sudo chmod -R 0700 /opt/flexlm/bin
sudo chmod 644 /opt/flexlm/lic/licence_2024a.lic
```
### Estructura final y ficheros del directorio /opt/flexlm

```
flexlm/bin
keycheck
lmborrow
lmdiag
lmdown
lmgrd
lmhostid
lmremove
lmreread
lmstat
lmswitchr
lmutil
lmver
MLM
ParallelServerLicenseCheck
flexlm7/lic
licence_r2024a.lic
licencia.lic -> licence_r2024a.lic
flexlm/tmp -> /usr/tmp
```
### Creación y configuración de servicio flexlm con systemd

Plantilla (lo más genérica para reutilizar) para systemd  de flexlm. El creamos y editamos el fichero flexlm.service. 
``` 
[Unit]
Description=Servicio para ejecutar el servidor licencias flexlm
After=network.target
[Service]
User=flexlm
Type=simple
WorkingDirectory=/opt/flexlm/bin
Restart=always
#Este es el código de retorno que utiliza flexlm de salida ok
SuccessExitStatus=15 
RestartSec=30
ExecStart=/opt/flexlm/bin/lmgrd -z -c /opt/flexlm/lic/licencia.lic -l +/var/log/flexlm/flexlm.log
ExecReload=/opt/flexlm/bin/lmutil lmreread -c licencia.lic
ExecStop=/opt/flexlm/bin/lmutil lmdown -c licencia.lic -q -force
[Install]
WantedBy=multi-user.target
```
Se copia el fichero flexlm.service  a  la carpeta: /etc/systemd/system/

Para ejecutarlo:
```
sudo systemctl start flexlm.service
sudo systemctl stop flexlm.service
```
Para lanzar el demonio lmgrd en el arranque :
```
sudo systemctl enable flexlm.service
```
### Anexo

#### Problemas con el paquete lsb-core en debian 12
Puede aparecer el siguiente error al ejecutar lmgrd
```
lmgrd -c /opt/flexlm/lic/License.lic -log /var/log/flexlm/flexlm.log bash: ./Linux\_Licensing\_Daemon/lmgrd: /lib/ld-lsb.so.3: bad ELF interpreter: No such file or directory
```
Revisamos la librerías que hacen falta para lmgrd proporcionado por Mathworks
```
 readelf -a lmgrd | grep NEEDED
 0x0000000000000001 (NEEDED)             Shared library: [libpthread.so.0]
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [librt.so.1]
```
```
 ldd lmgrd
        linux-vdso.so.1 (0x00007ffe9b9cd000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f49a340f000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f49a3330000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f49a3310000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f49a312f000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f49a312a000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f49a3123000)
        /lib64/ld-lsb-x86-64.so.3 => /lib64/ld-linux-x86-64.so.2 (0x00007f49a341b000)
```
```
file lmgrd
lmgrd: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-lsb-x86-64.so.3, for GNU/Linux 2.6.18, stripped
```
Necesitamos la librería ld-lsb-x86-64.so.3 en el directorio /lib/64. Esta librería esta en el paquete lsb_core que esta descontinuada.
Enlazamos la librería /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 con ld-lsb-x86-64.so.3 para corregir el error
```
cd /lib64
sudo ln -sf /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ld-lsb-x86-64.so.3
ls -l
total 0
lrwxrwxrwx 1 root root 42 Apr 30 21:07 ld-linux-x86-64.so.2 -> /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```
