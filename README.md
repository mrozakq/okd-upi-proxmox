# okd-upi-proxmox

## Cel projektu

Ręczna instalacja klastra OKD 4.19 metodą UPI (User Provisioned Infrastructure) na maszynach wirtualnych w Proxmoxie, na podstawie oficjalnej dokumentacji OKD dla bare metal. Celem jest nie tylko działający klaster, ale zrozumienie każdego etapu: od przygotowania infrastruktury (DNS, NTP, load balancer, serwer HTTP), przez generowanie plików ignition, po zatwierdzenie węzłów i weryfikację operatorów klastra.

## Architektura

Klaster kompaktowy: 3 węzły master, które pełnią też rolę worker, tymczasowy węzeł bootstrap oraz osobne maszyny pomocnicze.

- Domena bazowa: `lab.okd`
- Nazwa klastra: `okdupiklaster`
- Adres API: `api.okdupiklaster.lab.okd`
- Sieć: 172.18.22.0/24, brama 172.18.22.1
- DHCP rozdaje adresy od .100, adresy klastra są poza pulą

| Maszyna | Adres IP |
|---|---|
| HAProxy 1 | 172.18.22.61 |
| HAProxy 2 | 172.18.22.62 |
| VIP (keepalived) | 172.18.22.63 |
| Bootstrap | 172.18.22.64 |
| Master 1 | 172.18.22.65 |
| Master 2 | 172.18.22.66 |
| Master 3 | 172.18.22.67 |
| Bastion | 172.18.22.68 |
| DNS + NTP | 172.18.22.69 |

Rekordy `api`, `api-int` i `*.apps` wskazują na VIP. Za nim działa para HAProxy, której dostępność zapewnia keepalived.

## Środowisko

- Host fizyczny: Hyper-V Server 2019, 4× Intel Xeon E7-8837
- Proxmox VE 9.2 uruchomiony jako maszyna wirtualna (generacja 1, 80 GB RAM), adres 172.18.22.60
- Maszyny pomocnicze (bastion, DNS + NTP, HAProxy): Rocky Linux 9
- Na bastionie: `openshift-install` i `oc` w wersji 4.19

Dlaczego OKD 4.19: procesory E7-8837 nie mają instrukcji AVX i obsługują poziom x86-64-v2. Nowsze wydania OKD przechodzą na SCOS 10 oparty na CentOS Stream 10, który wymaga x86-64-v3. OKD 4.19 używa SCOS 9, zgodnego z tym sprzętem. Z tego samego powodu maszyny pomocnicze działają na Rocky Linux 9, a nie 10.

## Etapy instalacji

1. **Obraz SCOS** (zrobione). Adres ISO pobrałem poleceniem `openshift-install coreos print-stream-json` i wybrałem go przez `jq`. Stream JSON jest wbudowany w instalator, więc obraz pochodzi z tego samego wydania co `openshift-install`, a wersja specyfikacji Ignition jest zgodna. Pobrany plik zweryfikowałem sumą `sha256` z tego samego pliku JSON.
2. **Plan adresacji i nazw** (zrobione). Domena, nazwa klastra i stałe adresy wszystkich maszyn, opisane w sekcji Architektura.
3. DNS: rekordy A i PTR, weryfikacja `dig`
4. NTP
5. HAProxy i keepalived
6. Serwer HTTP dla plików ignition
7. `install-config.yaml`
8. Manifesty i pliki ignition
9. Instalacja węzłów w Proxmoxie
10. Bootstrap
11. Konfiguracja po instalacji
12. Druga instalacja od zera

Wiecej w dziennik -> https://github.com/mrozakq/okd-upi-proxmox/blob/main/docs/dziennik.md
