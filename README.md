# stm32h747

The STM32H747's register map, as sysl constants — ST's dual-core Cortex-M7/M4, the chip on the
STM32H747I-DISCO.

*An independent sysl package for the STM32H747. It is not affiliated with or endorsed by
STMicroelectronics.*

Base addresses, register offsets, field masks and shifts, and the names ST gives the values a field
takes. One module per peripheral, 124 of them (53 derived from another), 40,647 constants.

```
dependencies {
  stm32h747 { git = "github.com/sysl-lang/stm32h747", version = "0.1.1" }
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

Every `.sysl` file here but one comes from ST's own `STM32H747_CM7.svd`, corrected by
`patches/stm32h747.patch.hocon`, by way of [sysl-lang/svd](https://github.com/sysl-lang/svd) 0.2.0:

```
svd constants STM32H747_CM7.svd --patch patches/stm32h747.patch.hocon -o . -m sh.sysl.stm32h747
```

svd is a build tool, not a dependency: nothing here needs it to build or run.

| | |
|---|---|
| pack | `Keil.STM32H7xx_DFP` 4.1.3 — ST's CMSIS device family pack for the STM32H7 |
| downloaded from | <https://www.keil.com/pack/Keil.STM32H7xx_DFP.4.1.3.pack> (listed in <https://www.keil.com/pack/Keil.pidx>) |
| pack SHA-256 | `42a2358e0cb7344a9a54ee14c95eb17af20ccbe44bd2b0a08b24fefbfb67b8c8` |
| file | `CMSIS/SVD/STM32H747_CM7.svd`, SVD version 1.9 |
| SVD SHA-256 | `7b73dedce22eeec3ab9cd99ec8a102edb9ced5466f085e09344fd8fbb28b0ae2` |
| licence | Apache-2.0 (the file's own `SPDX-License-Identifier`, and the pack's `LICENSE-Apache-2.0`) |

A hand edit is lost the next time the chip is regenerated. A correction belongs in the patch file, the
generator, or a newer SVD.

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

## Corrections applied by `patches/stm32h747.patch.hocon`

ST's SVD 1.9 disagrees with RM0399 in places, and ST's own CMSIS header `stm32h747xx.h` sides with the
manual each time. Each is corrected by an entry in the patch file, whose `why` is quoted below; every
module a correction touched names the patch file, its SHA-256 and those reasons in its header, and
`tests.sysl` checks each corrected value against the manual.

| | the SVD says | corrected to, because |
|---|---|---|
| `RCC.APB3ENR.DSIEN` | absent | bit 4 — RM0399 RCC_APB3ENR: bit 4 DSIEN; stm32h747xx.h RCC_APB3ENR_DSIEN_Msk = 0x00000010 |
| `RCC.C1_APB3ENR.DSIEN` | absent | bit 4 — RM0399 RCC_C1_APB3ENR: bit 4 DSIEN, as RCC_APB3ENR; stm32h747xx.h uses RCC_APB3ENR_DSIEN for both |
| `RCC.APB3RSTR.DSIRST` | absent | bit 4 — RM0399 RCC_APB3RSTR: bit 4 DSIRST; stm32h747xx.h RCC_APB3RSTR_DSIRST_Msk = 0x00000010 |
| `RCC.APB3LPENR.DSILPEN` | absent | bit 4 — RM0399 RCC_APB3LPENR: bit 4 DSILPEN; stm32h747xx.h RCC_APB3LPENR_DSILPEN_Msk = 0x00000010 |
| `RCC.C1_APB3LPENR.DSILPEN` | absent | bit 4 — RM0399 RCC_C1_APB3LPENR: bit 4 DSILPEN, as RCC_APB3LPENR; stm32h747xx.h uses RCC_APB3LPENR_DSILPEN for both |
| EXTI's CPU2 block, `C2IMR1` … `C2PR3` | absent | at `0xC0`–`0xE8` — RM0399 EXTI register map: EXTI_C2IMR1, C2EMR1, C2PR1 … C2PR3 at 0xC0–0xE8, laid out as CPU1's at 0x80–0xA8; stm32h747xx.h EXTI_TypeDef has C2IMR1 at 0xC0 … C2PR3 at 0xE8 |
| EXTI's CPU1 block at `0x80` | named `CPUIMR1` … `CPUPR3` | `C1IMR1` … `C1PR3` — RM0399 names the block at 0x80 EXTI_C1IMR1 … EXTI_C1PR3 (CPU1's), beside CPU2's EXTI_C2… at 0xC0 |
| `SAI_ACR1.MCKDIV`, `SAI_BCR1.MCKDIV` | 4 bits, `0x00F00000` | 6 bits, `0x03F00000` — RM0399 SAI_xCR1: MCKDIV[5:0] is bits 25:20; stm32h747xx.h SAI_xCR1_MCKDIV_Msk = 0x03F00000 |
| `LTDC_SSCR.HSW` | 10 bits, `0x03FF0000` | 12 bits, `0x0FFF0000` — RM0399 LTDC_SSCR: HSW[11:0] is bits 27:16; stm32h747xx.h LTDC_SSCR_HSW_Msk = 0x0FFF0000 |
| `DMA2D_CR.MODE` | 2 bits, `0x00030000` | 3 bits, `0x00070000` — RM0399 DMA2D_CR: MODE[2:0] is bits 18:16; stm32h747xx.h DMA2D_CR_MODE_Msk = 0x00070000 |

The SAI correction is made on `SAI4`, from which the SVD derives `SAI1`–`SAI3`, so it reaches all four.

**One disagreement is deliberately not patched.** `HSEM`'s `COREID` is four bits, 11:8, in the SVD,
and eight, `0x0000FF00`, in `stm32h747xx.h`. RM0399 sides with the SVD: `HSEM_Rx`, `HSEM_RLRx` and
`HSEM_CR` each give `COREID[3:0]` at bits 11:8, with bit 12 reserved. The header is the one that is
wider than the manual, so the generated value stays, and `tests.sysl` checks it.

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
`sysl-lang/svd`, with the corrections listed above; nothing else in it was changed.
