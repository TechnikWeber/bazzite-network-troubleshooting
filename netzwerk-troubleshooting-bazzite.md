# Netzwerkabbrüche unter Bazzite / Fedora — Diagnose-Anleitung

> Für Claude: Diese Datei fasst eine frühere Fehlersuche zusammen. Bitte zuerst lesen,
> dann den großen Diagnose-Befehl unten ausführen lassen und Schritt für Schritt eingrenzen.
> **Wichtigste Lehre aus dem letzten Mal: DNS ZUERST prüfen, nicht zuletzt.**

## Symptom

Internet "reißt ab" — vor allem unter Last (Steam-Download + Firefox parallel).
Nach manuellem Deaktivieren/Aktivieren der Verbindung (oder Neustart) geht es sofort wieder.
Andere Geräte im Netzwerk sind normal schnell.

## System (Beispiel-Setup)

- Bazzite (immutable, rpm-ostree) — Fedora-basiert, atomic
- Realtek RTL8168h/8111h Netzwerkkarte, Treiber `r8169`
- Interface: `enp4s0`, Router/Fritzbox: `192.168.178.1`
- Anschluss über PPPoE (DSL) → echte MTU ist **1492**, nicht 1500

---

## Der eine große Diagnose-Befehl

Bei einem Abriss **zuerst** diesen Befehl ausführen — am besten DIREKT beim Abriss,
**bevor** die Verbindung neu gestartet wird:

```bash
echo "=== PING ROUTER ==="; ping -c 3 192.168.178.1
echo "=== PING IP EXTERN ==="; ping -c 3 8.8.8.8
echo "=== PING DNS-NAME ==="; ping -c 3 google.com
echo "=== DNS-SERVER ==="; resolvectl status | grep -A5 enp4s0
echo "=== LINK + STATS ==="; ip -s link show enp4s0
echo "=== MTU-TEST ==="; ping -M do -s 1464 8.8.8.8 -c 3
echo "=== OFFLOAD ==="; sudo ethtool -k enp4s0 | grep -E "tcp-seg|generic-seg|generic-rec"
echo "=== DMESG FRISCH ==="; sudo dmesg --ctime | grep -iE "r8169|enp4s0|timed out|reset|hang" | tail -15
echo "=== KERNEL-ARGS ==="; cat /proc/cmdline
```

(Interface-Name ggf. anpassen — mit `ip a` rausfinden, meist `enp...` oder `eno...`.
DNS-Tests gegen Router & Cloudflare zum Vergleich: `nslookup google.com 192.168.178.1`
und `nslookup google.com 1.1.1.1`.)

---

## So liest man das Ergebnis (Diagnose-Logik)

Das Drei-Ebenen-Ping ist der Schlüssel. Es trennt drei völlig verschiedene Fehlerquellen:

| Router (192.168.178.1) | IP extern (8.8.8.8) | Name (google.com) | → Ursache |
|---|---|---|---|
| ✓ geht | ✓ geht | ✗ scheitert | **DNS-Problem** (häufigster Fall!) |
| ✓ geht | ✗ scheitert | ✗ scheitert | Problem zwischen Router & Provider (Leitung/Fritzbox) |
| ✗ scheitert | ✗ scheitert | ✗ scheitert | Lokales Link-/Treiber-Problem |

**Wenn 8.8.8.8 geht aber google.com nicht → es ist DNS. Punkt.**
Für Browser & Steam sieht das exakt aus wie "Internet weg", obwohl die Leitung gesund ist.
Ein Interface-Neustart hilft nur, weil er nebenbei den DNS-Cache leert.

---

## Die Fixes (in Reihenfolge der Wahrscheinlichkeit)

### 1. DNS fest setzen — DER eigentliche Fix beim letzten Mal

Ursache war: die Fritzbox als DNS-Server warf unter Last Timeouts.
Lösung: Router-DNS ignorieren, direkt Cloudflare + Quad9 nutzen.

```bash
CON=$(nmcli -g NAME,DEVICE con show --active | grep enp4s0 | cut -d: -f1)
nmcli connection modify "$CON" ipv4.ignore-auto-dns yes ipv4.dns "1.1.1.1 9.9.9.9"
nmcli connection modify "$CON" ipv6.ignore-auto-dns yes ipv6.dns "2606:4700:4700::1111 2620:fe::fe"
nmcli connection up "$CON"
```

Prüfen: `resolvectl status | grep -A5 enp4s0` → "Current DNS Server" muss `1.1.1.1` zeigen.

### 2. MTU auf 1492 (bei PPPoE/DSL)

Test: `ping -M do -s 1472 8.8.8.8` → wenn "Fragmentierung benötigt (mtu=1492)" kommt,
ist die MTU zu hoch. Korrekter Wert steht in der Fehlermeldung.

```bash
CON=$(nmcli -g NAME,DEVICE con show --active | grep enp4s0 | cut -d: -f1)
nmcli connection modify "$CON" 802-3-ethernet.mtu 1492
nmcli connection up "$CON"
```

Gegenprobe: `ping -M do -s 1464 8.8.8.8 -c 3` → muss 0% packet loss zeigen
(1464 + 28 Header = 1492).

### 3. Offloading aus (RTL8168h Vorbeugung)

Über NetworkManager (zuverlässig, übersteht Reboot) — NICHT über udev-Regeln, die greifen nicht zuverlässig:

```bash
CON=$(nmcli -g NAME,DEVICE con show --active | grep enp4s0 | cut -d: -f1)
nmcli connection modify "$CON" ethtool.feature-tso off ethtool.feature-gso off ethtool.feature-gro off ethtool.feature-tx off ethtool.feature-rx off
nmcli connection up "$CON"
```

Prüfen: `sudo ethtool -k enp4s0 | grep -E "tcp-seg|generic-seg|generic-rec"` → alle `off`.

### 4. ASPM aus (nur falls Verdacht auf Treiber-/Link-Hang)

Stromsparfunktion, die manche Realtek-Karten zum Hängen bringt. Auf Desktop unschädlich.

```bash
rpm-ostree kargs --append=pcie_aspm=off
# danach Reboot nötig
```

Prüfen nach Reboot: `cat /proc/cmdline` muss `pcie_aspm=off` enthalten.

---

## Was NICHT die Ursache war (zum Ausschließen)

Beim letzten Mal lange verfolgt, aber alles Sackgassen:

- **NIC-Paketverlust**: `ethtool -S enp4s0 | grep -iE "err|drop|miss"` → alle 0 = Karte unschuldig
- **Treiber-Hang r8169**: kein frischer Fehler in `dmesg --ctime` beim Abriss = kein Hang
- **Conntrack-Tabelle voll**: `cat /proc/sys/net/netfilter/nf_conntrack_count` vs `_max`
  → war bei 0,2%, also weit weg = nicht das Problem
- **RX-Puffer**: bei dieser Karte fix auf 256, lässt sich nicht erhöhen = irrelevant

Treiberwechsel auf `r8168` oder Kauf einer Intel-Netzwerkkarte war NICHT nötig.

---

## Notfall-Sofortlösung (wenn es gerade abgerissen ist)

Verbindung neu anstoßen ohne Reboot:

```bash
sudo ip link set enp4s0 down && sleep 2 && sudo ip link set enp4s0 up
```

Als Alias zum Merken:

```bash
echo "alias fixnet='sudo ip link set enp4s0 down && sleep 2 && sudo ip link set enp4s0 up'" >> ~/.bashrc && source ~/.bashrc
# danach reicht: fixnet
```

---

## Einstellungen sichern (nach erfolgreichem Fix)

Die NetworkManager-Profile liegen in `/etc/` und überleben Updates/Rollbacks automatisch.
Trotzdem ein Backup, falls man selbst aus Versehen am Profil dreht:

```bash
sudo cp -r /etc/NetworkManager/system-connections ~/netzwerk-backup-$(date +%F)
```

### Bazzite-System-Snapshot (Schutz gegen kaputte Updates)

```bash
sudo ostree admin pin 0      # aktuelles Deployment festpinnen
rpm-ostree status            # prüfen: "Pinned: yes"
# Pin lösen falls nötig: sudo ostree admin pin --unpin 0
```

Updates laufen danach normal weiter, der gepinnte Stand bleibt als Rückfallebene erhalten.
Die `/etc/`-Einstellungen (DNS, MTU, Offloading) bleiben bei Updates UND Rollbacks bestehen.

---

## Merksatz für nächstes Mal

> **Erst die drei Ping-Ebenen (Router → IP → Name), dann gezielt fixen.**
> Wenn `8.8.8.8` geht aber `google.com` nicht → DNS. Das spart die ganze Hardware-Suche.
