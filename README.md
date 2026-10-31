# Zadanie rekrutacyjne Devops - sprawozdanie z wykonania

## Zadanie 1 (Deploy aplikacji webowej)

1. Wybór sposobu postawienia aplikacji: ponieważ posiadam własny serwer VPS oraz adres IPv4 na których hostuję różne aplikacje z których korzystam na codzień wybieram taką metodę postawienia web-solvro-docs. Korzystam tam z Docker Compose, więc wykorzystam go do postawienia tego systemu.
2. Napotkany problem: w swoich projektach do klonowania kodu z repozytorium automatycznie przez Dockera używałem w Docker Compose linku do repozytorium jako build context, ale wtedy Docker szuka Dockerfile w plikach repozytorium (nie mogę napisać własnego). Rozwiązanie: `dockerfile_inline`, okazuje się, że zawartość Dockerfile można wpisać bezpośrednio w `docker-compose.yml` korzystając z tego pola.
3. Do zmiany `allowScripts.sharp` korzystam z wbudowanej w npm komendy `npm pkg set`.
4. Do hostowania skompilowanej strony z katalogu `dist` wykorzystuję nginx. A dokładniej jego oficjalny, najlżejszy obraz `alpine-slim`.
5. Dodaję ten kontener także do wspólnej sieci z kontenerem nginx rozdzielający ruch przychodzący do mojego serwera na podstawie subdomen do odpowiednich kontenerów. Nie ma potrzeby ustawiania `ports`, ponieważ łączenie z kontenerem odbywa się za pośrednictwem kontenera rozdzielającego, a on widzi `web-solvro-docs` po nazwie.
6. Przesyłam folder `web-solvro-docs` na serwer (plik Docker Compose dostępny w tym repozutorium w `web-solvro-docs/docker-compose.yml`).
7. Napotkane problemy: zpomniane `|` przy `dockerfile_inline`, aby traktował kolejne wiersze jako osobne linie. Drugi problem to konieczność używania `docker compose` zamiast `docker-compose`, gdyż ten drugi jest starszą wersją i nie wspiera między innymi `dockerfile_inline`.
8. Kontener wystartował.
9. Teraz konfiguracja w moim kontenerze rozdzielającym ruch z internetu. Posiadam certyfikat SSL na wszystkie subdomeny, dlatego uruchamiam aplikację przez https na `https://web-solvro-docs.sze.one`. Do mojej konfiguracji dodałem do przekierowania 301 z http na https nową subdomenę oraz jej konfigurację:
```
server {
    listen 443 ssl;
    server_name web-solvro-docs.sze.one;

    ssl_certificate     /etc/nginx/ssl/live/sze.one/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/sze.one/privkey.pem;

    resolver 127.0.0.11 valid=10s;

    set $upstream web-solvro-docs:80;

    location / {
        proxy_pass http://$upstream;
    }
}
```
10. Przeładowuję konfigurację nginx (dzięki makefile wystarczy `make` w katalogu odpowiedniego kontenera).
11. Strona działa pod adresem `https://web-solvro-docs.sze.one`. :)
12. Napotkany problem: niektóre linki przekierowują do `http://web-solvro-docs/git-github/solvro/`. A taka strona oczywiście nie istnieje. Poprawki: przypadkowo usunięta linia `proxy_set_header Host $host;`z konfiguracji nginx. Po poprawie i przeładowaniu wszystko działa.
```
server {
    listen 443 ssl;
    server_name web-solvro-docs.sze.one;

    ssl_certificate     /etc/nginx/ssl/live/sze.one/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/sze.one/privkey.pem;

    resolver 127.0.0.11 valid=10s;

    set $upstream web-solvro-docs:80;

    location / {
        proxy_pass http://$upstream;
    }
}
```
13. Logi kontenera proxy (rozdzielającego ruch z internetu) `docker compose logs`: znajdują się tam tylko informacje o wchodzeniu na poszczególne podstrony web-solvro-docs wraz z informacjami o adresie IP oraz o urządzeniu.
14. Logi z kontenera aplikacji `docker compose logs`: podobnie jak w poprzednim punkcie (to również nginx), ale na początku znajdują się informacje o starcie.

## Zadanie 2 (VPN Wireguard)

1. Wybieram realizację poprzez kontenery Docker. Jako serwer posłuży laptop z linuxem.
2. Tworzę 3 kontenery alpine linux (w jednym pliku docker-compose) w trybie uprzywilejowanym ze stałymi adresami ip, będące w jednej sieci. Przy starcie instalują `wireguard-tools`, `tcpdump` i `iptables`.
3. w każdym kontenerze tworzę parę kluczy wiregurd.
4. Jako podsieć VPN wybieram `10.10.0.0/24` (zakres prywatny, inny niż LAN: `172.30.0.0/24`). Węzły dostają adresy `10.10.0.1`–`10.10.0.3`.
5. Tworzę konfigurację wg-quick dla każdego node'a (każdy node ma w konfiguracji pozostałe dwa jako peery, więc każda para łączy się bezpośrednio co tworzy sieć typu mesh), przykład dla 1. node'a:
```
[Interface]
Address = 10.10.0.1/24
PrivateKey = <prywatny node1>
ListenPort = 51820

[Peer]
# node2
PublicKey = <publiczny node2>
Endpoint = 172.30.0.12:51820
AllowedIPs = 10.10.0.2/32

[Peer]
# node3
PublicKey = <publiczny node3>
Endpoint = 172.30.0.13:51820
AllowedIPs = 10.10.0.3/32
```
6. Uruchamiam konfigurację i testuję poprzez ping każdego node'a z każdym przez adres vpn np. `docker exec node1 ping -c 2 10.10.0.2`.
7. Aby udowodnić, że ruch idzie przez vpn w jednym termialu uruchamiam nasłuchiwanie zawartości pakietów lan z node2: `docker exec node2 tcpdump -ni eth0 -X udp port 51820`. A w drugim pinguję node2 z node1. Dane wyglądają jak szum i są różne dla każdego pakietu.
8. Dla pełnego dodowdu powtarzam eksperyment nasłuchując `wg0` zamiast `eth0`. Pakiety są rozpoznawane jako ICMP echo request wraz z id co znaczy że są odszyfrowane "po wyjściu z vpn'a".
9. Na node2 stawiam prosty serwer httpd z jednym plikiem html. Ustawiam by nasłuchiwał na porcie 8080 na wszystkich interfejsach/adresach.
10. Za pomocą komendy `wget` uruchamianej w node1 pobieram serwowaną stronę zarówno przez 172.30.0.12:8080 (lan) jak i 10.10.0.2:8080 (vpn).
11. W node2 ustawiam regułę firewall'a, która odrzuca wszystkie pakiety tcp kierowane na port 8080, które nie przyszły przez `wg0` - vpn.
12. Ponownie próbuję połączyć się z node1 przez adres z lan (`http://172.30.0.12:8080`) i vpn (`http://10.10.0.2:8080`). Pierwszy zwraca timeout, a drugi zawartość strony (jak wcześniej oba).
13. Do routingu przez vpn wybieram podsieć `10.20.0.0/24`.
14. Włączam ip Forwarding i dodaję adres `10.20.0.1/24` do interfejsu `lo` w node3.
15. Dodaję w konfiguracji wireguard node1 i node2 adres `10.20.0.0/24` jako drugi w `AllowedIPs` w sekcji peer'a node3. Następnie przeładowuję konfigurację.
16. Pingując z node1 adres `10.20.0.1` jest odpowiedź. Gdy w drugim terminalu nasłuchuję w tym czasie na node3 przez `tcpdump` interfejsu `wg0` widzę pakiety icmp z `10.10.0.1` do `10.20.0.1`. Natomiast nasłuchując `eth0` widać tylko zaszyfrowany pakiet `udp`. Dodatkowo ping z node1 na `10.20.0.1` zrobiony przed dopisaniem `10.20.0.0/24` do `AllowedIPs` nie dostawał odpowiedzi (a po tej zmianie działa). Oprócz tego polecenie `ip route get 10.20.0.1` na node1 wskazuje, że ruch do tej podsieci idzie przez interfejs `wg0` czyli przez vpn.

## Zadanie 3 (Konfiguracja dostępów do maszyny wirtualnej)
Zadanie zakładało maszynę wirtualną. Punkty 1–17 wykonałem na drugim, fizycznym laptopie z Kubuntu pełniącym rolę serwera konfiguracja SSH, sudo i panelu dostępu jest w tym przypadku identyczna jak w VM. Maszyny wirtualnej (Alpine Linux w QEMU) użyłem do konsoli szeregowej (punkt 18).

1. Do tego zadania wykorzystam drugiego laptopa z zainstalowanym linuxem (Kubuntu), podpiętym do tej samej sieci.
2. Tworzę nowego użytkownika z uprawnieniami sudo: `sudo adduser test-user` i `sudo usermod -aG sudo test-user`.
3. Przelogowując się na niego faktycznie może wykonywać komendy z sudo np. `sudo whoami`.
4. Instaluję `openssh-server` i uruchamiam ssh w systemctl.
5. Za pomocą `ssh-copy-id` wysyłam klucz ssh na "serwer" czyli do laptopa z linuxem.
6. Następnie loguję się standardową komendą `ssh` lecz z dodatkowym parametrem -i po którym podaję adres do klucza ssh. Wszystko działa.
7. Aby zabezpieczy serwer przed koniecznością interwencji podczas aktualizacji ssh wystarczy customową konfigurację zapisać jako `/etc/ssh/sshd_config.d/*.conf`, bo wszystkie takie pliki są wczytywane przed domyślnym configiem aktualizowanym razem z całym ssh. Tworzę więc `/etc/ssh/sshd_config.d/01-tylko-klucze.conf` w którym przy okazji wyłączam możliwość logowania hasłem. Próbując się zalogować hasłem przez ssh faktycznie serwer (laptop z linuxem) na to nie pozwala.
8. Aby wykonać zadanie z uprawnieniami przypisanymi do konkretnych grup tworzę grupy: `ssh-users`, `ssh-pass-ip`, `ssh-pass-any`, `ssh-forward`, `ssh-gateway`, `ssh-x11`. Grupę `ssh-users` nadaję użytkownikowi test-user, żeby nie stracić zdalnego dostępu do komputera. Zastępuję także wcześniejszy plik konfiguracji `01-tylko-klucze.conf`, plikiem `01-grupy.conf`. Ustawiam w nim możliwość logowania przez ssh tylko dla grupy ssh-users; wyłączam logowanie hasłem; wyłączam przekazywanie portów; wyłączam przekazywanie x11. Ustawiam też wyjątki odpowiednio dla utworzonych grup. Na koniec restartuję ssh.
9. Do wyświetlenia i oceny algorytmów na serwerze używam `ssh-audit`. Tworzę plik `00-kryptografia.conf` zgodnie z rekomendacjami z ssh-audit i restartuję ssh. Ponowna ocena nie pokazuje rekomendacji.
10. Tworzę plik `02-port.conf` w którym oprócz standardowego portu ssh 22 dodaję 2222.
11. Po sprawdzeniu udało się zalogować przez port 2222.
12. Okazuje się, że mam starą wersję ssh, więc aby móc skorzystać z funkcji blokowania adresów ip muszę zaktualizować system razem z serwerem ssh.
TO DO: 13. Do utworzenia systemu banów używam tworzę plik konfiguracyjny `00-bany.conf`. Z `PerSourcePenalties authfail:1m noauth:10s max:1h min:30s` oraz wszystkimi moimi lokalnymi adresami ip jako wyjątki.
13. Tworzę użytkownika user-operator.
14. Sprawdzam pełną ściężkę (`which`) do komend `whoami` i `fdisk`, którą będzie wykonywał user-operator w "nie swoim imieniu". U mnie to `/usr/sbin/shutdown`.
15. Za pomocą visudo ustawiam możliwość wykonywania tylko `fdisk -l` jako root i `whoami` jako test-user bez hasła:
```
user-operator ALL=(root) /usr/sbin/fdisk -l
user-operator ALL=(test-user) NOPASSWD: /usr/bin/whoami
```
16. Przełączam się na `user-operator` i sprawdzam, że faktycznie może wykonywać z root tylko fdisk -l oraz whoami z root jako test-user bez hasła.
17. Instaluję panel administracyjny `cockpit`. Działa, można uzyskać dostęp przez przeglądarkę do informacji o systemie, konsoli, kontach, dysku itp.
18. Ponieważ wcześniejsze polecenia wykonywałem na laptopie z linuxem, maszynę wirtualną stworzę na nim. Uruchomiłem Alpine Linux w QEMU z opcją `-nographic`, która do dostępu do terminala wykorzystuje port szeregowy co potwierdza komenda `tty` zwracająca `/dev/ttyS0`.