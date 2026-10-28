# stm32h747

The STM32H747's register map, as sysl constants — ST's dual-core Cortex-M7/M4, the chip on the
STM32H747I-DISCO.

*An independent sysl package for the STM32H747. It is not affiliated with or endorsed by
STMicroelectronics.*

Base addresses, register offsets, field masks and shifts, and the names ST gives the values a field
takes. One module per peripheral, 124 of them (53 derived from another), 40,228 constants.

```
dependencies {
  stm32h747 { git = "github.com/sysl-lang/stm32h747", version = "0.1.0" }
}
```

```sysl
import sh.sysl.stm32h747.rcc
import sh.sysl.stm32h747.dsihost

// Turn on the DSI regulator, then wait for it to say it is ready.
write(dsihost.BASE + dsihost.DSI_WRPCR_OFFSET, dsihost.DSI_WRPCR_REGEN_MASK)
while read(dsihost.BASE + dsihost.DSI_WISR_OFFSET) & dsihost.DSI_WISR_RRS_MASK == 0 do ()
```

## It is generated, and must not be edited

Every `.sysl` file here but one comes from ST's own `STM32H747_CM7.svd` by way of
[sysl-lang/svd](https://github.com/sysl-lang/svd) 0.1.1:

```
svd constants STM32H747_CM7.svd -o . -m sh.sysl.stm32h747
```

| | |
|---|---|
| pack | `Keil.STM32H7xx_DFP` 4.1.3 — ST's CMSIS device family pack for the STM32H7 |
| downloaded from | <https://www.keil.com/pack/Keil.STM32H7xx_DFP.4.1.3.pack> (listed in <https://www.keil.com/pack/Keil.pidx>) |
| pack SHA-256 | `42a2358e0cb7344a9a54ee14c95eb17af20ccbe44bd2b0a08b24fefbfb67b8c8` |
| file | `CMSIS/SVD/STM32H747_CM7.svd`, SVD version 1.9 |
| SVD SHA-256 | `7b73dedce22eeec3ab9cd99ec8a102edb9ced5466f085e09344fd8fbb28b0ae2` |
| licence | Apache-2.0 (the file's own `SPDX-License-Identifier`, and the pack's `LICENSE-Apache-2.0`) |

A hand edit is lost the next time the chip is regenerated. A correction belongs in the generator, or
in a newer SVD.

**`sh/sysl/stm32h747/tests.sysl` is the one exception** — the only hand-written file, and the guard on
regeneration. It checks addresses and field positions from the peripherals benchtop models — RCC,
GPIO, LTDC, DSI, DMA2D, SAI, SDMMC, ADC, DAC, TIM, USART, HSEM, EXTI — against ST's reference manual
**RM0399** rather than against the SVD, so that a generator change or a newer SVD has something
independent to fail against. `sysl test .` runs it.

## One package for both cores

The pack carries an SVD per core, `STM32H747_CM7.svd` and `STM32H747_CM4.svd`. Their register maps are
the same: the generator emits byte-identical output from each. What differs is all outside the
register map, and none of it reaches this package:

- the device and `<cpu>` names (`STM32H747_CM7`/`CM7` against `STM32H747_CM4`/`CM4`);
- each core's own interrupt numbers — `HSEM0` (125) against `HSEM1` (126); `WWDG1` at 0 for the M7
  where the M4 has `WWDG2` at 0; the other core's send-event interrupt (`cm4_sev_it` 65 against
  `cm7_sev_it` 64); and "CPU1" against "CPU2" in two descriptions;
- where `DBGMCU` sits in the file.

The generator does not emit interrupts, so a program on either core uses this package as it is.

## Where ST's SVD is wrong

ST's SVD 1.9 disagrees with RM0399 in places, and ST's own CMSIS header `stm32h747xx.h` sides with the
manual each time. They were found while writing the tests, and the generated files are left as the SVD says
until ST ships a corrected one:

| | the SVD says | RM0399 says |
|---|---|---|
| `RCC_APB3ENR.DSIEN`, `RCC_APB3RSTR.DSIRST` | absent | bit 4 |
| `SAI_xCR1.MCKDIV` | 4 bits, `0x00F00000` | 6 bits, `0x03F00000` |
| `LTDC_SSCR.HSW` | 10 bits, `0x03FF0000` | 12 bits, `0x0FFF0000` |
| `DMA2D_CR.MODE` | 2 bits, `0x00030000` | 3 bits, `0x00070000` |
| EXTI's CPU2 block (`C2IMR1`… at `0xC0`) | absent; the CPU1 block at `0x80` is named `CPUIMR1`… | both |

The three mask widths are ignored tests in `tests.sysl`, written with the manual's values. A program
that needs one of these writes the manual's value itself, and should say why.

## What it does not do

**It knows nothing about a board.** No pin assignments, no "the LCD is on DSI lane 0". A board's
wiring is a board's business.

**It knows nothing about drivers.** It knows where `SAI1` is and which bits of `SAI_ACR1` mean what,
not how to play a sound.

**It does not cover the Cortex-M cores themselves.** NVIC, SCB, SysTick and the MPU are not in ST's
SVD; they are the same on every Cortex-M7 and belong to a package for the core.

## Capabilities

None. It declares constants — no function, no storage, no C, no allocator, no operating system — so it
is usable from a freestanding target (`sysl build-lib . --target thumb-freestanding`), which is the
only kind that wants it.

## Licence

ISC for the sysl in this repository (`LICENSE`). The register names, addresses and descriptions are
STMicroelectronics', from `STM32H747_CM7.svd`, Copyright (c) 2024 STMicroelectronics, under the Apache
License 2.0 (`LICENSE-Apache-2.0`). The generated files are that SVD converted to sysl declarations by
`sysl-lang/svd`; nothing else in it was changed.
