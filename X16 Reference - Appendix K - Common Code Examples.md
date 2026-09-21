
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

Under normal circumstances, it is not necessary to call `mouse_scan`as the kernal interrupt handler takes care of it, however if the kernal interrupt handler is completely replaced, the function will have to be called periodicly to actually get mouse updates.

```ASM
r0L = $02
r0H = $03
r1L = $04
r1H = $05

MOUSE_LEFT		= %00000001
MOUSE_RIGHT		= %00000010
MOUSE_MIDDLE	= %00000100
MOUSE_BTN4		= %00010000
MOUSE_BTN5		= %00100000

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
When an interrupt occurs, the CPU pushes the current Program Counter and the Flags onto the stack. It then jumps to the address stored in $FFFE-$FFFF.  

The current kernal (r49) has it's internal interrupt handler at address $038B which is the address stored at $FFFE-$FFFF. The function is called `__irq` and can be found in [kernal/drivers/x16/memory.s](https://github.com/X16Community/x16-rom/blob/master/kernal/drivers/x16/memory.s)  
To completely bypass the kernals handling of interrupts, the code at $038B must be replaced with a custom handler.
```ASM
IRQ_FUNC	= $038B

.proc my_interrupt_routine
	; Handle IO with custom code:
	;	Things like reading ps2 data, scanning mouse, joystick & keyboard
	; Nothing is handled automaticly by kernal
	rti		; Do a normal return from interrupt.
.endproc

my_isr:	jmp	my_interrupt_routine

.proc install_custom_interrupt_routine
	sei		; Disable interrupts
	; Overwrite kernal interrupt handler with jump to custom handler
	lda	my_isr+0
	sta	IRQ_FUNC+0
	lda	my_isr+1
	sta	IRQ_FUNC+1
	lda	my_isr+2
	sta	IRQ_FUNC+2
	cli		; Enable interrupts
	rts
.endproc
```
Usually it is not necessary to bypass the kernal interrupt handler completely. The most common way of installing an internal interrupt handler is to replace the address at $0314 with the address of the custom interrupt handler.  
The internal interrupt routine performs the following:  
1. Push register A and current ROM bank to stack
2. Push address of [__irq_ret](https://github.com/X16Community/x16-rom/blob/master/kernal/drivers/x16/memory.s) function to stack
3. Push flags to stack so `rti` can be used to jump to `__irq_ret` function
4. Push CBM IRQ stackframe to stack. Registers `A`, `X`, `Y` pushed in that order
5. Jump indirectly to address stored at `$0314`

That means, a total of 8 bytes have been pushed to the stack before the jump to the address stored at `$0314`. If control is not passed from the custom interrupt handler to the kernal interrupt function, the CBM IRQ stackframe must be popped from the stack before performing an `rti`.  
This will pass control to the `__irq_ret` function which in turn will take care of restoring the ROM bank and register `A` from the stack before actually returning from the interrupt call.
```ASM
IRQ_VECTOR = $0314

semaphore:	.res	1	; Variable being updated by interrupt routine

.proc my_interrupt_routine
	lda	$9F27			; Check if VSYNC is the source of the interrupt
	and	#$01
	beq	@vsync_end
	; Update semaphore variable on each VSYNC interrupt
	sta	semaphore
	; Acknowledge VSYNC interrupt by writing 1 to VERA_ISR
	sta	$9F27
@vsync_end:
	; Handle IO with custom code as kernal interrupt routine is not called
	...
	; Pop the CBM IRQ strackframe
	ply
	plx
	pla
	; Return from interrupt (let __irq_ret handle restore of ROM bank)
	rti
.endproc

.proc install_custom_interrupt_routine
	sei					; Disable interrupts
	; Overwrite address of kernal interrupt routine with address
	; of custom interrupt routine
	lda	#<my_interrupt_routine
	sta	IRQ_VECTOR+0
	lda	#>my_interrupt_routine
	sta	IRQ_VECTOR+1
	cli					; Enable interrupts
	rts
.endproc

main:
	jsr	install_custom_interrupt_routine

loop:
	wai
	lda	semaphore		; If semaphore=1, VSYNC interrupt has happened
	bne	loop
	; VSYNC interrupt has happened, reset semaphore variable
	stz	semaphore
	; Do stuff
	...
	bra	loop
```
The easiest option is to install a custome interrupt handler that simply passes control on to the kernal interrupt function when it has done it's job. This way, the kernal is still handling all of the IO operations such as reading keyboard, mouse and joystick as well as updating system time etc.
```ASM
IRQ_VECTOR	= $0314

semaphore	.res	1	; Variable being updated by interrupt routine
orig_isr	.res	2	; Variable to hold address of kernal interrupt routine

.proc my_interrupt_routine
	lda	$9F27			; Check if VSYNC is the source of the interrupt
	and	#$01
	beq	@vsync_end
	; Update semaphore variable on each VSYNC interrupt
	sta	semaphore
@vsync_end:
	; Pass control to kernal interrupt routine
	jmp	(orig_isr)		; Kernal acknowledges VSYNC interrupt
.endproc

.proc install_custom_interrupt_routine
	; Save kernal interrupt handler in orig_isr variable
	lda	IRQ_VECTOR+0
	sta	orig_isr+0
	lda	IRQ_VECTOR+1
	sta	orig_ISR+1
	sei					; Disable interrupts
	; Overwrite address of kernal interrupt routine with address
	; of custom interrupt routine
	lda	#<my_interrupt_routine
	sta	IRQ_VECTOR+0
	lda	#>my_interrupt_routine
	sta	IRQ_VECTOR+1
	cli					; Enable interrupts
	rts
.endproc

main:
	jsr	install_custom_interrupt_routine

loop:
	wai
	lda	semaphore		; If semaphore=1, VSYNC interrupt has happened
	bne	loop
	; VSYNC interrupt has happened, reset semaphore variable
	stz	semaphore
	; Do stuff
	...
	bra	loop
```
## C Language
## Prog8
<!-- For PDF formatting -->
<div class="page-break"></div>
