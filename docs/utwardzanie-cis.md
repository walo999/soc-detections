# Utwardzanie systemu względem CIS Debian 13 Benchmark

Dokument opisuje, co zostało utwardzone na hoście honeypota, czego **świadomie
nie wdrożyłem i dlaczego**, oraz które kontrole nie mogą zostać zaliczone
z powodu błędów w samej polityce oceny.

Źródło oceny: moduł SCA agenta Wazuha, polityka `cis_debian13`
(207 kontroli, z czego 17 nie dotyczy tej maszyny).

## Wynik

| | Niezaliczone | Zaliczone | Wynik polityki |
|---|---|---|---|
| Przed remediacją (01.10.2026) | 100 | 90 | — |
| Po remediacji (01.10.2026) | **66** | **124** | **65%** |

Domkniętych 34 kontrole. Pozostałe 66 rozkładają się tak:

| Kategoria | Liczba |
|---|---|
| Odrzucone świadomie, z uzasadnieniem | 59 |
| Niemożliwe do zaliczenia (błąd wzorca w polityce) | 7 |

## Co zostało wdrożone

| Obszar | Zmiana |
|---|---|
| Hasła | `pam_pwquality` (złożoność, `retry=3`), `pam_pwhistory` z `remember=24` i `enforce_for_root`, usunięty `nullok` z `pam_unix`, `yescrypt` jako algorytm skrótu |
| Starzenie haseł | `PASS_MAX_DAYS 365`, `PASS_MIN_DAYS 1`, `PASS_WARN_AGE 7`; dni ostrzeżenia dopisane 11 kontom systemowym, które miały puste pole |
| Konta | konto `root` zablokowane (`passwd -S root` = `L`), `su` ograniczone do pustej grupy `sugroup` przez `pam_wheel`, domyślne ustawienia `useradd` |
| Eskalacja uprawnień | `sudo` wymaga hasła (usunięty martwy wpis `NOPASSWD` po cloud-init), `timestamp_timeout=15`, własny dziennik `/var/log/sudo.log` |
| SSH | `AllowUsers debian`, wyłącznie klucze, restrykcyjne szyfry / MAC / KEX, `LoginGraceTime 30`, `MaxStartups 10:30:60`, bez przekierowania agenta i X11, baner przedlogowaniowy |
| Montowania | `/tmp` i `/dev/shm` jako `tmpfs`, `/var/tmp`, `/home` i `/var/log` jako montowania wiązane — wszystkie z `nosuid,nodev` i, gdzie to możliwe, `noexec` |
| Banery | `/etc/issue` i `/etc/issue.net` bez informacji o systemie; `/etc/motd` usunięty |
| Czas | `chrony` z jawną dyrektywą `user _chrony`, synchronizacja potwierdzona (stratum 3) |
| Docker | limity dziennika w `daemon.json`, kontener honeypota z `--cap-drop=ALL`, `--read-only`, `no-new-privileges`, limitami pamięci, PID i CPU |
| Aktualizacje | `unattended-upgrades` z automatycznym restartem o 04:30 i usuwaniem nieużywanych jąder |

Każda zmiana w pliku należącym do pakietu jest zapisana w
`/etc/soc-pliki-zmienione-swiadomie.txt` wraz z uzasadnieniem. Codzienny audyt
porównuje `dpkg --verify` z tą listą, więc każda **inna** podmiana pliku
pakietu zostaje wykryta.

## Kontrole odrzucone świadomie

### 1. auditd i 22 reguły zbierania zdarzeń (33245–33272)

Agent Wazuha czyta już `journald`, pliki konfiguracyjne (FIM) i logi aplikacji,
a własne reguły 100101–100111 opisują to, co faktycznie mnie interesuje.
Drugi demon audytu na maszynie z 2 GB RAM i czterema kontenerami to koszt
bez nowej informacji — te same zdarzenia trafiałyby do dwóch magazynów.

**Kontrola zastępcza:** FIM Wazuha, `journald`, reguły własne, `soc-wartownik.sh`.

### 2. AIDE (33284, 33285)

Integralności plików pilnują już trzy warstwy: FIM Wazuha, `dpkg --verify`
z listą świadomych zmian oraz `soc-wartownik.sh`, który codziennie o 05:00
porównuje siedem obszarów systemu ze stanem wzorcowym. AIDE dodałby czwartą
bazę odniesienia do utrzymania i czwarte źródło fałszywych alarmów.

### 3. rsyslog (33236, 33237) i systemd-journal-remote / upload (33229, 33231)

Logi idą do SIEM-a przez agenta. Drugi demon logów oraz nasłuchujący odbiornik
zdalnych logów to dodatkowa powierzchnia ataku bez odbiorcy po drugiej stronie.

### 4. AppArmor (33039, 33040)

Włączenie obowiązkowej kontroli dostępu pod działającymi kontenerami, bez
środowiska testowego, jest zmianą o wysokim ryzyku przestoju. Zaplanowane jako
osobny etap, poprzedzony migawką maszyny.

### 5. Osobne partycje i opcje montowania dla /var oraz /var/log/audit (33021–33023, 33032–33035)

VPS ma jeden wolumen; wydzielenie partycji wymaga przebudowy układu dysku.
Zrobione zostało wszystko, co dało się bez repartycjonowania — patrz tabela wyżej.

Osobno o `/var` z `nosuid` i `nodev`: `/var/lib/docker` przechowuje warstwy
obrazów kontenerów. Opcja `nosuid` na `/var` odebrałaby bit setuid plikom
**wewnątrz** kontenerów, a `nodev` węzłom urządzeń, które Docker tam tworzy.
To nie jest utwardzenie, to zepsucie działającej usługi.

### 6. Hasło bootloadera (33043)

W modelu VPS dostęp do konsoli KVM i trybu ratunkowego ma każdy, kto ma dostęp
do panelu dostawcy — hasło GRUB-a tego nie zmienia. Zmienia natomiast to, że
błędna konfiguracja `set superusers` blokuje nienadzorowany restart po
aktualizacji jądra, czyli uderza dokładnie w cel pracy bez nadzoru.
Właściwą kontrolą w tym modelu jest zabezpieczenie panelu dostawcy drugim
składnikiem, nie hasło w GRUB-ie.

### 7. Demon cron i uprawnienia jego katalogów (33098, 33099, 33100, 33103)

`cron` nie jest na tej maszynie zainstalowany. Cała automatyzacja działa na
timerach systemd z `Persistent=true`, które nadrabiają termin pominięty przez
wyłączenie maszyny — czego `cron` nie robi. Instalowanie crona tylko po to, by
zaliczyć kontrolę uprawnień jego katalogów, dodaje usługę i nie usuwa żadnego
ryzyka.

### 8. nftables, iptables i dwie kontrole „ufw nieużywany” (33136–33151, 33134, 33145)

Dwanaście kontroli wzajemnie wykluczających się z wybraną zaporą. Benchmark CIS
wymaga wybrania **jednej** z trzech; wybór dowolnej zostawia kontrole dwóch
pozostałych jako niezaliczone. W użyciu jest `ufw`: domyślnie `deny` dla ruchu
wchodzącego i trasowanego, jedna reguła wejściowa (`22/tcp` dla honeypota).

Osobna uwaga, ważniejsza od samego wyniku: **Docker wstawia własne reguły DNAT
w tabeli `nat` i omija `ufw`**. Port kontenera opublikowany przez `-p` jest
dostępny niezależnie od reguł `ufw`. To ograniczenie znane i uwzględnione —
dlatego kontener honeypota jest utwardzony po swojej stronie, a nie zabezpieczony
zaporą hosta.

### 9. ufw: domyślna polityka deny dla ruchu wychodzącego (33132)

Odrzucone po przeliczeniu korzyści, nie z lenistwa.

Ruch wychodzący hosta to `apt`, Telegram, Healthchecks i GitHub. Lista
dozwolonych celów wymagałaby utrzymania, a jej **cichy** błąd kasuje kanał
alarmowy — czyli dokładnie to, co ma działać bez nadzoru.

Dodatkowo nie ograniczyłaby honeypota, bo ruch kontenerów przechodzi przez
łańcuch `FORWARD`, nie `OUTPUT` hosta. Ograniczenie kontenera zrealizowane jest
inaczej: `--cap-drop=ALL`, `--read-only`, `no-new-privileges`, limity pamięci,
PID i CPU.

### 10. sshd DisableForwarding (33161)

Przekierowanie portów TCP jest jedynym kanałem dostępu do panelu Wazuha, który
celowo nie jest wystawiony publicznie. Wyłączone zostało natomiast
przekierowanie agenta SSH i X11 — czyli te formy przekierowania, których
nie używam.

### 11. pam_faillock (33187)

Debian nie dostarcza profilu `pam-auth-update` dla `pam_faillock`; wdrożenie
wymagałoby ręcznej edycji `common-auth`, czyli pliku **generowanego**, na
maszynie gdzie `sudo` jest jedyną drogą do roota (konto `root` zablokowane,
SSH wpuszcza tylko `debian`). Ryzyko trwałego odcięcia przeważa korzyść.

Niezależnie od tego kontrola ma tu niewielki sens: honeypot na porcie 22
przechwytuje masowe próby haseł, a `sshd` na 2222 przyjmuje wyłącznie klucze —
blokada po nieudanych próbach hasła nie ma na czym działać.

### 12. systemd-timesyncd (33094)

Wzajemnie wykluczające się z `chrony`, który jest uruchomiony i zsynchronizowany.

### 13. sudo timestamp_timeout w /etc/sudoers (33181)

Ustawienie jest w `/etc/sudoers.d/99-cis`, co benchmark dopuszcza. Kontrola SCA
wymaga tej samej wartości **dodatkowo** w `/etc/sudoers`. Nie modyfikuję tego
pliku, żeby nie ryzykować pytania `dpkg` o konflikt wersji podczas
nienadzorowanej aktualizacji pakietu `sudo`.

## Kontrole niemożliwe do zaliczenia

Siedem kontroli nie przejdzie niezależnie od konfiguracji, bo wzorce w polityce
SCA są zakodowane błędnie. Zapisuję to, bo rozpoznanie wadliwej kontroli jest
częścią pracy z narzędziem — alternatywą jest psucie konfiguracji, żeby
zadowolić zły wzorzec.

| ID | Czego żąda | Dlaczego nie przejdzie |
|---|---|---|
| 33300 | `^[\w@-]+:x:` w `/etc/passwd` | OS_Regex Wazuha nie rozwija `\w` wewnątrz klasy znaków. Sprawdzone niezależnie: **wszystkie** konta mają `x` w drugim polu |
| 33203 | `pam_pwhistory.so remember enforce_for_root use_authtok` | wzorzec dopuszcza `remember` **bez wartości**; w konfiguracji jest `remember=24`, czyli ostrzej niż domyślne 10 |
| 33202 | jak wyżej, bez `use_authtok` | ta sama przyczyna |
| 33188 | `pam_pwquality.so retry=(d+)` | nawiasy w OS_Regex są znakami dosłownymi, więc wzorzec żąda tekstu `retry=(d+)`. Linia w pliku jest dokładnie taka, jak zaleca CIS |
| 33205 | brak słowa `remember` w plikach `common-*` | szuka słowa w całym pliku, a nie w linii `pam_unix`. Intencja kontroli (`pam_unix` bez `remember`) jest spełniona — `remember` stoi w linii `pam_pwhistory`, gdzie ma stać |
| 33211 | `ENCRYPT_METHOD SHA512` **oraz** `ENCRYPT_METHOD YESCRYPT` | warunek `all` na dwóch wykluczających się wartościach jednego parametru. Ustawiony jest `yescrypt`, mocniejszy z dwóch |
| 33217 | `passwd -S root` zwracające `L` **oraz** `P` | warunek `all` na dwóch wykluczających się stanach konta. Konto `root` jest zablokowane (`L`) |

## Czego ten wynik nie mówi

65% to nie „65% bezpieczeństwa”. Dwadzieścia dwie z 66 niezaliczonych kontroli
to jeden świadomie odrzucony demon (`auditd`), a dwanaście — wybór jednej zapory
z trzech. Benchmark liczy kontrole, nie ryzyko. Wartość tej oceny polega na tym,
że zmusza do przejrzenia każdej pozycji i zapisania decyzji, a nie na samej
liczbie.
