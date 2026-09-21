
# Appendix K: Common Code Examples

This is a selection of examples to give an idea on how to program different parts of the Commander X16 and use various kernal functions and api's.

## Index

* [Assembler](#assembler)
	* [Startup](#startup)
* [C Language](#c-language)
* [Prog8](#prog8)

## Assembler
### Startup
Some assemblers, like the [ACME Cross-Assembler](https://sourceforge.net/projects/acme-crossass/) do not automatically provide a BASIC stub for starting the program. That means it is up to the programmer to create that stop, enabling the user to start the program by typing `RUN` after loading it.  
This is an example for ACME assembler on how to create the BASIC stub.
```assembler
*=$0801						; BASIC programs start at $0801
!word	last_line			; Pointer to next/last line of BASIC code
!word	$000A				; Line number $000A = 10
!byte	$9E					; BASIC token for SYS command
!byte	$30+(main/1000)%10	; Address of main function of program
!byte	$30+(main/100)%10	; in PETSCII
!byte	$30+(main/10)%10
!byte	$30+(main/1)%10
!byte	$00					; End of BASIC line
last_line:
!word	$0000				; End of BASIC program
; Above creates a BASIC program that looks like this:
; 10 SYS 2061
main:
	rts
```
## C Language
## Prog8
<!-- For PDF formatting -->
<div class="page-break"></div>
