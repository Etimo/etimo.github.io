---
layout: post
author: Erik Malm
title: Compile time - PonyLang del 3
image: ponyThree.png
caption: Att bygga något med en Pony
shortexcerpt: Dags att titta en gåvo-pony i tänderna med ett projekt, kommer utvecklaren att sparka bakut?
tag: [featured,compile-time,pony]
categories: [Compiled, Pony,Featured,Compile time]
excerpt_separator: <!--excerpt-->
---
Ponys har synats inledande i [del 1]({% post_url 2018-05-08-compile-time-pony %}) och [del 2]({% post_url 2018-09-08-compile-time-pony-partTwo %}).

I den här delen är det dags att försöka producera funktionalitet med hjälp av Pony, och efter det ge ett sammantaget intryck av att arbeta med denna unghäst. Är det en märr-y upplevelse eller ett dåligt vallack?

# Kod case - Kategorisera medelfärg i bilder.

Projektet i den här bloggen ska göra följande:  
1. Ta en katalog som argument.
1. Lista bilder i katalogen.
1. Läsa information från bilderna.
1. Räkna ut den genomsnittliga färgen i en bild (RGBA, gärna omvandlat till 8:a bitars djup).
1. Skriva ut resultatet för samtliga bilder.

## Del 1.  Ta en katalog som argument och lista bilder

För att programmet ska kunna lista bildfiler i en katalog behöver följande problem lösas med Pony.
- Ett argument som pekar ut den katalog där programmet ska leta efter bilder, pony har stöd för avancerade command-line flaggor, men vi nöjer oss med att det första argumentet som ges till programmet är en path till den bildfolder som ska analyseras.
<!--excerpt-->
- Filer i målkatalogen måste listas och de filer som är bildfiler behöver filtreras.
- Bild-filer bör sedan vidarebefodras till processning och dessa behöver skickas till processning.
För en första implementation:
- Argument kan läsas ur `env`, i en argv array. Ett av dessa kommer i programmet att vara en relativ path.

- Ponys standardbibliotek innehåller klasser för att representera filer, FilePath. Dessa skapas med en referens ponys "root capability" som är en referens till programmets working root och rättigheter att arbeta i denna, i koden kallas detta en AmbientAuth. När man har en filepath kan man göra en Directory operation på denna och sedan lista innehållet i foldern.

- Pony har även en FileCaps klass som definierar vilka filrättigheter som ges på en path. Dessa är betydligt mer fingranulära än t.ex. Pythons `rwx` rättigheter och ett första stop i projektet var att spåra upp vilka rättigheter som behövs.  
Exempel finns i Pony-projektet men det blir snabbt tydligt att det enklaste sättet att hitta informationen om detta är att gå direkt till `FileCaps` källkoden, hälpsamma tutorials lyser med sin frånvaro.

- För att filtrera filändelser körs varje filnamn sedan igenom ett lambda som returnerar sant eller falskt. Ett klassiskt filter/predicate med andra ord. Acceptabla filändelser lagras i en Array.
En första implementation av detta ser ut som nedan i Main actorn:

```javascript
use "collections"
use "inspect" //Pony library for printing objects.
use "./filactors"

actor Main
let _env:Env
let imageEndings:Array[String] val = [".jpg";".png";".bmp";".gif"] //Acceptable image extensions
u
let _filter:{(String):Bool} val = {(file:String):Bool =>  //Define a filter function based on file name
try 
let size = file.isize()?
imageEndings.contains(file.substring(size-4,size)) //Predicate function is not clearly documented, but default is to check for memory equality using is.
else
false
end
}

new create(env:Env) =>
_env=env
try initiateImageRead(env.args(1)?) else initiateImageRead(".") end //Read provided argument folder, or current folder.

be initiateImageRead(directory:String) => 
_env.out.print("reading images in "+ directory)
try 
FileList(directory,_env.root as AmbientAuth,FilterFiles(_filter,_env)) 
else
_env.out.print("Error in execution, bad root?")
end

/**
* Filter files and create a ColorCalculator if file ending.
*/    
actor FilterFiles 
let _env:Env
let _filter:{(String):Bool} val
new create(filter:{(String):Bool} val,env:Env) =>
_env=env
_filter=filter
@MagickWandGenesis[None]() //initiate MagickWand environment.    

be foundFile(file:String) =>
let filtered:Bool = _filter(file)
_env.out.print("Received"+ file+", passed:"+filtered.string())

be errorMessage(errorText:String) => _env.out.print(errorText)

```

- *FileList* är en separat actor som returnerar alla filer ur en katalog. Även detta görs non-blocking, där varja hittad fil skickas vidare som ett separat meddelande.
- *FileList* konstruktorn tar en "Reporter" som argument tillsammans med ett foldernamn och listar filer i katalogen, reportern får sedan ett meddelande per fil. 

```javascript
use "collections"
use "inspect"
use "lib:MagickWand-6.Q16"
use "./filactors"
use "./image"
actor Main
    let _env:Env
    let imageEndings:Array[String] val = [".jpg";".png";".bmp";".gif"]
    let _filter:{(String):Bool} val = {(file:String):Bool => 
            let size = file.size().isize() 
            imageEndings.contains(file.substring(size-4,size).lower(),{(k,l):Bool => k == l})
}
            
    new create(env:Env) =>
        _env=env
        try initiateImageRead(env.args(1)?)
             else initiateImageRead(".") end //Read provided argument folder, or current folder.

    be initiateImageRead(directory:String) => 

        _env.out.print("reading images in "+ directory)
        try 
         FileList(directory,_env.root as AmbientAuth,FilterFiles(_filter,_env)) 
        else
         _env.out.print("Error in execution, bad root?")
        end

    fun _final() => 
        @MagickWandTerminus[None]() //terminate MagickWand environment.
    be print(message:String) =>
        _env.out.print(message)
/**
* Filter files and create a ColorCalculator if file ending.
*/    
actor FilterFiles 
       let _env:Env
       let _filter:{(String):Bool} val
        
       new create(filter:{(String):Bool} val,env:Env) =>
        _env=env
        _filter=filter
        @MagickWandGenesis[None]() //initiate MagickWand environment.    
       
        be foundFile(file:String) =>
           let filtered:Bool = _filter(file)
           _env.out.print("Received "+ file+", passed: "+filtered.string())
           if filtered is true then
            ReadImage(file,
             HandleImage({(image:Image) =>  
                _env.out.print("I got some image!"+image.string().substring(0,300))
             },
               {(errorMessage:String) => _env.out.print("Error from image read: "+errorMessage)}  
             ))
            end

        be errorMessage(errorText:String) => _env.out.print(errorText)

```

# Del 2. Läsa bilder och FFI
Programmet behöver i nästa steg kunna läsa in pixel-information från de bildfiler som passerar matchningen, en inte helt trivial sak att implementera från grunden, och tyvärr saknar Pony klasser för bildhantering i standard-biblioteket. En lösning på behöver därmed bli något mer omständig än en Java, Kotlin eller Go implementation, dock är det ett tillfälle att se vad som går att åstadkomma med Ponys FFI stöd.

### ImageMagick och MagickWand
ImageMagick är en fantastisk mjukvara för bildhantering, alla som någon gång öppnat photoshop för att skala om upplösningen på en bild bör uppskatta det enkla nöjet i ett `convert largeImage.jpg --resize 500x500 smallImage.png`.  
ImageMagick erbjuder även flera C bibliotek för att hantera samma funktionalitet programmatiskt, inklusive att hämta ut en binär blob i vilket bildformat som helst (säkerligen med vissa undantag).
Av de C bibliotek som är kopplade til ImageMagick är *MagickWand* det bibliotek som verkar ge bäst balans mellan användarvänlighet och funktionalitet.  
MagickWand finns i Ubuntus repositories och kan installeras med:

`sudo apt install libmagickwand-dev`
För att få fram namnet på den specifika version av biblioteket vi nu har tillgängligt:
```bash
ldconfig -p|grep -i magickwand
libMagickWand-6.Q16.so.3 (libc6,x86-64) => /usr/lib/x86_64-linux-gnu/libMagickWand-6.Q16.so.3
libMagickWand-6.Q16.so (libc6,x86-64) => /usr/lib/x86_64-linux-gnu/libMagickWand-6.Q16.so
```
Även dokumentation för denna specifica version av MagickWand finns i Ubuntus repo och en nedladdning av denna är starkt reckommenderad, at hitta versions-specifik dokumentation på nätet är inte trivialt.
För den som är van att jobba med väldokumenterade Java bibliotek blir det snabbt tydligt att dokumentation för MagickWand är betydligt luddigare.

Att gräva fram de metoder som behöver användas för att kunna göra en första implementation tar tid, källkodsläsning och forum-postande. 
Bara att gräva upp inblandade Struct:ar är en smått ambitiös procedur som ofta slutar med läsning av en eller annan lokal header-fil.

#FFI variant 1 - anrop till externa C metoder.
### convert image.jpg image.txt
Eftersom projektet enbart vill ha färgdata för de olika pixlarna i en bild och inte en riktig konvertering vore det en vinst att skippa binära format över huvudtaget. ImageMagick har redan ett textformat för detta där varje pixel decodas till textinformation.
Ett convert på etimos logga ger t.ex. 
```bash
convert etimo.png pixel.txt
head -n 5 pixel.txt
Output:
# ImageMagick pixel enumeration: 200,81,65535,srgba
0,0: (0,0,0,0)  #00000000  none
1,0: (0,0,0,0)  #00000000  none
2,0: (0,0,0,0)  #00000000  none
3,0: (0,0,0,0)  #00000000  none

```
I detta data har vi pixel-koordinater följt av färgdata i olika format för aktuell pixel. 
Detta är tillräckligt med information för projektet så en första bildinläsningsimplementation kan försöka återskapa denna funktionalitet.  

### MagickGetImageBlob - Anrop till C bibliotek från Pony
Att reproducera konvertering till textformat i MagickWand med FFI är en något hårdare nöt att bita i. Efter en hel del grävande i både Pony och MagickWand dokumentation kunde en fungerande implementation skapas. Koden nedan använder MagickWands funktioner för att göra konverteringen i minne.
Detta görs i en Actor typ "ReadImage", denna actor läser in en bild m.h.a. MagickWand och skickar sedan som läser en bild och sedan skickar ett meddelande med bilden som text till sin callback.
Förutom `@`tecken och specificerade returtyper är det inte mycket som skiljer ett anrop till Pony-kod från ett FFI-anrop.

```javascript
use "collections"
use "lib:MagickWand-6.Q16"

primitive _Wand //Will use of primitives crash this in a multiple actor situation?
primitive _Pixelwand
primitive _Read

    
actor ReadImage 
    new create(file:String val,handler:ImageHandler tag) =>
        
        @MagickWandGenesis[None]()
        var wand = @NewMagickWand[Pointer[_Wand]]()
        if wand.is_null() then
            return
        end
        let readOk = @MagickReadImage[Bool](wand,file.cstring())
        if(readOk == false) then
            handler.handleError("Failed to read image "+file+" "+readOk.string()) 
        return
end
        let formatOk = @MagickSetImageFormat[Bool](wand,"txt".cstring()) 
        if((readOk != true) or (formatOk != true)) then
            handler.handleError("Failed to format image "+file) 
        else
    

            
        //Retrieve byte buffer in memory and create string
        var len = USize(0) //Return pointer for C method to set size in. Equivalent to size_t in C
        let bufferPointer:Pointer[U8] ref = @MagickGetImageBlob[Pointer[U8] ref](wand,addressof len)
        let converted:String ref=  String.from_cpointer(bufferPointer,len)

        handler.handleImage(file,converted.clone()) //If not cloned, cleaning up MagickWand would kill string.
        
        //@MagickWriteImage[Bool](wand,"txt:-".cstring())
         
    if wand.is_null() is false then
         wand = @DestroyMagickWand[Pointer[_Wand]](wand)
    end
end
```
Eureka!

Efter detta är det i teorin en smal sak att extrahera informationen kring pixlarna.

Ponys FFI stöd är i enklare fall verkligen smärtfritt, även om vissa av anropen i denna gäller ren minneshantering, t.ex. `DestroyMagickWand` anropet, som enbart syftar till att rensa minnet som bildinläsningen använder.
C är m.a.o. alltid C även om det anropas från Pony.

Största hindret för att skriva denna klass visade sig vara dokumentartionen för C biblioteket, den som är van vid att läsa Javadoc i Apache foundation projekt bör stålsätta sig och vara beredd på att läsa källfiler direkt. 

### FFI inläsning testkörning
En testkörning på en katalog av bilder från [](del 1 av pony bloggen): 
```bash
reading images in ../testImages
/home/erik/code/testImages true
Received /home/erik/code/testImages/animal-grass-horse-85681.jpg, passed: true
Received /home/erik/code/testImages/Dl6ygUvU4AAn1gD.jpg:large.jpeg, passed: false
Received /home/erik/code/testImages/dock-lake-sun-37981.jpg, passed: true
I got some image!# ImageMagick pixel enumeration: 2163,1368,65535,srgb
0,0: (16705,15163,11565)  #413B2D  srgb(65,59,45)
1,0: (16448,14906,11308)  #403A2C  srgb(64,58,44)
2,0: (16191,14649,11565)  #3F392D  srgb(63,57,45)
3,0: (16448,14906,11822)  #403A2E  srgb(64,58,46)
4,0: (17219,15420,12850)  #433C32  srgb(67,60,
Received /home/erik/code/testImages/animal-domestic-animal-farm-58875.jpg, passed: true
I got some image!# ImageMagick pixel enumeration: 3264,2017,65535,srgb
0,0: (29041,22359,23130)  #71575A  srgb(113,87,90)
1,0: (29555,22873,23644)  #73595C  srgb(115,89,92)
2,0: (29812,23130,23901)  #745A5D  srgb(116,90,93)
3,0: (29041,22359,23130)  #71575A  srgb(113,87,90)
4,0: (28527,21074,22102)  #6F5256  srgb(11
Received /home/erik/code/testImages/animals-herd-horses-37987.jpg, passed: true
Received /home/erik/code/testImages/animal-farm-foal-37983.jpg, passed: true
Received /home/erik/code/testImages/erik.png, passed: true
Received /home/erik/code/testImages/animal-equine-head-209045.jpg, passed: true
I got some image!# ImageMagick pixel enumeration: 4368,2912,65535,srgb
0,0: (39321,42662,46774)  #99A6B6  srgb(153,166,182)
1,0: (39321,42662,46774)  #99A6B6  srgb(153,166,182)
2,0: (39321,42662,46774)  #99A6B6  srgb(153,166,182)
3,0: (39578,42919,47031)  #9AA7B7  srgb(154,167,183)
4,0: (39321,43433,47288)  #99A9B8 
<Some output removed here>
I got some image!# ImageMagick pixel enumeration: 5485,3697,65535,srgb
0,0: (24929,20817,16962)  #615142  srgb(97,81,66)
1,0: (25700,21588,17733)  #645445  srgb(100,84,69)
2,0: (26471,22102,17990)  #675646  srgb(103,86,70)
3,0: (25700,21331,17219)  #645343  srgb(100,83,67)
4,0: (25700,20817,16962)  #645142  srgb(100
./pony-image ../testImages  2442,68s user 4,14s system 176% cpu 23:06,79 total

```
Bilder läses in framgångsrikt och de första pixlarna i varje bild skrivs ut av callback till main-actorn.
Låt oss titta på den slutgiltiga tiden för att *läsa in* ca. 10 bilder med denna metod.
`./pony-image ../testImages  2442,68s user 4,14s system 176% cpu 23:06,79 total`

2000 sekunder för fyra bilder, detta är tydligt helt olämpligt för seriös bildprocessning. Inläsningen via FFI är helt tydligt flaskhalsen. Inte ens i en inlärningsuppgift går det att ignorera så dålig performance en mer effektiv implementation är nödvändig.

## PixelWandIterator - ett bättre MagickWand API 

För undvika att konvertera bilden till text och  istället direkt-accessa pixel-information.  
MagickWand erbjuder också API:er för att direkt accessa pixelinformation via ett "PixelIteraror" interface. 
Att direkt integrera detta med Pony via FFI visade sig dock bli en soppa av nestade `struct` definitioner.
Kring 10 structar behövs definieras i pony för att kunna får okej interoperabilitet med PixelWandIterator-flödet. Efter ett antal seg-faults p.g.a inkompatibla struct-definitioner (dokumentatione här är också relativt frånvarande, bara att gräva fram rätt header-filer i bibliotekets kod) bytte projektet arbetssätt.

### Anrop till eget minimalt C bibliotek från Pony

En enklare variant är faktiskt att (nåja) skriva sitt eget minimala C bibliotek och sedan anropa detta enklare bibliotek.
Genom att bygga sit eget C bibliotek där vi definierar vilka structar som Pony- ska integrera mot försvinner en hel del av komplexiteten på Pony-sidan.
Sagt och gjort, efter mer dokumentationsläsning och djupdykningar i MagickWand forum kan följande fungerande C klass byggas.

```c
#include <wand/magick-wand.h>
#include <stdio.h>
#include <string.h>
#include <limits.h>

typedef struct Pixel {
  double red;
  double blue;
  double green;
  double alpha;
  double depth;
} Pixel;

Pixel** pixelWandGet(char*,size_t*,size_t*);
Pixel** pixelWandGet(char* file,size_t* outwidth,size_t* outheight)
{
  MagickWand *mwand = NULL;
  PixelIterator *iterator = NULL;

  size_t x = 0;
  size_t y = 0;
  size_t xpixel = 0;
  size_t ypixel = 0;

 // MagickWandGenesis(); //Genesis and Terminus code moved to parent pony program
  mwand=NewMagickWand();
  MagickBooleanType bool = MagickReadImage(mwand,file);
  MagickSetImageType (mwand, TrueColorType);
  MagickSetImageColorspace (mwand, sRGBColorspace);
  if(!bool){
    printf("Could not read provided file!");
    return NULL;
  }
    

  PixelWand **pixelWands;
  iterator=NewPixelIterator(mwand);
  unsigned long width;
  MagickPixelPacket pixel;

  xpixel=MagickGetImageWidth(mwand);
  ypixel=MagickGetImageHeight(mwand);
  *outwidth =xpixel;
  *outheight = ypixel;
  Pixel **pixels=(Pixel **)malloc(xpixel*sizeof(Pixel *));
  for(x=0;x<xpixel;x++){
    pixels[x]=(Pixel *)malloc(ypixel*sizeof(Pixel));
  } 
  for (y=0; y < ypixel-1; y++)
    {
      pixelWands=PixelGetNextIteratorRow(iterator,&width);

      for (x=0; x < xpixel; x++)
        {
          PixelGetMagickColor(pixelWands[x],&pixel);
          pixels[x][y].red = pixel.red; 
          pixels[x][y].blue = pixel.blue;
          pixels[x][y].green = pixel.green;
          pixels[x][y].alpha = pixel.opacity;
          pixels[x][y].depth = pixel.depth;
          //printf("Depth %u\n",pixel.depth);
          //printf("C-check: %zd %lu : %zd %zd => %f %f %f %f %u %s\n",xpixel,width,x,y,pixels[x][y].red,pixels[x][y].green,pixels[x][y].blue,pixels[x][y].alpha,pixel.colorspace,PixelGetColorAsString(pixelWands[x]));
        }
    }
  iterator=DestroyPixelIterator(iterator);
  //	MagickWandTerminus();
  //printf("Size %zd\n",width);
  printf("Processed %s!\n %f\n",file,pixels[0][0].red);
  return pixels;
}
```

Detta kan sedan byggas till en biblioteksfil (notera att header fils paths etc i detta kommando är Ubuntu 18.04 specifika, mer seriösa byggverktyg bör användas i riktiga projekt): 
```bash
gcc -c ./getPixel.c -o pixelwand.o -I /usr/include/ImageMagick-6/ -I /usr/include/x86_64-linux-gnu/ImageMagick-6/ #Kompilerar koden till en objekt-fil
ar rcs libpixelwand.a pixelwand.o #Bundla filen till ett bibliotek. Alla bibliotek måste namnges enl. lib-namnetpåbilioteket.
```
För att använda detta bibliotek måste en extra path också anges i pony-filen, där use `"path:./c"` instruerar pony kompilatorn i vart den ska leta, relativt till källkodsfilen.
### Image-klasser och C-Arrayer 
Pony kan sedan integreras med vårt eget C-bibliotek inuti en actor som:
- Tar emot en filpath.
- Anropar C-biblioteket med denna, samt pekare till två stycken pony-usize integers. Returtypen är definierad som en typle av 64-bitars floats för att matcha C bibliotekets doubles.
- C-biblioteket returnerar en 2D-array, denna mappas till om till en pony-array som sedan skickas tillbaka till ImageHandlern.

 ```javascript
use "collections"
use "lib:MagickWand-6.Q16"
use "path:./c"
use "lib:pixelwand"
use "inspect"
type TuplePixel is (F64,F64,F64,F64,F64)
 
      
actor ReadPixelWandLib
  new create(file:String val,handler:ImageHandler tag) =>
    //Run C library
    let actualPixels = recover iso Array[Array[Pixel val] val] end

    var width:USize = 0
    var height:USize = 0

    var cPointer : Pointer[Pointer[TuplePixel]] = @pixelWandGet[Pointer[Pointer[TuplePixel]]](
        file.cstring(),addressof width,addressof height)
    handler.handleError("Resolution "+width.string()+" : "+height.string())

    //Allocate 2D pixel array
    let arrayOne = Array[Pointer[TuplePixel]].from_cpointer(cPointer,width,width)
    for pointer in arrayOne.values() do
      let array =pixelFromTuple(Array[TuplePixel].from_cpointer(pointer,height,height)) //Copy data from FFI call
      actualPixels.push(array)
    end

    //Proceed to build image
    let valImageFinal  = recover val Image(file, (recover val consume actualPixels end),height, width) end //Create an image and recover val capability
    handler.handleError("Done with image: "+file) 
    handler.handleImage(valImageFinal)

  //Copies values from FFI tuple into pony class. Allows return value to have othe capability than var.
  fun pixelFromTuple(tuples:Array[TuplePixel val]):Array[Pixel val] val =>
    let pixelArray = recover iso Array[Pixel val] end
    for tp in tuples.values() do
      let pixel = recover val Pixel(tp) end
      pixelArray.push(pixel)
    end
    recover val consume pixelArray end

```

Efter anrop till vårt nya C bibliotek återstår ett FFI-problem. Att mappa uppe en 2D array från C är bökigare än man kanske skulle önska. 
- Returtypen till Pony är en Pekare, till flera Pekare till flera Pixlar. Denna C struktur behöver konverteras till något mer pony-nativt.
- Pony har en konstruktur i sin Array klass för pekare, men ingen möjlighet att göra detta iterativt.  Alltså behövs en loop för att göra om varje pointer i array:en.
- Det visar sig också att array:er som skapas på detta sätt alltid är `ref` capability och alltså inte kan skickas till en annan actor. Att mappa om till en `iso` eller `val` visar sig vara icke trivialt. 
Efter ett antal mislyckade försök med `recover`, clone- metoder och en support-förfrågan till [Ponys Mail-grupp](https://pony.groups.io/g/user/topic/is_it_possible_to_make_a/23784530?p=,,,20,0,0,0::recentpostdate%2Fsticky,,,20,2,0,23784530) visar det sig att det inte går att mappa om en ref-2d array till en val och det går inte att konstruera en `iso`-array direkt från en FFI-pekare.

Lösningen i det här fallet färgvärdena från varje pixel från C biblioteket kopieras till en pony Pixel-klass, och vidare in i en pony-array istället för att fortsätta att använda arrayen från FFI biblioteket. Denna pixel-array kopieras sen in i en Image klass som skickas tillbaka som meddelande till huvudactorn.
Snygg integration med FFI är helt klart möjlig från Pony, men hantering av collections via FFI är icke-trivialt. 

Kod för Image och pixelklasserna i Pony-nedan:

```javascript
use "collections"

class Image
  let imageName:String
  let pixels:Array[Array[Pixel val] val] val
  let height:USize
  let width:USize 
  let average:Pixel
  

  new create(file:String,inPixels:Array[Array[Pixel val ]val ] val,inHeight:USize,inWidth:USize) =>
    imageName = file
    pixels = inPixels
    height= inHeight
    width = inWidth
    average=Pixel.fromRaw(-99,-99,-99,-99,-99)

  new calculateAverage(img:Image,averagePixel:Pixel) =>
    imageName = img.imageName
    width = img.width
    height = img.height
    pixels = img.pixels
    average= averagePixel

  fun string():String => imageName.clone()
  /*
  Calculate average color of the image and return the value as a pixel.
  */


class Pixel 

  let _red:F64 val
  let _green:F64 val
  let _blue:F64 val
  let _alpha:F64 val
  let _depth:F64 val
  
  fun string():String =>
      ("RGBA ("+_red.string()+", "+_green.string()+", "+_blue.string()+", "+_alpha.string())+") "+_depth.string()+" bit depth"
  fun string8bit():String =>
    let factor = _get8BitFactor()
    "RGBA ("+(_red/factor).string()+", "+
             (_green/factor).string()+", "+(_blue/factor).string()+", "+(_alpha/factor).string()+")"

  fun _get8BitFactor():F64 =>
   (F64(2).pow(_depth)-1)/255

  new fromRaw(red:F64,green:F64,blue:F64,alpha:F64,depth:F64) =>
    _red=red
    _blue=blue
    _green=green
    _alpha=alpha
    _depth=depth

  new create(inPixel:TuplePixel) =>
    _red=inPixel._1
    _blue=inPixel._2
    _green=inPixel._3
    _alpha=inPixel._4
    _depth=inPixel._5

  fun getAlpha():F64 val=> 
    _alpha
  fun getRed():F64 val=> 
    _red
  fun getBlue():F64 val=> 
    _blue
  fun getGreen():F64 val=> 
    _green
  fun getDepth():F64 val=> 
    _depth
    
```

### Beräkna medel-färg:
Som ett sista steg behöver vi beräkna medelfärgen.
Detta görs när `Main`-actorn får tillbaks en bild m.h.a. en `CalculateColor` actor.
En sådan actor skapas per bild. När beräkningen av medelfärg är klar skrivs detta till `stdout`.
Eftersom bilder ibland har större färgdjup än 8-bitar (vilket ger ett maxvärde per färgkanal på 255) så sparas även bildens färgdjup. Den informationen används sen till att konvertera medlet till ett 8-bitarsvärde.

Slutgiltig kod för den här biten av appen blir:
```javascript
use "collections"
use "inspect"
use "lib:MagickWand-6.Q16"
use "./filactors"
use "./image"

actor Main
//This actor reads a folder of images and calculates their average color.
  let _env:Env
  let imageEndings:Array[String] val = [".jpg";".png";".bmp";".gif"] //Allowed file endings.

  let _filter:{(String):Bool} val = {(file:String):Bool => //Filter Lambda for filtering filenames by extension.
  let size = file.size().isize() 
  imageEndings.contains(file.substring(size-4,size).lower(),
  {(k,l):Bool => k == l}) //Predicate function is necessary or check will be for actual object, not equivalency.
  }
  
  new create(env:Env) =>
    _env=env
    try initiateImageRead(env.args(1)?)
      else initiateImageRead(".") end //Read provided argument folder, or current folder.

  be initiateImageRead(directory:String) => 

    _env.out.print("reading images in "+ directory)
    try 
      FileList(directory,_env.root as AmbientAuth,FilterFiles(_filter,_env)) 
    else
      _env.out.print("Error in execution, bad root?")
    end

  fun _final() =>
    @MagickWandTerminus[None]() //terminate MagickWand environment, FFI. 
  be print(message:String) =>
    _env.out.print(message)

    /**
    * Filter files and create a ColorCalculator if file ending.
    */    
actor FilterFiles 
  let _env:Env
  let _filter:{(String):Bool} val
  
  new create(filter:{(String):Bool} val,env:Env) =>
    _env=env
    _filter=filter
    @MagickWandGenesis[None]() //initiate MagickWand environment.    
    
  be foundFile(file:String) =>
    let filtered:Bool = _filter(file)
    _env.out.print("Received "+ file+", passed: "+filtered.string())
    if filtered is true then
      ReadPixelWandLib(file,
      HandleImage({(image:Image val) =>  
      _env.out.print("I got some image!"+image.string().substring(0,300))
      let calculator = CalculateColor(_env)
      calculator.calculateAverage(image)
      },
      {(errorMessage:String) => _env.out.print("Error from image read: "+errorMessage)}  
      ))
    end

  be errorMessage(errorText:String) => _env.out.print(errorText)


actor CalculateColor 
  let _env:Env
  
  new create(env:Env) =>
    _env=env
    
  be calculateAverage(image:Image val) =>
    let average:Pixel = calculateAveragePixel(image,USize(1))
    _env.out.print(image.string()+": Average color "+average.string()+
                                  " Average 8 bit: "+average.string8bit())

  fun calculateAveragePixel(image:Image val, nthPixel:USize):Pixel =>
    var tmpPixel = Pixel.fromRaw(0,0,0,0,0)
    var pixelCount:F64 = 0
    let height = image.height
    let width = image.width
    try
      for x in Range(0,width,nthPixel) do
        for y in Range(0,height) do
          tmpPixel = iteratePixel(tmpPixel,image.pixels(x)?(y)?)
          pixelCount = pixelCount+1
        end
      end
    end

    Pixel.fromRaw(
    tmpPixel.getRed()/pixelCount,
    tmpPixel.getGreen()/pixelCount,
    tmpPixel.getBlue()/pixelCount,
    tmpPixel.getAlpha()/pixelCount,
    tmpPixel.getDepth()
    )
    

  fun iteratePixel(pixelOne:Pixel,pixelTwo:Pixel val):Pixel =>
    Pixel.fromRaw(pixelOne.getRed()+pixelTwo.getRed(),
    pixelOne.getGreen()+pixelTwo.getGreen(),
    pixelOne.getBlue()+pixelTwo.getBlue(),
    pixelOne.getAlpha()+pixelTwo.getAlpha(),
    if pixelTwo.getDepth() > pixelOne.getDepth() then pixelTwo.getDepth() else pixelOne.getDepth() end //Some pixels are 0 depth, could be optimized
    )
```

## Testkörning av slutprodukt och slutsatser.
En körning av samma bildkatalog som tidigare ger en klart bättre performance.
Kommandot nedan inte bara läser in bildfiler som tidigare utan beräknar dessutom medelfärgen på ca 7,4s (för time kommandot behöver user time divideras med cpu-usage för att ge den faktiska tiden).
7,4s är klart snabbare än 2000s och optimeringen av FFI-beteendet får nog anses ha gett önskat resultat.

```bash
time ./pony-image ../testImages
reading images in ../testImages
/home/erik/code/testImages true
Received /home/erik/code/testImages/animal-grass-horse-85681.jpg, passed: true
Processed /home/erik/code/testImages/animal-grass-horse-85681.jpg!
Received /home/erik/code/testImages/Dl6ygUvU4AAn1gD.jpg:large.jpeg, passed: false
Received /home/erik/code/testImages/dock-lake-sun-37981.jpg, passed: true
I got some image!/home/erik/code/testImages/animal-grass-horse-85681.jpg
/home/erik/code/testImages/animal-grass-horse-85681.jpg: Average color RGBA (32515.6, 29086.6, 24633.7, 0) 16 bit depth Average 8 bit: RGBA (126.52, 113.177, 95.851, 0)
Received /home/erik/code/testImages/animal-domestic-animal-farm-58875.jpg, passed: true
Processed /home/erik/code/testImages/dock-lake-sun-37981.jpg!
Processed /home/erik/code/testImages/animals-herd-horses-37987.jpg!
Processed /home/erik/code/testImages/animal-domestic-animal-farm-58875.jpg!
I got some image!/home/erik/code/testImages/dock-lake-sun-37981.jpg
/home/erik/code/testImages/dock-lake-sun-37981.jpg: Average color RGBA (39354.6, 22213.3, 15881.7, 0) 16 bit depth Average 8 bit: RGBA (153.131, 86.433, 61.7965, 0)
Received /home/erik/code/testImages/animals-herd-horses-37987.jpg, passed: true
Received /home/erik/code/testImages/animal-farm-foal-37983.jpg, passed: true
Received /home/erik/code/testImages/erik.png, passed: true
Received /home/erik/code/testImages/animal-equine-head-209045.jpg, passed: true
/home/erik/code/testImages/animal-domestic-animal-farm-58875.jpg: Average color RGBA (32699.2, 31803, 27630.9, 0) 16 bit depth Average 8 bit: RGBA (127.234, 123.747, 107.513, 0)
Processed /home/erik/code/testImages/animal-farm-foal-37983.jpg!
Processed /home/erik/code/testImages/erik.png!
/home/erik/code/testImages/erik.png: Average color RGBA (35153.5, 32702.6, -nan, -nan) 16 bit depth Average 8 bit: RGBA (136.784, 127.247, -nan, -nan)
I got some image!/home/erik/code/testImages/animals-herd-horses-37987.jpg
/home/erik/code/testImages/animals-herd-horses-37987.jpg: Average color RGBA (23466.6, 18981.8, 14932.9, 0) 16 bit depth Average 8 bit: RGBA (91.3098, 73.8591, 58.1047, 0)
I got some image!/home/erik/code/testImages/animal-farm-foal-37983.jpg
/home/erik/code/testImages/animal-farm-foal-37983.jpg: Average color RGBA (26369.9, 20095.2, 13113.3, 0) 16 bit depth Average 8 bit: RGBA (102.606, 78.1913, 51.0244, 0)
Processed /home/erik/code/testImages/animal-equine-head-209045.jpg!
I got some image!/home/erik/code/testImages/animal-equine-head-209045.jpg
/home/erik/code/testImages/animal-equine-head-209045.jpg: Average color RGBA (26280.8, 25567.3, 25282.9, 0) 16 bit depth Average 8 bit: RGBA (102.26, 99.4836, 98.3772, 0)
./pony-image ../testImages  12,23s user 2,35s system 165% cpu 8,794 total

```
Ponys flertrådning kör per default lika många trådar som fysiska kärnor på processorn, i det här fallet en tvåkärnig i7:a. 
Antalet trådar programmet har tillgängligt kan sättas m.h.a av att anropa den kompilerade binären med `--ponythreads=1`.
En körning av programmet med `time ./pony-image ../testImages --ponythreads=1` ger totaltid `7,66s` och vår speedup är därmed 0,36s med en extra kärna. Inte överväldigande direkt, även om kataloger med fler bilder visar en större speedup. 

Pony är ett intressant språk, det är kompilerat, performant och trådsäkert. Ponys främsta nackdelar visar sig vara bristen på tredje-parts bibliotek och mängden kringliggande resureser.

Att behöva skriva stora delar av sin grundläggande funktionalitet själv begränsar den mängd nytta en ensam utvecklare kan få ur Pony rejält. Likaså är stödet för något annat än 64-bitars x86 processorer begränsat. Den som vill bygga 32-bitars Pony kan räkna med en betydligt mindre stabil upplevelse och behovet att kompilera kompilatorn.

Den som vill jobba på ARM... bör nog tänka om.

Likaså kan man snabbt glömma allt som har med en mer avancerad IDE att göra, highlighting plugins finns för ett flertal editorer men det verkar långt kvar tills den första Pony-IDEn.  
Pony självt är dock ett rent nöje att arbeta med, även om objekt-capabilities ibland gör det som känns simpelt rejält komplicerat är det ett utbyte, komplexiteten hålls i utvecklingsstadiet.När Pony-kod kompilerar har man en relativt stor säkerhet att den gör vad den ska.  

Pony är kort sagt värt att hålla ett öga på och bidra till där man kan, om tio år vore det en fantastisk utveckling om mer mjukvara utvecklades enligt ponys paradigmer och något projekt per år kommer jag säkerligen att utveckla i Pony fram tills dess.
Pony, I'm a föl for you.

