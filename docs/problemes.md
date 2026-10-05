# Problèmes rencontrés et solutions

[← Retour au README](../README.md) · [Guide d'installation](installation.md)

Les difficultés rencontrées pendant l'installation, et ce qui les a résolues.

**Le Pi n'apparaît pas sur le réseau.**
Ce problème m'est arrivé deux fois, dans deux situations différentes :

- **Sur la clé USB, près de la box** : je l'ai débranché et rebranché plusieurs fois, sans succès. Dès que je l'ai éloigné, il est apparu dans l'interface de la box, connecté au 2,4 GHz. Une fois cette première connexion faite, il a réussi à se connecter au 5 GHz près de la box. La cause reste inexpliquée : les journaux de ces démarrages n'ont pas été conservés (`journalctl --list-boots` ne les listait pas), et ceux des démarrages réussis ne montraient aucune erreur.
- **Sur le SSD, au premier démarrage** : le Pi n'apparaissait pas après 20 minutes. Un simple redémarrage a suffi.

Solution : redémarrer une fois. Si ça ne suffit pas, éloigner le Pi de la box pour la première connexion.

**Débit très faible (environ 36 Ko/s).**
Loin de la box, le Pi s'accrochait au 2,4 GHz avec un signal faible (-75 dBm). Plus près, il passe en 5 GHz. Pour voir la bande utilisée : `sudo wpa_cli -i wlan0 status | grep freq` (environ 2400 = 2,4 GHz, environ 5000 = 5 GHz).

**Erreur d'écriture de Raspberry Pi Imager à 100 %.**
La clé était dans un état incohérent. Solution, dans PowerShell en administrateur : `Clear-Disk -Number N -RemoveData`, où `N` est le numéro de la clé obtenu avec `Get-Disk` (colonne `Number`, à repérer grâce au nom, à la taille et au style `MBR`, comme dans le [guide d'installation](installation.md)). Windows demande une confirmation avant d'effacer : vérifier deux fois le numéro avant d'accepter. Relancer ensuite Imager en administrateur.

**`xzcat: Compressed data is corrupt` alors que `sha256sum` affichait OK.**
J'avais lancé `xzcat` en oubliant la suite de la commande (`| sudo dd ...`) : il a déversé plusieurs gigaoctets de données binaires dans le terminal, qui s'est figé, et j'ai débranché le Pi. Le fichier tout juste téléchargé n'était probablement pas encore entièrement écrit sur la clé (`sha256sum` avait vérifié la copie encore en mémoire), d'où la corruption. Solution : télécharger dans `/dev/shm`, faire `Ctrl+C` si une commande inonde le terminal, et toujours éteindre proprement (ou lancer `sync`) avant de couper le courant.

**`curl` ne renvoie rien sur `cdimage.ubuntu.com/releases/...`.**
L'adresse redirige vers `cdimage.ubuntu.com/ubuntu/releases/...` et `curl` ne suit pas les redirections par défaut. Solution : `curl -L`, ou utiliser l'adresse directe.

**`apt` bloqué : `Could not get lock ... unattended-upgr`.**
Les mises à jour automatiques du premier démarrage tournent. il faut attendre, ou lancer un `sudo systemctl stop unattended-upgrades`. Ne jamais supprimer le fichier de verrou.

**`mount: (hint) your fstab has been modified`.**
Le Pi n'a pas de pile pour garder l'heure : au démarrage, son horloge est en retard, et `/etc/fstab` semble plus récent que le chargement de systemd. Sans gravité, `sudo systemctl daemon-reload` fait disparaître le message.

**Pads thermiques.**
Un film de protection oublié m'a obligé à démonter le refroidisseur, dont les clips sont très difficiles à retirer, et un pad s'est décalé. Tout vérifier avant de clipser. Heureusement pour moi la température au repos (environ 40 °C) reste correcte.

**Le ventilateur ne tourne pas.**
Normal sous 50 °C environ. Pour connaître la température, lancer :

```bash
cat /sys/class/thermal/thermal_zone0/temp   # en millièmes de degré : 40000 = 40 °C
```

**Les LED après l'arrêt.**
Lors de mes premiers arrêts, les LED restaient allumées. Aujourd'hui, à l'arrêt, la LED bleue (ACT) du X1001 s'éteint et la LED STAT du Pi passe du vert au rouge : le Pi est éteint et l'alimentation peut être coupée.

