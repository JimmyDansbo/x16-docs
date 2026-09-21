
# Appendix K: Common Code Examples

This is a selection of examples to give an idea on how to program different parts of the Commander X16 and use various kernal functions and api's.

## Index

* [Assembler](#assembler)
	* [Startup](#startup)
	* [Mouse Handling](#mouse-handling)
	* [Interrupt](#interrupt)
* [C Language](#c-language)
* [Prog8](#prog8)

## Assembler
### Startup
Some assemblers, like the [ACME Cross-Assembler](https://sourceforge.net/projects/acme-crossass/) do not automatically provide a BASIC stub for starting the program. That means it is up to the programmer to create that stop, enabling the user to start the program by typing `RUN` after loading it.  
This is an example for ACME assembler on how to create the BASIC stub.
```ASM
*=$0801						; BASIC programs start at $0801
!word	@last_line			; Pointer to next/last line of BASIC code
!word	$000A				; Line number $000A = 10
!byte	$9E					; BASIC token for SYS command
!byte	$30+(main/1000)%10	; Address of main function of program
!byte	$30+(main/100)%10	; in PETSCII
!byte	$30+(main/10)%10
!byte	$30+(main/1)%10
!byte	$00					; End of BASIC line
@last_line:
!word	$0000				; End of BASIC program
; Above creates a BASIC program that looks like this:
; 10 SYS 2061
; 2061 is the same as $080D which is where the main label is currently placed

main:
	rts
```
### Mouse Handling
The kernal has three functions for handling the mouse.
* [mouse_config](https://github.com/JimmyDansbo/x16-docs/blob/code_examples/X16%20Reference%20-%2005%20-%20KERNAL.md#function-name-mouse_config): $FF68 - configure mouse pointer
* [mouse_scan](https://github.com/JimmyDansbo/x16-docs/blob/code_examples/X16%20Reference%20-%2005%20-%20KERNAL.md#function-name-mouse_scan): $FF71 - query mouse
* [mouse_get](https://github.com/JimmyDansbo/x16-docs/blob/code_examples/X16%20Reference%20-%2005%20-%20KERNAL.md#function-name-mouse_get): $FF6B - get state of mouse

Under normal circumstances, it is not necessary to call `mouse_scan`as the kernal interrupt handler takes care of it, however if the kernal interrupt handler is completely replaced, this function will have to be called periodicly to actually get mouse updates.

```ASM
r0L = $02
r0H = $03
r1L = $04
r1H = $05

MOUSE_LEFT	= %00000001
MOUSE_RIGHT	= %00000010
MOUSE_MIDDLE	= %00000100
MOUSE_BTN4	= %00010000
MOUSE_BTN5	= %00100000

MOUSE_CONFIG	= $FF68
MOUSE_GET		= $FF6B

mousex:	.word	0
mousey:	.word	0


	; Configure default mouse pointer on the standard 80x60 resolution
	lda	#1			; Default mouse pointer
	ldx	#80			; Screen width
	ldy	#60			; Screen height
	jsr	mouse_config

loop:
	wai				; delay until next interrupt
	ldx	#r0L		; store coordinates in r0L-r1H
	jsr	mouse_get
	; Copy mouse coordinates from zeropage to global variables
	ldy	r0L
	sty	mousex+0
	ldy	r0H
	sty	mousex+1
	ldy	r1L
	sty	mousey+0
	ldy	r1H
	sty	mousey+1
	cpx	#0			; Check if scroll wheel has moved since last call
	beq	:+			; to mouse_get
	jsr	my_scroll_wheel_handler
	; Check if left or right mouse button has been pressed
:	and	#MOUSE_LEFT|MOUSE_RIGHT
	beq	:+
	jsr	my_button_handler
:
	...
	bra	loop
```
### Interrupt

```ASM
```
## C Language
## Prog8
<!-- For PDF formatting -->
<div class="page-break"></div>
