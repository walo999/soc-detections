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
| 01.10 – 25.11.2026 | 56 dni pracy bez nadzoru; pięć automatycznych restartów po aktualizacjach, dwie ręczne aktualizacje (17.10, 11.11) |
| 04.10.2026 | Jedyna awaria ruchu kontrolnego: zdarzenie nie dotarło do indeksu, wykryte i zgłoszone przez `soc-canary.sh` |
| 24.10.2026 | Grupa 16 adresów przechodzi od profilowania systemu do przesyłania i uruchamiania pliku `.16` |
| 25.11.2026 | Analiza końcowa: luka w detekcji uploadów SCP/SFTP, reguły 100109 i 100112, raport końcowy, lista IoC; przeinstalowanie maszyny |

## Archiwum: migawki skali obserwacji

### 18.09.2026 — po 11 dniach pracy ciągłej

| | |
|---|---|
| Zdarzenia | 353 930 |
| Unikalne adresy źródłowe | 1 336 |
| Sesje | 46 041 |
| Udział bloku `109.160.32.0/24` | 81% |

### 01.10.2026 — po 23 dniach

| | |
|---|---|
| Zdarzenia | 773 478 |
| Unikalne adresy źródłowe | 2 987 |
| Sesje | 100 313 |
| Udział bloku `109.160.32.0/24` | 80% |

Obie migawki liczą surowe zdarzenia Cowrie. Stan końcowy z 25.11, w `README.md`
i w raporcie końcowym, liczy alerty z indeksu Wazuha — to inna miara i nie należy
jej porównywać wprost z powyższymi.

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

**Kod wyjścia nie jest wynikiem kontroli.** Ruch kontrolny miał 55 udanych
uruchomień na 55, a jedno z nich zgłosiło awarię — skrypt kończy się poprawnie
także wtedy, gdy wykryje problem i wyśle powiadomienie. Liczenie sukcesów po
kodach wyjścia dałoby obraz bezbłędny i fałszywy.

**Reguła opisuje typ zdarzenia, nie intencję.** Reguła „zapis pliku” obejmowała
jeden typ zdarzenia Cowrie (`file_download`), a przesłanie przez SCP/SFTP to drugi
(`file_upload`). Przez cały projekt uploady nie wywołały ani jednego alertu.
Wyszło to dopiero przy pytaniu, dlaczego `chmod` dotyczy pliku, którego zapisu
nie ma w indeksie.

**Korelacja po adresach to nie korelacja po sesjach.** 95% wspólnych adresów
dwóch aktywności i zero wspólnych sesji to dwa różne wnioski o tych samych danych.
