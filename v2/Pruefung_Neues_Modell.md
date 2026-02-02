# PRÜFUNG: Neues Flottenelektrifizierung-Modell

## Zusammenfassung

| Aspekt | Status | Kommentar |
|--------|--------|-----------|
| NB21 (Ladeunterbrechung) | ⚠️ **GLEICHER BUG** | Prüft nur benachbarte Zeitschritte |
| NB20 (Nacht-Anstecken) | ⚠️ **LÜCKE** | Mitternachts-Übergang fehlt |
| Routen-Daten | ❌ **FALSCH** | Zufällig generiert, nicht Fallstudie |
| Ladesäulen-Anzahl | ❌ **FALSCH** | Binär (0/1), nicht Integer |
| NB24 (Vollladen-Schutz) | ⚠️ **UNVOLLSTÄNDIG** | Erzwingt μ=1 nicht |
| Ladeverluste (η=0.95) | ✅ GUT | Neu und korrekt |
| Periodizität (NB0) | ✅ GUT | SOC[0] = SOC[95] |
| Chi-Verknüpfung (NB14a) | ✅ GUT | Lade-Indikator |

---

## 1. KRITISCH: NB21 hat den gleichen Bug wie unser Modell!

### Code:
```python
# NB21: No charging interruption (extended to full day)
for v in vehicles:
    for t in range(1, n_timesteps - 1):
        # chi[t-1] + chi[t+1] - 1 <= chi[t] + omega[t]
        h.addRow(-highspy.kHighsInf, 1, 4, ...)
```

### Das Problem:
Das Constraint prüft nur **direkt benachbarte** Zeitschritte!

**Beispiel - Ladepause über mehrere Zeitschritte:**
```
t=10: chi=1 (laden)
t=11: chi=0 (pause)
t=12: chi=0 (pause) 
t=13: chi=1 (wieder laden)
```

**Constraint-Prüfung bei t=11:**
```
chi[10] + chi[12] - chi[11] - omega[11] = 1 + 0 - 0 - 0 = 1 ≤ 1 ✓
```

**Constraint-Prüfung bei t=12:**
```
chi[11] + chi[13] - chi[12] - omega[12] = 0 + 1 - 0 - 0 = 1 ≤ 1 ✓
```

**Ergebnis:** Constraint wird erfüllt, obwohl Laden → Pause → Laden stattfindet!

### Fazit:
**Das neue Modell hat EXAKT das gleiche Problem wie unser Modell!**

---

## 2. KRITISCH: NB20 hat Lücke beim Mitternachts-Übergang

### Code:
```python
night_timesteps = list(range(72, 96)) + list(range(0, 24))
# = [72, 73, ..., 95, 0, 1, ..., 23]

for i, t in enumerate(night_timesteps[:-1]):
    t_next = night_timesteps[i + 1]
    if t_next == t + 1:  # ← NUR wenn aufeinanderfolgend!
        # Constraint nur dann erstellt
```

### Das Problem:
Bei `t=95` (23:45) ist `t_next=0` (00:00).

Aber: `0 ≠ 95 + 1 = 96`

→ **Kein Constraint für den Übergang von 23:45 zu 00:00!**

Ein E-LKW könnte um 23:45 angesteckt sein und um 00:00 abstecken - ohne dass es verhindert wird.

---

## 3. KRITISCH: Routen sind ZUFÄLLIG generiert!

### Code:
```python
np.random.seed(42)
route_params = pd.DataFrame({
    'Route': routes,
    'd_r': np.random.randint(150, 400, n_routes),  # ZUFÄLLIG!
    'd_maut_r': np.random.randint(50, 200, n_routes),  # ZUFÄLLIG!
    't_start': np.random.randint(24, 48, n_routes),  # ZUFÄLLIG!
    't_end': np.random.randint(60, 80, n_routes)  # ZUFÄLLIG!
})
```

### Das Problem:
Die Fallstudie definiert **konkrete Routen** mit spezifischen:
- Distanzen (z.B. t-4: 250 km, s-1: 120 km, etc.)
- Maut-Distanzen
- Start- und Endzeiten

Das neue Modell verwendet **zufällige Werte** → Ergebnisse sind nicht mit der Fallstudie vergleichbar!

---

## 4. KRITISCH: Ladesäulen-Anzahl ist BINÄR

### Code:
```python
# y_l: Charging station l is installed
binary_vars['y'] = {}
for l_idx, l in enumerate(station_types):
    var_idx = h.getNumCol()
    h.addVar(0, 1)  # ← BINÄR: nur 0 oder 1!
    h.changeColIntegrality(var_idx, highspy.HighsVarType.kInteger)
    binary_vars['y'][l] = var_idx
```

### Das Problem:
**Fallstudie:** Bis zu 3 Ladesäulen insgesamt, mehrere pro Typ möglich.

**Neues Modell:** Pro Ladesäulen-Typ nur 0 oder 1 möglich.

→ Kann nicht "2x Alpitronic-200" oder "1x Alpi-200 + 2x Alpi-50" modellieren!

---

## 5. PROBLEM: NB24a erzwingt μ=1 nicht

### Code:
```python
# NB24a: Full charge detection
# SOC >= mu * Q_max (when this type is assigned)
h.addRow(0, highspy.kHighsInf, 3,
         np.array([continuous_vars['SOC'][(v, t)], binary_vars['mu'][(v, t)], 
                binary_vars['tau'][(v, f)]], dtype=np.int32),
         np.array([1.0, -q_max_f, 0.0], dtype=np.float64))
```

### Das Problem:
Das Constraint ist: `SOC - μ * Q_max ≥ 0`

Das bedeutet nur: **Wenn μ=1, dann muss SOC ≥ Q_max**

Es fehlt die umgekehrte Richtung: **Wenn SOC = Q_max, dann muss μ=1**

→ Der Solver kann μ=0 wählen auch wenn die Batterie voll ist!
→ NB24b (Lade-Stopp nach Vollladen) greift dann nicht!

### Korrekte Formulierung würde benötigen:
```
SOC ≥ (Q_max - ε) → μ = 1

Umgesetzt als:
SOC ≤ Q_max - ε + M * μ
```

---

## 6. POSITIV: Gute Ergänzungen

### NB0: Periodizität ✅
```python
# SOC[v,0] = SOC[v,95]
h.addRow(0, 0, 2, 
         np.array([continuous_vars['SOC'][(v, 0)], continuous_vars['SOC'][(v, 95)]], ...))
```
→ 24h-Zyklus korrekt

### NB14: Ladeverluste ✅
```python
eta_charging = 0.95  # 95% efficiency
values_list = ... + [-eta_charging * delta_t] * len(charging_indices) + ...
```
→ Realistischer 5% Verlust

### NB14a: Chi-Verknüpfung ✅
```python
# Sum of charging power <= M_ch * chi
# Sum of charging power >= epsilon * chi
```
→ Saubere Lade-Indikator-Variable

---

## 7. Vergleich: Unser Modell vs. Neues Modell

| Feature | Unser Modell | Neues Modell | Wer ist besser? |
|---------|--------------|--------------|-----------------|
| NB_NOBREAK/NB21 | Buggy | **Gleicher Bug** | Keiner |
| Nacht-Anstecken | NB_STAY (korrekt) | NB20 (Lücke!) | **Unser** |
| Routen-Daten | Fallstudie | Zufällig! | **Unser** |
| Ladesäulen-Anzahl | Integer (1-3) | Binär (0/1) | **Unser** |
| Ladeverluste | Nicht | 95% | **Neues** |
| Periodizität | Vorhanden | Vorhanden | Gleich |
| Vollladen-Schutz | Nicht | Unvollständig | Neues (etwas) |

---

## 8. Empfehlung

### Das neue Modell ist NICHT besser als unser Modell!

**Probleme:**
1. Gleicher NB21-Bug wie unser NB_NOBREAK
2. Neue Lücke beim Mitternachts-Übergang
3. Falsche (zufällige) Routen-Daten
4. Falsche Ladesäulen-Modellierung

**Gute Ideen zum Übernehmen:**
1. `eta_charging = 0.95` (Ladeverluste)
2. `chi`-Variable für Lade-Indikator (haben wir als `is_charging`)
3. NB0 Periodizität (haben wir schon)

### Nächste Schritte:
1. **NB_NOBREAK korrekt formulieren** - das ist das Hauptproblem
2. Ladeverluste 95% einbauen
3. Vollladen-Schutz korrekt implementieren (optional)
