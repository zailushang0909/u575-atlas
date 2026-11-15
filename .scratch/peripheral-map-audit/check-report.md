# STM32U575 外设关系图 · 硬件关系核对报告

对象：`stm32u575-peripheral-map.html`（91 个节点、约 600 条关系边）
日期：2026-11-15

## 核对依据（本地一手资料）

| 资料 | 用途 |
|---|---|
| RM0456 Rev 5（英文版 PDF） | 互连表、总线矩阵、EXTI、触发源 |
| DS13737 Rev 10（stm32u575ri.pdf） | 特性表、引脚/AF 表、封装引脚范围 |
| `STM32U575.svd`（CubeMX DFP 1.2.0） | 外设基地址、RCC 使能位、寄存器字段 |
| `stm32u575xx.h`（CubeMX FW_U5 V1.9.0） | IRQn_Type 实装中断号 |
| `stm32u5xx_ll_tim.h` / `ll_adc.h` | ETRSEL、EXTSEL 等编码来源 |

## 结论

- IRQn 号：**逐条比对后全部正确**（含共享中断 DCMI_PSSI=119、GTZC=8、COMP=72）。
- 时钟使能位：91 个节点逐条比对 SVD 的 RCC_*ENR 字段，**除 BKPSRAM / CRS / ICACHE 外全部正确**。
- 触发编码（ADC1 规则/注入、ADC4、DAC TSEL）：**除注入组的 7 处编码外全部正确**。
- ITR 表、GPDMA/LPDMA REQSEL：**全部正确**。
- 结构性问题：**总线矩阵型号写错（AXI）**，主/从清单有误。

---

## 一、已改正的错误

### 1. 总线矩阵写成 AXI 64-bit（结构性错误）
- 原：`总线矩阵 · AXI 64-bit + AHB/APB 桥`
- 改：`32-bit 多层 AHB + AHB/APB 桥`
- 依据：RM0456 §2.1「32-bit multilayer AHB bus matrix」、DS13737 §3.15「A 32-bit multi-AHB bus matrix」；全文检索 RM0456 无 `AXI` 字样（U5 系列不是 AXI 架构）。

### 2. 总线矩阵主/从清单错误
- 原：「主：CM33 C-AHB/S-AHB · GPDMA1(×2) · LPDMA1 · SDMMC · OCTOSPI · FMC」
- 问题：
  - LPDMA1 不是主矩阵主口，它是 SRD 域内的主口（RM §2.1.13：SRD 有 2 个主 = 主 AHB 矩阵 + LPDMA1）。
  - OCTOSPI1/2、FMC 是**从口**（被访问的存储器接口），不是主设备。
  - 漏了 DMA2D（U575 主口：CM33 三总线、GPDMA1×2、SDMMC1、SDMMC2、DMA2D）。
- 改：主 = CM33（C-AHB 快/慢 + S-AHB）· GPDMA1(×2) · SDMMC1/2 · DMA2D；从 = FLASH · SRAM1-3 · AHB1（含 BKPSRAM、APB1/2 桥）· AHB2 · OCTOSPI1/2 · FMC · SRD(AHB3+SRAM4+APB3)。
- 依据：RM0456 Table 1 / Table 2 / §2.1.13。

### 3. OCTOSPI1、OCTOSPI2、FMC、DCMI、PSSI 被标成总线主设备
- 原：这 5 个节点带 `"1"`（MASTER 标记），被画进「总线主设备」列。
- 改：去掉 MASTER 标记，改由 AHB2 分组呈现，说明文字加「（AHB 从口）」。
- 依据：RM0456 Table 1 主口清单里没有它们；DCMI/PSSI 通过 GPDMA 请求搬运数据（REQSEL 86），自身没有主口。

### 4. NVIC 中断数
- 原：`125 条外部中断`
- 改：`IRQ 0–125，实装 121 条`
- 依据：`stm32u575xx.h` IRQn_Type 共 121 个外部中断号，缺号 28、93、97、108、109。

### 5. BKPSRAM 的总线与时钟
- 原：总线 `CORE`、时钟 `PWR 控制`
- 改：总线 `AHB1`、时钟 `AHB1ENR[28]`，说明补「另需 PWR DBP」
- 依据：SVD `RCC_AHB1ENR[28] = BKPSRAMEN`；RM0456 Table 2 把「2-Kbyte BKPSRAM」列在 AHB1 从设备。

### 6. GPIOI 引脚数
- 原：`16 引脚`
- 改：`8 引脚（PI0–PI7）`
- 依据：DS13737 框图与 UFBGA169 ballout 只有 PI0–PI7。

### 7. 数据手册表号引用错误
- 原：`DS13737 Table 15/16`
- 改：`DS13737 Table 27/28`（Table 27 = AF0–AF7，Table 28 = AF8–AF15；引脚定义是 Table 26）
- 出现位置：PINMUX 节点说明、AF 边标签。

### 8. GPIO AF 寄存器名
- 原：`GPIOx_AFR[1]/[2]`
- 改：`GPIOx_AFRL / GPIOx_AFRH`

### 9. ADC1 注入组触发编码（7 处虚构编码）
ADC12 注入通道只有 `adc_jext_trg0…10`（RM0456 Table 304），且表中**没有** TIM6、TIM15、LPTIM1/2/3/4。

| 位置 | 原写法 | 改为 |
|---|---|---|
| TIM3 → ADC1 | 注入 CC4=4 / CC3=11 / TRGO=12 / CC1=13 | 注入 CC4=4 |
| TIM6 → ADC1 | 规则 TRGO=13 · 注入 TRGO=14 | 规则 TRGO=13（无注入通路） |
| TIM15 → ADC1 | 规则 TRGO=14 · 注入 TRGO=15 | 规则 TRGO=14（无注入通路） |
| LPTIM1 → ADC1 | 规则 CH1=18 · 注入 CH2=18 | 规则 CH1=18 |
| LPTIM2 → ADC1 | 规则 CH1=19 · 注入 CH2=19 | 规则 CH1=19 |
| LPTIM3 → ADC1 | 规则 CH1=20 · 注入 CH1=20 | 规则 CH1=20 |
| LPTIM4 → ADC1 | 规则 OUT=21 · 注入 OUT=21 | 规则 OUT=21 |

TRIG_MATRIX 同 7 行同步改为「—」。
依据：RM0456 Table 303（规则）、Table 304（注入）。

### 10. 比较器消隐（COMP BLANKSEL）两条边无依据，已删除；一条标注不精确，已改
- 删除 `TIM1 → COMP2`：COMP2_BLANKSEL 只有 tim3_oc4 / tim8_oc5 / tim15_oc1，无 TIM1。
- 删除 `TIM8 → COMP1`：COMP1_BLANKSEL 只有 tim1_oc5 / tim2_oc3 / tim3_oc3，无 TIM8。
- `TIM1 → COMP1` 标签由 `OCx` 改为 `tim1_oc5`。
- 依据：RM0456 Table 354 / Table 355。

### 11. VREFBUF → COMP1 边无依据，已删除
- 原：`VREFBUF → COMP1 比较参考 · COMPx_CSR.INSEL = VREFINT`
- 比较器的参考是**内部 VREFINT**（经 1/4、1/2、3/4 分压，`COMPx_CSR.INMSEL`），不是 VREFBUF 的输出；VREFBUF 驱动的是 VREF+ 引脚（供 ADC/DAC）。RM 无 VREFBUF→COMP 互连。

### 12. DAC → COMP 输入写法
- 原：`DAC1_OUT1 → COMP1`、`DAC1_OUT2 → COMP2`，依据 `COMPx_CSR.INSEL`
- 改：`DAC1_OUT1/OUT2 → COMP1_INM`、`… → COMP2_INM`，依据 `COMPx_CSR.INMSEL = 0100/0101`
- 依据：RM0456 Table 351 / Table 353：INMSEL 0100 = DAC channel1，0101 = DAC channel2，**COMP1/COMP2 都可选任一路**，不是一对一。

### 13. OPAMP → ADC 的说明
- 原：`经引脚进入 ADC 通道 ⚠ / 无内部直连`
- 改：`OPAMP1_VOUT(PA3) → ADC1_IN8`、`OPAMP2_VOUT(PB0) → ADC4_IN18`，依据 RM Table 359
- 说明：关系确实存在但是**经 GPIO 引脚**（不是内部直连），原图标的 ⚠ 可以去掉。

### 14. 其它文字修正
- PWR 的中断由 `1 (PVD_PVM)` 改为 `77 (PWR_S3WU)`；PVD/PVM 走 EXTI16 → IRQ1，已由 `PWR → EXTI` 那条边表达。
- CRS 时钟由 `⚠` 补齐为 `APB1ENR1[24]`。
- ICACHE 时钟由 `⚠ 随 FLITF` 改为 `无 ENR 位`（RCC 无 ICACHEEN，只有 AHB1SMENR[29] 的 SmartRun 位）。
- DCACHE1 说明补容量 `4 KB`。
- ADC4 依据寄存器名 `ADC4_CFGR` → `ADC4_CFGR1`。
- `COMP1 → TIM5` 标签补注「TI1 需 TI1SEL=4」（TIM5 的 ti1_in1 是 LSI，comp1_out 在 ti1_in4；ETR 仍为 ETRSEL=1）。

---

## 二、核对通过（未改动）的重点项

- **IRQn**：WWDG 0、PVD_PVM 1、RTC 2/3/4、RAMCFG 5、FLASH 6/7、GTZC 8、RCC 9/10、EXTI 11–26、IWDG 27、GPDMA1 29–36/80–87、ADC1 37、DAC1 38、FDCAN1 39/40、TIM1 41–44、TIM2–5 45–48、TIM6/7 49/50、TIM8 51–54、I2C1 55/56、I2C2 57/58、SPI1 59、SPI2 60、USART1–3 61–63、UART4/5 64/65、LPUART1 66、LPTIM1/2 67/68、TIM15–17 69–71、COMP 72、OTG_FS 73、CRS 74、FMC 75、OCTOSPI1 76、SDMMC1 78、SDMMC2 79、I2C3 88/89、SAI1/2 90/91、TSC 92、RNG 94、HASH 96、LPTIM3 98、SPI3 99、I2C4 100/101、MDF1 102–105/121–122、UCPD1 106、ICACHE 107、LPTIM4 110、DCACHE1 111、ADF1 112、ADC4 113、LPDMA1 114–117、DMA2D 118、DCMI/PSSI 119、OCTOSPI2 120、CORDIC 123、FMAC 124。
- **时钟使能位**：AHB1ENR 0/1/2/3/8/12/16/17/18/24/30/31、AHB2ENR1 0–8/10/12/14/17/18/27/28/30/31、AHB2ENR2 0/4/8、AHB3ENR 0/2/5/6/9/10/12/31、APB1ENR1 0–5/11/14/17–22/24、APB1ENR2 1/5/9/23、APB2ENR 11–14/16–18/21/22、APB3ENR 1/5/6/7/11–15/20/21 —— 全部与 SVD 一致。
- **ADC1 规则触发**：trg0 tim1_oc1、1 tim1_oc2、2 tim1_oc3、3 tim2_oc2、4 tim3_trgo、5 tim4_oc4、6 exti11、7 tim8_trgo、8 tim8_trgo2、9 tim1_trgo、10 tim1_trgo2、11 tim2_trgo、12 tim4_trgo、13 tim6_trgo、14 tim15_trgo、15 tim3_oc4、16 exti15、17 保留、18 lptim1_ch1、19 lptim2_ch1、20 lptim3_ch1、21 lptim4_out —— 图中 21 条全对。
- **ADC1 注入触发**（保留部分）：0 tim1_trgo、1 tim1_oc4、2 tim2_trgo、3 tim2_oc1、4 tim3_oc4、5 tim4_trgo、6 exti15、7 tim8_oc4、8 tim1_trgo2、9 tim8_trgo、10 tim8_trgo2。
- **ADC4**：只有 trg0–7（tim1_trgo2 / tim1_oc4 / tim2_trgo / tim15_trgo / tim6_trgo / lptim1_ch1 / lptim3_ch2 / exti15），无注入组 —— 与图一致。
- **DAC TSEL**：1 tim1_trgo、2 tim2_trgo、3 tim4_trgo、4 tim5_trgo、5 tim6_trgo、6 tim7_trgo、7 tim8_trgo、8 tim15_trgo、11 lptim1_ch1、12 lptim3_ch1、13 exti9 —— 与图一致。
- **ITR**：TIM1（1–8）、TIM8（0–4、6–8）、TIM2/3/4/5（各自缺 1 个自环位，分别为 1/2/3/4）、TIM15（0–5、7、8）—— 与 Table 532/557/574 完全一致；TIM16/17 无 ITR 也正确。
- **DMA REQSEL**：GPDMA1（ADC1 0、ADC4 1、DAC1_CH1 2、SPI1 6/7、I2C1 12/13、USART1 24/25、SAI1 36/37、SAI2 38/39、TIM1 42/46/47）、LPDMA1（ADC4 7）—— 与 Table 136/146 一致。
- **模拟通路**：DAC1→ADC4 VIN[21]、DAC1→OPAMP1/2 VINP、COMP1/2→TIM1/8 ETR1/2 与 BKIN/BKIN2、COMP1/2→TIM2/3/4/5 ETR、COMP1→LPTIM1 trig6 / COMP2→trig7、ADC1 AWD1–3→ETR8–10、ADC4 AWD1–3→ETR11–13 —— 全部有依据。
- **存储/外设容量**：FLASH 2 MB、SRAM1 192 KB / SRAM2 64 KB / SRAM3 512 KB / SRAM4 16 KB / BKPSRAM 2 KB、ICACHE 8 KB、DCACHE1 4 KB、TIM2–5 为 32 位、TIM1/8/15/16/17 为 16 位、MDF1 6 个滤波器 —— 与 DS Table 19 及框图一致。
- **EXTI**：0–15 GPIO、16 PVD、17 COMP1、18 COMP2、19 VDDUSB、20 VDDIO2、21/22 VDDA、23 LSECSS/MSI_PLL_UNLOCK（U575 Rev Y 起），共 0–23 共 24 条 —— 与 RM Table 187 一致。

## 三、未改动，但属于「简化」而非错误的标注

- EXTI、IWDG 在图中总线写 `CORE`。两者确实没有 RCC 使能门控（EXTI 无 ENR 位，IWDG 由 LSI 驱动），但物理上分别位于 SRD（0x4602 2000）与 APB1 空间。保留 `CORE` 是为了不虚构使能位。
- GTZC / MPCBB / TZIC 安全域那 12 条边（页面底部折叠区）按你先前选择**未纳入本次核对**。

## 四、图里没有、但手册确有连线的项（供参考，未添加）

- COMP1/2 → TIM15/16/17（tim_ti1/ti2 与 BRK）
- COMP1/2 → LPTIM2/3/4 触发；COMP → LPTIM 输入捕获
- MDF1/ADF1 触发源（TIM1/3/4/6/7/8/16、LPTIM1、EXTI11/15）
- TIM1/8 的 tim_ocref_clr 来自 COMP1/2
- TIM3/TIM4/TIM5 的 ti1 还挂着 LSI/LSE/rtc_wut
