# Iso-Stadt — Projektdokumentation

Ein isometrisches Aufbauspiel im Browser, entstanden aus der Frage nach einem
kostenlosen Weg, so etwas wie *Forge of Empires* zu bauen.

**Ergebnis:** eine einzelne HTML-Datei, 100 KB, 2.385 Zeilen. Kein Build,
kein `npm install`, keine Bilddateien, kein Server. Doppelklick genügt.

---

## Inhalt

1. [Warum eine einzige Datei](#1-warum-eine-einzige-datei)
2. [Wie die Gebäude entstehen](#2-wie-die-gebäude-entstehen)
3. [Die Spielregeln und warum sie so sind](#3-die-spielregeln-und-warum-sie-so-sind)
4. [Alle Zahlen](#4-alle-zahlen)
5. [Kamera und Bedienung](#5-kamera-und-bedienung)
6. [Animation](#6-animation)
7. [Speichern](#7-speichern)
8. [Die Hilfeseite](#8-die-hilfeseite)
9. [Wie geprüft wurde](#9-wie-geprüft-wurde)
10. [Gefundene Fehler](#10-gefundene-fehler)
11. [Was fehlt](#11-was-fehlt)

---

## 1. Warum eine einzige Datei

Die Ausgangsfrage war, welches Framework sich eignet. Die Antwort lautete:
für den Anfang keines. Ein Aufbauspiel besteht zu etwa neunzig Prozent aus
Zustandsverwaltung und Oberfläche — Gebäudelisten, Zeitgeber, Ressourcen —
und nur zu zehn Prozent aus Grafik. Was fehlte, war also keine Engine,
sondern eine saubere Datenstruktur.

Der zweite Grund ist der Umfang. Vierzehn Gebäudevarianten in sechs Stufen
und drei Zuständen wären als fertige Bilder in dreifacher Auflösung mehrere
Megabyte. Als Zeichenanweisungen sind sie 100 KB — dieselbe Größenordnung
wie ein einziges mittelgroßes PNG.

Der dritte Grund zeigte sich erst später: Eine Formel lässt sich verändern,
ein Bild nicht. Als die Arkologie eine sich verjüngende Wand brauchte, war
das eine ausgetauschte Funktion. Als fertige Grafik wäre es ein neuer
Zeichenauftrag gewesen.

**Der Preis:** Alles muss aus Rechteck, Kreis und Linie zusammengesetzt
werden. Es gibt keine Texturen, keine weichen Schatten, keine Vegetation
außer der, die man selbst konstruiert.

**Wenn mehr Detail nötig wird**, ist der nächste Schritt kein weiterer Pfad,
sondern ein Verfahrenswechsel: in Blender modellieren, mit orthografischer
Kamera aus 30° rendern, als PNG exportieren. So macht es Forge of Empires
auch. Man bekommt Texturen und weiche Schatten für einen Bruchteil des
Aufwands — bezahlt aber mit einer Asset-Pipeline und Ladezeiten.

### Externe Abhängigkeiten

Genau eine: Google Fonts für zwei Schriftarten. Ohne Netz startet die Datei
trotzdem und funktioniert vollständig; sie sieht nur mit den Systemschriften
etwas anders aus. Wer sie völlig unabhängig will, entfernt die drei
`<link>`-Zeilen im Kopf.

---

## 2. Wie die Gebäude entstehen

### 2.1 Eine einzige Projektionsformel

```js
iso(gx, gy) = { x: (gx-gy) * 34, y: (gx+gy) * 17 }
```

Ein Feld nach rechts bedeutet 34 Pixel rechts und 17 Pixel runter. Daraus
ergeben sich die vier Eckpunkte jeder Bodenraute, und daraus alles Weitere.
Es wird nirgends mit Winkeln oder Sinus gerechnet.

### 2.2 Kein Pixel von Hand

Jede Koordinate ist ein Vielfaches von `bw`, `bh` oder `h`. Ein Gebäude ist
vollständig beschrieben durch sieben Werte:

```js
{ h:46, roofH:17, floors:3, style:'stone',
  wall:'#E6DCC6', roof:'#7C6F80', trim:'#8E836E' }
```

Ändert man `h`, wandern Fenster, Gesimse und Dach automatisch mit.

### 2.3 Eine Farbe, drei Helligkeiten

Das Licht kommt fest von links oben. Linke Wand −4, rechte Wand −42,
Deckfläche +10. Die Funktion `shade()` addiert auf alle drei Farbkanäle.

Deshalb braucht ein Gebäude drei Grundfarben statt zwölf, und deshalb wirkt
die Stadt einheitlich beleuchtet, obwohl jedes Haus anders gefärbt ist. Der
Zustand „kein Anschluss" ist bloß ein weiteres `dim = −40` auf denselben
Ausdruck — kein zweiter Farbsatz.

### 2.4 Die Flächenkoordinate — der eigentliche Kniff

```js
faceL = (a,v) => [L[0] + (B[0]-L[0])*a,
                  L[1] + (B[1]-L[1])*a - h*v]
```

`a` läuft von 0 bis 1 an der Wand entlang, `v` von 0 (Boden) bis 1 (Traufe).
Ein Fenster ist damit kein Vierpunkt-Polygon mehr, sondern:

```js
band(face, 0.28, 0.42, 0.46, 0.58)
```

Das ist der Grund, warum Fachwerk, Sprossen, Fensterläden, Warnstreifen und
Ziegelschichten überhaupt handhabbar sind.

Es zahlt sich doppelt aus. Als die Arkologie eine verjüngte Wand brauchte,
musste nur `face()` ausgetauscht werden:

```js
const k = 1 - v + v*tf;                    // Maßstab auf Höhe v
return [bp[0]*k, bp[1]*k - h*v];
```

Alle Fensterbänder und Terrassen blieben unverändert und folgen seitdem der
Schräge. Ein Trapez statt eines Parallelogramms, ohne eine Zeile
Fassadencode anzufassen.

### 2.5 Immer dieselbe Malreihenfolge

Schatten → Wände → Sockel → Fassade → Fenster → Dach → Aufbauten. Über die
Karte hinweg nach `x+y` sortiert, damit Hinteres zuerst kommt. Bricht man
das, scheinen sofort Kanten durch.

### 2.6 Neun Stile, ein Rumpf

`rustic`, `timber`, `stone`, `brick`, `plant`, `modern`, `glass`, `arc`,
`fusion`. Das sind keine neun Programme, sondern Verzweigungen an drei
Stellen: Fassade, Fenster, Dach. Der Rumpf mit Wänden, Schatten und
Flächenkoordinate ist bei sieben von neun identisch. Nur `arc` und `fusion`
steigen früh aus und bringen eigene Geometrie mit.

### 2.7 Der Zwischenspeicher

Ein Haus sind rund 200 Pfade. Bei voller Karte und 60 Bildern pro Sekunde
wären das Millionen Pfadoperationen pro Sekunde. Deshalb wird jede
Kombination aus Typ, Stufe und Zustand **einmal** in ein Offscreen-Canvas
gerendert und danach nur noch kopiert.

Der Rahmen wächst mit dem Gebäude — die Hütte bekommt 150×144 Pixel, die
Arkologie 150×192 —, denn ein fester Rahmen hätte die hohen Bauten oben
abgeschnitten. Die Sprites werden dreifach überabgetastet, damit sie beim
Hineinzoomen scharf bleiben.

### 2.8 Selbstauferkegte Regeln

Da das Ergebnis während der Entwicklung nicht durchgehend sichtbar war,
mussten harte Vorgaben her, die sich messen lassen:

- unter 68 Pixel Breite bleiben (Kachelbreite), sonst ragt ein Gebäude ins Nachbarfeld
- jede Stufe sichtbar höher als die vorige
- alle drei Zustände dieselbe Geometrie
- Silhouette noch bei 40 Pixel lesbar

Genau diese Punkte werden programmatisch nachgeprüft.

---

## 3. Die Spielregeln und warum sie so sind

### 3.1 Straßen

Ein Gebäude produziert nur, wenn eines der vier Nachbarfelder eine Straße
ist, **die lückenlos zum Rathaus führt**. Ermittelt wird das mit einer
Breitensuche vom Rathaus aus; abgeschnittene Straßenstücke werden rot
gestrichelt markiert und zählen nicht.

Das macht aus dem Raster ein Planungsproblem: Fläche muss zwischen
Produktion und Erschließung aufgeteilt werden.

### 3.2 Einwohner als Währung

Wohngebäude liefern Einwohner, Werkstätten verbrauchen sie. Ohne freie
Einwohner lässt sich weder bauen noch ausbauen.

Besetzt wird **nach Baualter**: die älteste Werkstatt behält ihre Leute.
Sonst würde bei jedem Abriss zufällig eine andere ausfallen. Deshalb bleibt
das Baualter auch beim Verschieben erhalten.

### 3.3 Zeitgeber: zwei verschiedene Verfahren

Das ist eine bewusste Ungleichbehandlung.

**Im laufenden Spiel** wird der *Zeitpunkt der Fertigstellung* gespeichert.
Würde man „noch 3:42" ablegen, hielte die Produktion an, sobald der Tab in
den Hintergrund rutscht — und jeder könnte die Systemuhr vorstellen.

**Im Spielstand** wird die *verbleibende Dauer* abgelegt. Sonst wäre nach
einer Woche Pause alles auf einen Schlag fertig.

### 3.4 Stillstand friert ein

Fällt die Straße oder das Personal weg, wird die Restzeit eingefroren, nicht
zurückgesetzt. Nach der Reparatur läuft die Produktion dort weiter, wo sie
stehengeblieben ist.

Andernfalls ließe sich der Straßenabriss als Pausenknopf missbrauchen — und
umgekehrt würden Spieler bestraft, deren Netz nachts kaputtgeht.

Zwei Störungsarten, zwei Zeichen: rotes Ausrufezeichen für „kein Weg zum
Rathaus", blaue Figur für „zu wenig Einwohner".

### 3.5 Verstärker

Brunnen und Bäume produzieren nichts, erhöhen aber den Ertrag der vier
angrenzenden Felder. Mehrere addieren sich. Sie brauchen **weder Straße noch
Einwohner** — und das ist der eigentliche Hebel des Spiels.

Ein Fusionswerk mit drei großen Bäumen liefert das Vierfache für dieselben
210 Einwohner. Vier einzelne Fusionswerke bräuchten 840 Einwohner und damit
vier Arkologien als Wohnraum. Auf die Fläche gerechnet: 1.275 gegen 1.410
Münzen pro Minute und Feld, bei 5 statt 8 belegten Feldern.

Der Boden färbt sich in vier Stufen wärmer, je stärker ein Feld verstärkt
wird.

### 3.6 Bauabschnitte

Die Karte besteht aus Abschnitten zu 4×4 Feldern. Start sind vier davon
(das bekannte 8×8-Feld). Ist **jedes Feld bebaut**, leuchten die
angrenzenden Abschnitte gestrichelt auf und lassen sich kaufen: der erste
für 10.000 Münzen, jeder weitere 50 % teurer. Insgesamt sind 36 Abschnitte
erreichbar, also bis zu 24×24 Felder.

Die Bedingung „alles bebaut" verhindert, dass man sich aus Platzmangel
freikauft, statt zu planen.

### 3.7 Das Rathaus wächst mit

Grundeinnahme 14, plus 1,6 Münzen je Einwohner der Stadt. Bei 5 Einwohnern
sind das 22 Münzen pro Zyklus, bei 429 Einwohnern 700. Ohne diese Kopplung
wäre das Rathaus nach einer Stunde bedeutungslos.

### 3.8 Ausbau: mehr, aber langsamer

Jede Stufe bringt mehr pro Zyklus, verlängert aber den Takt. Der Ertrag pro
Minute steigt trotzdem, nur immer flacher. Das ergibt ein anderes
Spielgefühl als bloß „mehr": frühe Häuser will man ständig abklicken, späte
laufen im Hintergrund.

### 3.9 Abriss und Verschieben

Abriss erstattet die Hälfte der **gesamten** Investition inklusive
Ausbaustufen. Verschieben ist kostenlos und erhält Stufe, Restzeit,
Investition und Baualter — es wandert dasselbe Objekt auf ein anderes Feld.

Ohne das Verschieben wäre eine Fehlplanung bei den Verstärkern nur durch
Abriss zu korrigieren, mit halbem Verlust. Genau dort stecken aber die
interessanten Entscheidungen.

---

## 4. Alle Zahlen

### Wohngebäude — Bau 50 Münzen

| Stufe | Name | Stil | Höhe | Ertrag | Takt | pro Min. | Einwohner | Ausbau |
|---|---|---|---:|---:|---:|---:|---:|---:|
| 1 | Hütte | rustic | 19 | 8 | 8 s | 60 | 2 | 170 |
| 2 | Fachwerkhaus | timber | 31 | 30 | 14 s | 129 | 6 | 520 |
| 3 | Stadthaus | stone | 46 | 95 | 26 s | 219 | 14 | 1.600 |
| 4 | Wohnblock | modern | 58 | 340 | 60 s | 340 | 34 | 5.200 |
| 5 | Hochhaus | glass | 68 | 1.150 | 110 s | 627 | 82 | 16.000 |
| 6 | Arkologie | arc | 80 | 4.200 | 240 s | 1.050 | 210 | — |

### Werkstätten — Bau 320 Münzen

| Stufe | Name | Stil | Höhe | Ertrag | Takt | pro Min. | braucht | Ausbau |
|---|---|---|---:|---:|---:|---:|---:|---:|
| 1 | Werkstatt | timber | 23 | 120 | 30 s | 240 | 4 | 820 |
| 2 | Manufaktur | stone | 34 | 520 | 70 s | 446 | 14 | 2.400 |
| 3 | Fabrik | brick | 56 | 1.560 | 180 s | 520 | 40 | 9.000 |
| 4 | Montagewerk | plant | 66 | 4.800 | 300 s | 960 | 95 | 30.000 |
| 5 | Fusionswerk | fusion | 76 | 15.000 | 600 s | 1.500 | 210 | — |

### Übrige

| Gebäude | Kosten | Ertrag | Verstärkung | Einwohner |
|---|---:|---|---:|---:|
| Rathaus | fest | 14 / 20 s + Steuern | — | 5 |
| Brunnen | 240 | — | +30 % | — |
| Kleiner Baum | 800 | — | +60 % | — |
| Großer Baum | 2.500 | — | +100 % | — |
| Straße | 10 | — | — | — |

### Epochenbilder

Die Stile bilden eine bewusste Zeitreise:

- **rustic** — Holzbretter, Reetdach, Eckpfosten
- **timber** — Fachwerk mit Schwelle, Rähm, Ständern, Streben; Fensterläden
- **stone** — Putzfassade, Eckquader, Gurt- und Traufgesims, Gaube
- **brick** — Lisenen, Sprossenfenster, Zahnschnitt, Sheddach, Ziegelschlot
- **plant** — Trapezblech, Rolltor, Warnstreifen, Lüfter, Silo
- **modern** — Sichtbeton, Fensterbänder, Balkone, Attika
- **glass** — Vorhangfassade, panelweise beleuchtet, Staffelgeschoss
- **arc** — verjüngter Turm, begrünte Terrassen, Feldring, schwebende Kugel
- **fusion** — Reaktorkuppel, Meridianrippen, Kühltürme, Leuchtbänder

---

## 5. Kamera und Bedienung

### Das Kameramodell

Anfangs rechnete alles direkt in Bildschirmpixeln mit einem festen
Einpassungsfaktor. Für Zoom und Verschieben musste das auf Weltkoordinaten
umgestellt werden. Die Kamera besteht seitdem aus drei Zahlen: Blickpunkt
`x`/`y` und Zoom `z`.

Zoomen auf einen Punkt folgt dem üblichen Verfahren: Weltpunkt unter dem
Zeiger merken, Zoom ändern, Kamera um die Differenz nachziehen. Gemessene
Abweichung: 0,0000 Pixel.

### Gesten

| Eingabe | Wirkung |
|---|---|
| Tippen | Bauen, ernten, ausbauen, abreißen — je nach Werkzeug |
| **Lang halten (380 ms)** | Gebäude aufnehmen und ziehen — mit jedem Werkzeug |
| Ziehen | Karte verschieben; mit Straßenwerkzeug pflastern |
| Zwei Finger | Zoomen und verschieben zugleich |
| Mausrad | Zoomt auf den Zeiger |
| Umschalt + Ziehen | Verschiebt immer |
| `+` `−` `0` | Näher, weiter, ganze Stadt |
| Pfeiltasten | Karte verschieben |
| `?` oder `h` | Hilfe |
| `Escape` | Hilfe schließen, Getragenes ablegen |

Die Abgrenzung war der eigentliche Aufwand. Bewegt sich der Finger vor
Ablauf der Haltezeit um mehr als 8 Pixel, wird der Timer abgebrochen und
daraus ein Kartenschieben — man kann also über Gebäude wischen, ohne sie
mitzunehmen. Ein zweiter Finger bricht ebenfalls ab, damit Zoomen nicht in
einem Umzug endet.

---

## 6. Animation

Die Regel lautet: **wenig bewegen, und das Bewegte billig halten.** Was im
Sprite liegt, ist eingefroren; alles außerhalb ist frei.

### Wind — der sparsamste Effekt

```js
wind = (0.62 + 0.38*Math.sin(now()/4300)) * 0.05;   // einmal pro Bild
ctx.transform(1, 0, swayOf(typ, phase), 1, 0, 0);   // je Baum
```

Eine **Scherung** statt einer Drehung: Der Versatz wächst mit dem Abstand
zum Boden, also stehen die Wurzeln still und die Krone wiegt sich. Der große
Baum schwingt automatisch weiter aus als der kleine, ohne dass das irgendwo
steht.

Kosten: eine geänderte Matrix vor dem ohnehin nötigen `drawImage`. Kein
zusätzlicher Zeichenaufruf, kein Sprite, kein Byte. Zwei überlagerte
Schwingungen und ein Phasenversatz je Feld verhindern Gleichtakt.

Bemerkenswert: Mit fertigen PNG-Sprites wäre dieser Effekt *schwieriger* —
dort müsste jemand acht Windphasen zeichnen.

### Rauch

Drei Wolken auf aufsteigender Bahn, live gezeichnet. Die Austrittsstelle
wird nicht geschätzt, sondern von der Zeichenroutine beim Rendern gemeldet
und am Sprite abgelegt. Gebäude ohne Schornstein rauchen nicht.

### Weitere

Pulsierendes Leuchten bei Erntereife, Staubwolken beim Bauen, aufsteigende
Ertragszahlen, Aufwachsen nach dem Bau, Gras im selben Wind.

**Alle Animationen respektieren `prefers-reduced-motion`** — bei aktivierter
Einstellung stehen drei Bilder über 3,4 Sekunden byteweise identisch.

---

## 7. Speichern

Die Datei läuft in sehr verschiedenen Umgebungen, deshalb wird kein
bestimmter Speicher vorausgesetzt, sondern der beste verfügbare gesucht:

1. Artefaktspeicher (`window.storage`)
2. Browserspeicher (`localStorage`)
3. keiner — dann bleibt der Weg über Datei oder Text

In allen drei Fällen funktioniert das Spiel; nur der Hinweis in der Hilfe
ändert sich. Gesichert wird 1,2 Sekunden nach jeder Änderung und sofort beim
Verlassen der Seite. Ein voller Spielstand mit acht Gebäuden umfasst
258 Zeichen.

Beschädigte Spielstände werden einzeln übergangen statt das Laden
abzubrechen: unbekannte Gebäudetypen, Felder außerhalb der gekauften Fläche,
kaputtes JSON. Ein fehlendes Rathaus wird wieder gesetzt.

Zusätzlich: „Sichern" und „Laden" oben links, sowie in der Hilfe der Weg
über Sicherungstext für Umgebungen, in denen Downloads gesperrt sind.

---

## 8. Die Hilfeseite

Erreichbar über `?` rechts unten oder die Tasten `?` und `h`.

Der entscheidende Punkt: **Die Gebäudebilder sind keine Kopien.** Sie blitten
dasselbe zwischengespeicherte Sprite wie das Spielfeld, und alle Zahlen
stammen direkt aus `TYPES`. Die Übersicht kann also nicht von der
Wirklichkeit abweichen — ändert man morgen eine Fassade, ändert sich die
Hilfe mit.

Die Karten einer Reihe teilen sich **einen Maßstab und eine Standlinie**,
sonst wäre der Größenzuwachs nicht ablesbar. Sie hängen an derselben
Bildschleife wie das Spiel und zeigen dieselben Bewegungen; ist die Hilfe
geschlossen, wird nichts gezeichnet.

Inhalt: alle Ausbauketten mit Nummern und Kennzahlen, Straßenregel, beide
Stillstandsarten, Verstärker, Bauabschnitte, Werkzeuge, Steuerung,
Kopfzeile, Speichern.

---

## 9. Wie geprüft wurde

Da das Ergebnis während der Entwicklung nicht durchgehend sichtbar war,
entstand parallel eine Prüfsammlung von rund 1.800 Zeilen in vierzehn
Dateien. Sie lädt das Skript aus der HTML-Datei in eine nachgebaute
Browserumgebung und misst tatsächlich gezeichnete Pixel.

| Datei | prüft |
|---|---|
| `verify.js` | jedes Sprite: Deckung, Randabstand, Höhenstaffelung, Balance |
| `check.js` | Kameramathematik über alle Zoomstufen |
| `sim.js` | Einwohnersystem, Bauabschnitte, Kauf |
| `trees.js` | Verstärker, Addition, Wirkung |
| `move.js` | Verschieben: Erhalt aller Eigenschaften |
| `hold.js` | Langdruck über echte Zeigerereignisse |
| `save.js` | Speichern in drei Umgebungen, beschädigte Stände |
| `wind.js` | Bewegungsausschlag, reduzierte Bewegung |
| `order.js` | Reihenfolge und Maßstab der Hilfeseite |
| `helpanim.js` | Animation der Hilfe, Rauchposition |
| `css.js` | Selektoren, die zu weit greifen |
| `ui.js` | Überlappungen der Bedienelemente |
| `taptest.js`, `final.js` | Werkzeuge, Rathaussteuern, Hilfeinhalt |

Beispiele für Zusicherungen: „alle 64 Kacheln werden bei Zoom 0,4 / 1 / 1,7
/ 3 korrekt getroffen", „der Ankerpunkt driftet um weniger als 0,01 Pixel",
„die Rauchquelle liegt auf Mauerwerk", „alle Gebäude einer Reihe stehen auf
einer Linie".

---

## 10. Gefundene Fehler

Diese Liste ist Teil der Dokumentation, weil sie zeigt, welche Fehlerarten
in einem solchen Projekt tatsächlich auftreten.

**Der schwerwiegendste — eine CSS-Regel, die zu weit griff.** Die Zeile
`canvas{position:fixed; inset:0}` galt für *jedes* Canvas im Dokument. Als
die Hilfeseite dazukam, wurden ihre elf Bilder alle an die linke obere Ecke
des Fensters geheftet und lagen übereinander. Behoben durch `#c{…}`. Die
Prüfungen hatten das nicht gefunden, weil sie kein CSS anwenden — seitdem
gibt es `css.js`.

**Zwei konkurrierende Windumsetzungen.** Zeitweise steckten zwei Verfahren
gleichzeitig in der Datei: die Scherung und eine zweite, die den Baum in
Stamm und Krone teilte. Zwei Sprites, zwei Zeichenaufrufe pro Baum. Damit
war eine frühere Aussage über die Kosten des Effekts für die ausgelieferte
Fassung falsch. Entfernt.

**Der Rauch kam nie aus dem Schornstein.** Die Position wurde über eine
Faustformel geschätzt und lag 9 bis 19 Pixel zu tief — bei der Fabrik quoll
er mitten aus dem Dach. Jetzt meldet die Zeichenroutine die echte Stelle.

**Eine Wettlaufsituation beim Start.** `bootSave()` läuft asynchron; ein
Klick in den ersten Millisekunden wäre vom geladenen Stand überschrieben
worden. Jetzt lädt er nur, wenn die Stadt unberührt ist.

**Eine Sackgasse beim Tragen.** Das Absetzen per Tipp war nur im
Verschieben-Werkzeug verdrahtet. Hielt man mit einem anderen Werkzeug lang
und ließ an Ort und Stelle los, kam man nur noch über Escape heraus. Jetzt
setzt jeder Tipp ab — das hat den Code sogar vereinfacht.

**`build()` konnte überbauen.** Über die Oberfläche nie erreichbar, weil
`tap()` vorher prüft. Aber die Sperre gehörte in die Funktion selbst.

**Ein Balance-Fehler, der den halben Spielinhalt entwertete.** Eine Fabrik
samt nötigem Wohnraum brachte auf drei Feldern 1.030 Münzen pro Minute —
drei Wohnblocks auf derselben Fläche 1.020. Ein Prozent Vorsprung für den
gesamten Industriezweig. Ursache: der Ertrag je gebundenem Einwohner fiel
von 60 über 22 auf 8,8. Jede Ausbaustufe machte den Arbeiter
unwirtschaftlicher. Nach der Korrektur liegt der Zweig 18 bis 46 Prozent
vorn.

**Geometriefehler**, die nur durch Nachmessen auffielen: Der Schornstein
schwebte neben dem Dach; die Giebelspitze saß am Dachüberstand statt auf der
Wandflucht; die Fabrik nutzte den Glas-Stil und sah aus wie ein Wohnhaus;
der Feldring des Fusionswerks ragte mit 69 Pixeln über die Kachelbreite von
68; die Arkologie war niedriger als das Hochhaus, die letzte Stufe wirkte
also kleiner als die vorletzte.

**Die Überabtastung stand fest auf 2×**, weil `const SS = Math.min(3, DPR+1)`
schon beim Einlesen ausgewertet wurde — da war `DPR` noch der Startwert 1.

**Und mehrere Fehler in den Prüfungen selbst**: ein Ausdruck, der in
`font-size:11.5px` die „5" traf; ein Test, der drei Bäume mit
gegenläufigen Phasen mittelte und deshalb keine Bewegung sah; ein
Größenvergleich zwischen Wandhöhe und Gesamtsilhouette. Falsche Alarme
kosten Zeit, aber ein Test, der nie anschlägt, ist wertloser.

---

## 11. Was fehlt

**Kein Server.** Zeitgeber laufen im Browser. Für ein echtes Spiel müssten
sie serverseitig nachgerechnet werden, sonst genügt ein Blick in die
Entwicklerwerkzeuge.

**Keine Mehrfeld-Gebäude.** Alles belegt genau ein Feld. Ein 2×2-Rathaus
bräuchte Belegungsprüfung, Ankerpunkt und angepasste Malreihenfolge.

**Keine Forschung, keine Ereignisse, kein Militär** — also die halbe
Substanz von Forge of Empires.

**Kein Ton.**

**Die Karte endet bei 36 Abschnitten.** Für mehr müsste das Zeichnen auf den
sichtbaren Ausschnitt beschränkt werden; derzeit läuft die Schleife über
alle freigeschalteten Felder.

---

## Anhang: Aufbau der Datei

| Bereich | Zeilen | Inhalt |
|---|---:|---|
| CSS | 140 | Oberfläche, Hilfeseite, Bedienelemente |
| Spieldaten | ~90 | `TYPES` — alle Gebäude und Stufen |
| Straßennetz | ~25 | Breitensuche vom Rathaus |
| Spiellogik | ~130 | Bauen, Ernten, Ausbauen, Verschieben, Abreißen |
| Kamera | ~60 | Weltkoordinaten, Zoom, Grenzen |
| Zeichnen | ~750 | Sprites, neun Stile, Boden, Effekte |
| Eingabe | ~150 | Zeiger, Gesten, Tastatur |
| Speichern | ~150 | Erkennung, Serialisierung, Dateiweg |
| Hilfeseite | ~200 | Aufbau aus den Spieldaten |
| Oberfläche | ~120 | Baumenü, Kopfzeile, Statuszeile |
