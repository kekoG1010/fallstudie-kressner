# Mathematische Modellformulierung: Elektrifizierung der LKW-Flotte

## Optimierungsproblem

**Ziel:** Minimierung der jährlichen Gesamtkosten für die Flottenplanung eines Logistikdepots unter Berücksichtigung von Diesel- und Elektro-LKW, Ladeinfrastruktur und Batteriespeicher.

**Modelltyp:** Gemischt-ganzzahliges lineares Programm (MILP)

**Solver:** Gurobi (via PuLP)

---

## 1. Indexmengen

| Symbol                | Beschreibung              | Aussprache  | Werte                                                                    |
| --------------------- | ------------------------- | ----------- | ------------------------------------------------------------------------ |
| $\mathcal{V}$         | Menge der Fahrzeuge       | „V"         | $\{v_1, v_2, \ldots, v_{20}\}$                                           |
| $\mathcal{F}$         | Menge der Fahrzeugtypen   | „F"         | $\{\text{ActrosL}, \text{eActros400}, \text{eActros600}\}$               |
| $\mathcal{F}^D$       | Menge der Diesel-Typen    | „F-Diesel"  | $\{\text{ActrosL}\}$                                                     |
| $\mathcal{F}^E$       | Menge der Elektro-Typen   | „F-Elektro" | $\{\text{eActros400}, \text{eActros600}\}$                               |
| $\mathcal{R}$         | Menge der Touren/Routen   | „R"         | 20 Touren                                                                |
| $\mathcal{L}$         | Menge der Ladesäulentypen | „L"         | $\{\text{Alpitronic-50}, \text{Alpitronic-200}, \text{Alpitronic-400}\}$ |
| $\mathcal{T}$         | Menge der Zeitschritte    | „T"         | $\{1, 2, \ldots, 96\}$ (je 15 Minuten)                                   |
| $\mathcal{T}^{Nacht}$ | Nacht-Zeitschritte        | „T-Nacht"   | $\{73, \ldots, 96\} \cup \{1, \ldots, 24\}$ (18:00–06:00)                |
| $\mathcal{T}^{Tag}$   | Tag-Zeitschritte          | „T-Tag"     | $\{25, \ldots, 72\}$ (06:00–18:00)                                       |

---

## 2. Parameter

### 2.1 Zeitparameter

| Symbol     | Beschreibung          | Aussprache | Wert | Einheit   |
| ---------- | --------------------- | ---------- | ---- | --------- |
| $\Delta t$ | Zeitschrittlänge      | „Delta t"  | 0,25 | h         |
| $D$        | Betriebstage pro Jahr | „D"        | 260  | Tage/Jahr |

### 2.2 Fahrzeugparameter (pro Typ $f \in \mathcal{F}$)

| Symbol          | Beschreibung                    | Aussprache    | ActrosL | eActros400 | eActros600 | Einheit                |
| --------------- | ------------------------------- | ------------- | ------- | ---------- | ---------- | ---------------------- |
| $c^{CAPEX}_f$   | Anschaffungskosten (annuisiert) | „c-Capex-f"   | 24.000  | 50.000     | 60.000     | EUR/Jahr               |
| $c^{OPEX}_f$    | Betriebskosten (Wartung etc.)   | „c-Opex-f"    | 6.000   | 5.000      | 6.000      | EUR/Jahr               |
| $c^{KFZ}_f$     | KFZ-Steuer                      | „c-KFZ-f"     | 556     | 0          | 0          | EUR/Jahr               |
| $c^{THG}_f$     | THG-Quoten-Erlös                | „c-THG-f"     | 0       | 1.000      | 1.000      | EUR/Jahr               |
| $\kappa_f$      | Verbrauch                       | „Kappa-f"     | 26      | 105        | 110        | L/100km bzw. kWh/100km |
| $Q^{max}_f$     | Batteriekapazität               | „Q-max-f"     | 0       | 414        | 621        | kWh                    |
| $P^{max,Fzg}_f$ | Max. Ladeleistung               | „P-max-Fzg-f" | 0       | 400        | 400        | kW                     |
| $SOC^{min}_f$   | Min. Ladezustand (10%)          | „SOC-min-f"   | 0       | 41,4       | 62,1       | kWh                    |

### 2.3 Routenparameter (pro Route $r \in \mathcal{R}$)

| Symbol        | Beschreibung            | Aussprache  | Einheit |
| ------------- | ----------------------- | ----------- | ------- |
| $d_r$         | Gesamtdistanz der Route | „d-r"       | km      |
| $d^{Maut}_r$  | Mautpflichtige Distanz  | „d-Maut-r"  | km      |
| $t^{start}_r$ | Startzeitschritt        | „t-start-r" | –       |
| $t^{end}_r$   | Endzeitschritt          | „t-end-r"   | –       |

### 2.4 Ladesäulenparameter (pro Typ $l \in \mathcal{L}$)

| Symbol        | Beschreibung       | Aussprache  | Alpi-50 | Alpi-200 | Alpi-400 | Einheit  |
| ------------- | ------------------ | ----------- | ------- | -------- | -------- | -------- |
| $c^{CAPEX}_l$ | Anschaffungskosten | „c-Capex-l" | 3.000   | 10.000   | 16.000   | EUR/Jahr |
| $c^{OPEX}_l$  | Betriebskosten     | „c-Opex-l"  | 1.000   | 1.500    | 2.000    | EUR/Jahr |
| $P^{max}_l$   | Max. Ladeleistung  | „P-max-l"   | 50      | 200      | 400      | kW       |
| $n^{Spots}_l$ | Anzahl Ladepunkte  | „n-Spots-l" | 2       | 2        | 2        | –        |

### 2.5 Energiekosten und Netzparameter

| Symbol           | Beschreibung              | Aussprache           | Wert   | Einheit  |
| ---------------- | ------------------------- | -------------------- | ------ | -------- |
| $p^{Arbeit}$     | Arbeitspreis Strom        | „p-Arbeit"           | 0,25   | EUR/kWh  |
| $p^{Leistung}$   | Leistungspreis Strom      | „p-Leistung"         | 150    | EUR/kW   |
| $p^{Grund}$      | Grundgebühr Strom         | „p-Grund"            | 1.000  | EUR/Jahr |
| $p^{Diesel}$     | Dieselpreis               | „p-Diesel"           | 1,60   | EUR/L    |
| $p^{Maut}$       | Mautgebühr (Diesel)       | „p-Maut"             | 0,34   | EUR/km   |
| $P^{Netz,Basis}$ | Basis-Netzanschluss       | „P-Netz-Basis"       | 500    | kW       |
| $P^{Netz,Erw}$   | Erweiterung Netzanschluss | „P-Netz-Erweiterung" | 500    | kW       |
| $c^{Netz,Erw}$   | Kosten Netzerweiterung    | „c-Netz-Erweiterung" | 10.000 | EUR/Jahr |
| $L^{max}$        | Max. Anzahl Ladesäulen    | „L-max"              | 3      | –        |

### 2.6 Speicherparameter

| Symbol          | Beschreibung               | Aussprache     | Wert  | Einheit |
| --------------- | -------------------------- | -------------- | ----- | ------- |
| $c^{Sp,P}$      | Speicher-CAPEX Leistung    | „c-Speicher-P" | 30    | EUR/kW  |
| $c^{Sp,E}$      | Speicher-CAPEX Kapazität   | „c-Speicher-E" | 350   | EUR/kWh |
| $\alpha^{OPEX}$ | OPEX-Rate Speicher         | „Alpha-OPEX"   | 0,02  | –       |
| $\eta$          | Round-Trip-Wirkungsgrad    | „Eta"          | 0,98  | –       |
| $\eta^{ch}$     | Ladewirkungsgrad Fahrzeuge | „Eta-charge"   | 0,95  | –       |
| $DoD$           | Max. Entladetiefe          | „DoD"          | 0,975 | –       |

### 2.7 Big-M Parameter

| Symbol        | Beschreibung              | Aussprache   | Wert   | Einheit |
| ------------- | ------------------------- | ------------ | ------ | ------- |
| $M^{SOC}$     | Big-M für SOC-Constraints | „M-SOC"      | 621    | kWh     |
| $M^{Sp}$      | Big-M für Speicher        | „M-Speicher" | 10.000 | kW      |
| $M^{ch}$      | Big-M für Ladeleistung    | „M-charge"   | 400    | kW      |
| $\varepsilon$ | Minimalladeleistung       | „Epsilon"    | 0,1    | kW      |

---

## 3. Entscheidungsvariablen

### 3.1 Binärvariablen (0 oder 1)

| Symbol         | Beschreibung                                                 | Aussprache  | Index                                                     |
| -------------- | ------------------------------------------------------------ | ----------- | --------------------------------------------------------- |
| $use_v$        | Fahrzeug $v$ wird eingesetzt                                 | „use-v"     | $v \in \mathcal{V}$                                       |
| $\tau_{v,f}$   | Fahrzeug $v$ ist vom Typ $f$                                 | „Tau-v-f"   | $v \in \mathcal{V}, f \in \mathcal{F}$                    |
| $\epsilon_v$   | Fahrzeug $v$ ist ein E-LKW                                   | „Epsilon-v" | $v \in \mathcal{V}$                                       |
| $x_{v,r}$      | Fahrzeug $v$ fährt Route $r$                                 | „x-v-r"     | $v \in \mathcal{V}, r \in \mathcal{R}$                    |
| $y_l$          | Ladesäule $l$ wird installiert                               | „y-l"       | $l \in \mathcal{L}$                                       |
| $w_{v,l,t}$    | Fahrzeug $v$ belegt Ladepunkt an Säule $l$ zum Zeitpunkt $t$ | „w-v-l-t"   | $v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}$ |
| $\omega_{v,t}$ | Fahrzeug $v$ ist auf Route zum Zeitpunkt $t$                 | „Omega-v-t" | $v \in \mathcal{V}, t \in \mathcal{T}$                    |
| $\chi_{v,t}$   | Fahrzeug $v$ lädt aktiv zum Zeitpunkt $t$                    | „Chi-v-t"   | $v \in \mathcal{V}, t \in \mathcal{T}$                    |
| $\gamma$       | Netzanschluss wird erweitert                                 | „Gamma"     | –                                                         |
| $\sigma_t$     | Speicher-Modus: 1=Laden, 0=Entladen                          | „Sigma-t"   | $t \in \mathcal{T}$                                       |
| $\delta_{v,r}$ | Fahrzeug $v$ fährt Route $r$ mit Diesel                      | „Delta-v-r" | $v \in \mathcal{V}, r \in \mathcal{R}$                    |
| $\phi_{v,r,f}$ | Fahrzeug $v$ fährt Route $r$ mit Typ $f$                     | „Phi-v-r-f" | $v \in \mathcal{V}, r \in \mathcal{R}, f \in \mathcal{F}$ |
| $\mu_{v,t}$    | Fahrzeug $v$ ist zum Zeitpunkt $t$ vollgeladen               | „Mü-v-t"    | $v \in \mathcal{V}, t \in \mathcal{T}$                    |
| $\nu_{v,t}$    | Fahrzeug $v$ benötigt Laden (SOC < Schwellwert)              | „Nü-v-t"    | $v \in \mathcal{V}, t \in \mathcal{T}$                    |

### 3.2 Kontinuierliche Variablen (≥ 0)

| Symbol           | Beschreibung                                             | Aussprache               | Index                                                     | Einheit |
| ---------------- | -------------------------------------------------------- | ------------------------ | --------------------------------------------------------- | ------- |
| $SOC_{v,t}$      | Ladezustand Fahrzeug $v$ zum Zeitpunkt $t$               | „SOC-v-t"                | $v \in \mathcal{V}, t \in \mathcal{T}$                    | kWh     |
| $p^{ch}_{v,l,t}$ | Ladeleistung Fahrzeug $v$ an Säule $l$ zum Zeitpunkt $t$ | „p-charge-v-l-t"         | $v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}$ | kW      |
| $p^{Netz}_t$     | Netzbezug zum Zeitpunkt $t$                              | „p-Netz-t"               | $t \in \mathcal{T}$                                       | kW      |
| $P^{Peak}$       | Maximale Bezugsleistung (Spitzenlast)                    | „P-Peak"                 | –                                                         | kW      |
| $P^{Sp}$         | Installierte Speicherleistung                            | „P-Speicher"             | –                                                         | kW      |
| $E^{Sp}$         | Installierte Speicherkapazität                           | „E-Speicher"             | –                                                         | kWh     |
| $SOC^{Sp}_t$     | Ladezustand Speicher zum Zeitpunkt $t$                   | „SOC-Speicher-t"         | $t \in \mathcal{T}$                                       | kWh     |
| $p^{Sp,ch}_t$    | Speicher-Ladeleistung zum Zeitpunkt $t$                  | „p-Speicher-charge-t"    | $t \in \mathcal{T}$                                       | kW      |
| $p^{Sp,dis}_t$   | Speicher-Entladeleistung zum Zeitpunkt $t$               | „p-Speicher-discharge-t" | $t \in \mathcal{T}$                                       | kW      |

---

## 4. Zielfunktion

$$\min \quad C^{Gesamt} = C^{LKW} + C^{Lade} + C^{Strom} + C^{Netz} + C^{Speicher} + C^{Diesel} + C^{Maut} - C^{THG}$$

**Aussprache:** „Minimiere C-Gesamt gleich C-LKW plus C-Lade plus C-Strom plus C-Netz plus C-Speicher plus C-Diesel plus C-Maut minus C-THG"

**Bedeutung:** Die jährlichen Gesamtkosten setzen sich zusammen aus den Fahrzeugkosten, Ladeinfrastrukturkosten, Stromkosten, Netzkosten, Speicherkosten, Dieselkosten und Mautkosten, abzüglich der THG-Quoten-Erlöse.

### 4.1 Kostenkomponenten

#### LKW-Kosten

$$C^{LKW} = \sum_{v \in \mathcal{V}} \sum_{f \in \mathcal{F}} \left( c^{CAPEX}_f + c^{OPEX}_f + c^{KFZ}_f \right) \cdot \tau_{v,f}$$

**Aussprache:** „C-LKW gleich Summe über alle Fahrzeuge v und alle Typen f von Klammer c-Capex-f plus c-Opex-f plus c-KFZ-f Klammer zu mal Tau-v-f"

**Bedeutung:** Summe der jährlichen Fahrzeugkosten (Anschaffung + Betrieb + Steuer) für alle eingesetzten Fahrzeuge.

#### THG-Quoten-Erlöse

$$C^{THG} = \sum_{v \in \mathcal{V}} \sum_{f \in \mathcal{F}^E} c^{THG}_f \cdot \tau_{v,f}$$

**Aussprache:** „C-THG gleich Summe über alle Fahrzeuge v und alle E-Typen f von c-THG-f mal Tau-v-f"

**Bedeutung:** Jährliche Erlöse aus der THG-Quote für alle Elektro-LKW.

#### Ladeinfrastrukturkosten

$$C^{Lade} = \sum_{l \in \mathcal{L}} \left( c^{CAPEX}_l + c^{OPEX}_l \right) \cdot y_l$$

**Aussprache:** „C-Lade gleich Summe über alle Ladesäulen l von Klammer c-Capex-l plus c-Opex-l Klammer zu mal y-l"

**Bedeutung:** Jährliche Kosten für installierte Ladesäulen.

#### Stromkosten

$$C^{Strom} = p^{Grund} + p^{Leistung} \cdot P^{Peak} + p^{Arbeit} \cdot D \cdot \sum_{t \in \mathcal{T}} p^{Netz}_t \cdot \Delta t$$

**Aussprache:** „C-Strom gleich p-Grund plus p-Leistung mal P-Peak plus p-Arbeit mal D mal Summe über alle t von p-Netz-t mal Delta-t"

**Bedeutung:** Stromkosten bestehend aus Grundgebühr, Leistungspreis (basierend auf Spitzenlast) und Arbeitspreis (basierend auf Jahresenergieverbrauch).

#### Netzkosten

$$C^{Netz} = c^{Netz,Erw} \cdot \gamma$$

**Aussprache:** „C-Netz gleich c-Netz-Erweiterung mal Gamma"

**Bedeutung:** Kosten für die optionale Erweiterung des Netzanschlusses.

#### Speicherkosten

$$C^{Speicher} = (1 + \alpha^{OPEX}) \cdot \left( c^{Sp,P} \cdot P^{Sp} + c^{Sp,E} \cdot E^{Sp} \right)$$

**Aussprache:** „C-Speicher gleich Klammer eins plus Alpha-OPEX Klammer zu mal Klammer c-Speicher-P mal P-Speicher plus c-Speicher-E mal E-Speicher Klammer zu"

**Bedeutung:** Jährliche Speicherkosten (CAPEX für Leistung und Kapazität, plus 2% OPEX).

#### Dieselkosten

$$C^{Diesel} = D \cdot p^{Diesel} \cdot \frac{\kappa_{ActrosL}}{100} \cdot \sum_{v \in \mathcal{V}} \sum_{r \in \mathcal{R}} d_r \cdot \delta_{v,r}$$

**Aussprache:** „C-Diesel gleich D mal p-Diesel mal Kappa-ActrosL durch hundert mal Summe über v und r von d-r mal Delta-v-r"

**Bedeutung:** Jährliche Dieselkosten für alle Diesel-LKW basierend auf gefahrenen Kilometern.

#### Mautkosten

$$C^{Maut} = D \cdot p^{Maut} \cdot \sum_{v \in \mathcal{V}} \sum_{r \in \mathcal{R}} d^{Maut}_r \cdot \delta_{v,r}$$

**Aussprache:** „C-Maut gleich D mal p-Maut mal Summe über v und r von d-Maut-r mal Delta-v-r"

**Bedeutung:** Jährliche Mautkosten für Diesel-LKW auf mautpflichtigen Strecken.

---

## 5. Nebenbedingungen

### NB1: Tourenabdeckung

$$\sum_{v \in \mathcal{V}} x_{v,r} = 1 \quad \forall r \in \mathcal{R}$$

**Aussprache:** „Summe über alle Fahrzeuge v von x-v-r gleich eins für alle Routen r"

**Bedeutung:** Jede Tour muss von genau einem Fahrzeug gefahren werden.

---

### NB2: Typzuweisung (NB_Typzuweisung)

$$\sum_{f \in \mathcal{F}} \tau_{v,f} = use_v \quad \forall v \in \mathcal{V}$$

**Aussprache:** „Summe über alle Typen f von Tau-v-f gleich use-v für alle Fahrzeuge v"

**Bedeutung:** Jedes aktive Fahrzeug hat genau einen Typ; inaktive Fahrzeuge haben keinen Typ.

---

### NB3: E-Fahrzeug-Identifikation (NB_IsElectric)

$$\epsilon_v = \sum_{f \in \mathcal{F}^E} \tau_{v,f} \quad \forall v \in \mathcal{V}$$

**Aussprache:** „Epsilon-v gleich Summe über alle E-Typen f von Tau-v-f für alle Fahrzeuge v"

**Bedeutung:** Ein Fahrzeug ist genau dann ein E-LKW, wenn es einem Elektro-Typ zugeordnet ist.

---

### NB4: Fahrzeug-Aktivierung (NB_Aktivierung)

$$x_{v,r} \leq use_v \quad \forall v \in \mathcal{V}, r \in \mathcal{R}$$

**Aussprache:** „x-v-r kleiner gleich use-v für alle Fahrzeuge v und alle Routen r"

**Bedeutung:** Ein Fahrzeug kann nur Routen fahren, wenn es aktiviert ist.

---

### NB5: Zeitliche Überlappung (NB_EineRoute)

$$\sum_{r \in \mathcal{R}(t)} x_{v,r} \leq 1 \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

wobei $\mathcal{R}(t) = \{r \in \mathcal{R} : t^{start}_r \leq t < t^{end}_r\}$

**Aussprache:** „Summe über alle zum Zeitpunkt t aktiven Routen r von x-v-r kleiner gleich eins für alle v und t"

**Bedeutung:** Ein Fahrzeug kann zu jedem Zeitpunkt höchstens eine Route fahren.

---

### NB6: Maximale Ladesäulenanzahl

$$\sum_{l \in \mathcal{L}} y_l \leq L^{max}$$

**Aussprache:** „Summe über alle Ladesäulen l von y-l kleiner gleich L-max"

**Bedeutung:** Es können maximal 3 Ladesäulen installiert werden.

---

### NB7: Netzleistungsbegrenzung

$$p^{Netz}_t \leq P^{Netz,Basis} + P^{Netz,Erw} \cdot \gamma \quad \forall t \in \mathcal{T}$$

**Aussprache:** „p-Netz-t kleiner gleich P-Netz-Basis plus P-Netz-Erweiterung mal Gamma für alle t"

**Bedeutung:** Der Netzbezug ist durch den (ggf. erweiterten) Netzanschluss begrenzt.

---

### NB8: Spitzenlasterfassung

$$P^{Peak} \geq p^{Netz}_t \quad \forall t \in \mathcal{T}$$

**Aussprache:** „P-Peak größer gleich p-Netz-t für alle t"

**Bedeutung:** Die Spitzenlast ist das Maximum aller Netzbezüge.

---

### NB9: Energiebilanz am Netzanschlusspunkt

$$p^{Netz}_t + p^{Sp,dis}_t = \sum_{v \in \mathcal{V}} \sum_{l \in \mathcal{L}} p^{ch}_{v,l,t} + p^{Sp,ch}_t \quad \forall t \in \mathcal{T}$$

**Aussprache:** „p-Netz-t plus p-Speicher-discharge-t gleich Summe der Ladeleistungen plus p-Speicher-charge-t für alle t"

**Bedeutung:** Netzbezug plus Speicherentladung deckt Fahrzeugladung plus Speicherladung.

---

### NB10: Speicher-SOC-Bilanz

$$SOC^{Sp}_t = SOC^{Sp}_{t-1} + (\eta \cdot p^{Sp,ch}_t - p^{Sp,dis}_t) \cdot \Delta t \quad \forall t \in \mathcal{T}$$

**Aussprache:** „SOC-Speicher-t gleich SOC-Speicher-t-minus-eins plus Klammer Eta mal p-Speicher-charge-t minus p-Speicher-discharge-t Klammer zu mal Delta-t"

**Bedeutung:** Der Speicher-SOC ergibt sich aus dem vorherigen SOC plus Ladung (mit Wirkungsgrad) minus Entladung.

---

### NB11: Speicher-Leistungsbegrenzung

$$p^{Sp,ch}_t \leq P^{Sp} \quad \forall t \in \mathcal{T}$$
$$p^{Sp,dis}_t \leq P^{Sp} \quad \forall t \in \mathcal{T}$$

**Bedeutung:** Lade- und Entladeleistung sind durch die installierte Speicherleistung begrenzt.

---

### NB12: Speicher-Modus (Exklusives Laden/Entladen)

$$p^{Sp,ch}_t \leq M^{Sp} \cdot \sigma_t \quad \forall t \in \mathcal{T}$$
$$p^{Sp,dis}_t \leq M^{Sp} \cdot (1 - \sigma_t) \quad \forall t \in \mathcal{T}$$

**Bedeutung:** Der Speicher kann zu jedem Zeitpunkt entweder laden ODER entladen, aber nicht beides gleichzeitig.

---

### NB13: Speicher-SOC-Grenzen

$$(1 - DoD) \cdot E^{Sp} \leq SOC^{Sp}_t \leq E^{Sp} \quad \forall t \in \mathcal{T}$$

**Bedeutung:** Der Speicher-SOC muss zwischen Minimum (basierend auf Entladetiefe) und Maximum (Kapazität) liegen.

---

### NB14: Fahrzeug-SOC-Bilanz

$$SOC_{v,t} = SOC_{v,t-1} + \eta^{ch} \cdot \sum_{l \in \mathcal{L}} p^{ch}_{v,l,t} \cdot \Delta t - \sum_{r \in \mathcal{R}(t)} \sum_{f \in \mathcal{F}^E} \phi_{v,r,f} \cdot \kappa^{step}_{r,f}$$

wobei $\kappa^{step}_{r,f} = \frac{\kappa_f}{100} \cdot \frac{d_r}{t^{end}_r - t^{start}_r}$ der Verbrauch pro Zeitschritt ist.

**Aussprache:** „SOC-v-t gleich SOC-v-t-minus-eins plus Eta-charge mal Laden minus Verbrauch"

**Bedeutung:** Der Ladezustand eines Fahrzeugs ergibt sich aus dem vorherigen SOC plus Ladung (mit Wirkungsgrad $\eta^{ch} = 0,95$, d.h. 5% Ladeverluste) minus Energieverbrauch während der Fahrt.

---

### NB15: Fahrzeug-SOC-Grenzen

$$SOC_{v,t} \leq \sum_{f \in \mathcal{F}^E} Q^{max}_f \cdot \tau_{v,f} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

$$SOC_{v,t} \geq \sum_{f \in \mathcal{F}^E} SOC^{min}_f \cdot \tau_{v,f} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

$$SOC_{v,t} \leq M^{SOC} \cdot \epsilon_v \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:** Der SOC muss im zulässigen Bereich des zugewiesenen Fahrzeugtyps liegen (10%-100%); Diesel-Fahrzeuge haben SOC = 0.

---

### NB16: Ladeleistung begrenzt durch Säule

$$p^{ch}_{v,l,t} \leq P^{max}_l \cdot w_{v,l,t} \quad \forall v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}$$

**Bedeutung:** Die Ladeleistung ist durch die Säulenleistung begrenzt und nur möglich wenn der Ladepunkt belegt ist.

---

### NB17: Laden nur für E-Fahrzeuge

$$p^{ch}_{v,l,t} \leq M^{ch} \cdot \epsilon_v \quad \forall v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}$$

**Bedeutung:** Nur E-LKW können laden.

---

### NB18: Gesamtladeleistung pro Fahrzeug

$$\sum_{l \in \mathcal{L}} p^{ch}_{v,l,t} \leq \sum_{f \in \mathcal{F}^E} P^{max,Fzg}_f \cdot \tau_{v,f} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:** Die Gesamtladeleistung eines Fahrzeugs ist durch dessen maximale Ladeleistung begrenzt.

---

### NB19: IsCharging-Verknüpfung (χ-Kopplung)

$$\sum_{l \in \mathcal{L}} p^{ch}_{v,l,t} \leq M^{ch} \cdot \chi_{v,t} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

$$\sum_{l \in \mathcal{L}} p^{ch}_{v,l,t} \geq \varepsilon \cdot \chi_{v,t} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:**

- Wenn $\chi_{v,t} = 1$, dann muss mindestens $\varepsilon$ kW geladen werden
- Wenn $\chi_{v,t} = 0$, dann darf nicht geladen werden
- Koppelt die Binärvariable χ exakt mit der tatsächlichen Ladeleistung

---

### NB20: Ladepunkt-Kapazität

$$\sum_{v \in \mathcal{V}} w_{v,l,t} \leq n^{Spots}_l \cdot y_l \quad \forall l \in \mathcal{L}, t \in \mathcal{T}$$

**Aussprache:** „Summe der Fahrzeuge an Säule l zum Zeitpunkt t kleiner gleich Anzahl Spots mal y-l"

**Bedeutung:** An jeder Ladesäule können maximal so viele Fahrzeuge gleichzeitig angeschlossen sein wie Ladepunkte vorhanden sind (nur wenn Säule installiert).

---

### NB21: Ladesäulen-Gesamtleistung

$$\sum_{v \in \mathcal{V}} p^{ch}_{v,l,t} \leq P^{max}_l \cdot y_l \quad \forall l \in \mathcal{L}, t \in \mathcal{T}$$

**Bedeutung:** Die Gesamtladeleistung an einer Säule ist durch deren maximale Leistung begrenzt.

---

### NB22: On-Route-Verknüpfung

$$\omega_{v,t} \geq x_{v,r} \quad \forall v \in \mathcal{V}, t \in [t^{start}_r, t^{end}_r), r \in \mathcal{R}$$

$$\omega_{v,t} \leq \sum_{r \in \mathcal{R}(t)} x_{v,r} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:** $\omega_{v,t} = 1$ genau dann, wenn Fahrzeug $v$ zum Zeitpunkt $t$ eine Route fährt.

---

### NB23: Nachts angesteckt bleiben (NB_STAY)

$$w_{v,l,t} - w_{v,l,t+1} \leq \omega_{v,t+1} \quad \forall v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}^{Nacht}$$

**Aussprache:** „w-v-l-t minus w-v-l-t-plus-eins kleiner gleich Omega-v-t-plus-eins für alle v, l und Nacht-Zeitschritte t"

**Bedeutung:** Nachts (18:00-06:00) darf ein Fahrzeug einen Ladepunkt NUR verlassen, wenn es im nächsten Zeitschritt auf Route geht ($\omega_{v,t+1}=1$). „Wer nachts ansteckt, bleibt bis zur Abfahrt."

---

### NB24: Ladeunterbrechungsverbot

$$\chi_{v,t} - \chi_{v,t+1} \leq \omega_{v,t+1} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Aussprache:** „Chi-v-t minus Chi-v-t-plus-eins kleiner gleich Omega-v-t-plus-eins"

**Bedeutung:** Ein laufender Ladevorgang darf nicht unterbrochen werden, außer das Fahrzeug fährt auf Route. Verhindert ineffiziente Lade-Pause-Lade-Muster.

---

### NB25: Vollladen-Erkennung

$$SOC_{v,t} \geq Q^{max}_f \cdot \mu_{v,t} - M^{SOC} \cdot (1 - \tau_{v,f}) \quad \forall v \in \mathcal{V}, t \in \mathcal{T}, f \in \mathcal{F}^E$$

$$\mu_{v,t} \leq \epsilon_v \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:** $\mu_{v,t} = 1$ wenn die Batterie voll ist ($SOC = Q^{max}$). Nur E-LKW können vollgeladen sein.

---

### NB26: Nach Vollladen nicht mehr laden

$$\chi_{v,t+1} \leq (1 - \mu_{v,t}) + \omega_{v,t+1} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Aussprache:** „Chi-v-t-plus-eins kleiner gleich eins minus Mü-v-t plus Omega-v-t-plus-eins"

**Bedeutung:** Wenn die Batterie zum Zeitpunkt $t$ voll ist ($\mu_{v,t} = 1$) UND das Fahrzeug nicht auf Route geht ($\omega_{v,t+1} = 0$), dann darf nicht mehr geladen werden ($\chi_{v,t+1} = 0$).

---

### NB27: Kein Säulenwechsel während Laden

$$w_{v,l,t} - w_{v,l,t+1} \leq \omega_{v,t+1} + (1 - \chi_{v,t}) \quad \forall v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}$$

**Bedeutung:** Ein Fahrzeug darf den Ladepunkt nur verlassen, wenn es auf Route geht ($\omega_{v,t+1} = 1$) ODER nicht aktiv lädt ($\chi_{v,t} = 0$). Verhindert Säulenwechsel während des aktiven Ladens.

---

### NB28: Zwangsfreigabe tagsüber bei Vollladung

$$\sum_{l \in \mathcal{L}} w_{v,l,t+1} \leq (1 - \mu_{v,t}) + \omega_{v,t+1} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}^{Tag}$$

**Bedeutung:** Tagsüber (06:00-18:00): Wenn ein LKW vollgeladen ist ($\mu_{v,t}=1$) und keine Route fährt ($\omega_{v,t+1}=0$), muss er den Ladepunkt freigeben. Verhindert Blockierung von Ladepunkten durch vollgeladene Fahrzeuge.

**Hinweis:** Nachts (18:00-06:00) gilt diese Regel NICHT - vollgeladene LKW bleiben am Ladepunkt angesteckt, siehe NB23.

---

### NB29: Kein Laden während Fahrt

$$\sum_{l \in \mathcal{L}} w_{v,l,t} \leq 1 - \omega_{v,t} \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:** Ein Fahrzeug kann nicht an einem Ladepunkt angesteckt sein, wenn es auf Route ist.

---

### NB30: Ein Fahrzeug maximal an einer Säule

$$\sum_{l \in \mathcal{L}} w_{v,l,t} \leq 1 \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Bedeutung:** Ein Fahrzeug kann nicht gleichzeitig an mehreren Ladesäulen angesteckt sein.

---

### NB31: Diesel nicht angesteckt

$$w_{v,l,t} \leq \epsilon_v \quad \forall v \in \mathcal{V}, l \in \mathcal{L}, t \in \mathcal{T}$$

**Bedeutung:** Diesel-Fahrzeuge dürfen nicht an Ladepunkten angesteckt sein.

---

### NB32: Sofortiges Anstecken nachts wenn Laden nötig (NB_SOFORT)

**Hilfsvariable:** $\nu_{v,t}$ (needs*charge) = 1 wenn $SOC*{v,t} <$ maximaler Routenverbrauch

$$SOC_{v,t} \geq \kappa^{max}_f \cdot \tau_{v,f} - M^{SOC} \cdot \nu_{v,t} - M^{SOC} \cdot (1 - \tau_{v,f}) \quad \forall v, t, f \in \mathcal{F}^E$$

$$\nu_{v,t} \leq \epsilon_v \quad \forall v \in \mathcal{V}, t \in \mathcal{T}$$

**Hauptconstraint:**

$$\sum_{l \in \mathcal{L}} w_{v,l,t+1} \geq \omega_{v,t} - \omega_{v,t+1} + \nu_{v,t+1} - 1 \quad \forall v \in \mathcal{V}, t \in \mathcal{T}^{Nacht}$$

**Bedeutung:** Wenn ein E-LKW nachts (18:00-06:00) von einer Route zurückkehrt ($\omega_{v,t}=1, \omega_{v,t+1}=0$) UND geladen werden muss ($\nu_{v,t+1}=1$), dann muss er sofort an einen Ladepunkt angeschlossen werden.

**Hinweis:** Wenn der SOC ausreicht für die längste Route, darf der LKW ohne Ladepunkt zwischengeparkt werden.

---

## 6. Modellgröße

| Komponente                | Anzahl         |
| ------------------------- | -------------- |
| Binärvariablen            | ca. 19.000     |
| Kontinuierliche Variablen | ca. 5.000      |
| **Variablen gesamt**      | **ca. 24.000** |
| **Nebenbedingungen**      | **ca. 75.000** |

---

## 7. Notation Zusammenfassung

| Griechisch    | Name            | Verwendung             |
| ------------- | --------------- | ---------------------- |
| $\tau$        | Tau             | Typzuordnung           |
| $\epsilon$    | Epsilon         | E-LKW-Indikator        |
| $\omega$      | Omega           | On-Route-Indikator     |
| $\chi$        | Chi             | Is-Charging-Indikator  |
| $\gamma$      | Gamma           | Netzerweiterung        |
| $\sigma$      | Sigma           | Speicher-Modus         |
| $\delta$      | Delta           | Diesel-Route           |
| $\phi$        | Phi             | Typ-Route              |
| $\mu$         | Mü              | Volllade-Indikator     |
| $\nu$         | Nü              | Needs-Charge-Indikator |
| $\eta$        | Eta             | Wirkungsgrad           |
| $\kappa$      | Kappa           | Verbrauch              |
| $\alpha$      | Alpha           | OPEX-Rate              |
| $\varepsilon$ | Epsilon (klein) | Minimalladeleistung    |

---

_Dokumentation erstellt für DHBW Fallstudie SCM – Elektrifizierung der Logistik_
_Stand: 03.02.2026_
