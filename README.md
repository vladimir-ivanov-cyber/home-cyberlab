

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

### Test 5 – SMB-Freigabenprüfung

**Datum:** 12.10.2026

**Ziel:** Windows Server `192.168.100.20`

**Werkzeuge:** Metasploit, Nmap, Security Onion

**Befehle:**
```bash
sudo nmap -p 445 --script smb-enum-shares 192.168.100.20
```

**Ergebnis:**
- Metasploit meldete `STATUS_ACCESS_DENIED`.
- Nmap konnte keine SMB-Freigaben auflisten.
- Security Onion zeichnete die Verbindungen mit Zeek auf.
- Ein untersuchtes Ereignis zeigte `RSTR` (Verbindung vom Server zurückgesetzt).

**Bewertung:** Netzwerküberwachung erfolgreich; keine erfolgreiche Freigabenauflistung nachgewiesen.

### Test 6 – SMB-Signing-Prüfung

**Datum:** 12.10.2026

**Ziel:** Windows Server `192.168.100.20`

**Befehl:**
```bash
sudo nmap -p 445 --script smb2-security-mode 192.168.100.20
```

**Ergebnis:**
- SMB 3.1.1 erkannt.

- ## Credential Harvesting – LLMNR/NBT-NS Poisoning mit Responder

**Datum:** 12.10.2026  
**Status:** Erfolgreich abgeschlossen

### Ziel

Simulation eines Credential-Harvesting-Angriffs in einer isolierten Laborumgebung und Überprüfung, ob Security Onion die zugehörige Netzwerkaktivität erkennt.

### Laborumgebung

| System | IP-Adresse | Rolle |
|---|---|---|
| Kali Linux | `192.168.100.50` | Angreifer / Responder |
| Windows Server (DC01) | `192.168.100.20` | Zielsystem |
| Security Onion | `192.168.100.10` | Netzwerküberwachung / SIEM |

### Durchführung

1. Testkonto `CYBERLAB\labtest` auf dem Windows Server vorbereitet.
2. Responder auf Kali Linux im aktiven Poisoning-Modus gestartet.
3. LLMNR-, NBT-NS- und mDNS-Anfragen des Windows Servers durch Responder beantwortet.
4. Vom Windows Server eine SMB-Verbindung zum Kali-System mit dem Testkonto ausgelöst:

   ```powershell
   net use \\192.168.100.50\labshare /user:CYBERLAB\labtest *
   ```

5. Die Verbindung wurde mit `System error 5 – Access is denied` abgelehnt. Trotzdem wurde eine NTLM-Authentifizierung übertragen.
6. Responder zeichnete die NTLMv2-SSP-Authentifizierungsantwort des Testkontos auf.
7. Die Ereignisse wurden in Security Onion Hunt untersucht.
8. Responder nach Abschluss der Simulation gestoppt.

### Ergebnisse

**Responder:**
- Poisoning-Antworten für LLMNR, NBT-NS und mDNS nachgewiesen.
- NTLMv2-SSP-Authentifizierungsantwort für `CYBERLAB\labtest` erfasst.
- Kein Klartextpasswort ermittelt.

**Security Onion:**
- `zeek.ntlm` – NTLM-Authentifizierungsverkehr protokolliert.
- `zeek.conn` – Netzwerkverbindung protokolliert.
- Zwei `suricata.alert`-Ereignisse nachgewiesen:
  - `ET INFO NTLM Session Setup Request - Negotiate`
  - `ET INFO NTLM Session Setup Request - Auth`
- Verbindung: `192.168.100.20` → `192.168.100.50`, TCP-Port `445`.
- Zeitpunkt: 12.10.2026, ca. 21:13 Uhr.
- Suricata-Einstufung: **Informational**.

### Fazit

Die Credential-Harvesting-Simulation war erfolgreich. Responder konnte eine NTLMv2-Authentifizierungsantwort des Testkontos erfassen.

Security Onion protokollierte den NTLM-Verkehr mit Zeek und erzeugte zwei Suricata-Signaturalarme. Diese Alarme weisen NTLM-Kommunikation nach, identifizieren aber für sich allein keinen Credential-Harvesting-Angriff. Der Zusammenhang wurde durch die Responder-Ausgabe bestätigt.

**Erkenntnis:** Auch bei einer abgelehnten SMB-Verbindung können NTLM-Authentifizierungsdaten übertragen werden. Netzwerküberwachung kann diese Kommunikation sichtbar machen.
- `Message signing enabled and required`.
- Security Onion erfasste zwei neue `zeek.conn`-Ereignisse um 18:47:45 Uhr.

**Bewertung:** SMB-Signing ist verpflichtend aktiviert. Der Testverkehr wurde erfolgreich überwacht.

