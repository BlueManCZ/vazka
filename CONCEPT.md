# Vážka

## Description
Vážní SW zajišťující komunikaci s váhou a dalšími periferiemi podle potřeb zákazníka a obsluhu vážení pomocí GUI.

## Concept
Struktura:
- vazka-server
  - vazka-scale - modul zajišťující obousměrnou komunikaci s váhou a poskytující jednotné modulární API pro různé druhy vah
    
    Pro nastavený typ váhy vrací dostupné moduly/metody.

    Aktivní modul pro váhu implementuje specifika komunikace:
    - scale-mettler-ind5xx - modul pro komunikaci s indikátory Mettler Toledo série 5xx
    - scale-...
  - vazka-printer - modul pro tisk etiket, dokumentů nebo tisk do souboru
  - vazka-router - modul pro zrcadlení/přesměrování komunikace s váhou na další porty pomocí různých protokolů
  - vazka-led - modul pro integraci různých LED displejů zobrazujících aktuální váhu
  - vazka-relay - modul pro spínání relé (např. semaforů, otevírání závory, atd).
- vazka-client

E.g. Zákazník potřebuje vytisknout etiketu v momentě, kdy se váha ustálí na nenulové hodnotě. Klient si vytvoří bind na server event `weight_fixed` a v momentě, kdy je event zachycen, tak za zavolá metodu serveru `print_data`, kterými server zahájí tisk etikety.

(Váha může být v alibi režimu a server se dotazuje na stabilizovanou hodnotu, na základě čehož vyvolá event `weight_fixed`. Váha však může být také v kontinuálním výstupu a server pak sleduje, zda po dobu `t` zůstává nenulová váha stejná. Klient však rozdíl z hlediska API nepozná. Jediný rozdíl tam může být v době `t`, kterou by v případě kontinuálního výstupu měl uživatel mít možnost nastavit.)

E.g. Zákazník chce navážit dodávku a po skončení vytisknout dodací list. Na klientu stiskne tlačítko pro vážení nové dodávky. Postupně z katalogu produktů vybírá různé produkty, váží je a po skončení stiskne tlačítko pro vytisknutí dodacího listu. Produkty mohou být buďto nahrané přímo ve váze, nebo lokálně. V obou případech bude katalog existovat na serveru a buď se bude s váhou sychronizovat, nebo ukládat produkty lokálně. Server po zaznamenání `weight_fixed` pošle jako součást eventu zvážený produkt. V případě výběru produktů přímo za váze si server tento produkt načte z váhy, v případě lokálního katalogu klient při výběru váženého produktu na serveru zavolá metodu `set_product`, kterou nastaví vážený produkt a server ho pak bude vracet v eventech. Klient může také načíst produkty uložené na serveru pomocí metody `get_catalog`.

E.g. Zákazník potřebuje zapínat pásovou linku podle váhy v zásobníku materiálu. Má na to program plánování výroby, který potřebuje přes dotaz-odpověď zjišťovat aktuální hmotnost na váze. Jedna možnost je výstup z váhy zapojit přímo do jejich programu. Druhá možnost je váhu připojit k Vážce a nastavit zrcadlení komunikace na další port či posílat komunikaci např. po ethernetu či úplně jiným protokolem, který jejich program vyžaduje. V tomto případě by šlo o aktivování modulu `vazka-router` klientem a nastavením jeho protokolu a portů.

### Note #1 (Ivo)

Napadá mě k tomu, že s tímto konceptem by mohlo být problematické vyřešit např. problémy typu 2 váhy na různých místech + 1 tiskárna společná pro obě.
Z konceptu jsem pochopil, že `vazka-scale`, `vazka-printer`, `vazka-router`, ... budou aktivovatelné součásti/moduly jednoho "monolitického"
programu `vazka-server` na PC/raspberry u jedné konkrétní váhy. To mi přijde zbytečně komplexní z pohledu programu a zároveň omezené z pohledu systému jako celku.
Napadá mě, jestli by nedávalo smysl jít opačnou cestou. Udělat z `vazka-scale`, `vazka-printer` mikroservisy, tzn. nezávislé služby, které by byly dostupné v síti.
Tím pádem bychom mohli udělat to, že `vazka-scale` by byla nainstalovaná u obou vah na Raspberry/Arduinu (stal by se z ní jen takový "firmware" pro váhy)
a pak buď na jednom z těch dvou Raspberry nebo na dalším Raspberry u tiskárny by běžel `vazka-printer` service, který by tiskl,
co uživatel potřebuje. A někde by ještě běžel servis `vazka-core`, který by zajistil orchestraci všeho dohromady, aby všechny servisy
správně komunikovaly. Měl by na starost předávání dat z `vazka-scale` na `vazka-printer`. Tento servis `vazka-core` by měl i server,
na který by se připojovali klienti - PC / mobilní aplikace a zároveň by eventuálně zajišťoval komunikaci s celým informačním systémem,
kdyby jim nějaký běžel.

Výhod v tomto řešení vidím několik:

- Nebude nám "boptnat" jeden Python program pro všechno, když budeme přidávat nové funkce.
- Nebudeme závislí na tom, že všechny funkce musí běžet na jednom zařízení. Můžeme je rozdělit podle toho, kde je potřeba je mít (u váhy, u tiskárny, ...).
- Budeme mít jasně oddělené části systému, což nám usnadní vývoj, testování a údržbu.
- Budeme mít větší flexibilitu v tom, jaké technologie použijeme pro jednotlivé části systému (kdyby některá část měla specifické požadavky, které by nebylo vhodné řešit v Pythonu).
- Budeme mít lepší škálovatelnost, protože jednotlivé části systému budou nezávislé a můžeme je škálovat podle potřeby.
- V extrémním případě můžeme všechny části spustit na jednom zařízení, pokud by to bylo potřeba, ale zároveň máme možnost je rozdělit, pokud by se ukázalo, že je to výhodnější.
- Z pohledu klienta by bylo jedno, jestli se připojuje k jednomu monolitickému programu nebo k několika mikroservisám. `vazka-core` by měl zajistit, že všechny potřebné informace a funkce budou dostupné přes jednotné API, takže pro klienta by to bylo transparentní.

Je zde samozřejmě i několik nevýhod, které by bylo potřeba zvážit:
- Budeme muset řešit komunikaci mezi jednotlivými servisy, což může být složitější než mít všechny funkce v jednom programu.
- Nasazení celého systému bude o něco složitější, protože budeme muset nasazovat a spravovat více služeb.
