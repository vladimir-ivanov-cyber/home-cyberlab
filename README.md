# Home Cyberlab

Dieses Projekt bildet eine kleine Unternehmensumgebung in einem virtuellen Labor nach. Die Architektur ist auf ein isoliertes Netzwerk im Hypervisor ausgelegt. Verbindungen zum Heimnetz oder Internet sollen nur gezielt ermöglicht werden.

**Kali Linux** simuliert Angriffe auf den **Windows Server**. Dieser erzeugt Protokolle und Sicherheitsereignisse. **Security Onion** sammelt und analysiert Netzwerkdaten sowie weitergeleitete Logs als SIEM. **Wazuh** ergänzt die Überwachung durch die Erfassung und Auswertung sicherheitsrelevanter Aktivitäten auf dem Windows-System.

Ziel ist, Angriffsfläche, Protokollierung, Erkennung und Untersuchung zusammenzuführen. Security Onion dient als Oberfläche für die Netzwerkanalyse. Die Endpoint-Perspektive wird in Wazuh untersucht; für eine zentrale Auswertung ist die Weiterleitung der Wazuh-Alerts an Security Onion vorgesehen.

## Laborumgebung

Virtualisierung: **Oracle VirtualBox**. Das Netzkonzept verwendet den Adressbereich **192.168.100.0/24**.

| System | IP-Adresse | Aufgabe |
| --- | --- | --- |
| Kali Linux | `192.168.100.50` | Angriffssimulationen und Responder |
| Windows Server / DC01 | `192.168.100.20` | Zielsystem, Active Directory und Wazuh-Agent |
| Security Onion 3.1.0 | `192.168.100.10` | SIEM-Oberfläche und Netzwerkauswertung |
| Ubuntu / Wazuh | `192.168.100.30` | Separater Wazuh-Manager und Dashboard |

Die Windows-Domäne lautet **CYBERLAB**, der Servername **DC01.cyberlab.local**. Die Adresse von Security Onion bezeichnet den Zugang zur Weboberfläche; der überwachte Netzwerkverkehr wird über die Sensoranbindung erfasst.

## Werkzeuge und Analyse

| Werkzeug | Zweck |
| --- | --- |
| Kali Linux | Ausführung kontrollierter Tests gegen freigegebene Laborziele |
| Responder | Untersuchung von Namensauflösung und NTLM-Authentifizierungsantworten |
| Zeek in Security Onion | Analyse von Verbindungen und Protokollereignissen |
| Suricata in Security Onion | Signaturbasierte Auswertung des Netzwerkverkehrs |
| Windows-Ereignisprotokolle | Untersuchung von Anmeldungen, Prozessen und Systemaktivitäten |
| Wazuh | Auswertung von Endpoint-Ereignissen und sicherheitsrelevanten Regeln |

Für jede Simulation werden Testhandlung und beobachtete Ereignisse zusammen untersucht. Dazu gehören Zeitpunkt und Zeitzone, beteiligte Systeme, Konten, Netzwerkadressen, Protokolle sowie die zugehörigen Ereignis- und Regelkennungen. Screenshots belegen die beschriebenen Ergebnisse.

Ein Netzwerkereignis bestätigt zunächst beobachtete Kommunikation. Eine Regelmeldung liefert einen Untersuchungsansatz. Die Zuordnung zu einem konkreten Angriff erfolgt anhand der zusammenpassenden Zeit-, Verbindungs- und Endpoint-Daten. Technische Grenzen werden in der jeweiligen Simulation festgehalten.

## Angriffssimulationen


### 1. Credential Harvesting mit Responder

#### A. Red Team Ausführung (Angriff)
* **Ziel:** Abfangen von NetNTLMv2-Hashes im lokalen Netzwerk mittels LLMNR/NBT-NS Spoofing.
* **Tool:** Responder (ausgeführt auf Kali Linux – `192.168.100.50`).
* **Ablauf:** 
  1. Die lokale Responder-Datenbank (`Responder.db`) und alte Logdateien wurden vollständig bereinigt, um einen Live-Angriff zu simulieren.
  2. Responder wurde im Listening-Modus gestartet, um auf Broadcast-Anfragen im Netzwerk zu warten (`sudo responder -I eth0 -w`).
  3. Auf dem Windows Server (`192.168.100.20`) wurde eine fehlerhafte SMB-Netzwerkfreigabe-Anfrage an einen nicht existierenden Server eingegeben (`\\nov-test-server\portfolio`).
* **Ergebnis:** Responder hat die LLMNR-Anfrage erfolgreich vergiftet (Poisoning), dem Opfer vorgetäuscht, der gesuchte Server zu sein, und den vollständigen NetNTLMv2-Authentifizierungshash für den Benutzer `CYBERLAB\Administrator` im Terminal abgefangen.

![Kali Linux Hash](kali-hash.png)
![Windows Server SMB Error](windows-error.png)


#### B. Blue Team Analyse (Erkennung im SIEM)
* **Plattform:** Security Onion (Zentrales SIEM – `192.168.100.10`) integriert mit Wazuh EDR & Sysmon logs.
* **Netzwerk-Erkennung (Zeek):** In der Security Onion *Hunt*-Konsole wurden die Metadaten-Events von `zeek.dns` identifiziert. Diese dokumentieren den genauen Zeitpunkt, zu dem der Windows Server die Broadcast-Anfrage für `nov-test-server` in das Netzwerk geschickt hat.
* **Endpoint-Erkennung (Sysmon/Wazuh):** Die Analyse des Windows Event Forwarding über das Sysmon-Modul lieferte den entscheidenden Beweis (Smoking Gun) direkt in der SIEM-Umgebung (`winlog.winlog`).

![Security Onion Hunt Overview](so-hunt-overview.png)
![Security Onion Events List](so-hunt-events-list.png)


#### C. Untersuchung der Artefakte (Investigation Findings / Sysmon)
Bei der Detailanalyse des expandierten Sysmon-Logs wurden folgende kritische Indikatoren dokumentiert:

* **`winlog.channel`:** `Microsoft-Windows-Sysmon/Operational` (Bestätigt die Tiefenüberwachung des Endpunkts).
* **`winlog.event_data.Image`:** `C:\WINDOWS\system32\svchost.exe` (Der legitime Windows-Prozess, welcher die DNS-Anfrage ausführte).
* **`winlog.event_data.QueryName`:** `nov-test-server` (Verbindet den Host-Log direkt mit dem原始 Angriff).
* **`winlog.event_data.QueryResults`:** `192.168.100.50` (Das finale Untersuchungsergebnis: Das Feld zeigt explizit, dass die Namensauflösung des Windows-Servers auf die IP-Adresse des Angreifers manipuliert wurde. Dies beweist den Erfolg des LLMNR-Poisoning-Angriffs).

![Sysmon Details](sysmon-detail.png)

#### D. Untersuchung der Artefakte (Investigation Findings / Wazuh EDR)
Die vertiefte Untersuchung der Event-Details direkt in der Wazuh EDR-Konsole lieferte den endgültigen hostbasierten Beweis:

* **`win.system.channel`:** `Microsoft-Windows-NTLM/Operational` (Überwachung aktiver NTLM-Authentifizierungsversuche).
* **`win.eventdata.clientUserName`:** `Administrator` (Der kompromittierte Benutzerkontext).
* **`win.eventdata.targetName`:** `cifs/nesustestvuvasht-server` (Bestätigt den genauen Ziel-String des manipulierten Netzwerkaufrufs).
* **`rule.description`:** `CyberLab: NTLM outbound authentication audit detected (Event 8001)` (Das EDR-System hat erfolgreich registriert, dass der Windows-Host versucht hat, Authentifizierungsdaten unverschlüsselt nach außen zu senden, ausgelöst durch den Responder-Angriff).

![Wazuh EDR Alert](wazuh-edr-alert.png)
![Wazuh Details Top](wazuh-details-top.png)
![Wazuh Details Bottom](wazuh-details-bottom.png)

#### E. Fazit und Gegenmaßnahmen (Conclusion & Mitigation)
Die erfolgreiche Durchführung und Analyse dieser Angriffssimulation liefert wichtige Erkenntnisse über die Risiken von Legacy-Protokollen in Windows-Umgebungen und die Effizienz einer kombinierten SIEM/EDR-Überwachung.

**Gegenmaßnahmen zur Absicherung der Infrastruktur:**
1. **Deaktivierung von LLMNR & NBT-NS:** Die primäre Ursache für diesen Angriff ist das automatische Zurückfallen von Windows auf unsichere Broadcast-Protokolle. Diese müssen zwingend über die Gruppenrichtlinien (GPO) im gesamten Active Directory deaktiviert werden (*"Turn off Multicast Name Resolution"*).
2. **Einführung von SMB Signing:** Um Relay-Angriffe mit den erbeuteten Hashes zu verhindern, sollte SMB Signing für alle Server und Clients im Netzwerk verpflichtend aktiviert werden.
3. **Einsatz von Windows Defender Credential Guard:** Aktivierung von Credential Guard, um das Auslesen und Missbrauchen von Authentifizierungsdaten aus dem LSASS-Prozess durch Virtualisierungsbasierte Sicherheit (VBS) effektiv zu blockieren.

**Analytisches Fazit:**
Durch die nahtlose Integration von **Security Onion** (Netzwerk-Sichtbarkeit über Zeek) und **Wazuh EDR** (Host-Sichtbarkeit über NTLM/Operational Auditing) konnte der Vorfall lückenlos korreliert werden. Während die Netzwerksensoren den Moment des Spoofings dokumentierten, lieferte die EDR-Plattform den präzisen forensischen Nachweis (Event ID 8001), welcher Prozess betroffen war und dass der Host manipuliert wurde. Dies beweist die kritische Notwendigkeit eines ganzheitlichen "Defense-in-Depth"-Ansatzes in modernen Security Operations Centers (SOC).


### 2. Exploitation und Detection

#### A. Red Team Ausführung (Aufklärung, Angriff und Privilegieneskalation)
* **Ziel:** Identifikation offener Dienste auf dem Windows Server, erfolgreiche Ausnutzung einer Schwachstelle zur Etablierung einer Reverse-Shell und anschließende vollständige Übernahme des Systems.
* **Tools:** NMAP & Metasploit Framework (Kali Linux – `192.168.100.50`).
* **Ablauf:**
  1. **Reconnaissance:** Ein aggressiver Portscan mit NMAP wurde durchgeführt, um aktive Dienste zu identifizieren (`sudo nmap -sV -sC -O -p 135,139,445,3389 192.168.100.20 -v`). Der SMB-Dienst (Port 445) wurde als aktiv bestätigt.
  2. **Exploitation:** Da es sich um ein gehärtetes Windows Server 2025 System handelt, wurde ein maßgeschneiderter Payload mit `msfvenom` generiert (`windows/x64/meterpreter/reverse_tcp`) und auf dem Host ausgeführt, um eine interaktive Command-and-Control (C2) Verbindung zu simulieren.
  3. **Privilege Escalation:** Nach dem Öffnen der Meterpreter-Sitzung auf Port `4444` wurde die Post-Exploitation-Technik `getsystem` ausgeführt. Das System wurde erfolgreich über die Technik **Named Pipe Impersonation (MITRE ATT&CK T1134.001)** kompromittiert, wodurch die höchsten Systemrechte (`NT AUTHORITY\SYSTEM`) erlangt wurden.

![NMAP Scan Results](kali-nmap.png)
![Meterpreter Access and Privilege Escalation](kali-meterpreter.png)

#### B. Blue Team Analyse (SIEM-Untersuchung & Patterns)
* **Plattform:** Security Onion (Zentrales SIEM – `192.168.100.10`) & Suricata IDS Engine.
* **Untersuchungsergebnisse (Hunt-Konsole):**
  Gemäß den Anforderungen der Analyse wurden die genauen mрежови мodelle (Network Patterns) im SIEM isoliert. Folgende Antworten wurden forensisch dokumentiert:
  * **What IPs:** Die Verbindung wurde eindeutig zwischen der Angreifer-IP `192.168.100.50` (Kali Linux) und der Opfer-IP `192.168.100.20` (Windows Server) korreliert.
  * **What Ports:** Der Datenstrom etablierte sich über den verdächtigen Port `4444` (Source Port des C2-Handlers) zu einem dynamischen Destination Port (`52890`) auf dem Windows-Host.
  * **What Patterns:** Die IDS-Engine (Suricata) schlug sofort mit kritischen Alarmen an. Es wurden Signaturen detektiert, die spezifisch für Schadsoftware-Verhalten sind:
    1. `ET MALWARE Possible Metasploit Payload Common Construct Bind_API (from server)` (Klassifiziert als **Critical / High Severity**).
    2. `ET HUNTING PE EXE Download over raw TCP` (Erkennung des binären Dateitransfers über das Netzwerk).

![Security Onion Hunt Metasploit Alerts](so-metasploit-overview.png)

#### C. Untersuchung der erweiterten JSON-Artefakte & EDR-Blindspot
Die Tiefenanalyse des Vorfalls in der SIEM-Umgebung lieferte wichtige forensische Erkenntnisse über das Verhalten von Endpoint-Detection-Systemen bei aggressiven Angriffen:

* **Endpoint Perspective (Wazuh EDR):** Während der Privilegieneskalation mittels *Named Pipe Impersonation* über das Metasploit-Framework stürzte der Wazuh-Agent auf dem Windows Server ab (Status: *Disconnected*). Dies simuliert ein realistisches Szenario, bei dem hochentwickelte Schadsoftware oder Kernel-Exploits EDR-Systeme gezielt umgehen oder deren Log-Übertragung blockieren.
* **SIEM-Kompensation (Suricata/Zeek):** Da der Host-Agent stumm war, übernahm die Netzwerkinfrastruktur von Security Onion die lückenlose Erkennung. Folgende tiefe JSON-Felder wurden in den `suricata.alert`-Metadaten analysiert:
  * **`rule.name`:** `ET MALWARE Possible Metasploit Payload Common Construct Bind_API (from server)` (Eindeutige Identifizierung des C2-Frameworks).
  * **`rule.metadata.tag`:** `Metasploit` (Direkte Signierung des Angriffs-Traffics).
  * **`event.severity_label`:** `Critical` (Automatische Einstufung der Bedrohung auf höchster Ebene).

![Suricata Deep JSON Forensic Details](o-json-detail.png)

#### D. Fazit und Gegenmaßnahmen (Conclusion & Mitigation)
1. **Network Egress Filtering:** Die Schadsoftware konnte nur deshalb eine Reverse-Shell aufbauen, weil ausgehender Datenverkehr auf Port `4444` erlaubt war. Firewalls müssen so konfiguriert werden, dass unübliche ausgehende Ports strikt blockiert werden.
2. **Einführung von AppLocker / WDAC:** Unsignierte Binärdateien wie `backdoor.exe` dürfen in einer Enterprise-Infrastruktur nicht ausgeführt werden. Richtlinien zur Anwendungssteuerung hätten den Angriff im Keim erstickt.
3. **Defense-in-Depth Notwendigkeit:** Dieses Szenario beweist, dass man sich niemals nur auf ein EDR-System verlassen darf. Fällt der Host-Schutz aus (Blind Spot), bleibt das Netzwerk-SIEM (Security Onion) die wichtigste Verteidigungslinie, um Command-and-Control-Verkehr (C2) zu erkennen.



### 3. Lateral Movement und C2-Simulation

#### A. Red Team Ausführung (Initial Access, C2-Taktiken und Lateral Movement)
* **Ziel:** Etablierung von Command-and-Control (C2) Aktivitäten mittels verschleierter PowerShell-Befehle und anschließende Durchführung von lateralen Bewegungen (Lateral Movement) im Netzwerk.
* **Tools:** Metasploit Framework, Windows Native Tools (`powershell.exe`, `winrs`).
* **Ablauf:**
  1. **Initial Access & C2-Simulation:** Über die bestehende Meterpreter-Sitzung wurde eine interaktive Windows-Shell geöffnet. Da PowerShell Empire in der isolierten Offline-Laborumgebung aufgrund harter externer Paketabhängigkeiten zu GitHub blockiert wurde, wurden die exakten Verhaltensmuster einer modernen C2-Plattform manuell simuliert.
  2. **Obfuscation (Verschleierung):** Es wurde ein stark verschleierter, Base64-codierter PowerShell-Befehl ausgeführt, welcher im Hintergrund versucht, verdeckt Schadcode auszuführen (`powershell.exe -nop -w hidden -encodedCommand cABvAHc...`). Dies entspricht der echten Arbeitsweise von PowerShell Empire Agents (MITRE ATT&CK T1059.001).
  3. **Lateral Movement (Horizontale Bewegung):** Zur Simulation eines Netzwerksprunges in der lokalen Laborumgebung wurde ein Loopback-Angriff durchgeführt. Mittels des administrativen Windows-Protokolls **WinRM (Windows Remote Management)** wurde eine authentifizierte Remote-Sitzung auf den lokalen Host selbst initiiert (`winrs -r:localhost cmd.exe /c "whoami"`), welche erfolgreich unter dem Kontext `cyberlab\administrator` ausgeführt wurde.

![Windows Server Lateral Movement Execution](kali-empire-simulation.png)

#### B. Blue Team Analyse (SIEM-Zentralisierung & Forensische Funde)
* **Plattform:** Security Onion (Hunt-Konsole) & Windows-Host-Logs (Sysmon).
* **Forensische Erkennung im SIEM:**
  Die Sicherheitsüberwachung hat die bösartige Aktivität erfolgreich erfasst und in der zentralen Log-Datenbank konsolidiert:
  * **C2-Erkennung (Sysmon Event ID 1 - Prozesserschaffung):** In der Hunt-Konsole wurden die Alerts für `powershell.exe` unter dem Benutzerkontext `Administrator` isoliert. 
  * **Deep Artifact Analysis:** Die Tiefenanalyse des expandierten JSON-Logs lieferte den ultimativen forensischen Nachweis. Das Feld `process.command_line` dokumentiert lückenlos den vollständigen verschleierten Aufruf inklusive der verdächtigen Parameter `-nop`, `-w hidden` und `-encodedCommand` zusammen mit dem Base64-Payload-String.

![Security Onion Hunt PowerShell Alerts](so-powershell-overview.png)
![Security Onion Deep JSON Forensic Details](so-powershell-json.png)

#### C. Fazit und Gegenmaßnahmen (Mitigation)
1. **Restriktion von PowerShell via GPO:** PowerShell sollte über die Gruppenrichtlinien streng reglementiert werden. Die Aktivierung des *Constrained Language Mode* verhindert das Ausführen von komplexen, unvollständigen oder verschleierten Base64-Inhalten im Arbeitsspeicher.
2. **Erweiterte Protokollierung (Script Block Logging):** Aktivierung von *PowerShell Script Block Logging (Event ID 4104)*. Dies ermöglicht es dem SOC-Team, den deobfuskieren (entschlüsselten) Code im Klartext zu lesen, selbst wenn der Angreifer Base64-Verschleierung nutzt.
3. **Absicherung von WinRM (Windows Remote Management):** Der WinRM-Dienst sollte auf allen Arbeitsstationen und Servern deaktiviert werden, sofern er nicht explizit für Remote-Management benötigt wird. Falls erforderlich, sollte der Zugriff über Host-Firewalls ausschließlich auf dedizierte Administrations-Systeme beschränkt werden.



### 4. Malware Detection

In dieser Simulation wurde die Erkennung von Schadsoftware auf dem Host `windows-server` (`192.168.100.20`) überprüft. Dazu wurde versucht, die standardisierte EICAR-Testzeichenfolge zu generieren. Dies löste eine sofortige Echtzeitreaktion von Windows Defender aus, wodurch die Datei blockiert und isoliert wurde.

Bei der anschließenden tieferen Analyse stellte sich heraus, dass die Antiviren-Plattform eine hochkritische Bedrohung vom Typ Trojaner (`backdoor.exe`) blockiert hatte, die aus den vorherigen Kompromittierungsphasen auf dem Desktop verblieben war.

![Windows Defender blocking the execution of the malicious file](defender-eicar-block.png)


### A. Baseline-Erweiterung und EDR-Integration
Da Windows Defender seine Sicherheitsereignisse in einem spezialisierten operativen Kanal protokolliert, den der Wazuh-Agent standardmäßig nicht überwacht, wurde die Konfiguration (`ossec.conf`) auf der Zielmaschine manuell erweitert. Dadurch wurde der Kanal `Microsoft-Windows-Windows Defender/Operational` an den zentralen Wazuh-Manager weitergeleitet:

```xml
<localfile>
  <location>Microsoft-Windows-Windows Defender/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### B. SIEM-Analyse (Investigate the Alert)
Nach einem Neustart des Wazuh-Dienstes konnte das SIEM-System die Sicherheitsereignisse erfolgreich verarbeiten. Über das **Discover-Panel** im Wazuh-Dashboard wurde gezielt nach der **Event-ID 1116** (Threat Detected) gefiltert, um den Vorfall präzise zu isolieren und zu untersuchen:

* **Bedrohungsname (Threat Name):** Trojan:Win32/Zusy.NCD!MTB
* **Betroffener Dateipfad (Path):** `file:_C:\Users\Administrator\Desktop\backdoor.exe`
* **Kritikalität (Wazuh Rule Level):** 12 (Severe)
* **Wazuh-Regel-ID (Rule ID):** 62123

![Wazuh Alert for Event ID 1116 showing Microsoft Defender telemetry](wazuh-defender-alert.png)
![Wazuh Alert for Event ID 1116 showing Microsoft Defender telemetry](wazuh-defender-alert2.png)

Die Analyse des Ereignisses bestätigt, dass hostbasierte Erkennungsmechanismen (EDR/SIEM) kritische Logs erfolgreich korrelieren können.

### C. Containment-Reaktion (Practice Containment Response)
Im Einklang mit der Eindämmungsphase des Incident-Response-Prozesses wurde eine vollständige Netzwerisolation des kompromittierten Windows-Servers durchgeführt. Ziel dieser Maßnahme war es, die Verbindung zum Angreifer-System unter Kali Linux (`192.168.100.50`) sofort zu trennen und eine laterale Ausbreitung im internen Netzwerk zu verhindern.

Die Windows-Firewall wurde über ein administratives PowerShell-Skript so konfiguriert, dass jeglicher ein- und ausgehende Datenverkehr blockiert wird. Es wurde jedoch eine Ausnahme für den Wazuh-Manager (`192.168.100.30`) eingerichtet, um den Fluss der Sicherheits-Telemetrie und die administrative Kontrolle aufrechtzuerhalten:

```powershell
# 1. Vollständige Blockierung des ein- und ausgehenden Datenverkehrs
New-NetFirewallRule -DisplayName "SOC_Isolation_In" -Direction Inbound -Action Block -Profile Any
New-NetFirewallRule -DisplayName "SOC_Isolation_Out" -Direction Outbound -Action Block -Profile Any

# 2. Explizite Ausnahme für den Wazuh-Manager zur Aufrechterhaltung des SOC-Monitorings
New-NetFirewallRule -DisplayName "Allow_Wazuh_In" -Direction Inbound -Action Allow -RemoteAddress 192.168.100.30 -Profile Any
New-NetFirewallRule -DisplayName "Allow_Wazuh_Out" -Direction Outbound -Action Allow -RemoteAddress 192.168.100.30 -Profile Any
```

![Inbound Firewall rules creation for host isolation](firewall-isolation-inbound.png)
*Abbildung 4.3: Erfolgreiche Implementierung der eingehenden Firewall-Regeln zur Isolation des Systems.*

![Outbound Firewall rules and Wazuh Manager exceptions](firewall-isolation-outbound.png)
*Abbildung 4.4: Konfiguration der ausgehenden Sperren mit expliziter Freigabe für die SIEM-Kommunikation.*

Die erfolgreiche Anwendung dieser Firewall-Regeln isoliert das System effektiv aus der Angreiferperspektive, während das SOC-Team im SIEM weiterhin Live-Analysen durchführen kann.

## Fazit

Das bisherige SOC-Labor hat die Phasen der Erkennung und ersten Eindämmung (Containment) von Schadsoftware erfolgreich demonstriert. Durch die Kombination von Windows Defender-Ereignissen und dem zentralen Wazuh-Monitoring konnten komplexe Angriffsvektoren in Echtzeit analysiert и neutralisiert werden, bevor eine laterale Ausbreitung stattfinden konnte.


### 5. Baseline und eigene Erkennungsregeln

<!-- Nach Durchführung ergänzen: Zeitraum normaler Aktivität, Vergleich mit Testaktivität, Regel, Prüfung auf Fehlalarme, Ergebnisse und Belege. -->

### 6. Incident Response

<!-- Nach Durchführung ergänzen: Auslöser, Zeitlinie, Eindämmung, Untersuchung, Bereinigung, Wiederherstellung und Nachprüfung. -->

## Fazit

<!-- Ergänzen: wichtigste Ergebnisse, Grenzen der Erkennung, gewonnene Erkenntnisse und daraus abgeleitete Verbesserungen. -->
