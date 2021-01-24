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
    - **fault** - opravitelný chyba, program může pokračovat (např. **Page Fault**)
    - **abort** - neopravitelná chyba
        - např. **Double Fault**, kdy dojde k Page Fault, ale handler je ve stránce, která není v RAM

#### Exception handler
- v případě, že procesor nedokáže zavolat obsluhu vyjímky, dojde k Double Fault
    - pokud ani pro ni nebude obsluha, dojde k **Triple Fault**
    - to je známkoou toho, že je něco špatně v obsluze první vyjímky
        - vyjímku nelze zamaskovat
        - buď budeme psát perfektní, bezchybný kód, anebo alespoň psát spolehlivé obsluhy vyjímek

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
