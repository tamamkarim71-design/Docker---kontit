
Tämä projekti tehtiin osana Docker-tehtävää. Tavoitteena oli tutustua Dockerin käyttöön sekä oppia rakentamaan ja ajamaan Node.js-sovellus Docker-kontissa Docker Composen avulla.

Projektin pohjana käytettiin Dockerin Getting Started -esimerkkisovellusta. Sovellus käyttää Node.js:ää ja Expressiä.

---

## 2. Dockerin asennus

Docker Desktop asennettiin Windows-koneelle.

Asennuksen toimivuus tarkistettiin seuraavilla komennoilla:

```bash
docker --version
docker compose version
```

Käytössä olevat versiot:

* Docker version 29.7.2
* Docker Compose version v5.4.0

### Kuvakaappaus

Lisää tähän kuvakaappaus Dockerin ja Docker Composen versioista.

---

## 3. Projektin lataaminen

Dockerin Getting Started -esimerkkisovellus kloonattiin GitHubista seuraavalla komennolla:

```bash
git clone https://github.com/docker/getting-started-app.git
```

Tämän jälkeen siirryttiin projektikansioon:

```bash
cd getting-started-app
```

Projektin Node.js-riippuvuudet asennettiin komennolla:

```bash
npm install
```

---

## 4. Sovelluksen testaaminen paikallisesti

Ennen Dockerin käyttöönottoa sovellus testattiin paikallisesti.

Sovellus käynnistettiin komennolla:

```bash
npm run dev
```

Palvelin käynnistyi porttiin 3000:

```text
Listening on port 3000
```

Sovellus avattiin selaimessa osoitteessa:

```text
http://localhost:3000
```

Sovellus toimi onnistuneesti paikallisesti.

### Kuvakaappaus

Lisää tähän kuvakaappaus paikallisesti toimivasta Todo-sovelluksesta.

---

## 5. Docker Init

Docker-konfiguraation luomiseen käytettiin komentoa:

```bash
docker init
```

Docker Init -toiminnossa valittiin seuraavat asetukset:

* Sovellusalusta: Node
* Node-versio: 24.19.0
* Pakettienhallinta: npm
* Käynnistyskomento: npm run dev
* Portti: 3000

Docker Init loi projektiin seuraavat tiedostot:

* `.dockerignore`
* `Dockerfile`
* `compose.yaml`
* `README.Docker.md`

Luodut Dockerfile-, compose.yaml- ja README.Docker.md-tiedostot käytiin läpi ja niiden sisältö tarkistettiin.

---

## 6. Dockerfile

Docker Init loi projektille Dockerfilen.

Dockerfile käyttää Node.js:n Alpine-imagea:

```dockerfile
ARG NODE_VERSION=24.19.0
FROM node:${NODE_VERSION}-alpine
```

Sovelluksen työhakemistoksi asetettiin:

```dockerfile
WORKDIR /usr/src/app
```

Sovellus käyttää porttia 3000:

```dockerfile
EXPOSE 3000
```

Ensimmäisen Docker-testauksen aikana sovelluksen käynnistyksessä tuli virhe:

```text
sh: nodemon: not found
```

Syynä oli se, että Docker-image rakennettiin production-ympäristöä varten:

```dockerfile
npm ci --omit=dev
```

`nodemon` kuuluu projektin kehitysriippuvuuksiin, joten sitä ei asennettu Docker-imageen.

Tämän vuoksi alkuperäinen käynnistyskomento:

```dockerfile
CMD npm run dev
```

muutettiin muotoon:

```dockerfile
CMD ["node", "src/index.js"]
```

Tämän jälkeen sovellus pystyttiin käynnistämään ilman nodemonia.

---

## 7. Käyttöoikeusongelman ratkaiseminen

Seuraavassa testissä sovellus käynnistyi, mutta SQLite-tietokannan luomisessa tuli käyttöoikeusvirhe:

```text
EACCES
syscall: 'mkdir'
path: '/etc/todos'
```

Sovellus yrittää tallentaa SQLite-tietokannan hakemistoon:

```text
/etc/todos/todo.db
```

Docker Init oli määrittänyt sovelluksen ajettavaksi `node`-käyttäjänä:

```dockerfile
USER node
```

Tällä käyttäjällä ei ollut oikeutta luoda tarvittavaa `/etc/todos`-hakemistoa.

Tämän harjoitusprojektin toiminnan mahdollistamiseksi kyseinen rivi poistettiin käytöstä:

```dockerfile
# USER node
```

Tämän muutoksen jälkeen sovellus pystyi luomaan SQLite-tietokannan ja käynnistyi onnistuneesti.

---

## 8. Host-osoitteen muuttaminen

Tehtävänannon mukaisesti Express-palvelimen host-osoitteeksi muutettiin:

```text
0.0.0.0
```

Tiedostossa `src/index.js` palvelimen käynnistys määriteltiin seuraavasti:

```javascript
app.listen(3000, '0.0.0.0', () => console.log('Listening on port 3000'));
```

`0.0.0.0` mahdollistaa yhteyksien vastaanottamisen myös Docker-kontin ulkopuolelta.

---

## 9. Docker-imagen rakentaminen ja sovelluksen ajaminen

Docker-image rakennettiin ja sovellus käynnistettiin Docker Composella:

```bash
docker compose up --build
```

Korjausten jälkeen Docker-image rakentui onnistuneesti ja sovellus käynnistyi:

```text
Using sqlite database at /etc/todos/todo.db
Listening on port 3000
```

Sovellusta pystyi käyttämään selaimessa osoitteessa:

```text
http://localhost:3000
```

### Kuvakaappaus

Lisää tähän kuvakaappaus Dockerissa toimivasta sovelluksesta.

---

## 10. Docker Compose Watch

Tehtävänannon mukaisesti `compose.yaml`-tiedostoon lisättiin Watch-konfiguraatio.

Konfiguraatio:

```yaml
develop:
  watch:
    - action: sync
      path: .
      target: /usr/src/app
      ignore:
        - node_modules/

    - action: rebuild
      path: package.json
```

Watch-tila käynnistettiin komennolla:

```bash
docker compose up --watch
```

Docker ilmoitti Watch-toiminnon käynnistyneen:

```text
Watch enabled
```

Watch-toiminnon avulla Docker Compose voi seurata projektin tiedostoihin tehtyjä muutoksia. Tavalliset lähdekoodimuutokset synkronoidaan konttiin, ja `package.json`-tiedoston muutos käynnistää imagen uudelleenrakentamisen.

### Kuvakaappaus

Lisää tähän kuvakaappaus terminaalista, jossa näkyy:

```text
Watch enabled
```

---

## 11. MongoDB-palvelun lisääminen

Projektia laajennettiin lisäämällä MongoDB erilliseksi Docker Compose -palveluksi.

`compose.yaml`-tiedostoon lisättiin:

```yaml
mongo:
  image: mongo:8
  restart: unless-stopped
  ports:
    - 27017:27017
  volumes:
    - mongo-data:/data/db
```

MongoDB käyttää porttia:

```text
27017
```

MongoDB:n tietojen säilyttämistä varten määriteltiin Docker-volume:

```yaml
volumes:
  mongo-data:
```

Volume mahdollistaa tietojen säilymisen myös silloin, kun MongoDB-kontti käynnistetään uudelleen.

Tässä projektissa MongoDB lisättiin ja testattiin erillisenä Docker-palveluna. Todo-sovellus käyttää edelleen alkuperäistä SQLite-tietokantaa.

---

## 12. Konttien testaaminen

Docker Compose -palveluiden tila tarkistettiin komennolla:

```bash
docker compose ps
```

Tuloksessa näkyivät molemmat palvelut:

* `server` – Node.js/Express-sovellus
* `mongo` – MongoDB

Node.js-sovellus käytti porttia:

```text
3000
```

MongoDB käytti porttia:



Molempien konttien tila oli `Up`, joten palvelut toimivat samanaikaisesti onnistuneesti.

### Kuvakaappaus

Lisää tähän kuvakaappaus `docker compose ps` -komennon tuloksesta.

---

## 13. Sovelluksen toiminnan testaaminen

Todo-sovelluksen toiminta testattiin selaimessa Docker-kontin ollessa käynnissä.

Sovellukseen lisättiin testimerkintä:

```text
Docker test
```

Merkintä tallentui ja näkyi Todo-listassa onnistuneesti.

Tämä osoitti, että Node.js/Express-sovellus toimii Docker-kontissa ja pystyy käyttämään SQLite-tietokantaa.

### Kuvakaappaus

Lisää tähän kuvakaappaus Todo-sovelluksesta, jossa näkyy `Docker test`.

---

## 14. Projektissa käytetyt komennot

Dockerin version tarkistaminen:

```bash
docker --version
docker compose version
```

Projektin kloonaaminen:

```bash
git clone https://github.com/docker/getting-started-app.git
cd getting-started-app
```

Node.js-riippuvuuksien asentaminen:

```bash
npm install
```

Sovelluksen paikallinen käynnistäminen:

```bash
npm run dev
```

Docker-konfiguraation luominen:

```bash
docker init
```

Docker-imagen rakentaminen ja konttien käynnistäminen:

```bash
docker compose up --build
```

Watch-tilan käyttäminen:

```bash
docker compose up --watch
```

Konttien tilan tarkistaminen:

```bash
docker compose ps
```

---

## 15. Lopputulos

Tehtävän aikana Node.js/Express-sovellus saatiin onnistuneesti toimimaan Docker-kontissa.

Tehtävän aikana:

* Docker Desktop asennettiin ja sen toiminta tarkistettiin.
* Docker Compose tarkistettiin.
* Docker Getting Started -esimerkkiprojekti kloonattiin GitHubista.
* Node.js-sovellus testattiin ensin paikallisesti.
* Docker Init -toimintoa käytettiin Docker-konfiguraation luomiseen.
* Dockerfile käytiin läpi ja sitä muokattiin sovellukselle sopivaksi.
* `compose.yaml` käytiin läpi ja sitä muokattiin.
* `README.Docker.md` tarkistettiin.
* Express-palvelimen host-osoitteeksi muutettiin `0.0.0.0`.
* Docker-image rakennettiin ja sovellus käynnistettiin kontissa.
* Docker Compose Watch otettiin käyttöön.
* Sovelluksen toiminta testattiin selaimessa.
* MongoDB lisättiin erilliseksi Docker Compose -palveluksi.
* Node.js- ja MongoDB-konttien toiminta tarkistettiin `docker compose ps` -komennolla.

Lopuksi Node.js/Express-sovellus toimii Docker-kontissa osoitteessa:

```text
http://localhost:3000
```

MongoDB toimii erillisessä kontissa portissa:

