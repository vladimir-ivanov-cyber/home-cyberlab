# Abgleich mit dem Tutoring

**Stand: 16.10.2026 · Referenz: bereitgestelltes Tutoring.docx**

Das Tutoring beschreibt einen Lernpfad aus Laboraufbau, sechs praktischen Übungen und nachvollziehbarer Untersuchung. Es ist eine Zielvorgabe und kein Nachweis, dass einzelne Funktionen im Home Cyberlab bereits umgesetzt wurden. Dieser Abgleich bewertet die zugängliche Dokumentation; neue Labortests wurden nicht durchgeführt.

## Aufbau und Telemetrie

| Anforderung des Tutorings | Dokumentierter Stand | Nächster Nachweis |
| --- | --- | --- |
| Virtualisierung und getrenntes Labornetz `192.168.100.0/24` | VirtualBox und passende Hostadressen dokumentiert | Adaptermodus, Netzmaske, Routing und tatsächliche Trennung vom Heimnetz festhalten |
| Kali `.50`, Windows `.20`, Security Onion `.10` | Rollen und Adressen dokumentiert; aktuelle Übung-1-Bilder vorhanden | Keine Neuinstallation erforderlich, solange kein konkreter Fehler belegt ist |
| Separater Wazuh-Manager als mögliche Architektur | Ubuntu/Wazuh `.30` und Windows-Agent `001` dokumentiert | Aktuellen Agentstatus bei Bedarf prüfen |
| Sensor sieht relevanten Laborverkehr | Zeek-Verkehr `.20 → .50` sichtbar | Sensor-Schnittstelle und Sichtbarkeit weiterer Richtungen und Protokolle prüfen |
| Windows Security, System, Application, Sysmon, PowerShell erfassen | Winlogs, Sysmon und PowerShell im bisherigen Stand genannt; 4624 in Wazuh konkret belegt | Je Kanal ein Ereignis, Transportweg und Zielansicht dokumentieren |
| Event-IDs 4624, 4625, 4688, 4672 untersuchen | 4624 konkret belegt; Prozesserstellungszeilen ohne vollständige Detailfelder sichtbar | Die übrigen Event-IDs gezielt prüfen; aus der Kurzbeschreibung allein nicht 4688 ableiten |
| Datei-FIM, Registry-FIM, Rootcheck | Datei-FIM mit Regel 550 historisch dokumentiert; Registry offen; Rootcheck als aktiv genannt | Pro Funktion Testhandlung und zugehörigen Treffer festhalten |
| Vulnerability Detection und YARA | Nicht abschließend geprüft | Konfiguration und gezielten Funktionstest separat dokumentieren |
| Wazuh-Alerts in Security Onion | Weiterleitung nicht nachgewiesen | Ein konkretes Wazuh-Ereignis auf beiden Seiten und seinen Transportweg belegen |
| OpenVAS / Greenbone, Behebung und erneuter Scan | Kein abgeschlossener Durchlauf belegt | Unauthentifizierten und credentialed Scan sinnvoll vergleichen; Behebung und Nachprüfung dokumentieren |

## Übung 1 im Detail

| Tutoring-Ziel | Vorhandener Beleg | Bewertung |
| --- | --- | --- |
| Responder auf Kali erfasst NTLM-Authentifizierungsdaten | Screenshot mit NTLMv2-SSP und `CYBERLAB\Administrator` | Capture belegt; Klartextkennwort oder Crack-Erfolg nicht belegt |
| Windows erzeugt SMB-Anfragen | Responder zeigt SMB-Capture mit IPv6-Client | Exakte auslösende Benutzeraktion und eigenständige Zuordnung des IPv6-Clients zu `.20` fehlen |
| LLMNR und NBT-NS in Security Onion untersuchen | Responder meldet diese Poisoning-Antworten; Hunt zeigt `zeek.conn` und `zeek.dns`, darunter UDP 5353 | Direkter LLMNR-/NBT-NS-Nachweis in Security Onion ist mit diesen Hunt-Ausschnitten nicht erbracht |
| Endpoint-Perspektive in Wazuh auswerten | 4624, Regel 92657, Level 6, NTLM V2, LogonType 3 | Anmeldeereignis belegt; eindeutige Zuordnung zum Capture fehlt |
| Befunde dokumentieren | Acht echte Bilder, Feldwerte, Zeit- und IP-Grenzen in der neuen Fassung | Dokumentation vorhanden; Übung nicht als vollständig korrelierte Angriffserkennung bewerten |

**Status: durchgeführt, Korrelation teilweise offen.** Die Übung muss nicht verworfen werden. Ihre belegten Ergebnisse sind aussagekräftig, solange die offenen Zuordnungen sichtbar bleiben. [Vollständige Übung 1](../uebung-1/README.md).

## Weitere Übungen

| Nr. | Lernziel | Nachweis für einen abgeschlossenen Durchlauf |
| --- | --- | --- |
| 2 | Exploitation und Detection | Konkreter Test gegen einen freigegebenen Labordienst, tatsächliches Ergebnis, Netzwerk- und Endpoint-Ereignisse, Bewertung der Erkennung |
| 3 | Lateral Movement | Ausgangszugang, getrennte Quelle und Ziel, beobachtete Authentifizierung beziehungsweise PowerShell-Aktivität, nachvollziehbare Ereigniskette |
| 4 | Malware Detection | Definierte Testdatei, tatsächlich auslösender Erkennungsmechanismus, Alarm und dokumentierte Reaktion |
| 5 | Baseline und Regeln | Zeitraum normaler Aktivität, Vergleich mit einem Test, eigene Regel, Prüfung gegen Normal- und Testdaten |
| 6 | Incident Response | Zeitlinie, Entscheidung zur Eindämmung, durchgeführte Maßnahmen, Bereinigung und überprüfte Wiederherstellung |

Für die Übungen 2–6 ist im zugänglichen Material kein vollständiger Abschluss belegt. Es werden keine neuen Ergebnisse dafür ergänzt. Die früher zusätzlich von ChatGPT erstellte Nmap-/SMB-Testreihe ist auf Wunsch des Nutzers nicht Teil dieses Projektdokuments.

## Fachliche Präzisierungen des Tutorings

### Wazuh zeigt keine automatisch erfasste NTLMv2-Response

Die Aufforderung, die erfassten Zugangsdaten in Wazuh zu untersuchen, ist als Untersuchung der Endpoint-Ereignisse zu verstehen. Der Hash beziehungsweise die NTLMv2-Response ist in den vorliegenden Belegen nur in Responder sichtbar. 4624 mit NTLM V2 ist keine Hashanzeige. Eine Regelbeschreibung mit einem möglichen Angriff ist ein Ermittlungsansatz, kein Beweis der Technik.

### Netzwerkmodus und Zeitsynchronisation konkret prüfen

Ein passend benanntes Netz oder ein NAT-Netz belegt die gewünschte Isolation nicht allein. Gateway, Routing, weitere Adapter und Monitoring-Schnittstellen gehören in die Aufbau-Dokumentation. Für Korrelation müssen Uhrensynchronisation und Zeitzonen aller beteiligten Ansichten nachvollziehbar sein. Im aktuellen Fall liegen Wazuh- und Zeek-Anzeigezeiten bei angenommener gleicher Zeitzone rund 13 Minuten auseinander.

### Versionsabhängige Installationsangaben nicht ungeprüft übernehmen

Das Tutoring nennt Winlogbeat, Logstash und einen festen Port. Der tatsächlich eingerichtete Erfassungsweg muss zur installierten Security-Onion-Version passen. Für Security Onion 3 beschreibt die offizielle Dokumentation die [Einbindung mit Elastic Agent](https://docs.securityonion.net/en/3/main/elastic-agent/). Daraus folgt kein Anlass, einen funktionierenden vorhandenen Erfassungsweg ohne Prüfung umzubauen.

Die im Tutoring einzeln genannten RAM-Zuteilungen für die vier VMs summieren sich auf **24 GB**, nicht auf mehr als 30 GB. Der Host und zusätzliche Dienste benötigen weitere Ressourcen. Eine zugesagte Vollauslastung aller Komponenten mit 16 GB lässt sich aus den Angaben nicht ableiten.

### YARA und Testdateien brauchen einen definierten Erkennungsweg

Eine Wazuh-Installation allein belegt weder einen allgemeinen RAM-Malware-Scan noch eine automatische Erkennung jeder Testdatei. Die offizielle [FIM-/YARA-Anleitung](https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/fim-yara.html) beschreibt eine gezielt einzurichtende Integration über Dateiänderungen und Active Response. Diese ist kein Nachweis eines eingerichteten Speicherscanners im Home Cyberlab.

Die im Transkript undeutlich bezeichnete Testdatei dürfte EICAR meinen; dies ist eine Interpretation. Für Übung 4 ist vorab festzulegen, ob ein Endpoint-Schutzprodukt, eine Wazuh-Logregel oder eine eingerichtete Integration den Test erkennen soll. Ein FIM-Ereignis allein bestätigt eine Dateiänderung und noch keine Malware-Klassifizierung.

### Schwachstellenscans eindeutig benennen

Das Tutoring verwendet „authenticated“ und „credentialed“ in einem vermeintlichen Gegensatz. Gemeint sein dürfte der Vergleich eines Scans ohne Anmeldedaten mit einem Scan unter Nutzung berechtigter Zugangsdaten. Im Testprotokoll sind deshalb die tatsächlich verwendete Scanart, Berechtigungen, Ergebnisse und Nachprüfung festzuhalten.

## Priorität für die Fortsetzung

1. Dokumentation bereinigen und Übung 1 zentral verlinken.
2. Netz- und Zeitgrundlagen sowie relevante Logquellen dokumentieren.
3. Die Korrelation von Übung 1 bei Bedarf mit einem abgegrenzten Testlauf schließen; besonders LLMNR/NBT-NS in Security Onion, IPv6-Zuordnung und Endpoint-Bezug prüfen.
4. Danach die weiteren Übungen mit jeweils einer untersuchbaren Fragestellung bearbeiten. Zusätzliche Werkzeuge erst für einen konkreten Lernschritt ergänzen.

## Quellen

- `Tutoring.docx`, vom Nutzer bereitgestellt: Abschnitte zu Architektur, Logweiterleitung, Wazuh, OpenVAS und „Exercise number one“ bis „Exercise number six“. Die Datei wurde gelesen und nicht verändert.
- Bereitgestellte GitHub-Screenshots: bisherige Haupt-README, Laboraufbau, offene Punkte und alter Credential-Harvesting-Abschnitt.
- Acht tatsächliche Screenshot-Dateien und ihre Prüfsummen: [Belegverzeichnis](../uebung-1/BELEGE.csv).
- Frühere lokale Projektunterlagen vom 12.10.2026: historischer Stand insbesondere zu Datei-FIM, Rootcheck und Logquellen.
- Offizielle technische Quellen stehen jeweils an den zugehörigen Präzisierungen. Sie ersetzen keine Nachweise der lokalen Konfiguration.
