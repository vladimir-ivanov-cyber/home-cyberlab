# Credential Harvesting mit Responder – Übung 1

## 1. Ziel der Übung

Simulation eines LLMNR/NBT-NS/mDNS-Poisoning-Angriffs in einer isolierten VirtualBox-Laborumgebung. Ziel war es, eine NTLMv2-Authentifizierungsantwort zu erfassen und die zugehörigen Netzwerkereignisse mit Security Onion zu analysieren.

## 2. Laborumgebung

| System | IP-Adresse | Rolle |
|---|---|---|
| Kali Linux | 192.168.100.50 | Angreifer / Responder |
| Windows Server (DC01) | 192.168.100.20 | Testsystem |
| Security Onion | 192.168.100.10 | Netzwerküberwachung / SOC |

Die Simulation wurde ausschließlich im eigenen, isolierten Home Cyberlab durchgeführt.

## 3. Durchführung und Ergebnis

Am 12.10.2026 wurde Responder 3.2.2.0 auf Kali Linux eingesetzt.

Responder beantwortete Namensauflösungsanfragen für `soc-lab` beziehungsweise `soc-lab.local` mit manipulierten Antworten.

Anschließend wurde eine SMB-basierte NTLMv2-SSP-Authentifizierungsantwort erfasst.

**Beobachtete Daten:**

- Client: `192.168.100.20`
- Benutzer: `CYBERLAB\labtest`
- Protokoll: SMB / NTLMv2-SSP
- Ergebnis: NTLMv2-Challenge-Response erfasst

Der erfasste Wert ist **kein Klartextpasswort**. Eine erfolgreiche Wiederherstellung des Passworts wurde nicht nachgewiesen.

### Screenshot 1 – Responder

![Responder – NTLMv2-SSP-Erfassung](./Екранна%20снимка%202026-10-12%20211408.png)

## 4. SOC-Analyse mit Security Onion

In Security Onion Hunt wurden fünf zusammengehörige Ereignisse für die Kommunikation zwischen Windows Server und Kali gefunden.

| Ereignistyp | Beobachtung |
|---|---|
| `zeek.ntlm` | NTLM-Authentifizierung des Benutzers `labtest` |
| `zeek.conn` | Netzwerkverbindung zwischen DC01 und Kali |
| `suricata.alert` | NTLM Session Setup Request – Negotiate |
| `suricata.alert` | NTLM Session Setup Request – Auth |
| `endpoint.events.network` | Netzwerkereignis auf DC01 |

Die beiden Suricata-Erkennungen zeigen, dass die NTLM-Aushandlung und die Authentifizierungsphase im Netzwerk sichtbar waren.

Die Regel **ET INFO NTLM Session Setup Request – Auth** hatte die Signatur-ID `2067087` und war als **Informational** klassifiziert.

Diese Erkennungen belegen NTLM-Netzwerkaktivität, sind für sich allein jedoch kein eindeutiger Nachweis eines bösartigen Angriffs.

### Screenshot 2 – Security Onion Hunt

![Security Onion – fünf korrelierte Ereignisse](./Екранна%20снимка%202026-10-12%20212004.png)

## 5. MITRE ATT&CK

**T1557.001 – Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay**

Die Simulation demonstriert das Spoofing lokaler Namensauflösungsantworten, um eine NTLM-Authentifizierung zu einem kontrollierten SMB-Dienst umzuleiten.

Ein tatsächlicher SMB-Relay-Angriff wurde nicht nachgewiesen.

## 6. Lessons Learned

- Unsichere lokale Namensauflösungsverfahren können für Spoofing missbraucht werden.
- NTLMv2-Challenge-Response-Daten sind nicht mit einem Klartextpasswort gleichzusetzen.
- Security Onion kann die zugehörige NTLM-Kommunikation über Zeek und Suricata sichtbar machen.
- Informationsmeldungen von Suricata müssen im Kontext analysiert werden.
- Die Kombination aus Netzwerkereignissen und Responder-Ausgabe liefert eine nachvollziehbare Angriffskette.

## 7. Abschluss

**Status: Erfolgreich abgeschlossen am 12.10.2026.**

Die NTLMv2-SSP-Antwort des Laborbenutzers `CYBERLAB\labtest` wurde erfasst und die Netzwerkkommunikation durch Security Onion dokumentiert.

Responder wurde anschließend gestoppt.

**Nächster Projektschritt:** Fortsetzung gemäß `Tutoring(10).docx`, ohne Wiederholung bereits abgeschlossener Übungen und ohne Änderungen an VirtualBox-Netzwerken oder Wazuh.
