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