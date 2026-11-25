# Od profilowania do droppera: jedna grupa adresów, dwie fazy

Okres: 10.09 – 10.11.2026 · źródła: alerty Wazuha, surowe logi Cowrie z ostatnich 30 dni · przygotowane 25.11.2026

## Streszczenie

Szesnaście adresów z pięciu sieci przez sześć tygodni uruchamiało na honeypocie
skrypt profilujący, który oprócz parametrów systemu zbierał informacje o karcie
graficznej i sprawdzał, czy powłoka zachowuje się jak prawdziwa. 24.10 skrypt
zniknął, a tego samego dnia te same adresy zaczęły wykonywać rozbudowaną baterię
testów powłoki, przesyłać plik wykonywalny przez SCP/SFTP i nadawać mu prawo
wykonania. Każdy z 16 adresów drugiej fazy uczestniczył wcześniej w pierwszej.

Analiza ujawniła lukę we własnych regułach: przesłania plików kanałem SCP/SFTP
nie wywoływały żadnego alertu. Luka została zamknięta przed wyłączeniem czujnika.

## Dowody powiązania

| | Faza 1: profilowanie | Faza 2: dropper |
|---|---|---|
| Sygnatura | blok `===SHELL_BEHAVIOR===` w poleceniu | testy `check_<32 hex>_…`, upload, `chmod +x` |
| Okres | 10.09 – 24.10.2026 | 24.10 – 10.11.2026 |
| Zdarzenia z sygnaturą | 3 904 | 2 014 |
| Adresy | 18 | 16 |

**16 z 16** adresów fazy 2 występuje w fazie 1, a przejście nastąpiło jednego dnia.
Najaktywniejsze w obu fazach: `2.57.122.150` (378 i 294 zdarzenia)
oraz `195.178.110.217` (303 i 194).

| Sieć | Adresy grupy |
|---|---|
| `2.57.122.0/24` | .150, .168, .209 |
| `80.94.92.0/24` | .55, .179, .234 |
| `92.118.39.0/24` | .14, .49, .50, .71, .77 |
| `193.32.162.0/24` | .84 |
| `195.178.110.0/24` | .217, .227, .228, .232 |

## Faza 1: profilowanie

Jedno wieloliniowe polecenie, które:

- ustawia pełną ścieżkę `PATH` i dla każdej informacji próbuje kilku wariantów
  (`uname`, `/bin/uname`, `/usr/bin/uname`, `busybox uname`) — jest przygotowane
  także na systemy okrojone, w tym urządzenia z BusyBoxem;
- zbiera wersję jądra, architekturę, czas pracy, liczbę i model procesorów;
- **szuka karty graficznej** (`lspci` filtrowane po `vga` i `nvidia`);
- odczytuje historię logowań (`last`);
- w bloku `===SHELL_BEHAVIOR===` uruchamia nieistniejący plik (`./xxxxxx`)
  i zapisuje komunikat błędu — emulowana powłoka często zwraca go inaczej niż prawdziwa;
- zwraca wynik w formacie `KLUCZ:wartość` (`UNAME:`, `ARCH:`, `CPUS:`, `GPU:`,
  `FILTER:`), gotowym do automatycznego przetwarzania.

Zainteresowanie GPU i liczbą procesorów wskazuje na selekcję maszyn pod kątem
mocy obliczeniowej.

## Faza 2: test powłoki, upload, wykonanie

Przebieg typowej sesji, odtworzony z surowych logów — wszystko w ramach jednego
połączenia SSH, w osobnych kanałach:

| Etap | Co się dzieje |
|---|---|
| 1 | przyjęte logowanie |
| 2 | w ciągu kilku sekund kilkadziesiąt poleceń testowych |
| 3 | zamknięcie kanału |
| 4 | po ok. 1–1,5 minuty: przesłanie pliku `.16` kanałem SCP/SFTP (`cowrie.session.file_upload`) |
| 5 | w tej samej sekundzie: `echo '111111' \| sudo -S sh -c 'chmod +x …'`, a jako rezerwa zwykłe `chmod +x '/root/.16'` |

Każda seria testów ma własny identyfikator `check_<32 znaki hex>_`:

| Test | Polecenie (skrót) | Co sprawdza |
|---|---|---|
| `small_control` | `printf` ze znacznikiem | czy wyjście w ogóle wraca |
| `small_math` | `"$((706*297+12))"` | czy powłoka liczy wyrażenia arytmetyczne |
| `small_pid` | `"$$"` | czy istnieje prawdziwy identyfikator procesu |
| argumenty | `set -- …`, `"$#"` | parametry pozycyjne |
| `small_function` | definicja i wywołanie funkcji | obsługa funkcji |
| zagnieżdżenie | `sh -c '…'` z wielokrotnym cytowaniem | cytowanie w podpowłoce |
| `transport_done` | `echo; echo '…'` | znacznik końca serii |

Serie powtarzają się w wariantach z `printf` i `echo`, z `sh -c` i bez. Odpowiedź
na każdy test jest znana z góry (706 × 297 + 12 = 209 694), więc porównanie
wyniku rozstrzyga, czy po drugiej stronie jest prawdziwa powłoka.

**Testy nie zatrzymały ataku.** W 30 dniach objętych surowymi logami plik
przesłano 1 176 razy. Narzędzie albo nie wykryło emulacji, albo jego wynik
nie wstrzymywał dalszych kroków.

## Ładunek

| SHA-256 (początek) | Nazwa | Rozmiar | Przesłań | Pierwsze przechwycenie |
|---|---|---|---|---|
| `a151d3f4f2422531…` | `.16` | 3,2 MB | 1 159 | 25.10.2026 |
| co najmniej siedem rzadszych wariantów | `.16`, `.b0s` | 0,4 – 2,8 MB | po 1–2 | 25.10 – 08.11.2026 |

Wszystkie to statycznie linkowane pliki ELF x86-64. Pełne skróty:
[`ioc/2026-11-25-przechwycone-pliki-sha256.txt`](ioc/2026-11-25-przechwycone-pliki-sha256.txt).
Pliki nie były uruchamiane ani wysyłane do zewnętrznych serwisów.

Lokalizacje docelowe: `/root/.16` (1 002 razy), `/dev/shm/.16` (156), pojedynczo
`/var/backups/.16` i `/root/.b0s`. Wariant z `/dev/shm` to dokładnie scenariusz,
któremu zapobiega opcja `noexec` na `/dev/shm`, wdrożona na tym hoście 01.10
(patrz [`docs/utwardzanie-cis.md`](../docs/utwardzanie-cis.md)).

## Luka w detekcji i jej zamknięcie

Reguła 100104 dopasowywała wyłącznie `cowrie.session.file_download`, czyli
pobrania i przechwycone przekierowania wyjścia. Przesłanie pliku przez SCP/SFTP
Cowrie zapisuje jako osobny typ zdarzenia, `cowrie.session.file_upload`, którego
żadna reguła nie opisywała.

| | |
|---|---|
| Alerty `file_upload` w indeksie, 10.09 – 25.11 | **0** |
| Zdarzenia `file_upload` w surowych logach, 25.10 – 25.11 | 1 274 |
| z tego z 16 adresów grupy | 1 176 (92%) |

Skali luki sprzed 25.10 nie da się już ustalić — surowe logi wyszły poza
retencję. Same pliki nie przepadły: Cowrie zapisywał je w katalogu
przechwyconych plików, więc ich skróty są na liście IoC.

Dodane reguły, sprawdzone `wazuh-logtest` na prawdziwych zdarzeniach z tej
kampanii i wdrożone:

| ID | Poziom | Zdarzenie | MITRE |
|---|---|---|---|
| 100109 | 12 | plik przesłany kanałem SCP/SFTP | T1105 |
| 100112 | 5 | test autentyczności powłoki (`check_<32 hex>_`, `===SHELL_BEHAVIOR===`) | T1497.001 |

Test objął też cztery przypadki kontrolne — pobranie pliku, czyste rozpoznanie,
zwykłą komendę i blok z fazy 1 — żeby potwierdzić, że istniejące reguły
zachowują się bez zmian. Po restarcie managera liczba załadowanych reguł wzrosła
dokładnie o dwie.

100112 ma celowo niski poziom: jedna sesja generuje kilkadziesiąt poleceń
testowych, a właściwym sygnałem w tym łańcuchu jest upload.

Mapowanie T1105 wraca do reguł po tym, jak 18.09 usunąłem je z 100104 — wtedy
zasadnie, bo pobrań z sieci nie było. Transfer narzędzi był, tylko innym kanałem.

Reguły wdrożono w dniu wyłączenia czujnika, więc **nie zostały potwierdzone na
żywym ruchu** — wyłącznie w `wazuh-logtest`.

## Wnioski

1. **Rozpoznanie poprzedziło instalację o sześć tygodni.** W prawdziwym
   środowisku alert na fazę 1 byłby oknem wczesnego ostrzeżenia, a korelacja po
   adresach — sposobem na powiązanie późniejszego incydentu z wcześniejszym
   rozpoznaniem.
2. **Brak alertu nie oznaczał braku zdarzenia.** Luka wyszła na jaw dopiero przy
   pytaniu, dlaczego `chmod` dotyczy pliku, którego zapisu nie ma w indeksie.
   Reguła opisująca „zapis pliku” obejmowała tylko jeden z dwóch typów zdarzeń,
   których Cowrie do tego używa.
3. **Dane z honeypota są skrzywione.** Narzędzia aktywnie sprawdzają autentyczność
   powłoki; te, które skutecznie rozpoznają emulację, mogą nie pokazywać właściwej
   fazy ataku.

## Ograniczenia

- Atrybucja wyłącznie behawioralna. Wspólne adresy nie przesądzają o jednym
  operatorze — mogą to być np. wynajmowane serwery tego samego dostawcy — ale
  16 z 16 i przejście jednego dnia czynią to najprostszym wyjaśnieniem.
- Faza 1 jest opisana na podstawie alertów z indeksu; jej surowe logi wyszły
  poza retencję.
- Analiza ładunku ograniczona do metadanych: typ, rozmiar, skrót, data.
  Bez analizy dynamicznej, z założenia.
