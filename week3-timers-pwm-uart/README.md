# Week 3 — Timers, PWM, and UART

## What I Built
- Hardware timer interrupt blinking LED at exactly 1Hz
- PWM on PA5 fading LED brightness up and down
- Live data printed to PC over UART via PuTTY at 115200 baud

## Key Concepts

### Interrupts
Hardware signals the CPU only when needed — CPU is free otherwise.

Three rules that must never be broken:
1. **Keep ISRs short** — set a flag, return, do work in main()
2. **Use `volatile` on shared variables** — forces compiler to always read
   fresh from memory instead of caching the value in a register
3. **Never call HAL_Delay() inside an ISR** — HAL_Delay uses SysTick,
   itself an interrupt — causes deadlock

### The Flag Pattern
```c
volatile uint8_t flag = 0;

void TIM3_IRQHandler(void) {
    TIM3->SR &= ~(1 << 0);  // clear SR flag — ALWAYS first line
    flag = 1;                 // leave note for main loop
}

while (1) {
    if (flag) {
        flag = 0;
        do_work();  // heavy work done here safely
    }
}
```

`flag` is a sticky note from the ISR to main(). The ISR sets it to 1,
main sees it, clears it, and handles the event. Without `volatile`,
the compiler would cache flag=0 and the if() would never trigger.

### Timer Math
Formula: `f = clock / (PSC+1) / (ARR+1)`

84MHz timer clock, target 1Hz:
```c
TIM3->PSC = 8399;   // 84MHz / 8400 = 10,000 Hz tick
TIM3->ARR = 9999;   // 10,000 / 10,000 = 1 Hz
```

### PWM
Pin goes HIGH while counter < CCR, LOW otherwise.
Duty cycle = CCR / (ARR+1) × 100%

### UART
USART2 routes through ST-Link to USB — shows in PC serial monitor.
APB1 bit 17. Always 115200 baud, 8N1.

## Bugs I Hit

### Used TIM2_IRQHandler instead of TIM3_IRQHandler
ISR name must exactly match the timer. Wrong name = ISR never called.

### 16-bit PSC overflow — the big one
Set PSC = 83999 expecting 1Hz on a 84MHz clock.
TIM3 is a **16-bit timer** — PSC max is 65535.
83999 silently overflowed to 18463, giving ~4.5Hz (220ms ticks).
Verified with HAL_GetTick() timestamps in UART output.

Fix: keep both PSC and ARR under 65535.
Note: **TIM2 is 32-bit** (safe for large PSC). **TIM3 is 16-bit** (max 65535).

### RCC wrong bus
Used `RCC->AHB1ENR` (GPIO bus) instead of `RCC->APB1ENR` (timer bus).
Timers live on APB1 — always.

## APB1 Peripheral Bit Reference
```
bit 0  = TIM2
bit 1  = TIM3
bit 17 = USART2
bit 21 = I2C1
```
