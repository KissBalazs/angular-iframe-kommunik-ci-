# Angular - iframe kommunikáció by Forest

## Feladat:
Adott egy **Angular** alkalmazás, ami tartalmaz egy `<iframe/>`-et, amely mögött egy külön domainen található alkalmazás fut.
A feladat: a szülő app és az `iframe`-be **beágyazott** alkalmazás között valósítsunk meg kommunikációt.

![Első kép](001.png "a")

**Miért nehéz ez?** 
Mert az Angular alkalmazás és az iframe nem egy domain-en vannak. A [Same Origin Scripting Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) 
miatt nem tudnak közvetlenül kommunikálni. Emiatt egy kommunikációs hidat építünk fel, a [window.postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) 
segítségével a **böngészőn keresztül** a következőképp:

## Előkészület 
Igazából mind1, hogy milyen **JavaScript** vagy egyéb keretrendszerről beszélünk, mert az elv azonos mindennél. Az én példámban Angular app volt a szülő, ezért abban **TypeScript**, a beágyazottban meg **natív JavaScript** kóddal oldottam meg. Ami a fontos, hogy a következő dolgokra lehetőségünk legyen a két alkamazásban:
- a **szülő** alkalmazásban legyen lehetőségünk referenciát szereznünk az `<iframe>`-re
- a **beégyazott** alkalmazás kódjához szintén férjünk hozzá és tudjuk kiegészíteni azt. 

Ha ez a két dolog adott, akkor mindent meg tudunk csinálni. :)  

Egyébként:
- Ha egy domain-en van a két app, akkor közvetlenül "belelátunk" a szülővel az `iframe`-be. 
- De ha nem, akkor bele kell tudnunk írni logikát a beágyazott kódba. 
- Ha egyik sem áll módunkban, akkor az a tipikus így járás esete :( (lásd: same origin policy).

## Működés lépései
Első végiggondolásra valami ilyesminek érezzük a kommunikáció lépéseit:
1. Mindkét alkalmazás felépül, elindul, definiáljuk a listenereket az üzenetek fogadására.
2. Az **Angular** alkalmazás üzenetet küld az **iframe**-nek, a saját origin információival.
3. Az **iframe** alkalmazás miután megismerte a saját szülőjét, vissza tud küldeni üzeneteket.

**Viszont!**  
- A kommunikáció **elsőnek egyirányú**, ugyanis - ha végig gondoljuk nyilván - a beágyazott app nem tudja hogy ő pontosan hova és kibe van beágyazva.
  - Ezért elsőnek a szülő app-nak kell felvenni a kapcsolatot a beágyazott alkalmazással (mert neki van hivatkozása az iframe-re). 
- A kommunikáció továbbá **késleltetett**, ugyanis - legalábbis az Angular keretrendszer esetén, de gyanúsan mindenhol is - az `iframe`-be **később tölt be a kód, mint a szülőbe 
és így később is indul el.**. 
  - Így ez a kézfogás valójában egy x. időnkénti poll-ozás, az első válaszig.
  
Tehát a lépések erre módosulnak:
1. Az **Angular** alkalmazás felépül, és elkezdi poll-ozni a **iframe**-et egy "kézfogás" üzenettel.
2. Az **iframe** egyszercsak elindul, és elkapja az egyik ilyen üzenetet.
3. Az **iframe** a kézfogásra válaszol, az **Angular** app leáll a poll-ozással
4. készen állnak a kommunikációra.

## Kód
### 1. Angular

HTML: 
Angular-os ref az iframe tag-re.
```html
    <iframe #iframeRef></iframe>
```

TS:
- Elsőként láthatjuk a `@ViewChild` hivatkozás definíciót, ami a rendering utántól érvényes objektumot ad vissza.
  - Ezért az első üzenetet őfelé az `ngOnInit` függvényben tudjuk küldeni. (ekkor már létezik az `<iframe>` tag, de a benne futó app még nem indult el!)
- Kommentezve is oda van, de külön kiemelném, hogy fontos, hogy megadjunk egy origin-t, és ne csillagot ha már tudjuk fixen a domain-eket.
  - Mert egyébként nem tudjuk, hogy honnan is jött _valójában_ az üzenet
- Láthatjuk a `postMessage()` szintaxisát, [itt egy leírás](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) hozzá.
  - A példa egyszerűsége miatt csak stringet küldök, de lehetne komplexebb adatokat is, ha megfigyelitek, az üres tömb helyett
- Definiálunk egy `HostListener`-t, amely a nekünk küldött üzenet eseményt fogja figyelni.
  - Ez az "Angular"-os módja egyébként a `document.addEventListener()` JS szintaxisnak (mint látni fogjuk), de az is működne elvileg. . 
- Indulás után nekilátunk pollozni az iframe-et. 
  - Mivel ez nem "igazi" pollozás, hanem valójában csak egy Message queue-ba "levél-feladás", így nekünk kell megírni azt, hogy válaszoljon rá az iframe.
  - **Csak az általunk definiált message-re** állítsuk le a pollozást, mert egyébként jön más üzenet is, amit még nem az iframe-en belül szereplő app küld.
    - Pl. nálam a Webpack DEV szerver küld egy "WebpackOK" üzenetet induláskor. Ezt ugye nyilván nem az iframe küldi, mégis becsorog az üzenetfigyelő függvénybe: az ilyenekre fel kell készülni.  
```JS
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html'
})
export class AppComponent {

  @ViewChild('iframeRef') iframeRef: ElementRef;

  status = 1;
  iframePollingtimer = null;
    iframePollingtimer = null;
    pollingIntervals = 1000;

  @HostListener('window:message', ['$event'])
  hostListenermessageHandler(event) {
    this.receiveMessage(event);
  }

  ngOnInit(): void {
    this.initStartupPolling();
  }

 initStartupPolling() {
    this.iframePollingtimer = setInterval(() => {
      console.log('Trying to reach iframe...');
      this.sendMessage();
    }, this.pollingIntervals);
  }

  sendMessage() {
    console.log('Sending message: Angular --> iframe ');
    // todo: Always specify an exact target origin, not *, when you use postMessage to send data to other windows.
    // malicious site can change the location of the window without your knowledge,
    // and therefore it can intercept the data sent using postMessage.
    this.iframeRef.nativeElement.contentWindow.postMessage('Üzenet az iframe-nek', '*', []);
  }

 public receiveMessage(event) {
    // Do we trust the sender of this message?  (might be
    // different from what we originally opened, for example). todo
    // if (event.origin !== 'http://example.com')
    //   return;

    console.log('Received a message From: iframe', event);
    if (event && event.data && typeof event.data === 'string') {
      // fontos: itt kell csak megállítani a poll-ozást!
      clearInterval(this.iframePollingtimer);
    }
  }
}

```

### 2. Iframe JavaScript:

- Ezek a kódok bele vannak definiálva az én konkrét példámnál egy önmagát egyszer meghívó függvénybe. 
- window.onload azért kell, mert jQuery segítségével hallgatóztam az iframe által elküldött XMLHTTPRequest-ekre.
  - ez most nem szorosan a problémakör része, csak egy esemény példa hogy lássuk, mikor törénik meg egy event (tehát ez lehetne egy onclick handler is, stb.), ami kiváltásával az iframe majd üzen vissza a szülőnek.
- definiljuk a send, meg accept függvényeket, és beregisztráljuk őket a `window` alá
- Tárolnunk kell a `parentReference` és `parentOrigin` értékeket 
  - vagy fixen beleégetni őket a kódba: igazából úgy érzem az lenne a biztonságosabb, mivel egyébként is hasznos csak azzal leállni bármit kommunikálni, akit megbízunk.
  - Láthatjuk azt is, hogy - mivel a példában csak egy üzenetet küld az angular - egyből válaszolunk is rá egy "okés, elindultam" messsage-el.

```JS
 window.onload = function(){
    console.log("onload complete, Ajax libary is ready to use.")

    $(document).ajaxComplete(function(event, request, settings){
      if(request.responseJSON.notification.status.toLowerCase() === 'completed'	){
        console.log("Completed status response detected, contacting frontend with the message...");
        sendDataToParent("code_1_document_created");
      }
    });
  };


  var parentReference = null;
  var parentOrigin = null;
  function receiveMessage(event){
    console.log("Received a message From: Angular", event);

    // if (event.origin !== "http://example.org:8080")
    // return;
    // todo: this is crucial for being secure!

    parentReference = event.source;
    parentOrigin = event.origin;
    sendDataToParent("code_0_iframe_started");
  }

  window.addEventListener("message", receiveMessage, false);


  function sendDataToParent(text){
    if(parentReference && parentOrigin){
      console.log("Sending message: iframe --> Angular")
      parentReference.postMessage(text, parentOrigin, []);
    } else {
      console.warn("Parent reference in the iframe is still undefined!");
    }
  }
```

Szóval to wrap it up:
- A két alkalmazás külön domain-en fut, a szülőbe be van ágyazva egy `iframe`-be a másik. 
- Két alkalmazás a böngészőn keresztül, `postMessage()` és message `eventListener` segítségével kommunikál.
- A szülőnek kell kezdeményeznie a kommmunikáció a beágyazott felé.
- A beágyazott kód késleltetve indul el. 
- A kommunikáció során illik az origin-t leellenőrizni.
