# Home Cyberlab – Projektstatus

**Stand:** 12.10.2026  
**Ziel:** Aufbau eines isolierten Cybersecurity-Labors für Angriffssimulationen, Angriffserkennung und Incident Response.

## 1. Laborumgebung

| System | IP-Adresse | Aufgabe |
|---|---|---|
| Kali Linux | 192.168.100.50 | Angriffssimulation |
| Windows Server / DC01 | 192.168.100.20 | Zielsystem, Active Directory |
| Security Onion 3.1.0 | 192.168.100.10 | Netzwerküberwachung und SIEM |
| Ubuntu / Wazuh | 192.168.100.30 | Wazuh-Manager |

Virtualisierung: Oracle VirtualBox.

## 2. Erfolgreich nachgewiesen

- Kali erreicht Windows Server über das Labornetzwerk.
- Nmap erkennt auf Windows Server die Ports 53, 88, 135, 139, 389 und 445.
- Nmap erkennt mehrere unterstützte SMB-Versionen.
- Metasploit-Modul `auxiliary/scanner/smb/smb_version` wurde erfolgreich ausgeführt.
- SMB 3.1.1 wurde als bevorzugter Dialekt erkannt.
- Security Onion erfasst den Datenverkehr zwischen Kali und Windows Server.
- Zeek-Verbindungsereignisse sind in Security Onion Hunt sichtbar.
- Suricata erzeugt Alerts zum Scanverkehr.
- Windows-Ereignisse einschließlich Sysmon und PowerShell wurden bereits in Security Onion nachgewiesen.

## 3. Angriffssimulationen – 12.10.2026

**Test 1: Nmap-Port- und Diensterkennung**

Ergebnis: Erfolgreich. Windows Server wurde erkannt; mehrere Active-Directory- und SMB-Dienste sind erreichbar.

**Test 2: SMB-Protokollerkennung mit Nmap**

Ergebnis: Erfolgreich. SMB-Dialekte wurden aufgelistet.

**Test 3: SMB-Versionserkennung mit Metasploit**

Ergebnis: Erfolgreich. SMB 3.1.1 wurde erkannt.

**Test 4: Überprüfung in Security Onion**

Ergebnis: Erfolgreich. Zeek-Ereignisse und Suricata-Alerts für Kali → Windows Server, Port 445, wurden gefunden.

**Bewertung:** Der grundlegende Angriff-Erkennung-Workflow funktioniert.

## 4. Offene Punkte

- Wazuh: Agent-Verbindungen und Erreichbarkeit erneut prüfen.
- Wazuh Registry-FIM (`syscheck`): Problem noch nicht geklärt.
- Rootcheck: Scans funktionieren grundsätzlich; ein positiver Rootkit-Erkennungstest fehlt.
- Schwachstellenerkennung: noch nicht abschließend geprüft.
- YARA-Integration: noch nicht abschließend geprüft.
- Weiterleitung von Wazuh-Alerts an Security Onion: noch nicht nachgewiesen.

## 5. Arbeitsregeln

1. Bereits erfolgreich getestete Funktionen nicht ohne Anlass erneut konfigurieren.
2. Pro Schritt nur eine Änderung durchführen.
3. Vor Änderungen den bisherigen Zustand dokumentieren.
4. Fehler gezielt beheben, ohne funktionierende Komponenten unnötig zu verändern.
5. Jeden Angriffstest mit Datum, Befehl, Ergebnis und Erkennung dokumentieren.

## 6. Nächster Schritt

Weitere Angriffssimulationen im isolierten Labor durchführen und die Erkennung in Security Onion kontrollieren. Wazuh-Probleme nur dann gezielt bearbeiten, wenn sie für den jeweiligen Test relevant sind.
