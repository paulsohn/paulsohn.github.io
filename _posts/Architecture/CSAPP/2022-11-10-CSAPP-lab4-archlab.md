---
layout: posts

title: "CSAPP - Lab 4. Architecture Lab"

categories: arch
tags:
  - CSAPP

date: 2022-11-12 23:20:00 +09:00
last_modified_at: 2022-11-12 23:20:00 +09:00

# published: true

---

# Arch Lab

A lab assignments using Y86-64 [ISA](https://en.wikipedia.org/wiki/Instruction_set_architecture), an educational-purpose simplified version of modern x86-64 architecture.

The lab consists of three parts:
- A. Write a Y86-64 assembly program to perform sum over linked list(both iterative and recursive), and copying memory.
- B. Modify sequential SEQ processor to support `iaddq` instruction. This is an intermediate step for C.
- C. Modify pipelined PIPE processor to support `iaddq` instruction, and use it to optimize `ncopy` function. The function `ncopy(src, dst, len)` copies `len` words of memory `src` to `dst` as well as returning number of positive elements.

## Part A

simple.

```
sum_list:
	irmovq $0,%rax		# val = 0
	andq %rdi,%rdi		# set CC
	je outside1
loop1:
	mrmovq (%rdi),%rsi  # rsi = ls->val
	addq %rsi,%rax		# val += rsi
	mrmovq 8(%rdi),%rdi	# ls = ls->next
	andq %rdi,%rdi		# set CC
	jne loop1
outside1:
	ret					# return val
```

```
rsum_list:
	andq %rdi,%rdi		# set CC
	jne else1
	irmovq $0,%rax
	ret                 # if(!ls) return 0;
else1:
	mrmovq (%rdi),%rcx  # val = ls->val;
	mrmovq 8(%rdi),%rdi
	pushq %rcx			# since we're doing recursion, local vars should be saved into the stack.
	call rsum_list		# %rax = rsum_list(ls->next)
	popq %rcx
	addq %rcx,%rax
	ret
```

```
copy_block:
    irmovq $8,%r9       # increment, const 8
    irmovq $1,%r10      # decrement, const 1
    irmovq $0,%rax
    andq %rdx,%rdx  # set CC
    je outside1
loop1:
    mrmovq (%rdi),%rcx  # val = *src
    addq %r9,%rdi       # src++
    rmmovq %rcx,(%rsi)  # *dest = val
    addq %r9,%rsi       # dest++
    xorq %rcx,%rax      # result ^= val
    subq %r10,%rdx      # len--
    andq %rdx,%rdx      # set CC
    jne loop1
outside1:
    ret
```

## Part B

`iaddq`, by its specification, needs constant word fetch(`valC`) and it should fed the adder as well as register B value(`valB`).
Others are same as `iirmovq`, which puts `valC` and zero to the adder.

```
# in seq/seq-full.hcl

bool instr_valid = icode in { INOP, (...), IIADDQ };
bool need_regids = icode in { (...), IIADDQ };
bool need_valC = icode in { (...), IIADDQ };

word srcB = [
	icode in { IOPQ, IIADDQ, IRMMOVQ, IMRMOVQ  } : rB; 
	(...)
];
word dstE = [
	icode in { IRRMOVQ } && Cnd : rB;
	icode in { IIRMOVQ, IOPQ, IIADDQ } : rB;
	(...)
];

word aluA = [
	(...)
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
	(...)
];
word aluB = [
	icode in { IRMMOVQ, IMRMOVQ, IOPQ, IIADDQ, (...) } : valB;
	(...)
];
bool set_cc = icode in { IOPQ, IIADDQ };
```

In the example code `y86-code/asumi.ys` for testing `iaddq` instruction, a loop check depends on CC after `iaddq`.

After feeding ALU, we're done - the result will be handled as same as other instructions.

## Part C

### Testing

Since I had to create many variations of PIPE and test them, I defined some macros in `pipe/Makefile` so that the build and compilation could be done with a single line.

```
test: clean psim
	(cd ../y86-code; make testpsim)
	(cd ../ptest; make SIM=../C_pipe/psim)

testi: clean psim
	(cd ../y86-code; make testpsim)
	(cd ../ptest; make SIM=../C_pipe/psim TFLAGS=-i)
```

`$ make test VERSION=<version>` for `iaddq`-unsupported versions, and `$ makei test VERSION=<version>` for supported ones.

### Implementing Immediate Addition

First implement `iaddq` in our pipelined processor also.
The modifications are similar to SEQ, except that every variable has stage prefixes(e.g. `d_srcB`)

### Implementing Other Branch Prediction Strategies

The default setting of branch prediction in PIPE is AT(always taken).
Depending on the access pattern, NT(never taken) or BTFNT(backward-taken forward-not taken) may perform better so I implemented both of them.
Here is how I modified PIPE architecture to apply BTFNT strategy. I omitted NT for it is simpler.

Note that this BTFNT implementation not yet supports `iaddq` - you may patch this into the above `iaddq` implementation.

```
# in pipe/pipe-btfnt.hcl
word f_pc = [
  # Mispredicted branch.
  M_icode == IJXX && !M_Cnd && M_valE < M_valA : M_valA; // mistakenly taken
  M_icode == IJXX && M_Cnd && M_valE >= M_valA : M_valE; // mistakenly not taken
  (...)
];
word f_predPC = [
  (...)
  f_icode == IJXX && f_valC < f_valP : f_valC; # BTFNT
	1 : f_valP; 
];

word aluA = [
	(...)
	E_icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ,
				IJXX } : E_valC; #//
	(...)
];
word aluB = [
	(...)
	E_icode in { IRRMOVQ, IIRMOVQ,
				IJXX } : 0; #//
	(...)
];

bool D_bubble = 
  # Mispredicted branch
  (E_icode == IJXX && ( (!e_Cnd && e_valE < E_valA)
						|| (e_Cnd && e_valE > E_valA) ) ) ||
  # Stalling at fetch wile ret passes through pipeline
  # but not condition for a load/use hazard
  (...)
bool E_bubble = 
  # Mispredicted branch
	(E_icode == IJXX && ( (!e_Cnd && e_valE < E_valA)
						|| (e_Cnd && e_valE > E_valA) ) ) ||
	# Conditions for a load/use hazard
	(...)
```

This time, `IJXX` also uses ALU - just for passing `valC` - as the handout material specifies:

> When some branches are predicted as not-taken,
> you need some way to get `valC` into pipeline register `M`,
> so that you can correct for a mispredicted branch.

And bubbling condition should be changed also.

---

Despite everything above, in this lab(and given datasets), AT outperformed slightly better than NT and BTFNT.

### Optimizing 'Array Copying and Counting Positive Elements' Function

I used `iaddq` to avoid redundancy for storing constants and `x8 -> x2` loop unrolling strategy to reduce step counts.

Loop unrolling is effective here, because in a 'fully-rolled' loop, every time we access the memory we encounter address increment -
* load `(%rdi)`
* increment `%rdi`
* load `(%rdi)`
* increment `%rdi`
* ... and so on.

however when we adopt loop unrolling, we can implement the same procedure with less steps:
* load `(%rdi)`
* load `8(%rdi)`
* increment `%rdi`
* ... and so on.

Thus, we get better performance when we apply larger unrolling - no problem for 'register spilling' since we are hardcoding the assembly and reuse registers once the values are stored.
The only reason we don't choose `x64` unrolling is the lab restriction of `ncopy.yo` less then 1000 bytes long.
The datapath for PIPE processor is trivial so multiple counter woudln't bring more efficiency.

For counting the number of positive elements, I copy-pasted this piece of lines everywhere when `%r8` (or other registers) is holding the loaded element.
```
  andq %r8, %r8
	jle Npos81
	iaddq $1, %rax
Npos81:
```

My final score for part C was `52.2/60.0`, with AT.
(BTFNT and NT gives `51.5` and `50.7`, )

### Using Conditional Moves?

In PIPE, the branch misprediction penalty is up to 2 bubbles - so I decided to make another version which minimizes the use of conditional jumps.
Instead of the counting-positive scheme above,

```
  rrmovq %rax, %rcx
	iaddq $1, %rcx
	andq %r8, %r8
	cmovg %rcx, %rax
```

The score however, was under `45`.
Although 'real' (x86-64) architectures have high misprediction penalty up to 19 clock cycles(according to CSAPP), this simple PIPE architecture has relatively low penalty of 2 bubbles and the average execution time of our jump-based count per element is 3.5, i.e. half of the penalty plus 2.5 - assuming the positive and non-positive elements in memory both appear half-half.
On contrary, our cmove-based count snippet requires constant 4-cycle per elements.

If I've been allowed to add another instructions, I would add hypothetical *conditional add* instruction so that I could rearrange count like this:

```
  andq %r8, %r8
  caddg $1, %rax
```
which costs constant 2-cycle per elements (and definitely I would get `60/60`!)
