# traps

## Trampoline

Het is je misschien opgevallen dat een pagina genaamd *trampoline* gemapt is in de adresruimte van de kernel en in de adresruimte van ieder proces.
Misschien herinner je je zelfs dat we verwezen hebben naar de trampolinepagina in de vorige oefenzitting.
Tijd om uit te leggen wat deze pagina daar doet.

Eerder hebben we vermeld dat, wanneer er zich een exception voordoet, of wanneer we een `ecall`-uitvoeren, de RISC-V hardware van uitvoermodus verandert en springt naar de trap handler.
Neem nu bijvoorbeeld de `ecall`.
Uitgevoerd vanuit user-mode zorgt dit ervoor dat de processor switcht naar supervisor-mode en vervolgens springt naar het adres van de trap handler.

> :information_source: Ook interrupts zorgen voor een sprong naar de trap handler. Dit wordt in een latere sessie behandeld.

De processor wil nu dus code van de kernel uitvoeren, om te system call af te kunnen handelen.
De code van de kernel is echter gemapt in een eigen adresruimte.
Op het moment dat de `ecall` wordt uitgevoerd, wijst het
`satp`-register echter nog steeds naar de page table van het user proces.
De processor springt dus naar de trap handler, de trap handler moet bijgevolg gemapt zijn in het user proces.

Om die reden mappen we de trampolinepagina in iedere adresruimte op hetzelfde adres.
Zo kan de RISC-V hardware simpelweg springen naar een vast adres en weten we zeker welke pagina daar gemapt zal staan.

* Bekijk de code van [`kernel/trampoline.S`][trampoline]

De trampoline zal alle registers bewaren in het trapframe bij het wisselen van user-space naar kernel-space.
Daarnaast zal de trampoline ook `satp` wisselen zodat deze wijst naar de top-level page table van de kernel.

Je vraagt je misschien af waarom de trampoline ook in de kernel address space gemapt moet staan.
Meteen nadat het `satp`-register aangepast wordt, wisselt meteen de volledige adresvertaling van de processor.
De programmateller laadt instructies uit het geheugen op basis van adressen.
Stel dat je `satp` zou wijzigen zonder op dezelfde plaats in de andere adresruimte dezelfde code te mappen, zou plots de code uitgevoerd worden die in de andere adresruimte op dezelfde plaats gemapt staat (en dus niet meer de trampolinecode).

* De trampolinepagina staat gemapt met `R` (read) en `X` (execute) permissies. Daarnaast is de pagina enkel toegankelijk in supervisor mode (de `U`-bit is inactief). Stel dat de trampolinepagina ook `W` (write) permissies zou hebben en user-mode access zou enabled zijn. Kan je bedenken hoe dit voor problemen zou kunnen zorgen? 

[trampoline]: https://github.com/besturingssystemen/xv6-riscv/blob/1f555198d61d1c447e874ae7e5a0868513822023/kernel/trampoline.S