# Příprava na zkoušku

## Užitečné odkazy
- [úvod do Intel Assembly](https://www.codeproject.com/Articles/1273844/The-Intel-Assembly-Manual-3)

## Základní informace

### Real mode
- režim procesoru, kdy má přístup pouze k 1MB paměti a kód může číst a zapisovat na libovolné místo v paměti
    - adresace pomocí 2O-bitů
    - přímý přístup do paměti (žádné stránkování či segmentace)

### Protected mode
- používaný k izolaci jádra od uživatelských procesů
    - pro adresaci paměti se používá **virtuálních adres**
- aby programy mohly běžet v protected-kode (32-bitová adresa, segmentace, stránkování, izolace procesů), je třeba vytvořit **GDT** a **LDT**
    - 32-bitová adresa umožňuje adresovat 4GB paměti
- **režim procesoru uložen** v registru `CR0`

### Unreal mode
- nejedná se o garantovanou vlastnost
    - spíše bug
- program přepne procesor do protected režimu, vytvoří/načte se deskriptor z GDT/LDT, segmentům se nastaví maximální velikost (limity) a procesor se přepne zpět do real režimu
- po návratu do režimu real zůstanou změněné limity zachovány, a tak je procesor prakticky v režimu **unreal**- tímto způsobem bude procesor v real režimu s možnosti používat virtuální adresy a adresovat tak 4GB opearční paměti 

### Segment Descriptor
- segment deskriptor se skládá z:
    - základní (base) adresy (lineární adresy)
    - velikosti (limit)
    - typ (kód, data, atd...)
    - přístupová práva
        - `00` - většinou jádro
        - `01` - může být jádro, když na `00` poběží hypervizor virtualizace
        - `10` - může být ovladač, který nesmí do jádra 
        - `11` - většinou uživatelský proces
    - další vlajky
- pro izolaci pocesů se vytváří dvě tabulky deskriptorů segmentů
    - globální používaná jádrem
        - uložena v registru `gdtr`
    - lokální používaná konkrétním jedním procesem
        - uložena v registru `ldtr`

### Vyjímky
- jedná se o přerušení generované procesorem, když dojde k:
    - **trap** - např. breakpoint
    - **fault** - opravitelná chyba, program může pokračovat (např. **Page Fault**)
    - **abort** - neopravitelná chyba
        - např. **Double Fault**, kdy dojde k Page Fault, ale handler je ve stránce, která není v RAM

#### Exception handler
- v případě, že procesor nedokáže zavolat obsluhu vyjímky, dojde k Double Fault
    - pokud ani pro ni nebude obsluha, dojde k **Triple Fault**
    - to je známkoou toho, že je něco špatně v obsluze první vyjímky
        - vyjímku nelze zamaskovat
        - buď budeme psát perfektní, bezchybný kód, anebo alespoň psát spolehlivé obsluhy vyjímek
- používá se k tomu **Thread Control Block**

### Kontext vlákna
- při změně vlákna se vždy uloží stav aktuálního vlákna a obnoví se dříve uložený stav nově vybraného vlákna
- kontext vlákna je dán jeho proměnnými:
    - některé proměnné jsou v registrech
    - jiné proměnné jsou např. v zásobníku, kam ukazuje `SS:RSP`
    - další proměnné jsou v paměti, kam mohou ukazovat další registry
    - ukazatelem na instrukci, která se má vykonat je `CS:RIP`
    - stavovou proměnnou OS, která říká, zda proces běží, je pozastaven, atd.

### Thread Control Block
- jádro spravuje vlákno podle jeho TCB
    - pod Windows/Linux přístupný přes `fs`/`gs` registry na x86, x86-64
- TCB obsahuje:
    - uložené hodnoty registrů procesoru
    - prioritu
    - stavovou proměnnou, čas dosavadního běhu vlákna
    - ukazatel na Process Control Block
    - a další specifické informace, jako jsou např. seznam obsluh vyjímek, skok na stránku s funkcí `syscall`, atd...

### Stav vlákna
- ve zjednodušenném pohledu je vlákno:
    - **Runnable** - může být naplánováno a spuštěno
    - **Running** - vlákno běží
    - **Blokované** - buď se usaplo, nebo třeba čeká na podmínkové proměnné, nebo na dokončení I/O operace
    - **Ukončené**
- ve skutečnosti je to složitější, protože OS potřebuje vědět, proč vlákno čeká
    - určitě je rozdíl např. mezi čekáním kvůli Page Fault a kritickou sekcí

### Plánování vláken
- **OS neplánuje procesy, ale vlákna**
    - proces je pouze kontajner vláken
    - Linux nedělá rozdíl mezi procesem a vláknem - všechno je pro něj **runnable task**

### Vyhledání dostupných procesorů
- po zapnutí počítače a inicializaci BIOSu je aktivní pouze jeden procesor (**BSP**), resp. jedno jeho jádro, a zavaděč OS musí zjistit, zda systém obsahuje i další procesorová jádra a aktivovat je
    - pokud se mu to nepodaří, aktivuje se uniprocesorové jádro
- hledá se **MP Floating Pointer Structure**, která začíná signaturou `_MP_` a popisuje dostupné procesory
    - hledá se v 1. kB **Extended BIOS Data Area**
    - v posledním kB základní paměti
    - v adresním rozsahu ROM-BIOSu
- jedna z položek MP Floating Structure je **MP Config Pointer** ukazující na seznam dostupných procesorů:
    - **BSP** - (_Bootstrap Processor_)
    - **AP** - (_Auxiliary Processor aka Application Processor_)
- přesný postup při použití **SMP Boostrap x86**:
    1. Po zapnutí PC jsou všechny CPU v real-mode
    2. BIOS vybere **BSP** a ostatní procesory zastaví 
    3. Kód běžící na **BSP** prohledá paměť, zda najde `_MP_`
    4. Pokud nenajde, tak se zavede jednoprocesorové jádro
    5. Pokud našel, tak se inicializuje **APIC** **BSP** 
        - děje se v protected-mode
    6. Kód běžící na **BSP** postupně vzbudí AP pomocí **Init-IPI** (_Inter-Processor Interrupt_)
    7. Kód běžící na AP ho přepně do protected-.mode a začně vykonávat svoji další činnost
    8. Jakmile jsou inicializovány všechny **AP**, **BSP** přepne **I/O APIC** do symetrického I/O režimu
        - routovací tabulka, která přesměruje přerušení od sběrnic periferií na některý lokální APIC
    9. Pokračuje se vlastní inicializací SMP jádra

## Zkratky
- **MBR** (Master Boot Record)
    - má za úkol načíst ze zbytek zavaděče ze správného/aktivního/vybraného disku
        - přímo zavaděč OS
        - manažer, který umožní si vybrat, který OS se má zavést, je-li jich nainstalováno více
        - může tam být i nějaký virus 
- **GPT** (Guid Partition Table)
    - načisto udělaná tabulka diskových oddílů, tak aby se nemusel vláčet omezující balast z minulosti
- **TSR** (Terminate and Stay Resident)
- **IRQ** (Interrupt Request)
- **PLT** (Procedure Linkage Table)
    - označení proměnných ukazujících na symboly (funkce, proměnné, atd...) se deklaruje pomocí:
        - `__declspec` (`dllimport`) pod Windows
        - `.rel.text` pod Linuxem
    - značka říká, že se má na příslušnou adresu zapsat hodnota získaná pomocí `dlsym/GetProcAddress` (zařídí zavaděč knihovny)
- **PIC** (Programmable Interrupt Controller)
    - překládá číslo IRQ na index do tabulky vektoru přerušení
- **APIC** (Advanced Programmable Interrupt Controller)
    - umožňuje směrovat mnohem více IRQ než PIC
    - umožňuje konstrukci SMP
        - každý CPU má svůj Local APIC
            - 224 IRQ
            - prvních 0-31 IRQ je vyhrazeno
    - naprogramováním APIC lze směrovat jednotlivé IRQ na jednotlivé CPU
- **PIC** (Position Independent Code)
    - při dynamickém linkování překladač musí zajistit, aby se veškerý kód adresoval relativně k program counteru (`IP` registr)
- **MMU** (Memory Management Unit)
    - stará se o převod virtuální adresy na fyzickou adresu
    - HW komponenta
    - [CPU] -> (virtuální adresa) -> [MMU] -> (fyzická adresa) -> [Paměť]
- **GOT** (Global Offset Table)
    - tabulka obsahující adresy symbolů
    - používá se při dynamickém linkování knihoven
    - privátní pro každý proces
    - nezměněný kód knihovny bude sdílený pro všechny procesy
- **BSP** (Bootstrap Processor)
    - po zapnutí počítače a inicializaci je aktivní pouze jeden procesor, respektive jedno procesorové jádro - tzv. **BSP** aneb _bootstrap procesor_

