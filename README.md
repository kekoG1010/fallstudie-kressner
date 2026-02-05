# Flottenoptimierung – Elektrifizierung der Logistik

MILP-Optimierungsmodell zur kostenminimalen Auslegung einer gemischten Diesel-/Elektro-LKW-Flotte mit Ladeinfrastruktur und Batteriespeicher.

## Inhaltsverzeichnis

- [Problemstellung](#problemstellung)
- [Projektstruktur](#projektstruktur)
- [Setup](#setup)
- [Solver](#solver)
- [Rahmenbedingungen](#rahmenbedingungen)
- [Autoren](#autoren)

## Problemstellung

Ein Logistikdepot muss täglich 20 fixe Touren bedienen. Das Modell optimiert:

- **Flottenkomposition**: Optimale Anzahl und Typen von Diesel- und Elektro-LKW
- **Ladeinfrastruktur**: Auswahl der Ladesäulen (50/200/400 kW) und Dimensionierung eines stationären Batteriespeichers
- **Operative Planung**: Zuweisung von Fahrzeugen zu Touren und zeitliche Ladeplanung

## Projektstruktur

```
├── Fallstudie/
│   ├── fallstudie_scm_optimization_GUROBI.ipynb      # Hauptnotebook (Teilaufgaben 1–3)
│   └── Optimierungsmodell.md                          # Mathematische Formulierung (NB1–NB32)
│
├── erweiterung/
│   ├── fallstudie_scm_optimization_GUROBI_Erweiterung.ipynb  # Erweiterung (Teilaufgabe 4)
│   └── Optimierungsmodell_Erweiterung.md                      # Erweiterte Formulierung (NB1–NB35)
│
└── README.md
```

### Fallstudie (Teilaufgaben 1–3)

Basismodell mit 32 Nebenbedingungen (NB1–NB32):

- Tourenabdeckung und Fahrzeugzuweisung
- Reichweiten- und Kapazitätsprüfung
- Ladeplanung mit SOC-Tracking (96 Zeitschritte à 15 min)
- Ladesäulenkonfiguration und Netzanschlussdimensionierung
- Batteriespeicher zur Lastspitzenreduktion
- Nachtparkregeln und operative Restriktionen

### Erweiterung (Teilaufgabe 4): CO₂-Emissionsobergrenze

Erweiterung des Basismodells um eine jährliche CO₂-Obergrenze (450 t/Jahr) mit drei zusätzlichen Nebenbedingungen (NB33–NB35):

- **Exklusive Stromtypwahl**: Binäre Entscheidung zwischen Ökostrom (0,35 EUR/kWh, 0 kg CO₂/kWh) und Industriestrom (0,25 EUR/kWh, 0,45 kg CO₂/kWh)
- **CO₂-Emissionsberechnung**: Diesel (2,65 kg CO₂/L) + Strom
- **CO₂-Obergrenze**: Harte Nebenbedingung ≤ 450 t/Jahr

## Setup

### Voraussetzungen

- Python 3.10+
- Gurobi Solver mit gültiger Lizenz (kostenlose akademische Lizenz verfügbar)

### Installation

```bash
pip install pulp gurobipy
```

### Gurobi-Lizenz einrichten

1. Account erstellen auf [gurobi.com](https://www.gurobi.com)
2. Akademische Lizenz beantragen unter *Downloads & Licenses → Academic License*
3. Lizenz aktivieren:

```bash
grbgetkey <LIZENZSCHLÜSSEL>
```

### Ausführung

1. Gewünschtes Notebook öffnen (Jupyter / VS Code)
2. Alle Zellen ausführen
3. Ergebnisse in den Visualisierungen ablesen

## Solver

Das Modell verwendet **Gurobi** als MILP-Solver (via PuLP-Schnittstelle).

|                          | Basismodell    | Mit Erweiterung |
|--------------------------|----------------|-----------------|
| Binärvariablen           | ca. 19.000     | ca. 19.000      |
| Kontinuierliche Variablen| ca. 5.000      | ca. 5.200       |
| **Variablen gesamt**     | **ca. 24.000** | **ca. 24.200**  |
| **Nebenbedingungen**     | **ca. 75.000** | **ca. 75.100**  |

- **Zeitlimit**: 2 Stunden

## Rahmenbedingungen

- 20 Touren täglich, 260 Betriebstage/Jahr
- Zeitdiskretisierung: 96 Schritte à 15 Minuten
- Netzanschluss: 500 kW (erweiterbar auf 1.000 kW)
- Max. 3 Ladesäulen installierbar
- Nachtparkregeln: 18:00–06:00 eingeschränkte Bewegung

## Autoren

DHBW Fallstudie – Supply Chain Management

- Sören Andreas Frank
- Viet Duc Nguyen
- Abdul Kerim Kaya
