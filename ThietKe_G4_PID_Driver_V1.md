# Thiết kế: Board driver đơn 1 động cơ encoder — G4_PID_Driver_V1

Nov 10, 2026 · @pham chinh · Trạng thái: **bản nháp thiết kế (giai đoạn 0 – chốt spec)**

Tài liệu gốc: [ThietKe_F1_PID_Driver_V2.md](ThietKe_F1_PID_Driver_V2.md). Board này **giữ nguyên toàn bộ tầng công suất và mạch an toàn SD/ENA của F1_PID_Driver_V2** (cầu H 4 FET + 2× gate driver chuẩn IR2104 — nay là IRS2104S, 12VG boost, buck 5 V, chống ngược cực Q5, TVS, tản nhiệt xuống vỏ), chỉ thay phần MCU bằng **STM32G431CBT6** và tận dụng ngoại vi tích hợp để bỏ IC ngoài.

Những gì **không nhắc lại** ở đây thì giữ như F1 V2 (mục 6.2–6.5 tính toán gate/bootstrap/nhiệt/đồng/TVS, mục 4.5 điều kiện USB không cách ly, mục 9 quy tắc layout công suất).

Lịch sử quyết định:
- Nov 10, 2026: tạo từ F1_PID_Driver_V2. **MCU STM32G431CBT6**; **USB native** (bỏ CH340C, nạp bằng ROM DFU); **đo dòng bằng OPAMP nội ở chế độ PGA** (bỏ INA181); **cắt dòng bằng COMP nội + DAC3 nội → TIM1_BKIN bên trong chip** (bỏ TLV7011 và mạch PWM + RC tạo VREF_OC).
- Nov 10, 2026: **bỏ FDCAN**. Trên LQFP48, FDCAN1 chỉ ra PA11/PA12 (USB) hoặc PB8/PB9 (PB8 = BOOT0) → giữ CAN buộc phải khóa chân BOOT0 bằng option byte và mất đường vào DFU bằng nút. Bỏ CAN thì: **PB8 trở lại làm BOOT0 + nút**, option byte để mặc định, và **bỏ luôn thạch anh** (chỉ CAN mới cần; USB dùng HSI48 + CRS, hệ thống chạy HSI16 → PLL 170 MHz).
- Nov 10, 2026: gom mã linh kiện **33 → 29 mã**, ~170 → **~159 linh kiện** (Phụ lục A).
- Nov 10, 2026: **bỏ mã tụ 1 µF (M08), gộp vào 10 µF/25 V 0805 (M09)**: bootstrap mỗi nửa cầu 2 × 1 µF → **1 × 10 µF** (~3–4 µF hiệu dụng ở 12 V do DC bias, ≥ 0,7 µF yêu cầu; R 10 Ω giữ dòng nạp lần đầu ~1,2 A); VCC EG2104, vào/ra LDO, VDDA, VREF+ → 10 µF. Bỏ C12, C14. Thêm R74 100 kΩ (EN MP2459 lên VIN) vào BOM. → **28 mã, ~158 linh kiện**.
- Nov 10, 2026: **buck 5 V đổi MP2459 → TI LMR51606XDBVR** (LCSC C21260865): 4–65 V làm việc, **70 V tuyệt đối** (MP2459: 55 V), đồng bộ 0,6 A, SOT-23-6, **cùng thứ tự chân** (1 CB, 2 GND, 3 FB, 4 EN, 5 VIN, 6 SW), V_FB 0,8 V → giữ chia áp FB (12 k / 2 k → 5,6 V). Lý do: MP2459 khó mua ổn định ở LCSC. Đồng bộ nên **bỏ D10** (diode bắt); EN được phép nối thẳng VIN (≤ VIN + 0,3 V) nên **bỏ R74**. Đã xét và loại TPS54202 (28 V làm việc / 30 V tuyệt đối < gai 42 V) và AP64060 (40 V). Dự phòng: SCT2601TVBR (60 V, không đồng bộ, cần lại D10 và tính lại FB vì V_FB ~0,765 V). Cần kiểm datasheet: hậu tố X/Y (tần số 400 kHz / 1,1 MHz, PFM/FPWM) và cuộn 22 µH phù hợp. → **28 mã, ~156 linh kiện**.
- Nov 10, 2026: **MOSFET Q1–Q5: TPH1R306PL → Infineon BSC014N06NS** (LCSC C113391): 60 V, **R_DS(on) max 1,45 mΩ @10 V**, gói PG-TDSON-8 trùng footprint `TDSON-8-1`, phổ biến và rẻ hơn ở LCSC. Dự phòng: BSC016N06NS (kiểm R_DS max ≤ 1,6 mΩ); HYG019N06LS1C2 (~1,9 mΩ, rẻ nhất, phải hạ định mức ~13–14 A). **Gate driver U1/U2: EG2104S → Infineon IRS2104STRPBF**: cùng chân IR2104, **V_IH IN/SD = 2,5 V min** (Infineon hạ từ 3,0 V xuống để tương thích logic 3,3 V) — đã xác nhận, trong khi V_IH của EG2104 không tìm được nguồn xác nhận. Dòng ra ~130/270 mA → chuyển mạch kiểu IR2104 như đã tính ở mục 6.3 của F1 V2. Dự phòng: onsemi NCP5104DR2G (V_IH 2,3 V min, kiểm thứ tự chân); EG2104S chỉ dùng lại nếu datasheet EG Micro ghi V_IH ≤ 2,5 V. MCU giữ STM32G431CBT6 (LCSC C529355 tồn kho lớn).

---

## 1. Mục tiêu & thứ tự ưu tiên

Giống F1 V2: board điều khiển vòng kín (dòng → tốc độ → vị trí) cho **một** động cơ DC có encoder, đặt **trong hộp kín**, pin 3S–6S, nối với bộ điều khiển chủ qua USB / UART / I2C / Step-Dir.

1. **An toàn phần cứng** (không thương lượng): mọi MOSFET tắt khi MCU reset/treo/đang ở bootloader; cắt dòng từng chu kỳ qua TIM1 break; ENA ngắt SD bằng phần cứng; TVS + tụ ceramic sát cầu H; chống ngược cực; giảm công suất theo nhiệt.
2. **Giá thành**: mục tiêu **≤ 320.000 đ/board** (lô 10, linh kiện + PCB + SMT; chưa gồm tấm tản nhiệt/vỏ).
3. **Kích thước**: mục tiêu **40 × 62 mm**, SMT một mặt, không connector công suất.

## 2. Thông số mục tiêu

| Thông số | Giá trị | Ghi chú |
| --- | --- | --- |
| Nguồn VM | **Pin 3S–6S: 9,0–25,2 V** | Như F1 V2 |
| Dòng động cơ | **15 A liên tục trong hộp kín** (có đường tản nhiệt xuống vỏ), cắt phần cứng **30 A** (chỉnh bằng firmware) | Không có đường tản nhiệt → hạ ~8–10 A |
| PWM | 20 kHz (16–40 kHz cấu hình được), PH/EN sign-magnitude, **duty ≤ 97 %** | 170 MHz center-aligned → 4250 bước ở 20 kHz (F1: 1800) |
| MCU | **STM32G431CBT6** (LQFP48, Cortex-M4F 170 MHz, FPU, CORDIC, FMAC, 128 KB Flash / 32 KB SRAM) | Nâng cấp cùng chân: G431C8T6/C6T6 (ít Flash hơn), G441CBT6, G474CBT6 — kiểm datasheet |
| Clock | **HSI16** (±1 %) → PLL 170 MHz; **HSI48 + CRS** cho USB | Không thạch anh. Footprint HSE 3225 + 2 tụ để **DNP** (không lắp) phòng khi cần |
| Encoder | 1× A/B, 3,3 V hoặc 5 V, đến 150 kHz/kênh | TIM2 **32 bit** |
| Truyền thông | **USB-C native** (CDC, không cách ly), **UART** 3,3 V (chịu 5 V), **I2C** 3,3 V | **Không có CAN** |
| Ngõ vào lệnh | Step/Dir hoặc xung RC (3,3–5 V), analog 0–3,3 V | Step/Dir đếm bằng phần cứng (TIM3 chế độ clock + direction) |
| I/O | **ENA** + **4 I/O đa dụng 3,3–5 V** | Trên chân FT |
| Nạp firmware | **USB DFU** (ROM bootloader): vào bằng lệnh GUI, hoặc **giữ nút BOOT0 + nhấn RST**; **SWD** dự phòng | Board mới từ JLCPCB nạp lần đầu được ngay qua USB, không cần ST-Link |
| Đấu nối | Pad hàn trực tiếp như F1 V2 | |
| Lắp cơ khí | 4 lỗ M2 NPTH ở 4 góc | |
| PCB | 4 lớp, đồng ngoài 2 oz, L2 GND liền | |

Ngoài phạm vi: CAN, nhiều trục, BLDC/step, cách ly.

## 3. Hệ quả của việc chọn STM32G431

### 3.1 Bỏ được gì so với F1 V2

| F1 V2 (F103) | G4 V1 (G431) | Lợi |
| --- | --- | --- |
| CH340C → USART1, ROM bootloader USART | **USB FS native** PA11/PA12, D+ pull-up nội, ROM bootloader **DFU** | −1 IC, −1 tụ; PA9/PA10 không còn bị giữ |
| INA181A1 ×20 + 2 RC lọc | **OPAMP3 PGA ×16** (chế độ có chân VINM0 làm điểm tham chiếu Kelvin) | −1 IC, −5 linh kiện thụ động |
| TLV7011 + PWM TIM3_CH4 + RC 2 bậc + R kéo xuống + pull-up nOC | **COMP1** (INM = **DAC3_CH1** nội) → **TIM1 break nội** | −1 IC, −8 linh kiện thụ động; đổi ngưỡng tức thì; có blanking theo timer |
| TIM1 CH1 + CH1N, đổi chiều bằng CCER | **TIM1_CH1 (PA8) + TIM1_CH2 (PA9) độc lập** | Firmware chỉ ghi CCR1/CCR2, có preload, đổi chiều không cần thủ thuật |
| Fixed-point 72 MHz, vòng dòng sát giới hạn CPU | **Float 170 MHz** + CORDIC | Hết rủi ro R11 của F1; vòng dòng 20–40 kHz dư sức |
| TIM2 16 bit + mở rộng bằng ngắt | **TIM2 32 bit** | Bớt code |
| I2C có errata, phải timeout + reset ngoại vi | I2C của G4 không có lỗi kẹt kiểu F1 | Vẫn giữ timeout (dây dài vẫn có thể treo bus) |
| Resonator 8 MHz (HSI F103 chỉ tới 64 MHz, trôi ±2–3 %) | **HSI16 ±1 %** → PLL 170 MHz | −1 linh kiện; UART đến 1 Mbaud vẫn trong dung sai |

### 3.2 Bẫy phần cứng của G431 (bắt buộc đọc trước khi vẽ)

| # | Bẫy | Xử lý trong thiết kế |
| --- | --- | --- |
| B1 | **PB8 là BOOT0.** Mặc định (`nSWBOOT0 = 1`) chân PB8 được đọc khi reset: mức cao → vào ROM bootloader | PB8 **chỉ** làm BOOT0: R 10 kΩ kéo xuống + nút lên 3V3 qua 1 kΩ. **Không** đặt I2C SCL ở PB8 như gán chân F1 (pull-up I2C sẽ làm chip luôn vào bootloader). Option byte để mặc định |
| B2 | **PB4, PB6 là chân CC của UCPD1**: R kéo xuống "dead battery" (~5,1 kΩ) **bật sẵn khi reset** đến khi firmware đặt `PWR_CR3.UCPD_DBDIS = 1` | Chỉ gán cho tín hiệu mà tải 5,1 kΩ lúc khởi động không gây hại: **PB4 = STEP** (master lái push-pull), **PB6 = LED STATUS**. Firmware đặt `UCPD_DBDIS` ngay đầu `main()` |
| B3 | **Chịu 5 V**: chân FT_a mất khả năng chịu 5 V khi ở chế độ analog; một số chân DAC/OPAMP là loại TT (≤ 3,6 V) | Mọi tín hiệu ngoài 5 V chỉ vào chân **FT ở chế độ số**. Cột "5 V?" trong mục 8 **phải đối chiếu bảng pin datasheet G431** trước khi vẽ |
| B4 | **Sơ đồ chân LQFP48 khác F103**: không thay thế chân-chân | Vẽ lại phần MCU; vị trí VDD/VSS/VDDA/VREF+/NRST theo datasheet G431 |
| B5 | **Không có chip nội địa thay thế phổ biến** như GD32/APM32F103 | Chấp nhận. Dự phòng cùng chân: G431C8T6 (64 KB), G441CBT6, G474CBT6 |
| B6 | ROM bootloader G4 thăm dò cả USART1 (PA9/PA10) khi đang ở DFU | PA9 = IN_B, PA10 = IO2: trong bootloader cầu H vẫn tắt nhờ SD = 0 (nDIS Hi-Z → Q8 dẫn). Nhiễu trên IO2 có thể làm bootloader chọn USART thay vì DFU → rút dây IO2 khi nạp nếu gặp |
| B7 | OPAMP nội có offset lớn hơn INA181 (cỡ mV, trong khi shunt 2 mΩ → 1 mV = 0,5 A) | Thêm **offset dương** ở ngõ vào (R 100 kΩ từ VDDA, mục 6.1) để không mất vùng gần 0 A; firmware **hiệu chỉnh 0 A** mỗi lần khởi động khi SD = 0 |
| B8 | Tín hiệu shunt mức mV phải đi từ vùng công suất sang MCU (trước đây INA181 khuếch đại ngay cạnh shunt) | Đi cặp Kelvin trên **L3 kẹp giữa L2 GND**; đặt MCU sao cho PB0/PB2 gần ranh giới công suất; dùng chế độ PGA lấy **VINM0 (PB2) làm tham chiếu Kelvin** để sụt áp GND giữa shunt và MCU chỉ vào ×1 thay vì ×16 (mục 4.4) |

## 4. Kiến trúc

### 4.1 Sơ đồ khối

```
 ┌──────────────── VÙNG CÔNG SUẤT (như F1 V2) ──────┐  ┌──────────────── VÙNG LOGIC ─────────────────┐
 │ pad VM+ ─┬─ TVS SMCJ26A                           │  │ USB-C ─ SRV05-4 ─ PA11/PA12 (USB FS native)  │
 │          ├─ 3× 220µF/50V + 4× 4,7µF/50V           │  │   CDC: giao thức JSON / GUI Web Serial       │
 │          ├─ nửa cầu A: Q1/Q2 ◄─ IRS2104 A ◄─ IN_A ┼──┤◄─ PA8  TIM1_CH1                             │
 │          ├─ nửa cầu B: Q3/Q4 ◄─ IRS2104 B ◄─ IN_B ┼──┤◄─ PA9  TIM1_CH2                             │
 │          │       └── SD (cả hai) ◄──────────── SD ┼──┤◄─ pad ENA qua 10 kΩ                         │
 │          │                                        │  │     Q8 AO3400 kéo SD = 0 ◄─ nDIS (PB14)     │
 │          │   M+ = SH_A, M− = SH_B ──► pad M+/M−   │  │     ENA đọc lại → PB15                      │
 │          └─ Buck LMR51606 → 5,6V ─[SS36]─ +5V ────┼──┤ +5V ─ LDO ─ 3V3 ─ MCU                        │
 │             +5V ─ boost MT3608 → +12VG (gate)     │  │ +5V ─ ferrite ─ encoder                      │
 │                                                   │  │                                              │
 │ shunt 2 mΩ ─┬ Kelvin+ ─ 1kΩ ─┬───────────────────┼──► PB0  OPAMP3_VINP ┐ PGA ×16                   │
 │             │       VDDA ─ 100kΩ ┘ (offset +33mV) │  │                     ├─► PB1 OPAMP3_VOUT        │
 │             └ Kelvin− ───────────────────────────┼──► PB2  OPAMP3_VINM0 ┘   ├─► ADC (giữa pha ON)   │
 │                                                   │  │                         └─► COMP1_INP          │
 │                                                   │  │        DAC3_CH1 ─► COMP1_INM                   │
 │                                                   │  │        COMP1_OUT ─► TIM1 BRK (nội, AOE = 1)   │
 │ chia áp VM → PA4, NTC → PA3                       │  │ ENC A/B ─ RC ─ 74LVC2G14 ─ PA0/PA1 (TIM2 32b)│
 │ pad VM− ─ Q5 N-FET chống ngược cực ─ GND (sao)    │  │ STEP/DIR ─ RC ─ PB4/PB5 (TIM3 clock+dir)     │
 │                                                   │  │ I2C PA15/PB7, UART PB10/PB11 (USART3)        │
 │ mặt dưới: đồng hở + via → pad tản nhiệt → vỏ nhôm │  │ IO1–IO4: PB3, PA10, PB12, PB13 (FT) + ESD    │
 └───────────────────────────────────────────────────┘  │ AIN PA2 · BOOT0 PB8 + nút · SWD PA13/PA14    │
                                                        │ Clock: HSI16 → 170 MHz, HSI48 + CRS → USB    │
                                                        └──────────────────────────────────────────────┘
```

### 4.2 Các quyết định chính

| Quyết định | Chọn | Lý do / phương án bị loại |
| --- | --- | --- |
| MCU | **STM32G431CBT6** LQFP48 | LQFP dễ hàn/sửa hơn UFQFPN48; 128 KB đủ cho firmware F1 + USB CDC |
| Gate driver | **2× IRS2104STRPBF** (cùng chân IR2104/EG2104 của F1 V2) | V_IH IN/SD = 2,5 V min đã xác nhận; dự phòng NCP5104 (2,3 V) |
| Điều khiển cầu H | **TIM1_CH1 → IN_A, TIM1_CH2 → IN_B**, hai kênh độc lập | Có chân vì USB native giải phóng PA9. Bỏ thủ thuật CH1/CH1N + CCER của F1 |
| Đo dòng | **OPAMP3 PGA ×16** + offset dương từ VDDA + tham chiếu Kelvin qua VINM0 | Bỏ INA181. Đo được cả dòng âm (hãm) đến ~−13 A |
| Cắt dòng | **COMP1** (INP = ngõ ra OPAMP3 trên PB1, INM = DAC3_CH1) → **TIM1 break nội**, AOE = 1; **blanking** theo TIM1_OC5 | Trễ < 1 µs (F1: 2–3 µs); không cần chân BKIN ngoài |
| Lớp bảo vệ 2 | ADC analog watchdog ~35 A → ngắt → nDIS = 1 | Như F1 |
| USB | **Native FS**, CDC cho giao thức, ROM **DFU** để nạp; **HSI48 + CRS** | Không cần thạch anh cho USB; CC 2 × (10 k ‖ 10 k); VBUS sense qua chia áp → PC13 để chỉ bật D+ pull-up khi có VBUS |
| Clock | **HSI16 → PLL 170 MHz**, không thạch anh (footprint DNP) | Không còn CAN nên không cần HSE; HSI16 ±1 % đủ cho UART và PWM |
| CAN | **Không có** | Chân FDCAN trên LQFP48 trùng USB hoặc BOOT0 (mục lịch sử) |
| Bootloader | Option byte mặc định; vào DFU bằng **lệnh GUI** hoặc **nút BOOT0 + RST**; SWD dự phòng | Như F1 V2, nhưng qua USB DFU thay vì USART1 |
| Encoder | Giữ **74LVC2G14** | Chịu 5 V chắc chắn, Schmitt; PA0/PA1 là chân FT_a nên không cho 5 V vào thẳng |
| Step/Dir | **TIM3 chế độ encoder "clock + direction"** (STEP = TI1 PB4, DIR = TI2 PB5) | G4 có chế độ này: đếm bước có chiều hoàn toàn bằng phần cứng. TIM3 16 bit → mở rộng bằng ngắt tràn (200 kHz → tràn mỗi ~0,33 s) |
| Tầng công suất, nguồn logic, tản nhiệt | **Như F1 V2** | Đã review kỹ, không có lý do đổi |

### 4.3 Điều khiển cầu H bằng TIM1_CH1 + TIM1_CH2

IRS2104: IN = 1 → HO bật; IN = 0 → LO bật; SD = 0 → cả hai tắt. Dead-time nằm trong từng IRS2104 → **TIM1 DTG = 0**, không tổ hợp IN nào gây shoot-through.

Cấu hình: PWM mode 1, center-aligned, ARR = 170 MHz / (2 × 20 kHz) = **4250**, preload CCR bật. **OSSR = OSSI = 1, OIS1 = OIS2 = 0** → khi break hoặc MOE = 0, PA8/PA9 bị lái mức 0.

| SD | CCR1 (IN_A) | CCR2 (IN_B) | Trạng thái |
| --- | --- | --- | --- |
| 0 | x | x | **Cả 4 FET tắt** — reset, bootloader, ENA thấp, nDIS = 1 |
| 1 | duty | 0 | Chạy thuận: A băm PWM, Q4 dẫn liên tục, slow decay qua Q2–Q4 |
| 1 | 0 | duty | Chạy nghịch |
| 1 | 0 | 0 | Phanh ngắn mạch qua hai low-side (chỉ dưới ~20 % tốc độ) |
| 1 | d | d | Cấu hình sai: không có áp lên động cơ, **an toàn** |
| 1 | d1 | d2 | Cấu hình sai: động cơ chạy theo d1 − d2, **không shoot-through** |

- Đổi chiều: ghi CCR1/CCR2 mới, preload áp dụng ở update event; firmware vẫn đi qua duty ≈ 0 và giới hạn dốc để tránh dòng ngược lớn.
- Duty kẹp ≤ 0,97 × ARR (bootstrap).
- Trước khi chạy: giữ SD = 1, CCR1 = CCR2 = 0 (cả hai low-side dẫn) ≥ 1 ms để nạp tụ bootstrap.
- Mạch SD / ENA / Q8: **giữ nguyên F1 V2 mục 4.3** (SD chỉ được kéo lên từ ENA qua 10 kΩ, Q8 AO3400 có gate kéo lên 3V3, nDIS = PB14). R 10 kΩ kéo xuống tại IN_A (PA8) và IN_B (PB13 → **PA9**).

### 4.4 Đo dòng và cắt dòng bằng ngoại vi nội

**Chuỗi tín hiệu:**

```
shunt+ (Kelvin) ── R30 1 kΩ ──┬── PB0 = OPAMP3_VINP
                    VDDA ── R69 100 kΩ ──┤
                              └── C25 100 pF ── GND (sát chân PB0)
shunt− (Kelvin) ─────────────────────── PB2 = OPAMP3_VINM0 (không R nối tiếp)

OPAMP3: PGA, gain 16, chế độ "đáy thang điện trở nối VINM0"
  → VOUT = 16·V(PB0) − 15·V(PB2)  (ra chân PB1)
PB1 ─► ADC1 (injected, TIM1 kích giữa pha ON)
PB1 ─► COMP1_INP ;  DAC3_CH1 ─► COMP1_INM ;  COMP1_OUT ─► TIM1 BRK (BKCMP1E)
```

- **Tham chiếu Kelvin qua VINM0**: nếu GND tại MCU lệch V_g so với đầu âm shunt thì V(PB0) = V_sh + V_g, V(PB2) = V_g → VOUT = 16·V_sh + V_g. Sai lệch GND chỉ vào **×1** thay vì ×16 như khi để đáy thang nối VSSA.
- **Không đặt R nối tiếp ở PB2**: VINM0 nối vào đáy thang điện trở hồi tiếp, R nối tiếp sẽ làm sai hệ số khuếch đại.
- **Phương án dự phòng** (nếu chế độ VINM0 không như kỳ vọng ở bài thử): chuyển PGA sang chế độ đáy thang nối VSSA bằng firmware, không sửa phần cứng; đường PB2 để nguyên.
- **Cần kiểm trong RM0440** trước khi vẽ: bảng mux OPAMP3 (VINP0 = PB0, VINM0 = PB2, VOUT = PB1), chế độ PGA_GAIN có VINM0, COMP1 INP = PB1, COMP1 INM = DAC3_CH1, nguồn blanking COMP1 = TIM1_OC5, TIM1 break từ COMP1. Chạy STM32CubeMX để xác nhận.

**Cắt dòng từng chu kỳ:** COMP1 lật → TIM1 break → MOE = 0 → PA8 = PA9 = 0 → cả hai low-side dẫn, dòng xả qua slow decay; **AOE = 1** bật lại ở update kế tiếp. COMP1 có thể đặt hysteresis nội nhỏ; blanking TIM1_OC5 che ~300–500 ns sau mỗi cạnh bật high-side (gai reverse recovery).

**Firmware đổi ngưỡng** bằng cách ghi DAC3 (không cần chờ lọc RC 16 Hz như F1).

**Khi reset/bootloader**: COMP/DAC/OPAMP tắt, nhưng PA8/PA9 Hi-Z → kéo xuống 10 kΩ, và nDIS Hi-Z → Q8 dẫn → SD = 0. Lớp an toàn chính vẫn là SD, như F1.

### 4.5 USB native

- USB-C 16P, CC1/CC2 mỗi chân 2 × 10 kΩ song song (5 kΩ, Rd), SRV05-4 trên D+/D− (VDD → VBUS), ferrite trên VBUS → SS36 → +5V (OR với buck như F1).
- **D+ pull-up nội** (USB_BCDR.DPPU): firmware chỉ bật khi **VBUS có mặt** (chia áp 10 kΩ / 10 kΩ → PC13, ~2,5 V), tránh cấp ngược vào host khi board chạy pin mà cáp USB cắm vào host đang tắt.
- Clock USB: **HSI48 + CRS** đồng bộ theo SOF.
- Lớp ứng dụng: **CDC-ACM** → giữ giao thức JSON + lệnh `describe`, GUI Web Serial và MCP server của F1 V2 (Web Serial làm việc với CDC như với CH340).
- Điều kiện dùng USB không cách ly: **như F1 V2 mục 4.5**.

### 4.6 Nạp firmware

Option byte giữ **mặc định** (`nSWBOOT0 = 1`, `nBOOT0 = 1` → chân PB8-BOOT0 quyết định khởi động); chỉ chỉnh `BOR_LEV` (~2,8 V) nếu cần.

| Bước | Cách làm |
| --- | --- |
| Lần đầu (board mới từ JLCPCB) | Giữ **BOOT0** + nhấn **RST** → USB DFU → `dfu-util` hoặc STM32CubeProgrammer. Không cần ST-Link |
| Cập nhật thường ngày | GUI gửi `bootloader` → firmware tắt cầu (nDIS = 1, MOE = 0), tắt USB và ngắt, trả clock về HSI16, đặt MSP và nhảy vào ROM bootloader (**G4: 0x1FFF 0000** — kiểm AN2606) → USB DFU |
| Firmware hỏng | Nút BOOT0 + RST như lần đầu; SWD (pad `SWD: 3V3 DIO CLK RST GND`) là đường cuối |

Trong bootloader PA8/PA9 Hi-Z → kéo xuống 10 kΩ, nDIS Hi-Z → Q8 dẫn → SD = 0 → cầu H tắt.

## 5. Linh kiện chính (chỉ liệt kê phần khác F1 V2)

| Khối | Linh kiện | Ghi chú thiết kế |
| --- | --- | --- |
| MCU | **STM32G431CBT6** (LQFP48) | 100 nF mỗi chân VDD + 10 µF bulk 3V3 gần MCU; **VDDA** qua ferrite + 10 µF + 100 nF; **VREF+** nối VDDA + 10 µF + 100 nF (dùng VDDA làm tham chiếu ADC/DAC → tỉ lệ thuận với offset từ VDDA); **VBAT → 3V3 + 100 nF**; **PB8-BOOT0 → 10 kΩ GND + nút lên 3V3 qua 1 kΩ**; NRST (PG10) 100 nF + nút. Không còn BOOT1 |
| HSE | **Không lắp.** Footprint thạch anh 3225 + 2 tụ 0603 trên PF0/PF1 để **DNP** | Không tính vào BOM |
| Đo dòng | Shunt 2 mΩ 2512 3 W (như F1) + R30 1 kΩ + C25 100 pF + R69 100 kΩ từ VDDA | Dùng mã sẵn có, không thêm mã |
| USB | USB-C 16P + CC 2 × (10 k ‖ 10 k) + SRV05-4 + ferrite VBUS + chia áp VBUS 10 k / 10 k → PC13 | Không còn CH340C |
| Các khối còn lại | Như F1 V2 mục 5 | |

## 6. Tính toán thiết kế

### 6.1 Đo dòng và cắt dòng

- Ngõ vào PB0: V_in = V_sh × 100/101 + VDDA × 1/101 ≈ **0,990·V_sh + 32,7 mV** (VDDA = 3,3 V).
- Ngõ ra PB1: V_out ≈ 16 × V_in = **0,523 V + 31,7 mV/A** (ước tính danh định; offset OPAMP và sai số R được hiệu chỉnh ở 0 A).
- Dải đo (OPAMP ra cách rail ~0,1 V): **+84 A … −13 A**. Đo được dòng âm khi hãm tái sinh qua high-side (F1 không đo được).
- Độ phân giải ADC 12 bit: 3,3 V / 4096 / 31,7 mV/A ≈ **25 mA/LSB**.
- **I_trip = 30 A** → V(COMP1_INM) = 0,523 + 30 × 0,0317 = **1,474 V** → DAC3 = 1,474 / 3,3 × 4095 ≈ **1829**. Firmware tính từ offset đã hiệu chỉnh: `DAC = (V_out0 + I_trip × k) / VDDA × 4095`.
- Lọc ngõ vào: 1 kΩ × 100 pF = 100 ns (như F1). Băng thông PGA ×16 cỡ vài trăm kHz (GBW OPAMP G4 ~13 MHz — kiểm datasheet).
- **Trễ cắt dòng** ≈ lọc 0,1 µs + PGA ~0,5 µs + COMP ~0,05 µs + break → **< 1 µs** (F1: 2–3 µs).
- Hiệu chỉnh: khi khởi động, SD = 0, lấy trung bình 1024 mẫu → V_out0; lặp lại khi SD = 0 lâu (dừng) để bù trôi nhiệt.
- Lớp 2: ADC analog watchdog trên kênh injected ở ~35 A → ngắt → nDIS = 1.
- Shunt: 0,45 W ở 15 A, 1,8 W lúc chặt 30 A → 2512 3 W (như F1).

### 6.2 Gate drive, bootstrap — như F1 V2 mục 6.2

### 6.3 Nhiệt trong hộp kín — như F1 V2 mục 6.3

Tổng ~3,5 W; với đường tản nhiệt xuống vỏ nhôm Tj FET nóng nhất ≈ 70 °C. MCU G431 tiêu thụ nhiều hơn F103 ~10–20 mA (≈ +0,05 W) nhưng bỏ CH340C (−15 mA) → không đáng kể.

### 6.4 Nguồn logic

- Tải 5 V: G431 @170 MHz + ngoại vi analog ~40–50 mA (qua LDO) + encoder ≤ 50 mA + boost 12VG ~25 mA + LED/logic ~10 mA → **~130 mA**. LMR51606 0,6 A ✔.
- Chỉ cắm USB (không pin): VBUS ≤ 500 mA; không quay động cơ → đủ.
- LDO 3,3 V tải ~60 mA: AP2112K ✔, tổn hao ~0,12 W.

### 6.5 Clock

- HSI16 / M = 4 → 4 MHz × N = 85 → 340 MHz; /R = 2 → **SYSCLK 170 MHz** (Boost mode, Flash 4 wait state).
- USB: HSI48 + CRS đồng bộ theo SOF (chuẩn USB FS cho phép thiết bị không thạch anh theo cách này).
- UART: HSI16 ±1 % trên dải nhiệt → sai số baud của board ≤ ~1 %, còn dư so với dung sai UART ~±2–3 % tổng hai phía. Nếu master cũng dùng RC nội kém chính xác thì giữ baud ≤ 1 Mbaud và kiểm ở 85 °C (mục 12). Cần chính xác hơn → lắp thạch anh vào footprint DNP.

### 6.6 Encoder, I/O, analog — như F1 V2 mục 6.7–6.9

Thay đổi: đo VM chuyển sang **PA4**, NTC **PA3**, AIN **PA2**; chia áp VM 100 k / 10 k (thang 36,3 V).

## 7. Checklist schematic (trước ERC)

- [ ] **Trạng thái an toàn khi reset / bootloader**: gate Q8 kéo lên 3V3; SD kéo xuống 100 kΩ; PA8 và PA9 kéo xuống 10 kΩ; nút ENA kéo xuống 10 kΩ → 4 FET tắt (giữ nguyên F1 V2 mục 7).
- [ ] **PB8-BOOT0**: R 10 kΩ xuống GND + nút lên 3V3 qua 1 kΩ, **không** nối chức năng nào khác (đặc biệt không pull-up I2C).
- [ ] **PB4 / PB6**: không gán tín hiệu mà R kéo xuống 5,1 kΩ lúc khởi động gây hại (PB4 = STEP, PB6 = LED).
- [ ] PA8 → IN_A, **PA9** → IN_B nối thẳng; IRS2104S có V_IH = 2,5 V min (nếu thay bằng mã khác phải kiểm lại).
- [ ] Kelvin shunt: shunt+ → R30 1 kΩ → PB0; R69 100 kΩ từ **VDDA** (sau ferrite) → PB0; C25 100 pF PB0 → GND sát chân; shunt− → **PB2 trực tiếp**.
- [ ] PB1 (OPAMP3_VOUT) để pad test, **không** nối linh kiện nào khác.
- [ ] VREF+ nối VDDA, có 10 µF + 100 nF riêng; VDDA qua ferrite.
- [ ] Footprint thạch anh + 2 tụ trên PF0/PF1 đánh dấu **DNP**.
- [ ] USB: CC mỗi chân 5 kΩ xuống GND; **không** R pull-up ngoài trên D+; VBUS chia áp → PC13; SRV05-4 VDD → VBUS.
- [ ] **Mọi tín hiệu ngoài 3,3–5 V chỉ vào chân FT ở chế độ số** (mục 8, đối chiếu datasheet). Không tín hiệu ngoài nào vào PA0–PA7, PB0–PB2.
- [ ] Encoder qua **74LVC2G14**; VDD SRV05-4 các kênh tín hiệu → **+5V**.
- [ ] Tụ VM ghi `50V`, tụ điện phân ghi `105°C`; Q5 drain → VM−, source → GND board.
- [ ] PWR_FLAG cho VM, +12VG, +5V, +3V3, VDDA, GND; No-Connect cho PA5, PA6, PA7, PB9, PC15 (PF0/PF1 nối footprint DNP).

## 8. Gán chân MCU (STM32G431CBT6, LQFP48)

> **Chưa xác nhận** — phải kiểm bằng STM32CubeMX và bảng pin/AF trong datasheet G431 trước khi vẽ. SWD only (PA13/PA14) để giải phóng PA15, PB3. Cột "5 V?" là chân **dự kiến FT**, cần đối chiếu (B3).

| Chức năng | Chân | Ngoại vi | 5 V? | Ghi chú |
| --- | --- | --- | --- | --- |
| Encoder A / B | PA0 / PA1 | TIM2_CH1/CH2 (AF1), 32 bit | ✗ (FT_a) | Qua 74LVC2G14 |
| AIN | PA2 | ADC1 | ✗ | R 10 kΩ + 100 nF |
| NTC | PA3 | ADC1 | ✗ | |
| Đo VM | PA4 | ADC2 | ✗ | |
| Dự phòng | PA5, PA6, PA7 | ADC / OPAMP2_VOUT | ✗ | Pad test hoặc NC |
| IN_A | PA8 | TIM1_CH1 (AF6) | | Kéo xuống 10 kΩ |
| IN_B | PA9 | TIM1_CH2 (AF6) | | Kéo xuống 10 kΩ |
| **IO2** | PA10 | GPIO/EXTI | FT? | Xem B6 |
| USB D− / D+ | PA11 / PA12 | USB FS | | |
| SWDIO / SWCLK | PA13 / PA14 | SWD | | |
| I2C SCL | PA15 | I2C1 (AF4) | FT? | |
| Shunt+ (Kelvin) | PB0 | OPAMP3_VINP0 | ✗ | Analog |
| Ngõ ra PGA | PB1 | OPAMP3_VOUT → COMP1_INP, ADC1 | ✗ | Pad test |
| Shunt− (Kelvin) | PB2 | OPAMP3_VINM0 | ✗ | Không R nối tiếp |
| **IO1** | PB3 | GPIO/EXTI | FT? | Mặc định HOME |
| STEP / RC | PB4 | TIM3_CH1 (AF2) | FT? | UCPD dead-battery (B2) |
| DIR | PB5 | TIM3_CH2 (AF2) | FT? | TIM3 encoder clock + direction |
| LED STATUS | PB6 | GPIO | | UCPD dead-battery (B2), vô hại với LED |
| I2C SDA | PB7 | I2C1 (AF4) | FT? | |
| BOOT0 | PB8-BOOT0 | — | | R 10 kΩ GND + nút (B1) |
| Dự phòng | PB9 | GPIO | FT? | Pad test (có thể làm IO5) |
| UART TX / RX | PB10 / PB11 | USART3 (AF7) | FT? | Kiểm PB11 có trên gói LQFP48 |
| **IO3** | PB12 | GPIO/EXTI | FT? | |
| **IO4** | PB13 | GPIO/EXTI | FT? | |
| nDIS | PB14 | GPIO | | → gate Q8 (1 = tắt cầu) |
| ENA đọc lại | PB15 | GPIO/EXTI | FT? | |
| VBUS sense | PC13 | GPIO | | Chia áp 10 k / 10 k |
| Nút USER | PC14 | GPIO, pull-up nội | | |
| Dự phòng | PC15 | GPIO | | NC |
| HSE (DNP) | PF0 / PF1 | OSC_IN / OSC_OUT | | Footprint thạch anh không lắp |
| NRST | PG10-NRST | | | 100 nF + nút |

## 9. Layout PCB

Stackup, bố trí vùng công suất và quy tắc 1–10 **như F1 V2 mục 9**. Thay đổi:

```
 ┌○───────────────────────────────── 62 mm ────────────────────────────────○┐
 │ [VM+][VM−]  TVS  [C][C][C]bulk   IRS2104 ─ Q1 Q2 ─ [M+]                  │  ← mép công suất
 │   Q5          4,7µF×4  shunt═╗          4,7µF×2                          │
 │  (ngược cực)  ← GND sao      ║ Kelvin  IRS2104 ─ Q3 Q4 ─ [M−]  NTC buck boost│ 40 mm
 │ ·························· ║ (L3, kẹp dưới L2 GND) ························│
 │ [USB-C] ESD   R30/C25/R69 ─ STM32G431 LQFP48  (XTAL DNP)  Q8  LDO   ESD   │
 │ ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ●  pad 2,54      │  ← mép tín hiệu
 └○────────────────────────────────────────────────────────────────────────○┘
```

1. **Cặp Kelvin shunt → PB0/PB2**: đi song song, sát nhau, trên **L3** (L2 GND liền phía trên, không cắt), càng ngắn càng tốt; xoay MCU để PB0/PB1/PB2 hướng về phía shunt. R30, C25, R69 đặt **sát chân PB0**. Không đi cặp này dưới nút SH, cuộn buck/boost, hay thạch anh.
2. **Footprint thạch anh (DNP)** sát PF0/PF1; không đường tín hiệu nào chạy dưới.
3. **USB-C** ở mép vùng logic, D+/D− đi cặp 90 Ω vi sai, ngắn, không via nếu được; ESD ngay sau connector.
4. Hàng pad tín hiệu: `ENC: 5V GND A B` · `CMD: STEP DIR ENA GND` · `UART: TX RX GND` · `I2C: SCL SDA 3V3 GND` · `IO: 1 2 3 4 AIN GND` · `SWD: 3V3 DIO CLK RST GND` (như F1).
5. Bỏ CH340C, INA181, comparator, resonator → vùng logic rộng hơn, ưu tiên đặt MCU gần ranh giới để Kelvin ngắn.
6. Nút BOOT0 và RST đặt cạnh nhau, dễ bấm cùng lúc.

## 10. Firmware

Giữ **giao thức JSON + lệnh `describe`**, GUI Web Serial và MCP server của F1 V2 (chạy qua USB CDC thay cho CH340). Vòng điều khiển viết bằng **float**.

- **Khởi động**: đặt `PWR_CR3.UCPD_DBDIS = 1` đầu tiên (B2); nDIS = 1; HSI16 → PLL 170 MHz; HSI48 + CRS.
- **Nền tảng**: TIM1 CH1/CH2 20 kHz center-aligned, DTG = 0, OSSR = OSSI = 1, OIS = 0, break từ COMP1 (nội) cực tính phù hợp, AOE = 1; TIM1_OC5 tạo cửa sổ blanking cho COMP1; TIM1 TRGO2 kích ADC1 injected (PB1) ở giữa pha ON; ADC regular (VM, NTC, AIN) qua DMA với oversampling phần cứng; OPAMP3 PGA ×16 (chế độ VINM0); DAC3_CH1 → COMP1_INM; TIM2 encoder 32 bit; TIM3 clock + direction (STEP/DIR) hoặc input capture (RC); IWDG; PVD; BOR qua option byte.
- **Lấy mẫu dòng**: chỉ tin mẫu khi pha ON ≥ ~1,5 µs (duty ≥ ~3 % ở 20 kHz); dưới mức đó giữ giá trị trước / ước lượng.
- **Hiệu chỉnh 0 A**: lúc khởi động và mỗi khi dừng > 1 s (SD = 0).
- **Vòng điều khiển**: dòng PI 20 kHz (lên 40 kHz được) → tốc độ PI 1–2 kHz → vị trí PID 1 kHz, profile hình thang/S-curve; dùng CORDIC/FMAC khi cần.
- **Nguồn lệnh** (mỗi lúc một nguồn): USB CDC, UART (USART3), I2C slave, Step/Dir, RC, analog.
- **I/O, bảo vệ**: như F1 V2 (kẹt rotor, I²t, thấp/quá áp VM, NTC 85/100/105 °C, timeout lệnh, ENA thấp → `ENA_OFF`, nghi ngắn mạch → nDIS = 1).
- **Nạp firmware**: lệnh `bootloader` → nDIS = 1, MOE = 0, tắt USB/ngắt, de-init clock về HSI16, nhảy **0x1FFF 0000** → USB DFU (mục 4.6).

## 11. BOM & chi phí ước tính (lô 10 board)

| Khối | So với F1 V2 | Ước tính (đ) |
| --- | --- | --- |
| MCU | STM32G431CBT6 (không thạch anh) | 65–115k (**cần báo giá LCSC**) |
| Gate driver, rail gate, MOSFET, bảo vệ VM, tụ VM, nguồn logic | Như F1 | 69–118k |
| Đo/cắt dòng | Chỉ còn shunt + 3 linh kiện thụ động | 4–7k (F1: 9–14k) |
| USB | USB-C, SRV05-4 (bỏ CH340C) | 3–6k (F1: 8–12k) |
| ESD I/O, encoder | Như F1 | 4–8k |
| Thụ động, nút, LED | Ít hơn ~10 linh kiện | 11–17k |
| PCB | Như F1 | 30–60k |
| SMT | Bớt 4 mã Extended (CH340C, INA181, TLV7011, resonator) | 33–72k |
| **Tổng board** | | **~215–400k** |
| Ngoài board | Pad tản nhiệt (+ tấm nhôm) | 10–30k |

MCU đắt hơn F103 nhưng bù bằng 3 IC bỏ đi và ít mã Extended hơn → tổng gần như ngang F1 V2. Muốn ≤ 320k: như F1 (MOSFET nội địa đạt R_DS max, lô 20, đổi sang Basic). Giá chỉ là ước tính.

## 12. Kiểm thử & tiêu chí nghiệm thu

Như F1 V2 mục 12, với các thay đổi/bổ sung:

| Bài thử | Cách làm | Tiêu chí đạt |
| --- | --- | --- |
| Nạp lần đầu | Board mới: giữ BOOT0 + nhấn RST, cắm USB | Máy tính thấy thiết bị DFU; nạp bằng `dfu-util` OK; trong DFU đo SD = 0 |
| Chỉ USB | Cắm USB, chưa có VM | Enumerate CDC; lệnh `bootloader` → vào DFU, nạp lại OK; thả BOOT0 + RST → chạy firmware |
| VBUS sense | Chạy pin, cắm/rút USB | D+ pull-up chỉ bật khi có VBUS; không áp ngược vào VBUS khi rút |
| Clock / UART | Đo tần số PWM ở 25 °C và 85 °C; UART 1 Mbaud với master | PWM 20 kHz ±1 %; không lỗi khung UART trong 10 phút ở cả hai nhiệt độ; USB không rớt |
| Hiệu chỉnh 0 A | SD = 0, 25 °C và 85 °C (máy sấy) | V_out0 trong dải dự kiến (~0,45–0,6 V); trôi < 0,2 A tương đương |
| Đo dòng | So đồng hồ −10…+15 A (cả chiều hãm) | Sai số < 5 % toàn thang sau hiệu chỉnh |
| Kelvin / GND | 15 A liên tục, so đo PGA với đồng hồ | Sai lệch do GND < 0,3 A (kiểm hiệu quả chế độ VINM0; nếu kém → thử chế độ VSSA và so) |
| Cắt dòng | Kẹt rotor ở 25,2 V, ngưỡng 30 A | Dòng chặt ~30 A mỗi chu kỳ; **trễ break < 1 µs**; không cắt giả ở 15 A nhờ blanking; lặp 10 lần |
| Đổi ngưỡng | Đổi DAC3 10 A / 20 A / 30 A | Dòng chặt theo đúng ngưỡng ±10 % |
| Step/Dir | 200 kHz, đổi DIR liên tục | Đếm đúng (TIM3 clock + direction), không mất bước khi tràn 16 bit |
| Nhiễu / USB | 15 A đảo chiều 30 phút, USB + encoder cùng nối | Không reset, không mất USB, không sai xung |
| Các bài còn lại | Nguồn, ngược cực, trạng thái an toàn, dead-time, encoder, I/O, ENA, hãm, nhiệt, giảm công suất, điều khiển | Như F1 V2 |

## 13. Rủi ro

Giữ R1–R9 của F1 V2 (nhiệt, pad tản nhiệt, V_IH gate driver — đã giảm nhờ IRS2104S có V_IH 2,5 V xác nhận, ngắn mạch M± xuống GND, nhiễu, USB không cách ly, reset do nhiễu, gai VM, tụ bulk). Bỏ R10 (thay chip tương đương), R11 (CPU), R12 (I2C F103), R3b (CCER). Thêm:

| # | Rủi ro | Biểu hiện | Cách phòng |
| --- | --- | --- | --- |
| G1 | Nhiễu trên PB8-BOOT0 lúc reset | Thỉnh thoảng khởi động vào bootloader (vẫn an toàn vì SD = 0) | R 10 kΩ kéo xuống sát chân, đường từ nút ngắn, xa vùng công suất |
| G2 | Lệnh nhảy bootloader lỗi | Không vào DFU từ GUI | Nút BOOT0 + RST; SWD là đường cuối |
| G3 | Offset/trôi nhiệt OPAMP nội, sai số R69/R30 | Dòng đo lệch, vòng dòng kém ở dòng nhỏ | Offset dương + hiệu chỉnh 0 A mỗi lần dừng; R 1 % |
| G4 | Tín hiệu shunt mức mV đi xa bị nhiễu | Dòng đo nhiễu, cắt dòng giả | Kelvin trên L3 kẹp GND, lọc RC sát chân, blanking COMP, chế độ VINM0 |
| G5 | Bảng mux OPAMP/COMP/blanking khác dự kiến | Không nối được PGA → COMP | **Xác nhận RM0440 + CubeMX trước khi vẽ**; dự phòng nối PB1 ↔ chân COMP khác bằng đường mạch |
| G6 | Chân gán là TT/FT_a chứ không phải FT | Hỏng chân khi nhận 5 V | Đối chiếu bảng pin datasheet (B3) trước khi vẽ |
| G7 | HSI16 trôi nhiệt khi master cũng dùng RC nội | Lỗi khung UART ở baud cao | Giữ ≤ 1 Mbaud; bài thử 85 °C; lắp thạch anh vào footprint DNP nếu cần |
| G8 | Nguồn hàng G431 (một nhà sản xuất) | Thiếu chip | Cùng chân: G431C8T6, G441CBT6, G474CBT6; mua dư cho lô |
| G9 | Bootloader G4 chọn nhầm USART1 khi ở DFU (nhiễu PA10/IO2) | DFU không nhận | Rút dây IO2 khi nạp; tài liệu hướng dẫn |

## 14. Câu hỏi mở (chốt trước giai đoạn 1)

- [x] **USB**: native, CDC + ROM DFU.
- [x] **Đo dòng**: OPAMP3 nội, PGA ×16, offset dương, tham chiếu Kelvin qua VINM0.
- [x] **CAN**: bỏ (Nov 10, 2026) — chân FDCAN trên LQFP48 trùng USB hoặc BOOT0.
- [x] **Clock**: HSI16 + HSI48/CRS, không thạch anh (footprint DNP).
- [ ] **Xác nhận bằng CubeMX / RM0440**: mux OPAMP3–COMP1–DAC3, blanking TIM1_OC5, TIM3 clock + direction trên PB4/PB5, PB11 có trên LQFP48, danh sách chân FT (mục 8).
- [ ] **Vỏ hộp** nhôm hay nhựa, **nhiệt độ môi trường** tối đa, **I/O kéo tải** hay chỉ mức logic (mang từ F1 V2).

---

## Phụ lục A — BOM dự kiến (đã gom mã linh kiện)

**28 mã**, ~**156 linh kiện**. RefDes giữ theo F1 V2 khi cùng chức năng; linh kiện bỏ đi để trống số; linh kiện mới đánh từ số tiếp theo. Mọi điện trở 0603 1 %, mọi tụ ceramic nhỏ 0603 50 V (trừ ghi khác).

### A0. Bảng mã linh kiện hợp nhất (28 mã)

| Mã | Linh kiện | Gói | SL | Dùng ở |
| --- | --- | --- | --- | --- |
| M01 | Điện trở 10 Ω 1 % | 0603 | 7 | R gate ×4, nối tiếp diode bootstrap ×2, lọc 12VG |
| M02 | Điện trở 1 kΩ 1 % | 0603 | 25 | FB buck/boost, gate Q5, gate Q8, ENA, Kelvin shunt+, LED, nút BOOT0, encoder, STEP/DIR, IO, UART |
| M03 | Điện trở 10 kΩ 1 % | 0603 | 28 | G–S FET, kéo xuống IN, NTC, FB, Q8, ENA, PB8, CC, AIN, pull-up IO, I2C, chia áp VM, chia áp VBUS |
| M04 | Điện trở 100 kΩ 1 % | 0603 | 4 | Chia áp VM, G–S Q5, kéo xuống SD, offset PGA |
| M05 | Shunt 2 mΩ 1 % 3 W, TCR ≤ 100 ppm | 2512 | 1 | Đo dòng |
| M06 | Tụ 100 pF 50 V C0G | 0603 | 10 | Lọc PB0, ENA, encoder ×2, STEP/DIR ×2, IO ×4 |
| M07 | Tụ 100 nF 50 V X7R | 0603 | 15 | Tách nguồn IC, VM nửa cầu, BST buck, NRST, AIN, đo VM, VREF+ |
| ~~M08~~ | ~~Tụ 1 µF~~ — **bỏ, gộp vào M09** | | 0 | |
| M09 | Tụ 10 µF 25 V X5R | 0805 | 16 | Boost, lọc 12VG, buck, bulk 5 V, bulk 3V3, bootstrap ×2, VCC gate driver ×2, LDO ×2, VDDA, VREF+ |
| M10 | Tụ 4,7 µF 50 V X7R | 1210 | 6 | Commutation ×4, lọc π buck ×2 |
| M11 | Tụ điện phân 220 µF 50 V 105 °C low-ESR/polymer | SMD 10 × 10,5 | 3 | Bulk VM |
| M12 | **Infineon BSC014N06NS** (LCSC C113391): N-MOSFET 60 V, R_DS max 1,45 mΩ @10 V | PG-TDSON-8 (5×6) | 5 | Q1–Q4, Q5 |
| M13 | AO3400 | SOT-23 | 1 | Q8 |
| M14 | 1N4148W | SOD-123 | 6 | Bootstrap ×2, tắt nhanh gate ×4 |
| M15 | Schottky SS36 | SMA | 3 | Boost, OR nguồn 5 V ×2 |
| M16 | TVS SMCJ26A | SMC | 1 | VM |
| M17 | LED xanh lá | 0603 | 2 | PWR, STATUS |
| M18 | **Infineon IRS2104STRPBF** (V_IH 2,5 V min; dự phòng NCP5104DR2G) | SOIC-8 | 2 | Gate driver |
| M19 | **MT3608** | SOT-23-6 | 1 | Boost 12VG |
| M20 | **LMR51606XDBVR** (TI, LCSC C21260865; dự phòng SCT2601TVBR) | SOT-23-6 | 1 | Buck 5 V đồng bộ, 4–65 V |
| M21 | **AP2112K-3.3** | SOT-23-5 | 1 | LDO 3,3 V |
| M22 | **STM32G431CBT6** | LQFP-48 | 1 | MCU |
| M23 | **74LVC2G14** | SOT-23-6 | 1 | Schmitt encoder |
| M24 | **SRV05-4** | SOT-23-6 | 5 | ESD: USB, encoder + STEP/DIR, IO, UART + I2C, ENA + AIN |
| M25 | Cuộn 22 µH, I_sat ≥ 1 A | 4 × 4 mm | 2 | Boost, buck |
| M26 | Ferrite 600 Ω @100 MHz, ≥ 1 A | 0805 | 4 | Lọc π buck, VDDA, VBUS, nguồn 5 V encoder |
| M27 | NTC 10 kΩ B3950 | 0603 | 1 | Nhiệt FET |
| M28 | Nút nhấn SMD 3 × 4 mm | SMD | 3 | RST, BOOT0, USER |
| M29 | USB-C 16P (USB 2.0) | SMD | 1 | Cổng USB |

Bỏ so với F1 V2: INA181A1, TLV7011, CH340C, resonator 8 MHz, tụ 1 µF → 33 − 5 = **28 mã**. Footprint thạch anh + 2 tụ trên PF0/PF1 là **DNP**, không tính.

### A1. Nguồn vào & bảo vệ VM — như F1 V2 (Q5, R1, R2, D2, C1–C10, R3, R4)

Đo VM → **PA4** (C10 100 nF tại PA4).

### A2. Cầu H & gate driver — như F1 V2 (Q1–Q4, U1, U2, D3–D8, R5–R17, C11, C13, C15–C18, TH1)

R15, R16 kéo xuống **IN_A (PA8), IN_B (PA9)**. TH1 → **PA3**. Bootstrap: **C11, C13 = 10 µF/25 V 0805** (bỏ C12, C14); VCC gate driver: **C15, C16 = 10 µF/25 V 0805**. U1, U2 = IRS2104STRPBF; Q1–Q4 = BSC014N06NS.

### A3. Rail gate +12VG — như F1 V2 (U3, L1, D9, C19–C22, R18–R21)

### A4. Mạch SD / ENA — như F1 V2 (Q8, R22–R28, C23, JP1)

### A5. Đo & cắt dòng

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| R29 | M05 | Shunt 2 mΩ | 1 | Đo Kelvin |
| R30 | M02 | 1 kΩ | 1 | Kelvin shunt+ → PB0 |
| C25 | M06 | 100 pF | 1 | PB0 → GND, sát chân |
| R69 | M04 | 100 kΩ | 1 | VDDA → PB0 (offset +32,7 mV) |
| — | — | Đường Kelvin shunt− → PB2 | — | Không linh kiện |

Bỏ so với F1: U5 INA181, U6 TLV7011, C24, C26, C27, R31, R32, R33, R34, R35, C28, C29.

### A6. Nguồn logic 5 V / 3,3 V — như F1 V2 (FB1, C30–C38, U7, D11, D12, L2, R36–R41, U8, LED1, FB4)

FB4 cấp +5V cho encoder. **C36, C37 (vào/ra LDO) = 10 µF/25 V 0805.** **U7 = LMR51606XDBVR**, EN nối thẳng VIN; bỏ D10 (buck đồng bộ) và R74. FB giữ (10 k + 1 k + 1 k) / (1 k + 1 k) → 0,8 × 7 = 5,6 V trước Schottky OR (~5,2 V sau).

### A7. MCU

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| U9 | M22 | STM32G431CBT6 | 1 | |
| Y1, C57, C58 | — | Thạch anh + 2 tụ tải | DNP | Footprint trên PF0/PF1, không lắp |
| C39–C42 | M07 | 100 nF | 4 | VDD ×3 + VBAT (kiểm số chân VDD của gói) |
| FB2 | M26 | Ferrite | 1 | VDDA |
| C43, C44 | M09 + M07 | 10 µF + 100 nF | 2 | VDDA |
| C59, C60 | M09 + M07 | 10 µF + 100 nF | 2 | VREF+ |
| R42 | M03 | 10 kΩ | 1 | PB8-BOOT0 kéo xuống |
| C45 | M07 | 100 nF | 1 | NRST |
| SW1–SW3 | M28 | Nút | 3 | RST, BOOT0 (lên 3V3 qua R44), USER (PC14) |
| R44 | M02 | 1 kΩ | 1 | Nối tiếp nút BOOT0 |
| LED2, R45 | M17 + M02 | LED + 1 kΩ | 2 | STATUS (PB6) |

Bỏ so với F1: R43 (BOOT1), Y1 resonator.

### A8. USB (native, không cách ly)

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| J5 | M29 | USB-C 16P | 1 | |
| R46–R49 | M03 | 10 kΩ | 4 | 2 × 10 kΩ song song mỗi CC |
| FB3 | M26 | Ferrite | 1 | VBUS |
| U10 | M24 | SRV05-4 | 1 | ESD D+/D−, VDD → VBUS |
| R70, R71 | M03 | 10 kΩ / 10 kΩ | 2 | Chia áp VBUS → PC13 |

Bỏ so với F1: U11 CH340C, C46.

### A9. Encoder — như F1 V2 (U12, C47, R50–R53, C48, C49, U13)

### A10. Lệnh, I/O, UART, I2C

Như F1 V2 (R54, R55, C50, C51, R56, C52, R57–R64, C53–C56, U14, R65–R68, U15, U16). Gán lại chân: STEP → PB4, DIR → PB5, IO1 → PB3, IO2 → PA10, IO3 → PB12, IO4 → PB13, I2C → PA15/PB7, UART → PB10/PB11, AIN → PA2.

### A11. Tổng hợp

| Nhóm | Số mã | SL |
| --- | --- | --- |
| Điện trở + shunt | 5 | 65 |
| Tụ | 5 | 50 |
| Bán dẫn rời (MOSFET, AO3400, diode, TVS, LED) | 6 | 18 |
| IC | 7 | 12 |
| Cuộn, ferrite, NTC, nút, USB-C | 5 | 11 |
| **Tổng** | **28** | **~156** |

Mã chắc chắn Extended ở JLCPCB: MOSFET, IRS2104S, MT3608, LMR51606, STM32G431CBT6, 74LVC2G14, SRV05-4, tụ 220 µF, shunt 3 W (F1: 12 mã → G4: 9 mã). Kiểm lại từng mã khi lên BOM thật.
