# Ginger - HackMyVM (Hard) - Writeup

![Ginger Icon](Ginger.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Ginger" (Schwierigkeitsgrad: Hard).

## Metadaten

*   **Maschine:** Ginger (HackMyVM - Hard)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Ginger](https://hackmyvm.eu/machines/machine.php?vm=Ginger)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** [Datum nicht im Original-Writeup spezifiziert]
*   **Original Writeup:** [https://alientec1908.github.io/Ginger_HackMyVM_Hard/](https://alientec1908.github.io/Ginger_HackMyVM_Hard/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mittels `nmap` identifizierte einen SSH-Dienst (Port 22) und einen Apache-Webserver (Port 80). Die Web-Enumeration (`gobuster`, `wpscan`) offenbarte eine WordPress-Installation unter `/wordpress/` mit einem anfälligen Plugin (`cp-multi-view-calendar`). Eine SQL-Injection-Schwachstelle in diesem Plugin wurde mit `sqlmap` ausgenutzt, um die `wp_users`-Tabelle zu dumpen und den Passwort-Hash des Benutzers `webmaster` zu extrahieren.

Der Hash wurde mittels `hashcat` und `rockyou.txt` geknackt (Passwort: `sanitarium`). Mit diesen Zugangsdaten wurde der WordPress-Adminbereich betreten. Über den Theme-Editor wurde die `404.php`-Datei modifiziert, um eine PHP-Webshell (`system($_GET['cmd']);`) einzuschleusen. Diese wurde genutzt, um eine Reverse Shell als Benutzer `www-data` zu erhalten.

Als `www-data` wurde das Passwort für den Benutzer `sabrina` (`dontforgetyourpasswordbitch`) in den Kernel-Meldungen (`dmesg`) gefunden, nachdem ein Hinweis in `/home/sabrina/password.txt` entdeckt wurde. Dies ermöglichte einen SSH-Login als `sabrina`.

Als `sabrina` wurde festgestellt, dass via `sudo` ein Python-Skript (`/opt/app.py`) als Benutzer `webmaster` ausgeführt werden konnte. Dieses Skript startete eine Flask-Anwendung im Debug-Modus. Über eine SSH-Portweiterleitung wurde auf die Flask-App zugegriffen und eine Server-Side Template Injection (SSTI)-Schwachstelle ausgenutzt, um eine Reverse Shell als `webmaster` zu erlangen.

Als `webmaster` wurde ein Reverse-Shell-Skript (`/tmp/backup.sh`) erstellt und in `/home/caroline/backup/` kopiert, wobei eine bestehende Datei überschrieben wurde (dies impliziert Schreibrechte über Gruppenzugehörigkeit). Nach Ausführung dieses Skripts durch `caroline` (vermutlich Cronjob/Login-Skript) wurde eine Shell als `caroline` erhalten und die User-Flag gelesen.

Für die finale Eskalation zu `root` wurde als `caroline` der Befehl `sudo /srv/code &` ausgeführt. Es wurde festgestellt, dass dieser Befehl `/etc/passwd` für kurze Zeit schreibbar macht. In diesem Zeitfenster wurde mittels `echo` ein neuer Benutzer `hacker` mit UID 0 und einem bekannten Passwort (`ben`) zu `/etc/passwd` hinzugefügt. Ein `su hacker`-Befehl mit dem Passwort `ben` gewährte Root-Zugriff, woraufhin die Root-Flag gelesen wurde.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `nmap`
*   `grep`
*   `gobuster`
*   `wpscan`
*   `searchsploit`
*   `sqlmap`
*   `vi` / Editor
*   `hashcat`
*   WordPress (Theme Editor)
*   `php` (`system()`)
*   `bash`
*   `nc` (netcat)
*   `sudo`
*   `ls`, `cd`
*   `cat`
*   `strings`
*   `dmesg`
*   `ssh`
*   `python` / `python3` (`pty`, `os` modules)
*   Flask (Debugger/SSTI)
*   `openssl` (`passwd`)
*   `echo`
*   `rm`
*   `cp`
*   `mkdir`
*   `export`
*   `su`

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`), Portscan (`nmap`) -> Port 22 (SSH), Port 80 (HTTP/Apache).
2.  **Web Enumeration:** `gobuster` -> `/wordpress/`. `wpscan` -> Plugin `cp-multi-view-calendar` identifiziert.
3.  **SQL Injection:** `searchsploit` (implied) / `sqlmap` -> SQLi in `cp-multi-view-calendar` bestätigt. Dump der `wp_users` Tabelle -> Hash für `webmaster`.
4.  **Password Crack:** `hashcat -m 400 ... rockyou.txt` -> Passwort `sanitarium` für `webmaster`.
5.  **WordPress Exploit:** Login als `webmaster:sanitarium` in `/wp-admin`. Modifizieren von `404.php` (Theme Editor) -> PHP Webshell einfügen (`system($_GET['cmd']);`).
6.  **Initial Access (RCE):** `nc`-Listener starten. Webshell via URL (`.../404.php?cmd=...bash_reverse_shell...`) triggern -> Shell als `www-data`. Stabilisieren.
7.  **Lateral Movement (`www-data` -> `sabrina`):** `/home/sabrina/password.txt` (Hinweis) und `dmesg | grep pass` (Passwort) -> Passwort `dontforgetyourpasswordbitch` für `sabrina` gefunden. Login via `ssh sabrina@<IP>`.
8.  **Lateral Movement (`sabrina` -> `webmaster`):** `sudo -l` -> `(webmaster) NOPASSWD: /usr/bin/python /opt/app.py`. Flask-App im Debug-Modus starten. SSH Port Forward (`ssh -L 5000:localhost:5000...`). Flask-App via `127.0.0.1:5000` erreichen. SSTI in `?name=` ausnutzen -> Reverse Shell als `webmaster`.
9.  **Lateral Movement (`webmaster` -> `caroline`):** Reverse-Shell-Skript in `/tmp/backup.sh` erstellen. `/home/caroline/backup/backup.sh` damit überschreiben (via Gruppenberechtigung). Warten auf Ausführung -> Reverse Shell als `caroline`. (Optional: SSH-Key für Persistenz).
10. **User Flag:** `cat /home/caroline/user.txt`.
11. **Privilege Escalation (`caroline` -> `root`):** `sudo -l` (implied) -> `/srv/code`. `sudo /srv/code &` starten. In kurzem Zeitfenster `echo 'hacker:...' >> /etc/passwd` ausführen, um Root-User `hacker` (Passwort `ben`) hinzuzufügen. Mit `su hacker` und Passwort `ben` wechseln -> Root-Shell.
12. **Root Flag:** `cat /root/root.txt`.

## Flags

*   **User Flag (`/home/caroline/user.txt`):** `f65aaadaeeb04adaccba45d7babf5f8c`
*   **Root Flag (`/root/root.txt`):** `ae426c9d237d676044e5cd8e8af9ef7f`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---
