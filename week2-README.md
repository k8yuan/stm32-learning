# Week 2 — STM32 GPIO and LED Blink

## What I Built
First real hardware program — blinked the onboard LED (PA5/LD2) on the Nucleo-F446RE
using direct register writes, no HAL abstraction.

## Key Concepts

### The Clock Rule
Every peripheral starts powered off. Before touching any GPIO register,
enable its clock via RCC. Forgetting this is the number one cause of
"why does nothing work" — the registers exist but nothing responds.

```c
RCC->AHB1ENR |= (1 << 0);   // bit 0 = GPIOA clock
```

### GPIO Registers
Three registers control every pin:
- `MODER` — direction. 2 bits per pin: 00=input, 01=output, 10=alt fn, 11=analog
- `ODR` — drives pins HIGH or LOW
- `IDR` — reads current pin state

PA5 uses bits 11:10 in MODER because pin 5 × 2 = bit position 10.

```c
GPIOA->MODER &= ~(3 << 10);  // clear bits first — important
GPIOA->MODER |=  (1 << 10);  // 01 = output

GPIOA->ODR |=  (1 << 5);     // LED on
GPIOA->ODR &= ~(1 << 5);     // LED off
GPIOA->ODR ^=  (1 << 5);     // toggle
```

### Why Clear Before Set?
MODER uses 2 bits per pin. If you only OR without clearing first,
you might end up with 11 (analog mode) if one bit was already set.
Always clear the field first, then set it.

## What Tripped Me Up
- Forgetting `RCC->AHB1ENR` — nothing worked, no error
- Not clearing MODER bits before setting — caused wrong pin mode
