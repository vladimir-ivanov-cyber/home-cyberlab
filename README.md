# Home Cyberlab

Dieses Projekt bildet eine kleine Unternehmensumgebung in einem virtuellen Labor nach. Nach dem Architekturkonzept des Tutorings arbeiten die Systeme in einem isolierten Netzwerk im Hypervisor. Verbindungen zum Heimnetz oder Internet sollen nur gezielt ermöglicht werden.

Der vorgesehene Ablauf: **Kali Linux** simuliert Angriffe auf den **Windows Server**. Dieser erzeugt Protokolle und Sicherheitsereignisse. **Security Onion** sammelt und analysiert Netzwerkdaten sowie weitergeleitete Logs als SIEM. **Wazuh** ergänzt die Überwachung durch die Erfassung und Auswertung sicherheitsrelevanter Aktivitäten auf dem Windows-System.

Ziel ist, Angriffsfläche, Protokollierung, Erkennung und Untersuchung zusammenzuführen. Security Onion soll als zentrale Untersuchungsoberfläche dienen; dafür ist auch die Weiterleitung der Wazuh-Alerts vorgesehen. Welche Teile dieses Konzepts bereits nachgewiesen sind und welche noch offen bleiben, ist im folgenden Projektstatus dokumentiert.

## Dokumentation

- [Übung 1: Credential Harvesting mit Responder](uebung-1/README.md) – aktuelle Fassung mit acht Screenshots, einschließlich Wazuh-Auswertung.
- [Abgleich mit dem Tutoring](docs/TUTORING-ABGLEICH.md) – Anforderungen, bisheriger Stand und nächste Schritte.

## Laborumgebung

| System | Adresse | Aufgabe |
| --- | --- | --- |
| Kali Linux | `192.168.100.50` | Testsystem und Responder |
| Windows Server / DC01 | `192.168.100.20` | Windows-Zielsystem, Active Directory, Wazuh-Agent |
| Security Onion 3.1.0 | `192.168.100.10` | Hunt-Oberfläche und Netzwerkauswertung mit Zeek/Suricata |
| Ubuntu / Wazuh | `192.168.100.30` | Separater Wazuh-Manager und Dashboard |

Virtualisierung: **Oracle VirtualBox**. Domäne in den Übung-1-Belegen: `CYBERLAB`, Computername `DC01.cyberlab.local`.

Das Tutoring sieht das Netz `192.168.100.0/24` vor. Die gezeigten Adressen passen dazu; Netzmaske, Adaptermodus, Routing, Sensoranbindung und tatsächliche Isolation sind damit noch nicht vollständig belegt. Die Webadresse von Security Onion beschreibt keine separate Monitoring-Schnittstelle.

## Stand der sechs Tutoring-Übungen

| Übung | Thema | Dokumentierter Stand |
| --- | --- | --- |
| 1 | Credential Harvesting | Responder-Capture, Zeek-Verkehr und Wazuh-Anmeldung belegt; eindeutige Korrelation offen. [Belege und Bewertung](uebung-1/README.md) |
| 2 | Exploitation und Detection | Kein abgeschlossener Durchlauf mit Ausnutzung eines Dienstes und zugehöriger Netzwerk-/Endpoint-Auswertung belegt |
| 3 | Lateral Movement | Kein abgeschlossener Durchlauf mit zugehöriger Untersuchung belegt |
| 4 | Malware Detection | Kein abgeschlossener Test samt Alarm- und Reaktionsnachweis belegt |
| 5 | Baseline und eigene Erkennungsregeln | Keine dokumentierte Normalaktivitäts-Baseline samt geprüfter eigener Regel belegt |
| 6 | Incident Response | Kein vollständiger Durchlauf aus Eindämmung, Untersuchung, Bereinigung und Nachprüfung belegt |

„Nicht belegt“ bezeichnet den Dokumentationsstand und bedeutet nicht automatisch, dass eine Funktion nicht installiert ist oder nicht funktioniert. Die Übungsnummerierung folgt ausschließlich dem Tutoring.

## Aktuelles Ergebnis von Übung 1

Responder zeigt einen **NTLMv2-SSP-Capture für `CYBERLAB\Administrator`** sowie gemeldete mDNS-, NBT-NS- und LLMNR-Poisoning-Antworten. Security Onion zeigt `zeek.conn`- und `zeek.dns`-Einträge von `.20` zu `.50`. Wazuh zeigt **Event 4624**, **Regel 92657**, **Level 6**, **NTLM V2** und **LogonType 3**.

**Der Hash beziehungsweise die NTLMv2-Response ist nur in Responder und nicht in Wazuh sichtbar. Die Wazuh-Quell-IP `192.168.100.20` erlaubt keine eindeutige Zuordnung zum Responder-Angriff.** Außerdem unterscheiden sich die sichtbaren Zeiten; der Responder-Capture nennt einen IPv6-Client. Pass-the-Hash, Relay, Kennwortwiederherstellung oder RDP sind dadurch nicht nachgewiesen.

Die aktuelle [Übung-1-Dokumentation](uebung-1/README.md) ersetzt die bisherige Credential-Harvesting-Beschreibung in der Haupt-README. Die ältere Beschreibung vom 12.10. mit `CYBERLAB\labtest`, einem `net use`-Befehl und älteren NTLM-/Suricata-Angaben gehört zu einem anderen beschriebenen Durchlauf. Ihre Aussagen werden nicht als Belege für den Administrator-Capture vom 15.10. übernommen. Der vorherige Text bleibt in der Git-Historie erhalten, sofern diese Ersatzfassung im bestehenden Repository committed wird.

## Überwachung und offene Funktionen

| Bereich | Bisheriger Stand | Noch zu klären |
| --- | --- | --- |
| Netzwerk | Aktuelle Zeek-Belege zu Übung 1 vorhanden | Sichtbarkeit aller relevanten Protokolle und konkrete Alarmzuordnung |
| Windows-Logs in Security Onion | Winlogs, Sysmon und PowerShell im bisherigen Projektstand als sichtbar dokumentiert | Aktueller Erfassungsweg, Kanäle und Vollständigkeit |
| Wazuh-Windows-Agent | Agent `001` / `windows-server` liefert das belegte 4624-Ereignis | Aktuellen Verbindungsstatus separat prüfen; frühere Erreichbarkeitsprobleme nicht pauschal als fortbestehend annehmen |
| Datei-FIM | Frühere Änderung an `C:\fim_test\test.txt` mit Regel 550 dokumentiert | Keine vollständige FIM-Abdeckung daraus ableiten |
| Registry-FIM | Im bisherigen Projektstand ungeklärt | Gezielte Teständerung und zugehöriges Ereignis dokumentieren |
| Rootcheck | Im bisherigen Projektstand aktiv | Ein positiver Erkennungsnachweis fehlt |
| Schwachstellenerkennung und YARA | Nicht abschließend geprüft | Konfiguration, Test und Ergebnis getrennt nachweisen |
| Wazuh → Security Onion | Weiterleitung nicht nachgewiesen | Transportweg und dasselbe Testereignis auf beiden Seiten belegen |
| OpenVAS / Greenbone | Im zugänglichen Projektstand kein Scan belegt | Eigenständigen Scan- und Remediation-Durchlauf planen |

## Nächste Schritte

1. Aktuelle Dokumentation und Bildlinks prüfen. Für Übung 1 ist der Abschlussstatus **durchgeführt, Korrelation teilweise offen** angemessen.
2. Vor weiteren Tests Netzisolation, Zeitzonen und Uhrensynchronisation sowie die beteiligten Logquellen festhalten.
3. Falls Übung 1 vollständig korreliert werden soll: einen klar abgegrenzten Durchlauf mit Testkonto, Start-/Endzeit, auslösender Aktion und den zugehörigen Netzwerk-/Hostereignissen dokumentieren. Bereits funktionierende Komponenten dafür nicht pauschal neu konfigurieren.
4. Danach die weiteren Tutoring-Übungen einzeln bearbeiten; pro Übung Ergebnisse, Erkennung, Grenzen und Reaktion dokumentieren. OpenVAS, YARA und zentrale Wazuh-Weiterleitung bleiben eigene Arbeitspakete.

## Arbeitsweise

- Tests nur auf den eigenen freigegebenen Laborzielen durchführen.
- Vor Änderungen den Zustand und bei Bedarf einen VM-Snapshot festhalten.
- Pro Untersuchung eine nachvollziehbare Änderung oder Testhandlung ausführen.
- Funktionierende Komponenten nur bei konkretem Anlass verändern.
- Befehl, Zeitpunkt mit Zeitzone, Konto, Quell-/Zielsystem, Ergebnis, Logquelle und Beleg gemeinsam erfassen.
- Beobachtungen, Regelhinweise und bestätigte Angriffsergebnisse getrennt beschreiben.

## Quellen und Nachweisgrenzen

Grundlagen sind das bereitgestellte `Tutoring.docx`, die Screenshots der bisherigen GitHub-README, die vorhandenen Projektunterlagen vom 12.10.2026 sowie die acht Bildbelege in Übung 1. Die Datei- und Registry-FIM-Prüfungen sind keine Nachweise für Responder-Erkennung.

Der [Tutoring-Abgleich](docs/TUTORING-ABGLEICH.md) trennt Lernziele von technisch zu prüfenden oder missverständlich formulierten Aussagen. Diese Überarbeitung ändert die Dokumentation, nicht die laufenden Labor-VMs.
