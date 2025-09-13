<h1 align="center">
  <a href="https://github.com/williamcanin/my-archlinux">
    <img alt="Arch Linux" src="https://raw.githubusercontent.com/williamcanin/my-archlinux/refs/heads/main/assets/archlinux.png" width="480">
  </a>
</h1>

Ei, beleza? 👍

Este é o guia que fiz, e uso atualmente para instalar o [Arch Linux](https://archlinux.org/) em
minha máquina.

Este guia irá ter detalhes e comentários RESUMIDO de cada comando, caso queira um guia com apenas os
comandos, sem muita "verbosidade" de comentários, use este outro guia que fiz em modo `.txt`:
[https://bit.ly/archlinux_installation](https://bit.ly/archlinux_installation).

> NOTA: Nesses guias, talvez nem tudo sirva para seu gosto e/ou suporte de sua máquina, então se
for usar algo, tenha consciência se é compatível com seu setup. Não me responsabilizo por qualquer
dano que sua máquina venha sofrer.


# Preparação de Flash Drive

Baixei a imagem do Arch Linux em [Arch Linux Download](https://archlinux.org/download/).
Eu uso o BitTorrent para evitar corromper a imagem durante o download, e para gravar a imagem, se
eu estiver no Linux, uso `dd` com o comando abaixo:

```shell
dd bs=4M if=archlinux-<VERSION>-x86_64.iso of=/dev/sdX conv=fsync oflag=direct status=progress
```

> Nota: Substitua o sdX pelo flash drive real.

# Iniciando a instalação

Quando já estou dentro da ISO do **Arch Linux**, sigo esses passos:

# Layout

Atribuo layout do teclado para `br-abnt2`, que é o que eu uso:

```shell
loadkeys br-abnt2
```

# Conexão com a internet

**Via Cabo:**

Apenas conecto o cabo de rede e já tenho internet na ISO do **Arch Linux**.

<details>
  <summary>Via Wi-Fi:</summary>

```shell
systemctl start iwd;
iwctl
```

Quando entrar dentro do `[iwd]#`, os passos de comandos que uso serão esses basicamente:

```shell
device list
device <NAME> set-property Powered on
adapter set-property Powered on
station list
station <IFACE_NAME> scan
station <IFACE_NAME> get-networks
station <IFACE_NAME> connect '<NETWORK_NAME>'
Passphrase:
quit
```
</details>

> NOTA: Após configurar a internet, faço um `ping 8.8.8.8` para verificar. 😆

# Conexão SSH

Geralmente faço a instalação do **Arch Linux** na minha máquina via SSH, ou seja, de outra máquina,
assim eu consigo abrir este guia e apenas copiar e colar os comandos do que ficar digitando cada
comando.

**(1)** - Para habilitar o SSH na ISO do **Arch Linux** é simples, a ISO já vem com o SSH instalado,
 então basta ativar:

```shell
systemct start sshd
```

**(2)** - Após isso crio a senha do root da imagem ISO para poder fazer a conexão SSH:

```shell
passwd
```

**(3)** - Agora na outra máquina apenas me conecto:

```shell
ssh root@192.168.X.XX
```

# Particionamento

Aqui é a parte MAIS DELICADA, tenho o MÁXIMO de atenção para não escolher a unidade errada. hehe 😁

Não vou colocar comandos de como realizar o particionamento, apenas relatar algumas informações
IMPORTANTES e o esboço (tabela) de como o particionamento para a instalação do **Arch Linux** fica
em minha máquina.

## Tabela

| Dispositivo | Tamanho |    Tipo             |  Local  |
|-------------|---------|---------------------|---------|
| /dev/sda1   | 1,5G    | Sistema EFI         | /boot   |
| /dev/sda2   | 120G    | Linux LVM           |         |
| /dev/sdb1   | 1T      | Linux filesystems   | /home   |

<details>
  <summary><strong>>>> Informações interessantes 🤔</strong></summary>

## Boot

O **Arch Linux** precisa apenas de uma partição de boot, a `/boot` do tipo **EFI System**,
MAS, caso queira fazer um dual-boot com outras distros, que necessita de duas
partições de boot separadas, uma `/boot` do tipo **Linux filesystems** e outra `/boot/efi` do tipo
**EFI System**, por exemplo, **Fedora 42**, e queira COMPARTILHAR o bootloader, no caso o
**systemd-boot** (que eu uso) entre ambas, então deve instalar o **Arch Linux** com a partição de
boot separada em duas também.

Toda vez que o **Arch Linux** gera o "**vmlinuz-linux-lts**" e "**initramfs-linux-lts.img**", ele
gera no diretorio `/boot`, isso porque a configuração padrão é para esté diretório, mas com a EFI
apontando para `/boot/efi`, tive que modificar essa configuração no arquivo
`/etc/mkinitcpio.d/linux-lts.preset` e reinstalar o kernel. Na seção de
**Instalando o bootloader systemd-boot** você irá ver informações sobre essa modificação.

* Sabendo disso, nesses guias NÃO VOU USAR duas partição de boot porque não uso mais dual-boot e
nem compartilhamento do **systemd-boot** com outros sistemas, porém, eu vou relatado cada passo que
precisa fazer caso seja uma instalação com `/boot` e `/boot/efi`.

## Sistema

Instalo o **Arch Linux** em um SSD de **250 Gigabytes** (*250Gb*), mas eu apenas deixo **120Gb**,
não uso mais que isso para o sistema **Arch Linux**. Atualmente estou usando o sistema de
arquivo `ext4`.

## Home

Tenho um HDD de **1 Terabyte** (*1Tb*) para minha `/home`, e criptografo a mesma usando o
LUKS (*dm-crypt*), com o sistema de arquivos `ext4`.

## Tabela

Tabela com a partição de boot separada em duas deve ficar assim:

| Dispositivo | Tamanho |    Tipo             |  Local    |
|-------------|---------|---------------------|-----------|
| /dev/sda1   | 2G      | Linux filesystems   | /boot     |
| /dev/sda2   | 2G      | Sistema EFI         | /boot/efi |
| /dev/sda3   | 120G    | Linux LVM           |           |
| /dev/sdb1   | 1T      | Linux filesystems   | /home     |

</details></br></br>

Para realizar o particionamento, geralmente eu uso o `cfdisk`:

```shell
cfdisk /dev/sdX
```

> Nota: Substituo o sdX pelo dispositivo real, `/dev/sda` e `/dev/sdb`.


# Criando estrutura LVM para o sistema

No **LVM**, precisa criar um **Volume Físico** (PV), **Grupo** (VG), e um **Volume Lógico** (LV),
onde o grupo vai fazer parte de um volume físico, e o volume lógico vai estar dentro de um grupo.

Gosto de usar **LVM** para ter controle sobre minhas partições, caso eu queira aumentar ou diminuir
sem ter problema de corromper dados. Para isso, os comando que uso são simples:

```shell
pvcreate /dev/sda3;
vgcreate linux /dev/sda3;
lvcreate -L 120G linux -n arch;
```

> NOTA: No segundo comando, o nome `linux` é o nome do grupo que defino (pode ser qualquer nome), no
terceiro comando, crio um volume lógico especificando o grupo (`linux`).

# Criando e criptografando a unidade /home

**(1)** - Criptografando a unidade `/dev/sdb1`:

```shell
cryptsetup -y -v luksFormat /dev/sdb1;
```

> IMPORTANTE!!! Se você já tem a unidade `/dev/mapper/home` criptografada com seus arquivos não tem
necessidade deste passo **1** senão irá perder os arquivos. PULE para o passo **2**.

**(2)** - Criando/abrindo a unidade criptografada:

```shell
cryptsetup open /dev/sdb1 home;
```

# Formatação

Agora formato cada unidade que foi criada:

```shell
mkfs.fat -F 32 /dev/sda1;
mkfs -t ext4 /dev/mapper/linux-arch;
mkfs -t ext4 /dev/mapper/home;
```

<details>
  <summary><strong>Com partição de boot separada</strong></summary>

```shell
mkfs -t ext4 /dev/sda1;
mkfs.fat -F 32 /dev/sda2;
mkfs -t ext4 /dev/mapper/linux-arch;
mkfs -t ext4 /dev/mapper/home;
```
</details></br>

> IMPORTANTE!!! Se já tiver a partição `/dev/mapper/home` com arquivos, não formatar senão perde
TODOS os arquivos.

Depois de todas unidades estarem criadas e formatadas, gosto de verificar com o comando: `lsblk -f`:

```
NAME        FSTYPE      FSVER     LABEL     UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1      vfat        FAT32               BA60-4D21                                 1,5G    12% /boot
└─sda2      LVM2_member LVM2 001            8YUXnI-FwmY-Vc8V-fUHy-cVdF-zi9X-MDAK0s
  └──linux-arch
           ext4        1.0                  0a73a608-5260-45c8-9bdd-8285c4a4a84b     89,8G    44% /
sdb
└─sdb1      crypto_LUKS 2                   a4fd06b1-a253-4661-b5a2-47ae92e68efe
  └─home    ext4        1.0                 65660251-8451-4722-adbd-ff5850c5df6d    999,7G    37% /home
```

<details>
  <summary><strong>Com partição de boot separada</strong></summary>

```
NAME        FSTYPE      FSVER     LABEL     UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1      ext4        1.0                 69660251-8451-4322-cdbd-ff5850c5df6d      1,5G    12% /boot
├─sda2      vfat        FAT32               BA60-4D21                                 1,5G    12% /boot
└─sda3      LVM2_member LVM2 001            8YUXnI-FwmY-Vc8V-fUHy-cVdF-zi9X-MDAK0s
  └──linux-arch
           ext4        1.0                  0a73a608-5260-45c8-9bdd-8285c4a4a84b     89,8G    44% /
sdb
└─sdb1      crypto_LUKS 2                   a4fd06b1-a253-4661-b5a2-47ae92e68efe
  └─home    ext4        1.0                 65660251-8451-4722-adbd-ff5850c5df6d    999,7G    37% /home
```
</details></br>


# Montagem das unidades

Tudo em ordem, agora faço o `mount` das unidades:

```shell
mount /dev/mapper/linux-arch /mnt;
mount --mkdir /dev/sda1 /mnt/boot;
mount --mkdir /dev/mapper/home /mnt/home;
```

<details>
  <summary><strong>Com partição de boot separada</strong></summary>

```shell
mount /dev/mapper/linux-arch /mnt;
mount --mkdir /dev/sda1 /mnt/boot;
mount --mkdir /dev/sda2 /mnt/boot/efi;
mount --mkdir /dev/mapper/home /mnt/home;
```
</details></br>

# Instalando o sistema base do Arch Linux

Aqui eu atualizo os `mirrorlist` para o `Brazil` e `US` usando `reflector` já disponível na ISO do
**Arch Linux**, e logo em seguida atualizo as chaves e o cache, para depois fazer instalação do
sistema base com o kernel LTS, e alguns pacotes que acho essenciais durante a instalação.

```shell
reflector --verbose --country Brazil,US --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist;
pacman -Syy;
pacman -Sy archlinux-keyring;
pacman-key --populate archlinux;
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux-firmware systemd systemd-ukify sudo vim dhcpcd wireless_tools wpa_supplicant;
```

# Gerando o /etc/fstab

Aqui não tenho muito o que dizer, apenas gero o `/etc/fstab` para que todas minhas partições montadas
sejam configuradas durando o boot da máquina.


```shell
genfstab -U -p /mnt >> /mnt/etc/fstab
```

# Entrando no sistema pré-instalado

```shell
arch-chroot /mnt /bin/bash
```

# Atribuindo senha de `root`

A primeira coisa que gosto de fazer é atribuir uma senha para o usuário `root`:

```shell
passwd
```

# Configurando internet

Como atualmente uso uma conexão via cabo, não tenho necessidade de usar o `NetworkManager` como
gerenciador de conexão com internet para ficar me dando várias configurações insignificantes.

Eu apenas quero me conectar e pronto. Acho ele um pouco pesado em consumo de memória pra uma
finalidade muito específica.

Então, eu uso o `systemd-networkd` que é mais leve e objetivo.

**(1)** - Caso eu já tenho o `NetworkManager` instalado, eu apenas desabilito e faço o `mask`:

```shell
systemctl disable --now NetworkManager.service;
systemctl mask NetworkManager.service;
```

**(2)** - Depois habilito o `systemd-networkd` e `systemd-resolved`:

```shell
systemctl enable --now systemd-networkd.service systemd-resolved.service
```

**(3)** - Abro o arquivo de configuração do `systemd-networkd` e coloco o seguinte:

```conf
[Match]
Name=eno1 # Nome da minha interface de rede

## Conexão com IP Estático
[Network]
Address=192.168.0.2/24
Gateway=192.168.0.1
DNS=8.8.8.8

## Conexão via DHCP
# [Network]
# DHCP=yes
```

**(4)** - Depois crio um link simbólico para o `DNS`:

```shell
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

# Configurando o Pacman

Aqui habilito o repositório `[multilib]` e ignoro alguns pacotes de serem instalados e atualizados.

> NOTA: Como eu uso kernel *LTS*, não tenho mania de ficar atualizando kernel sempre, e também
não uso os driver da minha GPU (**NVIDIA**) diretamente do repo do **Arch Linux**. Como o
**Arch Linux** é rolling-release e sempre disponibiliza a "última" versão dos pacotes, tive alguns
problemas com a útilma versão da **NVIDIA** em relação a minha GPU 😠, então instalo o driver (`.run`)
baixado do próprio [site da NVIDIA](https://www.nvidia.com/en-us/drivers/unix/) com uma versão
anterior, mas especificamente a *Latest New Feature Branch Version*.

**(1)** - Abro o **/etc/pacman.conf**:

```shell
vim /etc/pacman.conf
```

**(2)** - Descomento as seguintes linhas do `[multilib]` deixando assim:

```conf
[multilib]
Include = /etc/pacman.d/mirrorlist
```

**(3)** - Ignoro atualização/instalação de alguns pacotes do repo do **Arch Linux** que não uso,
adicionando o seguintes:

```conf
IgnorePkg  = linux-lts linux linux-zen linux-headers linux-zen-headers linux-lts-headers
nvidia-utils nvidia-settings nvidia lib32-nvidia cuda
```

**(4)** - Adiciono meu próprio repo de algumas configurações que fiz para minha máquina 😎:

```conf
[canin]
SigLevel = Optional TrustAll
Server = https://williamcanin.gitlab.io/archlinux/stable/x86_64
```

**(5)** - Atualizo o cache do pacman:

```shell
pacman -Syy
```

# Configurando /home

O arquivo de configuração para dispositivos criptografados lançados "durante o boot" no
**Arch Linux**, é o `/etc/crypttab.initramfs`. Por padrão ele não existe, então eu crio o mesmo
atribuindo o `UUID` do dispositivo criptografado LUKS, que no caso é o `/dev/sdb1`.

**(1)** - Crio o arquivo `/etc/crypttab.initramfs` inserindo o `UUID` com a ajuda do `blkid`:

```shell
cat << EOF >> /etc/crypttab.initramfs
# /dev/sdb1
home UUID=$(blkid -s UUID -o value /dev/sdb1) none luks,tries=0,timeout=0
EOF
```

> ATENÇÃO!!! Não confundir `/dev/sdb1` (dispositivo LUKS) com `/dev/mapper/home`
(partição com sistema de arquivos).

**(2)** - Agora para a partição `/dev/mapper/home` iniciar com o sistema, insiro a mesma no arquivo
`/etc/fstab`. Sigo basicamente a mesma ideia do comando de acima, copiando o `UUID` com o `blkid`
mas dessa vez no `/etc/fstab`:

```shell
cat << EOF >> /etc/fstab
# /dev/mapper/home
UUID=$(blkid -s UUID -o value /dev/mapper/home) /home ext4 rw,relatime,data=ordered 0 2
EOF
```

> ATENÇÃO!!! Observe que para inserir a configuração no `/etc/fstab`, estou usando **tee -a**, este
parâmetro **-a** significa **append**, adicionar, se emitir ele, irá sobrescrever o `/etc/fstab`.

# Configurando o /etc/mkinitcpio.conf

**(1)** - Aqui adiciono os módulos que preciso que carreguem durante o boot:

```shell
sed -i "s|^MODULES=.*|MODULES=(usbhid xhci_hcd ehci_hcd)|g" /etc/mkinitcpio.conf
```

O driver `usbhid` é esencial para reconhecer dispositivos como teclados e mouses que se conectam
via USB. Já o `xhci_hcd` e `ehci_hcd` são responsáveis por fazer a ponte entre o hardware e os
dispositivos USB.

> Bug: Como eu tenho a partição `/home` criptografada que necessita colocar a senha durante o boot,
eu não acrescento os módulos da minha GPU (NVIDIA) para não dar "*flicker*" na tela durante o boot
ocasionando quebra de linha do cursor no passphrase da `/home`.

**(2)** - Nos HOOKS adiciono a opção de criptografia e **LVM**. Então faço assim:

```shell
sed -i "s|^HOOKS=.*|HOOKS=(base systemd autodetect keymap modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)|g" /etc/mkinitcpio.conf
```

> Nota: Antigamente eu usava o `plymouth` depois de `keymap` para ter um boot com splash, mas hoje
prefiro o boot verboso para averiguar alguma mensagem de erro, ou demora caso ocorra.

**(3)** - Agora instalo o pacote `lvm2`:

```shell
pacman -S lvm2
```

# Instalando o bootloader

Já faz um bom tempo que uso `systemd-boot` por achar o `GRUB` pesado e com recursos que nem preciso.

Minha máquina é **EFI**, por que eu teria que ter um bootloader pra gerenciar *Legacy* também?! 🤔


**(1)** - Primeiro instalo o `efibootmgr` e `intel-ucode`:

```shell
pacman -S --noconfirm efibootmgr intel-ucode
```

> Nota: O `efibootmgr` é um "gerenciador" de bootloader EFI, e o `intel-ucode` é um microcódigo de
segurança para CPU Intel. Caso eu tenha AMD como CPU, instalo o `amd-code`.

**(2)** - Depois faço de fato a instalação o `systemd-boot` como bootloader:

```shell
bootctl --path=/boot install
```

<details>
  <summary><strong>Com partição de boot separada</strong></summary>

Se instalou o sistema com a partição de boot separada, então a instalação é assim:

```shell
bootctl --path=/boot/efi install
```
</details></br>

**(3)** - Crio o loader do `systemd-boot`:

```shell
ESP_DIR="";
cat << EOF > /boot/${ESP_DIR}loader/loader.conf
default arch.conf
timeout 3
console-mode max
editor no
EOF
```

> IMPORTANTE: Está variável de ambiente `ESP_DIR` é temporária, é apenas para o momento de instalação.
Se instalou o sistema com a partição de boot separada, um em `/boot` e outra `/boot/efi`, então a
variável `ESP_DIR` DEVE ser assim: `ESP_DIR="efi/"`. Caso contrário deixe vazio.

## Usando UKI (Unified Kernel Image)

Atualmente estou usando `systemd-boot` + `UKI` (Unified Kernel Image) ❤️, e esses são os passos que
faço para instalar.

**(1)** - Crio um backup do "preset" primeiro:

```shell
cp /etc/mkinitcpio.d/linux-lts.preset /etc/mkinitcpio.d/linux-lts.preset.backup
```

**(2)** - Depois crio um novo `/etc/mkinitcpio.d/linux-lts.preset` com as configurações abaixo:

```shell
cat << EOF > /etc/mkinitcpio.d/linux-lts.preset
ESP_DIR="${ESP_DIR}"

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/\${ESP_DIR}vmlinuz-linux-lts"
ALL_cmdline="root=UUID=$(blkid -s UUID -o value /dev/mapper/linux-arch) rw loglevel=3 nvidia_drm.modeset=1 video=1920x1080@75"
PRESETS=('default' 'fallback')

default_config="/etc/mkinitcpio.conf"
default_image="/boot/\${ESP_DIR}initramfs-linux-lts.img"
default_uki="/boot/\${ESP_DIR}EFI/Linux/arch-linux-lts.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

fallback_config="/etc/mkinitcpio.conf"
fallback_image="/boot/\${ESP_DIR}initramfs-linux-lts-fallback.img"
fallback_uki="/boot/\${ESP_DIR}EFI/Linux/arch-linux-lts-fallback.efi"
fallback_options="-S autodetect"
EOF
```

> **IMPORTANTE:** Se usar a EFI fora do `/boot`, em `/boot/efi`, deixar assim `ESP_DIR="efi/"`.

> Dica: Caso eu queira um boot menos verboso e com splash, eu adiciono na opção `ALL_cmdline` os
parâmentros: `quiet splash loglevel=3 systemd.show_status=auto rd.udev.log_level=3`. E depois
instalo o pacote `plymouth`, e adiciono a flag `plymouth` nos HOOKS do `/etc/mkinitcpio.conf` depois
de `keymap`.

**(3)** - Agora crio as entradas do `systemd-boot` padrão:

```shell
cat << EOF > /boot/${ESP_DIR}loader/entries/arch.conf
title   Arch Linux LTS
efi     /EFI/Linux/arch-linux-lts.efi
EOF
```

**(4)** - E por final, crio as entradas do `systemd-boot` de fallback:

```shell
cat << EOF > /boot/${ESP_DIR}loader/entries/arch-fallback.conf
title   Arch Linux LTS (Fallback)
efi     /EFI/Linux/arch-linux-lts-fallback.efi
EOF
```

**(5)** - Reinstalo o kernel:

```shell
pacman -S --noconfirm linux-lts
```


<details>
  <summary><strong>Usando Padrão</strong></summary>

Aqui a configuração do systemd-boot muda, em vez de usar UKI, usa os arquivos
**vmlinuz-linux-lts**, **initramfs-linux-lts.img** e **intel-ucode.img** da iniciar o boot.

**(1)** - Primeiro remover qualquer imagem `.efi` gerada:

```shell
rm -f /boot/${ESP_DIR}EFI/Linux/arch-linux-lts.efi /boot/${ESP_DIR}EFI/Linux/arch-linux-lts-fallback.efi
```

**(2)** - Depois eu crio a entrada padrão assim:

```shell
cat << EOF > /boot/${ESP_DIR}loader/entries/arch.conf
title Arch Linux (Default)
linux /vmlinuz-linux-lts
initrd  /intel-ucode.img
initrd /initramfs-linux-lts.img
options root=UUID=$(blkid -s UUID -o value /dev/mapper/linux-arch) rw nvidia_drm.modeset=1 video=1920x1080@75
EOF
```

**(2)** - E a entrada de fallback assim:

```shell
cat << EOF > /boot/${ESP_DIR}loader/entries/arch-fallback.conf
title Arch Linux (Fallback)
linux /vmlinuz-linux-lts-fallback
initrd  /intel-ucode.img
initrd /initramfs-linux-ltsfallback.img
options root=UUID=$(blkid -s UUID -o value /dev/mapper/linux-arch) rw nvidia_drm.modeset=1 video=1920x1080@75
EOF
```
</details></br>

# Instalação de drivers gráficos

Agora vamos de fatos ir para ambiente gráfico. Então começo a instalar alguns drivers essenciais e
API, como **Vulkan**, **OpenGL**, etc:

```shell
pacman -S --needed --noconfirm xorg wayland dialog mesa lib32-mesa xf86-video-vesa vulkan-icd-loader \
lib32-vulkan-icd-loader vulkan-tools
```

**Intel:**

Como uso [Intel](https://www.intel.com.br/content/www/br/pt/products/details/processors/core.html), então também instalo esses drivers para GPU integrada:

```shell
pacman -S --needed --noconfirm mesa-vulkan-intel vulkan-intel linux-firmware-intel
```

<details>
  <summary><strong>AMD</strong></summary>

Não estou usando AMD, mas vou deixar os drivers necessários caso eu use futuramente:

```shell
pacman -S --needed --noconfirm mesa-vulkan-radeon vulkan-radeon linux-firmware-radeon
```
</details></br>

**NVIDIA (Nouveau)**

Sempe bom ter os drivers da NVIDIA open-source caso a NVIDIA faça alguma nhaca de incompatibilidade:

```shell
pacman -S --noconfirm  xf86-video-nouveau vulkan-nouveau
```

<details>
  <summary><strong>NVIDIA (proprietary) 🙄</strong></summary>

Como já relatei acima, não uso o driver proprietário da NVIDIA do repo do **Arch Linux** por
algumas incompatibilidades que tive na última versão 😡, mas mesmo assim vou deixar os pacotes
essenciais que se deve instalar:

```shell
pacman -S --needed --noconfirm nvidia nvidia-utils lib32-nvidia-utils nvidia-settings opencl-nvidia;
systemctl set-default multi-user.target
```
</details></br>

# Instalação de drivers de áudio

```shell
pacman -S --needed --noconfirm pipewire pipewire-audio pipewire-pulse pipewire-alsa pipewire-jack \
lsp-plugins-lv2 mda.lv2 zam-plugins-lv2 zam-plugins-lv2
```

# Ambiente de trabalho (XFCE)

Meu ambiente principalmente atualmente é o XFCE ❤️, leve e funcional:

```shell
pacman -S --needed --noconfirm xfce4 xfce4-goodies appmenu-gtk-module libdbusmenu-glib lightdm \
lightdm-gtk-greeter
```

<details>
  <summary><strong>Instalação do ambiente de trabalho (GNOME)</strong></summary>

Minha realação com GNOME é entre amor e ódio. Instalo mas deixo com um ambiente de fallback:

**Mínimo:**

```shell
pacman -S --needed --noconfirm gnome-shell gnome-control-center gnome-terminal nautilus \
gnome-browser-connector gnome-shell-extensions gnome-tweaks gdm
```

**Completo:**

```shell
pacman -S --needed --noconfirm gnome gnome-extra gnome-desktop gnome-shell-extensions \
gnome-browser-connector gnome-tweaks gdm
```
</details></br>

# Instalação de aplicações

Algumas aplicações básicas que uso:

```shell
pacman -S --needed --noconfirm pacman-contrib dkms xdg-user-dirs ntfs-3g udisks2 dosfstools mtools \
cpupower reflector samba git openssh tor virtualbox-guest-utils vlc transmission-gtk gvfs gvfs-smb \
ttf-dejavu ttf-dejavu-nerd terminator veracrypt zip unzip xarchiver gimp inkscape pavucontrol make \
gcc go ruby perl tk python nodejs npm arch-wiki-docs arch-wiki-lite zeal qemu-full virt-manager \
piper steam-native-runtime firefox libreoffice-fresh libreoffice-fresh-pt-br terminator galculator \
leafpad calf smplayer gparted rofimoji easyeffects gnome-keyring seahorse
```

# Habilitando serviços

Neste momento habilito alguns servições para iniciar durante o boot:

```shell
systemctl enable iptables.service smb.service nmb.service tor.service
```

# Complementando o /etc/fstab

Meu computador não tem leitor de disquete e CD/DVD (e quem tem?), mas mesmo asim eu mantenho a
configuração no `/etc/fstab`, e também já deixa comentado para uma partição **Windows**, caso eu
tenha um dia. Para essas configurações, eu faço os comandos:

```shell
mkdir -p /media/cdrom0; mkdir /mnt/floppy; mkdir /mnt/windows;
ln -s /media/cdrom0 /media/cdrom;
cat << EOF >> /etc/fstab
### CDROM
/dev/sr0  /media/cdrom0  udf,iso9660 user,noauto  0 0

### Floppy
/dev/fd0  /mnt/floppy  auto  defaults,user,noauto  0 0

### Windows (optional)
#UUID=XXXXX-XXXXX-XXXXX /mnt/windows  ntfs-3g defaults,user,rw,auto  0 0
EOF
```

# ZRAM

Geralmente não forço tanto meu computador a ponto de usar **zram**, mas mesmo assim eu configuro:

**1** - Instalando o gerenciador de zram:

```shell
pacman -S --needed --noconfirm zram-generator
```

**2** - Configurando um perfil equilibrado para **zram**:

```shell
cat << "EOF" > /etc/systemd/zram-generator.conf
[zram0]
zram-size = ram / 4
compression-algorithm = zstd
swap-priority = 50
fs-type = swap
EOF
```

> Nota: Caso eu queira um perfil mais agressivo, para jogar por exemplo, que necessite de **zram**,
então eu uso este abaixo:

<details>
  <summary><strong>ZRAM: Perfil Agressivo </strong></summary>

```shell
cat << "EOF" > /etc/systemd/zram-generator.conf
[zram0]
zram-size = ram * 3/4
compression-algorithm = lz4
swap-priority = 100
fs-type = swap
EOF
```
</details></br>

**3** - Depois de configurar, eu faço um *reset* no daemon e habilito o serviço de **ZRAM**:

```shell
systemctl daemon-reload;
systemctl enable --now systemd-zram-setup@zram0.service
```

# Adicionando um usuário

**1** - Antes, vou liberar o grupo `sudo` no arquivo `/etc/sudoers` para meu usuário pertencer a
esse grupo e ter privilégios de sudo:

```shell
sed -i "s|# %sudo ALL=(ALL:ALL) ALL|%sudo ALL=(ALL:ALL) ALL|g" /etc/sudoers
```

**2** - Agora começo a criação do grupo do meu usuário, e a criação do meu usuário em si:

```shell
USERNAME_TEMP="will";
groupadd $USERNAME_TEMP;
useradd -m -g $USERNAME_TEMP -G users,tty,wheel,games,power,optical,storage,scanner,lp,audio,video,input,mail,root -s /bin/zsh $USERNAME_TEMP;
groupadd sudo -U $USERNAME_TEMP;
passwd $USERNAME_TEMP;
```

# Idioma e localidade

Esses comandos são necessário para configurar o teclado e idioma do sistema, onde cada linha é um
comando:

```shell
timedatectl set-timezone America/Sao_Paulo;
echo "KEYMAP=br-abnt2" > /etc/vconsole.conf;
sed -i "s|#en_US.UTF-8 UTF-8|en_US.UTF-8 UTF-8|g" /etc/locale.gen;
sed -i "s|#pt_BR.UTF-8 UTF-8|pt_BR.UTF-8 UTF-8|g" /etc/locale.gen;
locale-gen;
echo LANG=pt_BR.UTF-8 | tee /etc/locale.conf;
rm -f /etc/localtime && ln -s /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime;
hwclock --systohc;
echo "archlinux" | tee /etc/hostname;
printf "127.0.0.1        archlinux\n" >> /etc/hosts;
echo KEYMAP=br-abnt2 | tee /etc/vconsole.conf;
```
