# eZ80F91 Silicon Bugs (?)
My own Errata for Zilog eZ80F91 AcclaimPlus! ASSP current production silicon.

According to Zilogs own Errata UP012703-1110, there are no more bugs in 
chips with manufacturing datecode after 0929 (so since 2009!). IMHO, that's not the case.

Unfortunately, i can't find a way to get in touch with them directly, nor
did i ever got any response to my inquiries through the contact form on the Zilog
web site. So i'll try to describe my own findings here. If anyone else is struggeling
with mysterious behavior of that chip and can't find any explanation for it on the web,
maybe, it is of help to someone out there. 

If someone over at Zilog reads this, pretty please get in touch with me.
Maybe i am doing something wrong, that can be fixed by someone who knows the architecture of the eZ80 core. 
Maybe the errata sheet needs an update and/or, much appreciated, there is a workaround, that i can't see ATM.

I am currently developing a product using EZ80F91AZA50EK with newly purchased chips
in 2025. 

The system consists mainly of the eZ80F91 at 33 MHz, 4 Mbytes of 10 ns SRAM and an FPGA, 
that is able to access this SRAM by use of BUSREQ and BUSACK handshaking. 

Timing on the bus is verified and holds up to the specs. Bus-turnaround-timing
is relaxed. The FPGA releases the bus one clockcycle early before
releasing BUSREQn. It also only starts driving the bus one clock cycle
after it detects BUSACKn to be Low.
The bus has pull-up resistors on all signals, to prevent them from floating.

## 1)  BUSREQn confuses the eZ80 while executing prefixed instructions:
If the BUSREQn signals becomes low at the same time as an instruction
with a memory mode prefix is fetched from memory, the eZ80 will grant the bus by
bringing BUSACKn Low but will fetch the next instruction from the wrong address after
it has bus ownership again. 
Mainly, it seems like the wrong processor mode is used for the instruction fetch, depending on 
the prefix used, before. 

Here is a Code snippet with a .SIL prefixed instruction while running in ADL mode:
```
F80F61  B7        OR   A,A
F80F62  52 ED 42  SBC.SIL HL,BC
F80F65  EB        EX   DE,HL           <- On this Address, bits[23:16] are 00 after BusRq/Ack Cycle
F80F66  ED 33 03  LEA  IY,IY+%3
F80F69  20 DE     JR   NZ,%F49
```
If the BUSREQ signals hits exactly at the instruction with .SIL prefix, the first instruction
after the eZ80 reclaiming the bus will be read with the upper byte of PC at 00.
If running in Z80 Mode, then the same happens with .LIS/.LIL prefixes.
With a Long prefix in Z80 Mode, the CPU will read the next Instrucion with the upper
Byte of PC set to an arbitrary value, if BUSREQ/ACK happens to hit exactly on the prefixed
instruction. 

Here is a real world logic-analyzer screenshot of that problem happening with the above code:\
<img width="1033" height="459" alt="REQ_ACK_Annotated" src="https://github.com/user-attachments/assets/d66d8c66-3310-4c0d-9408-d289a8397603" />
Sampling rate is four times the bus clock of 33 MHz (132 MHz).

## 2) IM Instruction is ignored entirely. Likely not a bug, but on purpose:
The eZ80F91 has a vectored interrupt controller, that handles a whole lot of interrupt sources and uses
a pretty large vector table. In my application, i need only one of the timers to produce interrupts, nothing else.\
Also, the I register is (ab)used for temporary storage of some mode settings in a pretty large existing code base.\
So vectored mode is no option for me. On the original Z80, this code ran in IM 1 with all interrupts causing a jump
to 38h.

Of course, yes, the datasheet says:
```
(quote) 
On the eZ80F91 device, all maskable interrupts use the CPU’s vectored interrupt function
(endquote)
```
But the IM 0-2 instructions are there, the datasheet mentiones them in the opcode maps.\
Unfortunately, it seems like they are ignored entirely.  It just won't let me have "all" of my interrupts at 38h,
but continues to use the vector table, even if i have IM 1 in my code.

Furthermore, historically, the IRQ input was used to trigger either of the three possible responses. Why stop doing so?\
One of them was, indeed, pulling a vector number from the bus, then index into the vector table and pull 
the handler address from there.  Thats perfectly fine for IM 2 and the internal vector controller could provide that vector number.
If IM 1 is selected, just jump to 38h and leave the vector controller alone. Easy...? not.  

I then resorted to use NMI instead of normal maskable interrupts and implement that one timer in FPGA logic.
That lead to issue #3 further down. 

## 3)  NMI confuses the eZ80 while executing prefixed instructions:
After settling on using NMI instead of normal maskable interrupts due to the eZ80F91 ignoring the IM instruction, 
i found another case of missbehavior with regard to prefixes in conjuction with NMI handling.

Example code in Z80 Mode:
```
018C  5B 21 04 00 70  LD.LIL HL, 700000h
0191  11 42 02        LD DE, 0242h
0194  01 40 00        LD BC, 0400h
0197  49 ED C3        OTIRX.LIS
019A  AF              XOR A
```
This will copy 1024 bytes from 70:0000 to I/O Port 0242h. Unless an NMI hits the OTIRX.LIS Instruction.
The eZ80 will first interrupt the OTIRX.LIS instruction as expected. It then pushes a two byte return address, 
but erratically uses the ADL stack pointer (SPL) instead of the Z80 Mode stack pointer (SPS). 
After the successful jump to 00:0066h, even the first instruction of the NMI Handler is executed like in 
ADL Mode and my PUSH AF there is executed by pushing three bytes to SPL instead of two bytes to SPS.  
After that, execution continues  with proper Z80 mode memory accesses. 
This leads to the fact, that the POP AF and RETN  at the end of my NMI service routine will now pull 
data from the Z80 Mode stack (SPS), where it was never pushed to, in the first place.
This will obviously end in a crash, as well.

Here is a Logic Analyzer Screenshot of that bug happening: 
<img width="1364" height="477" alt="NMI_Annotated" src="https://github.com/user-attachments/assets/1de91811-71b8-4ad3-ba1d-024394d21588" />
Sampling clock is PHI (Clock Out)  from the processor itself.

This _COULD_ be a response to entering the NMI with Mixed Memory Mode Bit enganged. But i do not enable Mixed Mode anywhere. 
I'd also suspect, that the two byte return address store would then have been a three byte store with the source memory mode
in the third byte.  But that isn't the case, either.  Also, this happens only, if the NMI hits a prefixed instruction. Any other
time, it reacts as expected.

## 4)  IRQ confuses the eZ80 while executing prefixed instructions: 
While looking after some other strange effects with NMI, i wanted to have a look at IRQ, too. 
Turns out: When running in Z80 mode and using prefixed instructions like in example #3, the outcome
is exactly the same as in #3.  So, no... IRQ doesn't work in Z80 mode while using memory mode prefixes, either.

## What _SEEMS_ to work as a workaround is the following: 
When running in Z80 Mode and you need to use interrupts and prefixed instructions in your application at once (oh, behave!!)
then try with mixed memory mode. 
- Setup the 16 bit I Register to the beginning of the vector table.
- point SPS to a valid stack location
- point SPL to a valid !different! stack location
- set the vector for the peripheral to the service routine
  -> That service routine needs to be written in ADL Mode, obviously!! 
- enable interrupts.
- enjoy

That way, no more strange prefix related issues occured for me.

As my application needs Z80 Mode interrupt Handling, i came up with the following Service Routine: 
```
  .ORG 38h
Z80MODE_IRQ_ROUTINE:
  ... do things in Z80 Mode
  ei
  reti

  .ORG xyz
INTENTRY:       ; This is the address, i placed in the vector table. Routine starts in ADL Mode due to MIXED MODE
  PUSH IX       ; Save IX to SPL
  PUSH HL       ; Save HL to SPL
  LD IX,0
  ADD IX,SP     ; IX now Points to the bottom of SPL Stack
  LD L, (IX+7)  ; Get the return Address from SPL into HL
  LD H, (IX+8) 
  PUSH.S HL     ; Push return Address to SPS (Z80 Mode Stack)
  POP HL
  POP IX        ; Restore IX,HL from SPL
  INC SP
  INC SP
  INC SP        ; Get rid of the return Address on SPL including the memory mode bit
  JP.SIS Z80MODE_IRQ_ROUTINE  ; Jump to Z80 Mode service routine using mode switching JP.
```
