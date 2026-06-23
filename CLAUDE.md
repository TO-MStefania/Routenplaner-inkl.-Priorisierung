# Projekt: Hotel Route Planner

Webapp für **priorisierte Außendienst-Routenplanung im Hotelvertrieb**.

> **Wichtig:** Claude soll **nicht** selbst als KI die Route optimieren.
> Die App arbeitet **algorithmisch und deterministisch**. Gleiche Eingabe → gleiche Ausgabe.

---

## 1. Ziel

Außendienstler im Hotelvertrieb laden eine CSV-Kundenliste hoch. Die App:

1. parst die CSV,
2. berechnet pro Kunde einen **Prioritätsscore**,
3. zeigt die Kunden in einer Tabelle und auf einer Karte,
4. erzeugt eine **sinnvolle Besuchsreihenfolge** (wirtschaftlicher Nutzen vor reiner Distanz),
5. ermöglicht den **Export der geplanten Route** als CSV.

## 2. Nicht-Ziele (Out of Scope im MVP)

- Keine KI-/LLM-basierte Routenoptimierung
- Kein Geocoding (Adresse → Lat/Lng) — Koordinaten werden aus der CSV erwartet
- Keine echte Straßennetz-Routenführung (nur Luftlinie + modulares Interface für späteren Austausch)
- Keine HubSpot- oder sonstige CRM-Integration
- Kein Login, keine Multi-User-Verwaltung
- Keine Persistenz (kein Backend, kein DB-Store) — Daten leben im Browser-State

---

## 3. Tech-Stack

| Bereich | Wahl |
|---|---|
| Framework | Next.js 14+ (App Router) |
| Sprache | TypeScript (strict) |
| Styling | Tailwind CSS |
| Karten | Leaflet + react-leaflet + OpenStreetMap Tiles (kein API-Key nötig) |
| CSV-Parser | PapaParse |
| Package Manager | pnpm bevorzugt (npm/yarn ok) |
| Tests (optional) | Vitest für `lib/`-Module |

Keine Server-Komponenten für State-haltige Teile — `/plan` ist eine Client-Page.

---

## 4. Datenmodell

### 4.1 CSV-Schema (Input)

Spalten (Header in Zeile 1, Komma-getrennt, UTF-8):

| Spalte | Typ | Pflicht | Beispiel | Bemerkung |
|---|---|---|---|---|
| `id` | string | ja | `H001` | Eindeutig |
| `name` | string | ja | `Hotel Adler` | Anzeigename |
| `address` | string | nein | `Hauptstr. 1, 70173 Stuttgart` | Frei |
| `lat` | number | ja | `48.7758` | WGS84 |
| `lng` | number | ja | `9.1829` | WGS84 |
| `revenue_potential` | number | ja | `45000` | EUR / Jahr |
| `close_probability` | number | ja | `0.6` | 0..1 (Komma `.`) |
| `status` | enum | ja | `prospect` | `key_account` \| `active` \| `prospect` \| `lead` \| `inactive` |
| `strategic_value` | integer | ja | `4` | 1..5 |
| `window_start` | string | nein | `09:00` | HH:MM, sonst leer |
| `window_end` | string | nein | `17:00` | HH:MM, sonst leer |
| `visit_duration_min` | integer | nein | `45` | Default 30 |
| `notes` | string | nein | `Vorab anrufen` | Frei |

**Validierung:**
- Fehlt eine Pflichtspalte → Upload mit klarer Fehlermeldung ablehnen.
- Ungültige `lat`/`lng` (außerhalb -90/90 bzw. -180/180) → Zeile mit Warnung überspringen.
- Unbekannter `status` → als `lead` interpretieren, Warnung loggen.

### 4.2 Internes Hotel-Modell (`types/hotel.ts`)

```ts
export type HotelStatus = 'key_account' | 'active' | 'prospect' | 'lead' | 'inactive';

export interface Hotel {
  id: string;
  name: string;
  address?: string;
  lat: number;
  lng: number;
  revenuePotential: number;
  closeProbability: number;   // 0..1
  status: HotelStatus;
  strategicValue: number;     // 1..5
  windowStart?: string;       // 'HH:MM'
  windowEnd?: string;         // 'HH:MM'
  visitDurationMin: number;   // default 30
  notes?: string;
}

export interface ScoredHotel extends Hotel {
  score: number;              // 0..1
  scoreBreakdown: Record<keyof ScoreWeights, number>;
}
```

---

## 5. Priorisierung (Scoring)

**Wirtschaftlicher Nutzen > reine Distanz.** Score-Funktion ist gewichtete Summe normalisierter Faktoren:

```
score = w_revenue   * norm(revenue_potential)
      + w_prob      * close_probability
      + w_status    * statusWeight(status)
      + w_strategic * (strategic_value / 5)
      + w_proximity * (1 - norm(distance_from_start))
      + w_window    * windowFit(window_start, window_end)
```

### 5.1 Default-Gewichte (`lib/scoring.ts`)

```ts
export const DEFAULT_WEIGHTS: ScoreWeights = {
  revenue:   0.30,
  prob:      0.20,
  status:    0.15,
  strategic: 0.10,
  proximity: 0.15,
  window:    0.10,
};
```

Summe = 1.0. Die Gewichte sind im UI verstellbar (`<ScoreWeightsPanel />`), Änderungen triggern Re-Scoring.

### 5.2 Sub-Berechnungen

- `norm(x)` = Min-Max-Normalisierung über den aktuellen Datensatz auf `[0,1]`. Sonderfall: Bei nur einem Hotel oder identischen Werten → `0.5`.
- `statusWeight`: `key_account=1.0`, `active=0.8`, `prospect=0.6`, `lead=0.4`, `inactive=0.1`.
- `distance_from_start`: Haversine-Distanz in km vom Startpunkt (User-Position oder erstes Hotel).
- `windowFit`: `1.0` wenn kein Zeitfenster gesetzt; `1.0` wenn aktuelle Zeit + geschätzte Anfahrt im Fenster liegt; sonst linear absteigend auf 0 außerhalb.

### 5.3 Score-Breakdown

Jedes `ScoredHotel` enthält neben `score` ein `scoreBreakdown` mit den sechs gewichteten Teilwerten. Die Tabelle zeigt diese als Tooltip beim Hover über den Score.

---

## 6. Routing-Logik (MVP)

**Algorithmus:** *Score-First Greedy mit Distanz-Penalty.*

```
α = 0.3   // konfigurierbar, 0 = nur Score, 1 = nur Nearest-Neighbor
current = start
remaining = alle Hotels
route = []
while remaining ist nicht leer:
    next = argmax_{h ∈ remaining} ( score(h) − α · norm(distance(current, h)) )
    route.append(next)
    current = next
    remaining.remove(next)
return route
```

**Eigenschaften:**
- Deterministisch (bei stabiler Sortierung als Tiebreaker).
- O(n²) — für MVP-Größen (≤ 200 Hotels) ausreichend.
- Kein TSP-Anspruch — Score dominiert. Akzeptiert als "sinnvolle Reihenfolge".

**Distanz:** Haversine-Luftlinie in km. Echtes Straßennetz folgt später über `RoutingService`-Interface.

---

## 7. Modulare Architektur

### 7.1 RoutingService-Interface (`lib/routing/types.ts`)

```ts
export interface RoutePlanRequest {
  start: { lat: number; lng: number };
  stops: ScoredHotel[];
  options?: {
    alpha?: number;        // default 0.3
    respectWindows?: boolean; // default false im MVP
  };
}

export interface RoutePlanResult {
  ordered: ScoredHotel[];
  totalDistanceKm: number;
  totalScore: number;
  legs: { fromId: string; toId: string; distanceKm: number }[];
}

export interface RoutingService {
  plan(req: RoutePlanRequest): Promise<RoutePlanResult>;
}
```

### 7.2 MVP-Implementierung

`lib/routing/localGreedy.ts` implementiert `RoutingService` mit dem Algorithmus aus §6.
`lib/routing/index.ts` exportiert eine Factory `getRoutingService()`, die später per ENV-Variable z.B. einen `OsrmRoutingService` oder `GoogleRoutingService` zurückgeben kann.

---

## 8. UI-Anforderungen

### 8.1 Seiten

- **`/`** — Landing mit CSV-Upload (Drag & Drop + Button), Link "Beispiel-CSV laden".
- **`/plan`** — Hauptansicht nach Upload:
  - Links: Score-Gewichte-Panel + sortierbare Hoteltabelle
  - Rechts: Karte mit Markern (Farbe nach Status, Größe nach Score) + nummerierte Routenpolyline
  - Oben: Aktionen "Route neu berechnen", "Route exportieren (CSV)", "Neue CSV laden"

### 8.2 Komponenten (`components/`)

| Komponente | Zweck |
|---|---|
| `CsvUpload` | Drag&Drop + File-Picker, ruft PapaParse, validiert, leitet auf `/plan` weiter |
| `HotelTable` | Sortierbare Tabelle: #, Name, Status, Score, Distanz, Zeitfenster |
| `PriorityBadge` | Farbcodierter Score-Badge (rot/gelb/grün) |
| `RouteMap` | Leaflet-Karte mit Markern + nummerierter Polyline |
| `RouteList` | Geordnete Stop-Liste mit ETA-Schätzung |
| `ScoreWeightsPanel` | Slider für die sechs Gewichte, Normalisierung auf Summe=1 |

### 8.3 State

Lokaler React-State in `/plan` (kein Redux/Zustand nötig). Optional: Hotels im `sessionStorage` für Refresh-Resistenz.

---

## 9. Projektstruktur

```
/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                  # Landing + Upload
│   └── plan/page.tsx             # Hauptansicht (Client Component)
├── components/
│   ├── CsvUpload.tsx
│   ├── HotelTable.tsx
│   ├── PriorityBadge.tsx
│   ├── RouteMap.tsx              # dynamic import (Leaflet = SSR-unfreundlich)
│   ├── RouteList.tsx
│   └── ScoreWeightsPanel.tsx
├── lib/
│   ├── csv.ts                    # parseCsv() + exportRouteToCsv()
│   ├── scoring.ts                # computeScores(hotels, weights, start)
│   ├── distance.ts               # haversineKm()
│   └── routing/
│       ├── types.ts
│       ├── localGreedy.ts
│       └── index.ts              # getRoutingService()
├── types/
│   └── hotel.ts
├── public/
│   └── sample_hotels.csv         # Kopie der Beispieldaten
├── package.json
├── tsconfig.json
├── tailwind.config.ts
└── README.md
```

---

## 10. CSV-Export (Output)

Format der Routen-Export-CSV (`exportRouteToCsv`):

```
stop_no,id,name,address,lat,lng,score,distance_to_prev_km,cumulative_km,eta,notes
1,H003,Hotel Adler,Hauptstr. 1...,48.77,9.18,0.87,0.0,0.0,09:00,
2,H001,Park Inn,...,48.81,9.20,0.74,3.8,3.8,09:35,Vorab anrufen
...
```

---

## 11. MVP-Akzeptanzkriterien (aus Issue #1, verfeinert)

- [ ] CSV-Upload funktioniert (Drag&Drop + File-Picker)
- [ ] `sample_hotels.csv` lässt sich fehlerfrei einlesen
- [ ] Kundenliste wird als sortierbare Tabelle gezeigt
- [ ] Prioritätsscore wird gemäß Formel aus §5 berechnet
- [ ] Default-Sortierung: Score absteigend
- [ ] Score-Breakdown per Tooltip einsehbar
- [ ] Gewichte im UI verstellbar (Re-Scoring live)
- [ ] Routenreihenfolge wird via `localGreedy` erzeugt
- [ ] Karte zeigt Marker + nummerierte Polyline der Route
- [ ] CSV-Export der Route gemäß §10
- [ ] Code modular gegliedert wie in §9
- [ ] `RoutingService` hinter Interface (späterer Austausch möglich)
- [ ] README erklärt: Install, Dev-Start, Build, CSV-Format

---

## 12. Dev-Workflow

```bash
pnpm install
pnpm dev                # http://localhost:3000
pnpm build && pnpm start
pnpm test               # optional, Vitest
pnpm lint
```

**Konventionen:**
- ESLint + Prettier mit Next-Defaults
- Commit-Stil: Conventional Commits (`feat:`, `fix:`, `refactor:` ...)
- Eine Datei pro Komponente, benannt wie der Default-Export

---

## 13. Roadmap (Post-MVP)

1. Geocoding-Service (modular): Adresse → lat/lng
2. Echtes Straßenrouting via OSRM oder OpenRouteService (drop-in via `RoutingService`)
3. Zeitfenster + Besuchsdauer harter Constraint (statt Soft-Score)
4. HubSpot-Connector als Datenquelle alternativ zur CSV
5. Persistenz / Multi-User (Auth + DB)
6. Tagesplanung: mehrere Tage, Tagesstart/-ende, Übernachtung
