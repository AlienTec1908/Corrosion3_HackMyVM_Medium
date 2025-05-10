# Corrosion3 - HackMyVM (Medium)

![Corrosion3 Icon](Corrosion3.png)

## Übersicht

*   **VM:** Corrosion3
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Corrosion3)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 25. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Corrosion3_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die virtuelle Maschine "Corrosion3" von HackMyVM (Schwierigkeitsgrad: Medium) wurde durch eine Kette von Schwachstellen kompromittiert. Der initiale Zugriff erfolgte nach der Entdeckung von Klartext-Credentials in öffentlich zugänglichen Web-Logdateien. Diese ermöglichten zusammen mit einer aufgedeckten Port-Knocking-Sequenz (ermittelt via LFI) den SSH-Login als Benutzer `randy`. Die Privilegienerweiterung zum Benutzer `bob` gelang durch das Überschreiben eines weltweit schreibbaren Python-Skripts, das von einem Cronjob als `bob` ausgeführt wurde. Der finale Root-Zugriff wurde durch die Ausnutzung einer unsicheren `sudo`-Regel erreicht, die es `bob` erlaubte, `runc` als Root auszuführen, was einen Container-Escape ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   `gobuster`
*   `wfuzz`
*   `curl`
*   `knock` (für Port Knocking)
*   `ssh`
*   `python` / `python3` (für Reverse Shell und Shell-Stabilisierung)
*   `nc` (netcat)
*   `stty`
*   `runc` (als Exploit-Vektor)
*   Standard Linux-Befehle (`ls`, `sudo`, `find`, `getcap`, `ss`, `cd`, `cat`, `nano`, `id`, `uname`, `dmesg`, `pwd`, `mkdir`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Corrosion3" erfolgte in diesen Schritten:

1.  **Reconnaissance & Web Enumeration:**
    *   Ziel-IP (`192.168.2.116`, Hostname `corrosion.hmv`) via `arp-scan` identifiziert.
    *   `nmap` zeigte Port 22 (SSH - gefiltert) und Port 80 (HTTP - Apache 2.4.41).
    *   `nikto` und `gobuster` fanden das Verzeichnis `/website/` mit Unterverzeichnissen wie `/website/logs/` und der Datei `/website/sales_detail.php`.
    *   Zugriff auf `/website/logs/` offenbarte Logdateien mit Klartext-Credentials: `randy:RaNDY$SuPer!Secr3etPa$$word`.
    *   Die Datei `/website/sales_detail.php` wurde als anfällig für Local File Inclusion (LFI) über den Parameter `shared` identifiziert (`sales_detail.php?shared=/etc/passwd`).

2.  **Initial Access (randy via SSH & Port Knocking):**
    *   Mittels der LFI-Schwachstelle wurde die Datei `/etc/knockd.conf` ausgelesen. Diese enthielt die Port-Knocking-Sequenz zum Öffnen von SSH (Port 22): SYN-Pakete auf 1110, 2220, 3330.
    *   `knock 192.168.2.116 1110 2220 3330` öffnete den SSH-Port.
    *   Erfolgreicher SSH-Login als Benutzer `randy` mit dem zuvor gefundenen Passwort `RaNDY$SuPer!Secr3etPa$$word`.

3.  **Privilege Escalation (randy -> bob via Cronjob):**
    *   Enumeration als `randy` zeigte keine direkten `sudo`-Rechte oder ungewöhnliche SUID-Dateien.
    *   Das Verzeichnis `/opt/` enthielt ein Python-Skript `simpleurlencode.py`, das `root` gehörte, aber **weltweit schreibbar** war.
    *   Es wurde angenommen (und später bestätigt), dass ein Cronjob dieses Skript als Benutzer `bob` ausführt.
    *   Das Skript `/opt/simpleurlencode.py` wurde mit einer Python-Reverse-Shell-Payload überschrieben, die sich zum Angreifer-Port 1234 verband.
    *   Nach kurzer Zeit (Cronjob-Intervall) wurde eine Reverse Shell als Benutzer `bob` empfangen.
    *   Die User-Flag (für `bob`) wurde aus `/home/bob/user.txt` gelesen.

4.  **Privilege Escalation (bob -> root via runc):**
    *   `sudo -l` für `bob` zeigte: `(root) NOPASSWD: /usr/sbin/runc`.
    *   Die Standard-Technik zum `runc`-Container-Escape wurde angewendet:
        1.  Ein Container-Verzeichnis (`~/container01/rootfs`) wurde erstellt.
        2.  `runc spec` generierte eine `config.json`.
        3.  Die `config.json` wurde modifiziert, um das Root-Verzeichnis (`/`) des Hosts in das Root-Verzeichnis des Containers zu mounten.
        4.  `sudo /usr/sbin/runc run mycont` startete den Container.
    *   Innerhalb des Containers wurde eine Shell mit Root-Rechten auf dem Host-System erlangt (`euid=0`).
    *   Die Root-Flag wurde aus `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Local File Inclusion (LFI):** Zum Auslesen der Port-Knocking-Konfiguration.
*   **Informationslecks in Logdateien:** Preisgabe von Klartext-Credentials.
*   **Port Knocking:** Als Schutzmechanismus für SSH, der umgangen wurde.
*   **Unsichere Dateiberechtigungen:** Ein weltweit schreibbares Skript im Besitz von `root`, das von einem Cronjob ausgeführt wurde.
*   **Cronjob Exploitation:** Ausnutzung eines Cronjobs, der ein manipulierbares Skript ausführt.
*   **Unsichere `sudo`-Konfiguration (`runc`):** Erlaubte einem Benutzer die Ausführung von `runc` als Root, was zu einem Container-Escape und Root-Zugriff führte.

## Flags

*   **User Flag (`/home/bob/user.txt`):** `d3a6cef5b73fa1fb233ed6a0e3b9de01`
*   **Root Flag (`/root/root.txt`):** `18e8141ab1333a87c35e1fad5b394d66`

## Tags

`HackMyVM`, `Corrosion3`, `Medium`, `LFI`, `Log Leak`, `Port Knocking`, `SSH`, `Cronjob Exploitation`, `SUID`, `runc`, `Container Escape`, `Privilege Escalation`, `Linux`
