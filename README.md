# Smart EV Charging (PV + HA + go-e + Bluelink)

## Overview

System do inteligentnego ładowania EV:
- Fronius (PV + grid)
- go-e (ładowarka)
- Bluelink (SoC)
- Home Assistant (logika)

Cel:
- max PV
- min grid
- ochrona baterii
- przewidywalne zachowanie

---

## Architektura

Fronius -> HA -> decyzje  
Bluelink -> HA -> SoC  
go-e -> HA -> wykonanie

---

## SoC

- SoC real: Bluelink co 10–15 min
- SoC est: liczony z energii go-e

```
soc_est = soc_anchor + (delta_kWh * eff / 39.2)
```

- anchor = moment ostatniego odczytu z auta

---

## Tryby

### 1. pv_surplus

- ładuj tylko z nadwyżki
- brak importu

```
available = export - buffer
```

- jeśli < min → stop
- jeśli > → dobierz prąd

---

### 2. turbo

- max prąd
- ignoruj PV
- stop przy target

---

### 3. scheduled

#### Parametry:
- target_soc
- safe_soc (np. 80)
- deadline

---

### Strefy

#### < safe_soc
- PV first
- można ładować wcześniej

#### >= safe_soc
- just-in-time
- nie ładować tylko dlatego że jest PV
- cel: osiągnąć target blisko deadline

---

### Planowanie

```
missing_kWh
hours_left
required_kw
required_amp
```

---

### Start

jeśli required_amp < min:
- policz latest_start
- czekaj

jeśli now >= latest_start:
- start

---

### Reguła prądu

```
set_amp = required_amp
```

---

## Harmonogram

np:
- pon 07:00 → 80%
- czw 07:00 → 80%

zawsze bierz najbliższy target

---

## Priorytety

dom -> auto -> bateria -> sieć

---

## Bateria

- 80% ≠ magiczna granica
- ważne: czas przy wysokim SoC

### zakresy:
- 20–70% ideal
- 70–85% ok
- 85–95% umiarkowane
- 95–100% tylko przed jazdą

---

## Pętla sterowania

- co 10–15s decyzja
- co 10–15 min sync z Bluelink

---

## Anti flicker

- start delay ~60s
- stop delay ~60–120s

---

## Zasady

- steruj po export/import, nie po PV
- licz energię z go-e, nie komendy
- nie budź auta bez potrzeby
- nie trzymaj długo 100%

---

## TL;DR

- PV do 80% agresywnie
- powyżej 80% planuj
- turbo gdy trzeba
- SoC = real + estymacja
