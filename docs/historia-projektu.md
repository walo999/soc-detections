# Historia projektu

Plik archiwalny. Trzyma migawki danych, które w `README.md` zostały zastąpione
nowszymi, oraz kalendarium etapów — żeby główny opis pozostał aktualny,
a przebieg pracy nie zniknął.

## Kalendarium

| Data | Etap |
|---|---|
| 06.09.2026 | Uruchomienie VPS, podstawowa konfiguracja, przeniesienie `sshd` na port 2222 |
| 08.09.2026 | Start honeypota Cowrie na porcie 22. Pierwsze zdarzenie: 08.09, 20:17 |
| 08–09.09.2026 | Wazuh (manager, indexer, dashboard) w kontenerach, agent na hoście, reguły własne 100101–100104 |
| 10.09.2026 | Pierwsze ustalenie: powracający aktor polujący na węzły Solany |
| 18.09.2026 | Analiza 11 dni ruchu. Kampania Outlaw/Dota, reguły 100105–100107, korekta błędnego mapowania ATT&CK, reguła 100108 |
| wrzesień 2026 | Automatyzacja: timery systemd, raport dzienny na Telegram, czujnik martwego człowieka, wartownik stanu systemu, ruch kontrolny (canary) |
| wrzesień 2026 | Audyt powłamaniowy i ocena możliwości włamania — 12 punktów kontrolnych, bez śladów dostępu osób trzecich |
| 01.10.2026 | Utwardzanie względem CIS Debian 13, jeden skrypt audytu całości, przegląd repozytorium |

## Archiwum: migawki skali obserwacji

### 18.09.2026 — po 11 dniach pracy ciągłej

| | |
|---|---|
| Zdarzenia | 353 930 |
| Unikalne adresy źródłowe | 1 336 |
| Sesje | 46 041 |
| Udział bloku `109.160.32.0/24` | 81% |

Migawka aktualna znajduje się w `README.md`.

## Wnioski metodyczne

Zapisuję je, bo każdy wynikł z konkretnego błędu, który popełniłem przy tej
maszynie, i każdy zmienił sposób, w jaki weryfikuję późniejsze zmiany.

**Pusty wynik nie jest zaliczoną kontrolą.** Polecenie, które nic nie wypisało,
równie dobrze może oznaczać brak problemu, jak i zły wzorzec, złą ścieżkę albo
błąd składni wyciszony przekierowaniem. Kontrola jest warta zaufania dopiero
wtedy, gdy widziałem ją **sfałszowaną** — celowo usunąłem plik z listy wyjątków
i sprawdziłem, że audyt go zgłasza.

**Testuję, czy stare przestało działać, nie czy nowe działa.** Po wyłączeniu
`nullok` sprawdzenie, że logowanie hasłem działa, nie dowodzi niczego. Dowodem
jest brak `nullok` w wygenerowanym stosie PAM i to, że konto bez hasła nie wchodzi.

**Pliku zapasowego nie odkłada się w katalogu, który jest skanowany.** Kopia
`unix.przed-cis` zapisana w `/usr/share/pam-configs/` została przez
`pam-auth-update` wczytana jako **drugi profil** i zdublowała wpisy `pam_unix`.
Ten sam błąd popełniłem wcześniej w `sshd_config.d/`. Kopie leżą teraz w `/root`.

**Szukam po starej wartości, nie po nazwie parametru.** Po zmianie hasła do API
Wazuha panel przestał się łączyć. Szukałem wpisów `API_PASSWORD` i nie znalazłem
nic, bo stare hasło było wpisane pod kluczem `password:` w innym pliku. Należało
szukać samego hasła.

**Narzędzie oceny też może się mylić.** Siedem kontroli CIS w polityce SCA ma
wzorce, których nie da się spełnić poprawną konfiguracją. Wykryłem to, czytając
wzorzec każdej niezaliczonej kontroli i porównując go z faktycznym stanem pliku,
zamiast „naprawiać” konfigurację pod czerwony wynik.
