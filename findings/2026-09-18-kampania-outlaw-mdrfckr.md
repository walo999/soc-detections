# Kampania Outlaw/Dota — persystencja przez klucz SSH "mdrfckr"

| | |
|---|---|
| Data analizy | 2026-09-18 |
| Okres obserwacji | 2026-09-08 – 2026-09-18 (11 dni) |
| Czujnik | Cowrie 3.0.13, port 22/tcp, VPS OVH (Limburg, DE) |
| Materiał | 353 930 zdarzeń, 1336 unikalnych IP, 46 041 sesji |
| Reguły | 100104 → 100105, 100107 |
| MITRE ATT&CK | T1098.004 — Account Manipulation: SSH Authorized Keys |

## Podsumowanie

Dominującym działaniem po uzyskaniu dostępu do honeypota nie było pobranie
narzędzia z sieci, lecz dopisanie klucza publicznego SSH do pliku
`authorized_keys`. Klucz jest jednoznacznie identyfikowalny — jego suma
SHA-256 odpowiada publicznie opisanemu wskaźnikowi kampanii **Outlaw**
(znanej też jako **Dota**): perlowego botnetu kryptokoparkowego z backdoorem
IRC, propagującego się przez słabe poświadczenia SSH. Wskaźnik jest w obiegu
od lipca 2018 r.

## Obserwacja

Zarejestrowano **232 zapisy** tego samego klucza do plików `authorized_keys`
różnych kont. Cowrie kwalifikuje je jako `cowrie.session.file_download`
z komunikatem *"Saved redir contents"* — jest to przechwycone przekierowanie
wyjścia powłoki, a nie transfer pliku po sieci. Pole `url` pozostaje puste we
wszystkich zdarzeniach z całego okresu.

Publikowany schemat operacji kampanii:

    cd ~ && rm -rf .ssh && mkdir .ssh && echo "ssh-rsa AAAAB3...mdrfckr" >> .ssh/authorized_keys && chmod -R go= ~/.ssh

Istotny jest człon `rm -rf .ssh`: operacja nie tylko ustanawia trwały dostęp,
lecz również usuwa klucze prawowitego właściciela oraz konkurencyjnych
aktorów. W tym samym materiale zaobserwowano też dwukrotny zapis do
`/etc/hosts.deny`, co jest spójne z wypychaniem konkurencji z zajętego hosta.

## Rozkład kont docelowych

| Konto | Liczba zapisów |
|---|---|
| `/root` | 99 |
| `admin` | 7 |
| `ubuntu` | 6 |
| `user` | 3 |
| `testuser`, `rbs`, `mpiuser`, `ftpuser`, `elastic`, `dev`, `deploy`, `alex` | po 2 |
| `zwj`, `es`, `user1` | po 1 |

Rozkład wskazuje na priorytet konta `root` oraz na listę kont typowych dla
środowisk chmurowych i aplikacyjnych, przechodzoną kolejno.

## Wskaźniki (IoC)

**SHA-256 klucza:**

    a8460f446be540410004b1a8db4083773fa46f7fe76fa84219c93daa1669f8f2

**Klucz publiczny:**

    ssh-rsa AAAAB3NzaC1yc2EAAAABJQAAAQEArDp4cun2lhr4KUhBGE7VvAcwdli2a8dbnrTOrbMz1+5O73fcBOx8NVbUT0bUanUV9tJ2/9p7+vD0EpZ3Tz/+0kX34uAx1RV/75GVOmNx+9EuWOnvNoaJe0QXxziIg9eLBHpgLMuakb5+BgTFB+rKJAw9u9FSTDengvS8hX1kNFS4Mjux0hJOK8rvcEmPecjdySYMb66nylAKGwCEE6WEQHmd1mUPgHwGQ0hWCwsQk13yCGPK5w6hYp5zYkFnvlC8hGmd4Ww+u97k6pfTGTUbJk14ujvcD9iUKQTTWYYjIIu5PmUux5bsZ0R4WFwdIe6+i6rBLAsPKgAySVKPRK+oRw== mdrfckr

**Przykładowe adresy źródłowe:** 175.45.204.121, 14.103.107.214,
197.220.92.185, 130.12.180.51.

Ten sam klucz pojawia się z wielu niepowiązanych adresów, co czyni sumę
SHA-256 lepszym korelatorem niż adres IP. Adresy są rotowane, klucz nie.

## Kontekst ruchu

Blok **109.160.32.0/24** odpowiada za 287 979 z 353 930 zdarzeń (81%).
Poszczególne adresy w tym bloku wykazują identyczne liczby zdarzeń
(np. jedenaście adresów po 11 480), co wskazuje na jeden skrypt rozłożony
równomiernie na pulę adresów, a nie na niezależnych aktorów.

## Detekcja

Reguła `100107` dopasowuje sumę kontrolną klucza (poziom 14). Reguła `100105`
pozostaje dopasowaniem ogólnym na zapis do `authorized_keys` (poziom 13) i
obejmie inne kampanie o tym samym wzorcu zachowania. W zebranym materiale
`100105` nie zadziałała samodzielnie, ponieważ wszystkie zapisy pochodziły
z jednej kampanii.

## Znaczenie na hoście produkcyjnym

Na realnym systemie to zdarzenie oznacza pełną kompromitację konta: dostęp
utrzymuje się po zmianie hasła i po ponownym uruchomieniu. Sama zmiana
poświadczeń nie usuwa dostępu — konieczne jest wyczyszczenie `authorized_keys`
oraz weryfikacja, czy nie ustanowiono dodatkowych mechanizmów przetrwania.
Procedurę opisuje playbook PB-003.

## Ograniczenia analizy

- Zdarzenia z flagą `duplicate: true` oznaczają powtórny zapis treści już
  znanej czujnikowi; liczba 232 obejmuje zarówno zapisy pierwotne, jak
  i powtórzenia.
- Nie skorelowano dotąd zapisów klucza z dokładnym ciągiem polecenia
  w `cowrie.command.input` — do wykonania przy kolejnej analizie.
- Honeypot emuluje powłokę, więc dalsze etapy kampanii (pobranie i
  uruchomienie koparki XMRig) nie zostały zaobserwowane. Wniosek o
  charakterze botnetu opiera się na źródłach zewnętrznych, nie na
  materiale własnym.

## Źródła

- Kaspersky Securelist — *Outlaw botnet detected in an incident contained by Kaspersky*: https://securelist.com/outlaw-botnet/116444/
- port22 — *mdrfckrs, part one*: https://blog.port22.dk/mdrfckrs-part-one/
