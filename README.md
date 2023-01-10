# ESXi-Twaeks-und-Befehle
Kleine Sammlung von Kommandos für jeden Tag, die man immer wieder sucht

# ESXi Version anzeigen
`vmware -vl`

`uname -a`

## Achtung bei Installation von 7.x (VMFSL, Virtueller Flash)
Bei der Installation von ESXi 7.x aufwärts, werden z.B. auf einer 250GB SSD **120GB** für VMFSL Partition reserviert, dabei werden nur nur ca. ***4GB*** davon aktiv genutzt !!!. Um dies zu vermeiden gibt es zwei Möglichkeiten:

Man muss man **während** der Installation **SHFT + O** drücken und folgenden Parameter hinzufügen:
a.) den nicht 'supporteten' **autoPartitionOSDataSize**

`autoPartitionOSDataSize=8192`

b.) der offizieler Paraneter lautet:

`systemMediaSize=min`
womit aber **24GB** reserviert werden.


# BIOS Ausgabe z.B. wieviele RAM Module sind installiert
`smbiosDump|grep Location, Manufacturer,Part Number, Size, Max. Speed`

## Warnung auf der Oberfläche deaktivieren
`ESXi Shell for the Host has been enabled`

`vim-cmd hostsvc/advopt/update UserVars.SuppressShellWarning long 1`

## Aktuelle Aufgabe hängt (Restart Management)
Manchmal passiert, dass Aufgaben hängen bleiben und auch ein Neustart der VM nicht hilft. Dann kann man damit versuchen:
```
/etc/init.d/hostd restart
/etc/init.d/vpxa restart
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

## Speicher Sharing TPS aktivieren
Es ist heute aus Sicherheitsgründen deaktiviert. Es ist auch nur dann nötig, wenn RAM Mangelware ist. So kann es Überprüft werden, wie der Wert gesetzt ist
```
esxcli system settings advanced list -o /Mem/ShareForceSalting
esxcli system settings advanced list -o /Mem/AllocGuestLargePage
```

Einschalten / Aktivieren für alle Maschinen. Damit es wirkt, müssen die VMs neu gestartet werden und es braucht ein wenig Zeit damit es die Wirkung entfaltet.
```
esxcli system settings advanced set -o /Mem/ShareForceSalting -i 0
esxcli system settings advanced set -o /Mem/AllocGuestLargePage -i 0
```
Man kann dies auch via WebOberfläche ändern.
<img width="1524" alt="sharesalting" src="https://user-images.githubusercontent.com/35377000/159718627-9fc7f4b2-f8e3-4149-b7be-340220a05b96.png">

# Updates
##  Online Update auf die neuste Version
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

### Update mit Offline Dopot
+ Falls der Host keinen Zugang in die Außenwelt hat
+ Das ZIP muss bei VMWARE herutergeladen werden (nicht entpacken)
+ https://customerconnect.vmware.com/patch#search

```
esxcli software sources profile list -d /vmfs/volumes/ssd/VMware-ESXi-8.0-20513097-depot.zip

Name                          Vendor        Acceptance Level  Creation Time        Modification Time
----------------------------  ------------  ----------------  -------------------  -----------------
ESXi-8.0.0-20513097-standard  VMware, Inc.  PartnerSupported  2022-09-23T18:59:28  2022-09-23T18:59:28
ESXi-8.0.0-20513097-no-tools  VMware, Inc.  PartnerSupported  2022-09-23T18:59:28  2022-09-23T18:59:28

esxcli software profile update -p ESXi-8.0.0-20513097-standard -d /vmfs/volumes/ssd/VMware-ESXi-8.0-20513097-depot.zip
```

## Update Probleme
Wenn das Problem "**no space left**" kommt...
```
[InstallationError]
 [Errno 28] No space left on device
       vibs = VMware_locker_tools-light_11.3.5.18557794-18812553
 Please refer to the log file for more details.
```

Wobekommt man die neusten Tools her?


`esxcli software sources vib list --depot=https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml | grep tools-light | grep Update | sort`
Dann muss man die Tools manuel installieren.
```
cd /tmp
wget http://hostupdate.vmware.com/software/VUM/PRODUCTION/main/esx/vmw/vib20/tools-light/VMware_locker_tools-light_11.2.5.17337674-17700514.vib
esxcli software vib install -f -v /tmp/VMware_locker_tools-light_11.3.5.18557794-18812553.vib
```

- oder jemand, der das besser erklärt https://blog.andreas-schreiner.de/2017/11/06/vmware-esxi-online-offline-update/


## DependencyError
Mögliche Probleme beim Update
```
esxcli software profile update -p ESXi-7.0U3c-19193900-standard -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-i
ndex.xml --dry-run

 [DependencyError]
 VIB Realtek_bootbank_net55-r8168_8.045a-napi requires vmkapi_2_2_0_0, but the requirement cannot be satisfied within the ImageProfile.
 VIB Realtek_bootbank_net55-r8168_8.045a-napi requires com.vmware.driverAPI-9.2.2.0, but the requirement cannot be satisfied within the ImageProfile.
 Please refer to the log file for more details.
```
Lösung


## Tools Updaten
VMWare Tool im allgemeinen bekommt man hier https://www.vmware.com/go/tools

`esxcli software vib install -d /vmfs/volumes/nvme/111temp/VMware-Tools-12.0.0-core-offline-depot-ESXi-all-19345655.zip`


## Fehler beim Upgrade einer Dell Custom ISO Version ###
Missing depedency vibs error | Upgrading vSphere 6.7 to 7.0 using the Dell custom ISO
![esxi-upgrade-error](https://user-images.githubusercontent.com/35377000/147673666-c8b5bdd2-d6f6-4071-a35e-3d1839fde18b.png)

Lösung:
Es müssen Die Dell-Treiber vor dem Upgrade entfernt werden
```
esxcli software vib list | grep qed
esxcli software vib remove -n qedf
```

# Storage / VMDK
## Sicherung
Das [OVF Tool](https://developer.vmware.com/web/tool/4.4.0/ovf) kann man bei VMWare herunterladen. Es ist auchVMWare Workstation enthalten `c:\Program Files (x86)\VMware\VMware Workstation\OVFTool\`
Mit den OVF Tool kann man eine **Vmeware Workstation VM** auf einen ESX bringen `ovftool.exe -dm=thin d:\VMWARE\Debian\Debian.vmx  vi://192.168.16.200`
Keine richtige Sicherung aber ein Möglichkeit Maschinen abzuziehen. Es besteht aus zwei Schritten.
* Die orginale VMX Datei sichern. (_namm kann einfach WinSCP nehmen_)

`scp -r root@192.168.16.200:/vmfs/volumes/nvme/*/*.vmx a:\ESX200\`
* Vorlagedatei exportieren, d.h. OVF oder OVA (_alles in einer Datei_)

`ovftool.exe -tt=ova vi://root@192.168.16.200/Win10-01 r:\ESX2000\`

## Wiederherstellung
Sie erfolgt in umgekehrter Reihenefolge, mit einen kleinen Zusatzschritt - alte VMX unterjubeln
* OVA von der Festplatte auf den ESX host importieren
* Maschine **deregistrieren** (_Registrierung aufheben_)
* alte VMX-Datei auf den Host kopieren
* Maschine wieder registrieren

* Die Einfachste Variante kann so aussehen. (_man könnet sogar auf **-dm=thin** verzichten_)

`ovftool -dm=thin d:\OVT\Win10-01\ vi://root@192.168.16.200`
* Hier die erwiterte Möglichkeit. Die Parameter sind eigentlich selbsterklärend.

`ovftool -ds=ssd -dm=thin -n=Win10-05 --maxVirtualHardwareVersion=15 d:\OVT\ESX200\Win10.ova vi://root@192.168.16.200`
Möchte man z.B. eine VMWARE Workstation Maschine auf den ESX übertragen, so muss das Verzeichniss angeben
`ovftool -ds=ssd -dm=thin -n=Win10-05 --maxVirtualHardwareVersion=15 d:\OVT\Win11\ vi://root@192.168.16.200`

## Automatische Sicherung
Sicherung der Maschine vom Rechner aus, inclusiver Heruterfahren und starten.
```
plink.exe -ssh root@%esx% -pw %pass% vim-cmd vmsvc/power.shutdown 84
timeout /t 20
%ovf% -tt=ova vi://root:%pass%@%esx%/61 s:\ESXi2\
timeout /t 3
plink.exe -ssh root@%esx% -pw %pass% vim-cmd vmsvc/power.on 84
```


## Dateien auf oder von den ESXi Host kopieren
SCP ist sehr schnell, ca. 90MB/s Download- sowie ca. 60MB/s Uploadgeschwindigkeit.

`scp -r r:\_ESX-alt_\ESXi8\8\ root@192.168.16.200:/vmfs/volumes/ssd/`

Einfach eine Maschine am Stück von Host holen.  
**Achtung:** bei Thin-provisinierten Platten wird die volle größe expnadiert.

`scp -r root@192.168.16.200:/vmfs/volumes/ssd/8/ r:\_ESX-alt_\ESXi7\8\`

Man kann mit Wildcards arbeiten *.vmdk
`scp -r root@192.168.16.82:/vmfs/volumes/nvme/*/*.vmx c:\temp\`

## Hat eine VM Snapshots
`ls /vmfs/volumes/*/*/*Snapshot*.*`

## vswp-Datei nicht erstellen
`sched.swap.vmxSwapEnabled=FALSE`

## Scoreboard Dateien nicht erstellen ##
`vmx.scoreboard.enabled = "FALSE"`

## VMDK Thick to Thin Konvertierung 
`vmkfstools -i /vmfs/volumes/vmfs/Debian/Debian.vmdk -d thin /vmfs/volumes/vmfs/Debian/Debian-thin.vmdk`

## Compact VMDK
Verkleinern der Datei z.B. nach Windows update und anschlessenden Bereinigung mit Datenträgerverwaltung.
Es wird die noramle VMDK Datei, nicht die -flat gewählt.

`vmkfstools -K /vmfs/volumes/vmfs/Debian/Debian.vmdk`

## real size VMDK
```
ls -shl /vmfs/volumes/vmfs/Debian/Debian-flat.vmdk
du -h /vmfs/volumes/vmfs/Debian/Debian-flat.vmdk
```

### Festplatte mit 'fremder' Signatur mounten, z.B. aus einen anderen ESX
`esxcfg-volume -l`

m - temporär, M - dauerhaft

`esxcfg-volume -m 5d967053-3b238502-382b-c81f66d03187`

## SMART Werte einer Festplatte auslesen ###
```
esxcli storage nmp device list
esxcli storage core device smart get -d naa.5e83a975f4156eb1
```
```
esxcli storage core device list
esxcli storage core device smart get -d t10.ATA_____WDC_WD2502ABYS2D18B7A0________________________WD2DWCAT1H751520
```

## Hardware Version 16.2.x
Mit der version von VMWare Workstation 16.2 kommt auch ein neuer Hardwarelever `virtualHW.version = "19"` es ermöglicht einen virtuellen TPM 2.0 (für Win11) zu aktivieren ohne die Maschine zu verschlüsseln. Man aktiviert es in der VMX Datei mit `managedvm.autoAddVTPM = "software"`.

Achtung es ist experimentel:
+ Snapshots vorher entfernen
+ Snapshots nur im ausgeschalteten zustan erstellen

## macOS VMWare Workstation Unlocker 3.0.3
Funktioniert auch mit VMWare Workstation 16.2.1 build-18811642
https://github.com/BDisp/unlocker

## macOS ESX Unlocker für ESX 7.x
Eine Möglichkeit macOS bis hin zu Moterey laufen zu lassen
+ [https://github.com/erickdimalanta/esxi-unlocker](https://github.com/erickdimalanta/esxi-unlocker) - es ist zwar nur Version 3.0.2, die funktioniert bei mir auch mit der aktuellen Version **ESXi-7.0U3f-20036589** (2022)
+ [https://github.com/netgc/esxi-unlocker](https://github.com/netgc/esxi-unlocker) - dieses Repository ist viel aktueller, fuktionierte bei mir aber nicht

Probleme / Lösungen
File unlocker.tgz does not exist
`tar zcf unlocker.tgz etc`

`./esxi-smctest.sh: Operation not permitted`
SecureBoot im BIOS deaktivieren - ja, genau :-) !!!

## ESXi Einstellungen sichern
Wenn man z.B. die Festplatte tauschen muss.
Wichtig dabei ist dass der Restore nur auf gleicher Version funktioniert.

`/bin/firmwareConfig.py --backup /tmp/`
`/bin/firmwareConfig.py --restore /tmp/configBundle.tgz`

## Installation auf nicht unterstützer Hardware / Whitebox
Wenn bei der Installation z.B. so was kommt **no network adapters are physically connected to the system**, dann gibt es zwei Möglichkeiten
+ nach einen Custom-ISO vom Hersteller suchen
(HP, Dell, Lenovo bieten so waas an). Da gibts die Trieber für die vorher nicht erkannte Hardware.
+ ESX-Customizer-PS von **v-front**

Das hat aber auch seine Grenzen **z.B. Realtek R8168 funktioniert ab ESX 7.0 nicht mehr!**. 

## ESX-Customizer-PS | USB-LAN Adapter
Auf der Webseite ist es sehr gut erklärt https://www.v-front.de/p/esxi-customizer-ps.html
Hier trotzdem ein paar Hinweise um USB-LAN Adapter von Anker für einen HP ProDesk 400 G6 in Betrieb zu nehmen

Community Netzwork Driver herunterladen
https://flings.vmware.com/usb-network-native-driver-for-esxi nach z.B. c:\1

`.\ESXi-Customizer-PS.ps1 -v80 -nsc -sip -pkgDir c:\1`

```
Logging to a:\TempUSER\ESXi-Customizer-PS-10752.log ...
Running with PowerShell version 5.1 and VMware PowerCLI version .. build

Connecting the VMware ESXi Software depot ... [OK]
Getting Imageprofiles, please wait ... [OK]
Select Base Imageprofile:
-------------------------------------------
1 : ESXi-8.0a-20842819-standard
2 : ESXi-8.0a-20842819-no-tools
3 : ESXi-8.0.0-20513097-standard
4 : ESXi-8.0.0-20513097-no-tools
-------------------------------------------
```
