# PB-001 — Udane logowanie na honeypocie

**Reguła:** 100102 (poziom 6) · MITRE T1078 Valid Accounts
**Źródło:** Cowrie, `cowrie.login.success`

## Co widzę
Ktoś podał parę poświadczeń, którą honeypot zaakceptował. Cowrie przyjmuje
niemal każde hasło, więc alert nie oznacza kompromitacji — oznacza, że dana
para krąży po sieci i jest aktywnie próbowana.

## Jak potwierdzam
1. Sprawdzam `data.src_ip` — czy adres występował wcześniej i z jakimi loginami.
2. Sprawdzam, czy po logowaniu padły komendy (reguła 100103, ta sama sesja).
3. Odrzucam własny ruch: grupa `canary` albo login zaczynający się od `soctest`.

## Co eskaluję
Sam alert — nie. Eskaluję dopiero wzorzec: ten sam adres wracający po dobach,
poświadczenia wskazujące na konkretną usługę, albo próbę pobrania pliku.
W środowisku produkcyjnym każde użycie tej pary na prawdziwym hoście byłoby
incydentem wysokiej wagi.

## Jak zamykam
Dopisuję parę login/hasło do zestawienia, adres do listy obserwowanych.
Zamykam jako "rozpoznanie, brak działania".

## Znane fałszywe alarmy
Własne testy — odfiltrowane grupą `canary`.
