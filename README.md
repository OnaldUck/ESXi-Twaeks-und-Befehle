# ESXi-Twaeks-und-Befehle
Kleine Sammlung von Kommandos für jeden Tag, die man immer wieder sucht

###  Online Update auf die neuste Version
+ Für gewöhnlich muss der Host neu gestartet werden.
+ Falls was schief geht, kann beim Start in der Konsole die alte Version mit **Shift + R** wiederhergstellt werden.
+ Um die Installation durchzuführen müssen Sie das `--dry-run` entfernen

```
esxcli network firewall ruleset set -e true -r httpClient
esxcli software sources profile list -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml | grep 'ESXi-7.0' | sort
esxcli software profile update -p ESXi-7.0U2d-18538813-standard -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml
esxcli network firewall ruleset set -e false -r httpClient
```
Dieser Befehl funktionierte jahrelang, ist aber falsch: **`software vib update`**.

`esxcli software vib update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml --dry-run`
Hat bei mir bis jetzt immer funktioniert. Beim Upgrade von 6.7 auf 7.0 wurde anschliessend die NVMe nicht gefunden. Bein "richtigen" Upgrad mit **software profile update** war anschliessend alles in Ordnung.



### Update Probleme
Wenn das Problem "no space left" kommt...
```
[InstallationError]
 [Errno 28] No space left on device
       vibs = VMware_locker_tools-light_11.3.5.18557794-18812553
 Please refer to the log file for more details.
```
Dann muss man die Tools manuel installieren.
```
cd /tmp
wget http://hostupdate.vmware.com/software/VUM/PRODUCTION/main/esx/vmw/vib20/tools-light/VMware_locker_tools-light_11.2.5.17337674-17700514.vib
esxcli software vib install -f -v /tmp/VMware_locker_tools-light_11.3.5.18557794-18812553.vib
```

## Copy & Paste aktivieren (Isolation)
Öffnen Sie die `/etc/vmware/config` Datei mit einem Texteditor.
Fügen Sie folgende Zeilen hinzu und speichern anschließend wieder die Datei.
Reboot des Hosts notwendig.
```
vmx.fullpath = "/bin/vmx"
isolation.tools.copy.disable="FALSE"
isolation.tools.paste.disable="FALSE"
```

## Storage / VMDK
### Dateien auf oder von den ESXi Host kopieren
`scp -r r:\_ESX-alt_\ESXi8\81\ root@192.168.16.200:/vmfs/volumes/ssd/`

Einfach eine Maschine am Stück von Host holen.
**Achtung:** bei Thin-provisinierten Platten wird die volle größe expnadiert.
`scp -r root@192.168.6.82:/vmfs/volumes/ssd/082/ r:\_ESX-alt_\ESXi7\105\`
Man kann mit Wildcard arbeiten *.vmdk

### Hatt eine VM Snapshots
`ls /vmfs/volumes/*/*/*.vmx`

### vswp-Datei nicht erstellen
`sched.swap.vmxSwapEnabled=FALSE`

### VMDK Thick to Thin Konvertierung 
`vmkfstools -i /vmfs/volumes/vmfs/Debian/Debian.vmdk -d thin /vmfs/volumes/vmfs/Debian/Debian-thin.vmdk`

### Compact VMDK
Verkleinern der Datei z.B. nach Windows update und anschlessenden Bereinigung mit Datenträgerverwaltung.
Es wird die noramle VMDK Datei, nicht die -flat gewählt.

`vmkfstools -K /vmfs/volumes/vmfs/Debian/Debian.vmdk`

### real size VMDK
```
ls -shl /vmfs/volumes/vmfs/Debian/Debian-flat.vmdk
du -h /vmfs/volumes/vmfs/Debian/Debian-flat.vmdk
```

### Festplatte mit 'fremder' Signatur mounten, z.B. aus einen anderen ESX
`esxcfg-volume -l`

m - temporär, M - dauerhaft

`esxcfg-volume -m 5d967053-3b238502-382b-c81f66d03187`

### SMART Werte einer Festplatte auslesen ###
```
esxcli storage nmp device list
esxcli storage core device smart get -d naa.5e83a975f4156eb1
```
```
esxcli storage core device list
esxcli storage core device smart get -d t10.ATA_____WDC_WD2502ABYS2D18B7A0________________________WD2DWCAT1H751520
```

### Hardware Version 16.2.x
Mit der version von VMWare Workstation 16.2 kommt auch ein neuer Hardwarelever `virtualHW.version = "19"` es ermöglicht einen virtuellen TPM 2.0 (für Win11) zu aktivieren ohne die Maschine zu verschlüsseln. Man aktiviert es in der VMX Datei mit `managedvm.autoAddVTPM = "software"`.

Achtung es ist experimentel:
+ Snapshots vorher entfernen
+ Snapshots nur im ausgeschalteten zustan erstellen

### Unlocker 3.0.3
Funktioniert auch mit VMWare Workstation 16.2.1 build-18811642
https://github.com/BDisp/unlocker

### ESX Unlocker für ESX 7.0U2

https://github.com/erickdimalanta/esxi-unlocker

File unlocker.tgz does not exist

`tar zcf unlocker.tgz etc`

