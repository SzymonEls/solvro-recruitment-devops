# Zadanie rekrutacyjne Devops - sprawozdanie z wykonania

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