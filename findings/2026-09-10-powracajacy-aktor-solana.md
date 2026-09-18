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

---

## Aktualizacja — 18.09.2026 (11 dni obserwacji)

Hipoteza z 10.09 potwierdzona na pełnym materiale (08–18.09, 353 930 zdarzeń).
Zmieniają się dwie rzeczy: aktor ma **dwa** adresy, nie jeden, a jego
zachowanie okazuje się węższe, niż zakładano.

### Infrastruktura

| Adres | Okres aktywności | Zdarzenia |
|---|---|---|
| `2.57.122.238` | 09–14.09 | (blok `2.57.122.0/24`: 2 484) |
| `195.178.110.30` | od 13.09 | 13 942 — czwarty najaktywniejszy adres w całym materiale |

Zdarzenia z 08.09 i 10.09 opisane wyżej pochodziły z `2.57.122.238`.
Adresy nakładają się czasowo 13–14.09, co wskazuje na przejęcie zadania przez
nową infrastrukturę, a nie na migrację.

### Poświadczenia (liczby z 11 dni)

| Para | Wystąpienia |
|---|---|
| `solana` / `solana` | 91 |
| `sol` / `sol` | 91 |
| `sol` / `sol123` | 90 |
| `solv` / `solv` | 88 |
| `solv` / `12345678` | 88 |
| `solv` / `123456` | 87 |
| `solana` / `1234567890` | 84 |
| `solana` / `123` | 84 |
| `solv` / `solv123` | 83 |
| `solana` / `12345678` | 83 |
| `sol` / `solana` | 83 |

Do listy dochodzi konto `firedancer` — co najmniej 30 udanych logowań między
09 a 15.09 (wydruk obcięty na 30 pozycjach, liczba rzeczywista jest wyższa).

Nazewnictwo nie pozostawia wątpliwości co do celu: **Firedancer** to klient
walidatora Solany, **solv** to menedżer walidatora. Słownik jest budowany pod
konkretną technologię, nie pod ogólny serwer linuksowy.

### Kadencja

Logowania na konto `firedancer` występują parami w odstępie ok. 27 minut,
a pary powtarzają się co 2–3 godziny, także w nocy. To harmonogram zadania
automatycznego, nie praca operatora.

### Kluczowy wniosek: wyłącznie rekonesans

Oba adresy wykonały **wyłącznie** `/bin/./uname -s -v -n -r -m` — łącznie
1 893 razy. Zero prób persystencji, zero zapisów plików, zero modyfikacji
konfiguracji, zero pobrań. Kontrast z kampanią Outlaw (patrz
`2026-09-18-kampania-outlaw-mdrfckr.md`), która po wejściu natychmiast
ustanawia trwały dostęp, jest całkowity.

Wartość tego findingu leży w tym, czego aktor **nie** zrobił. Zachowanie jest
spójne z dwoma scenariuszami, których na tym materiale nie da się rozstrzygnąć:

1. **Walidacja poświadczeń** — sprawdzenie, które pary login/hasło działają,
   i zapisanie wyniku do późniejszego użycia.
2. **Budowa inwentarza** — ustalenie wersji jądra i architektury, by dobrać
   ładunek przed kolejnym etapem.

### Ograniczenie analizy

Honeypot nie udaje węzła walidatora: nie zawiera plików ani procesów
charakterystycznych dla Solany. Aktor mógł zakwalifikować host jako
nieinteresujący zaraz po odczycie `uname` i z tego powodu nie przejść do
dalszych etapów. Brak obserwacji kolejnej fazy **nie jest** dowodem, że taka
faza w kampanii nie istnieje.

### Konsekwencja dla detekcji

Potwierdza się wniosek z 10.09 o obfuskacji ścieżki. Dodatkowo: 37 579
wykonań `uname -s -v -n -r -m` w całym materiale oznacza, że reguła 100103
(poziom 10) jest przestrzelona — rozpoznanie to stan normalny czujnika
wystawionego do internetu, a nie zdarzenie wymagające eskalacji.
