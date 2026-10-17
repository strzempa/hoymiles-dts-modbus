# Hoymiles HIT + stick DTS-WL-G3 — część sprzętowa i sieciowa

Jak dostać się lokalnie do falownika **Hoymiles HIT-10L-G3** (hybryda 3-fazowa z baterią) bez chmury Hoymiles. Nic nie trzeba lutować ani dokładać — wystarczy stick, który i tak jest w zestawie, i sieć domowa.

## Co jest potrzebne

- Falownik Hoymiles HIT (tu: HIT-10L-G3, firmware 10502).
- Stick **DTS-WL-G3** (komunikacja Hoymiles) podłączony do sieci domowej — **kablem LAN** (zalecane; Wi-Fi też działa, ale jest mniej stabilne).
- Komputer albo Home Assistant w tej samej sieci (ta sama podsieć albo routing do sticka).
- Aplikacja **S-Miles** — tylko do porównania wartości i do konfiguracji falownika; lokalny odczyt jej nie potrzebuje.

## Co wystawia stick

- **Jedyny otwarty port TCP: 502 (Modbus TCP).** Nie ma lokalnego API HTTP ani MQTT.
- Unit id **1**.
- Dwie przestrzenie rejestrów: **SunSpec** (FC 0x03 od 40000) i **blok producenta** (FC 0x04 od 0x0000) — mapa w [`README.md`](README.md).
- Stick dalej wysyła dane do chmury Hoymiles; lokalny odczyt tego nie zmienia.

## Zachowanie sticka (ważne dla programisty)

| Cecha | Co z tym zrobić |
|---|---|
| **Jeden klient Modbus naraz** | Uruchamiaj jeden program czytający. Dwa naraz = ucięte odpowiedzi i zerwane połączenia u obu. |
| **Zamyka gniazdo po każdej transakcji** | Każde zapytanie: nowe połączenie TCP → zapytanie → odpowiedź → zamknięcie. |
| **Milknie przy złym zakresie** (zamiast wyjątku Modbus) | Timeout ~5 s na transakcję; nie pytaj o zakresy spoza mapy. |
| **Maks. ~100 rejestrów na zapytanie** | Dziel odczyt na kawałki ≤ 100 (blok 0x0000–0x00C1 = 2 zapytania). |
| ~0,5 s na transakcję | Odczyt co 10 s jest bezpieczny; SunSpec model 1 / 702 czytaj raz, limit mocy co minutę. |
| Zerwanie połączenia wygląda różnie (macOS: czyste FIN, Linux: ECONNRESET) | Obsłuż oba jako „utracona próbka”, nie jako awarię programu. |

## Jak znaleźć falownik w sieci

Stick dostaje adres z DHCP, więc adres potrafi się zmienić.

1. **Najlepiej: rezerwacja DHCP** dla sticka na routerze — wtedy wpisujesz stały adres i start jest natychmiastowy.
2. **Po MAC:** prefiks (OUI) Hoymiles to **`48:4D:25`** — szukaj go w tablicy ARP / liście klientów DHCP routera.
3. **Po numerze seryjnym:** zapytaj kandydata o SunSpec model 1 (FC 0x03, od 40000) i porównaj numer seryjny z tabliczką falownika. To jedyna pewna weryfikacja — inne urządzenie z portem 502 (np. inny falownik, konwerter Modbus) też odpowie.
4. Gdy nic nie pomaga: skan podsieci na porcie 502 i sprawdzenie numeru seryjnego u każdego, kto odpowie.

Program w [strzempa/HoymilesMonitor](https://github.com/strzempa/HoymilesMonitor) (`hoymiles_hit.py`) robi kroki 2–4 automatycznie i zapamiętuje ostatni adres.

## Weryfikacja, że działa

1. `nc -vz <adres-sticka> 502` — port otwarty.
2. Odczyt SunSpec: rejestry 40000–40001 muszą dać ASCII `SunS`.
3. Odczyt bloku FC 0x04 od 0x0000 (100 rejestrów) — w dzień 0x001D / 0x0020 / 0x0023 (moc MPPT) powinny być > 0.
4. Porównaj z S-Miles **w tej samej minucie**, najlepiej przy dużym, znanym obciążeniu (np. czajnik, ładowanie auta) — przy 500–900 W różnice łatwo zbyć jako szum.

## Bezpieczeństwo

- Projekt **tylko czyta**. Nie wiadomo, czy stick przyjmuje zapis (SunSpec 704 ma limit mocy) — nie testuj zapisu na działającej instalacji.
- Port 502 nie ma żadnej autoryzacji — **nie wystawiaj go do internetu**. Dostęp zdalny tylko przez VPN (np. Tailscale / WireGuard).
- Zmiany trybu pracy i parametrów baterii (tryb AI, Max. Charging Power, Min. SOC) robi się w S-Miles albo przez serwis Hoymiles, nie lokalnie.

## Co dalej (software)

- Mapa rejestrów: [`README.md`](README.md).
- Home Assistant: [`home-assistant.md`](home-assistant.md).
