- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Traps](#traps)
  - [Traps in RISC-V](#traps-in-risc-v)
    - [CSR Registers](#csr-registers)
    - [Delegation register](#delegation-register)
    - [Trap vectors](#trap-vectors)
  - [Virtual memory](#virtual-memory)
    - [Trampoline](#trampoline)
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
* Hoofdstuk 4 van het [xv6 boek](https://github.com/besturingssystemen/xv6-riscv) te hebben gelezen.

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

* **Oefening** Voeg floating point support toe (**TODO**)
  * Wij geven instructies over hoe te enablen in hardware
  * Bewerk dus trapframe
  * Verifeer m.b.h.v. multithreaded programma
  * *Doel:* Aanpassing moeten maken in de trap code zelf

# Exceptions

## Error handling

* **TODO**
* **Oefening** Decodeer bepaalde traps gebruik makend van de trap tabellen

## System calls

* **TODO**

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
