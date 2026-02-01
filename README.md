# Flottenoptimierung – Elektrifizierung der Logistik

MILP-Optimierungsmodell zur kostenminimalen Auslegung einer gemischten Diesel-/Elektro-LKW-Flotte mit Ladeinfrastruktur und Batteriespeicher.

## Problemstellung

Ein Logistikdepot muss täglich 20 fixe Touren bedienen. Das Modell optimiert:

- **Flottenkomposition**: Welche LKW-Typen (Diesel vs. Elektro) in welcher Anzahl?
- **Ladeinfrastruktur**: Welche Ladesäulen (50/200/400 kW) installieren?
- **Operative Planung**: Welches Fahrzeug fährt welche Tour? Wann wird geladen?
- **Batteriespeicher**: Lohnt sich ein stationärer Speicher zur Lastspitzenreduktion?

## Fahrzeugtypen

| Typ | Antrieb | Batterie | Verbrauch |
|-----|---------|----------|-----------|
| ActrosL | Diesel | – | 26 L/100km |
| eActros400 | Elektro | 414 kWh | 105 kWh/100km |
| eActros600 | Elektro | 621 kWh | 110 kWh/100km |

## Quickstart

```bash
# In Google Colab oder lokal mit Jupyter
pip install pulp highspy
```

1. `fallstudie_scm_optimization_improved.ipynb` öffnen
2. Erste Zelle ausführen (Installation)
3. Alle weiteren Zellen ausführen
4. Ergebnisse in den Visualisierungen ablesen

## Modell

- **Typ**: Gemischt-ganzzahliges lineares Programm (MILP)
- **Variablen**: ~20.000 (davon ~15.000 binär)
- **Nebenbedingungen**: ~62.500
- **Solver**: HiGHS (2h Zeitlimit)

Details siehe `Optimierungsmodell_Dokumentation.md`.

## Outputs

Das Notebook liefert:

1. **Zusammenfassung** – Optimale Flotte, Infrastruktur, Gesamtkosten
2. **Gantt-Diagramm** – Tagesablauf aller Fahrzeuge
3. **SOC-Verläufe** – Batterieladezustand der E-LKW
4. **Netzlast-Diagramm** – Leistungsbezug über 24h

## Dateien

```
├── fallstudie_scm_optimization_improved.ipynb  # Hauptnotebook
├── Optimierungsmodell_Dokumentation.md         # Mathematische Formulierung
├── Fallstudientext.pdf                         # Aufgabenstellung
└── README.md
```

## Rahmenbedingungen

- 20 Touren täglich, 260 Betriebstage/Jahr
- Zeitdiskretisierung: 96 Schritte à 15 Minuten
- Netzanschluss: 500 kW (erweiterbar auf 1000 kW)
- Max. 3 Ladesäulen installierbar
- Nachtparkregeln: 18:00–06:00 eingeschränkte Bewegung

## Autor

DHBW Fallstudie – Supply Chain Management
