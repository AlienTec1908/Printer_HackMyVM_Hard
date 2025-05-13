# Printer - HackMyVM (Hard)

![Printer.png](Printer.png)

## Übersicht

*   **VM:** Printer
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Printer)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 9. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Printer_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Printer" zu erlangen. Der initiale Zugriff erfolgte durch Ausnutzung einer unsicher konfigurierten NFS-Freigabe (`/home/lisa`), die für alle (`*`) zugänglich war. Durch Erstellen eines lokalen Benutzers mit der passenden UID/GID von `lisa` auf dem Angreifer-System konnte auf die gemountete NFS-Freigabe geschrieben und ein eigener SSH Public Key in `~/.ssh/authorized_keys` platziert werden, was einen SSH-Login als `lisa` ermöglichte. Die finale Rechteausweitung zu Root gelang durch Ausnutzung eines unsicheren Mechanismus: Ein als Root laufender Prozess reagierte auf `logger`-Nachrichten ("fatal error !"), indem er eine Datei, auf die ein Symlink im `/opt`-Verzeichnis (`log_file` oder `root_log`, zeigend auf `/root/.ssh/id_rsa`) verwies, in ein ZIP-Archiv (`/opt/logs/journal.zip`) packte. Dieses Archiv konnte dann heruntergeladen und der private SSH-Schlüssel des Root-Benutzers extrahiert werden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster` (versucht)
*   `curl` (versucht)
*   `showmount`
*   `mkdir`
*   `mount` / `umount`
*   `useradd`
*   `groupmod`
*   `su`
*   `echo`
*   `ssh`
*   `ln`
*   `logger`
*   Python3 (`http.server`)
*   `wget`
*   `unzip`
*   Standard Linux-Befehle (`vi`, `cat`, `id`, `ls`, `chmod`, `rm`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Printer" gliederte sich in folgende Phasen:

1.  **Reconnaissance & NFS Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.125) mit `arp-scan` identifiziert. Hostname `printer.hmv` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 8.4p1), 111 (rpcbind), 2049 (NFS) und diverse hohe RPC-Ports (mountd, nlockmgr). Kein Webserver auf Standardports.
    *   Web-Enumerationsversuche auf Port 80 (mit `gobuster`, `curl`) schlugen fehl, da kein Webserver lief.
    *   `showmount -e 192.168.2.125` zeigte die NFS-Freigabe `/home/lisa *` (für alle zugänglich).

2.  **Initial Access (SSH als `lisa` via NFS Exploit):**
    *   Auf dem Angreifer-System wurde ein Verzeichnis (`/mount`) für die NFS-Freigabe erstellt.
    *   Die NFS-Freigabe wurde mit `mount -t nfs 192.168.2.125:/home/lisa /mount` gemountet.
    *   Durch (implizites) `ls -la /mount` wurde die UID/GID des Benutzers `lisa` auf dem Zielsystem zu 1098 ermittelt.
    *   Auf dem Angreifer-System wurde ein Benutzer `lisa` mit UID 1098 und GID 1098 erstellt (`useradd --uid 1098 lisa; groupmod --gid 1098 lisa`).
    *   Als lokaler Benutzer `lisa` wurde mit `su lisa` gewechselt.
    *   Die User-Flag (`f590b7e83e4c8cd11d06849f9c1a8f6d`) konnte aus `/mount/user.txt` gelesen werden.
    *   Der eigene SSH Public Key wurde in `/mount/.ssh/authorized_keys` geschrieben (`echo "ssh-rsa AAA..." > .ssh/authorized_keys`), wodurch Schreibzugriff auf die NFS-Freigabe als `lisa` bestätigt wurde.
    *   Erfolgreicher SSH-Login als `lisa@printer.hmv` mit dem entsprechenden privaten SSH-Schlüssel.

3.  **Privilege Escalation (von `lisa` zu `root` via `logger` & Symlink Exploit):**
    *   Als `lisa` wurde festgestellt, dass das Verzeichnis `/opt` für alle beschreibbar war.
    *   In `/opt` wurden symbolische Links (`root_log` und `log_file`) erstellt, die auf `/root/.ssh/id_rsa` zeigten (`ln -s /root/.ssh/id_rsa log_file`).
    *   Durch Ausführen von `logger "fatal error !"` wurde ein Mechanismus auf dem Zielsystem ausgelöst (vermutlich ein als Root laufender Prozess, der Logs überwacht).
    *   Dieser Mechanismus erstellte daraufhin eine ZIP-Datei (`journal.zip`) im Verzeichnis `/opt/logs/`, die den Inhalt der Datei enthielt, auf die der Symlink zeigte (also `/root/.ssh/id_rsa`).
    *   Ein Python-HTTP-Server wurde im Verzeichnis `/opt/logs/` gestartet, um `journal.zip` bereitzustellen.
    *   Von der Angreifer-Maschine wurde `journal.zip` heruntergeladen.
    *   `unzip journal.zip` entpackte den privaten SSH-Schlüssel des Root-Benutzers (obwohl `unzip` nach einem Passwort fragte, schien der Inhalt dennoch zugänglich oder das Passwort wurde implizit gefunden/war nicht relevant).
    *   Der extrahierte Root-SSH-Schlüssel wurde in `root_ssh` gespeichert, die Berechtigungen auf `600` gesetzt.
    *   Erfolgreicher SSH-Login als `root@printer.hmv` mit dem Schlüssel `root_ssh`.
    *   Die Root-Flag (`ba72c777ca3351ac5a837e0cd8efa0ed`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Unsichere NFS-Freigabe:** Eine NFS-Freigabe (`/home/lisa`) war für alle (`*`) zugänglich und ermöglichte Schreibzugriff, nachdem ein lokaler Benutzer mit passender UID/GID erstellt wurde. Dies erlaubte das Platzieren eines SSH Public Keys.
*   **Informationsleck / Unsicherer Prozess durch `logger`:** Ein als Root laufender Prozess reagierte auf spezifische Log-Nachrichten (`logger "fatal error !"`) und archivierte unsicher Dateien aus einem benutzerkontrollierbaren Pfad (`/opt`, wo Symlinks platziert werden konnten).
*   **Symlink-Missbrauch:** Durch Erstellen eines Symlinks im `/opt`-Verzeichnis, der auf den privaten SSH-Schlüssel von Root zeigte, konnte der `logger`-getriggerte Prozess dazu gebracht werden, diesen sensiblen Schlüssel zu archivieren.
*   **Globale Schreibrechte:** Das Verzeichnis `/opt` war für alle beschreibbar, was das Platzieren des Symlinks ermöglichte.
*   **Exponierter Root-SSH-Schlüssel:** Der private SSH-Schlüssel des Root-Benutzers war nicht passwortgeschützt und konnte nach Extraktion direkt verwendet werden.

## Flags

*   **User Flag (`/home/lisa/user.txt`):** `f590b7e83e4c8cd11d06849f9c1a8f6d`
*   **Root Flag (`/root/root.txt`):** `ba72c777ca3351ac5a837e0cd8efa0ed`

## Tags

`HackMyVM`, `Printer`, `Hard`, `NFS Exploit`, `Symlink Abuse`, `logger Exploit`, `Information Leak`, `SSH Key Leak`, `Linux`, `Privilege Escalation`, `OpenSSH`, `ProFTPD`
