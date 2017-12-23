---
layout: page
title: osa 4
permalink: /osa4/
---

<div class="important">
  <h1>KESKEN, ÄLÄ LUE</h1>
</div>

## Osan 4 oppimistavoitteet

- Node/express
  - Router
  - Helmet.js https://www.npmjs.com/package/helmet
- Node-sovellusten testaut
  - ava/supertest
- Mongoose
  - Monimutkaasemmat skeemat
  - Viittaukset kokoelmien välillä
- Web
  - Http-operaatioiden safety ja idempotency
  - Token-autentikaatio
  - JWT
- Muu
  - lint
- JS
  - async/await
- React
  - Lisää formeista: mm refs
  - Bootstrap (reactstrap) tai Semantic UI
  - Periaatteita: Virtual dom
  - Proptype
  - child https://reactjs.org/docs/composition-vs-inheritance.html
- Frontendin testauksen alkeet
  - Ava jsdom enzyme

## Sovellukuksen rakenteen parantelu

Muutetaan sovelluksen rakennetta, siten että projektin juuressa oleva _index.js_ lähinnä ainoastaan konfiguroi sovelluksen tietokannan ja middlewaret ja siirretään rotejen määrittely omaan tiedostoonsa.

Routejen tapahtumankäsittelijöitä kutsutaan usein _kontrollereiksi_. Luodaankin hakemisto _controllers_ ja sinne tiedosto _notes.js_ johon tulemme siirtämään kaikki muistiinpanoihin liittyvien reittien määrittelyt.

Tiedoston sisältö on seuraava:

```js
const routerRouter = require('express').Router()
const Note = require('../models/note')

const formatNote = (note) => {
  const formattedNote = { ...note._doc, id: note._id }
  delete formattedNote._id

  return formattedNote
}

routerRouter.get('/', (request, response) => {
  Note
    .find({}, '-__v')
    .then(notes => {
      response.json(notes.map(formatNote))
    })
})

routerRouter.get('/:id', (request, response) => {
  Note
    .findById(request.params.id)
    .then(note => {
      if (note) {
        response.json(formatNote(note))
      } else {
        response.status(404).end()
      }
    })
    .catch(error => {
      response.status(400).send({ error: 'malformatted id' })
    })
})

routerRouter.delete('/:id', (request, response) => {
  Note
    .findByIdAndRemove(request.params.id)
    .then(result => {
      console.log(result)
      response.status(204).end()
    })
    .catch(error => {
      response.status(400).send({ error: 'malformatted id' })
    })
})

routerRouter.post('/', (request, response) => {
  const body = request.body

  if (body.content === undefined) {
    response.status(400).json({ error: 'content missing' })
  }

  const note = new Note({
    content: body.content,
    important: body.content === undefined ? false : body.important,
    date: new Date(),
  })

  note
    .save()
    .then(note => {
      return formatNote(note)
    })
    .then(formattedNote => {
      response.json(formattedNote)
    })

})

routerRouter.put('/:id', (request, response) => {
  const body = request.body

  const note = {
    content: body.content,
    important: body.important,
  }

  Note
    .findByIdAndUpdate(request.params.id, note, { new: true })
    .then(updatedNote => {
      response.json(formatNote(updatedNote))
    })
    .catch(error => {
      console.log(error)
      response.status(400).send({ error: 'malformatted id' })
    })
})

module.exports = routerRouter;
```

Käytännössä kyse melkein suora copypaste tiedostosta _index.js_.

Muutoksia on pari. Tiedoston alussa luodaan [router](http://expressjs.com/en/api.html#router)-olio:

```js
const routerRouter = require('express').Router()

//...

module.exports = routerRouter
```

Tiedoston määrittelemä moduuli tarjoaa moduulin käyttäjille routerin.

Kaikki määriteltävät routet liitetään router-olioon, samaan tapaan kuin aiemmassa versiossa routet liitettiin sovellusta edustavaan olioon.

Huomioinarvoinen seikka routejen määrittelyssä on se, että polut ovat typistyneet, aiemmin määrittelimme esim.

```js
app.delete('/api/notes/:id', (request, response) => {
```

nyt riittää määritellä

```js
routerRouter.delete('/:id', (request, response) => {
```

Mistä routereissa oikeastaan on kyse? Expressin manuaalin sanoin

> A router object is an isolated instance of middleware and routes. You can think of it as a “mini-application,” capable only of performing middleware and routing functions. Every Express application has a built-in app router.

Router on siis _middleware_, jonka avulla on mahdollista määritellä joukko "toisiinsa liittyviä" routeja yhdessä paikassa.

Ohjelman käynnistyspiste, eli määrittetyt tekevä _index.js_ ottaa määrittelemämme routerin käyttöön seuraavasti:

```js
const notesRouter = require('./controllers/notes')
app.use('/api/notes', notesRouter)
```

Näin määrittelemäämme routeria käytetään _jos_ polun alkuosa on _/api/notes_. notesRouter-olion sisällä tarvitsee tämän takia käyttää ainoastaan polun loppuosia, eli tyhjää polkua _/_ tai pelkkää parametria _/:id_.

### sovelluksen muut osat

Sovelluksen käynnistyspisteenä _index.js_ näyttää muutosten jälkeen seuraavalta:

```js
const http = require('http')
const express = require('express')
const mongoose = require('mongoose')
const app = express()
const bodyParser = require('body-parser')
const cors = require('cors')
const utils = require('./utils')

if ( process.env.NODE_ENV!=='production' ) {
  require('dotenv').config()
}

const url = process.env.MONGODB_URI
mongoose.connect(url, { useMongoClient: true })
mongoose.Promise = global.Promise

app.use(cors())
app.use(bodyParser.json())
app.use(express.static('build'))
app.use(utils.loggerMiddleware)

const notesRouter = require('./controllers/notes')
app.use('/api/notes', notesRouter)

app.use(utils.errorMiddleware)

const PORT = process.env.PORT
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`)
})
```

Tiedostossa siis otetaan käyttöön joukko middlewareja, näistä yksi on polkuun _/api/notes_ kiinnitettävä _notesRouter_ (tai notes-kontrolleri niinkuin jotkut sitä kutsuisivat) avataan yhteys tietokantaan ja käynnistetään sovellus.

Middlewareista kaksi _utils.loggerMiddleware_ ja _utils.errorMiddleware_ on määritelty hakemiston _utils_ tiedostossa _index.js_:

```js
const loggerMiddleware = (request, response, next) => {
  console.log('Method:', request.method)
  console.log('Path:  ', request.path)
  console.log('Body:  ', request.body)
  console.log('---')
  next()
}

const errorMiddleware = (request, response) => {
  response.status(404).send({ error: 'unknown endpoint' })
}

module.exports = {
  loggerMiddleware,
  errorMiddleware
}
```

Tietokantayhteyden muodostaminen on nyt siirretty konfiguraatiot tekevän _index.js_:n vastuulle. Hakemistossa _models_ oleva tiedosto _note.js_ sisältää nyt ainoastaan muistiinpanojen skeeman määrittelyn.

```js
const mongoose = require('mongoose')

const Note = mongoose.model('Note', {
  content: String,
  date: Date,
  important: Boolean
})

module.exports = Note
```

Tämän hetkinen koodi on kokonaisuudessaan githubissa repositoriossa xxx.

Express-sovelluksien rakenteelle, eli hakemistojen ja tiedostojen nimennälle ei ole olemassa mitään standardia (samaan tapaan kuin esim Ruby on Railsissa). Tässä käyttämämme malli noudattaa eräitä internetissä mainittuja hyviä käytäntöjä.

[express cli](https://expressjs.com/en/starter/generator.html)

Tiedostonimi index

## node-sovellusten testaaminen

Olemme laiminlyöneet ikävästi yhtä oleellista ohjelmistokehityksen osa-aluetta, automatisoitua testausta.

Aloitamme yksikkötestauksesta. Sovelluksemme logiikka on sen verran yksinkertaista, että siinä ei ole juurikaan mielekästä yksikkötestattavaa. Lisätäänkin tiedostoon _utils/index.js_ pari yksinketaista funktiota testattavaksi:

```js
//...
const palindrom = (string) => {
  const letters = []
  for(let i = 0; i < string.length; i++) {
    letters.push(string.charAt(i))
  }

  return letters.reverse().join('')
}

const average = (array) => {
  const reducer = (sum, item) => {
    return sum + item
  }

  return array.reduce(reducer, 0) / array.length
}

module.exports = {
  loggerMiddleware,
  errorMiddleware,
  palindrom,
  average
}
```

Javascriptiin on tarjolla runsaasti erilaisia testikirjastoja eli _test runneria_. Käytämme tällä kurssilla nopeasti suosiota saavuttanutta uutta tulokasta nimeltään [ava](https://github.com/avajs/ava). Ava on erityisen käyttökelpoinen asynkronista koodia testatessa ja se onkin nopeasti valtaamassa alaa "edellisen sukupolven" suosituimmalta testirunnerilta [Mochalta]() joka on toki edelleen paljon käytössä ja kelpo vaihtoehto mihin tahansa projektiin.

Koska testejä on tarkoitus suorittaa ainoastaan sovellusta kehitettäessä, asennetaan _ava_ kehitysaikaiseksi riippuvuudeksi komennolla

```bash
npm install --save-dev ava
```

määritellään _npm_ skripti _test_ suorittmaan testaus avalla ja raportoimaan testien suorituksesta _verbose_-tyylillä:

```bash
{
  //...
  "scripts": {
    "start": "node index.js",
    "watch": "node_modules/.bin/nodemon index.js",
    "test": "node_modules/.bin/ava --verbose test"
  },
  //...
}
```

Tehdään testejä varten hakemisto _test_ ja sinne tiedosto _palindrom.js_ ja sille sisältö

```js
const test = require('ava')
const palindrom = require('../utils').palindrom

test('palindrom of a', t => {
  const result = palindrom('a')

  t.is('a', result)
})

test('palindrom of a', t => {
  const result = palindrom('react')

  t.is('tcaer', result)
})

test('palindrom of saippuakauppias', t => {
  const result = palindrom('saippuakauppias')

  t.is('saippuakauppias', result)
})
```

Testitiedosto ottaa alussa ava-kirjaston käyttöön ja sijoittaa sen konvention mukaisesti muuttujaan _test_. Kirjasto ottaa käyttöön myös testattavan funktion sijoittaen sen muuttujaan _palindrom_.

Testimetodit ovat yksinkertaisia. Yksittäinen testi kirjoitetaan nuolifunktiosyntaksilla. Nuolifunktion muuttujaa _t_ käytetään _assertointiin_, eli testattavan metodin vastauksen oikeellisuuden tarkastamiseen.

Kuten odotettua, testit menevät läpi:

![]({{ "/assets/4/1.png" | absolute_url }})

Avan antamat virheilmoitukset ovat kohtuullisen hyviä, rikotaan testi

```js
test('palindrom of react', t => {
  const result = palindrom('react')

  t.is('tkaer', result)
})
```

seurauksena on seuraava virheilmotus

![]({{ "/assets/4/2.png" | absolute_url }})

Lisätään muutama testi metodille _average_, tiedostoon _test/average.js_

```js
const test = require('ava')
const average = require('../utils').average

test('average of one', t => {
  const result = average([1])

  t.is(1, result)
})

test('average of many', t => {
  const result = average([1, 2, 3, 4, 5, 6])

  t.is(3.5, result)
})

test('average of empty array', t => {
  const result = average([])

  t.is(0, result)
})
```

Testi paljastaa, että metodi toimii väärin tyhjällä taulukolla:

![]({{ "/assets/4/3.png" | absolute_url }})

Metodi on helppo korjata

```js
const average = (array) => {
  const reducer = (sum, item) => {
    return sum + item
  }
  return array.length == 0 ? 0 : array.reduce(reducer, 0) / array.length
}
```

## Tehtäviä

### tee testit ja toteuta metodit...

## api:n testaaminen

Joissain tilanteissa voisi olla mielekästä suorittaa ainakin osa backendin testauksesta siten, että oikea tietokanta eristettäisiin testeistä ja korvattaisiin "valekomponentilla" eli mockilla, eräs tähän sopiva ratkaisu olisi [mongo-mock](https://github.com/williamkapke/mongo-mock)

Koska sovelluksemme backendin on koodiltaan kuitenkin suhteellisen yksinkertainen päätämme testata sitä kokonaisuudessaan, siten että testeissä käytetään myös tietokantaa. Tämänkaltaisia, useita sovelluksen komponetteja yhtäaikaa käyttäviä testejä voi luonnehtia _integraatiotesteiksi_.

### test-ympäristö

Edellisen osan luvussa [Sovelluksen vieminen tuotantoon](osa3/##-Sovelluksen-vieminen tuotantoon) mainitsimme, että kun sovellusta suoritetaan Herokussa, on se _production_-moodissa.

Noden konventiona on määritellä projektin suoritusmoodi ympäristömuuttujan _NODE_ENV_ avulla. Lataammekin sovelluksen nykyisessä versiossa tiedostossa _.env_ määritellyt ympäristömuuttujat ainoastaan jos sovellus _ei ole_ production moodissa:

```js
if (process.env.NODE_ENV !== 'production') {
  require('dotenv').config()
}
```

Yleinen käytäntö on määritellä sovelluksille omat moodinsa myös sovelluskehitykseen ja testaukseen.

Määrtellään nyt tiedostossa _package.js_, että testejä suorittaessa sovelluksen _NODE_ENV_ saa arvokseen _test_:

```json
{
  // ...
  "scripts": {
    "start": "node index.js",
    "watch": "node_modules/.bin/nodemon index.js",
    "test": "NODE_ENV=test node_modules/.bin/ava --verbose test"
  },
  // ...
}
```

Nyt voimme konfiguroida sovelluksen käyttäytymistä, testien aikana, erityisesti voimme määritellä että testejä suoritettaessa ohjelma käyttää erillistä, testejä varten luotua tietokantaa.

Sovelluksen testikanta voidaan luoda tuotantokäyttöön ja sovellukehitykseen tapaan _mlabiin_. Ratkaisu ei kuitenkaan ole optimi, jos sovellusta kehittää yhtä aikaa usea henkilö. Testien suoritus edellyttää yleensä sitä, että samaa testi-instanssia ei ole yhtä aikaa käyttämässä useampia testiajoja.

Testaukseen kannattaakin käyttää verkossa olevaa jaettua tietokantaa mielummin esim. sovelluskehittäjän paikallisen koneen tietokantaa. Optimiratkaisu olisi tietysti se, jos jokaista testiajoa varten olisi käytettävissä oma tietokanta, sekin periaatteessa onnistuu suhteellisen helposti mm. dockerin avulla. Etenemme kuitenkin nyt lyhyemmän kaavan mukaan.

Tehdään sovelluksen käynnistyspisteenä toimivaan tiedostoon _index.js_ muutama muutos:

```js
// ...
const envIs = (which) => process.env.NODE_ENV === which

if (!envIs('production')) {
  require('dotenv').config()
}

const url = envIs('test') ? process.env.TEST_MONGODB_URI : process.env.MONGODB_URI

// ...

const PORT = envIs('test') ? process.env.TEST_PORT : process.env.PORT

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`)
})

module.exports = app
```

Koodi lataa ympäristömuuttujat tiedostosta _.env_ jos se ei ole sovelluskehitysmoodissa.
Tiedostossa _.env_ on nyt määritelty erikseen sekä sovelluskehitysympäristön ja testausympäristön tietokannan osoite ja portti:

```bash
MONGODB_URI=mongodb://localhost/muistiinpanot
PORT=3001

TEST_PORT=3002
TEST_MONGODB_URI=mongodb://localhost/test
```

Eri porttien käyttö mahdollistaa sen, että sovellus voi olla käynnissä testien suorituksen aikana.

Tiedoston loppuun on myös lisätty komento

```js
module.exports = app
```

tämä mahdollistaa sen, että sovellusolioon pääsee tarvittaessa käsiksi tiedoston ulkopuolelta, tämä on oleellista kohta käyttöönottamallamme testikirjastolle.

### supertest

Käytetään API:n testaamiseen avan apuna [supertest](https://github.com/visionmedia/supertest)-kirjastoa.

Kirjasto asennetaan kehitysaikaiseksi riippuvuudeksi komennolla

```bash
npm install --save-dev supertest
```

Luodaan heti ensimmäinen testi tiedostoon _test/note_api.js:_

```js
const test = require('ava')
const supertest = require('supertest')
const app = require('../index')
const api = supertest(app)

test('notes are returned as json', async t => {
  const res = await api
    .get('/api/notes')
    .expect(200)
    .expect('Content-Type', /application\/json/)

  t.pass()
})
```

Alussa testi käynnistää backendin ja käärii sen funktiolla _supertest_ ns. [superagent](https://github.com/visionmedia/superagent)-olioksi. Tämä olio sijoitataan muuttujaan _api_ ja sen kautta testit voivat tehdä HTTP-pyyntöjä backentdiin.

Testimetodi tekee HTTP GET -pyynnön osoitteeseen _api/notes_ ja varmistaa, että pyyntöön vastataan statuskoodilla 200 ja että data palautetaan oikeassa muodossa, eli että _Content-Type_:n arvo on _application/json_.

Testissä on muutama detalji joihin tutustumme vasta myöhemmin. Testin määrittelevä nuolifunktio alkaa sanalla _async_ ja _api_-oliolle tehtyä metodikutsua edeltää sama _await_. Teemme ensin muutamia testejä ja tutustumme sen jälkeen async/await-magiaan. Tällä hetkellä niistä ei tarvitse välittää, kaikki toimii kun kirjoitat testimetodit esimerkin mukaan.

Huomionarvoista on myös testin lopettava komento <code>t.pass()</code>, joka "ei testaa mitään" vaan merkkaa testin suoritetuksi. Koska testissä ei ole varsinaisia "asserteja", kuten aiempien testiemme _t.is("tcaer", result)_ joudumme kertomaan avalle testien päättymisestä.

Tehdään pari testiä lisää:

```js
test('there are five notes', async t => {
  const res = await api
    .get('/api/notes')

  t.is(5 ,res.body.length)
})

test('the first note is about HTTP methods', async t => {
  const res = await api
    .get('/api/notes')

  t.is('HTTP-protokollan tärkeimmät metodit ovat GET ja POST', res.body[0].content)
})
```

Testit menevät läpi. Testit ovat kuitenkin huonoja, niiden läpimeno riippu tietokannan tilasta. Jotta saisimme robustimmat testit, tulee tietokannan tila nollata ensin testien alussa ja sen jälkeen kantaan voidaan laittaa hallitusti testien tarvitsemaa dataa.

### Error: listen EADDRINUSE :::3002

Jos jotain patologista tapahtuu voi käydä niin, että testien suorittama palvelin jää päälle. Tällöin uusi testiajao aiheuttaa ongelmia

![]({{ "/assets/4/4.png" | absolute_url }})

Ratkaisu tilanteeseen on tappaa palvelinta suorittava prosessi. Portin 3002 varaava prosessi löytyy OSX:lla esim. komennolla <code>lsof -i :3002</code>

```bash
COMMAND  PID     USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
node    8318 mluukkai   14u  IPv6 0x5428af4833b85e8b      0t0  TCP *:redwood-broker (LISTEN)
```

Komennon avulla selviää ikävyyksiä aiheuttavan prosesin PID eli prosessi id. Prosessin saa tapettua komennolla <code>KILL 8318</code> olettaen että PID on niin kuin kuvassa. Joskus prosessi on sitkeä eikä kuole ennen kuin se tapetaan komennolla <code>KILL -9 8318</code>.

En tiedä toimiiko _lsof_ samoin Linuxissa. Windowsissa se ei ei toimi ainakaan. Jos joku tietää, kertokoon asiasta Telegramissa. Tai lisätköön tähän pull requestilla.

### kannan nollaaminen





## async-await

## Mongoose
  - Monimutkaisemmat skeemat
  - Viittaukset kokoelmien välillä

## Web
  - Token-autentikaatio
  - JWT

## Muu
  - lint


## React
  - Lisää formeista: mm refs
  - Bootstrap (reactstrap) tai Semantic UI
  - Periaatteita: Virtual dom
  - Proptype
  - child https://reactjs.org/docs/composition-vs-inheritance.html

## Frontendin testauksen alkeet
  - Ava jsdom enzyme

## misc
  - Http-operaatioiden safety ja idempotency
  - Helmet.js https://www.npmjs.com/package/helmet