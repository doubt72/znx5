# ZNX5 Processor

This is perhaps the second-most pointless thing I've ever done.

## Basic Properties

* Load/store architecture
* 32-bit CPU
* 32-bit address space (in 32-bit words; 4GW, 16GB byte equivalent)
* Single or multi-core
* 3-level pipeline: fetch, decode, execute
* 32-bit instructions only, all instructions take single clock to
  decode/execute (fetch varies depending on cache)
* Dual mode operation (user, supervisor/interrupt)
* 32 32-bit integer registers: %i0-%if (passed/indirect) and %d0-%df (data)
* 16 64-bit floating point registers: %f0-%ff
* 4 256-bit SIMD registers: %m0-%m3
* 256-deep interrupt buffer (additional interrupts dropped)
* No data need be stored on interrupt; supervisor has own %i0-%if,
  %d0-%de (data) registers (%df/%pc shared, %if/%lr contains last %pc on
  interrupt)
* %de = %sp = user stack pointer (USP) or supervisor stack pointer (SSP),
  depending on mode
* %df = %pc (program counter)
* %if = %lr (link register), no data stored/retrieved on jsr/rts, %pc
  and %lr swapped
* Status register %st: Z, N, V, C, FZ, FN, FINF, FNNF, FNAN, FDN, S
  [zero, negative, overflow/divide by zero, carry, float zero, float
  negative, infinity, negative infinity, not a number, denormalized,
  supervisor mode]; separate status registers for each mode
  (user/supervisor)
* %d0-%d8 stored to SSP on call, popped off SSP on ret, %st cleared
* MMU on-die 16kW/256kW(L3) tagged (ASN) TLB using 16kW pages (pre-cache
  translation)
* %asn = Address Space Number (ASN) register can be set and read in
  supervisor mode; asn is enforced in user mode, but ignored during
  supervisor mode.

## Instruction Set

### 3 Operand Integer Instructions [15/16]

```
  op   x     y                 z
1 2345 6789A BCDEF0123456789AB CDEF0
1 xxxx xxxxx xyyyyyyyyyyyzzzzz xxxxx

op = { 0=and, ... }
x, z = %r
y = zzzzz = %r                          if x = 0 ( yyyyyyyyyyy = ignored )
    yyyyyyyyyyyzzzzz = -2^15,2^15-1     if x = 1
```

X and Z are registers, Y is either a register or a literal number
ranging from -2^15 to 2^15-1.  Result is stored in Z, which can equal
X or Y.

```
operation          flags    description

and X, Y, Z        Z,N      ;bitwise and (1100 x 1010 = 1000)
or X, Y, Z         Z,N      ;bitwise or (1100 x 1010 = 1110)
xor X, Y, Z        Z,N      ;exclusive or (1100 x 1010 = 0110)
add X, Y, Z        Z,N,V,C  ;addition (overflow > 2^9, carry > 2^10)
sub X, Y, Z        Z,N,V,C  ;subtraction (X - Y => Z)
addc X, Y, Z       Z,N,V,C  ;addition with carry (X + Y + C -> Z)
subc X, Y, Z       Z,N,V,C  ;subtraction with carry (X - Y - (1-C) -> Z)
mullo X, Y, Z      Z,N      ;multiplication, stores low
mulhi X, Y, Z      Z,N      ;multiplication, stores high
mulslo X, Y, Z     Z,N      ;signed multiplication, stores low
mulshi X, Y, Z     Z,N      ;signed multiplication, stores high
div X, Y, Z        Z,N,V    ;division (X / Y => Z, V,Z=0 if divide by zero)
divs X, Y, Z       Z,N,V    ;signed division (V,Z=0 if divide by zero)
mod X, Y, Z        Z,N,V    ;modulo (X % Y => Z, V,Z=0 if divide by zero)
mods X, Y, Z       Z,N,V    ;signed modulo (V,Z=0 if divide by zero)
```

### 2 Operand Integer Instructions [11/16]

```
   op   x     y      fill
12 3456 789AB CDEF01 23456789ABCDEF0
01 xxxx xxxxx xyyyyy xxxxxxxxxxxxxxx

op = { 0=ld, ... }
x = %r
y = yyyyy = %r          if x = 0
    yyyyy = 0,31        if x = 1
fill = ignored
```

X is a register, Y is either a register or a literal number ranging
from 0,31 (for shifts or rotations).  V is set if any positive bits
overflow, C is the final bit.  X is modified in place for shifts or
rotations.  Y can equal X (though for cp that's a nop)

```
operation      flags    description

ld X, Y        Z,N      ;load register X with value at address in register Y
st X, Y                 ;store register X value at address in register Y
cp X, Y                 ;copy register X value to register Y
cmp X, Y       Z,N      ;compare (Z if equal, N if Y > X)
bit X, Y       Z,N      ;bitwise test (same flags as and)
asl X, Y       Z,N,V,C  ;arithmetic shift left (0 fill, Y bits)
asr X, Y       Z,N,V,C  ;arithmetic shift right (MSB fill, Y bits)
shl X, Y       Z,N,V,C  ;logical shift left (0 fill, Y bits)
shr X, Y       Z,N,V,C  ;logical shift right (0 fill, Y bits)
rol X, Y       Z,N,V,C  ;rotate left (Y bits)
ror X, Y       Z,N,V,C  ;rotate right (Y bits)
```

### 1 Operand Integer/Control Instructions [38/64]

```
    op     x
123 456789 ABCDEF0123456789ABCDEF0
001 xxxxxx xyyyyyyyyyyyyyyzzzzzzzz

op = { 00=push, ... }
x = zzzzz = %r                                  if x = 0 ( yyy... = ignored)
    yyyyyyyyyyyyyyyyyzzzzz = -2^21,2^21-1       if x = 1
```

X is a register, or for jumps/branches can be a literal ranging from
-2^22,2^22-1, in which case the jump/branch is relative.  For calls,
%d0-%d7 and %st are stored on the supervisor stack (SSP) and zeroed
out.

```
operation      flags    description

push X                  ;store register on user stack
pop X          Z,N      ;pop register off user stack
nor X          Z,N      ;inverts bit value
neg X          Z,N,V    ;negate (2's complement)
clr X                   ;clears register (all 0's)
set X                   ;sets register (all 1's)
inc X          Z,N,V    ;increment value
dec X          Z,N,V    ;decrement value
beq X                   ;branch when zero set (to register address)
bne X                   ;branch when zero clear
bgt X                   ;branch when zero clear and negative set (>)
blt X                   ;branch when zero clear and negative clear (<)
bge X                   ;branch when zero set or negative set (>=)
bpl X                   ;branch when negative clear
bmi X                   ;branch when negative set
bvc X                   ;branch when overflow clear
bvs X                   ;branch when overflow set
bcc X                   ;branch when carry clear
bcs X                   ;branch when carry set
bfeq X                  ;branch when float zero set
bfne X                  ;branch when float zero clear
bfgt X                  ;branch when float zero clear and negative set (>)
bflt X                  ;branch when float zero clear and negative clear (<)
bfge X                  ;branch when float zero set or negative set (>=)
bfpl X                  ;branch when float negative clear
bfmi X                  ;branch when float negative set
bfinf X                 ;branch when infinity set
bfninf X                ;branch when infinity not set
bfnnf X                 ;branch when negative infinity set
bfnnnf X                ;branch when negative infinity not set
bfnan X                 ;branch when not a number set
bfnnan X                ;branch when not a number clear
bfdn X                  ;branch when denormalized set
bfndn X                 ;branch when denormalized clear
jmp X                   ;jump to register address
call X                  ;call function at register address

ble X                   ;alias for bpl, not a distinct instruction (<=)
bfle X                  ;alias for bfpl, not a distinct instruction (<=)

supervisor only:

ld %asn, X              ;set address space number
st %asn, X              ;load address space number
```

### 3 Operand Floating Point Instructions [5/8]

```
      op  f    g    h    fill
12345 678 9ABC DEF0 1234 56789ABCDEF0
00011 xxx xxxx xxxx xxxx xxxxxxxxxxxx

op = { 00=fadd, ... }
f, g, h = %f
fill = ignored
```

F, G, and H are floating point registers.  Result is stored in H.

```
operation      flags    description

fadd F, G, H   [all F]  ;floating point addition
fsub F, G, H   [all F]  ;floating point subtraction
fmul F, G, H   [all F]  ;floating point multiplication
fdiv F, G, H   [all F]  ;floating point division
fatan F, G, H  [all F]  ;partial arctangent
```

### 2 Operand Floating Point/Mixed Instructions [11/16]

```
       op   x/f   y/g   fill
123456 789A BCDEF 01234 56789ABCDEF0
000101 xxxx xyyyy xyyyy xxxxxxxxxxxx

op = { 00=ld, ... }
x, y = xyyyy = %r       if opcode uses mixed
f, g = yyyy = %f        if opcode uses floating point (y = ignored)
fill = ignored
```

X is a regular 32-bit integer register, F and G are floating point
registers. The result is stored in whichever register comes second (if
relevant).  Note that floating point values are 64-bit, so get stored
in/loaded from two words in memory starting at the specified address.

```
operation      flags    description

ld X, G        [all F]     ;load floating point register G with value at address X
st X, G        [all F]     ;store floating point register G at address X
cp F, G        [all F]     ;copy register F value into register G
fitof X, G     FZ,FN       ;convert X to float and store in G
fftoi F, Y     Z,N,V       ;convert F to integer and store in Y
fsqrt F, G     [all F]     ;square root
fsin F, G      [all F]     ;sin
fcos F, G      [all F]     ;cos
flog F, G      [all F]     ;log2 of F+1
fexp F, G      [all F]     ;(2^F) - 1
fcmp F, G      FZ,FN,FNAN  ;compare (FZ if =, FN if G > F, FNAN if either NaN)
```

### 1 Operand Floating Point Instructions [7/8]

```
        op  f    fill
1234567 89A BCDE F0123456789ABCDEF0
0001001 xxx xxxx xxxxxxxxxxxxxxxxxx

op = { 00=clr, ... }
f = %f
fill = ignored
```

F is a floating point register.  The push and pop instructions will
write/read two words into/from the user stack.

```
operation      flags                description

push F                              ;push F onto the user stack
pop F          [all F]              ;pop F onto the user stack
fneg F         [all F]              ;invert sign of F
fabs F         FZ, FINF, FNAN, FDN  ;absolute value
clr F          FZ                   ;set F to 0.0
fpi F                               ;load F with value of PI
fe F                                ;load F with value of E
```

### 3 Operand SIMD Instructions [20/32]

```
          op    m  n  p  fill
123456789 ABCDE F0 12 34 56789ABCDEF0
000100010 xxxxx xx xx xx xxxxxxxxxxxx

op = { 00=pand, ... }
m, n, p = %m
fill = ignored
```

M, N, and P are 256-bit SIMD registers containing either 8 32-bit
integer values or 4 64-bit floating point values.  Result is stored in
P.

```
operation

pand M, N, P       ;bitwise 8x8 and
por M, N, P        ;bitwise 8x8 or
pxor M, N, P       ;bitwise 8x8 xor
padd M, N, P       ;integer 8x8 addition (no carry)
psub M, N, P       ;integer 8x8 subtraction (no carry)
paddc M, N, P      ;integer 8x8 addition (with immediate carry, lo -> hi)
psubc M, N, P      ;integer 8x8 subtraction (with immediate carry, hi -> lo)
pmullo M, N, P     ;integer 8x8 multiplication (store low)
pmulhi M, N, P     ;integer 8x8 multiplication (store high)
pmulslo M, N, P    ;integer 8x8 signed multiplication (store low)
pmulshi M, N, P    ;integer 8x8 signed multiplication (store high)
pdiv M, N, P       ;integer 8x8 division
pdivs M, N, P      ;integer 8x8 signed division
pmod M, N, P       ;integer 8x8 modulo
pmods M, N, P      ;integer 8x8 signed modulo

pfadd M, N, P      ;floating point 4x4 addition
pfsub M, N, P      ;floating point 4x4 subtraction
pfmul M, N, P      ;floating point 4x4 multiplication
pfdiv M, N, P      ;floating point 4x4 division
pfatan M, N, P     ;floating point 4x4 partial arctangent
```


### 2 Operand SIMD/Mixed Instructions [26/32]

```
          op    x/f/m y/g/n/lit fill
123456789 ABCDE F0123 456789ABC DEF0
000100010 xxxxx xyyzz uvvvxyyzz xxxx

op = { 00=nop, ... }
x, y = xyyzz = %r          if opcode uses integer and u = 0 (vvv = ignored)
f, g = yyzz = %f           if opcode uses float (uvvvx = ignored)
m, g = zz = %m             if opcode uses SIMD (uvvvxyy = ignored)
       vvvxyyzz = 0,255    if u = 1 (shift/rotate operations only)
fill = ignored
```

X and Y are 32-bit integer registers, F and G are 64-bit floating
point registers, M and N are 256-bit SIMD registers.  The result is
stored in whichever register comes second (if relevant).

```
ld X, N          ;load SIMD register N with 8 words starting at X
st X, N          ;store SIMD register N at 8 words starting with X
cp M, N          ;copy SIMD register M into register N

cp X, N          ;copy value in integer register X into N (8 copies)
cp F, N          ;copy value in floating point register F into N (4 copies)

pasl M, Y        ;8x arithmetic shift left (0 fill, Y bits each)
pasr M, Y        ;8x arithmetic shift right (MSB fill, Y bits each)
pshl M, Y        ;8x logical shift left (0 fill, Y bits each)
pshr M, Y        ;8x logical shift right (0 fill, Y bits each)
prol M, Y        ;8x rotate left (Y bits each)
pror M, Y        ;8x rotate right (Y bits each)
plrol M, Y       ;long rotate left (Y bits total)
plror M, Y       ;long rotate right (Y bits total)

pfihitof M, N    ;4x convert M high words to float and store in N
pfilotof M, N    ;4x convert M low words to float and store in N
pfftoihi M, N    ;4x convert M float to integer and store in N high words
pfftoilo M, N    ;4x convert M float to integer and store in N low words

pfsqrt M, N      ;4x square root
pfsin M, N       ;4x sin
pfcos M, N       ;4x cos
pflog M, N       ;4x log2 of word+1
pfexp M, N       ;4x (2^word) - 1

pmax M, Y        ;maximum of 32-bit integer words in M, store in Y
pmin M, Y        ;minimum of 32-bit integer words in M, store in Y

pfmax M, G       ;maximum of 64-bit float double words in M, store in G
pfmin M, G       ;minimum of 64-bit float double words in M, store in G
```

### 1 Operand SIMD Instructions [9/16]

```
          op   m
123456789 ABCD EF0123456789ABCDEF0
000100010 xxxx xxxxxxxxxxxxxxxxxxx

op = { 00=nop, ... }
m = xx = %m
```

```
ppush M    ;store 8 32-bit words onto user stack
ppop M     ;pop 8 32-bit words from user stack and load in M
pnor M     ;invert bit value
pneg M     ;8x negate (2's complement)
pclr M     ;clear register (all 0's)
pset M     ;set register (all 0's)

pfneg M    ;4x invert sign (floating point)
pfabs M    ;4x absolute value
pfclr M    ;4x set value to 0.0
```

### 0 Operand Instructions [20/32]

```
     op    n        fill
1234 56789 ABCDEF01 23456789ABCDEF0
0000 xxxxx xxxxxxxx xxxxxxxxxxxxxxx

op = { 00=nop, ... }
n = x = 0,255
fill = ignored
```

Interrupts will store %pc in %lr, so the rti instruction will copy %lr
back to %pc.  The ret instruction will pop %d0-%d7 and %st.  The S bit
is set during interrupts and reset during rti instructions and cannot
be set or cleared.  The pushu instruction doesn't push %pc to the
stack, instead it copies it to %lr.  The popu instruction doesn't do
the opposite, that requires a user instruction.

```
operation      flags    description

nop                     ;no operation
pushs                   ;store status on stack
pops           [all]    ;pop status off stack
clrz           Z        ;clear zero bit
clrn           N        ;clear negative bit
clrc           C        ;clear carry bit
clrv           V        ;clear overflow bit
setz           Z        ;set zero bit
setn           N        ;set negative bit
setc           C        ;set carry bit
setv           V        ;set overflow bit
jsr                     ;jump to subroutine, swaps %pc and %lr
rts                     ;return from subroutine, loads %lr in %pc
ret                     ;return from call, pop program stack
int N          S        ;trigger interrupt N, copies %i0-%i7 to supervisor
stop                    ;wait until next interrupt

supervisor only:

pushu                   ;push user registers/status to supervisor stack
popu                    ;pop user registers/status to supervisor stack
user           S        ;jump to %lr, switch to user mode
rti                     ;return from interrupt
```

## Modules

### Processor

All cache is L1/L2/L3 2-way associative write through cache in
256-word blocks. L1 per core, L2/L3 shared among all cores. L1 = 1
cycle latency, L2 = 4 cycles, L3 = 16 cycles.

* ZN15C4 = 1 Core, 16kW/256kW/4MW cache
* ZN15C8 = 1 Core, 32kW/512kW/8MW cache
* ZN25C8 = 2 Core, 16kW/512kW/8MW cache
* ZN45C8 = 4 Core, 8kW/512kW/8MW cache
* ZN45C16 = 4 Core, 16kW/1MW/16MW cache
* ZN85C16 = 8 Core, 8kW/1MW/16MW cache

### Memory

* MM64M = 64MW/256kB Memory Module
* MM128M = 128MW/512kB Memory Module
* MM256M = 256MW/1GB Memory Module

Max 4MM/bus.  200 cycle latency.

### Attached Storage

* SSSM10G = 2.5GW/10GB Solid State Storage Module
* SSSM20G = 5GW/20GB Solid State Storage Module
* SSSM50G = 12.5GW/50GB Solid State Storage Module

1000 cycle latency.

### Other

* Keyboard
* Monitor
