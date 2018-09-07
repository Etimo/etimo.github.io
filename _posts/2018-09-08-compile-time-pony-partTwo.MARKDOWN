---
layout: post
author: Erik Malm
title: Compile time - PonyLang del 2
image: ponyToo.png
caption: Pony syntax och språkidéer
shortexcerpt: En enkel genomgång av Ponys syntax och egenheter innan det är dags för att faktiskt producera något. 
tag: [featured,compile-time,pony]
categories: [Compiled, Pony,Featured,Compile time]
excerpt_separator: <!--excerpt-->
---

Compile Time är en blog-serie där en van Java-utvecklare breddar vyerna genom att bada i kompilerade språk.
För att få mer bakgrund kring Pony och blogserien kan det vara trevligt att börja med [del 1 av bloggen - En titt på Pony]({% post_url 2018-04-23-compile-time-pony %}).

# Pony part two - Electric bogaloo
Del 2 av titten på pony tittar övergripande på ponys språksyntax, typsystem och interaktion med C bibliotek.  

# Syntax
Pony har en relativt intressant kodstruktur, glöm allt om indentering a-la Python eller måsvingar från JVM språken (fast kanske inte riktigt allt). 
I Pony avgränsas metoder kod med nyckelord och rad-positionering. Enbart. 
Resultatet är en tydlig avgränsning av komponenter som helt saknar möjligheter att bygga en alltför komplex hierarki.
En klass kan aldrig definieras i en klass, eller en actor i en actor t.ex.

Nedan är ett exempel som illustrerar deklaration av en actor med flera fält, behaviours och funktioner:

```javascript
actor Main //Klasser och actors definieras med ett start keyword.
let fieldOne:U32 = 1 // Fält definieras innan konstruktor.
let stringMeAlong:String =" me along"; let oneMore:U32 = 2 
let _env:Env //Det 
new create(env:Env) => //First konstruktor
env.out.print("This has a scope in the constructor")
let local="I am in constructor scope" //Indentering har inget med scope att göra.
let localToo = "I am also in constructor scope"

be aggressive(message:String) =>
_env.out.print(" Be be "+message) //Det här är nu i ett nytt scope.

fun newScope() =>
1+1 //Nytt scope, nu en metod .
fun newestScope() =>
1+1 //Nytt scope för ytterligare en metod.

class ClassicClass //Nytt scope, nu en klass
let fieldInClass
new create(field:String) =>
fieldInClass = field
```

<!--excerpt-->
 Exemplet illustrerar bl.a. att:
- Metoder och behaviours, samt klasser antas fortgå tills dess att något annat nyckelord bryter deras kontext.
D.v.s. dessa startas med ett `actor`, `class`, `fun` eller `be` nyckelord och antas fortsätt tills nästa `actor`,`class`, `fun` eller `be` nyckelord dyker upp i koden. 
Även `if`, `for`, `while` och `try` loopar avgränsas med nyckelord, dessa avslutas med nyckelorder `end`. 

- Fältdeklarationer, alltså variabler som hör till ett objekt snarare än lokalt metod-scope kan enbart definieras i en klass/ actor innan man har definierat den första konstruktorn.  
Alla variabler deklarerade efter detta kommer att få ett lokalt scope. Detta kan ses i Main-actor samt i klassen ClassicClass
Ett fält kan aldrig vara utan värde, ett värde måste anges antingen vid deklaration eller i konstruktorn. Ett fält kan aldrig lämnas tomt av en konstruktor.

- Privata fält definieras genom att namnge dessa med ett `_` i början, t.ex. `let _privat:String="Private`.  

- Kroppen i metoder, behaviours, och lambdas startas genom en dubbel-pil `=> //logik här` 

- Statements på samma rad kan separeras med ett `;`.

Pony ger garantier kring att ren pony-kod aldrig krashar, och därför måste eventuella fel hanteras explicit.
- Metoder som kan ge fel måste anropas med ett följande `?`. Ett `try` block kan användas för att styra kodflödet vid ett fel, pony kommer vid ett fel att leta sig uppåt i anrops-stacken till det första inneslutande try-blocket. Kompilatorn kommer att kontrollera att ingen metod som kan generera fel anropas utan ett motsvarande 'try' block längre upp i flödet.

```javascript
if 2 > 1 then 3 else 50 end
for name in list.values() do env.out.println("Out") end
try methodThatMightCauseError("Error parameter")? else env.out.print("There was an error!")
```

# Typer
Pony har ett typsystem som skiljer sig något från språk som Java eller C#.
Bl.a. så har PonyC har ingen möjlighet till klassisk inheritance mellan klasser.
Ponys subtyping är istället helt baserad på Traits och Interfaces. 
## Trait och Interface
### Traits och nominell subtyping
`Trait` är det språkkonstrukt som närmast motsvarar ett `Interface` i Java.
Detta fastställer ett kontrakt om vilka metoder en klass måste implementera för att kunna vara en subtyp. Likt interfaces från Java 8 och framåt kan traits även innhålla default-implementationer av.

En klass kan sedan deklareras med en implementation av ett `Trait` och den klassen är då en subtyp.

```javascript
trait ExampleTrait
    fun doSomethingFunction():U32
    fun identify(env:Env) =>
        env.out.print("I am a default implementation")

class ImplementeraMera is ExampleTrait //Traits måste specificeras  vid klassdeklaration.
    fun doSomethingFunction() =>
        1+1 //Doing something
    //identify använder default implementationen.
Ett trait måste alltså deklareras för att användas, vilket inte är fallet med interfaces.
```

### Interfaces och strukturell subtyping
Pony implementerar även `Interfaces` som är funktionellt lika `Traits`. En klass behöver dock inte deklareras med ett explicit beroende på ett interface.  
Det räcker att en klass implementerar de metoder som specificeras i ett interface för att räknas som en subtyp. Subtypningen sker m.a.o. med avseende på klassens struktur, strukturell subtyping. 
Interfaces kan även de innehålla default implementations, men dessa användands enbart vid en `is InterfaceName` deklaration.

```javascript
//Interface exempel
interface ExampleInterface 
    fun name():String

class Named
    fun name():String => "I am Bob" //Klassen named är nu en subtyp av ExampleInterface
```
Interfaces kan därmed ge möjlighet att använda generics utan att skriva om klasser, alla metoder som innehåller en `string()` metod i Pony, räknas t.ex. automatiskt som en variant av `Stringable` interfacet och kan därmed användas på samma sätt runt om i koden.

### Lambdan
En lambda-funktion i Pony deklareras som funktioner men inneslutna i `{}` vilket är den enda explicita scoping med parenteser som verkar finnas i språket.
En lambda funktion som anropas med två U32 integers och returnerar en sträng deklareras t.ex.:

```javascript
{(one:U32,two:U32):String => one.string()+two.string()} 
```
### Unions, Intersektions och Tuples och typalias.
Pony har sin egen twist på flera vanliga objekt, t.ex. tuples.
I Pony kan dessa även användas som typalias definitioner som sedan kan återanvändas i koden.
- Tuples: Används för att samla flera värden i ett objekt på ett standardiserat sätt. 

```javascript
let tuple:(U32,U32,String) = (1,3,"String) //Direct use of a tuple.
//Or....
type IntIntString is (U32,U32,String)
let typeAlias:IntintString = (2,6,"Another string")
```
- Unions: Pony är starkt typat men tillåter att en variabel kan vara en av flera möjliga klasser genom att definiera det som en `Union`.
Även unions kan användas för typalias.
En variabel som kan vara av antingen typ `String` eller `None` kan då deklareras:
```javascript
let unionString:(String|None)="String in this case" //Direkt assignment av unions.
//With type alias
type UnionJack is (String|None)
let jack:UnionJack = None 
```
För att använda en union på ett vettigt sätt måste typen först kontrolleras t.ex. genom ponys stöd för pattern matching, eller så behöver alla möjliga typer dela vissa interface etc.

- Intersections, intersections används i pony för att kräva kombinationer av typer, främst då `Trait` och `Interface`.
```javascript
type IntersectionExample is (Stringable & ExampleTrait & ExampleInterface) //Demands a class that implements all three Types.
```
Samtliga av dessa alias kan också anbvändas tillsammans med andra typalias, för att skapa kombinerade alias:
```
type MetaUnionAlias = (InterSectionExample|None)
type MetaTuple = (InterSectionExample,UnionJack,IntIntString)
type MetaIntersection = (InterSectionExample & Color)
```
Dessa typer gör Ponys typsystem expressivt och ersätter till stor del de behov som nested klasser fyller i Java. 

## FFI - Foreign Function Interface
PonyC har bra stöd för att interagera med C bibliotek, vilket kan bli nödvändigt då många situationer kräver bibliotek som Pony inte har. 
Att importera ett delat C bibliotek är enkelt. Med LD som dynamisk linker kan man hitta namnet via LDconfig och importera biblioteket med:
`use "lib:madeUpRandomNumberLib"`.  

För att anropa en C function behöver man sedan (i enklare fall) bara lägga till ett prefix till funktions-anrop:  
`let randomNumber = @cGenerateRandomNumber[U32]()`  

Detta anropar C functionen getRandomNumber och `[U32]` direkt efter metodnamn specificerar C metodens returtyp. 
I detta fall kan denna mappas direkt på pony en pony U32 typ.

I mer komplexa fall kan man behöva specificera up tuples för enklare C structs:   
`type SimpleStruct = (U32,U32,Pointer[I64)` definierar en struct med två 32 bitars unsigned int:ar och en pointer till en signed 64:a bitars integer som fält.  
I fall där C har definierat structs som refererar andra structs kan dessa definieras i Pony. Det kan rätt snabbt bli rörig kod och återimplementering av stora bitar C, men Ponys FFI funktioner ger oftast ren och enkel interop med C.

En hel del semantic verkar existera i Pony specifikt för interoperabilitet med C, vilket kommer att visa sig i kodprojektet.

### Oändliga mängder sätt att definiera en Integer.

Pony har precis som C oändliga (nåja) möjligheter att definiera en Integer.  
Att explicit deklarara signade eller osignade 8,16,32 eller 64 bitars integers finns det alla möjligheter till med respektive typ:  `U8`,`U16`,`U32`,`U64`,`I8`,`I16`,`I32` eller `I64`. 
C implementerar även `size_t` och `ssize_t` en osignad och signad integer där processor-typen hos den kompilerande datorn avgör hur många bitar integer-typen kan innehålla. Pony stödjer motsvarande med `USize` eller `ISize`, och dessa typer kan ges som argument till C metoder som vill ha 'size_t' eller 'ssize_t' som argument. 

För ännu mer komplext C data kan man definiera motsvarande Struct:ar i PonyC, t.ex. för Struct:ar som innehåller andra Struct. 
C funktioner returnerar ofta resultat genom att ta emot pekare som argument och sedan modifiera dessa på plats i minnet. 
Pony har därför funktioner för att interagera med pekare från C, skicka in retur-pekare och retur-lambdan etc finns också.

### FFI:er beware

Ponys FFI gör att många ABI (Application Binary Interface) kompatibla bibliotek kan användas relativt smärtfritt från PonyC, men då givetvis utan alla garantier om minnessäkerhet och brist på krascher som Pony ger. Att inkorrekt specificera upp en returtyp från ett anrop till C ger en total krash under runtime, vilket är omöjligt att uppnå med ett rent Pony-program.

ABI kompatibla bibliotek kan skapas i ett flertal språk, så det finns möjlighet till språkbryggor till mer än ren C.

Pony känns intuitivt och har medvetet rensat bort mycket komplexitet, sådana saker som privat scopade klasser som i t.ex. Java finns inte. Namnkonflikter mellan klasser och variabler bekämpas med eftertryck av kompilatorn och antalet parenteser av olika slag som behövs är minimalt.

Det är enkelt att vara produktiv utan att skjuta sig själv i foten.
Det återstår dock att se hur annat än testkod går att bygga och vilka problem som uppstår.

Eftersom kodcaset tog upp ungefär lika mycket plats som resten av inlägget har det brutits ut till sin egen post, vilket förhoppningsvis gör det något mer lättläst.
Del 3 bör laddas upp inom en dag eller två.
