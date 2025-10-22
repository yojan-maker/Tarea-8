# Tarea: Creación y Prueba de Máquinas Virtuales (Rocky, Kali, Windows) — Guía paso a paso

## Descripción
En esta tarea se documenta el proceso para crear tres máquinas virtuales (Rocky Linux, Kali Linux y Windows) usando **QEMU/KVM** en un host Linux, verificar que se comuniquen entre sí y organizar la documentación en un repositorio.  
> Nota importante: **No fue posible instalar Windows** en este equipo por limitaciones de hardware (recursos insuficientes). Más abajo se explica por qué y se deja el procedimiento teórico completo para quien tenga máquina más potente.

---

## Requisitos previos (host)
Estas instrucciones suponen un host Linux (ej. Ubuntu/Debian). Ajusta comandos para otras distros.

1. Actualizar e instalar paquetes necesarios:
```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst virt-manager qemu-img bridge-utils
```
2. Verificar KVM/libvirt:
```bash
sudo systemctl status libvirtd
ls /dev/kvm         # debe existir
```
3. Crear directorios de trabajo:
```bash
mkdir -p ~/vm/isos ~/vm/images
cd ~/vm
```
Recomendaciones de recursos (ajustar según tu PC):
Rocky / Kali: 2 vCPU, 2–4 GB RAM, disco 20–30 GB (qcow2).
Windows: 4 vCPU, 6–8 GB RAM, disco 40–80 GB (recomendado).
Si tu equipo no tiene suficiente RAM/CPU/disk, no intentes instalar Windows — el instalador y la ejecución fallarán o dejarán el equipo inutilizable.

### Red: cómo hacer que todas las VMs se comuniquen
Dos opciones comunes:

**A) Usar la red default de libvirt (NAT)**

- Es la opción más simple. Todas las VMs conectadas a network=default estarán en la misma subred virtual y podrán comunicarse entre sí.
- En virt-install: --network network=default
- Para acceder desde host: usar reenvío de puertos (--network network=default,port=... o hostfwd con qemu-system).

**B) Usar un bridge (puente) para que las VMs estén en la LAN física**

- Más complejo pero las VMs tendrán IP en la misma LAN que el host.
- Crear bridge (ejemplo, usando netplan o nmcli) y usar --network bridge=br0 en virt-install.
- Requiere privilegios y potencialmente modificar la configuración de red del host.

---

## 1) Crear máquina virtual: Rocky Linux (paso a paso)
- 1.1 Descargar ISO

Coloca la ISO en ~/vm/isos/rocky.iso (descárgala desde la web oficial cuando tengas conexión).
```bash
qemu-img create -f qcow2 ~/vm/images/rocky.qcow2 25G
```
- 1.2 Crear disco qcow2
```bash
qemu-img create -f qcow2 ~/vm/images/rocky.qcow2 25G
```
- 1.3 Comando virt-install (recomendado)
```bash
virt-install \
  --name rocky-vm \
  --ram 2048 \
  --vcpus 2 \
  --disk path=~/vm/images/rocky.qcow2,format=qcow2 \
  --cdrom ~/vm/isos/rocky.iso \
  --os-variant centos8 \
  --network network=default \
  --graphics vnc \
  --boot cdrom,hd
```
Si prefieres instalación por consola (no gráfica) usa --graphics none y --extra-args según ISO.

**1.4 Pasos dentro del instalador**

- Seleccionar idioma, zona horaria.
- Elegir particionado (recomendado disco completo / automática).
- Configurar red (DHCP o IP estática).
- Crear usuario y contraseña root.
- Instalar paquetes mínimos o servidor según necesidad.

**1.5 Arrancar VM una vez instalado**
```bash
virsh start rocky-vm
virsh console rocky-vm   # console si config permitida
# o usar virt-manager para conectarte por VNC/console
```
**1.6 Post-instalación (dentro del guest)**
```bash
sudo dnf update -y
sudo dnf install -y openssh-server qemu-guest-agent
sudo systemctl enable --now sshd qemu-guest-agent
```
---
## 2) Crear máquina virtual: Kali Linux (paso a paso)

**2.1 Descargar ISO**
Guarda la ISO en ~/vm/isos/kali.iso.

**2.2 Crear disco**
```bash
qemu-img create -f qcow2 ~/vm/images/kali.qcow2 30G
```
**2.3 Instalar con virt-install**
```bash
virt-install \
  --name kali-vm \
  --ram 2048 \
  --vcpus 2 \
  --disk path=~/vm/images/kali.qcow2,format=qcow2 \
  --cdrom ~/vm/isos/kali.iso \
  --os-variant debian10 \
  --network network=default \
  --graphics vnc \
  --boot cdrom,hd
```
**2.4 Pasos en el instalador de Kali**
- Seleccionar “Graphical install” o “Install”.
- Configurar idioma, teclado, red (DHCP o estático).
- Particionado (guiado, LVM o manual).
- Crear usuario (Kali ahora recomienda crear usuario normal y no root por defecto).
- Instalar paquetes y esperar.

**2.5 Post-instalación**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server qemu-guest-agent
sudo systemctl enable --now ssh qemu-guest-agent
```

### 3) Crear máquina virtual: Windows (procedimiento teórico / recomendado)
>- IMPORTANTE: En este equipo no se instalará Windows por falta recursos (RAM/CPU/disk). 
A continuación quedan las instrucciones teóricas completas para quien cuente con una máquina host con suficiente capacidad (mínimo 8 GB RAM disponibles, 100 GB disco libre recomendado).

**3.1 Requisitos y archivos necesarios**

- ISO de Windows 10/11 (o instalador)
- ISO de drivers VirtIO (virtio-win ISO) para controladores de disco y red: virtio-win.iso
- Disco qcow2 grande (ej. 60–80 GB)

**3.2 Crear disco**
```bash
qemu-img create -f qcow2 ~/vm/images/windows.qcow2 60G
```

**3.3 Instalar con virt-install (ejemplo)**
```bash
virt-install \
  --name windows-vm \
  --ram 8192 \
  --vcpus 4 \
  --disk path=~/vm/images/windows.qcow2,format=qcow2,bus=virtio \
  --cdrom ~/vm/isos/Win10.iso \
  --disk path=~/vm/isos/virtio-win.iso,device=cdrom \
  --os-variant win10 \
  --network network=default,model=virtio \
  --graphics spice
```
- Durante la instalación de Windows, en la parte donde pide controlador de disco (no aparece disco), usar la opción Load driver y 
seleccionar los controladores desde virtio-win.iso → viostor para que Windows detecte el disco.
- Instalar drivers de red NetKVM también desde virtio-win.iso.

**3.4 Configuración post-instalación (Windows)**

- Instalar drivers virtio completos.
- Habilitar RDP y/o instalar OpenSSH (Windows 10/11 tienen opción de OpenSSH).
- Ajustar recursos si hace falta.
  
**3.5 Por qué NO instalar Windows en este host**

- Windows requiere mucho RAM y CPU durante la instalación y ejecución (especialmente con GUI).
- Si tu PC tiene <8 GB RAM total o menos de 4 vCPU libres, la VM puede dejar el host sin memoria/respuesta.
- Por seguridad y estabilidad del equipo, no se realizó la instalación real en este ejercicio.

### 4) Probar que todas las máquinas virtuales se comuniquen entre sí

**4.1 Requisitos para comunicación**
  
- Todas las VMs deben estar en la misma red virtual (network=default o bridge=br0).
- Servicios mínimos: tener SSH habilitado en las VMs Linux.

**4.2 Pasos de verificación (ejemplo)**

1. Listar interfaces y IPs en cada VM:
Linux:
```bash
ip addr show
```
Windows (si instalado):
```bash
ipconfig
```
2. Desde Rocky (ejemplo), hacer ping a Kali:
```bash
ping -c 4 <IP_KALI>
```
3. Desde Kali a Rocky:
```bash
ping -c 4 <IP_ROCKY>
```
4. Desde el host a cada VM (si hay hostfwd, usar ssh -p):
```bash
ssh -p 2222 usuario@localhost   # ejemplo para la VM con hostfwd 2222
```
5. Comprobar conectividad entre todas (haz una pequeña tabla en tu documentación con resultados PING, latencia y si hubo pérdida):

- Rocky ↔ Kali: OK / fallo
- Rocky ↔ Windows: OK / fallo (si aplica)
- Kali ↔ Windows: OK / fallo (si aplica)
- Host ↔ cada VM: OK / fallo

6. (Opcional) Escanear la red con nmap desde una VM:
```bash
sudo nmap -sP 192.168.x.0/24   # detecta hosts activos
```

