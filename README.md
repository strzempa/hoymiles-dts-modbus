# Hoymiles HIT + stick DTS — mapa rejestrów Modbus TCP + integracja z Home Assistant

> **EN:** Register map of the **Hoymiles HIT-10L-G3** three-phase hybrid inverter (with battery), read locally over **Modbus TCP** through the **DTS-WL-G3** stick (port 502). Hoymiles publishes no map for the vendor block; everything below was reverse-engineered on one installation and cross-checked against physics (V×I = P per phase, sum of MPPTs = PV total, DC balance with the battery) and against the S-Miles app. **Read-only** — no register is ever written. Docs are in Polish — tables are mostly self-explanatory.

> [!IMPORTANT]
> ## 🔎 SZUKAM INFORMACJI O POZOSTAŁYCH REJESTRACH — POMÓŻ!
>
> Ta mapa powstała metodą prób i obserwacji na jednej instalacji. **Wiele rejestrów jest wciąż nieznanych (❓) albo tylko prawdopodobnych (🟡).** Jeśli wiesz coś więcej, **odezwij się** — każda informacja pomoże mnie i innym użytkownikom falowników Hoymiles.
>
> Szczególnie szukam:
>
> - **liczników energii** (dziennej / całkowitej: PV, sieć, bateria) — dziś energię trzeba całkować z mocy,
> - **pomiaru sieci** (import / eksport na przyłączu) — w bloku nie znalazłem rejestru wymiany z siecią,
> - **temperatur** baterii / radiatora / BMS (kandydaci 0x0024, 0x0067–0x0069, 0x0071–0x0073),
> - znaczenia **0x0060, 0x00A2, 0x00A4, 0x0079–0x007B, 0x00E8–0x00F0** i słów stanu **0x0005, 0x0019, 0x0025, 0x00F2–0x00F4**,
> - **trybu pracy** (np. AI / zużycie własne / TOU) i **parametrów baterii** (Max. Charging Power, Min. SOC) — także, czy da się je zmienić lokalnie,
> - oficjalnej albo nieoficjalnej **dokumentacji Modbus** Hoymiles dla HIT / HYS / HAS i sticka DTS-WL-G3 / DTS-Ultra.
>
> **Jak się odezwać:** załóż [Issue](../../issues) albo napisz na moim profilu GitHub [@strzempa](https://github.com/strzempa). Mile widziane zrzuty z S-Miles z dokładnym czasem, numery rejestrów i wartości, nawet „chyba”.
>
> **EN:** 🔎 **I'm looking for information about the remaining registers!** Especially: energy counters (daily / total), a measured grid exchange register, battery / heatsink / BMS temperatures, the meaning of 0x0060, 0x00A2, 0x00A4, 0x0079–0x007B, 0x00E8–0x00F0 and status words 0x0005, 0x0019, 0x0025, 0x00F2–0x00F4, the working mode (AI / self-consumption / TOU) and battery parameters, and any Hoymiles Modbus documentation for HIT / HYS / HAS inverters and DTS sticks. **If you know anything, please open an [Issue](../../issues) or contact me via my GitHub profile [@strzempa](https://github.com/strzempa).** Thank you!

**Dokumenty:**

- [`README.md`](README.md) — mapa rejestrów (ten plik),
- [`hoymiles-hardware.md`](hoymiles-hardware.md) — stick DTS-WL-G3, sieć, jak znaleźć falownik i jak z nim rozmawiać,
- [`home-assistant.md`](home-assistant.md) — co zrobić, żeby falownik działał w Home Assistant (MQTT, panel Energia, karty).

Kod, z którego pochodzi ta mapa (klient Modbus, odczyt, logger, monitor na macOS/Linux): [strzempa/HoymilesMonitor](https://github.com/strzempa/HoymilesMonitor) — plik `hoymiles_hit.py`. To obserwacje z jednej instalacji, a nie dokumentacja producenta. Stan na **17.10.2026**.

Oznaczenia pewności:

- ✅ **potwierdzone**: sprawdzone fizyką (V×I = P, sumy, bilans DC) albo z aplikacją S-Miles;
- 🟡 **prawdopodobne**: zgadza się z obserwacją, ale nie ma twardego testu;
- ❓ **nieznane**: rejestr się zmienia albo ma stałą wartość, znaczenie nieustalone.

---

## 1. Połączenie

- **Interfejs:** Modbus TCP, port **502** (jedyny otwarty port TCP sticka), unit id **1**.
- **Jeden klient naraz.** Drugi klient (np. drugi skrypt, inny logger) dostaje ucięte odpowiedzi albo zerwane połączenie.
- Stick **zamyka gniazdo po każdej transakcji** — każde zapytanie to nowe połączenie TCP. Na zapytanie o zły zakres **nie odsyła wyjątku Modbus, tylko milknie** — potrzebny krótki timeout (≈ 5 s).
- **Maks. ~100 rejestrów w jednym zapytaniu** — powyżej stick ucina albo się wiesza. Czytać kawałkami.
- Odpowiedź ~0,5 s na transakcję; odczyt co **10 s** jest bezpieczny.
- Szczegóły sprzętu i sieci: [`hoymiles-hardware.md`](hoymiles-hardware.md).

### Dwie przestrzenie rejestrów

| Przestrzeń | Funkcja | Adresy | Co zawiera |
|---|---|---|---|
| **SunSpec** | FC 0x03 (holding) | od **40000** (`"SunS"`) | identyfikacja, tabliczka, status, temperatury, limit mocy |
| **Blok producenta** | FC 0x04 (input) | **0x0000–0x00FF** | wszystkie pomiary na żywo: PV, AC, dom, bateria |

Firmware **10502** wypełnia modele SunSpec 1 / 702 / 703 / 704, ale **pomiary AC w modelu 701 stoją na zerze** — z 701 działają tylko stany i dwie temperatury. Pomiary na żywo są wyłącznie w bloku producenta.

### Typy danych (blok producenta)

| Typ | Opis |
|---|---|
| `u16` | słowo bez znaku |
| `s16` | słowo ze znakiem (U2) — **dotyczy też prądów**: prądy bez znaku dają ~655 A przy małych ujemnych wartościach |
| `V ×0.1` | napięcie w 0,1 V |
| `A ×0.01` | prąd w 0,01 A, **ze znakiem** |
| `W` | moc w W, ze znakiem |
| `Hz ×0.01` | częstotliwość w 0,01 Hz |
| `s32` | para słów: starsze (0 albo −1 = 0xFFFF) + młodsze; zachowuje się jak znak rozszerzony na 32 bity |

**Wartość 5** w wielu nieużywanych rejestrach (0x0018, 0x001A, 0x0027–0x002D, …) wygląda na wypełniacz — rejestr nic nie mierzy.

---

## 2. SunSpec (FC 0x03, od 40000) ✅

Łańcuch modeli zaczyna się od sygnatury `SunS` w 40000–40001; potem pary (id modelu, długość). Ten falownik ogłasza 5 modeli.

| Model | Pole | Offset w modelu | Opis | Pewność |
|---|---|---|---|---|
| 1 | Mn / Md / Opt / Vr / SN | 0 / 16 / 32 / 40 / 48 (ASCII) | producent, model, opcje, firmware, numer seryjny | ✅ |
| 701 | St / InvSt / ConnSt | 1 / 2 / 3 | stan (0 OFF, 1 ON), stan falownika (1 OFF, 2 SLEEPING, 3 RUNNING, 4 THROTTLED, 5 SHUTTING_DOWN, 6 FAULT, 7 STANDBY), połączenie z siecią (1 = tak) | ✅ |
| 701 | TmpAmb / TmpSnk | 33 / 35 (`s16`, °C) | temperatura otoczenia / radiatora | 🟡 (wartości realne, ale „otoczenie” to wnętrze obudowy) |
| 701 | pomiary AC | — | **zawsze 0** w firmware 10502 | ✅ (nie używać) |
| 702 | WMax / VAMax / VArMaxInj | 0 / 5 / 6 | moc znamionowa (W, VA, var) | ✅ |
| 704 | WMaxLimPctEna / WMaxLimPct | 12 / 13 | limit mocy czynnej (włączony, %) | ✅ |
| 704 | WMaxLimPct_SF | 52 | mnożnik skali dla limitu (10^SF) | ✅ |

Offsety liczone od początku danych modelu (adres modelu + 2). Model 1, 702 i łańcuch modeli się nie zmieniają — wystarczy przeczytać raz; limit mocy to ustawienie, wystarczy co minutę.

---

## 3. Blok producenta (FC 0x04, 0x0000–0x00FF)

Mapa z odczytów 14.08–24.09.2026 (≈ 140 tys. zrzutów całego bloku co ~30 s, od nocy przez pełne słońce po ładowanie auta 11,9 kW). Czytane w praktyce: **0x0000–0x00C1** (194 rejestry = 2 transakcje).

### 3.1. PV — trzy trackery MPPT ✅

| Adres | MPPT | Wielkość | Typ |
|---|---|---|---|
| 0x001B / 0x001C / 0x001D | 1 | napięcie / prąd / moc | V ×0.1 / A ×0.01 / W |
| 0x001E / 0x001F / 0x0020 | 2 | napięcie / prąd / moc | V ×0.1 / A ×0.01 / W |
| 0x0021 / 0x0022 / 0x0023 | 3 | napięcie / prąd / moc | V ×0.1 / A ×0.01 / W |
| **0x00BF** | — | **moc PV łącznie** | W (suma MPPT ±12 W) |

Kontrola: V×I = P na każdym trackerze; PV→AC sprawność stabilnie 94,8–95,0 %. Prąd MPPT potrafi być lekko ujemny w nocy (−0,02…−0,31 A) — czytać jako `s16`.

### 3.2. Wyjście AC falownika (3 fazy) ✅

| Adres | L1 | L2 | L3 | Typ |
|---|---|---|---|---|
| napięcie | 0x0043 | 0x0044 | 0x0045 | V ×0.1 |
| prąd | 0x0046 | 0x0047 | 0x0048 | A ×0.01, ze znakiem |
| moc czynna | 0x004A | 0x004B | 0x004C | W, ze znakiem |
| moc bierna | 0x004E | 0x004F | 0x0050 | var, ze znakiem 🟡 |
| moc (kopia, `s32`) | 0x00C2:0x00C3 | 0x00C4:0x00C5 | 0x00C6:0x00C7 | W 🟡 (zgadza się z 0x004A–0x004C, r = 1,00) |
| częstotliwość | 0x0041 | | | Hz ×0.01 |

Moc AC jest **ujemna**, gdy falownik pobiera z sieci (np. ładowanie baterii z sieci). Suma faz = moc wyjściowa falownika.

### 3.3. Sieć — napięcia ✅, wymiana mocy ❌ (nie ma)

| Adres | Wielkość | Typ | Pewność |
|---|---|---|---|
| 0x003E / 0x003F / 0x0040 | napięcie sieci L1 / L2 / L3 | V ×0.1 | ✅ |
| 0x0042 | częstotliwość sieci | Hz ×0.01 | ✅ |
| 0x0090 / 0x0091 / 0x0092 | trzeci zestaw napięć faz | V ×0.1 | 🟡 |
| 0x00D7 / 0x00D8 / 0x00D9 | czwarty zestaw napięć faz | V ×0.1 | 🟡 |
| 0x00DA | częstotliwość (kopia) | Hz ×0.01 | 🟡 |

**Moc wymiany z siecią nie jest mierzona w żadnym znalezionym rejestrze.** Sprawdzone na dwóch punktach pracy różniących się o 2,4 kW PV: żaden rejestr 16- ani 32-bitowy nie śledzi (PV − dom). Trzeba ją **wyliczyć**:

```
sieć [W] = moc AC falownika (0x004A+0x004B+0x004C) − zużycie domu (0x00B6)
          dodatnia = oddawanie do sieci, ujemna = pobór
```

To działa także przy pracy baterii: moc AC jest mierzona po stronie AC, więc zawiera to, co oddaje bateria (bilans DC to potwierdza).

### 3.4. Zużycie domu (obciążenie) ✅

| Adres | L1 | L2 | L3 | Typ |
|---|---|---|---|---|
| napięcie | 0x0051 | 0x0052 | 0x0053 | V ×0.1 |
| prąd | 0x0054 | 0x0055 | 0x0056 | A ×0.01 |
| moc pozorna S | 0x0058 | 0x0059 | 0x005A | VA (S = V×I z dokładnością 0,2 %) |
| **moc czynna P** | **0x005C** | **0x005D** | **0x005E** | **W** |
| **moc czynna łącznie** | **0x00B6** | | | **W** (= suma 0x005C–0x005E w każdej próbce) |

> [!WARNING]
> **0x005C–0x005E to moc CZYNNA, nie bierna.** Łatwo się pomylić (bo leżą po mocy pozornej). Dowody: (a) ich suma = 0x00B6 w każdej próbce, (b) P/S rośnie z obciążeniem jak cos φ: 0,51 przy 0,5–1 kVA (noc, zasilacze), 0,89 przy 2–4 kVA, 0,998 przy 11,9 kVA (ładowarka auta). 0x00B6 zgadza się z „zużyciem” w S-Miles (11 869 W vs 11,89 kW). Traktowanie ich jako mocy biernej i liczenie √(S²−Q²) daje **moc bierną zamiast czynnej** — w nocy zawyża, a ładowanie auta 11,9 kW pokazuje jako ~750 W.

S-Miles nie dolicza do domu ~65 W poboru własnego falownika — drobne różnice z aplikacją są normalne.

### 3.5. Bateria ✅ / 🟡

| Adres | Wielkość | Typ | Pewność |
|---|---|---|---|
| **0x00C1** | **moc baterii (strona DC)** | W, `s16` (z 0x00C0 jako `s32`) | ✅ **dodatnia = rozładowanie, ujemna = ładowanie** |
| 0x0033 | moc baterii (kopia, ±12 W) | W, `s16` (z 0x0032 jako `s32`) | ✅ |
| **0x009F** | **SOC** | % | ✅ (10–100; zgadza się z S-Miles) |
| 0x00B4 | SOC (kopia) | % | ✅ |
| 0x003A / 0x003B | napięcie baterii / szyny DC | V ×0.1 (380–452 V) | 🟡 |
| 0x003C | suma 0x003A + 0x003B (napięcie szyny DC?) | V ×0.1 | 🟡 |
| 0x00D3 / 0x00D4 / 0x00F1 | kolejne kopie napięcia baterii | V ×0.1 | 🟡 |
| 0x0031 | ±15 800, `r = 1,00` z mocą baterii (z 0x0030 jako `s32`) → **prąd baterii**? | ? | ❓ |
| 0x00A4 | 0–8546, rośnie z SOC (r = 0,82) → **energia w baterii [Wh]**? | ? | ❓ |
| 0x002E | 500–571, koreluje z SOC (r = 0,87) | ? | ❓ |
| 0x0024 | 283–389, koreluje z SOC (r = 0,76) → **temperatura baterii ×0,1 °C**? | ? | ❓ |

Jak ustalono moc baterii: zrzut całego bloku przez zachód słońca. 0x00C1 śledzi (AC − PV × 0,95) z r = 0,9999 przez cały wieczorny rozładunek, stoi na 0 przy pełnej i bezczynnej baterii, a stosunek do strony AC jest stały (≈ 92 % — straty DC→AC). Przy ładowaniu jest ujemna, a każdy wzrost SOC w danych przypada na wartość ujemną.

**Pojemności baterii nie ma w żadnym znanym rejestrze** (tu: 16 kWh z tabliczki).

### 3.6. Temperatury i inne kandydaty ❓

| Adres | Zakres | Obserwacja | Hipoteza |
|---|---|---|---|
| 0x0067 / 0x0068 / 0x0069 | 4–47 | wolnozmienne, jak temperatura | temperatury (°C) — moduły / fazy? |
| 0x0071 / 0x0072 / 0x0073 | 33–62 | rosną z PV (r ≈ 0,5–0,7) | temperatury radiatora / IGBT (°C)? |
| 0x006A / 0x006B / 0x006C | −65…+48 | ze znakiem, per faza | ? |
| 0x0060 | 5184–6279 | prawie stałe, lekko z PV (r = 0,71); raz trafiło w moc eksportu — przypadek | ❓ |
| 0x0062 | 6445 / 6451 | dwie wartości | ❓ |
| 0x0064 | 9966–10008 | blisko 10 000 | współczynnik mocy ×10⁻⁴? |
| 0x0074 / 0x00FC | 1195–1216 | wolnozmienne | ❓ |
| 0x0079 / 0x007A / 0x007B | 0–11 741 | schodkowe, mało wartości, rosną w ciągu dnia | **energia dzienna MPPT 1–3 [Wh]**? |
| 0x007D / 0x007E / 0x007F | −378…+354 | ze znakiem, per faza | moc bierna / niezrównoważenie? |
| 0x0080 / 0x0081 / 0x0082 | −35…+39 | ze znakiem, per faza | ❓ |
| 0x00A2 | 0–10 000 | ≈ 0,95 × suma mocy faz, 0 w nocy, sufit 10 000 | moc dostępna / limit? |
| 0x00E8 / 0x00E9 / 0x00EA | 1603–2808 | per faza | ❓ |
| 0x00EB–0x00ED, 0x00EE–0x00F0 | ±160, −523…+556 | per faza, ze znakiem | ❓ |

### 3.7. Słowa stanu i stałe ❓

| Adres | Wartości | Hipoteza |
|---|---|---|
| 0x0005 | 8–13 | stan / tryb pracy |
| 0x000E / 0x000F / 0x0010 | 0 / 1 | flagi |
| 0x0019 | 0–7 | stan falownika (jak InvSt?) |
| 0x0025 | 0–3, r = 0,93 z PV | etap pracy PV (noc / start / produkcja)? |
| 0x00F2 | 1474–1558 (4 wartości) | ❓ |
| 0x00F3 / 0x00F4 | 1–4 | tryb pracy baterii / falownika? |
| 0x0030, 0x0032, 0x006D–0x006F, 0x00AE–0x00B3, 0x00C0–0x00C6 (parzyste) | 0 / −1 | starsze słowa wartości `s32` (znak) |
| 0x0000–0x0007, 0x000D, 0x0066, 0x0097, 0x0098, 0x00C9, 0x00CA | stałe (np. 10300, 10000, 9009, 8720, 10200, 1011, 10106, 10071) | wersje firmware / sprzętu, model? |
| 0x0076–0x0078 = 0x00F5, 0x00FA, 0x00FB | 508 / 149 / 708 (±1) | ❓ |

**Nic nie było zapisywane.** Nie wiadomo, czy blok producenta albo SunSpec 704 przyjmuje zapis przez stick — **nie testować na działającej instalacji bez pewności**.

---

## 4. Ważne pułapki

- **Nie ratuj uciętej odpowiedzi.** Odpowiedź krótsza o parzystą liczbę bajtów rozpakuje się „poprawnie”, a sklejone kawałki przesuną wszystkie dalsze rejestry — wychodzi sieć 500 V albo 24 Hz zamiast błędu. Sprawdzać transaction id, kod funkcji, długość i liczbę bajtów; przy niezgodności **odrzucić próbkę**.
- **Prądy są ze znakiem** (`s16`), także prądy MPPT w nocy.
- **Sieci nie ma w rejestrach** — tylko wyliczenie (sekcja 3.3).
- **Energii (kWh) nie ma w rejestrach** (albo jej nie znaleźliśmy) — liczyć całką z mocy (trapezy, bez łatania przerw > 2 min).
- **Tryb AI falownika** (w S-Miles) sam ładuje baterię z sieci i wieczorem sprzedaje ją do sieci — moc AC i moc baterii mogą być duże przy zerowym PV. To normalne, nie błąd odczytu.
- **Jeden klient Modbus** — wyłącz inne loggery, zanim podłączysz Home Assistant.

---

## 5. Do zrobienia

- Znaleźć liczniki energii (dziennej / całkowitej) — kandydaci 0x0079–0x007B.
- Potwierdzić prąd baterii (0x0030:0x0031), energię w baterii (0x00A4) i temperatury (0x0024, 0x0067–0x0069, 0x0071–0x0073) — porównać z S-Miles w tej samej minucie.
- Rozszyfrować słowa stanu 0x0005, 0x0019, 0x0025, 0x00F2–0x00F4 (zmiana trybu w S-Miles i obserwacja bloku).
- Sprawdzić, czy istnieją rejestry poza 0x00FF i inne przestrzenie (FC 0x03 poniżej 40000).
- Ustalić, czy tryb pracy i parametry baterii (Max. Charging Power, Min. SOC) są w ogóle widoczne lokalnie.

---

## Historia zmian

- **17.10.2026:** pierwsza wersja repo: mapa z `hoymiles_hit.py` (HoymilesMonitor) i dodatku `hoymiles2mqtt` 1.1.3; statystyki całego bloku 0x0000–0x00FF z ≈ 35 tys. zrzutów (14.08–24.09.2026) → tabele kandydatów ❓.
- **16.10.2026:** prądy czytane jako `s16` (wcześniej ~655 A przy małych ujemnych prądach).
- **24.09.2026:** 0x005C–0x005E rozpoznane jako moc czynna domu (wcześniej błędnie jako bierna).

---

## Źródła i podziękowania

- Specyfikacja **SunSpec** (modele 1, 701–704) — sunspec.org.
- Aplikacja **S-Miles** (Hoymiles) — porównanie wartości w tej samej minucie.
- Własne obserwacje: zrzuty całego bloku co ~30 s od 14.08.2026, testy przy znanych obciążeniach (ładowanie auta 11,9 kW), bilans DC baterii przez zachód słońca.

Rejestry pochodzą z jednej instalacji: **HIT-10L-G3** (firmware 10502), stick **DTS-WL-G3** podłączony kablem, 3 trackery MPPT, bateria 16 kWh. Inne modele i wersje firmware mogą się różnić. Korzystasz na własne ryzyko; projekt niczego nie zapisuje do falownika.
