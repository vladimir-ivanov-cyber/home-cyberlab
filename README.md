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

[Dokumentation mit acht Screenshot-Belegen öffnen](uebung-1/README.md)

Responder erfasste eine **NTLMv2-SSP-Authentifizierungsantwort für `CYBERLAB\Administrator`**. Security Onion zeigt Zeek-Verkehr von `192.168.100.20` zu `192.168.100.50`. Wazuh zeigt **Event 4624**, **Regel 92657**, **Level 6**, **NTLM V2** und **LogonType 3**.

**Der Hash beziehungsweise die NTLMv2-Response ist nur in Responder und nicht in Wazuh sichtbar. Die Wazuh-Quell-IP `192.168.100.20` erlaubt keine eindeutige Zuordnung zum Responder-Angriff.** Die zeitlichen Unterschiede und der IPv6-Client im Capture werden in der verlinkten Untersuchung erläutert.

### 2. Exploitation und Detection

[Dokumentation mit Screenshot-Belegen öffnen](uebung-2/README.md)

Im isolierten Cyberlab wurde auf dem Windows Server **DC01 (192.168.100.20)** ein bewusst verwundbarer HTTP-Testdienst auf **TCP-Port 8080** bereitgestellt. Die Anwendung enthielt eine absichtlich fehlende Authentifizierungsprüfung am Endpunkt `/admin/proof`.

**Nmap** bestätigte den erreichbaren Dienst. Anschließend demonstrierte ein selbst erstelltes **Metasploit-Auxiliary-Modul** die unautorisierte Ausführung einer festgelegten Testaktion. Die Ausgabe `Unauthorized action confirmed` bestätigte die erfolgreiche Interaktion mit der verwundbaren Anwendung.

**Security Onion und Kibana** zeigten die Netzwerkkommunikation zwischen Kali Linux (`192.168.100.50`) und DC01. Zeek protokollierte `GET /admin/proof` mit dem HTTP-Status `200 OK`. Suricata erzeugte für einen separaten curl-Aufruf den Alarm **ET HUNTING curl User-Agent to Dotted Quad (SID 2034567)**.

**Wazuh** lieferte die Endpoint-Perspektive. Ein Sysmon-Event 11 löste Wazuh-Regel **92213 (Level 15)** für eine temporäre PowerShell-Datei aus. Ein direkter Zusammenhang dieses Alarms mit dem erfolgreichen Metasploit-Test wurde nicht nachgewiesen.

Die Schwachstelle wurde bewusst für diese Laborübung erstellt. Es handelt sich weder um eine bekannte Windows-CVE noch um eine Remote-Code-Execution-Schwachstelle.

### 3. Lateral Movement

<!-- Nach Durchführung ergänzen: Datum, Ausgangssystem und Ziel, Ablauf, Authentifizierungs- und Prozessereignisse, Zeitlinie, Bewertung. -->

### 4. Malware Detection

<!-- Nach Durchführung ergänzen: Datum, Testdatei, Erkennungsmechanismus, Alarm, Belege, Reaktion und Bewertung. -->

### 5. Baseline und eigene Erkennungsregeln

<!-- Nach Durchführung ergänzen: Zeitraum normaler Aktivität, Vergleich mit Testaktivität, Regel, Prüfung auf Fehlalarme, Ergebnisse und Belege. -->

### 6. Incident Response

<!-- Nach Durchführung ergänzen: Auslöser, Zeitlinie, Eindämmung, Untersuchung, Bereinigung, Wiederherstellung und Nachprüfung. -->

## Fazit

<!-- Ergänzen: wichtigste Ergebnisse, Grenzen der Erkennung, gewonnene Erkenntnisse und daraus abgeleitete Verbesserungen. -->
