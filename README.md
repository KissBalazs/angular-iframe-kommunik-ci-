# KofaxEmbedTest

# Futtatás
`ng serve`
`ng serve --port 1337`

# Telepítés
Szükség van egy kofax iframe url-re. Legegyszerűbben így tudod megszerezni:
1. Vizmu dev megnyitása: `http://192.168.62.110:8003/`
2. Új létrehozása / Új dokumentum
3. Kötelező mezők kitöltése, "Rendben" gomb
4. Dokument előállítása folyamatban töltőképernyőt megvárni
5. Ha betöltött az iframe, F12 vagy CTRL + SHIFT + i
6. Kikeresed az iframe-et az "Elements" fülön 
7. kimásolod az src url-t 
8. az angular app "Kofax iframe URL" részébe beilleszted  
    - VAGY: az app.component.ts-ben átírod az `url` változót

Ilyesmit keress:
```
<iframe src="http://192.168.62.91:8081/start/doxshare/doxshare.html
?workflowid=workflow_camunda%7Ckofax%7Cac5eda7b-7a06-11e8-8c3d-0242ac110009
&amp;sessionid=14488c34eefab84c1fc671b5c1e37a9c34fa078d" 
style="display: flex; width: 100%; flex: 10000 1 0%;"></iframe>
```


# Működés
Az Angular alkalmazás és az iframe nem egy domain-en vannak. A [Same Origin Scripting Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) miatt nem tudnak közvetlenül kommunikálni. Emiatt egy kommunikációs hidat építünk fel, a [window.postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) segítségével a **böngészőn keresztül** a következőképp:
- Az Angular app tartalmaz egy referenciát az iframe-re: `this.kofaxIFrame`.
  - Az induláskor elkezdünk üzenetet küldeni az iframe felé a `postMessage` segítségével.
  - Amíg nem érkezik válasz(= amíg nem tölt be az iframe app), addig x. időnként újra próbálkozunk.
- Az Angular app definiál `HostListener`-t a window.message eseményre.
  - Ezen az `EventListener`-en keresztül figyeljük hogy az iframe küld-e felénk üzenetet.
  
- A kofax-ba a projekten található, /kofax-config/services.js fájl a 
  - 192.168.62.91 jelenleg a távoli asztali kapcsolat (INNODOX\Administrator és Pr1n7s0f7)
  - `Apache Software Foundation \ Tomcat 8.5 \ instance-CCMRuntime-5.1 \ webapps-CM \ start \ doxshare \ jsdoxshare` mappában ül
  - Ezt a szkriptet hívja be a kofax induláskor, ezért ha ide beillesztünk JS kódot, akkor az bekerül az alkalmazásba.
- A `services.js` fájlban a kód a következőket csinálja:
   - `window.onload` rész azért kell, mert meg kell hogy várjuk majd az AJAX lekérések elkapásához a jQuery betöltését. 
     -  Ebben a függvényben van definiálva a "dokumentum készen áll" response figyelése a backend-től.
   - Definiálunk szintén egy `eventListener`-t, méghozzá `recieveMessage` néven (és alatta beregisztráljuk)
     - Ez a függvény az első kapott üzenet során lementi a parent adatait, hogy később tudjon neki értesétíseket küldeni. 
     
     
A működés lépései:
1. Elindul az **Angular** keret, elkezdi poll-ozni az **iframe**-et, hogy elindult-e már? (*1. fázis*)
2. A kofax-os **iframe** app elindul, és az egyik ilyen üzenetet elkapja, amire válaszol (*2. fázis*), és lementi a saját szülő **Angular** app-jának az elérhetőségét
3. A kofax-os **iframe** app, miután betöltött a jQuery, elkezdi hallgatni a saját kommunikációját a szerverrel.
4. Ha a kofax(**iframe**)-backend kommunikáció során érkezik egy olyan `response`, ami a "dokumentum elkészült" státuszra utal, üzenetet küld az **Angular** app-nak (*3. fázis*).

# FYI
- Ha valaki nézi a konzolt: 
  - elsőnek jön egy "webpackOK" message az iframe-ből, ekkor még valójában nincs betöltve a kofax. 
  - Az Ajax betöltés után próbálunk szólni az iframe-ből az Angular-nak, hogy betöltődtünk: de legtöbb esetben ekkor még nem történ meg a kapcsolat felvétele az Angular-tól az iframe-felé, ezért nem lesz még meg a referenciánk.
    - (logikailag úgy éreztem fontos látni azért, egyébként ezt el is lehetne hagyni a jelenlegi működés (első üzenet fogadásakor válaszolok a backend-nek) mellett. 
- a kódban több helyen kikommentezve található pár olyan megoldás, ami a jelenlegi implementációban nem szükséges,  de hasznos / tanulságos.
  - pl. a natív Ajax kommunikáció hallgatózás, vagy gombok klikkelésére való feliratkozás. 
- **Fontos**: minden `message recieve` függvényt elvileg azzal kéne kezdeni, hogy a message-ek fogadása során, ha már majd ismerjük a kofax URL-jét, akkor ellenőrizzünk rá, hogy tőle jött-e a message, mert kártékny más alkalmazások itt meg tudják támadni az appot! 
 
