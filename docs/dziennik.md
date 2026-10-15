# Dziennik prac

## 2026-10-14: Etap 3 (część 1)

Co zrobiłem
Na podstawie tabeli wymagań DNS z dokumentacji OKD 4.19 i planu adresacji rozpisałem rekordy dla klastra `okdupiklaster.lab.okd`:

| Nazwa | Adres | PTR |
|---|---|---|
| ha1 | 172.18.22.61 | nie |
| ha2 | 172.18.22.62 | nie |
| api | 172.18.22.63 (VIP) | tak |
| api-int | 172.18.22.63 (VIP) | tak |
| *.apps | 172.18.22.63 (VIP) | nie |
| bootstrap | 172.18.22.64 | tak |
| cp0 | 172.18.22.65 | tak |
| cp1 | 172.18.22.66 | tak |
| cp2 | 172.18.22.67 | tak |
| bastion | 172.18.22.68 | nie |
| dns | 172.18.22.69 | nie |

**Co nie zadziałało i dlaczego**
- W pierwszej wersji `api` i `api-int` wskazywały na adresy pojedynczych HAProxy (.61 i .62). Awaria jednego load balancera odcięłaby dostęp do API. Poprawka: oba rekordy wskazują na VIP, który keepalived przenosi na sprawny HAProxy.
- Brakowało rekordu `*.apps`, przez który idzie ruch do aplikacji w klastrze, m.in. do konsoli webowej.
- Węzły nazwałem najpierw `comp0` do `comp2`, co sugerowało workery (compute). Zmieniłem na `cp0` do `cp2`, zgodnie z rolą control plane.
- Literówka `apt-int` zamiast `api-int`. Taki błąd zatrzymałby bootstrap, bo węzły nie znalazłyby wewnętrznego API.

**Czego się nauczyłem**
- `api`, `api-int` i `*.apps` to rekordy usług, nie maszyn. Wszystkie wskazują na load balancer (VIP), a nie na konkretny węzeł.
- PTR jest wymagany dla `api`, `api-int`, bootstrapu i węzłów klastra. Bastion i serwer DNS go nie potrzebują.
- W klastrze kompaktowym nie ma osobnych workerów, więc nie ma rekordów compute.
- Każdy rekord trzeba zweryfikować `dig`, bo literówka w nazwie jest łatwa do przeoczenia, a jej skutki trudne do zdiagnozowania.

## 2026-10-14: Etap 3 (część 2), maszyna DNS i pliki stref

**Co zrobiłem**
- Postawiłem VM `dns.okdupiklaster.lab.okd` (Rocky Linux 9, 172.18.22.69): użytkownik z sudo (grupa wheel), nazwa hosta, stały adres przez `nmcli`.
- Zainstalowałem BIND, napisałem strefę prostą `okdupiklaster.lab.okd` i odwrotną `22.18.172.in-addr.arpa` w `/var/named/`.
- Obie strefy przeszły `named-checkzone`. Pliki mają właściciela `root:named`, prawa 640 i kontekst SELinux `named_zone_t`.

**Co nie zadziałało i dlaczego**
- Pierwsze pliki stref zapisałem na bastionie zamiast na maszynie DNS, bo VM DNS jeszcze nie istniała. Wniosek: przed pracą sprawdzać `hostname`.
- `sudo chown ... /var/named/*.zone` zwracał „nie ma takiego pliku”. Gwiazdkę rozwija powłoka użytkownika przed uruchomieniem sudo, a zwykły użytkownik nie widzi zawartości `/var/named`. Rozwiązanie: pełne nazwy plików.
- W strefach zostały pozostałości ze wzorca z dokumentacji (`smtp`, `MX`, `helper`, rekord NS wskazujący na VIP). Usunąłem je, a NS wskazuje teraz na `dns` (.69).
- `named-checkzone` zgłosił brak TTL i „SOA not at top of zone”. Tekst wkleiłem do vi w trybie normalnym, więc pierwsze znaki zadziałały jak polecenia i zniknęła linia `$TTL` oraz znak `@`. Rozwiązanie: `:set paste`, `i`, dopiero potem wklejanie.

**Czego się nauczyłem**
- Rekord NS musi wskazywać na maszynę, na której działa `named`, nie na load balancer.
- Numer seryjny strefy zapisuje się w formacie RRRRMMDDNN i zwiększa przy każdej zmianie.
- Pliki stref w Rocky leżą w `/var/named/` i muszą mieć kontekst SELinux `named_zone_t`, inaczej `named` ich nie odczyta mimo poprawnych praw.
- Błąd na linii 1 w `named-checkzone` to sygnał, żeby najpierw obejrzeć początek pliku (`head`).

## 2026-10-15: Etap 3 (część 3), konfiguracja BIND i weryfikacja DNS

**Co zrobiłem**
- W `/etc/named.conf` dodałem nasłuch na 172.18.22.69, zapytania z sieci 172.18.22.0/24, przekazywanie zapytań zewnętrznych do bramy oraz deklaracje stref `okdupiklaster.lab.okd` i `22.18.172.in-addr.arpa`.
- Uruchomiłem `named`, otworzyłem usługę `dns` w firewalld.
- Zweryfikowałem z bastionu `dig`: `api`, `api-int`, `*.apps`, `cp0`, PTR dla .63 i .65 oraz rozwiązywanie `quay.io`. Wszystkie testy poprawne.
- Bastion i maszyna DNS używają teraz systemowo serwera 172.18.22.69.

**Co nie zadziałało i dlaczego**
- Brakowało średników w `listen-on`, `allow-query` i `forwarders`. W BIND każdy element w nawiasach klamrowych kończy się średnikiem.
- W `listen-on` wpisałem nazwę hosta. Ta opcja przyjmuje tylko adresy IP, bo BIND startuje, zanim cokolwiek rozwiąże.
- Nowe bloki `zone` skopiowałem ze wzoru razem z nazwą `"."` i bez zamknięć `};`, a przy okazji uszkodziłem oryginalny blok `"."`. `named-checkconf` wskazał linię z błędem.

**Czego się nauczyłem**
- `named-checkconf` sprawdza składnię, a `named-checkconf -z` dodatkowo wczytuje strefy.
- `forwarders` pozwala serwerowi z własnymi strefami rozwiązywać też nazwy z internetu, co jest potrzebne węzłom do pobierania obrazów z `quay.io`.
- Testy DNS robię z innej maszyny niż serwer, bo sprawdzają przy okazji firewall i adres nasłuchu.
- Jeden adres IP może mieć kilka rekordów PTR, tak jak VIP z nazwami `api` i `api-int`.
