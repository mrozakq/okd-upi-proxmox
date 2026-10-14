# Dziennik prac
o zrobiłem**
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
