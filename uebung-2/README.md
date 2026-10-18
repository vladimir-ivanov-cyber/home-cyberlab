# Übung 2 – Exploitation und Detection

## 1. Ziel der Übung

Ziel dieser Übung war es, eine Schwachstelle auf einem Windows Server innerhalb eines isolierten Cybersecurity-Labors auszunutzen und die dadurch entstandenen Netzwerk- und Endpoint-Ereignisse zu untersuchen.

Dazu wurden **Nmap**, **Metasploit**, **Security Onion**, **Zeek**, **Suricata**, **Kibana** und **Wazuh** eingesetzt.

Da die Untersuchung der vorhandenen SMB-Dienste keine zuverlässig ausnutzbare Schwachstelle nachwies, wurde auf dem Windows Server eine bewusst verwundbare HTTP-Testanwendung eingerichtet.

Die Schwachstelle bestand in einer fehlenden Authentifizierungsprüfung. Sie wurde mit einem eigens erstellten Metasploit-Auxiliary-Modul demonstriert.

## 2. Laborumgebung

| System | IP-Adresse | Aufgabe |
|---|---|---|
| Kali Linux | 192.168.100.50 | Angriffssimulation mit Nmap und Metasploit |
| Windows Server 2025 (DC01) | 192.168.100.20 | Zielsystem mit verwundbarem HTTP-Testdienst |
| Security Onion 3.1.0 | 192.168.100.10 | Netzwerküberwachung mit Zeek und Suricata |
| Ubuntu / Wazuh | 192.168.100.30 | Endpoint-Überwachung des Windows Servers |

Die Systeme wurden innerhalb eines isolierten VirtualBox-Labornetzes betrieben.

## 3. Reconnaissance mit Nmap

Zunächst wurde der Windows Server auf erreichbare TCP-Dienste untersucht.

```bash
sudo nmap -sT -sV -Pn --open 192.168.100.20
```

Der Scan zeigte verschiedene Windows- und Active-Directory-Dienste, darunter DNS, Kerberos, LDAP, SMB und RPC.

Ein zusätzlicher SMB-Scan bestätigte, dass SMB-Signing aktiviert und erforderlich war. Metasploit erkannte SMBv2 und SMBv3 mit SMB 3.1.1 als bevorzugtem Dialekt.

Der MS17-010-Scanner konnte die Prüfung wegen eines SMB-IPC$-Anmeldefehlers nicht abschließen. Eine MS17-010-Schwachstelle wurde daher nicht nachgewiesen.

Für die anschließende kontrollierte Angriffssimulation wurde auf DC01 ein separater HTTP-Testdienst eingerichtet.

```bash
nmap -sV -p 8080 192.168.100.20
```

**Ergebnis:** TCP-Port 8080 war geöffnet. Nmap klassifizierte den Dienst vorläufig als `http-proxy`, konnte die selbst entwickelte Anwendung jedoch keiner bekannten Produktversion zuordnen.

## 4. Verwundbarer HTTP-Testdienst

Auf Windows Server DC01 wurde ein PowerShell-basierter HTTP-Dienst auf Port 8080 eingerichtet.

Der Endpunkt `/admin/proof` enthielt absichtlich keine Authentifizierungsprüfung.

Ein Aufruf dieses Endpunkts konnte die fest definierte Datei

`C:\Users\Public\Cyberlab-Exercise2\proof.txt`

erstellen beziehungsweise überschreiben.

Die Anwendung akzeptierte keine beliebigen Systembefehle über HTTP. Es handelte sich somit um eine **Broken-Access-Control-Schwachstelle**, nicht um eine Remote-Code-Execution-Schwachstelle.

### Funktionstest mit curl

```bash
curl --max-time 10 -i http://192.168.100.20:8080/admin/proof
```

Beobachtete Antwort:

```text
HTTP/1.1 200 OK
Content-Length: 45
Connection: close

VULNERABILITY CONFIRMED: proof marker created
```

Die Anwendung führte die festgelegte administrative Testaktion aus, obwohl keine Anmeldung stattgefunden hatte.

## 5. Exploitation mit Metasploit

Da für die selbst entwickelte Testanwendung kein vorhandenes Metasploit-Modul existierte, wurde ein eigenes Auxiliary-Modul erstellt.

**Modul:** `auxiliary/scanner/http/cyberlab_proof`

Es sendete eine nicht authentifizierte HTTP-GET-Anfrage an `/admin/proof` und überprüfte die Antwort auf die erwartete Erfolgsmeldung.

```text
use auxiliary/scanner/http/cyberlab_proof
set RHOSTS 192.168.100.20
run
```

Beobachtetes Ergebnis:

```text
[+] 192.168.100.20:8080 - Unauthorized action confirmed
[*] Auxiliary module execution completed
```

**Bewertung:** Die absichtlich fehlende Zugriffskontrolle wurde erfolgreich mit Metasploit demonstriert. Es wurde keine Shell geöffnet und keine bekannte Windows-CVE ausgenutzt. Das Auxiliary-Modul bestätigte die HTTP-Antwort; eine separate Prüfung des Dateiinhalts wurde nicht dokumentiert.

## 6. Network Detection – Security Onion und Suricata

In Security Onion wurde der Netzwerkverkehr zwischen Kali Linux und DC01 untersucht.

Die Hunt-Abfrage lautete:

```text
source.ip: 192.168.100.50 AND destination.ip: 192.168.100.20 AND destination.port: 8080
```

Die Suche zeigte acht passende Ereignisse, darunter Datensätze von `zeek.http`, `zeek.conn`, `zeek.file`, Endpoint-Netzwerktelemetrie und `suricata.alert`.

### Suricata-Alert

| Merkmal | Beobachtung |
|---|---|
| Zeitpunkt | 18.10.2026, 15:52:02 Uhr |
| Quell-IP | 192.168.100.50 |
| Quellport | 35810 |
| Ziel-IP | 192.168.100.20 |
| Zielport | 8080/TCP |
| Signatur-ID | 2034567 |
| Signatur | ET HUNTING curl User-Agent to Dotted Quad |
| Kategorie | Potentially Bad Traffic |
| Severity | 2 / Medium |
| Aktion | allowed |

Die Signatur reagierte auf einen HTTP-Aufruf mit dem User-Agent `curl`, bei dem eine numerische IP-Adresse als Ziel verwendet wurde.

Die betreffende Anfrage lautete:

```http
GET /admin/proof HTTP/1.1
Host: 192.168.100.20:8080
User-Agent: curl/8.18.0
```

**Bewertung:** Suricata erkannte ein verdächtiges HTTP-Muster beim separaten curl-Test. Die Regel erkannte nicht spezifisch die erfolgreiche Ausnutzung der Zugriffskontrollschwäche. Der Datenverkehr wurde nicht blockiert.

## 7. Kibana – Untersuchung der HTTP-Kommunikation

Mit Kibana Discover und der Datenansicht `logs-*` wurde der Netzwerkverkehr im Zeitraum vom 18.10.2026 zwischen 15:50 und 16:00 Uhr analysiert.

Die Untersuchung ergab einen Zeek-HTTP-Datensatz um 15:55:19 Uhr.

| Feld | Wert |
|---|---|
| Quelle | 192.168.100.50 |
| Quellport | 37907 |
| Ziel | 192.168.100.20 |
| Zielport | 8080 |
| HTTP-Methode | GET |
| HTTP-URI | /admin/proof |
| HTTP-Status | 200 OK |
| HTTP-Response-Body-Länge | 45 Bytes |
| Datenquelle | zeek.http |

Der HTTP-User-Agent war als Browserkennung mit `Mozilla/5.0` und `Chrome/131.0.0.0` angegeben.

Die Anfrage lag zeitlich beim erfolgreichen Metasploit-Test und verwendete denselben verwundbaren Endpunkt.

**Bewertung:** Zeek und Kibana bestätigten die Netzwerkkommunikation, die aufgerufene URI und die erfolgreiche HTTP-Antwort. Die Logdaten unterstützen die Zuordnung zum Metasploit-Test, beweisen allein jedoch keine erfolgreiche Dateierstellung.

## 8. Endpoint Detection – Wazuh

Auf dem Windows Server war ein Wazuh-Agent mit dem Namen `windows-server` eingerichtet.

Im Untersuchungszeitraum 15:45–16:05 Uhr wurden zunächst 145 Wazuh-Ereignisse für den Windows-Agenten gefunden.

Bei einer gezielten Suche nach PowerShell- und Dateiaktivitäten erschien ein Alert mit folgenden Informationen:

| Feld | Ergebnis |
|---|---|
| Wazuh-Agent | windows-server |
| Agent-IP | 192.168.100.20 |
| Zeitpunkt | 18.10.2026, 15:50:14 Uhr |
| Regel-ID | 92213 |
| Regel-Level | 15 |
| Beschreibung | Executable file dropped in folder commonly used by malware |
| Sysmon Event ID | 11 – File Create |
| Prozess | powershell.exe |
| Benutzer | CYBERLAB\Administrator |

Die erkannte Datei lag im temporären Verzeichnis des Administrators und begann mit `__PSScriptPolicyTest_`.

Dieses Dateinamensmuster kann im Zusammenhang mit der PowerShell-Ausführungsrichtlinienprüfung auftreten.

### Bewertung

Wazuh erkannte eine PowerShell-bezogene Dateierstellung und bewertete diese mit Level 15.

Der Alert konnte jedoch nicht eindeutig dem erfolgreichen Metasploit-Test zugeordnet werden. Die beobachtete Datei war nicht `proof.txt`, und der Alarm trat zeitlich vor dem dokumentierten Metasploit-Test auf.

Es handelt sich daher um einen möglichen False Positive beziehungsweise um ein separates PowerShell-Ereignis.

Eine direkte Wazuh-Erkennung der absichtlich ausgenutzten Zugriffskontrollschwäche wurde nicht nachgewiesen.

## 9. Vergleich der Erkennungsergebnisse

| Werkzeug | Beobachtung | Ergebnis |
|---|---|---|
| Nmap | HTTP-Dienst auf Port 8080 erkannt | Erfolgreich |
| Metasploit | Unautorisierte Testaktion bestätigt | Erfolgreich |
| Zeek | HTTP-Anfragen und TCP-Verbindungen aufgezeichnet | Erfolgreich |
| Suricata | Curl-basierte HTTP-Anfrage erkannt | Alert erzeugt |
| Kibana | URI, IP-Adressen, Ports und HTTP-Status korreliert | Erfolgreich |
| Wazuh | PowerShell-Dateiereignis aufgezeichnet | Alert vorhanden, Zuordnung nicht bestätigt |

## 10. Technische Grenzen

Diese Übung nutzte eine bewusst selbst erstellte Schwachstelle. Der Test ist deshalb keine Demonstration einer veröffentlichten Windows-CVE.

Das Metasploit-Auxiliary-Modul löste eine fest definierte Aktion aus, ohne beliebigen Code auf dem Zielsystem auszuführen.

Die Suricata-Erkennung bezog sich auf einen separaten curl-Aufruf. Der Wazuh-Alert bezog sich auf eine temporäre PowerShell-Datei und nicht auf die nachzuweisende Markerdatei.

Die Untersuchung zeigt somit sowohl die Möglichkeiten der Netzwerk- und Endpoint-Überwachung als auch die Grenzen einer automatischen Angriffserkennung.

## 11. Fazit

In Übung 2 wurde eine bewusst verwundbare HTTP-Anwendung auf Windows Server DC01 erfolgreich getestet.

Nmap identifizierte den erreichbaren Dienst. Ein selbst erstelltes Metasploit-Modul demonstrierte die fehlende Zugriffskontrolle. Security Onion, Zeek und Kibana ermöglichten die Untersuchung der beteiligten IP-Adressen, Ports und HTTP-Muster.

Suricata erkannte verdächtigen curl-Verkehr, während Wazuh eine separate PowerShell-Dateierstellung protokollierte. Nicht jede erkannte Aktivität konnte eindeutig dem erfolgreichen Schwachstellentest zugeordnet werden.

Die Übung verdeutlichte insbesondere, dass ein Netzwerkereignis, ein Sicherheitsalarm und der tatsächliche Nachweis einer erfolgreichen Ausnutzung unterschiedliche Aussagen darstellen.

## 12. Screenshot-Belege

Die folgenden Screenshots dokumentieren die Durchführung und Untersuchung von Übung 2.

Die Belege umfassen:

1. **Nmap-Reconnaissance:** Identifizierung des HTTP-Testdienstes auf TCP-Port 8080.
2. **HTTP-Schwachstellentest:** Erfolgreiche Anfrage an `/admin/proof` ohne Authentifizierung.
3. **Metasploit:** Erfolgreiche Ausführung des eigenen Auxiliary-Moduls.
4. **Security Onion:** Netzwerkereignisse zwischen Kali Linux und DC01.
5. **Suricata:** Untersuchung des HTTP-Alarms mit Signatur-ID 2034567.
6. **Wazuh:** Endpoint-Analyse des Sysmon-Dateiereignisses und der Wazuh-Regel 92213.
7. **Kibana:** Analyse des HTTP-Requests `/admin/proof` und der Antwort `200 OK`.

Die Screenshots werden nachfolgend mit den im Repository gespeicherten Dateinamen verlinkt.

### Screenshot 1 – Nmap-Reconnaissance
![Nmap-Reconnaissance](<Екранна снимка 2026-10-18 154525.png>)

### Screenshot 2 – HTTP-Schwachstellentest
![HTTP-Schwachstellentest](<Екранна снимка 2026-10-18 155247.png>)

### Screenshot 3 – Metasploit
![Metasploit-Exploitation](<Екранна снимка 2026-10-18 155544.png>)

### Screenshot 4 – Security Onion
![Security Onion Hunt](<Екранна снимка 2026-10-18 172543.png>)

### Screenshot 5 – Suricata
![Suricata-Alert](<Екранна снимка 2026-10-18 172741.png>)

### Screenshot 6 – Wazuh
![Wazuh Endpoint-Analyse](<Екранна снимка 2026-10-18 174345.png>)

### Screenshot 7 – Kibana
![Kibana HTTP-Analyse](<Екранна снимка 2026-10-18 180032.png>)
