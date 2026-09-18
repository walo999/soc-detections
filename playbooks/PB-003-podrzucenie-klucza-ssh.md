# PB-003 — Podrzucenie klucza SSH do authorized_keys

**Reguły:** 100105 (poziom 13), 100107 (poziom 14) · MITRE T1098.004 Account Manipulation: SSH Authorized Keys
**Źródło:** Cowrie, `cowrie.session.file_download`, pole `destfile`

## Co widzę
Ktoś zapisał klucz publiczny SSH do pliku `authorized_keys`. Na honeypocie
zapis jest emulowany i host pozostaje nietknięty — alert mówi o zamiarze
napastnika, nie o stanie maszyny. Na realnym systemie to samo zdarzenie
oznacza trwały dostęp, który przetrwa zmianę hasła i restart.

Reguła 100107 zawęża do konkretnego klucza kampanii Outlaw/Dota
(SHA-256 `a8460f44…`, komentarz `mdrfckr`). Reguła 100105 łapie ten sam
wzorzec zachowania dla dowolnego innego klucza.

## Jak potwierdzam
1. Odczytuję `data.shasum`. Jeśli to `a8460f44…`, mam do czynienia ze znanym
   wskaźnikiem — sprawdzam w zestawieniu, nie w VirusTotal przez wgranie pliku.
2. Odczytuję `data.destfile` — które konto było celem. `/root` podnosi wagę.
3. Sprawdzam sesję wstecz: czy poprzedziło ją `login.success` (100102) i jakie
   poświadczenia zadziałały.
4. Sprawdzam, czy ten sam `shasum` występował z innych adresów — klucz jest
   lepszym korelatorem niż IP, bo adresy są rotowane, a klucz nie.
5. Odrzucam własny ruch: grupa `canary`.

## Co eskaluję
Każde zadziałanie 100107 — to potwierdzony wskaźnik znanej kampanii, nie
hipoteza. Na honeypocie eskalacja oznacza wpis do zestawienia; na hoście
produkcyjnym byłby to incydent krytyczny z natychmiastowym odcięciem maszyny.

## Kroki na hoście produkcyjnym
Kolejność ma znaczenie — odwrócenie jej pozwala napastnikowi wrócić:

1. Odciąć host od sieci lub zablokować 22/tcp poza znanym adresem administracyjnym.
2. Zebrać materiał **przed** czyszczeniem: `authorized_keys` wszystkich kont,
   `last`, `/var/log/auth.log`, lista procesów, `crontab -l` dla każdego konta.
3. Usunąć obce wpisy z `authorized_keys` — wszystkich kont, nie tylko tego
   z alertu. Kampania iteruje po liście kont.
4. Sprawdzić pozostałe mechanizmy przetrwania: cron, jednostki systemd,
   `~/.bashrc`, `~/.profile`, `/etc/rc.local`, nowe konta w `/etc/passwd`.
5. Dopiero teraz zmienić poświadczenia i klucze administracyjne.
6. Ocenić, czy host był używany jako skocznia — sprawdzić wychodzące połączenia
   SSH i obecność cudzych kluczy prywatnych w `/tmp`.

**Sama zmiana hasła niczego nie naprawia.** Dostęp po kluczu działa niezależnie
od haseł — to najczęstszy błąd przy obsłudze tego zdarzenia.

## Jak zamykam
Dopisuję `shasum` i adresy źródłowe do zestawienia wskaźników. Jeśli klucz jest
nowy (zadziałała 100105, nie 100107), zakładam osobny finding i rozważam regułę
zawężającą na jego sumę kontrolną.

## Znane fałszywe alarmy
Na honeypocie brak. Na hoście produkcyjnym: zautomatyzowane wdrożenia i
narzędzia zarządzania konfiguracją legalnie dopisują klucze do `authorized_keys`
— przed eskalacją potwierdzam, czy zapis nie pochodzi z własnego procesu CI/CD.
