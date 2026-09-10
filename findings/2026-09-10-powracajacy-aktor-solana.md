# Powracający aktor polujący na węzły Solany

**Okres:** 8–10.09.2026 · **Źródło:** honeypot Cowrie, port 22

## Obserwacja
Ten sam adres wrócił po dwóch dobach, z innymi poświadczeniami,
wykonując identyczną komendę rozpoznawczą.

| Data | Login / hasło | Komenda |
|---|---|---|
| 08.09, 18:30 | `sol` / `sol` | `/bin/./uname -s -v -n -r -m` |
| 10.09, 11:49 | `solana` / `1234567890` | `/bin/./uname -s -v -n -r -m` |

## Wnioski
1. To nie jest przypadkowy skan. Powrót po dobach i iterowanie po
   liście kont wskazują na kampanię, nie na jednorazowy strzał.
2. Dobór nazw kont (`sol`, `solana`) sugeruje polowanie na węzły
   walidatorów Solany, a nie na dowolny serwer.
3. Komenda jest identyczna co do znaku — ten sam zestaw narzędzi.

## Znaczenie dla detekcji
Zapis `/bin/./uname` zamiast `/bin/uname` wykonuje ten sam plik, ale
omija regułę dopasowującą dosłowny ciąg `^uname` lub `/bin/uname`.
Detekcja musi opisywać zachowanie, nie ścieżkę.
