# Passer un GPU Nvidia P400 sur un LXC Proxmox

1 - Ajout des dépôts non-free et contrib sur l'hôte PVE
```ssh
nano /etc/apt/sources.list
```
```ssh
deb http://ftp.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://ftp.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb http://security.debian.org bookworm-security main contrib non-free non-free-firmware
```
Save & exit

2 - Install du pve-header sur l'hôte PVE
```ssh
apt install pve-headers
```

3 - Vérification de la présence du GPU sur l'hôte PVE
```ssh
lspci | grep -i nvidia
```

4 - Install des drivers Nvidia
```ssh
apt install nvidia-driver
```

5 - Editer conf noyau
```ssh
nano /etc/modules-load.d/modules.conf
```
Et ajouter les lignes suivantes :
```ssh
nvidia-drm
nvidia
nvidia_uvm
```

6 - Création du fichier rules nvidia
```ssh
nano /etc/udev/rules.d/70-nvidia.rules
```
Et ajouter les lignes suivantes :
```ssh
# Create /nvidia0, /dev/nvidia1 … and /nvidiactl when nvidia module is loaded
KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L && /bin/chmod 666 /dev/nvidia*'"
# Create the CUDA node when nvidia_uvm CUDA module is loaded
KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u && /bin/chmod 0666 /dev/nvidia-uvm*'"
```
Save, exit & reboot PVE

7 - Vérification de la sortie /dev/nvidia et /dev/dri
```ssh
ls -al /dev/nvidia*
```

Il devrait avoir un retour de ce type
```ssh
root@pve:~# ls -al /dev/nvidia
"crw-rw-rw- 1 root root 195,   0 Dec 14 17:12 /dev/nvidia0"
crw-rw-rw- 1 root root 195, 255 Dec 14 17:12 /dev/nvidiactl
crw-rw-rw- 1 root root 195, 254 Dec 14 17:12 /dev/nvidia-modeset
crw-rw-rw- 1 root root 509,   0 Dec 14 17:12 /dev/nvidia-uvm
crw-rw-rw- 1 root root 509,   1 Dec 14 17:12 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
drw-rw-rw-  2 root root     80 Dec 14 17:12 .
drwxr-xr-x 20 root root   5080 Dec 14 17:12 ..
cr--------  1 root root 234, 1 Dec 14 17:12 nvidia-cap1
cr--r--r--  1 root root 234, 2 Dec 14 17:12 nvidia-cap2"
```

Si pas de retour des 5 lignes /dev/nvidia alors le pilote n'est surement pas installé

```ssh
ls -al /dev/dri*
```

Il devrait avoir un retour de ce type
```ssh
root@pve:~# ls -al /dev/dri*
total 0
drwxr-xr-x  3 root root        140 Dec 14 17:12 .
drwxr-xr-x 20 root root       5080 Dec 14 17:12 ..
drwxr-xr-x  2 root root        120 Dec 14 17:12 by-path
crw-rw----  1 root video  226,   0 Dec 14 17:12 card0
crw-rw----  1 root video  226,   1 Dec 14 17:12 card1
crw-rw----  1 root render 226, 128 Dec 14 17:12 renderD128
crw-rw----  1 root render 226, 129 Dec 14 17:12 renderD129
```

Prendre en compte les nombres de la 5eme colonne : 195, 509 et 226

8 - Vérification du retour GPU

```ssh
nvidia-smi
```

Un retour de ce type devrait être à l'écran :

```ssh
root@pve:~# nvidia-smi
Thu Dec 14 17:54:41 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.147.05   Driver Version: 525.147.05   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P400         On   | 00000000:01:00.0 Off |                  N/A |
| 34%   33C    P8    N/A /  30W |      1MiB /  2048MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

9 - Installation sur le LXC

Nous pouvons voir ci dessus que le pilote utilisé est le 525.147.05.
On va donc reprendre ce dernier.

```ssh
wget http://us.download.nvidia.com/XFree86/Linux-x86_64/525.147.05/NVIDIA-Linux-x86_64-525.147.05.run
```
* URL modifiable (remplacer la version du pilote par le nouveau)

Exécuter l'installation sans le module noyau :

```ssh
chmod +x NVIDIA-Linux-x86_64-525.147.05.run
sudo ./NVIDIA-Linux-x86_64-525.147.05.run --no-kernel-module
```

10 - Modification de la config LXC

Maintenant que les pilotes sont installés, nous devons passer par le GPU.
Arrêtez le conteneur et ajoutez les éléments suivants à la configuration de votre conteneur /etc/pve/lxc/###.conf
En veillant à modifier les nombres que nous avons enregistrés précédemment s'ils diffèrent :

```ssh
#<div align='center'><img src='https%3A//jellyfin.org/images/logo.svg'/></a>
## LXC
#</div>
# Autoriser l'acc%C3%A8s au groupe de contr%C3%B4le 
# Passer par les fichiers de p%C3%A9riph%C3%A9rique 
arch: amd64
cores: 6
features: nesting=1
hostname: jellyfin
memory: 8192
net0: name=eth0,bridge=vmbr0,gw=192.168.1.254,hwaddr=BC:24:11:82:D9:49,ip=192.168.1.50/24,type=veth
onboot: 1
ostype: ubuntu
rootfs: ssd-vm:vm-103-disk-0,size=8G
startup: up=120
swap: 2048
tags:  
lxc.cgroup2.devices.allow: a
lxc.cap.drop: 
lxc.cgroup2.devices.allow: c 188:* rwm
lxc.cgroup2.devices.allow: c 189:* rwm
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 243:* rwm
lxc.mount.entry: /dev/serial/by-id  dev/serial/by-id  none bind,optional,create=dir
lxc.mount.entry: /dev/ttyUSB0       dev/ttyUSB0       none bind,optional,create=file
lxc.mount.entry: /dev/ttyUSB1       dev/ttyUSB1       none bind,optional,create=file
lxc.mount.entry: /dev/ttyACM0       dev/ttyACM0       none bind,optional,create=file
lxc.mount.entry: /dev/ttyACM1       dev/ttyACM1       none bind,optional,create=file
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.cgroup2.devices.allow: c 509:* rwm
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
```

11 - Démarrage du LXC

Démarrez le conteneur et confirmez le relais effectué en exécutant ls -al /dev/nvidia*et ls -al /dev/dri/*. 
De plus, nvidia-smivous devriez maintenant afficher un résultat identique à celui de l'hôte Proxmox :

```ssh
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.147.05   Driver Version: 525.147.05   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P400         On   | 00000000:01:00.0 Off |                  N/A |
| 34%   33C    P8    N/A /  30W |      1MiB /  2048MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```
