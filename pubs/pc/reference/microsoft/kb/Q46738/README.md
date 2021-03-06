---
layout: page
title: "Q46738: Mixed-Model Programming and long int Helper Library Routines"
permalink: /pubs/pc/reference/microsoft/kb/Q46738/
---

## Q46738: Mixed-Model Programming and long int Helper Library Routines

	Article: Q46738
	Version(s): 5.10
	Operating System: MS-DOS
	Flags: ENDUSER | SR# G890606-19866
	Last Modified: 25-JUL-1989
	
	Question:
	
	I notice that the Microsoft C Optimizing Compiler Version 5.10
	implicitly generates calls to "helper" routines in to handle long
	integer arithmetic for operations such as division, left-shift,
	right-shift, etc.
	
	I see a problem in situations such as the following:
	
	My program is compiled in small model, so all code and data pointers
	are near. However, I explicitly declare a far pointer to a long
	integer, and then I perform a left-shift on this long integer.
	
	Won't the library routines be expecting near pointers to data and,
	therefore, fail?
	
	Response:
	
	The compiler is aware of this mixed-model programming issue and
	generates appropriate calls to ensure that the helper routines don't
	make the wrong assumption as to near versus far pointers.
	
	To demonstrate this, consider the following test program:
	
	long lNear;                     /* this resides in DGROUP */
	long far lFar;                  /* this goes in a FAR_BSS segment */
	
	void main(void);
	
	void main(void)
	{
	    long lStack;                /* this goes in SS, part of DGROUP */
	
	    lNear = 30000;
	    lFar  = 15000;
	    lNear <<=4;                 /* force call to helper routines */
	    lFar <<= 4;
	    lStack = lNear + lFar;
	    lStack <<= 4;
	
	}
	
	The above comments point to the salient points of this test. There are
	three long variables, one in DGROUP, one in a far segment, and one on
	the stack. The compiler is asked to left-shift the near and far longs
	by 4 bits to force a call to the helper routines. Some of the compiled
	code generated by the above source is shown in the following:
	
	...
	
	EXTRN   __acrtused:ABS
	EXTRN   __chkstk:NEAR
	EXTRN   __aNNalshl:NEAR
	EXTRN   __aNFalshl:NEAR
	_BSS      SEGMENT
	COMM NEAR       _lNear: BYTE:    4
	COMM FAR        _lFar:  BYTE:    4
	
	...
	
	;|***     lNear <<=4;
	; Line 12
	        *** 000029      b0 04        mov     al,4
	        *** 00002b      50           push    ax
	        *** 00002c      b8 00 00     mov     ax,OFFSET DGROUP:_lNear
	        *** 00002f      50           push    ax
	        *** 000030      e8 00 00     call    __aNNalshl
	;|***     lFar <<= 4;
	; Line 13
	        *** 000033      b0 04        mov     al,4
	        *** 000035      50           push    ax
	        *** 000036      b8 00 00     mov     ax,OFFSET _lFar
	        *** 000039      ba 00 00     mov     dx,SEG _lFar
	        *** 00003c      52           push    dx
	        *** 00003d      50           push    ax
	        *** 00003e      e8 00 00     call    __aNFalshl
	
	Note that even though this program compiled in small model, it created
	a far segment (FAR_BSS), and it put lFar in it with the declaration
	"COMM FAR _lFar: Byte: 4". Note also the difference between how it
	left-shifted the lNear variable (__aNNalshl) versus the way it shifted
	the lFar variable (__aNFalshl). The routine __aNNalshl is for small
	model (near code pointer plus near data pointer), whereas __aNFalshl
	is compact model (near code pointer plus far data pointer); this makes
	complete sense for the mixed-model program.
	
	To summarize, the compiler knows about these near/far dependencies
	between source code and library routines, and puts in requests for the
	appropriate routines even if you're in a memory model that wouldn't
	normally use them (in this case, you wouldn't normally generate a call
	to __aNFalshl in small model).
