<h1 align="center">
  <a href="https://github.com/williamcanin/my-archlinux">
    <img alt="Arch Linux" src="https://raw.githubusercontent.com/williamcanin/my-archlinux/refs/heads/main/assets/archlinux.png" width="480">
  </a>
</h1>

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

## Layout

Atribuo layout do teclado para `br-abnt2`, que é o que eu uso:

```shell
loadkeys br-abnt2
```

## Conexão com a internet

**Via Cabo:**

Apenas conecto o cabo de rede e já tenho internet na ISO do **Arch Linux**.

> NOTA: Após configurar a internet, faço um `ping 8.8.8.8` para verificar.

<details>
  <summary>Via Wi-Fi:</summary>

```shell
systemctl start iwd
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

## Particionamento

Aqui é a parte MAIS DELICADA, tenho o MÁXIMO de atenção para não escolher a unidade errada. hehe

Não vou colocar comandos de como realizar o particionamento, apenas relatar algumas informações
IMPORTANTES e o esboço (tabela) de como o particionamento para a instalação do **Arch Linux** fica
em minha máquina.

### Tabela

| Dispositivo | Tamanho |    Tipo             |  Local  |
|-------------|---------|---------------------|---------|
| /dev/sda1   | 1,5G    | Sistema EFI         | /boot   |
| /dev/sda2   | 120G    | Linux LVM           |         |
| /dev/sdb1   | 1T      | Linux filesystems   | /home   |

<details>
  <summary><strong>>>> Informações</strong></summary>

### Boot

O **Arch Linux** precisa apenas de uma partição de boot, a `/boot` do tipo **EFI System**,
MAS, caso queira fazer um dual-boot com outra distro, que necessita de duas
partições de boot separadas, uma `/boot` do tipo **Linux filesystems** e outra `/boot/efi` do tipo
**EFI System**, e queira COMPARTILHAR o bootloader, no caso o **systemd-boot** (que eu uso)
entre ambas, então deve instalar o **Arch Linux** com a partição de boot separada em duas.

* `Relato:`
Uma vez usei duas partições de boot separadas porque queria fazer um dual-boot com o **Fedora +42**, porém,
ao fazer isso, toda vez que atualizava o **Arch Linux** e gerava uma nova modificação do **vmlinuz-linux**
e **initramfs-linux.img**, tinha que fazer uma copia de ambos para a partição **EFI**, no caso, para
a `/boot/efi`, isso porque quando esses dois arquivos são gerados/modificados, eles "instalam" por
padrão em `/boot`, e o **systemd-boot** não consegue reconhecer "nada" fora do `/boot/efi`, por isso
é NECESSÁRIO a copia.
* Para fazer essa copia automatizada, eu tive que criar [este hooks](https://github.com/williamcanin/my-archlinux/blob/main/hooks/90-systemd-boot) em `/etc/initcpio/post/90-systemd-boot`.
Geralmente essa configuração é realizada no pos instalação do **Arch Linux**, mas vou deixar relatado aqui mesmo.
* Sabendo disso, nesses guias NÃO VOU USAR duas partição de boot porque não uso mais dual-boot e
nem compartilhamento do **systemd-boot** com outros sistemas, porém, eu vou deixar as duas tabelas
de como fica ambos os casos, a de uma partição de boot, e a de duas partições de boot.

### Sistema

Instalo o **Arch Linux** em um SSD de 250 Gigabytes (250Gb), mas eu apenas deixo 120Gb, não uso mais
que isso para o sistema. Atualmente estou usando o sistema de arquivo `ext4`.

### Home

Tenho um HDD de 1 Terabyte (1Tb) para minha `/home`, e criptografo a mesma usando o LUKS (dm_crypt),
com o sistema de arquivos `ext4`.

### Tabela

Tabela com a partição de boot separada em duas:

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


## Criando estrutura LVM para o sistema

No **LVM**, precisa criar um **Volume Físico** (PV), **Grupo** (VG), e um **Volume Lógico** (LV),
onde o grupo vai fazer parte de um volume físico, e o volume lógico vai estar dentro de um grupo.

Gosto de usar **LVM** para ter controle sobre minhas partições, caso eu queira aumentar ou diminuir
sem ter problema de corromper dados. Para isso, os comando que uso são simples:

```shell
pvcreate /dev/sda3
vgcreate linux /dev/sda3
lvcreate -L 120G linux -n arch
```

> NOTA: No segundo comando, o nome `linux` é o nome do grupo que defino (pode ser qualquer nome), no
terceiro comando, crio um volume lógico especificando o grupo (`linux`).

## Criando e criptografando a unidade /home

```shell
cryptsetup -y -v luksFormat /dev/sdb1
cryptsetup open /dev/sdb1 home
```

## Formatação

Agora formato cada unidade que foi criada:

```shell
mkfs.fat -F 32 /dev/sda1
mkfs -t ext4 /dev/mapper/linux-arch
mkfs -t ext4 /dev/mapper/home
```

<details>
  <summary><strong>Com partição de boot separada</strong></summary>

```shell
mkfs -t ext4 /dev/sda1
mkfs.fat -F 32 /dev/sda2
mkfs -t ext4 /dev/mapper/linux-arch
mkfs -t ext4 /dev/mapper/home
```
</details></br>

> IMPORTANTE! Se já tiver a partição `/dev/mapper/home`, não formatar senão perde TODOS os dados.

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

## Montagem das unidades

Tudo em ordem, agora faço o `mount` das unidades:

```shell
mount /dev/mapper/linux-arch /mnt
mount --mkdir /dev/sda1 /mnt/boot
mount --mkdir /dev/mapper/home /mnt/home
```

<details>
  <summary><strong>Com partição de boot separada</strong></summary>

```shell
mount /dev/mapper/linux-arch /mnt
mount --mkdir /dev/sda1 /mnt/boot
mount --mkdir /dev/sda2 /mnt/boot/efi
mount --mkdir /dev/mapper/home /mnt/home
```
</details></br>

## Instalando o sistema base do Arch Linux

Aqui eu atualizo os `mirrorlist` para o `Brazil` e `US` usando `reflector` já disponível na ISO do
**Arch Linux**, e logo em seguida atualizo as chaves e o cache, para depois fazer instalação do
sistema base com o kernel LTS, e alguns pacotes que acho essenciais durante a instalação.

```shell
reflector --verbose --country Brazil,US --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
pacman -Syy
pacman -Sy archlinux-keyring
pacman-key --populate archlinux
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux-firmware systemd systemd-ukify sudo vim dhcpcd wireless_tools wpa_supplicant
```

## Gerando o /etc/fstab

Aqui não tenho muito o que dizer, apenas gero o `/etc/fstab` para que todas minhas partições montadas
sejam configuradas durando o boot da máquina.


```shell
genfstab -U -p /mnt >> /mnt/etc/fstab
```

## Entrando no sistema pré-instalado

```shell
arch-chroot /mnt /bin/bash
```

### Atribuindo senha de `root`

A primeira coisa que gosto de fazer é atribuir uma senha para o usuário `root`:

```shell
passwd
```

### Configurando o Pacman

Aqui habilito o repositório `[multilib]` e ignoro alguns pacotes de serem instalados e atualizados.

> NOTA: Como eu uso kernel *LTS*, não tenho mania de ficar atualizando kernel sempre, e também
não uso os driver da minha GPU (**NVIDIA**) diretamente do repo do **Arch Linux**. Como o **Arch Linux** é
rolling-release e sempre disponibiliza a "última" versão dos pacotes, tive alguns problemas com
a útilma versão da **NVIDIA** em relação a minha GPU, então instalo o driver (`.run`) baixado do próprio
[site da NVIDIA](https://www.nvidia.com/en-us/drivers/unix/) com uma versão anterior, mas especificamente
a *Latest New Feature Branch Version*.

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
adicionando as seguintes linhas:

```conf
IgnorePkg  = linux-lts linux linux-zen linux-headers linux-zen-headers linux-lts-headers
nvidia-utils nvidia-settings nvidia lib32-nvidia cuda
```

**(4)** - Atualizo o cache do pacman:

```shell
pacman -Syy
```

### Configurando a rede de internet

Como atualmente uso uma conexão via cabo, não tenho necessidade de usar o `NetworkManager` como
gerenciador de conexão com internet para ficar me dando várias configurações. Eu apenas quero
me conectar e pronto. Acho ele um pouco pesado em consumo de memória pra uma conexão
muito específica.

Então, eu uso o `systemd-networkd`, e para configurar faço assim:

**(1)** - Caso eu já tenho o `NetworkManager` instalado, eu apenas desabilito e faço o `mask`:

```shell
systemctl disable --now NetworkManager.service
systemctl mask NetworkManager.service
```

**(2)** - Habilito o `systemd-networkd` e `systemd-resolved`:

```shell
systemctl enable --now systemd-networkd.service systemd-resolved.service
```

**(3)** - Abro o arquivo de configuração do `systemd-networkd` e coloco o seguinte:

```conf
[Match]
Name=eno1 # Nome da minha interface de rede

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

### Configurando /home criptografado

O arquivo de configuração para dispositivos criptografados lançados "durante o boot" no **Arch Linux**,
é o `/etc/crypttab.initramfs`. Por padrão ele não existe, então eu crio o mesmo atribuindo o `UUID` do
dispositivo criptografado LUKS, que no caso é o `/dev/sdb1`.

**(1)** - Crio o arquivo `/etc/crypttab.initramfs` inserindo o `UUID` com a ajuda do `blkid`:

```shell
echo "\n\n #/dev/sdb1" | tee -a /etc/crypttab.initramfs
echo "home UUID=$(blkid -s UUID -o value /dev/sdb1) none luks,tries=0,timeout=0" | tee -a /etc/crypttab.initramfs
```

> ATENÇÃO!!! Não confundir `/dev/sdb1` (dispositivo LUKS) com `/dev/mapper/home` (partição com sistema de arquivos).

**(2)** - Agora para a partição `/dev/mapper/home` iniciar com o sistema, insiro a mesma no arquivo
`/etc/fstab`. Sigo basicamente a mesma ideia do comando de acima, copiando o `UUID` com o `blkid` mas dessa vez no `/etc/fstab`:

```shell
echo "\n\n #/dev/mapper/home" | tee -a /etc/fstab
echo "UUID=$(blkid -s UUID -o value /dev/mapper/home) /home ext4 rw,relatime,data=ordered 0 2" | tee -a /etc/fstab
```

> ATENÇÃO!!! Observe que para inserir a configuração no `/etc/fstab`, estou usando **tee -a**, este
parâmetro **-a** significa **append**, adicionar, se emitir ele, irá sobrescrever o `/etc/fstab`.

### Configurando MODULES e HOOKS do /etc/mkinitcpio.conf

**(1)** - Aqui adiciono os módulos que preciso:

```shell
sed -i "s|^MODULES=.*|MODULES=(usbhid xhci_hcd ehci_hcd)|g" /etc/mkinitcpio.conf
```

O driver `usbhid` é esencial para reconhecer dispositivos como teclados e mouses que se conectam
via USB. Já o `xhci_hcd` e `ehci_hcd` são responsáveis por fazer a ponte entre o hardware e os
dispositivos USB.

> Nota: Como eu tenho a partição `/home` criptografada que necessita colocar a senha durante o boot,
eu não acrescento os módulos da minha GPU (NVIDIA) para não dar "*flicker*" na tela durante o boot
ocasionando quebra de linha do cursor no passphrase da `/home`.

**(2)** - Nos HOOKS adiciono a opção de criptografia e LVM. Antigamente eu usava o `plymouth`
depois de `keymap` para ter um boot com splash, mas hoje prefiro o boot verboso para averiguar
alguma mensagem de erro, ou demora caso ocorra. Então faço assim:


```shell
sed -i "s|^HOOKS=.*|HOOKS=(base systemd autodetect keymap modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)|g" /etc/mkinitcpio.conf
```

**(3)** - Agora instalo o pacote `lvm2`:

```shell
pacman -S lvm2
```

### Instalando o bootloader systemd-boot

Já faz um bom tempo que uso `systemd-boot` por achar o `GRUB` pesado e com recursos que nem preciso.
Minha máquina é EFI, por que eu teria que ter um bootloader pra gerenciar Legacy também?!

Atualmente estou usando `systemd-boot` + `UKI` (Unified Kernel Image), e esses são os passos que
faço para instalar.

**(1)** - Primeiro instalo o `efibootmgr` e `intel-ucode` (O `efibootmgr` é um "gerenciador" de bootloader
EFI, e o `intel-ucode` é um microcódigo de segurança para CPU):

```shell
pacman -S --noconfirm efibootmgr intel-ucode
```

> Nota: Caso tenha AMD como CPU, instalar o `amd-code`.

**(2)** - Depois faço de fato a instalação o `systemd-boot` como bootloader com o comando abaixo:

```shell
bootctl --path=/boot install
```

**(3)** - Abro o `/etc/mkinitcpio.d/linux-lts.preset` com `vim` e adiciono a configuração abaixo:

```conf
ESP_DIR="<ESP_DIR>"

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="${ESP_DIR}/vmlinuz-linux-lts"
ALL_cmdline="root=UUID=<UUID> rw loglevel=3 nvidia_drm.modeset=1 video=1920x1080@75"
PRESETS=('default' 'fallback')

default_config="/etc/mkinitcpio.conf"
default_image="${ESP_DIR}/initramfs-linux-lts.img"
default_uki="${ESP_DIR}/EFI/Linux/arch-linux-lts.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

fallback_config="/etc/mkinitcpio.conf"
fallback_image="${ESP_DIR}/initramfs-linux-lts-fallback.img"
fallback_uki="${ESP_DIR}/EFI/Linux/arch-linux-lts-fallback.efi"
fallback_options="-S autodetect"
```

**(4)** - Altero o `<UUID>` pelo `UUID` da partição `/dev/mapper/linux-arch` com o comando:

```shell
ESP_DIR="/boot"
sed -i "s|<ESP_DIR>|$ESP_DIR|g" /etc/mkinitcpio.d/linux-lts.preset
sed -i "s|<UUID>|$(blkid -s UUID -o value /dev/mapper/linux-arch)|g" /etc/mkinitcpio.d/linux-lts.preset
```

> IMPORTANTE: Se usa a EFI fora do `/boot`, em `/boot/efi`, deixe assim `ESP_DIR="/boot/efi"`.

**(5)** - Crio as entradas do `systemd-boot` padrão:

```shell

echo "title   Arch Linux LTS" | tee -a /boot/loader/entries/arch.conf
echo "efi     $ESP_DIR/EFI/Linux/arch-linux-lts.efi" | tee -a /boot/loader/entries/arch.conf
```

**(6)** - Crio as entradas do `systemd-boot` de fallback:

```shell
echo "title   Arch Linux LTS (Fallback)" | tee -a /boot/loader/entries/arch-fallback.conf
echo "efi     $ESP_DIR/EFI/Linux/arch-linux-lts-fallback.efi" | tee -a /boot/loader/entries/arch-fallback.conf
```

**(7)** - Reinstalo o kernel:

```shell
pacman -S --noconfirm linux-lts
```
