# Guide d'installation

[← Retour au README](../README.md) · [Problèmes rencontrés](problemes.md)

Montage du matériel et installation d'Ubuntu Server 26.04 LTS sur le SSD NVMe du Raspberry Pi 5.

## Sommaire

- [Montage](#montage)
- [Installation du système](#installation-du-système)
- [Après l'installation](#après-linstallation)
- [Références](#références)

## Montage

1. **Refroidisseur.** Dans mon cas, les pads thermiques étaient fournis à part. Retirer **les deux films** de chaque pad, poser chaque pad sur sa puce, descendre le radiateur bien droit, enfoncer les clips jusqu'au clic, puis brancher le ventilateur sur la prise `FAN`.
2. **Câble plat (FFC).** Les deux connecteurs ne s'ouvrent pas de la même façon : sur le Pi, le loquet **coulisse vers le haut** ; sur le X1001, il **pivote**. Ouvrir le loquet, insérer le câble jusqu'à sentir une résistance, puis refermer le loquet. L'extrémité marquée `RPi5` va sur le Pi. La partie colorée (le renfort) du câble regarde **vers l'extérieur**, pas vers le Pi.
3. **Empilage.** Fixer le X1001 au-dessus du Pi avec les 3 entretoises de 17 mm et les vis M2.5×5.
4. **SSD.** L'insérer en biais (environ 30°) dans le connecteur M.2, l'encoche face au détrompeur, l'abaisser puis le fixer avec la vis M2×4 sur le plot du format 2280.
5. **Tester avant de fermer le boîtier** (voir l'installation ci-dessous) : un câble mal enfoncé se corrige bien plus facilement boîtier ouvert.
6. **Boîtier.** Le câble plat se retrouve plaqué contre la paroi, comme dans la vidéo officielle. Vérifier qu'il ne fait pas de pli net et qu'il n'est pincé nulle part. Un morceau de ruban isolant sur la paroi est une protection facultative.

Après fermeture, vérifier que le SSD n'a pas souffert :

```bash
lsblk
sudo dmesg | grep -i -E "nvme|pcie|aer" | grep -i -E "error|fail|timeout"
```

`lsblk` doit afficher le SSD `nvme0n1` avec ses deux partitions montées : `nvme0n1p1` sur `/boot/firmware` (le démarrage) et `nvme0n1p2` sur `/` (le système et le stockage).

```text
nvme0n1     259:0    0 931.5G  0 disk
├─nvme0n1p1 259:1    0   512M  0 part /boot/firmware
└─nvme0n1p2 259:2    0   931G  0 part /
```

La seconde commande ne doit rien afficher : sinon, le câble plat souffre et il faut rouvrir.

## Installation du système

Principe : Raspberry Pi Imager écrit Ubuntu sur une clé USB, le Pi démarre dessus, on écrit une image fraîche sur le SSD, on y copie la configuration, puis on démarre sur le SSD.

### 1. Préparer la clé USB (sur le PC)

Il faut une clé USB de 16 Go minimum. Elle ne sert qu'au premier démarrage.

Dans Raspberry Pi Imager, choisir `Raspberry Pi 5` › `Other general-purpose OS` › `Ubuntu` › `Ubuntu Server 26.04.x LTS (64-bit)`, puis la clé USB comme stockage.

Quand Imager propose de personnaliser le système, remplir les champs suivants :

- **Nom d'hôte** : le nom du Pi sur le réseau (par exemple `pi5`)
- **Nom d'utilisateur** et **mot de passe**
- **Wi-Fi** : nom du réseau, mot de passe et pays `FR`
- **SSH** : l'activer (authentification par mot de passe)

Ajouter ensuite à la fin de `config.txt`, sur la partition `system-boot` de la clé :

```ini
[all]
dtparam=nvme
```

Le port PCIe est désactivé par défaut. Sans cette ligne, le SSD n'apparaît pas.

**Sous Windows, si la clé est détectée mais que `system-boot` n'apparaît pas dans l'explorateur de fichiers**, ouvrir PowerShell en mode administrateur :

```powershell
Get-Disk
```

Repérer la clé grâce à son nom (colonne `FriendlyName`, par exemple `Generic Flash Disk`), sa taille et son style de partition (`MBR`). Ne jamais choisir le disque du PC. Noter son numéro (colonne `Number`), noté `N` ci-dessous.

```powershell
Get-Partition -DiskNumber N
```

La clé contient deux partitions :

- une petite partition `FAT32` d'environ 512 Mo : c'est `system-boot` (la partition 1 dans mon cas) ;
- une grande partition `Unknown` de plusieurs Go : c'est le système Linux, que Windows ne sait pas lire. Ne pas y toucher.

Donner une lettre de lecteur à la partition `FAT32`, en remplaçant `N` par le numéro du disque et `P` par le numéro de la partition `FAT32` (colonne `PartitionNumber`) :

```powershell
Add-PartitionAccessPath -DiskNumber N -PartitionNumber P -AssignDriveLetter
```

Si Windows propose de formater le disque : appuyer sur **Annuler**.

### 2. Démarrer sur la clé

1. Brancher la clé sur un port USB bleu ou noir (j'ai utilisé un port noir), puis l'alimentation en dernier. Le port bleu est plus rapide si la clé est en USB 3.0.
2. Attendre 5 à 10 minutes (le premier démarrage est long).
3. Se connecter depuis le PC :

```bash
ssh <utilisateur>@<nom_hôte>.home
```

Le suffixe `.home` a fonctionné depuis mon PC Windows, mais pas depuis mon téléphone. Sinon, chercher l'adresse IP du Pi dans l'interface de la box et utiliser `ssh <utilisateur>@<ipv4_du_pi>`.

4. Vérifier que le SSD est visible : `lsblk` doit afficher `nvme0n1`.

### 3. Écrire Ubuntu sur le SSD

Conseil (que je n'ai pas suivi) : lancer ces commandes dans `tmux`, pour qu'elles survivent à une coupure SSH.

```bash
cd /dev/shm
wget https://cdimage.ubuntu.com/ubuntu/releases/26.04/release/ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz
wget https://cdimage.ubuntu.com/ubuntu/releases/26.04/release/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing      # doit afficher OK
xz -t ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz   # ne doit rien afficher

# Vérifier que la cible est bien nvme0n1 et pas sda, qui est la clé USB
xzcat ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz | sudo dd of=/dev/nvme0n1 bs=4M status=progress conv=fsync
```

`/dev/shm` est stocké en mémoire vive : l'image ne transite pas par la clé USB, et ce qui est vérifié est exactement ce qui est écrit. C'est aussi bien plus rapide qu'une clé USB 2.0 comme la mienne.

### 4. Copier la configuration sur le SSD

```bash
sudo mount /dev/nvme0n1p1 /mnt
sudo cp /boot/firmware/user-data /boot/firmware/network-config /mnt/
diff /boot/firmware/current/cmdline.txt /mnt/current/cmdline.txt   # vide = identiques
```

Ouvrir ensuite `config.txt` du SSD avec un éditeur (`nano`, `vi` ou `vim`) :

```bash
sudo vim /mnt/config.txt # vous pouvez utiliser nano, vi ou un autre éditeur de type TUI
```

et ajouter à la fin du fichier :

```ini
[all]
dtparam=nvme
```

Enfin, vérifier l'ordre de démarrage et démonter :

```bash
sudo rpi-eeprom-config | grep BOOT_ORDER
sudo umount /mnt
```

- Ubuntu range `cmdline.txt` dans `current/` à cause de la ligne `os_prefix=current/` de `config.txt`.
- `BOOT_ORDER=0xf461` se lit de droite à gauche : carte SD (`1`), NVMe (`6`), USB (`4`), recommencer (`f`). Le SSD passe donc avant la clé.

### 5. Démarrer sur le SSD

1. `sudo systemctl poweroff`, attendre la fin de l'activité, couper l'alimentation.
2. **Retirer la clé** : ses partitions portent les mêmes noms que celles du SSD (`system-boot`, `writable`).
3. Rallumer. Si le Pi n'apparaît pas sur le réseau au bout d'une dizaine de minutes, redémarrer une fois (voir [Problèmes rencontrés](problemes.md)).
4. Sur le PC, l'avertissement `REMOTE HOST IDENTIFICATION HAS CHANGED` est normal (système neuf) :

```bash
ssh-keygen -R <ip_du_pi>
ssh <utilisateur>@<ip_du_pi>
```

5. Vérifier que le Pi tourne bien sur le SSD :

```bash
lsblk
df -h /
```

- `lsblk` liste les disques, leurs partitions et l'endroit où chacune est montée : `/` doit être sur `nvme0n1p2` et `/boot/firmware` sur `nvme0n1p1`. La clé USB (`sda`) ne doit plus apparaître.
- `df -h /` affiche la taille et l'occupation du système de fichiers principal : environ 900 Go, ce qui confirme que la partition s'est agrandie automatiquement à tout le SSD au premier démarrage.

## Après l'installation

**Mises à jour**

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install iw   # permet au service Netplan de régler le pays Wi-Fi
```

**Shell : zsh et oh-my-zsh**

```bash
sudo apt install -y zsh git curl
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Deux extensions facultatives, que j'apprécie beaucoup : l'auto-suggestion propose les commandes de l'historique pendant la frappe, et la coloration syntaxique affiche en rouge une commande mal tapée.

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

Dans `~/.zshrc` :

```bash
plugins=(git sudo zsh-autosuggestions zsh-syntax-highlighting)
```

`zsh-syntax-highlighting` doit être chargé **en dernier**, comme l'indiquent ses auteurs : il colore la ligne de commande en s'accrochant aux fonctions de saisie (les « widgets ») définies par les autres plugins. Chargé avant eux, il ne prendrait pas en compte leurs fonctions.

**Accès de secours depuis un téléphone Android (Termux)**

Je n'ai pas d'autre PC : si le mien tombe en panne, mon téléphone reste un moyen d'accéder au Pi.

```bash
pkg install openssh
ssh-keygen -t ed25519          # avec une passphrase
ssh-copy-id <utilisateur>@<ip_du_pi>
```

Dans mon cas, l'adresse IPv4 du Pi n'a jamais changé. Si elle venait à changer, il suffirait de réserver une adresse fixe au Pi (bail DHCP statique) dans l'interface de la box.

## Références

- [Wiki Geekworm — X1001](https://wiki.geekworm.com/X1001)
- [Wiki Geekworm — P579](https://wiki.geekworm.com/P579)
- [Wiki Geekworm — H505](https://wiki.geekworm.com/H505)
- [Vidéo d'installation du X1001](https://youtu.be/X1GxqLKpPfg)
- [Documentation Raspberry Pi — M.2 HAT+](https://www.raspberrypi.com/documentation/accessories/m2-hat-plus.html)
- [Ubuntu pour Raspberry Pi](https://ubuntu.com/download/raspberry-pi)
- [Images Ubuntu 26.04](https://cdimage.ubuntu.com/ubuntu/releases/26.04/release/)
