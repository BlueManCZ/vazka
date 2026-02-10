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
- vazka-client

E.g. Zákazník potřebuje vytisknout etiketu v momentě, kdy se váha ustálí na nenulové hodnotě. Klient si vytvoří bind na server event `weight_fixed` a v momentě, kdy je event zachycen, tak za zavolá metodu serveru `print_data`, kterými server zahájí tisk etikety.

(Váha může být v alibi režimu a server se dotazuje na stabilizovanou hodnotu, na základě čehož vyvolá event `weight_fixed`. Váha však může být také v kontinuálním výstupu a server pak sleduje, zda po dobu `t` zůstává nenulová váha stejná. Klient však rozdíl z hlediska API nepozná. Jediný rozdíl tam může být v době `t`, kterou by v případě kontinuálního výstupu měl uživatel mít možnost nastavit.)

E.g. Zákazník chce navážit dodávku a po skončení vytisknout dodací list. Na klientu stiskne tlačítko pro vážení nové dodávky. Postupně z katalogu produktů vybírá různé produkty, váží je a po skončení stiskne tlačítko pro vytisknutí dodacího listu. Produkty mohou být buďto nahrané přímo ve váze, nebo lokálně. V obou případech bude katalog existovat na serveru a buď se bude s váhou sychronizovat, nebo ukládat produkty lokálně. Server po zaznamenání `weight_fixed` pošle jako součást eventu zvážený produkt. V případě výběru produktů přímo za váze si server tento produkt načte z váhy, v případě lokálního katalogu klient při výběru váženého produktu na serveru zavolá metodu `set_product`, kterou nastaví vážený produkt a server ho pak bude vracet v eventech. Klient může také načíst produkty uložené na serveru pomocí metody `get_catalog`.

E.g. Zákazník potřebuje zapínat pásovou linku podle váhy v zásobníku materiálu. Má na to program plánování výroby, který potřebuje přes dotaz-odpověď zjišťovat aktuální hmotnost na váze. Jedna možnost je výstup z váhy zapojit přímo do jejich programu. Druhá možnost je váhu připojit k Vážce a nastavit zrcadlení komunikace na další port či posílat komunikaci např. po ethernetu či úplně jiným protokolem, který jejich program vyžaduje. V tomto případě by šlo o aktivování modulu `vazka-router` klientem a nastavením jeho protokolu a portů.
