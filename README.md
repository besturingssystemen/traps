- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Traps](#traps)
  - [Traps in RISC-V](#traps-in-risc-v)
    - [CSR Registers](#csr-registers)
    - [Delegation register](#delegation-register)
    - [Trap vectors](#trap-vectors)
  - [Virtual memory](#virtual-memory)
    - [Trampoline](#trampoline)
  - [Floating point ondersteuning](#floating-point-ondersteuning)
- [Exceptions](#exceptions)
  - [Error handling](#error-handling)
  - [System calls](#system-calls)
  - [On-demand paging](#on-demand-paging)
  - [Debugging](#debugging)
- [Interrupts](#interrupts)
  - [Timer interrupts](#timer-interrupts)
  - [Device drivers](#device-drivers)

# Voorbereiding

Ter voorbereiding van deze oefenzitting word je verwacht:

* De oefenzitting virtual memory te hebben voltooid.
* Hoofdstuk 4 en 5 van het [xv6 boek](https://github.com/besturingssystemen/xv6-riscv) te hebben gelezen.

# GitHub classroom

* **TODO**

# Traps

De uitvoering van machinecode door een processor is meestal erg voorspelbaar:

1. Een instructie wordt geladen op de locatie waarnaar de program counter verwijst.
2. De instructie wordt uitgevoerd
3. De programmateller wordt verhoogd
4. Herhaal

Sommige machine-instructies, zoals jumps, passen expliciet de programmateller aan.
Zo implementeren we lussen, functies, enzovoort.

Het kan echter ook gebeuren dat een instructie *faalt*.
Denk bijvoorbeeld aan de volgende simpele instructie:

```asm
sw t0, (t1)
```

De waarde in register `t0` wordt opgeslagen op het adres in register `t1`.
Indien we deze instructie uitvoeren op een RISC-V processor waarop *virtual memory* enabled is, in bijvoorbeeld *user mode*, kan het zijn
dat deze instructie niet mag uitgevoerd worden.
Het kan namelijk zijn dat de pagina van het adres niet gemapt is als *writable*.
Hoe kan de processor deze instructie dan uitvoeren?

Dit specifiek scenario is een voorbeeld van een *exception*.
Iedere *exception* veroorzaakt een *trap*.
In plaats van de *faulting instruction* uit te voeren, zal de processor springen naar een andere locatie in het geheugen.
Op die locatie vinden we een *trap handler*.
Dit is code, typisch een deel van het besturingssysteem, op een vaste locatie in het geheugen.
Deze code heeft de specifieke taak om te identificeren wat het probleem was en om vervolgens te beslissen of het al dan niet opgelost kan worden.

Na het lezen van bovenstaande paragraaf maak je misschien enkele terechte bedenking: in welke adresruimte bevindt zich de trap handler?
In de adresruimte van de kernel, of die van het user proces?
Of staat de trap handler op een fysiek adres?
Wijzigt het privilegeniveau van de processor bij het afhandelen van een trap?

Om hierop te antwoorden moeten we even uitleggen exact hoe een trap in RISC-V afgehandeld wordt.

## Traps in RISC-V

### CSR Registers

RISC-V heeft een verzameling speciale registers genaamd *control and status registers* (CSR).
Deze registers worden specifiek gebruikt om bepaalde speciale functionaliteiten te kunnen implementeren, zoals exceptions.
In onderstaande afbeelding uit de [RISC-V specificaties](https://riscv.org/technical/specifications/) worden enkele van deze CSR registers gerelateerd aan exception handling beschreven:

![CSR-registers](img/CSR.png)

Merk op dat elk van deze registers de letter `u` als prefix heeft.
Deze `u` staat voor user-mode.
Deze specifieke registers kunnen gebruikt worden door de processor, wanneer deze in user-mode (of een hoger privilegeniveau) uitvoert.
Elk van deze registers hebben ook een `s`- en `m`-equivalent (bijvoorbeeld `sstatus` en `mstatus`).

### Delegation register

Op het moment dat een trap optreedt, moet beslist worden in welk privilegeniveau deze afgehandeld moet worden.
Standaard is dit in machine mode, het hoogste privilegeniveau.
Er is echter een mogelijkheid voorzien om traps te forwarden naar supervisor mode of user mode.

Om traps te forwarden wordt gebruik gemaakt van speciale delegatieregisters (CSRs).
In totaal zijn er vier verschilende delegatieregisters: `medeleg`, `sedeleg` voor exceptions en `mideleg` en `sideleg` voor interrupts.
Op het gedeelte interrupts komen we later in de sessie terug.

Indien de oorzaak van de trap een exception was, wordt gekeken in de `medeleg` register.
Elke bit van dit register is gelinkt aan een specifieke exception.
Indien de bit van een specifieke exception op `1` staat, wordt deze exception doorgestuurd naar het lagere privilegeniveau.
Indien de specifieke processor een supervisor mode heeft, zal de exception dus in supervisor mode afgehandeld worden.
Via `sedeleg` kunnen vervolgens exceptions verder doorgestuurd worden naar user mode.

### Trap vectors

Eenmaal bepaald is in welke modus een exception moet afgehandeld worden, worden de overeenkomstige CSRs geconsulteerd.
De registers `mtvec`, `stvec` en `utvec` bevatten respectievelijk de adressen van de machine mode trap handler, supervisor mode trap handler en user mode trap handler.

Stel dat de exception in machine mode wordt afgehandeld.
De oorzaak van de exception wordt geschreven naar het CSR `mcause`.
De waarde van de program counter op het moment dat de exception zich voordeed, wordt geschreven naar `mepc`.
Indien de exception veroorzaakt werd door een adres aan te spreken, zal de waarde van dit adres in `mtval` geplaatst worden.
In de andere privilegeniveaus gebeurt exact hetzelfde maar dan met de registers van dat privilegeniveau.

De tabellen uit de specs voor [mcause](img/mcause.png) en [scause](img/scause.png) kunnen gebruikt worden om te bepalen welke waarde in deze registers geplaatst worden per interrupt of exception.

**TODO** ucause tabel?

* **Oefening** Een programma in user-mode voert de instructie op adres `0x0x4B1D` uit: `sb t0, 0x0FF1CE`. Dit adres bevindt zich in een pagina die niet schrijfbaar is. De waarde van het `medeleg` register staat op `0xffffffff`, de waarde van het `sedeleg` register op `0x0`. In `mtvec` staat het adres `0xdeadbeef`, in `stvec` staat het adres `0xcafebabe` en in `utvec` staat het adres `0xBAADF00D`. Op welk adres bevindt zich de trap handler die de exception zal afhandelen? In welk privilegeniveau wordt de exception afgehandeld? Welke waarden zullen er in de `xcause`, `xepc` en `xtval` registers geplaatst worden (en welke letter is *x*)?

## Virtual memory

We weten nu het hardware-mechanisme waarmee traps afgehandeld kunnen worden.
Er rest ons echter nog een belangrijke vraag om te beantwoorden: worden virtuele of fysieke adressen gebruikt bij het afhandelen van traps?
Dit hangt volledig af van de huidige configuratie van de processor.
Indien paging aanstond op het moment dat de trap zich voordoet, zal de waarde in de `xtvec` registers verwijzen naar virtuele adressen.
Het adresvertalingsmechanisme blijft dus actief.

Zeer attentieve studenten merken hierbij misschien een probleem op.
Stel dat we van privilegeniveau wijzigen door de exception.
Neem bijvoorbeeld aan dat we een exception hebben die zich voordoet in user-mode en afgehandeld wordt in supervisor-mode.
Op het moment dat de processor springt naar de waarde in `stvec` zal `satp` nog steeds verwijzen naar de page table van het user proces.
De adresvertaling gebeurt dus nog steeds volgens de mapping van het user proces. De processor zal dus springen naar code in de user space met een hoger privilegeniveau (in dit geval supervisor mode).

Een besturingssysteem zal er dus voor zorgen dat er kernel code geschreven wordt op het adres `stval` en dat het proces zelf deze code niet kan bewerken (door de pagina niet schrijfbaar te maken).
De code op dat adres kan vervolgens de waarde van `satp` wijzigen, en zo de page tables van de kernel activeren, om ervoor te zorgen dat we verder kunnen uitvoeren in de adresruimte van de kernel zelf.

De code die verantwoordelijk is voor het opvangen van traps in xv6 bevindt zich in de trampolinepagina.
De trampolinepagina is gemapt op het hoogste adres in de virtuele adresruimte van *elk* user-space proces.

### Trampoline

Het is je misschien opgevallen dat de *trampoline* niet enkel gemapt is in de adresruimte van elk user-space proces maar ook in de adresruimte van de kernel zelf.
Ook in de kernel is deze pagina gemapt op het hoogst mogelijke virtuele adres.

Eerder hebben we vermeld dat de trampolinepagina `satp` zal wijzigen.
`satp` wijzigen zorgt er echter voor dat plotseling alle virtuele adressen volgens een volledig andere mapping vertaald worden.
De program counter zelf bevat het virtuele adres van de huidige locatie in de code!
Indien, bij een wisseling van user space naar kernel space, de trampolinepagina niet op hetzelfde virtuele adres gemapt is in beide adresruimten, zal de volgende instructie die geladen wordt door de processor zich niet bevinden in de trampolinepagina.
Om dat te vermijden wordt deze pagina dus op hetzelfde virtuele adres gemapt in iedere adresruimte.

* Bekijk nu de code van [`kernel/trampoline.S`][trampoline]

Zoals je kan zien voert de trampoline twee taken uit:

1. Het bewaren (en herstellen) van alle registerwaarden in de [`trapframe`][trapframe]. Dit is nodig om te verzekeren dat bij terugkeer uit de trap, het actieve programma kan voortgezet worden met correcte registerwaarden.
2. Het wijzigen van de `satp` om zo te wisselen van page tables bij transities tussen user-space en kernel-space

Bij de overgang van user-space naar kernel-space wordt, aan het einde van de trampolinepagina, de kernel-functie [`usertrap()`][usertrap] opgeroepen in `kernel/trap.c`.
Om terug te keren van kernel-space naar user-space definieert de trampolinepagina de functie `userret`.

* **Oefening** De trampolinepagina staat gemapt met `R` (read) en `X` (execute) permissies. Daarnaast is de pagina enkel toegankelijk in supervisor mode (de `U`-bit is inactief). Stel dat de trampolinepagina ook `W` (write) permissies zou hebben en user-mode access zou enabled zijn. Wat voor probleem zou dit kunnen opleveren?

## Floating point ondersteuning

Het is je misschien opgevallen dat alle user space programma's die we tot nu toe geschreven hebben, enkel _integer_ operaties gebruiken.
RISC-V ondersteunt ook _floating point_ operaties om niet-gehele getallen te bewerken.
Op de meeste processoren, alsook op RISC-V, worden zulke operaties uitgevoerd door een zogenaamde _floating point unit_ (FPU).
Deze FPU zal berekeningen uitvoeren op speciale registers die onafhankelijk zijn van de general purpose registers die gebruikt worden voor integer operaties.
Op RISC-V zijn er 32 floating point registers genaamd `f0` tot `f31`.

1. Schrijf een user space programma dat gebruik maakt van floating point operaties.
   Als je bewerkingen uitvoert op variabelen van het type `double`, zal de compiler floating point instructies genereren.
   Op RISC-V beginnen alle floating point instructies met een `f`, bijvoorbeeld: `fld` (float load), `fadd`, `fmul`.
   Verifieer dat je programma daadwerkelijk zulke instructies gebruikt via `objdump` (vervang `user/_test` door de executable van je eigen programma):
   ```shell
   riscv64-linux-gnu-objdump -d user/_test
   ```

Als je je programma uitvoert, zul je merken dat het crasht door een exceptie.

2. Welke exceptie wordt er precies gegenereerd en door welke instructie?

Op RISC-V staat de FPU uit na het opstarten.
Het `mstatus` CSR bevat twee bits (`FS`) die de status van de FPU beheren.
De FPU kan aangezet worden door `FS` op `01` te zetten.

3. Zorg ervoor dat de FPU globaal aanstaat.
   De initiële waarde voor `mstatus` [wordt gezet][write mstatus] in de functie [`start`][start], de eerste C-functie die in de kernel wordt uitgevoerd.
   Je kan zien dat `mstatus` eerste gelezen wordt in de variabele `x` (geweldige naam!) via de functie [`r_mstatus`][r_mstatus].
   Dan worden er een aantal configuratie bits gezet in `x` (je hoeft niet te begrijpen wat deze precies doen) vooraleer `x` terug naar `mstatus` geschreven wordt via [`w_mstatus`][w_mstatus].
   Je kan de FPU dus aanzetten door de eerste bit van `FS` (bit 13 van `mstatus`) op 1 te zetten in de variabele `x`.

> :information_source: De Linux kernel staat voor efficiëntie redenen geen floating point code toe in kernel mode.
> De FPU zal daar dus niet globaal aanstaan maar enkel bij het switchen naar user mode aangezet worden.
> Ook de meeste user mode programma's maken geen gebruik van de FPU en het kan dus qua energieverbruik beter zijn om ook voor user space programma's de FPU niet standaard aan te zetten.
> De kernel zou bijvoorbeeld de exceptie die gegenereerd wordt wanneer er een floating point instructie gebruikt wordt terwijl de FPU uitstaat op kunnen vangen en de FPU dan pas aanzetten.
> Om deze oefening eenvoudig te houden, zette we de FPU globaal aan maar we houden jullie niet tegen om een efficiëntere implementatie uit te proberen.

Verifieer nu dat je test programma uitgevoerd kan worden zonder excepties te genereren.

4. Breid je test programma uit door een child te `fork`en en in parent _en_ child floating point operaties uit te voeren.
   Zorg er ook voor dat parent en child een aantal keer naar kernel mode moet switchen (door, bijvoorbeeld, een syscall zoals `sleep` te gebruiken).
   Print te resultaten van de berekeningen in parent en child af.
   Merk je iets op?
   Voer je test programma meerdere keren uit en bekijk de resultaten.
   > :warning: De `printf` versie in xv6 ondersteunt het afprinten van `double`s niet.
   > Cast een `double` die je wilt printen dus eerst naar een `int` (`(int)val`).

Je hebt waarschijnlijk gemerkt dat je inconsistente resultaten krijgt (zo niet, ga terug naar punt 4).
Zoals eerder uitgelegd, moet de code in de trampoline de waarden in de general purpose register opslaan in het trapframe om de waarden niet te verliezen.
De FPU gebruikt echter andere registers en deze worden niet bewaard in het trapframe.

5. Zorg ervoor dat de floating point registers juist opgeslagen en herstelt worden door de trampoline code:
    - Breid [`struct trapframe`][trapframe] uit.
    - Voeg code toe aan de [trampoline][trampoline] om alle floating point registers op te slaan in (gebruik de `fsd` instructie) en weer te herstellen uit (`fld`) het trapframe.
6. Verifieer dat je test programma nu _wel_ consistente resultaten geeft.

# Exceptions

We kennen nu het mechanisme waarmee traps worden afgehandeld.
Tijd om te kijken naar toepassingen dit mechanisme.
We concentreren ons eerst op toepassingen van exceptions die, zoals eerder vermeld, een trap veroorzaken.
Nadien bespreken we interrupts, die ook door traps afgehandeld worden.

## Error handling

De meest standaard toepassing waaraan gedacht wordt bij exceptions is het afhandelen van fouten.

Zo is er bijvoorbeeld een regel in RISC-V dat een `lw` (load word) instructie enkel mag opgeroepen worden met een adres dat een veelvoud is van 4. Een word is 4 bytes groot, als we ons geheugen opdelen in words is het byte-adres van ieder word namelijk een veelvoud van 4.
Wanneer we een `lw` instructie proberen uitvoeren met een adres dat geen veelvoud is van 4, wordt de exception *load address misaligned* gesmeten.

Zoals we weten, wordt vervolgens de controle doorgegeven aan de trap handler in één van de verschillende privilegeniveau's, afhankelijk van de waarde van de delegatieregisters.
Laten we even aannemen dat de exception in de kernel wordt afgehandeld, in supervisor mode.

Wat kan een kernel doen om dit probleem op te lossen?
De handler zou kunnen beslissen om de `lw` instructie over te slaan, maar dan zou het uitvoerende proces incorrect werken.
In de meeste gevallen is het antwoord dus simpelweg, het proces vroegtijdig beeïndigen en een boodschap geven dat het proces een exception heeft veroorzaakt met daarbij informatie over de specifieke exception.

De boodschap *segmentation fault* die je krijgt in vele Linux-distributies is een voorbeeld van een exception die niet opgelost kan worden door het besturingssysteem als gevolg van een fout in het programma.

* **Oefening:** **TODO** In jullie xv6 repository hebben we enkele simpele programma's toegevoegd, die elk exceptions veroorzaken.
Voer deze programma's uit en gebruik de tabel [scause](img/scause.png) om te bepalen wat er misloopt.
Gebruik het commando `risc64-linux-gnu-objdump -d <executable>` om te achterhalen naar welke regel code de programmateller verwees op het moment dat de exception zich voordeed. Probeer ten slotte het probleem op te lossen door het `.S` bronbestand te wijzigen.

## System calls

Misschien is het je opgevallen dat environment calls (system calls) ook in de scause tabel stonden.
Zoals eerder reeds vermeld in de oefenzitting over system calls, worden deze geïmplementeerd met behulp van exceptions.

De `ecall` instructie genereert expliciet een exception.
De code van de exception verschilt afhankelijk van het privilegeniveau waarin de `ecall` uitgevoerd wordt.
Zo kan je ook vanuit de kernel een `ecall` uitvoeren, die dan in machine mode opgevangen zou kunnen worden.

* Bekijk de code in [`usertrap()`][usertrap]. Herinner je dat deze functie opgeroepen wordt door de trampoline. Je zou ondertussen de code in usertrap moeten begrijpen. Merk op dat deze code ofwel een system call oproept, ofwel een interrupt afhandelt (hierover meer in volgende sectie) ofwel de error message print die we in vorige oefening hebben gedecodeerd.

In het geval van een system call wordt dus [`syscall()`][syscall] opgeroepen, een methode die jullie ondertussen goed zouden moeten kennen.

## On-demand paging

* **TODO**
* **Oefening** Implementeer `sbrk` door;  
  * Grote regio te reserveren maar nog niet te mappen
  * Bij page fault te checken of dereference in de reserved maar unmapped regio valt
  * Soort on-demand paging in die regio
  * *Doel:* Hoe exceptions gebruiken om functionaliteit te implementeren

## Debugging

* **TODO**
* **Oefening** (misschien?): basic debugger schrijven voor xv6:
  * `./dump_registers <symbol> <executable> <args>`
  * Instructie op locatie `<symbol>` vervangen door `ebreak`
  * User-mode handler in dump_registers.S
  * In geval van ebreak exception
  * Dump alle registers naar stdout
  * Vervang `ebreak` terug door originele instructie
  * continue

# Interrupts

* **TODO**

## Timer interrupts

* **TODO**

## Device drivers

* **TODO**

[trampoline]: https://github.com/besturingssystemen/xv6-riscv/blob/1f555198d61d1c447e874ae7e5a0868513822023/kernel/trampoline.S
[trapframe]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/proc.h#L52
[usertrap]: https://github.com/besturingssystemen/xv6-riscv/blob/27057bc9b467db64a3de600f27d6fa3239a04c88/kernel/trap.c#L37
[syscall]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/syscall.c#L133
[start]: https://github.com/besturingssystemen/xv6-riscv/blob/103d9df6ce3154febadcf9a67791d526ec6b07ac/kernel/start.c#L19
[write mstatus]: https://github.com/besturingssystemen/xv6-riscv/blob/103d9df6ce3154febadcf9a67791d526ec6b07ac/kernel/start.c#L23-L27
[r_mstatus]: https://github.com/besturingssystemen/xv6-riscv/blob/103d9df6ce3154febadcf9a67791d526ec6b07ac/kernel/riscv.h#L23
[w_mstatus]: https://github.com/besturingssystemen/xv6-riscv/blob/103d9df6ce3154febadcf9a67791d526ec6b07ac/kernel/riscv.h#L31
