# Raport końcowy — honeypot SSH soc01

Okres pracy czujnika: 08.09 – 25.11.2026 · przygotowane 25.11.2026, w dniu wyłączenia

Dokument zamyka projekt. Podsumowuje cały zebrany materiał, wynik pracy bez
nadzoru i ograniczenia analizy. Szczegółowe ustalenia są w osobnych plikach
w katalogu `findings/`.

## Streszczenie

- Przez 76 dni od wdrożenia reguł honeypot wygenerował **905 094 alerty
  z 4 385 adresów**. Ruch rósł przez cały okres: średnio ok. 8,5 tys. alertów
  dziennie we wrześniu, 10,9 tys. w październiku i 16,1 tys. w listopadzie.
- Pięć bloków `/24` odpowiada za 85% alertów, w tym jeden — `109.160.32.0/24` —
  za 57,6%.
- Opisane kampanie: Outlaw/Dota (ok. 2 tys. adresów przez cały okres), aktor
  polujący na węzły Solany oraz grupa 16 adresów, która 24.10 przeszła od
  profilowania systemu do przesyłania i uruchamiania własnego pliku.
- Analiza końcowa ujawniła lukę we własnych regułach: przesłania plików przez
  SCP/SFTP nie wywoływały żadnego alertu. Luka została zamknięta przed
  wyłączeniem czujnika (reguła 100109).
- Ostatnie 56 dni maszyna pracowała bez nadzoru. Kontrola łańcucha zakończyła się
  wynikiem OK 1 332 razy na 1 332. Ruch kontrolny dotarł do indeksu w 54 dniach
  na 55 — jedyną awarię wykrył i zgłosił sam mechanizm kontrolny.

## Źródła danych i miary

| Źródło | Zakres | Uwagi |
|---|---|---|
| Alerty w indeksie Wazuha | 10.09 – 25.11.2026 | główna miara raportu; od wdrożenia reguł dla Cowrie, bez własnego ruchu kontrolnego |
| Surowe zdarzenia Cowrie | 25.10 – 25.11.2026 | retencja 30 dni; wcześniejsze migawki w [`docs/historia-projektu.md`](../docs/historia-projektu.md) |
| Dziennik systemu | 08.09 – 25.11.2026 | uruchomienia, logowania, `sudo`, zadania cykliczne |
| Przechwycone pliki | wrzesień – listopad 2026 | 137 plików; do repozytorium trafiają wyłącznie skróty |

Alert to zdarzenie dopasowane przez jedną z reguł: próba logowania, komenda albo
zapis pliku. Połączenia, wymiana kluczy i rozłączenia alertów nie wywołują,
dlatego liczby w tym raporcie są niższe niż liczby surowych zdarzeń z wcześniejszych
migawek i nie należy ich porównywać wprost.

## Skala

| | |
|---|---|
| Alerty | 905 094 |
| Unikalne adresy źródłowe | 4 385 |
| Sesje z co najmniej jednym alertem | ok. 390 tys. (agregacja przybliżona) |
| Przechwycone pliki | 137 |

| Miesiąc | Alerty | Średnio dziennie |
|---|---|---|
| wrzesień (od 10.09) | 174 460 | ok. 8,5 tys. |
| październik | 336 424 | ok. 10,9 tys. |
| listopad (do 25.11) | 394 210 | ok. 16,1 tys. |

## Alerty według reguł

| Reguła | Zdarzenie | Alerty |
|---|---|---|
| 100102 | przyjęte poświadczenia | 386 654 |
| 100108 | rozpoznanie systemu | 296 447 |
| 100103 | wykonanie komendy | 215 541 |
| 100107 | klucz kampanii Outlaw/Dota | 3 970 |
| 100101 | nieudane logowanie | 1 767 |
| 100104 | zapis pliku bez dopasowania szczegółowego | 561 |
| 100106 | materiał do ruchu bocznego | 146 |
| 100105 | inny klucz dopisany do `authorized_keys` | 8 |
| 100109 | plik przesłany przez SCP/SFTP | wdrożona 25.11 |
| 100112 | test autentyczności powłoki | wdrożona 25.11 |

- Przyjętych logowań jest ponad 200 razy więcej niż nieudanych. To potwierdza
  decyzję z 18.09 o obniżeniu poziomu 100102: Cowrie przyjmuje niemal każde
  hasło, więc przyjęte logowanie jest stanem normalnym czujnika.
- Z 4 685 zapisów plików 85% to klucz Outlaw/Dota.
- Reguła 100108 działa od 18.09; wcześniejsze komendy rozpoznania liczą się do 100103.

## Źródła ruchu

| Blok | Udział w alertach |
|---|---|
| `109.160.32.0/24` | 57,6% |
| `195.178.110.0/24` | 8,9% |
| `77.239.124.0/24` | 8,8% |
| `92.118.39.0/24` | 5,3% |
| `2.57.122.0/24` | 4,4% |

W trzech z tych bloków (`195.178.110`, `92.118.39`, `2.57.122`) znajdują się
adresy grupy opisanej w ustaleniu o profilowaniu i dropperze.

## Poświadczenia

| Login | Próby | Hasło | Próby |
|---|---|---|---|
| `root` | 109 890 | `123456` | 17 987 |
| `0` | 15 238 | `0` | 15 518 |
| `admin` | 11 207 | `123` | 8 515 |
| `ubuntu` | 10 533 | `1234` | 6 900 |
| `user` | 5 799 | `12345678` | 4 798 |

Login `claude` jest na 10. miejscu: 2 092 próby ze 138 adresów, równo przez cały
okres (656 / 631 / 805 miesięcznie), najczęściej z dominujących bloków
`109.160.32.0/24` i `77.239.124.0/24`. To nie osobna kampania, tylko element
słownika masowych skanerów — prawdopodobnie sygnał, że nazwy kont narzędzi AI
trafiły już do słowników.

## Kampanie

### Outlaw/Dota

| | Zdarzenia | Adresy | Okres |
|---|---|---|---|
| zdjęcie blokady `.ssh` (`chattr -ia .ssh; lockr -ia .ssh`) | 4 286 | 2 119 | 10.09 – 25.11 |
| zapis klucza `mdrfckr` (reguła 100107) | 3 970 | 1 998 | 18.09¹ – 25.11 |

¹ Data wdrożenia reguły, nie początek kampanii.

Średnio ok. dwa zdarzenia na adres i brak dominujących źródeł — obraz zgodny
z botnetem, który rozprzestrzenia się przez zainfekowane hosty. Szczegóły:
[ustalenie z 18.09](2026-09-18-kampania-outlaw-mdrfckr.md).

### Login `345gs5662d34`: te same hosty, inne sesje

2 993 próby z 1 752 adresów. 1 667 z nich (95%) zapisywało też klucz Outlaw —
na poziomie adresów wygląda to na tę samą kampanię. Na poziomie sesji wspólnych
jest **zero**: ten login i zapis klucza nigdy nie wystąpiły w jednym połączeniu.
Te same zainfekowane hosty prowadzą dwie osobne aktywności, a w listopadzie ich
drogi się rozchodzą: 316 prób `345gs5662d34` wobec 1 316 zapisów klucza.

Korelacja po adresach sugerowała więcej, niż wykazała korelacja po sesjach.

### Od profilowania do droppera

16 adresów w pięciu sieciach. Profilowanie systemu z wykrywaniem honeypota:
10.09 – 24.10. Testy powłoki, przesyłanie pliku i nadawanie mu prawa wykonania:
24.10 – 10.11. Każdy adres drugiej fazy uczestniczył w pierwszej. Szczegóły:
[ustalenie z 25.11](2026-11-25-od-profilowania-do-droppera.md).

### Aktor polujący na węzły Solany

Opisany w [ustaleniu z 10.09](2026-09-10-powracajacy-aktor-solana.md) wraz
z aktualizacją z 18.09.

## Przechwycone pliki

137 plików: 32 we wrześniu, 63 w październiku, 42 w listopadzie.

| Typ | Liczba |
|---|---|
| ELF x86-64 | 65 |
| tekst ASCII | 23 |
| ELF ARM | 21 |
| skrypt bash | 6 |
| ELF i386 | 4 |
| inne, m.in. ELF MIPS | 18 |

Zestaw architektur — x86-64, ARM, MIPS, i386 — jest typowy dla botnetów, które
jedną kampanią celują jednocześnie w serwery i urządzenia sieciowe. Trzy
największe pliki, po ok. 38 MB, to prawdopodobnie warianty jednego programu dla
różnych odmian MIPS, przechwycone tego samego dnia (17.10). Na liście jest też
plik z kluczem kampanii Outlaw (`a8460f44…`).

Pełna lista skrótów:
[`ioc/2026-11-25-przechwycone-pliki-sha256.txt`](ioc/2026-11-25-przechwycone-pliki-sha256.txt).
Żaden plik nie został uruchomiony ani wysłany do zewnętrznego serwisu;
weryfikacja wyłącznie po skrócie. Próbki zostały zniszczone razem z maszyną.

## Praca bez nadzoru: 01.10 – 25.11.2026

| Mechanizm | Wynik |
|---|---|
| Kontrola łańcucha, co godzinę | 1 332 uruchomienia, wszystkie z wynikiem `kontrola: OK` |
| Wartownik stanu systemu | 56 przebiegów, za każdym razem „bez zmian” |
| Ruch kontrolny | dotarł do indeksu w 54 dniach na 55; 04.10 — `brak alertu w indeksie`, wykryte i zgłoszone |
| Utrzymanie i raport dzienny | po 56 uruchomień bez błędu |
| Aktualizacje | 96 aktualizacji pakietów, pięć automatycznych restartów o 04:30 |
| Restarty ręczne | 17.10 i 11.11 — `apt upgrade` z własnego konta tuż przed restartem, potwierdzone dziennikiem `sudo` |
| Dostęp administracyjny | 15 logowań, wszystkie jednym kluczem; zero logowań hasłem |
| CIS Debian 13 | 65% (124 zaliczone / 66 niezaliczonych), bez zmian względem 01.10 mimo 96 aktualizacji |

W całym okresie od 08.09 maszyna uruchamiała się 12 razy. Jedyna dłuższa przerwa
trwała ok. 3 godzin (22.09) — ręczne wyłączenie w czasie awarii sieci u dostawcy.

**Zero błędów nie oznaczało zera awarii.** Licząc po kodach wyjścia, wszystkie
55 uruchomień ruchu kontrolnego zakończyły się sukcesem, bo skrypt kończy się
poprawnie także wtedy, gdy wykryje awarię i wyśle powiadomienie. Awaria z 04.10
była widoczna dopiero w treści dziennika i w braku zdarzenia w indeksie. Tego dnia
kontrola godzinowa przechodziła, więc zawiódł pojedynczy sygnał kontrolny, nie
potok. Przyczyny nie da się już ustalić: surowe logi z tego dnia wyszły poza
retencję.

## Wnioski z całości

1. **Detekcję weryfikuje się na danych, nie na założeniach.** Reguły powstały
   przed zebraniem materiału i wymagały czterech korekt: usunięcia błędnego
   mapowania T1105, dwóch zmian poziomów i reguły potomnej dla rozpoznania.
   Ostatnią lukę — brak alertów na przesłanie pliku — znalazłem dopiero przy
   analizie końcowej.
2. **Brak alertu nie jest brakiem zdarzenia.** Dotyczy to zarówno detekcji
   (uploady niewidoczne dla SIEM-a), jak i nadzoru (kody wyjścia ukrywające
   awarię ruchu kontrolnego).
3. **Korelacja po adresach to nie korelacja po sesjach.** 95% wspólnych adresów
   i zero wspólnych sesji to dwa różne wnioski o tych samych danych.
4. **Praca bez nadzoru wymaga mechanizmów, które same się zgłaszają** — a ich
   cisza jest coś warta dopiero wtedy, gdy da się wykazać, że działały:
   56 meldunków „bez zmian”, 1 332 wyniki `OK`, 54 zdarzenia kontrolne w indeksie.

## Ograniczenia

- Honeypot SSH przyciąga ruch zautomatyzowany; obraz nie jest reprezentatywny
  dla ataków ukierunkowanych.
- Część narzędzi aktywnie sprawdza autentyczność powłoki. Materiał jest
  skrzywiony w stronę tych, które emulacji nie rozpoznają albo nie reagują na wynik.
- Alerty są dostępne od 10.09; pierwsze dwa dni pracy Cowrie nie mają
  odpowiednika w indeksie.
- Uploadów sprzed 25.10 nie da się policzyć: nie było na nie reguły, a surowe
  logi wyszły poza retencję.
- Liczba sesji jest przybliżona.
- Analiza plików ograniczona do metadanych, bez analizy dynamicznej, z założenia.
- Atrybucja wyłącznie behawioralna, bez wskazywania operatorów.

## Zamknięcie projektu

W repozytorium zostają: reguły (zgodne z ostatnią wersją produkcyjną, łącznie
z 100109 i 100112), playbooki, ustalenia, lista skrótów przechwyconych plików
oraz dokumentacja utwardzania i utrzymania.

Razem z maszyną zostają zniszczone: surowe logi, próbki, indeks Wazuha
i poświadczenia usług. Przed przeinstalowaniem odwołane zostają klucz dostępu
do GitHuba, token bota Telegram i kontrola w Healthchecks.
