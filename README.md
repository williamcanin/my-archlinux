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


## Preparação de Flash Drive

Baixe a imagem do Arch Linux em [Arch Linux Download](https://archlinux.org/download/).
Recomendo usar BitTorrent para evitar corromper a imagem durante o download.
Para grava a imagem, se estiver no Linux, a recomendação é o `dd` com o comando abaixo:

```shell
dd bs=4M if=archlinux-<VERSION>-x86_64.iso of=/dev/sdX conv=fsync oflag=direct status=progress
```

> Nota: Substitua o sdX pelo do seu flash drive.

## Iniciando a instalação

### Layout

**(1)** - Atribuir layout do teclado para `br-abnt2`:

```shell
loadkeys br-abnt2
```

### Conexão com a internet

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


### Particionamento


