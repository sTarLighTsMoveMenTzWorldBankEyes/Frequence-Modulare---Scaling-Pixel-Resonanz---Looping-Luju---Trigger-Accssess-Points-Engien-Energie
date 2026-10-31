# Flow-Engine: Sensor → Trigger → Settlement
## Detaillierte Systembeschreibung

### 1. Einleitung und Zielsetzung

Das hier beschriebene System ist eine **Flow-Engine**, die kontinuierliche Signale (akustische Frequenzen, Bewegungen, Systemdaten) in diskrete Ereignisse umwandelt und daraus automatisierte Aktionen (Settlements) ableitet. 

Im Gegensatz zu klassischen gestapelten Systemen arbeitet die Engine **fließend**: Events überlappen sich, beeinflussen sich gegenseitig und bilden einen geschlossenen Kreislauf. Das Ziel ist nicht die bloße Anzeige von Daten, sondern die Schaffung eines lebendigen, reaktiven Systems, in dem Messung, Entscheidung und Ausführung untrennbar miteinander verbunden sind.

---

### 2. Die drei Kernschichten

#### 2.1 Sensor-Schicht (Messung)

Die Sensor-Schicht bildet die Grundlage. Sie erfasst kontinuierlich Rohdaten aus verschiedenen Quellen und stellt sie in normalisierter Form bereit.

**Quellen:**

- **Spektral-Sensor**  
  Analysiert Audiodaten in Echtzeit (Mikrofon, Audiodatei oder Stream).  
  Liefert:
  - Spectral Energy (0–1): Gesamtenergie des Signals
  - Spectral Centroid (0–1): Schwerpunkt der Frequenzen
  - Optional: Band-Energien (Bass, Mitten, Höhen), Onset-Stärke, BPM

- **Bewegungs-Sensor**  
  Erfasst Maus- oder Touch-Bewegungen.  
  Liefert:
  - Winkel der Kreisbewegung
  - Rotationsgeschwindigkeit
  - Anzahl vollständiger Umdrehungen
  - Scroll-Velocity und Frame-Overlap (Delay-Effekt)

- **System- und Stream-Sensor** (erweiterbar)  
  Kann CPU-/GPU-Last, Frame-Times, Streaming-Statistiken oder Pixel-Änderungen einbeziehen.

Alle Sensoren arbeiten mit einer hohen Abtastrate (typischerweise 60 Hz) und geben ihre Werte kontinuierlich weiter.

#### 2.2 Trigger-Schicht (Ereignis-Erzeugung)

Die Trigger-Schicht wandelt die kontinuierlichen Sensorwerte in diskrete, bedeutungsvolle Events um. Sie entscheidet, wann ein Wert relevant genug ist, um eine Aktion auszulösen.

**Trigger-Typen:**

| Trigger | Auslösebedingung | Typische Bedeutung |
|---------|------------------|--------------------|
| Beat-Trigger | Energy überschreitet Schwelle + Onset erkannt | Rhythmischer Impuls |
| Sync-Trigger | Mehrere Sensorwerte liegen innerhalb definierter Toleranzen | Synchronisation erreicht |
| Kreis-Trigger | Vollständige Umdrehung der Maus | Manueller Modulator |
| Peak-Trigger | Energy-Spitze über festem Schwellwert | Starker Impuls |
| Flow-Trigger | Mehrere Events innerhalb eines kurzen Zeitfensters | Dynamische Sequenz |
| Delay-Trigger | Hohe Scroll-Velocity mit Frame-Overlap | Visuelle Überlappung |

Jedes Event enthält:
- Typ
- Zeitstempel
- Stärke (0–1)
- Zugehörige Sensorwerte

Die Trigger-Schicht ist so konzipiert, dass Events sich überlappen und gegenseitig verstärken können. Dadurch entsteht der charakteristische **Flow** statt eines starren Stapelns.

#### 2.3 Settlement-Schicht (Ausführung)

Die Settlement-Schicht empfängt die Events und führt konkrete Aktionen aus. Sie ist die ausführende Instanz der Engine.

**Mögliche Settlements:**

1. **Token-bezogene Aktionen**  
   Minting, Belohnung oder Aktualisierung eines Balances auf Basis der Event-Stärke.

2. **State-Updates**  
   Speicherung der Event-Historie, Aktualisierung von Scores (Sync-Score, Flow-Score).

3. **Visuelles Feedback**  
   Steuerung von Partikeln, Pads oder anderen visuellen Elementen in Abhängigkeit von den Events.

4. **Kettenreaktionen**  
   Ein Event kann die Bedingungen für nachfolgende Trigger verändern oder neue Trigger freischalten.

Die Settlement-Schicht arbeitet transaktional: Jedes Event wird verarbeitet, der Systemzustand wird aktualisiert und die Ausgabe wird sichtbar gemacht.

---

### 3. Der geschlossene Loop

Der Datenfluss verläuft wie folgt:

```
Sensoren (kontinuierlich)
        ↓
Trigger-Engine (Events)
        ↓
Settlement (Aktionen)
        ↓
Visualisierung + State + Token
        ↓
zurück zu den Sensoren
```

**Eigenschaften des Loops:**
- Kontinuierlich und live
- Events überlappen sich
- Ein Trigger kann den nächsten beeinflussen
- Manuelle Eingaben (Mauskreise) und automatische Sensoren sind gleichberechtigt
- Der Loop ist erweiterbar, ohne die Grundstruktur zu verändern

---

### 4. Datenfluss im Detail

1. Die Sensoren liefern in jedem Frame aktuelle Werte.
2. Die Trigger-Engine prüft die Werte gegen definierte Bedingungen.
3. Bei Erfüllung einer Bedingung wird ein Event erzeugt.
4. Das Event wird an die Settlement-Schicht übergeben.
5. Die Settlement-Schicht führt die zugehörige Aktion aus und aktualisiert den Systemzustand.
6. Die aktualisierten Werte fließen wieder in die Sensor- und Trigger-Schicht zurück.

Dieser Kreislauf ermöglicht es, dass sich das System selbst beobachtet und auf seine eigenen Zustände reagiert.

---

### 5. Erweiterungsmöglichkeiten

- **Mehrere parallele Layer**: Verschiedene Sensor-Gruppen können unabhängig voneinander arbeiten und später zusammengeführt werden.
- **Unterschiedliche Token-Typen**: Je nach Trigger-Art können verschiedene Token ausgegeben werden.
- **On-Chain-Anbindung**: Events können über ein Backend oder Orakel an einen Smart Contract weitergeleitet werden.
- **Multi-User-Synchronisation**: Mehrere Teilnehmer können ihre Trigger in einem gemeinsamen Loop zusammenführen.
- **Historisierung**: Alle Events können als Zeitreihe gespeichert und später analysiert werden.

---

### 6. Abgrenzung und Realismus

Das System ist eine leistungsfähige **Event-Flow-Engine**. Es wandelt kontinuierliche Signale in strukturierte Aktionen um und ermöglicht komplexe, fließende Interaktionen.

Es erzeugt jedoch **keinen automatischen Geldfluss**. Tokens oder Settlements haben nur dann Wert, wenn eine Nachfrage oder ein Nutzen außerhalb des Systems existiert. Die Engine dokumentiert und reagiert auf Ereignisse – sie schafft nicht von selbst wirtschaftlichen Wert.

---

### 7. Zusammenfassung

Die Flow-Engine besteht aus drei klar getrennten, aber eng verbundenen Schichten:

- **Sensor**: Misst kontinuierlich
- **Trigger**: Entscheidet und erzeugt Events
- **Settlement**: Führt aus und aktualisiert den Zustand

Durch den geschlossenen Loop entsteht ein System, das nicht starr stapelt, sondern fließt. Es reagiert auf Spektren, Bewegungen und Systemzustände und kann diese Reaktionen in visuelle, zustandsbasierte oder tokenbezogene Ausgaben übersetzen.

Das Konzept ist technisch umsetzbar und modular erweiterbar. Sein Wert liegt in der Klarheit der Architektur und in der Möglichkeit, kontinuierliche Signale in bedeutungsvolle, fließende Aktionen zu verwandeln.
