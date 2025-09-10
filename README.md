<h1 align="center">
  <a href="https://github.com/williamcanin/my-archlinux">
    <img alt="Arch Linux" src="https://raw.githubusercontent.com/williamcanin/my-archlinux/refs/heads/main/docs/archlinux.png" width="480">
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

Baixe a imagem do Arch Linux em [Arch Linux Download](https://archlinux.org/download/).
Recomendo usar BitTorrent para evitar corromper a imagem durante o download.
Para grava a imagem, se estiver no Linux, a recomendação é o `dd` com o comando abaixo:

```shell
dd bs=4M if=archlinux-<VERSION>-x86_64.iso of=/dev/sdX conv=fsync oflag=direct status=progress
```

> Nota: Substitua o sdX pelo do seu flash drive.

# Iniciando a instalação

## Layout

**(1)** - Atribuir layout do teclado para `br-abnt2`:

```shell
loadkeys br-abnt2
```

## Conexão com a internet

**Via Wi-Fi:**

```shell
systemctl start iwd
iwctl
```

Quando entrar dentro do `[iwd]#`, os passos de comandos serão esses basicamente:

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

**Via Cabo:**

Apenas conecte o cabo de rede e já terá internet.

> NOTA: Após configurar a internet, faça um `ping 8.8.8.8`.


## Particionamento

### Sistema

Aqui é a parte MAIS DELICADA, tendo o MÁXIMO de atenção para não escolher a unidade errada.
Não vou colocar comando de como realizar o particionamento, apenas relatar algumas informações
importantes e o esboso (tabela) de como o particionamento para a instalação do **Arch Linux** deve ficar.

Basicamente o **Arch Linux** precisa apenas de uma partição de boot, a `/boot` do tipo **EFI System**, mas,
caso queira fazer um dual-boot com outra distro, que necessita de duas
partições de boot separadas, uma `/boot` do tipo **Linux filesystems** e outra `/boot/efi` do tipo
**EFI System**, e queira COMPARTILHAR o bootloader, no caso o **systemd-boot** (que eu uso)
entre ambas, então deve instalar o **Arch Linux** com a partição de boot separado em duas.

* `Relato:`
Eu usei duas partições de boot separadas porque queria fazer um dual-boot com o **Fedora +42**, porém,
ao fazer isso, toda vez que atualizava o **Arch Linux** e gerava uma nova modificação do **vmlinuz-linux**
e **initramfs-linux.img**, tinha que fazer uma copia de ambos para a partição **EFI**, no caso, a `/boot/efi`,
isso porque quando esses dois arquivos são gerados/modificados, eles "instalam" por padrão em `/boot`,
e o **systemd-boot** não consegue reconhecer "nada" fora do `/boot/efi`, por isso é NECESSÁRIO a
copia.
* Para fazer essa copia automatizada, eu tive que criar [este hooks](https://github.com/williamcanin/my-archlinux/blob/main/hooks/90-systemd-boot) em `/etc/initcpio/post/90-systemd-boot`.
Geralmente essa configuração é realizada no pos instalação do **Arch Linux**, mas vou deixar relatado aqui mesmo.
* Sabendo disso, nesses guias NÃO VOU USAR duas partição de boot porque não quero fazer dual-boot e
nem compartilhamento do **systemd-boot** com outros sistemas, porém, eu vou deixar as duas tabelas
de como fica ambos os casos, a de uma partição de boot, e a de duas partições de boot.

**Com a partição de boot EFI ÚNICA:**

| Dispositivo | Tamanho |    Tipo     |  Local  |
|-------------|---------|-------------|---------|
| /dev/sda1   | 1,5G    | Sistema EFI | /boot   |
| /dev/sda2   | 221,6G  | Linux LVM   |         |

**Com a partição de boot EFI SEPARADA:**

| Dispositivo | Tamanho |    Tipo           |  Local    |
|-------------|---------|-------------------|-----------|
| /dev/sda1   | 2G      | Linux filesystems | /boot     |
| /dev/sda2   | 2G      | Sistema EFI       | /boot/efi |
| /dev/sda3   | 219,1G  | Linux LVM         |           |

### Home

