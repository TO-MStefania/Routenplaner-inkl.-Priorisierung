# Hotel Route Planner

Webapp für **priorisierte Außendienst-Routenplanung im Hotelvertrieb**. Lade eine CSV mit deinen Hotelkunden hoch — die App berechnet einen Prioritätsscore, zeigt die Kunden auf einer Karte und schlägt eine sinnvolle Besuchsreihenfolge vor.

> Die Routenoptimierung erfolgt **algorithmisch und deterministisch**, nicht über KI. Wirtschaftlicher Nutzen wird höher gewichtet als reine Distanz.

---

## Features

- 📤 CSV-Upload (Drag & Drop oder File-Picker)
- 📊 Prioritätsscore aus sechs gewichteten Faktoren
- 🎛️ Gewichte live im UI verstellbar
- 🗺️ Interaktive Karte mit nummerierter Route (Leaflet + OpenStreetMap)
- 📋 Sortierbare Hoteltabelle mit Score-Breakdown
- 📥 Route als CSV exportieren
- 🔌 Modulares Routing-Interface (späterer Austausch gegen OSRM/Google möglich)

---

## Quickstart

**Voraussetzungen:** Node.js ≥ 18, pnpm (oder npm/yarn).

```bash
# Repo klonen
git clone https://github.com/TO-MStefania/Routenplaner-inkl.-Priorisierung.git
cd Routenplaner-inkl.-Priorisierung

# Abhängigkeiten installieren
pnpm install

# Dev-Server starten
pnpm dev
```

Die App läuft dann auf [http://localhost:3000](http://localhost:3000).

**Produktion:**

```bash
pnpm build
pnpm start
```

---

## Nutzung

1. Auf der Startseite **CSV hochladen** oder „Beispiel-CSV laden" klicken.
2. Auf `/plan`:
   - **Tabelle** zeigt alle Hotels mit Score, Status und Distanz.
   - **Gewichte-Panel** links erlaubt das Justieren der Scoring-Faktoren.
   - **Karte** rechts zeigt Marker (Farbe = Status, Größe = Score) und die nummerierte Route.
3. **„Route exportieren"** speichert die berechnete Reihenfolge als CSV.

---

## CSV-Format

Die Eingabe-CSV erwartet folgende Spalten (UTF-8, Komma-getrennt, Header in Zeile 1):

| Spalte | Typ | Pflicht | Beispiel |
|---|---|---|---|
| `id` | string | ✅ | `H001` |
| `name` | string | ✅ | `Hotel Adler` |
| `address` | string | – | `Hauptstr. 1, 70173 Stuttgart` |
| `lat` | number | ✅ | `48.7758` |
| `lng` | number | ✅ | `9.1829` |
| `revenue_potential` | number (EUR/Jahr) | ✅ | `45000` |
| `close_probability` | number 0–1 | ✅ | `0.6` |
| `status` | enum | ✅ | `prospect` |
| `strategic_value` | int 1–5 | ✅ | `4` |
| `window_start` | `HH:MM` | – | `09:00` |
| `window_end` | `HH:MM` | – | `17:00` |
| `visit_duration_min` | int | – | `45` |
| `notes` | string | – | `Vorab anrufen` |

**Erlaubte Status-Werte:** `key_account`, `active`, `prospect`, `lead`, `inactive`.

Eine funktionierende Beispieldatei liegt unter [`sample_hotels.csv`](./sample_hotels.csv).

---

## Scoring-Modell

Der Prioritätsscore eines Hotels ist die gewichtete Summe sechs normalisierter Faktoren:

```
score = w_revenue   · norm(revenue_potential)
      + w_prob      · close_probability
      + w_status    · statusWeight(status)
      + w_strategic · (strategic_value / 5)
      + w_proximity · (1 − norm(distance_from_start))
      + w_window    · windowFit(window_start, window_end)
```

**Default-Gewichte** (Summe = 1.0):

| Faktor | Gewicht |
|---|---|
| Umsatzpotenzial | 0.30 |
| Abschlusswahrscheinlichkeit | 0.20 |
| Kundenstatus | 0.15 |
| Strategische Bedeutung | 0.10 |
| Geografische Nähe | 0.15 |
| Zeitfenster-Passung | 0.10 |

Die Gewichte sind im UI verstellbar — Änderungen rechnen den Score sofort neu.

### Routen-Algorithmus

*Score-First Greedy mit Distanz-Penalty.* Ab dem Startpunkt wird wiederholt das Hotel gewählt, das `score − α · norm(distance)` maximiert (α=0.3). Score dominiert, aber der Weg wird mitbeachtet. Deterministisch, O(n²), reicht für ≤ 200 Stops.

---

## Tech-Stack

- [Next.js](https://nextjs.org/) 14+ (App Router)
- TypeScript (strict)
- [Tailwind CSS](https://tailwindcss.com/)
- [Leaflet](https://leafletjs.com/) + [react-leaflet](https://react-leaflet.js.org/) + OpenStreetMap
- [PapaParse](https://www.papaparse.com/) für CSV
- [Vitest](https://vitest.dev/) für Unit-Tests (optional)

---

## Projektstruktur

```
app/          # Next.js App Router (Pages, Layout)
components/   # UI-Komponenten (Tabelle, Karte, Upload …)
lib/          # Reine Logik (CSV, Scoring, Distanz, Routing-Service)
types/        # Geteilte TypeScript-Typen
public/       # Statische Assets inkl. sample_hotels.csv
```

Details siehe [`CLAUDE.md`](./CLAUDE.md).

---

## Roadmap

- [ ] Geocoding-Service (Adresse → Koordinaten)
- [ ] Echtes Straßenrouting via OSRM / OpenRouteService
- [ ] Zeitfenster als harter Constraint
- [ ] HubSpot-Connector als Datenquelle
- [ ] Persistenz + Multi-User
- [ ] Mehrtagesplanung mit Tagesstart/-ende

---

## Lizenz

Noch festzulegen.
