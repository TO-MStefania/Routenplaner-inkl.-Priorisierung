# Projekt: Hotel Route Planner

Baue eine Webapp für priorisierte Außendienst-Routenplanung im Hotelvertrieb.

Claude soll NICHT selbst als KI die Route optimieren.
Die App soll algorithmisch arbeiten.

## Ziel
Nutzer laden eine CSV mit Hotelkunden hoch.
Die App berechnet Prioritätsscores.
Die App zeigt Kunden auf einer Karte.
Die App erstellt eine sinnvolle Besuchsreihenfolge.

## Priorisierung
Wirtschaftlicher Nutzen ist wichtiger als reine Distanz.

Bewertung:
- Umsatzpotenzial
- Abschlusswahrscheinlichkeit
- Kundenstatus
- strategische Bedeutung
- geografische Nähe
- Zeitfenster

## MVP
1. CSV Upload
2. Kundenliste anzeigen
3. Prioritätsscore berechnen
4. Karte vorbereiten
5. einfache Routenlogik
6. Export als CSV

## Technische Vorgabe
- Next.js
- TypeScript
- Tailwind
- keine HubSpot-Integration im ersten Schritt
- Routing-Service modular vorbereiten
