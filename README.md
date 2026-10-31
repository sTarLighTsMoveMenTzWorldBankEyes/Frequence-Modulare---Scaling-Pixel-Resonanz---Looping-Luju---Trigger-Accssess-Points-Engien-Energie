**Architektur: Sensor → Trigger → Settlement (Flow-Engine)**

Hier ist die detaillierte, ausgearbeitete Architektur des Systems, das wir zusammen entwickelt haben.

### 1. Überblick der drei Schichten

| Schicht | Rolle | Input | Output | Charakter |
|---------|-------|-------|--------|-----------|
| **Sensor** | Misst | Audio, Maus, System, Streams | Rohwerte (Energy, Centroid, BPM, Winkel, Pixel) | Kontinuierlich |
| **Trigger** | Entscheidet | Sensor-Werte + Schwellen | Events (Beat, Sync, Kreis-Durchlauf, Peak) | Diskret / Ereignisbasiert |
| **Settlement** | Führt aus | Trigger-Events | Token-Mint, Reward, State-Update, Visual-Feedback | Transaktional |

Der Fluss ist ein **geschlossener Loop**, kein einmaliger Durchlauf.

---

### 2. Sensor-Schicht (Messung)

Die Sensor-Schicht sammelt kontinuierlich Daten aus mehreren Quellen:

**A. Spektral-Sensor (Web Audio API / Essentia)**
- `spectralEnergy` (0–1): Durchschnittliche Energie der Frequenzen
- `spectralCentroid` (0–1): Schwerpunkt der Frequenzen (tief ↔ hoch)
- Optional: BPM-Erkennung, Onset-Detection, Band-Energien (Bass, Mitten, Höhen)

**B. Bewegungs-Sensor (Maus / Touch)**
- Winkel der Kreisbewegung
- Geschwindigkeit der Drehung
- Anzahl der vollen Umdrehungen
- Scroll-Velocity + Overlap (Delay-Effekt beim schnellen Scrollen)

**C. System- / Stream-Sensor (erweiterbar)**
- CPU-/GPU-Last, Frame-Times
- Streaming-Statistiken (YouTube Views, Spotify Audio Features)
- Pixel-Änderungen im Viewport (optional)

Alle Sensoren liefern Werte in einem festen Takt (z. B. 60× pro Sekunde).

---

### 3. Trigger-Schicht (Ereignis-Erzeugung)

Die Trigger-Schicht wandelt kontinuierliche Werte in diskrete Events um.

**Trigger-Typen:**

| Trigger | Bedingung | Beispiel |
|---------|-----------|----------|
| **Beat-Trigger** | Energy überschreitet Schwelle + Onset | Starker Bass-Schlag |
| **Sync-Trigger** | Energy + Centroid + BPM innerhalb Toleranz | Rhythmus passt zum Metronom |
| **Kreis-Trigger** | Maus vollendet eine Umdrehung | 360°-Durchlauf |
| **Peak-Trigger** | Energy-Spitze über X | Lautes Geräusch |
| **Flow-Trigger** | Mehrere Events innerhalb kurzer Zeit | Schnelle Folge von Beats |
| **Delay-Trigger** | Scroll-Velocity hoch + Overlap erkannt | Schnelles Scrollen erzeugt gestapelte Frames |

Jeder Trigger erzeugt ein Event-Objekt:

```js
{
  type: "beat" | "sync" | "circle" | "peak" | "flow",
  timestamp: Date.now(),
  strength: 0.0–1.0,
  data: { energy, centroid, angle, ... }
}
```

---

### 4. Settlement-Schicht (Ausgabe / Tokenisierung)

Die Settlement-Schicht reagiert auf die Events und führt Aktionen aus.

**Mögliche Settlements:**

1. **Token-Mint / Reward**
   - Bei Sync-Trigger oder Flow-Trigger → Mint eines Tokens oder Erhöhung eines Balances
   - Stärke des Events bestimmt die Menge

2. **State-Update**
   - Speichert Event-Historie on-chain oder off-chain
   - Aktualisiert einen „Sync-Score“ oder „Flow-Score“

3. **Visual-Feedback**
   - Partikel reagieren stärker (Größe, Helligkeit, Farbe)
   - Pads leuchten auf
   - Neue Layer werden eingeblendet

4. **Ketten-Reaktion (Flow)**
   - Ein Trigger kann den nächsten Trigger vorbereiten oder verstärken
   - Beispiel: 3 Beat-Trigger innerhalb von 2 Sekunden → Flow-Trigger

**Settlement-Logik (vereinfacht):**

```
Wenn Event.strength > Schwelle:
  → Settlement ausführen
  → Score erhöhen
  → Visual aktualisieren
  → Optional: nächsten Trigger freischalten
```

---

### 5. Der Loop (Flow-Engine)

```
[Sensoren] 
    ↓ (kontinuierlich)
[Trigger-Engine] 
    ↓ (Events)
[Settlement] 
    ↓
[Visual + Token + State]
    ↓
[zurück zu Sensoren]  ← geschlossener Kreislauf
```

**Eigenschaften des Flows:**
- Nicht gestapelt (high), sondern fließend (flow)
- Events überlappen sich
- Ein Trigger kann den nächsten beeinflussen
- Die Mauskreisung und das Spektrum sind gleichberechtigte Eingaben
- Der Loop läuft live und kontinuierlich

---

### 6. Erweiterungsmöglichkeiten

| Erweiterung | Beschreibung |
|-------------|--------------|
| **Mehrere Layer** | Verschiedene Sensor-Gruppen (Audio, Maus, System) parallel |
| **Token-Typen** | Unterschiedliche Tokens je nach Trigger-Typ (Beat-Token, Sync-Token, Flow-Token) |
| **On-Chain Settlement** | Smart Contract empfängt Events über Orakel oder Backend |
| **Multi-User** | Mehrere Nutzer synchronisieren ihre Trigger |
| **Historie** | Alle Events werden als Zeitreihe gespeichert |

---

### 7. Zusammenfassung

| Schicht | Was sie macht | Was sie nicht macht |
|---------|---------------|---------------------|
| **Sensor** | Misst Frequenzen, Bewegung, System | Erzeugt keinen Wert |
| **Trigger** | Wandelt Werte in Events um | Erzeugt kein Geld |
| **Settlement** | Führt Aktionen aus (Mint, Update, Visual) | Erzeugt Wert nur, wenn Nachfrage existiert |

Das System ist eine **saubere Event-Flow-Engine**.  
Der Geldfluss entsteht nicht automatisch aus den Pixeln oder Frequenzen, sondern nur dann, wenn die erzeugten Tokens oder Settlements von jemandem gewollt und genutzt werden.

---

Möchtest du, dass ich daraus als Nächstes einen konkreten Prototyp baue (z. B. HTML mit Sensor + Trigger + visueller Settlement-Anzeige), oder soll ich die Settlement-Logik als Smart-Contract-Beispiel ausarbeiten?
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
Flow-Engine
Sensor → Trigger → Settlement
Detaillierte Systembeschreibung
1. Einleitung und Zielsetzung
Das hier beschriebene System ist eine Flow-Engine, die kontinuierliche Signale (akustische
Frequenzen, Bewegungen, Systemdaten) in diskrete Ereignisse umwandelt und daraus
automatisierte Aktionen (Settlements) ableitet.
Im Gegensatz zu klassischen gestapelten Systemen arbeitet die Engine fließend: Events überlappen
sich, beeinflussen sich gegenseitig und bilden einen geschlossenen Kreislauf. Das Ziel ist nicht die
bloße Anzeige von Daten, sondern die Schaffung eines lebendigen, reaktiven Systems, in dem
Messung, Entscheidung und Ausführung untrennbar miteinander verbunden sind.
2. Die drei Kernschichten
2.1 Sensor-Schicht (Messung)
Die Sensor-Schicht bildet die Grundlage. Sie erfasst kontinuierlich Rohdaten aus verschiedenen
Quellen und stellt sie in normalisierter Form bereit.
Quellen:
• Spektral-Sensor – Analysiert Audiodaten in Echtzeit (Mikrofon, Audiodatei oder Stream). Liefert
Spectral Energy (0–1), Spectral Centroid (0–1) sowie optional Band-Energien, Onset-Stärke und
BPM.
• Bewegungs-Sensor – Erfasst Maus- oder Touch-Bewegungen. Liefert Winkel der
Kreisbewegung, Rotationsgeschwindigkeit, Anzahl vollständiger Umdrehungen sowie
Scroll-Velocity und Frame-Overlap.
• System- und Stream-Sensor (erweiterbar) – Kann CPU-/GPU-Last, Frame-Times,
Streaming-Statistiken oder Pixel-Änderungen einbeziehen.
Alle Sensoren arbeiten mit einer hohen Abtastrate (typischerweise 60 Hz) und geben ihre Werte
kontinuierlich weiter.
2.2 Trigger-Schicht (Ereignis-Erzeugung)
Die Trigger-Schicht wandelt die kontinuierlichen Sensorwerte in diskrete, bedeutungsvolle Events
um. Sie entscheidet, wann ein Wert relevant genug ist, um eine Aktion auszulösen.
Trigger Auslösebedingung Typische Bedeutung
Beat-Trigger Energy überschreitet Schwelle + Onset
erkannt
Rhythmischer Impuls
Sync-Trigger Mehrere Sensorwerte liegen innerhalb
definierter Toleranzen
Synchronisation erreicht
Kreis-Trigger Vollständige Umdrehung der Maus Manueller Modulator
Peak-Trigger Energy-Spitze über festem Schwellwert Starker Impuls
Flow-Trigger Mehrere Events innerhalb eines kurzen
Zeitfensters
Dynamische Sequenz
Delay-Trigger Hohe Scroll-Velocity mit Frame-Overlap Visuelle Überlappung
Jedes Event enthält Typ, Zeitstempel, Stärke (0–1) und die zugehörigen Sensorwerte. Die
Trigger-Schicht ist so konzipiert, dass Events sich überlappen und gegenseitig verstärken können.
Dadurch entsteht der charakteristische Flow statt eines starren Stapelns.
2.3 Settlement-Schicht (Ausführung)
Die Settlement-Schicht empfängt die Events und führt konkrete Aktionen aus. Sie ist die ausführende
Instanz der Engine.
Mögliche Settlements:
1. Token-bezogene Aktionen – Minting, Belohnung oder Aktualisierung eines Balances auf
Basis der Event-Stärke.
2. State-Updates – Speicherung der Event-Historie, Aktualisierung von Scores (Sync-Score,
Flow-Score).
3. Visuelles Feedback – Steuerung von Partikeln, Pads oder anderen visuellen Elementen in
Abhängigkeit von den Events.
4. Kettenreaktionen – Ein Event kann die Bedingungen für nachfolgende Trigger verändern oder
neue Trigger freischalten.
Die Settlement-Schicht arbeitet transaktional: Jedes Event wird verarbeitet, der Systemzustand wird
aktualisiert und die Ausgabe wird sichtbar gemacht.
3. Der geschlossene Loop
Der Datenfluss verläuft wie folgt:
Sensoren (kontinuierlich)
 ↓
Trigger-Engine (Events)
 ↓
Settlement (Aktionen)
 ↓
Visualisierung + State + Token
 ↓
zurück zu den Sensoren
Eigenschaften des Loops:
• Kontinuierlich und live
• Events überlappen sich
• Ein Trigger kann den nächsten beeinflussen
• Manuelle Eingaben (Mauskreise) und automatische Sensoren sind gleichberechtigt
• Der Loop ist erweiterbar, ohne die Grundstruktur zu verändern
4. Datenfluss im Detail
1. Die Sensoren liefern in jedem Frame aktuelle Werte.
2. Die Trigger-Engine prüft die Werte gegen definierte Bedingungen.
3. Bei Erfüllung einer Bedingung wird ein Event erzeugt.
4. Das Event wird an die Settlement-Schicht übergeben.
5. Die Settlement-Schicht führt die zugehörige Aktion aus und aktualisiert den Systemzustand.
6. Die aktualisierten Werte fließen wieder in die Sensor- und Trigger-Schicht zurück.
Dieser Kreislauf ermöglicht es, dass sich das System selbst beobachtet und auf seine eigenen
Zustände reagiert.
5. Erweiterungsmöglichkeiten
• Mehrere parallele Layer – Verschiedene Sensor-Gruppen können unabhängig voneinander
arbeiten und später zusammengeführt werden.
• Unterschiedliche Token-Typen – Je nach Trigger-Art können verschiedene Token
ausgegeben werden.
• On-Chain-Anbindung – Events können über ein Backend oder Orakel an einen Smart Contract
weitergeleitet werden.
• Multi-User-Synchronisation – Mehrere Teilnehmer können ihre Trigger in einem gemeinsamen
Loop zusammenführen.
• Historisierung – Alle Events können als Zeitreihe gespeichert und später analysiert werden.
6. Abgrenzung und Realismus
Das System ist eine leistungsfähige Event-Flow-Engine. Es wandelt kontinuierliche Signale in
strukturierte Aktionen um und ermöglicht komplexe, fließende Interaktionen.
Es erzeugt jedoch keinen automatischen Geldfluss. Tokens oder Settlements haben nur dann
Wert, wenn eine Nachfrage oder ein Nutzen außerhalb des Systems existiert. Die Engine
dokumentiert und reagiert auf Ereignisse – sie schafft nicht von selbst wirtschaftlichen Wert.
7. Zusammenfassung
Die Flow-Engine besteht aus drei klar getrennten, aber eng verbundenen Schichten:
• Sensor: Misst kontinuierlich
• Trigger: Entscheidet und erzeugt Events
• Settlement: Führt aus und aktualisiert den Zustand
Durch den geschlossenen Loop entsteht ein System, das nicht starr stapelt, sondern fließt. Es
reagiert auf Spektren, Bewegungen und Systemzustände und kann diese Reaktionen in visuelle,
zustandsbasierte oder tokenbezogene Ausgaben übersetzen.
Das Konzept ist technisch umsetzbar und modular erweiterbar. Sein Wert liegt in der Klarheit der
Architektur und in der Möglichkeit, kontinuierliche Signale in bedeutungsvolle, fließende Aktionen zu
verwandeln.
Flow-Engine Konzept – Sensor → Trigger → Settlement
**Gute Eurorack-Module** für Wave, Akustik, Sequenz, BPM, Frequenz und Wave-Sync:

### Sequencer & BPM
| Modul | Hersteller | Stärke |
|-------|------------|--------|
| **Metropolix** | Intellijel | Sehr starker, musikalischer Sequencer mit vielen Rhythmus-Optionen |
| **Voltage Block** | Malekko | 16-Step Sequencer mit hoher Präzision |
| **Pamela’s New Workout** | ALM Busy Circuits | Der Klassiker für Clock, BPM, Divisions und Sync |
| **Tempi** | Make Noise | Flexible Clock- und Timing-Steuerung |
| **Stillson Hammer MKII** | Industrial Music Electronics | Leistungsstarker Sequencer mit vielen Tracks |

### Wave / Oscillator / Sound
| Modul | Hersteller | Stärke |
|-------|------------|--------|
| **Plaits** | Mutable Instruments | Extrem vielseitig, viele Algorithmen, sehr musikalisch |
| **Rings** | Mutable Instruments | Physikalische Modellierung (akustisch klingend) |
| **Elements** | Mutable Instruments | Modal Synthesis, sehr akustisch |
| **DPO** | Make Noise | Klassischer komplexer Oscillator |
| **STO** | Make Noise | Sauberer, präziser Oscillator |

### Frequenz / Spectral / Filter
| Modul | Hersteller | Stärke |
|-------|------------|--------|
| **Ripples** | Mutable Instruments | Sehr guter multimode Filter |
| **QPAS** | Make Noise | Stereo-Filter mit viel Charakter |
| **Spectraphon** | Make Noise | Spectral Oscillator (sehr interessant für Frequenz-Arbeit) |
| **Morphagene** | Make Noise | Granular / Spectral, gut für akustische Texturen |

### Sync / Clock / Modulation
| Modul | Hersteller | Stärke |
|-------|------------|--------|
| **Pamela’s New Workout** | ALM | Beste Allround-Lösung für BPM + Sync |
| **Pexp** | Expert Sleepers | Sehr präzise Clock und Sync |
| **Midi 1U / FH-2** | Expert Sleepers | Starke MIDI-zu-CV Lösung für externe Sync |
| **Stages** | Mutable Instruments | Flexible Modulation und Timing |

### Empfehlung für dein Konzept (Flow / Spectral / BPM)
Wenn du die Ideen aus dem Gespräch (Spektrum, BPM, Trigger, Flow) modular umsetzen willst, sind diese besonders passend:

1. **Pamela’s New Workout** – BPM + Sync-Herzstück  
2. **Plaits** oder **Rings** – Sound-Erzeugung  
3. **Metropolix** oder **Voltage Block** – Sequenz  
4. **Spectraphon** oder **Morphagene** – Frequenz- / Spectral-Ebene  
5. **Stages** – Modulation und Trigger-Verknüpfung  

Möchtest du eine konkrete kleine Modul-Kombination (z. B. 6–8 Module) für ein System, das zu deiner Flow-Engine-Idee passt?
**Konkrete Modul-Kombination** für dein Flow-/Spectral-/BPM-Konzept:

### Empfohlenes System (ca. 84–104 HP)

| Modul | Hersteller | Funktion im System | HP |
|-------|------------|--------------------|----|
| **Pamela’s New Workout** | ALM Busy Circuits | BPM, Clock, Sync, Divisions | 8 |
| **Metropolix** | Intellijel | Sequencer (Rhythmus + Melodie) | 34 |
| **Plaits** | Mutable Instruments | Haupt-Soundquelle (viele Algorithmen) | 12 |
| **Rings** | Mutable Instruments | Akustische / physikalische Klänge | 14 |
| **Spectraphon** | Make Noise | Spectral Oscillator (Frequenz-Ebene) | 26 |
| **Stages** | Mutable Instruments | Modulation, Trigger-Logik, Flow | 14 |
| **Ripples** | Mutable Instruments | Filter | 8 |
| **Veils** | Mutable Instruments | VCA (Lautstärke-Steuerung) | 10 |

**Gesamt:** ca. 126 HP (passt in ein 104 HP Case mit etwas Planung oder in ein größeres Case)

### Signalfluss (vereinfacht)
```
Pamela’s (BPM/Clock)
    ↓
Metropolix (Sequenz)
    ↓
Plaits + Rings + Spectraphon (Sound)
    ↓
Ripples (Filter)
    ↓
Veils (VCA)
    ↓
Ausgang

Stages steuert Modulation, Trigger und Verbindungen zwischen den Layern
```

### Warum diese Kombination
- **Pamela’s** ist das Timing-Herz (BPM + Sync)
- **Metropolix** liefert starke Sequenzen
- **Plaits + Rings** decken elektronische und akustische Klänge ab
- **Spectraphon** bringt die Spectral-/Frequenz-Ebene
- **Stages** verbindet alles und ermöglicht Flow- und Trigger-Logik

Möchtest du davon eine visuelle Darstellung (Eurorack-Rack mit diesen Modulen)?
Hier sind die offiziellen Seiten und Repositories zu den Modulen:

### Pamela’s New Workout / PRO Workout
- **Offiziell:** [busycircuits.com/alm034](https://busycircuits.com/pages/alm034) (PRO Workout – aktuelle Version)  
- Hinweis: Die NEW Workout ist discontinued. Die PRO Workout ist der Nachfolger.

### Metropolix
- **Offiziell:** [intellijel.com/shop/eurorack/metropolix](https://intellijel.com/shop/eurorack/metropolix/)  
- Manual: Auf der Intellijel-Seite verfügbar

### Mutable Instruments Module (Plaits, Rings, Stages, Ripples, Veils)
Diese sind **Open Source**.

- **Haupt-Repository:** [github.com/pichenettes/eurorack](https://github.com/pichenettes/eurorack)  
  (enthält Firmware, Schematics und Code für alle Mutable-Module)

Einzelne Dokumentationen:
- **Plaits:** [mutable-instruments.net/modules/plaits](https://mutable-instruments.net/modules/plaits)
- **Rings:** [mutable-instruments.net/modules/rings](https://mutable-instruments.net/modules/rings)
- **Stages:** [mutable-instruments.net/modules/stages](https://mutable-instruments.net/modules/stages)
- **Ripples:** [mutable-instruments.net/modules/ripples](https://mutable-instruments.net/modules/ripples)
- **Veils:** [mutable-instruments.net/modules/veils](https://mutable-instruments.net/modules/veils)

### Spectraphon
- **Offiziell:** [makenoisemusic.com/modules/spectraphon](https://www.makenoisemusic.com/modules/spectraphon/)  
- Manual und Firmware-Updates sind dort verfügbar.  
- Kein öffentliches GitHub-Repo (closed source).

### Zusammenfassung

| Modul | Offizielle Seite | GitHub / Open Source |
|-------|------------------|----------------------|
| Pamela’s PRO Workout | [busycircuits.com](https://busycircuits.com/pages/alm034) | Nein |
| Metropolix | [intellijel.com](https://intellijel.com/shop/eurorack/metropolix/) | Nein |
| Plaits | [mutable-instruments.net](https://mutable-instruments.net/modules/plaits) | Ja ([eurorack](https://github.com/pichenettes/eurorack)) |
| Rings | [mutable-instruments.net](https://mutable-instruments.net/modules/rings) | Ja |
| Stages | [mutable-instruments.net](https://mutable-instruments.net/modules/stages) | Ja |
| Ripples | [mutable-instruments.net](https://mutable-instruments.net/modules/ripples) | Ja |
| Veils | [mutable-instruments.net](https://mutable-instruments.net/modules/veils) | Ja |
| Spectraphon | [makenoisemusic.com](https://www.makenoisemusic.com/modules/spectraphon/) | Nein |

Die Mutable-Module sind besonders interessant, weil du den kompletten Code und die Schematics frei einsehen und sogar selbst bauen kannst.
Hier sind die Repositories und offiziellen Quellen für alle Module:

### Open Source (Mutable Instruments)

**Haupt-Repository für alle Mutable-Module:**  
[github.com/pichenettes/eurorack](https://github.com/pichenettes/eurorack)

Darin enthalten sind Firmware, Schematics und Code für:

| Modul | Dokumentation | Im Repo |
|-------|---------------|---------|
| **Plaits** | [mutable-instruments.net/modules/plaits](https://mutable-instruments.net/modules/plaits) | Ja |
| **Rings** | [mutable-instruments.net/modules/rings](https://mutable-instruments.net/modules/rings) | Ja |
| **Stages** | [mutable-instruments.net/modules/stages](https://mutable-instruments.net/modules/stages) | Ja |
| **Ripples** | [mutable-instruments.net/modules/ripples](https://mutable-instruments.net/modules/ripples) | Ja |
| **Veils** | [mutable-instruments.net/modules/veils](https://mutable-instruments.net/modules/veils) | Ja |

### Closed Source (keine öffentlichen Repos)

| Modul | Offizielle Seite |
|-------|------------------|
| **Pamela’s PRO Workout** | [busycircuits.com/alm034](https://busycircuits.com/pages/alm034) |
| **Metropolix** | [intellijel.com/shop/eurorack/metropolix](https://intellijel.com/shop/eurorack/metropolix/) |
| **Spectraphon** | [makenoisemusic.com/modules/spectraphon](https://www.makenoisemusic.com/modules/spectraphon/) |

---

**Zusammengefasst:**  
Nur die Mutable-Instrumente-Module (Plaits, Rings, Stages, Ripples, Veils) haben ein öffentliches GitHub-Repo. Alle anderen Module sind closed source und haben nur die Herstellerseiten.
**Module für Klangsynthese und Effekt** aus der empfohlenen Kombination:

### Klangsynthese (Sound-Erzeugung)

| Modul | Hersteller | Art | Beschreibung |
|-------|------------|-----|--------------|
| **Plaits** | Mutable Instruments | Macro-Oscillator | Vielseitigster Soundgenerator – 16 Algorithmen (VA, FM, Wavetable, Modal, Drums usw.) |
| **Rings** | Mutable Instruments | Resonator | Physikalische Modellierung – klingt wie Saiten, Membranen, Rohre |
| **Spectraphon** | Make Noise | Spectral Oscillator | Spektralanalyse + Resynthese – erzeugt komplexe, sich entwickelnde Klänge aus vorhandenem Audio |

### Effekt / Klangformung

| Modul | Hersteller | Art | Beschreibung |
|-------|------------|-----|--------------|
| **Ripples** | Mutable Instruments | Multimode-Filter | Klassischer analoger Filter (LP/BP/HP) mit Resonanz |
| **Veils** | Mutable Instruments | VCA | Steuert die Lautstärke / Dynamik der Klänge |

### Übersicht nach Funktion

```
Synthese:     Plaits  +  Rings  +  Spectraphon
                    ↓
Effekt:            Ripples (Filter)
                    ↓
Dynamik:           Veils (VCA)
```

**Plaits**, **Rings** und **Spectraphon** erzeugen den Klang.  
**Ripples** formt ihn (Filter), **Veils** steuert die Lautstärke.

Möchtest du zu einem dieser Module mehr Details oder eine visuelle Darstellung?**Module für Sequenzierung und Steuerung** aus der empfohlenen Kombination:

### Sequenzierung

| Modul | Hersteller | Funktion | Beschreibung |
|-------|------------|----------|--------------|
| **Metropolix** | Intellijel | Sequencer | Leistungsstarker Performance-Sequencer mit 2 Tracks, Pitch, Gate, Modulation Lanes und vielen Live-Controls |

### Steuerung / Timing / Modulation

| Modul | Hersteller | Funktion | Beschreibung |
|-------|------------|----------|--------------|
| **Pamela’s PRO Workout** | ALM Busy Circuits | Clock / BPM / Sync | Herzstück für Timing – BPM, Clock-Divisions, Euclidean, LFOs, Envelopes |
| **Stages** | Mutable Instruments | Modulation / Trigger-Logik | Flexibler Segment-Generator – kann als Envelope, LFO, Sequencer oder Trigger-Logik arbeiten |

### Übersicht nach Funktion

```
Timing:        Pamela’s PRO Workout (BPM + Clock + Sync)
                    ↓
Sequenz:       Metropolix (Pitch, Gate, Modulation)
                    ↓
Modulation:    Stages (Envelopes, LFOs, Trigger-Logik)
```

**Pamela’s** liefert den Takt und die Synchronisation.  
**Metropolix** erzeugt die musikalischen Sequenzen.  
**Stages** steuert und verbindet alles mit Modulation und Logik.

Möchtest du zu einem dieser Module mehr Details oder eine visuelle Darstellung?**Synchronisation** in einem modularen System bedeutet, dass verschiedene Module zeitlich aufeinander abgestimmt arbeiten – sie teilen sich denselben Takt.

### Grundprinzip

Ein Modul (meist ein Clock-Generator) erzeugt ein regelmäßiges Taktsignal. Alle anderen Module, die mit diesem Takt verbunden sind, folgen diesem Signal. Dadurch spielen Sequenzen, LFOs, Envelopes und andere Prozesse im gleichen Rhythmus.

### Im empfohlenen System

| Modul | Rolle bei der Synchronisation |
|-------|-------------------------------|
| **Pamela’s PRO Workout** | Erzeugt den Master-Clock (BPM). Gibt Clock-Signale an andere Module weiter und kann Divisions (z. B. /2, /4, x2) und Euclidean-Rhythmen erzeugen. |
| **Metropolix** | Empfängt den Clock von Pamela’s und spielt seine Sequenzen synchron dazu. Kann auch selbst Clock ausgeben. |
| **Stages** | Kann Clock empfangen und darauf basierend Envelopes, LFOs oder Trigger erzeugen, die ebenfalls synchron laufen. |

### Typische Sync-Verbindungen

```
Pamela’s (Clock Out)
    ↓
Metropolix (Clock In)     → Sequenz läuft im Takt
    ↓
Stages (Clock In)         → Modulation läuft im Takt
    ↓
andere Module (optional)  → weitere Elemente folgen dem gleichen Takt
```

### Wichtige Sync-Arten

- **Clock-Sync** – Module folgen einem gemeinsamen Taktsignal (häufigster Fall).
- **Reset** – Ein Signal setzt eine Sequenz oder einen LFO auf den Anfang zurück.
- **BPM-Sync** – Der Takt ist an eine feste Geschwindigkeit (Beats per Minute) gebunden.
- **Division / Multiplication** – Ein Modul läuft halb so schnell, doppelt so schnell usw. wie der Master-Clock.

### Im Zusammenhang mit deinem Flow-Konzept

Die Synchronisation ist der Mechanismus, der aus einzelnen Triggern einen zusammenhängenden Flow macht. Ohne Sync laufen die Module unabhängig voneinander. Mit Sync entsteht ein gemeinsamer Rhythmus, in dem Sequenzen, Modulationen und Effekte aufeinander reagieren.

Möchtest du, dass ich die Sync-Verbindungen für das konkrete System noch genauer (mit Patch-Beispielen) darstelle?**Sync-Verbindungen für das konkrete System**

### Master-Clock

**Pamela’s PRO Workout** ist der Master-Clock.

- Sie erzeugt den BPM-Takt.
- Alle anderen Module folgen diesem Takt.

### Wichtige Verbindungen

```
Pamela’s PRO Workout
├── Clock Out  →  Metropolix (Clock In)
├── Clock Out  →  Stages (Clock / Trigger In)
└── Optional: weitere Clock-Ausgänge an andere Module
```

### Modul für Modul

**1. Pamela’s → Metropolix**
- Pamela’s Clock Out → Metropolix Clock In  
- Metropolix spielt seine Sequenzen synchron zum BPM von Pamela’s.
- Optional: Metropolix Reset ← Pamela’s (wenn du Sequenzen gemeinsam zurücksetzen willst).

**2. Pamela’s → Stages**
- Pamela’s Clock Out → Stages Clock/Trigger In  
- Stages erzeugt Envelopes, LFOs oder Trigger, die im gleichen Takt laufen.
- Stages kann die Modulation an Plaits, Rings oder Spectraphon weitergeben.

**3. Metropolix → Sound-Module**
- Metropolix Pitch Out → Plaits / Rings / Spectraphon (1V/Oct)
- Metropolix Gate Out → Stages oder direkt an VCAs / Envelopes
- Dadurch folgen die Klänge der Sequenz und dem gemeinsamen Takt.

**4. Stages → Sound-Module (Modulation)**
- Stages Ausgänge → CV-Inputs von Plaits, Rings, Spectraphon oder Ripples  
- Die Modulation bleibt synchron zum Clock.

### Typisches Patch-Beispiel

```
Pamela’s Clock Out
    ├── Metropolix Clock In
    └── Stages Clock In

Metropolix Pitch Out → Plaits 1V/Oct
Metropolix Gate Out  → Stages Trigger In

Stages Envelope Out → Veils (VCA)
Stages LFO Out      → Ripples (Filter Cutoff)
```

### Ergebnis

- Alles läuft im gleichen BPM.
- Sequenzen, Modulationen und Filterbewegungen sind zeitlich aufeinander abgestimmt.
- Änderungen am BPM von Pamela’s wirken sich sofort auf das ganze System aus.

Möchtest du, dass ich daraus ein konkretes Patch-Diagramm oder eine visuelle Darstellung mache?
