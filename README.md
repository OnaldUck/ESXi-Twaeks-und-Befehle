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

Hat bei mir bis jetzt immer funktioniert. Beim Upgrade von 6.7 auf 7.0 wurde anschliessend die NVMe nicht gefunden.  
Beim "richtigen Upgrade" mit **software profile update** war anschliessend alles in Ordnung.

###  Offline Update 
+ Falls der Host keinen Zugang in die Außenwelt hat
+ Das ZIP muss bei VMWARE herutergeladen werden (nicht entpacken)

```
esxcli software sources profile list -d=/vmfs/volumes/ssd/VMWare-ESXi-7.0U2a-17867351-depot.zip
esxcli software sources profile update -p VMWare-ESXi-7.0U2a-17867351-standard -d=/vmfs/volumes/ssd/VMWare-ESXi-7.0U2a-17867351-depot.zip
```

### Update Probleme
Wenn das Problem "**no space left**" kommt...
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
### Fehler beim Upgrade einer Dell Custom ISO Version ###
Missing depedency vibs error | Upgrading vSphere 6.7 to 7.0 using the Dell custom ISO
![esxi-upgrade-error](https://user-images.githubusercontent.com/35377000/147673666-c8b5bdd2-d6f6-4071-a35e-3d1839fde18b.png)

Lösung:
Es müssen Die Dell-Treiber vor dem Upgrade entfernt werden
```
esxcli software vib list | grep qed
esxcli software vib remove -n qedf
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
SCP ist sehr schnell, ca. 90MB/s Download- sowie ca. 60MB/s Uploadgeschwindigkeit.

`scp -r r:\_ESX-alt_\ESXi8\8\ root@192.168.16.200:/vmfs/volumes/ssd/`

Einfach eine Maschine am Stück von Host holen.  
**Achtung:** bei Thin-provisinierten Platten wird die volle größe expnadiert.

`scp -r root@192.168.16.200:/vmfs/volumes/ssd/8/ r:\_ESX-alt_\ESXi7\8\`

Man kann mit Wildcards arbeiten *.vmdk
`scp -r root@192.168.16.82:/vmfs/volumes/nvme/*/*.vmx c:\temp\`

### Hat eine VM Snapshots
`ls /vmfs/volumes/*/*/*Snapshot*.*`

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

### Einstellungen sichern
Wenn man z.B. die Festplatte tauschen muss.
Wichtig dabei ist dass der Restore nur auf gleicher Version funktioniert.

`/bin/firmwareConfig.py --backup /tmp/`
`/bin/firmwareConfig.py --backup /tmp/`

## Installation auf nicht unterstützer Hardware / Whitebox
Wenn bei der Installation z.B. so was kommt **no network adapters are physically connected to the system**, dann gibt es zwei Möglichkeiten
+ nach einen Custom-ISO vom Hersteller suchen
(HP, Dell, Lenovo bieten so waas an). Da gibts die Trieber für die vorher nicht erkannte Hardware.
+ ESX-Cuszomizer-PS von **v-front**


### ESX-Cuszomizer-PS
Auf der Webseite ist es sehr gut erklärt https://www.v-front.de/p/esxi-customizer-ps.html
Hier trotzdem ein paar Hinweise um Netztwerkkartntreiber für einen HP ProDesk 400 G6 

Community Netzwork Driver herunterladen
https://flings.vmware.com/community-networking-driver-for-esxi

`.\ESXi-Customizer-PS.ps1 -v70 -nsc -sip -pkgDir d:\Downloads\ESXi\vib\`

```
Logging to a:\TempUSER\ESXi-Customizer-PS-10752.log ...
Running with PowerShell version 5.1 and VMware PowerCLI version .. build

Connecting the VMware ESXi Software depot ... [OK]
Getting Imageprofiles, please wait ... [OK]
Select Base Imageprofile:
-------------------------------------------
1 : ESXi-7.0U3c-19193900-standard
2 : ESXi-7.0U3c-19193900-no-tools
3 : ESXi-7.0U2e-19290878-standard
4 : ESXi-7.0U2e-19290878-no-tools
5 : ESXi-7.0U2d-18538813-standard
6 : ESXi-7.0U2d-18538813-no-tools
7 : ESXi-7.0U2sc-18295176-standard
8 : ESXi-7.0U2sc-18295176-no-tools
```

autoPartitionOSDataSize=8192
systemMediaSize=min
