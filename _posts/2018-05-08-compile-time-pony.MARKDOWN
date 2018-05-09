---
layout: post
author: Erik Malm
title: Compile time - PonyLang del 1
image: pony.png
caption: Pony, ett språk för färskt för att användas?
shortexcerpt: Det våras för kompilerade språk. En inbiten java-utvecklare ser om det verkar vettigt att skippa bytecode genom att titta på så många han hinner. 
featured: true
excerpt_separator: <!--excerpt-->
---

# Compile time - PonyC

Det är dags för denne inbitne Java-utvecklare att titta på språk som körs kompilerade till nativa binärer.  

## Compile curious
Planen är att kika på flera nyare kompilerade språk där ordningen avgörs helt av hur nyfiken jag är på respektive språk.  
Målsättningen är att sätta sig in hjälpligt i ett språk i taget under året som kommer där den fritid som finns tillgänglig avgör hur många språk som får sig en genomgång.  
För varje språk kommer jag att kolla innehålla setup av tooling kring språket, kod för en enkel "Hello world" och sedan ett mindre kodprojekt för att ge vana och titta på språkets styrkor.  
Utöver dessa delar diskuteras det som artikelförfattaren rent subjektivt tycker är intressantast med varje specifikt kodbyggande.

Som borde vara uppenbart med fritidsinlärning som grund är det inte frågan om den ultimata sanningen utan mer om en första smak. Lite som att stå vid en buffè och smaka på allt innan man slutligen bestämmer sig för vad som ska tas med till bordet.

### De nykläckta språken.
Språken som intresserar mig personligen sammanfattas i listan nedan, som sagt ordnad med fokus nyfikenhet. 

- `Pony / PonyLang` - Objekt orienterat, statiskt typat, type safe språk fokuserat på actor-modellen. Känns lite som ett snabbt kompilerbart Erlang, eller Akka Actors närmare metallen. 
Verkar dock ha en syntax som gör det enkelt att jobba med, och ett fokus på pragmatisk utveckling som tilltalar mig. 
- `Rust` - Mozillas C/C++ ersättare garanterar minnessäkerhet vid kompilation och gör det enkelt att flertråda. Sägs det. Själva språket rör sig fortfarande snabbt, men mycket p.g.a. av bra omkringliggande tooling (i.e. RustUp och Cargo) är ekosystemet lätt att jobba med. Rust dogfoodas hårt av Mozilla och känns klart värt att titta på. 
- `GoLang` - Googles c/c++/java ersättare med fokus på flertrådning som standard. Används brett även utanför Google, t.ex. inom Canonical, Ubuntus föräldraföretag. Brist på generics känns spontant som största bristen, men ska bli intressant att se hur detta påverkar enklare kod i praktiken.
- `Kotlin Native` - Kotlin utvecklas av JetBrains som en Java ersättare och ger en syntax som är någonstans mitt emellan Scala och Java. Jetbrains har nu även satsat på Kotlin som ett LLVM/Clang kompilerat språk, med interoperabilitet med C / C++ världen. Enkel syntax för en Java-utvecklare men med frågan om likheten med JVM språk hjälper eller stjälper i slutändan.
- Om tiden räcker vore en koll på `Scala Native`  en bonus. Även detta baserat på LLVM / Clang. Något låg utvecklingstakt och än så länge enkeltrådat men intressant för vad det i förlängningen kan bli. 

## Java-vill-du?
Som utvecklare har jag än så länge mer eller mindre främst sysslat med språk som inte kompileras till nativa binärer.
Först inom ren dataprocessning med R och Python (2.7 for life!), sedan inom backend-utveckling med Java och Scala (som kompilerar till bytecode,tolkas av en JVM samt bitvis kompileras med en JIT compiler till nativ kod) samt olika varianter av JavaScript (Sååå många frameworks. Så mycket JIT).
Har dock börjat dyka upp en hel del use cases för mig där det är klart bättre att köra native, flera av mina mindre projekt kör på ARM-developer boards runtom i huset. Detta sätter JVM:ens minneshantering på sin spets. Att manuellt behöva specificera ett maxanvändande och min-användande för minne är t.ex. något jag gärna vill komma bort ifrån när det totala tillgängliga ram:et är på ca 1 gb.  
I andra fall handlar det om rena command line utilities där man helst vill slippa jvm:ens uppstart.

# Först ut - PonyC
Givetvis har man alltid egentligen velat ha en Pony, detta tillsammans med mängden pony-relaterade gif:ar som finns där ute gör att det PonyLang är först ut. 
<!--excerpt-->

## Att skaffa en Pony - Installation och tooling
{% include image.html image='pony.png' caption='Pony är ett ungt språk, detta märks på det tunna ekosystemet' %}

Ponyc kompilatorn distribueras på nästintill varje sätt som går. 
Som docker-container, .deb, .rpm, nix-paket, via Homebrew, Bintray och Linuxbrew. Än så länge ingen distribution via Flatpak eller Snap-paket, men det är få system som inte har en enkel installationsmetod för Ponyc kompilatorn. 
 
Efter att ha testat att kompilera hello-world (nedan) med docker-ponyc gick jag vidare med en helt vanlig .deb installation. Allt väl såhär långt.

Pony är ett ungt språk och detta visar sig i den totala bristen på en mer integrerad IDE, som tur är
Dock finns en syntax-highlighting plugin för Vim (phew!) och tillsammans med YouCompleteMe kan man utveckla någorlunda effektivt. Liknande syntax highlighters verkar också finnas för bland annat visual studio code. 
Debugging sker också enbart via command-line utilities.

Back to basics här helt enkelt.

Pony har en dedikerad dependency manager vid namn pony-stable, som dock visar sig betydligt svårare att installera än själva pony kompilatorn. Som tur är ligger dependency hantering utanför ambitionen för inledande pony projekt.

# Pony basics
Här följer koncept kring PonyC som jag själv tycker gör koden nedan enklare att läsa.  
Först och främst:  
 Pony är ett starkt typat, objekt-baserat och kompilerat språk, viss type inference finns men inget dynamiskt så långt ögat når.

Utöver detta har Pony riktigt intressanta egenskaper.  
Kompilerade pony program är garanterade att vara  data-race och därmed deadlock-fria (!) och för detta finns till och med matematiska bevis.
Detta är i sig själv enormt, att debugga deadlocks och data-races är i bästa fall svårt och i värsta fall något som får en person att gå över till något mer värdigt istället för programmering - som att skyffla gödsel.

Detta ger också att språket kan vara garbage-collected utan de pauser som t.ex. JVM:ens garbage collection ger, då isolationen av variabler i minnet kan tas för given.

### Actor like you mean it.
PonyC är helt baserat på actor-modellen, dvs uppbyggt av ett antal enkeltrådade processer som kommunicerar genom att skicka meddelanden till varandra. Meddelanden processas av mottagar-actor i mottagande-ordning, ren FIFO. 
Pony sköter scheduling av meddelanden på samtliga tillgängliga trådar, en actor är alltså inte låst till en specifik tråd.  
En actor som inte tar emot meddelanden har en extremt liten overhead och Pony-program kan därför skrivas med hundratusentals actors i olika roller.

PonyC har öven ett inbyggt backpressure beteende som begränsar actors från att skicka meddelanden snabbare än mottagande actor kan konsumera den, detta kan tweakas av den insatte men ger även out-of-the box en stor lättnad för ett av de svårare problemen med effektiv actor-baserad kod.

#### Inget blocking
Inga blocking operationer finns i pony, så inga `Thread.sleep` någonstans. Även operationer såsom fil I/O körs i icke blockerande actors. 
{% include image.html image='pony-concurrent.png' caption='Precis som riktiga ponys löper PonyLang parallelt i flera fållor.' %}
## Variabler
PonyC har två typer av variabler - mutable och immutable.
Variabler är alltid typade, med vissa hjälpsamma mellanlägen i form av Unions (tas upp i nästa blog i serien).
Ponys mutability system innehåller också en uppsättning "reference capabilities" som täcks längre ned.
```
var mutable:String = "Change we can believe in!" //Mutable string variable
mutable = "Yes we can!" //OK!

let immutable = "Make America great again!" //Very conservative immutable variable
immutable = "I know the human being and fish can coexist peacefully" //Will cause compile time error
```
Variabelreferenser kan också konsumeras vilket lämnar den tidigare referensen tom.  
`let a = consume b`  lämnar variabel b tom och ger kompileringsfel om denna används efter detta.

### Destructive read variabel=variabel=variabel
Assignment av en variabel returnerar i pony det gamla värdet från variabeln längst till vänster i en assignment, sk. destructive read (?), dvs `a=b`kommer att returnera `a`:s värde innan assignment. Detta gäller för varje assignment från höger till vänster, `c=a=b` ger `c` det gamla värdet av `a`.  
Reassignments kan därmed ofta göras utan temporära mellanvariabler. 
Ett exempel:
```javascript
        var johan = "Johan"
        var jassyr = "Jassyr"
        var joakim = "Joakim"
        
        let tmp=johan=jassyr=joakim=joakim+"- med lite extra text"
        
        env.out.print("tmp:"+tmp+", johan:"+johan+", jassyr:"+jassyr+", joakim:"+joakim)
        //Output: tmp:Johan, johan:Jassyr, jassyr:Joakim, joakim:Joakim- med lite extra text
```

## Expressions

If-satser, loopar etc är expressions och går att kombinera.  
Loopar returnerar det sista värdet som går igenom dem och if satser beroende på vad de utvärderas till.  

Ett exempel:
```javascript
let check:Bool = true
let count:U32 = 100 + if (24 + if check then 100 else 24 end) > 100 then 20 else 0 end 
```
Kommer att ge `count == 244` då båda if statements kommer att utvärderas som sanna. 
Om man istället satte `let check:Bool=false` skulle utfallet bli `count=148`.

### Object capabilities
För att säkerställa trådsäkerhet har PonyC ett antal varianter på värden.
Förutom en typ har även en variabel en access-typ, en reference capability.  
Reference capabilities avgör hur ett objekt kan läsas från, skrivas till eller refereras till av olika variabler.
Detta används för att tvinga koden att bete sig på ett trådsäkert sätt och kollas främst när referenser skickas som meddelanden mellan actosr.
Att deklarera en variabel som 'let' gör t.ex. att variabeln aldrig kan sättas att referera till ett nyt objekt, men att deklarera typen som `let klassNamn: KlassNamn val=KlassNamn(arg1)` gör själva objektet immutable. 

Den här kombon kallas Object-Capability model och ligger till grund för ponys matematiska trådsäkerhet, det gör det möjligt för kompilatorn att enkelt definiera vilka objekt som kan skickas mellan trådar och hur.

Capabilites reglerar också hur många referenser som kan finnas till ett objekt.
T.ex. kan ett objekt med `iso` kapabilitet enbart refereras till av en enda variabel åt gången.

Nedan är en inkomplett lista på variabler som pekar på objekt med varierande reference capabilities:

- `let hello:String ref="I am a ref"` definierar en immutable referens till en mutable sträng. En standard `ref` kan refereras till från flera variabler och ett ref objekt kan modifieras. En ref kan dock aldrig skickas från en actor till en annan eftersom detta skulle medföra risk för samtida modifiering.
- `let hello:String val="Hello!"` definierar strängen hello som en `val` referens. Detta garanterar att datat är immutable och tillåter att objektet kan refereras till av hur många variabler som helst. Eftersom en val inte kan ändras kan den skickas mellan actors utan risk för sidoeffekter. 
Eftersom datat är immutable kan flera trådar läsa från objektet utan att medföra risker för sidoeffekter. 
- `var hello:String iso="Hola!!"` Definierar istället strängen hello som en mutable `iso` , en isolerad referens. I praktiken innebär detta att det i hela programmet enbart kan finnas en referens till objektet. Ett iso objekt kan skickas mellan actors, men kräver då att original-referensen förstörs så att det totalt fortfarande bara finns en referens till objektet. 
Kompilatorn kommer att kontrollera att detta följs.  
Eftersom det enbart finns en samtida referens till ett `iso` objekt finns ingen risk för samtida modifiering eller sidoeffekter.
- `tag` är ytterligare en unik capability.  En `tag`referens till ett objekt ger vare sig skriv eller läsrättigheter men tillåter att objekt kan identifieras. 
En tag kan referera till ett mutable objekt, men eftersom en tag inte tillåter att källobjektet läses av kan den utan problem skickas mellan actors.

## Standard-library och bibliotek
PonyC har ett väldigt litet standard-bibliotek för den som är van vid Java. För den som är van vid C är det enormt med grundfunktionalitet, t.o.m inbyggda HttpServer och Klient-klasser vilket är riktigt användbart för ett par kommande projekt.

Det är dock ont om ponyc tredejparts bibliotek, för den som är van bid Python eller Java är det lite av en öken vilket utlämnar pony åt integration med C.


### Funktioner och Behaviours
Grovt tillyxat hanterar pony två typer av metod-anrop:

`Funktioner` är metoder som anropas med argument, dessa utförs omedelbart vid anrop och kan returnera ett objekt som svar.
`fun giveMeFive():String => "Five"` är en funktion som returnerar strängen "Five".
Även funktioner ingår i Ponys capability concept. 

Om inget anges anses funktioner ha capabilityn `box` vilket innebär att de inte får ändra något state utanför sitt eget interna scope.

T.ex:

```
class Mutable  
    var _mutableString:String
    new create(initialString:String) =>
        _mutableString=initialString
    fun  setString(setToThis:String) =>
        _mutableString=setToThis
```

Kommer att ge ett kompilatorfel eftersom `setString` inte har rätt capability:
```
/home/erik/code/pony-image/main.pony:21:23: cannot write to a field in a box function. If you are trying to change state in a function use fun ref
        _mutableString=setToThis
```

Enbart funktioner med capability `ref` tillåts sätta state. Att ändra setString funktionen enligt nedan kommer att låta kompilatorn göra sitt:
```
    fun ref setString(setToThis:String) =>
        _mutableString=setToThis
```
Även:
```
let mutable:Mutable("Totally mutable")
fun mutateMutable => mutable.setString("This won't compile")
```
Ger kompilationsproblem. 
Default funktionerna i Pony är m.a.o. "pure" functions som inte ger någon förändring i state utan enbart ger ett resultat beroende på mottagna argument.


`Behaviours` är både metod-definitioner som hanterar actors meddelanden och en definition av ett meddelande-typ.
Detta är elegant jämfört med t.ex. Akka-actors där ett stort antal meddelande-definitioner ofta görs som klasser som sedan hanteras av ett Switch / Match-block.
Dessa kan ta emot argument på sätt som liknar en funktion men kan aldrig ha en returtyp.
```
actor Onion
//Behaviour definition
be handleMessage(message:String) => //Ett anrop skickar ett meddelande till Onion, som sedan processas nedanför.
    env.out.print("Ok,ok... I got the message: "+message)
```

Ett anrop till ett behaviour ser ut som ett funktionsanrop men skickar egentligen ett meddelande till mottagarens inkorg.
Den mottagande actorn utför jobbet när den börjar behandla just det meddelandet.

Det kan kan därför vara något oklart i vilken ordning dina anrop kommer att genomföras, om flera olika actors skickar meddelanden process de i den ordning de inkommer.
Inom ett *behaviour* är alla operationer däremot helt synkrona och atomic vs actorns interna state.

Kort sagt:
Anrop till ett behaviour utförs asynkront.
Kod inom ett behaviour utförs alltid helt synkront.
Eftersom parametrar som skickas till ett behaviour processas av en annan tråd får enbart trådsäkra referens-typer skickas
(val, tag eller en iso vars gamla referens konsumeras). 
Det är alltid trådsäkert att anropa ett behaviour från vilken tråd som helst, och dessa kan därmed anropas t.o.m. när det bakomliggande objektet inte kan läsas t.ex. för en `tag`.

Med denna pöldjupa förståelse för pony är det dags att skriva kod.

## Hello ponyc dissektion!
### Hello world
Eftersom Pony är helt baserat på Actor modellen känns det naturligt att vårt "Hello world" är Actor baserat. 
Först ett minimalistiskt exempel:
```javascript
actor Main
    new create(env:Env) =>
        env.out.print("Hello world!")
```
Detta kompileras med `ponyc .` och den resulterande filen kan köras med `./main`
Resultatet är *Hello world* på skärmen. Hitills är allt ganska bekant.

Main-actorn motsvarar vår huvudklass, denna skapas vid uppstart och får en referens till det underliggande systemet med StdOut som en `OutStream`.
Denna används sedan för att skriva ut "Hello world!" i konstruktorn. 

Detta skiljer sig inte särskilt mycket från flertalet andra språk.
Main actorn är för övrigt ponys motsvarighet till en main metod i andra språk. 

### Async exempel
Det enkla "Hello world" exemplet visar inte upp så många av de egenskaper som gör Pony kul att jobba med, det sker inget asynkront i koden.  
Låt oss därför sätta upp ett onödigt komplext exempel.

- Först en  `Hello` actor som kan skriva ut när den mottar meddelanden.
Dessutom innehåller "Hello" actorn en räknare, den kan räknas upp på lite olika sätt och läsas av genom att skicka in ett lambda som anropas med räknarens värde.
```javascript
actor Hello
        var _counter:U32
        let _out:OutStream tag
        new create(out:OutStream tag) =>
        _counter=0
        _out=out

    be world(message:String) => 
        _out.print("Printing message:"+message+ ". Counter at: "+_counter.string())

    be worldAndCount(message:String) => 
        _counter=_counter+1
        _out.print("Printing message:"+message+ ". Counter at: "+_counter.string())

    be readCounter(fn: {(U32)} val) => //Mata inkommande lambda med nuvarande värde på _counter
            fn(this._counter)

    be setCounter(counter:U32) =>  //Sätter räknaren till inkommande värde
            _out.print("Setting counter value! Previous:"+this._counter.string()+", New value:"+counter.string())
            this._counter=counter

    be addToCounter(addMe:U32 val) =>
          let previous = _counter = _counter+addMe
            _out.print("Setting counter value! Previous:"+previous.string()+", New value:"+_counter.string())
```
- Flera `Incrementer` aktörer. 
```javascript
actor Incrementer
    let _hello:Hello tag 
    new create(hello:Hello) => 
        _hello = hello
    be asyncIncrement() =>
        _hello.readCounter({(counter:U32) => 
        let newValue = counter+1
        _hello.setCounter(newValue)
        } val)
    be syncIncrement() =>
        _hello.addToCounter(1)
```
Varje incrementer har behaviours för att räkna upp räknaren i `Hello` 
1.  `asyncIncrement` skickar ett meddelande med ett lambda till readCounter behaviour i `Hello` actorn. 
Lambdat läser Hello-räknarens värde, lägger til ett och anropar behaviour `setCounter` för att sätta nytt värde. 
1. `syncIncrement` anropar `addToCounter` beahaviour i hello med `1` och ökar därmed räknaren.

 
- Slutligen beter sig `Main` actorn beter på detta viset:
```javascript
actor Main
    let _env:Env 
    let _hello:Hello

    let helloMessage:String = "Message in a bottle"
    new create(env: Env) => 
        _hello = Hello(env.out)
        _env = env
    for i in collections.Range(0,5) do
             //Incrementer(_hello).syncIncrement()
             Incrementer(_hello).asyncIncrement()
            _hello.world(helloMessage)
        end
    be hello(message:String) => 
        _env.out.print(message)
```

1. Main Actorn skapar en `Hello` actor.
1. Main actorn går in i en for loop i fem cykler.
I varje cykel sker följande:  
1. En `Incrementer` actor skapas och `asyncIncrement` behaviour på denna anropas.
1. Ett meddelande skickas till `Hello` actorn som den ska skriva ut. Detta skriver även ut det nuvarande värdet i räknaren.

Detta hello-world exempel är även flertrådat.  
De involverade actorsen kör per default på ett antal trådar som är lika stort som antalet kärnor, 4 i detta fall, men detta kan ökas eller minskas efter behag.

Här är den kompletta koden för vårt actor baserade exempel:

```javascript
use collections = "collections"

actor Main
    let _env:Env 
    let _hello:Hello

    let helloMessage:String = "Message in a bottle"
    new create(env: Env) => 
        _hello = Hello(env.out)
        _env = env
    for i in collections.Range(0,5) do
             //Incrementer(_hello).syncIncrement()
             Incrementer(_hello).asyncIncrement()
            _hello.world(helloMessage)
        end
    be hello(message:String) => 
        _env.out.print(message)


actor Hello
        var _counter:U32

        let _out:OutStream tag
        new create(out:OutStream tag) =>
        _counter=0
        _out=out

    be world(message:String) => 
        _out.print("Printing message:"+message+ ". Counter at: "+_counter.string())

    be worldAndCount(message:String) => 
        _counter=_counter+1
        _out.print("Printing message:"+message+ ". Counter at: "+_counter.string())
    be addToCounter(addMe:U32 val) =>
          let previous = _counter = _counter+addMe
            _out.print("Setting counter value! Previous:"+previous.string()+", New value:"+_counter.string())
    be readCounter(fn: {(U32)} val) => 
            fn(this._counter)

    be setCounter(counter:U32) => 
            _out.print("Setting counter value! Previous:"+this._counter.string()+", New value:"+counter.string())
            this._counter=counter


actor Incrementer
    let _hello:Hello tag 
    new create(hello:Hello) => 
        _hello = hello
    be asyncIncrement() =>
        _hello.readCounter({(counter:U32) => 
        let newValue = counter+1
        _hello.setCounter(newValue)
        } val)
    be syncIncrement() =>
        _hello.addToCounter(1)
```


Output från detta program blir:

```bash
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Setting counter value! Previous:0, New value:1
Setting counter value! Previous:1, New value:1
Setting counter value! Previous:1, New value:1
Setting counter value! Previous:1, New value:1
Setting counter value! Previous:1, New value:1
```

Den här outputen förklaras av:
1. `Main` actorn kör en for loop
1. I varje vända av loopen skapas en ny Incrementer actor och ett meddelande skickas till den nyskapade incrementern's `asynIncrement` behaviour.
 Denna incrementer skickar då ett meddelanden till `Hello` actorn.
 Detta extra steg i skickande av meddelanden till `Hello` gör att samtliga meddelanden från incrementers behandlas efter medelandena från `Main`.
 1. `Incrementer`  skickar ett lambda som  läser av det nuvarande värdet på _counter variabeln och använder den informationen att skicka ett meddelande för att sätta räknaren till `counter+1`.  Samtliga meddelanden som läser av värdet behandlas av `Hello` innan det första om sätter ett nytt värde.  
 Därmed är räknaren alltid på `0` när `_counter+1` beräknas. 
Därmed blir värdet på _counter aldrig större än ett.

Detta eftesom avläsning och skrivning sker i två olika behaviours.  

Samma program med en liten ändring i for-loopen i `Main`ger ett annat output: 
Dvs. vi ändrar for-loopen `Main` till:
```javascript
    for i in collections.Range(0,5) do
             //Incrementer(_hello).asyncIncrement()
             Incrementer(_hello).syncIncrement()
            _hello.world(helloMessage)
        end
```
Output:
```bash
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Printing message:Message in a bottle. Counter at: 0
Setting counter value! Previous:0, New value:1
Setting counter value! Previous:1, New value:2
Setting counter value! Previous:2, New value:3
Setting counter value! Previous:3, New value:4
Setting counter value! Previous:4, New value:5
```

Detta visar att:
1. Meddelandena till `.world` når fortfarande till `Hello` actorn före samtliga meddelanden till `addToCounter` som ökar räknaren.
1. Eftersom både avläsning och inkrementeringen av räknaren sker i samma behaviour i `Hello` actorn blir räknaren ett större per meddelande.

Slutligen kan vi ändra loopen till att anropa `worldAndCount`, för att räkna upp `_counter` och skriva meddelande i samma behaviour:
```javascript
    for i in collections.Range(0,5) do
             //Incrementer(_hello).asyncIncrement()
             Incrementer(_hello).syncIncrement()
            _hello.worldAndCount(helloMessage)
    end
```

Vilket ger output:

```
Printing message:Message in a bottle. Counter at: 1
Printing message:Message in a bottle. Counter at: 2
Printing message:Message in a bottle. Counter at: 3
Printing message:Message in a bottle. Counter at: 4
Printing message:Message in a bottle. Counter at: 5
Setting counter value! Previous:5, New value:6
Setting counter value! Previous:6, New value:7
Setting counter value! Previous:7, New value:8
Setting counter value! Previous:8, New value:9
Setting counter value! Previous:9, New value:10
```
Av detta kan vi bekräfta flera saker vi redan visste om Pony: 
1. En actor alltid är enkeltrådad och utför ett behaviour i taget.
1. Ett behaviour (eller en funktion) är enkeltråd.
1. Att meddelanden anländer i en ordning som inte alltid är lätt att avgöra.

Ett bra pony program kan göras parallelt väldigt enkelt så länge som den ordnign behaviours utförs i inte påverkar resultatet.

# Pony - nästa blogpost
Nästa blog-post tittar på:
### Lite mer språkdiskussion

- Klasshantering i Pony med Trait, Interface, Class och Actor 
- Lambdas 
- Typer och hur de relaterar till klasser, traits och interfaces.
- Foreign funtion interface - Pony kan enkelt interagera med C bibliotek.
- Pony:s kodsyntax, hur strukturerar man kod i pony?

###  Mer kodprojekt.

Ett tappert försök att kombinera alla ponys språkidéer till faktisk fungerande mjukvara.
En länk till del 2 kommer att placeras här.

