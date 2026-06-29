# Week 1 — C for Microcontrollers

## What I Built
No hardware this week — pure C concepts that underpin everything in embedded development.
Covered memory layout, pointers, and bit manipulation.

## Key Concepts

### Memory Layout
The STM32 divides memory into distinct regions:
- `.text` — compiled code, stored in flash (read-only)
- `.data` — global variables with initial values, costs flash space
- `.bss` — uninitialized globals, zeroed at startup for free (no flash cost)
- `stack` — local variables, fixed size, grows downward — overflow is silent and deadly
- `heap` — dynamic allocation via malloc(), avoided in embedded due to fragmentation

The trickiest part was the difference between `.bss` and `.data`. Both are globals,
but `.data` costs flash space to store the initial value, while `.bss` is just zeroed
at startup with no flash overhead.

### Pointers
In embedded C, pointers are how you talk to hardware. Every peripheral register
is a fixed memory address — you write to it with a pointer:

```c
uint32_t *reg = (uint32_t *)0x40020018;  // GPIOA output register
*reg |= (1 << 5);                         // set pin 5 HIGH
```

### Bit Manipulation
The four operations used constantly in register programming:

```c
reg |= (1 << n);      // SET bit n
reg &= ~(1 << n);     // CLEAR bit n
reg ^= (1 << n);      // TOGGLE bit n
(reg >> n) & 1;       // CHECK bit n
```

`1 << 3` shifts the number 1 left by 3 positions, turning bit 3 on (result = 8).
Writing `1 << 5` is clearer than writing `32` — you can immediately see which
bit position is being targeted without converting in your head.

## What Tripped Me Up
- `.bss` vs `.data` — both globals, different flash cost
- The `~` in `&= ~(1 << n)` — tilde inverts all bits first, making a mask
  with 0 only at position n, clearing just that bit without touching others

## Quiz Score
5/5
