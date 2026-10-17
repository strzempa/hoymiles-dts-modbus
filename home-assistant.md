# Hoymiles HIT w Home Assistant — krok po kroku

Jak wpiąć falownik **Hoymiles HIT** (przez stick DTS-WL-G3, Modbus TCP) do Home Assistant tak, żeby działał panel **Energia**, karty przepływu mocy i automatyzacje — bez chmury Hoymiles.

Architektura użyta u mnie:

```
falownik HIT ── stick DTS-WL-G3 ──(Modbus TCP :502)──▶ dodatek „hoymiles2mqtt” (Python)
                                                         │  MQTT (Mosquitto)
                                                         ▼
                                              Home Assistant (MQTT Discovery)
```

Dodatek to lokalny add-on HAOS: co 10 s czyta blok producenta (mapa w [`README.md`](README.md)), spłaszcza wynik do JSON-a, publikuje go do MQTT i ogłasza encje przez **MQTT Discovery** — nic nie trzeba wpisywać do `configuration.yaml`. Logika odczytu jest w `hoymiles_hit.py` z [strzempa/HoymilesMonitor](https://github.com/strzempa/HoymilesMonitor).

Można też użyć gotowej integracji **Modbus** w HA i zdefiniować rejestry z mapy ręcznie — ale trzeba wtedy pilnować limitu 100 rejestrów, znaków (`s16`) i wyliczenia mocy sieci (szablon).

---

## 1. Stick: sieć

1. Podłącz stick **kablem LAN** i zrób **rezerwację DHCP** na routerze (stały adres).
2. Sprawdź port: `nc -vz <adres-sticka> 502`.
3. **Wyłącz wszystkie inne programy czytające Modbus** (logger na komputerze itp.) — stick obsługuje jednego klienta.

Szczegóły: [`hoymiles-hardware.md`](hoymiles-hardware.md).

## 2. Home Assistant: broker MQTT

1. **Ustawienia → Dodatki (Aplikacje) → Sklep** → **Mosquitto broker** → Zainstaluj, włącz *Uruchamiaj przy starcie* i *Watchdog*.
2. **Ustawienia → Urządzenia i usługi**: HA wykryje **MQTT** → Konfiguruj → potwierdź.

## 3. Dodatek odczytu (lokalny)

1. Skopiuj katalog dodatku do `/addons/<nazwa>/` na HAOS (udział Samby `addons`, albo SSH).
2. **Sklep → ⋮ → Sprawdź aktualizacje** → sekcja **Lokalne dodatki** → Zainstaluj → Uruchom.
3. Opcje, które warto mieć:

   | Opcja | Znaczenie |
   |---|---|
   | `host` | adres sticka albo `auto` (szukanie po MAC / numerze seryjnym) |
   | `serial` | numer seryjny falownika do potwierdzenia, że to ten host |
   | `interval` | co ile sekund odczyt (10 s) |
   | `base_topic` | np. `hoymiles` → `hoymiles/state`, `hoymiles/availability` |

4. Dobre zachowanie pętli odczytu (sprawdzone w praktyce):
   - błąd odczytu → **nie nadpisuj** stanu w MQTT (HA trzyma ostatnią dobrą wartość),
   - po 3 błędach z rzędu → `availability = offline` (encje „niedostępne”),
   - po 6 błędach → szukaj sticka od nowa (zmiana adresu, restart sticka).

## 4. Encje, które warto mieć

| Encja (przykład) | Skąd | Uwagi |
|---|---|---|
| moc PV | 0x00BF | + per MPPT 0x001D / 0x0020 / 0x0023 |
| zużycie domu | **0x00B6** (moc czynna) | **nie** liczyć z 0x005C–0x005E jako mocy biernej — patrz ostrzeżenie w README |
| moc AC falownika | suma 0x004A–0x004C | ze znakiem |
| **sieć** | **wyliczona**: AC − dom | dodatnia = oddawanie; rozbić na `pobór` i `oddawanie` (dwie nieujemne encje) |
| moc baterii | 0x00C1 | **+ = rozładowanie, − = ładowanie**; rozbić na `ładowanie` i `rozładowanie` |
| SOC | 0x009F | % |
| napięcia / prądy faz, temperatury | sekcje 3.2–3.4 README, SunSpec 701 | jako encje diagnostyczne |

**Prądy czytaj jako `s16`** — inaczej przy małych ujemnych prądach wyjdzie ~655 A.

## 5. Panel Energia

Falownik nie podaje liczników kWh (albo ich nie znamy), więc energię trzeba **całkować z mocy**:

- metoda trapezów co próbkę, **przerw > 2 min nie zasypywać** (lepiej brak danych niż wymyślone kWh),
- liczniki zapisywać trwale (np. w `/data` dodatku), żeby przeżyły restart,
- encje z `device_class: energy`, `state_class: total_increasing`, jednostka kWh.

| Panel Energia | Encja |
|---|---|
| Sieć → Zużycie z sieci | energia poboru (z wyliczonej mocy sieci < 0) |
| Sieć → Zwrot do sieci | energia oddawania (moc sieci > 0) |
| Panele słoneczne → Produkcja | energia PV |
| Magazyn → Energia do baterii / z baterii | energia ładowania / rozładowania |

Pierwsze słupki pojawią się po pełnej godzinie. Historię z wcześniejszego loggera można wgrać do statystyk HA przez websocket `recorder/import_statistics`.

## 6. Karta przepływu mocy (Power Flow Card Plus, HACS)

Wbudowana karta `energy-distribution` pokazuje **sumy od północy**, nie moc chwilową. Do „na żywo” użyj **Power Flow Card Plus** (HACS → Frontend):

```yaml
type: custom:power-flow-card-plus
entities:
  grid:
    entity:
      consumption: sensor.hoymiles_pobor_z_sieci        # moc poboru (W, ≥ 0)
      production: sensor.hoymiles_oddawanie_do_sieci    # moc oddawania (W, ≥ 0)
  solar:
    entity: sensor.hoymiles_moc_pv
  battery:
    entity:
      # UWAGA: w PFC+ dla BATERII consumption = moc Z baterii (rozładowanie),
      # production = moc DO baterii (ładowanie) — odwrotnie niż intuicja.
      consumption: sensor.hoymiles_moc_rozladowania_baterii
      production: sensor.hoymiles_moc_ladowania_baterii
    state_of_charge: sensor.hoymiles_bateria_soc
  home:
    entity: sensor.hoymiles_zuzycie_domu
    override_state: true   # pokazuj zmierzone zużycie, nie wyliczenie karty
```

Nazwy encji zależą od Twojego dodatku — podmień na swoje.

## 7. Dobre praktyki

- **Jeden klient Modbus.** Dodatek w HA = jedyny czytający.
- **Odrzucaj podejrzane próbki** (sieć 500 V, 24 Hz, prąd 655 A) zamiast je publikować.
- Porównuj z S-Miles **w tej samej minucie** i przy dużym znanym obciążeniu.
- Wyklucz z rekordera encje „śmieciowe” (np. chwilowa sprawność PV→AC) — dużo zapisów, mało wartości.
- **Tryb AI** w S-Miles sam ładuje baterię z sieci (np. w tanich godzinach) i wieczorem sprzedaje ją do sieci. Duży pobór z sieci przy zerowym PV może być planem AI, nie błędem — przy automatyzacjach „nadwyżki PV” licz nadwyżkę jako eksport minus rozładowanie baterii i tylko przy PV > ~300 W.
- Jeśli ładowanie z sieci obciąża fazy za bardzo (bezpiecznik), serwis Hoymiles może zmniejszyć **Max. Charging Power** (lokalnie tego nie ustawisz).

## 8. Problemy

| Objaw | Przyczyna / rozwiązanie |
|---|---|
| ucięte odpowiedzi, losowe błędy | drugi klient Modbus — wyłącz go |
| brak odpowiedzi po zmianie zakresu | stick milknie przy złym zakresie — wróć do mapy, max 100 rejestrów |
| encje „niedostępne” po restarcie routera | stick dostał nowy adres — rezerwacja DHCP albo `host: auto` |
| zużycie domu w nocy za duże, ładowanie auta za małe | liczysz √(S²−Q²) z 0x005C–0x005E — to moc czynna, użyj 0x00B6 |
| prąd ~655 A | prąd czytany bez znaku — użyj `s16` |
| sieć nie zgadza się z licznikiem | sieć jest wyliczana (AC − dom); różnica ~65 W to pobór własny falownika |
