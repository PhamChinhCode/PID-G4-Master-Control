# Thiết kế: Board driver đơn 1 động cơ encoder — F1_PID_Driver_V2

Nov 5, 2026 · @pham chinh · Trạng thái: **bản nháp thiết kế (giai đoạn 0 – chốt spec)**

Tài liệu gốc: [KeHoach_G4_Dual_Driver_V2.md](../Docs/KeHoach_G4_Dual_Driver_V2.md) (chống nhiễu, giao thức, firmware) và [REVIEW_F1_PID_Driver_V1.md](../F1_PID_Driver_V1/REVIEW_F1_PID_Driver_V1.md) (các lỗi đã gặp). Board này là **bản kế nhiệm của F1_PID_Driver_V1**: vẫn STM32F103, một động cơ, gate driver bootstrap kiểu IR2104 như V1 nhưng sửa toàn bộ lỗi đã review, MCU chung GND với công suất (bỏ opto), có đo dòng và cắt dòng phần cứng.

Lịch sử quyết định:
- Bản nháp `ThietKe_G4_Single_Driver_V1.md` → đổi sang **STM32F103CBT6**, **15 A liên tục**, **pin 3S–6S**, **I2C + UART + I/O**, **pad hàn**, **4 lỗ M2 ở 4 góc**, USB không cách ly.
- Nov 5, 2026: **bỏ DRV8701E, thay bằng 2× EG2104 + INA181 + comparator** để giảm ~25–33k/board (mục 4.2).
- Nov 5, 2026: **15 A liên tục trong hộp kín** → thêm đường tản nhiệt xuống vỏ/tấm nhôm (mục 6.3); **bỏ CAN**; **4 I/O mức 3,3–5 V**; **thiết kế cho F103CBT6, giữ khả năng thay chip tương đương** (mục 3.1).
- Nov 5, 2026: **chốt buck MP2459** (55 V, dư áp cho gai ~42 V của VM). Đã xét và loại: MP2303 (28 V), LMR16006X (khó mua, đắt), XL7005A (150 kHz → cần cuộn lớn, thêm mã), SCT2401 (40 V → cần thêm R + TVS cục bộ). Nguồn thứ hai cùng chân: MSKSEMI MP2459GJ-Z-MS (LCSC C45367195) — kiểm V_FB, thứ tự chân và thời gian bật tối thiểu (~2 MHz) trước khi dùng.
- Nov 5, 2026: **Q8 đổi 2N7002 → AO3400** (cùng footprint SOT-23, dẫn hoàn toàn ở 3,3 V, phổ biến hơn).
- Nov 5, 2026: **gom mã linh kiện** từ ~69 xuống **33 mã** (Phụ lục A): 4 giá trị điện trở, 6 giá trị tụ, resonator thay thạch anh, gate Q5 lấy từ 12VG (bỏ zener), bỏ mạch auto-boot, SRV05-4 thay USBLC6.
- Nov 5, 2026: **bỏ 74HCT08** — TIM1_CH1 (PA8) và TIM1_CH1N (PB13) lái thẳng IN_A/IN_B, SD chỉ được kéo lên từ ENA, MCU tắt cầu qua AO3400 (mục 4.3). Điều kiện: EG2104 có V_IH ≤ 2,5 V.

---

## 1. Mục tiêu & thứ tự ưu tiên

Board điều khiển vòng kín (dòng → tốc độ → vị trí) cho **một** động cơ DC có encoder, đặt **trong hộp kín** trên robot chạy pin 3S–6S, nối với bộ điều khiển chủ qua UART / I2C / Step-Dir / USB.

1. **An toàn phần cứng** (không thương lượng): mọi MOSFET tắt khi MCU reset/treo; comparator cắt dòng từng chu kỳ qua TIM1_BKIN; ENA ngắt SD bằng diode (không qua firmware); TVS + tụ ceramic sát cầu H; chống ngược cực; giảm công suất theo nhiệt.
2. **Giá thành**: mục tiêu **≤ 320.000 đ/board** (lô 10, linh kiện + PCB + SMT; chưa gồm tấm tản nhiệt/vỏ).
3. **Kích thước**: mục tiêu **40 × 62 mm**, SMT một mặt (mặt dưới để trống làm mặt tản nhiệt), không connector công suất.

## 2. Thông số mục tiêu

| Thông số | Giá trị | Ghi chú |
| --- | --- | --- |
| Nguồn VM | **Pin 3S–6S: 9,0–25,2 V** | Gate driver nuôi bằng rail 12 V riêng (boost từ 5 V) |
| Dòng động cơ | **15 A liên tục trong hộp kín**, cắt phần cứng **30 A** (chỉnh được) | Chỉ đạt khi mặt dưới board ép qua pad tản nhiệt cách điện xuống vỏ nhôm/tấm nhôm (mục 6.3). Không có đường tản nhiệt → hạ còn ~8–10 A |
| Nhiệt độ môi trường | Giả định ngoài hộp ≤ 40 °C | Cần xác nhận (mục 14) |
| PWM | 20 kHz (cấu hình được 16–25 kHz), kiểu PH/EN, **duty tối đa 97 %** | Giới hạn do bootstrap |
| MCU | **STM32F103CBT6** (LQFP48, Cortex-M3 72 MHz, 128 KB Flash / 20 KB RAM) | Giữ chân tương thích GD32F103CBT6 / APM32F103CBT6 (mục 3.1) |
| Clock | **Resonator gốm 8 MHz có tụ tích hợp** (±0,5 %) → 72 MHz | Không còn CAN nên không cần thạch anh; vẫn cần HSE vì HSI chỉ cho tối đa 64 MHz và trôi ±2–3 % khi nóng trong hộp kín → UART tốc độ cao dễ lỗi. Resonator thay thạch anh + 2 tụ tải (bớt 2 mã) |
| Encoder | 1× A/B, 3,3 V hoặc 5 V, Hall 13–16 xung đến quang 1000 xung/vòng | |
| Truyền thông | **UART** 3,3 V (chịu 5 V), **I2C** 3,3 V, **USB-C** (CH340C → USART1, không cách ly) | **Không có CAN** |
| Ngõ vào lệnh | Step/Dir hoặc xung RC (3,3–5 V), analog 0–3,3 V | |
| I/O | **ENA** + **4 I/O đa dụng, mức logic 3,3–5 V** | Cả 4 I/O trên chân FT; ngõ ra 3,3 V push-pull hoặc open-drain kéo lên 5 V phía ngoài |
| Đấu nối | **Pad hàn trực tiếp**: VM+, VM−, M+, M− pad lớn có lỗ; tín hiệu hàng pad 2,54 mm | |
| Lắp cơ khí | **4 lỗ M2 ở 4 góc**, NPTH | Ốc M2 đồng thời ép board xuống pad tản nhiệt |
| PCB | **4 lớp, đồng ngoài 2 oz**, L2 GND liền | |
| Gia công | JLCPCB PCB + SMT, ưu tiên linh kiện Basic | |

Ngoài phạm vi: CAN, nhiều trục, BLDC/step, cách ly.

## 3. Hệ quả của việc chọn STM32F103

| Đặc điểm F103 | Ảnh hưởng | Cách xử lý trong thiết kế |
| --- | --- | --- |
| ROM bootloader chỉ nạp qua **USART1** (không có USB DFU) | PA9/PA10 bị giữ cho CH340 → **không dùng được TIM1_CH2/CH3 làm PWM** | CH340C nối USART1; vào bootloader bằng lệnh firmware (nhảy vào ROM bootloader) hoặc nút BOOT0 + RST. Cầu H chỉ dùng **một kênh TIM1_CH1**: PA8 (CH1) → IN_A, PB13 (CH1N) → IN_B, đổi chiều bằng thanh ghi CCER (mục 4.3) |
| Có USB FS native (PA11/PA12) nhưng cần bootloader tự viết | | **Không dùng.** CH340 + ROM bootloader không thể "brick", và chạy y hệt trên GD32/APM32. PA11/PA12 dùng làm I/O chịu 5 V |
| **Không có DAC** | | Ngưỡng comparator = PWM (TIM3_CH4) + lọc RC 2 bậc, có R kéo xuống (mục 6.1) |
| **Không có comparator, op-amp** | | INA181A1 (×20) + comparator ngoài → TIM1_BKIN. Lớp 2: ADC analog watchdog |
| **Không FPU**, 72 MHz | | Vòng điều khiển **fixed-point (Q15/Q31)**; dòng 10–20 kHz, tốc độ 1 kHz, vị trí 500 Hz–1 kHz |
| Timer 16 bit | | Mở rộng encoder bằng phần mềm (ngắt tràn) |
| PA0–PA7, PB0, PB1, PB5 **không chịu 5 V** | | Encoder qua **74LVC2G14**; mọi tín hiệu 3,3–5 V từ ngoài (I/O, STEP/DIR, ENA, UART, I2C) chỉ vào chân FT |
| Ngoại vi I2C có errata | I2C slave dễ kẹt | Timeout + reset ngoại vi I2C |

### 3.1 Giữ khả năng thay chip tương đương (GD32F103CBT6, APM32F103CBT6)

Thiết kế cho F103CBT6 nhưng không dùng gì làm khó việc thay chip sau này:

- **Phần cứng**: footprint LQFP48 chuẩn; VDD/VDDA tách như F103 (đủ cho GD32/APM32); HSE 8 MHz bằng resonator (mọi bản tương đương đều hỗ trợ); BOOT0/BOOT1 kéo xuống; NRST 100 nF; tụ VDD đặt đủ cả 3 cặp chân. Không dùng ngoại vi mà bản tương đương làm khác (USB native, CAN, RTC/VBAT).
- **Nạp firmware**: chỉ dựa vào ROM bootloader **USART1** — có trên cả GD32F103 và APM32F103.
- **Firmware**: tầng HAL/LL tách riêng (một file cấu hình clock + flash wait state cho mỗi chip); không phụ thuộc timing tuyệt đối của HSE startup (GD32 khởi động HSE chậm hơn → timeout dài hơn); không dựa vào errata riêng của ST.
- Khi đổi chip: chạy lại bài thử clock, ADC (sai số, thời gian lấy mẫu), PWM/BKIN, USART1 bootloader.

## 4. Kiến trúc

### 4.1 Sơ đồ khối

```
 ┌──────────────── VÙNG CÔNG SUẤT ─────────────────┐  ┌──────────────── VÙNG LOGIC ─────────────────┐
 │ pad VM+ ─┬─ TVS SMCJ26A                           │  │ USB-C ─ SRV05-4 ─ CH340C ─ USART1 (PA9/PA10)│
 │          ├─ 3× 220µF/50V + 4× 4,7µF/50V           │  │   nạp: lệnh GUI → nhảy ROM, hoặc BOOT0 + RST │
 │          ├─ nửa cầu A: Q1/Q2 ◄─ EG2104 #A ◄─ IN_A ┼──┤◄─ PA8  TIM1_CH1  (chạy thuận: PWM)          │
 │          ├─ nửa cầu B: Q3/Q4 ◄─ EG2104 #B ◄─ IN_B ┼──┤◄─ PB13 TIM1_CH1N (chạy nghịch: PWM)         │
 │          │       └── SD (cả hai) ◄──────────── SD ┼──┤◄─ pad ENA qua 10 kΩ (SD chỉ cao khi ENA cao)│
 │          │   M+ = SH_A, M− = SH_B ──► pad M+/M−   │  │     Q8 AO3400 kéo SD = 0 ◄─ nDIS (PB14,     │
 │          │                                        │  │       kéo lên 3V3: reset = tắt); ENA → PB15  │
 │          └─ Buck MP2459 → 5,67V ─[Schottky]─ +5V ─┼──┤ +5V ─ LDO ─ 3V3 ─ MCU, CH340                │
 │             +5V ─ boost MT3608 → +12VG (gate)     │  │ +5V ─ ferrite ─ encoder                      │
 │ shunt 2 mΩ ─ INA181A1 ─ SO ───────────────────────┼──► PA4 ADC                                    │
 │                         └─ comparator ◄─ VREF_OC ◄┼─── PB1 TIM3_CH4 PWM + RC                      │
 │                               └─ nOC ─────────────┼──► PB12 TIM1_BKIN (AOE = cắt từng chu kỳ)      │
 │ chia áp VM → PA5, NTC → PA3                       │  │ ENC A/B ─ RC ─ 74LVC2G14 ─ PA0/PA1 (TIM2)   │
 │ pad VM− ─ Q5 N-FET chống ngược cực ─ GND (sao)    │  │ STEP/DIR ─ RC ─ PB6/PB7 (TIM4, FT)          │
 │                                                   │  │ I2C PB8/PB9 (remap), UART PB10/PB11 (USART3)│
 │ mặt dưới: đồng hở + via → pad tản nhiệt → vỏ nhôm │  │ IO1–IO4: PA15, PB3, PB4, PA11 (FT) + ESD    │
 └───────────────────────────────────────────────────┘  │ AIN PA2                                      │
                                                        └─────────────────────────────────────────────┘
```

### 4.2 Các quyết định chính

| Quyết định | Chọn | Lý do / phương án bị loại |
| --- | --- | --- |
| Gate driver | **2× EG2104S** (SOP-8, half-bridge 1 input + SD, dead-time nội, bootstrap); thay thế cùng chân: FD2104 hoặc bản khác **có V_IH ≤ 2,5 V** | Rẻ hơn DRV8701E ~25–33k/board kể cả linh kiện phụ. FD6288Q cần 4 PWM, đụng PA9 của bootloader. IR2104S (V_IH ~3,0 V) chỉ dư 0,3 V với mức 3,3 V → không dùng trực tiếp |
| Rail gate | **+12VG bằng boost MT3608 từ +5V** | Pin 3S cạn còn 9 V, sát UVLO họ IR2104 nếu nuôi thẳng từ VM |
| Logic điều khiển | **Không có IC logic**: TIM1_CH1 (PA8) → IN_A, TIM1_CH1N (PB13) → IN_B; SD lấy mức từ ENA qua 10 kΩ, MCU tắt bằng AO3400 (Q8) | Bỏ 74HCT08: −3–5k/board, −1 IC, giải phóng PB0. Không dùng open-drain kéo lên 5 V cho IN/SD: khi reset chân Hi-Z sẽ bị kéo lên → bật high-side (lỗi S5 của V1) |
| Đo dòng | Shunt 2 mΩ low-side + **INA181A1** (×20) | |
| Cắt dòng | **Comparator** open-drain → **TIM1_BKIN**, **AOE = 1** | Cắt dòng từng chu kỳ |
| MOSFET | 4× 60 V PDFN 5×6, **R_DS(on) max ≤ 1,6 mΩ @10 V** | Siết từ ≤ 2 mΩ vì hộp kín; 60 V vì TVS kẹp ~42 V |
| Tản nhiệt | **Mặt dưới đồng hở dưới FET/shunt + via nhiệt → pad silicone cách điện → vỏ nhôm / tấm nhôm** | Không khí trong hộp kín không đủ mang ~4 W ra ngoài (mục 6.3) |
| Chống ngược cực | **N-MOSFET low-side** (Q5, cùng loại FET cầu H) | Dẫn hai chiều → hãm tái sinh trả về pin |
| Cầu chì | **Ngoài board**, trên dây pin (blade/ATO 20 A) | |
| USB | **CH340C → USART1**, không cách ly | ROM bootloader, chạy trên mọi chip tương đương |
| I/O 3,3–5 V | 4 chân **FT** + R 1 kΩ + ESD SRV05-4 (VDD ESD = 5 V) | PB5 (không FT) bị loại khỏi I/O, thay bằng PA11 (giải phóng do bỏ CAN) |

### 4.3 Điều khiển cầu H bằng một kênh TIM1 (CH1 + CH1N), không IC logic

EG2104: IN = 1 → HO bật; IN = 0 → LO bật; SD = 0 → cả hai tắt. Dead-time và chống trùng dẫn nằm trong từng EG2104, nên **không tổ hợp IN nào gây shoot-through**.

PA8 (TIM1_CH1) nối thẳng IN_A, PB13 (TIM1_CH1N) nối thẳng IN_B. Cả hai ra cùng một OC1REF; chọn chân nào băm PWM bằng bit CCER. Dead-time TIM1 đặt **DTG = 0** (EG2104 đã có dead-time nội). **OSSR = 1, OSSI = 1, OIS1 = OIS1N = 0** để chân không dùng và chân ở trạng thái break luôn được lái mức 0.

| SD | CC1E | CC1NE | CC1NP | IN_A (PA8) | IN_B (PB13) | Trạng thái |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | x | x | x | x | x | **Cả 4 FET tắt** (thả trôi) — khi reset / ENA thấp / nDIS (PB14) = 1 |
| 1 | 1 | 0 | 0 | PWM | 0 | Chạy thuận: A băm PWM, Q4 dẫn liên tục, slow decay qua Q2–Q4 |
| 1 | 0 | 1 | **1** (đảo cực → CH1N cùng pha OC1REF) | 0 | PWM | Chạy nghịch |
| 1 | 0 | 0 | — | 0 | 0 | Phanh ngắn mạch qua hai low-side (chỉ dưới ~20 % tốc độ) |
| 1 | 1 | 1 | 1 | PWM | PWM | Cấu hình sai: hai nửa cầu cùng pha → không có áp lên động cơ, **an toàn** |
| 1 | 1 | 1 | 0 | PWM | /PWM | Cấu hình sai: kiểu locked anti-phase → động cơ chạy theo duty − 50 %, **không shoot-through** |

**SD = ENA AND (MCU cho phép), không IC logic:**
- Net SD (chung cho hai EG2104) **chỉ được cấp mức cao từ chính ENA**: pad ENA → R 1 kΩ (bảo vệ) → nút ENA (R 10 kΩ kéo xuống GND) → R 10 kΩ → SD; SD có R 100 kΩ kéo xuống GND.
- MCU tắt cầu bằng **Q8 AO3400**: drain → SD, source → GND, gate ← PB14 (**nDIS**) qua 1 kΩ, gate kéo **lên** 3V3 bằng 10 kΩ. PB14 = 0 → cho phép; PB14 = 1 hoặc Hi-Z → Q8 dẫn → SD = 0. AO3400 (V_GS(th) 0,65–1,45 V) dẫn hoàn toàn ở 3,3 V; C_iss ~0,7 nF → reset tắt cầu sau ~15–20 µs (τ = 10 kΩ × C_iss), PB14 tắt qua 1 kΩ trong < 1 µs. Đặt Q8 + R 1 kΩ sát nhau, xa nút SH (ngưỡng thấp → nhiễu gate có thể gây dừng ngoài ý muốn, nghiêng về an toàn).
- ENA thấp hoặc đứt dây: nút ENA bị kéo xuống 10 kΩ, SD không có nguồn nào kéo lên → SD = 0 → cả 4 FET tắt, firmware không can thiệp được.
- MCU reset (PB14 Hi-Z): pull-up 10 kΩ bật Q8 → SD = 0.
- ENA cao 3,3–5 V và PB14 = 0: SD ≈ ENA × 100k / 110k ≈ 3,0–4,5 V ≥ V_IH ✔ (SD của EG2104 chịu tới VCC 12 V). Bộ điều khiển chủ cấp ~0,5 mA vào ENA.
- Không dùng ENA: jumper hàn JP1 nối nút ENA lên +5V qua 1 kΩ (nút ENA ≈ 4,5 V).
- (Mạch AND bằng diode kiểu "PB14 → R → SD, diode SD → ENA" **không dùng**: khi đứt dây ENA chỉ còn R kéo xuống phía ENA, SD vẫn lên ~3 V.)

Khi comparator kích BKIN, MOE = 0 → PA8 và PB13 về idle 0 → IN_A = IN_B = 0 → dòng xả qua hai low-side; AOE = 1 bật lại ở chu kỳ PWM sau (cắt dòng từng chu kỳ).

Đổi chiều: duty về 0 → đổi CC1E/CC1NE/CC1NP tại sự kiện update (preload CCPC/COMG nếu cần) → tăng duty lại. Bootstrap: trước khi chạy, firmware giữ SD = 1 với CC1E = CC1NE = 0 (IN_A = IN_B = 0) trong ≥ 1 ms để nạp tụ bootstrap; duty kẹp ≤ 97 %.

**Điều kiện bắt buộc**: V_IH của IN và SD trên EG2104 ≤ 2,5 V (chân F103 ra ~3,3 V khi tải µA, đảm bảo ≥ 2,9 V). V_IL của SD ≥ 0,8 V (Q8 kéo SD xuống < 0,1 V). Nếu EG2104 không đạt → chọn gate driver cùng chân đạt, hoặc thêm lại một bộ đệm 5 V (HCT) chỉ để chuyển mức.

### 4.4 Những gì mất đi so với DRV8701 — và cách bù

| Mất | Hậu quả | Bù |
| --- | --- | --- |
| VDS overcurrent | M+ hoặc M− chạm GND: shunt low-side không thấy | Cầu chì 20 A trên dây pin; firmware phát hiện "VM sụt mạnh, SO ≈ 0" → tắt SD |
| Duty 100 % | Tốc độ tối đa giảm ~3 % | Chấp nhận |
| UVLO, cảnh báo nhiệt tích hợp | | UVLO của EG2104 trên 12VG; NTC; firmware đo VM |
| nFAULT | | Ngõ ra comparator (nOC) → BKIN |

### 4.5 USB không cách ly — điều kiện dùng

- USB-C và CH340C ở mép **vùng logic**, xa cầu H; dòng động cơ không đi qua GND USB (GND sao, mục 9).
- SRV05-4 (VDD nối VBUS) trên D+/D−; ferrite trên VBUS; VBUS cấp logic qua Schottky → cắm USB là cấu hình/nạp được khi chưa có pin.
- CH340C chạy 3,3 V (V3 nối VCC).
- Tune ở dòng lớn: laptop chạy pin hoặc USB isolator rời.

## 5. Linh kiện chính

| Khối | Linh kiện | Ghi chú thiết kế |
| --- | --- | --- |
| MCU | STM32F103CBT6 (LQFP48) | 100 nF mỗi VDD + 4,7 µF chung; VDDA qua ferrite + 1 µF + 100 nF; **VBAT → 3V3 + 100 nF**; **BOOT1/PB2 → 10 kΩ GND**; BOOT0 → 10 kΩ GND + nút; NRST 100 nF + nút |
| HSE | Resonator gốm 8 MHz có tụ tích hợp (vd Murata CSTNE8M00G55), −40…+85 °C | Không cần tụ tải riêng |
| Gate driver | **2× EG2104S** (SOP-8) | VCC = +12VG, 1 µF + 100 nF sát chân; COM nối source low-side của nửa cầu đó. **Kiểm datasheet**: **V_IH IN/SD ≤ 2,5 V** (bắt buộc, vì IN/SD nối thẳng MCU 3,3 V), dòng ra, dead-time, UVLO, nhiệt độ làm việc |
| Bootstrap | Mỗi nửa cầu: **1N4148W** (100 V) + 10 Ω nối tiếp + **2 × 1 µF/50 V X7R 0603** song song | Mục 6.2 |
| Mạch gate | Mỗi FET: R gate **10 Ω** 0603 + diode 1N4148W song song (tắt nhanh) + R **10 kΩ** G–S | 10 Ω dùng chung mã với lọc 12VG; điện trở nội driver họ IR2104 đã ~45–90 Ω nên 10 Ω chỉ chậm thêm ~10–15 % |
| Rail gate | **MT3608** 5 V → **12,6 V** + 22 µH + SS36 + 2 × 10 µF/25 V; FB (10 k + 10 k) / 1 k; lọc 10 Ω + 10 µF trước VCC driver | |
| Mạch SD / ENA | **Q8 AO3400** (SOT-23) + R 10 kΩ ENA→SD + R 100 kΩ SD↓ + R 10 kΩ kéo xuống nút ENA + R 10 kΩ kéo lên gate Q8 + R 1 kΩ gate | Không IC logic. R 10 kΩ kéo xuống tại IN_A/IN_B (PA8, PB13) |
| MOSFET cầu H | 4× N-ch 60 V PDFN 5×6, **R_DS(on) max ≤ 1,6 mΩ @10 V**, Q_gd ≤ 15 nC, Tj max 150–175 °C | Tham chiếu: TPH1R306PL (max ~1,3–1,6 mΩ, kiểm datasheet); hàng nội địa phải đạt cùng R_DS **max**, không phải typ |
| Chống ngược cực | Q5 cùng loại; **gate lấy từ +12VG** qua 1 kΩ, 100 kΩ G–S (không cần zener) | Low-side nên +12VG cho V_GS ≈ 12,5 V ở mọi mức pin; ngược cực → board không có nguồn → Q5 tắt; mất 12VG → EG2104 UVLO tắt cầu → không có dòng qua thân diode |
| Shunt | 2 mΩ 2512 **3 W**, đo Kelvin, hệ số nhiệt ≤ 100 ppm/°C | Nóng trong hộp → chọn TCR thấp |
| Khuếch đại dòng | **INA181A1** (×20, SOT-23-6) | REF → GND. Đặt sát shunt |
| Comparator | TLV7011 / LMV331 (open-drain), nuôi 3V3 | IN− = SO, IN+ = VREF_OC; **không R trễ** (BKIN chốt đến chu kỳ sau); pull-up 10 kΩ → PB12 |
| TVS | **SMCJ26A** | |
| Tụ VM | 3× 220 µF/50 V **105 °C**, low-ESR hoặc polymer + 4× 4,7 µF/50 V X7R 1210 + 100 nF | Hộp kín: tụ điện phân 105 °C, ≥ 5000 h; kiểm ripple current ~7,5 A rms |
| Buck 5 V | MP2459 + Schottky SS36 + cuộn 22 µH | FB (10 k + 1 k + 1 k) / (1 k + 1 k) → 5,67 V (V_FB 0,81 V, kiểm datasheet); lọc vào π ferrite + 2 × 4,7 µF/50 V |
| LDO 3,3 V | AP2112K-3.3 / ME6211C33 | Chỉ sau rail 5 V |
| OR nguồn 5 V | 2× SS36 | +5V ≈ 5,25 V khi chạy pin, ≈ 4,6 V khi chỉ có USB |
| USB | USB-C 16P + CC 2 × (10 kΩ ‖ 10 kΩ) + SRV05-4 + CH340C | Không mạch auto-boot: firmware nhảy vào ROM bootloader theo lệnh GUI |
| I2C | Pull-up 10 kΩ lên 3V3 qua jumper hàn, không R nối tiếp | 100 kHz dây ngắn; 400 kHz → master thêm pull-up mạnh hơn.  Chân FT → bus 5 V có pull-up phía ngoài vẫn dùng được (khi đó tháo jumper) |
| UART ngoài | USART3 PB10/PB11 (FT), R 1 kΩ nối tiếp | Nhận 3,3–5 V; ngõ ra TX 3,3 V (đủ cho ngưỡng TTL của mạch 5 V) |
| **4 I/O 3,3–5 V** | Mỗi kênh: R 1 kΩ nối tiếp + 100 pF + pull-up 10 kΩ lên 3V3 (DNP được); ESD **SRV05-4** (VDD = +5V) cho IO1–IO4 | Vào: 3,3 V hoặc 5 V. Ra: push-pull 3,3 V, hoặc **open-drain** + pull-up 5 V phía ngoài (tháo pull-up 3V3) |
| STEP/DIR/ENA | R 1 kΩ + 100 pF vào chân FT; ESD SRV05-4 thứ hai (dùng chung với 2 kênh encoder) | ENA cấp mức cho SD (mục 4.3), đọc lại ở PB15 |
| Encoder | 74LVC2G14 + RC 1 kΩ/100 pF + pull-up 1 kΩ (jumper 3,3/5 V) + ferrite trên nguồn 5 V | Không polyfuse: chập nguồn encoder → buck tự giới hạn dòng → MCU brown-out → SD tắt |
| Pad công suất | 4 pad 5 × 10 mm, mỗi pad 2 lỗ mạ 1,6–2,0 mm | Dây silicon 12–14 AWG |
| Pad tín hiệu | Hàng pad 2,54 mm, lỗ 1,0 mm | |
| SWD | 4 pad + NRST | |
| Tản nhiệt (ngoài BOM SMT) | Pad silicone cách điện 3–6 W/m·K, dày 0,5–1,0 mm, điện áp đánh thủng ≥ 1 kV; vỏ nhôm hoặc tấm nhôm 2 mm | Cắt theo vùng công suất ~25 × 40 mm |

## 6. Tính toán thiết kế

### 6.1 Đo dòng và cắt dòng phần cứng

- SO = 20 × I × R_sense = 0,04 V/A. Tại 15 A: SO = 0,6 V.
- ADC đo tới 3,3 V / 0,04 = **82 A** (~20 mA/LSB).
- I_trip = VREF_OC / 0,04. I_trip = 30 A → **VREF_OC = 1,20 V**.
- VREF_OC: PWM TIM3_CH4 (PB1) ~70 kHz qua **2 tầng RC 10 kΩ / 1 µF** (fc ≈ 16 Hz) + **R 100 kΩ kéo xuống** (hệ số DC 0,83: VREF_OC = 0,83 × D × 3,3 V → 1,2 V ở D = 43,6 %) → MCU reset thì VREF_OC → 0 → comparator giữ BKIN tích cực (an toàn hai lớp cùng với SD).
- Trễ phản ứng ≈ **2–3 µs** (INA181 ~350 kHz + comparator ~1 µs + BKIN).
- RC 1 kΩ + 100 pF (τ = 100 ns) trước comparator và trước PA4. Không có R trễ: khi comparator lật, BKIN tắt PWM đến hết chu kỳ (AOE bật lại ở update) nên dao động quanh ngưỡng không gây hại.
- Lớp 2: ADC analog watchdog ~35 A → ngắt → tắt SD.
- Shunt: 15² × 0,002 = 0,45 W; 30² × 0,002 = 1,8 W trong lúc chặt → 2512 **3 W**, hạ định mức theo nhiệt (mục 6.3).

### 6.2 Gate drive và bootstrap

- Dòng rail 12VG: 4 × 70 nC × 20 kHz + tĩnh ≈ 7 mA → ~25 mA từ rail 5 V.
- C_boot ≥ 10 × 70 nC / 1 V = 0,7 µF → **2 × 1 µF X7R 50 V 0603** (~0,6–0,7 µF hiệu dụng mỗi tụ ở 12 V do DC bias → ~1,3 µF).
- Diode bootstrap **1N4148W** (V_R 100 V ≥ VM + gai 42 V; t_rr 4 ns) + **10 Ω** nối tiếp: giới hạn dòng nạp lần đầu ~1,2 A; ở duty 97 % sụt áp V_BS cân bằng ~0,5 V — chấp nhận được.
- Duty 97 % ở 20 kHz → thời gian tắt 1,5 µs ≥ dead-time (~0,5 µs, kiểm datasheet) + nạp bootstrap.
- Thời gian chuyển mạch: nếu dòng ra kiểu IR2104 (0,13/0,27 A) với R gate 10 Ω: t_r ≈ 130 ns, t_f ≈ 60 ns (tắt qua diode); nếu EG2104 mạnh hơn: ~50/25 ns.

### 6.3 Nhiệt trong hộp kín (VM = 25,2 V, I = 15 A, 20 kHz)

**Tổn hao** (FET R_DS max 1,6 mΩ → nóng ≈ 2,4 mΩ):

| Nguồn | Tính | Công suất |
| --- | --- | --- |
| Nửa cầu PWM (dẫn D + (1−D), cộng chuyển mạch kiểu IR2104) | 15² × 0,0024 + ½ × 25,2 × 15 × 170 ns × 20 kHz | 0,54 + 0,64 = **1,18 W** (tập trung ở high-side khi D cao) |
| Low-side nửa cầu tĩnh (dẫn liên tục) | 15² × 0,0024 | **0,54 W** |
| Q5 chống ngược cực | 15² × 0,0024 | **0,54 W** |
| Shunt | 15² × 0,002 | **0,45 W** |
| Đồng, pad, tụ (ESR gợn), nguồn logic | ước | **~0,8 W** |
| **Tổng trong hộp** | | **~3,5 W** |

**Không có đường tản nhiệt xuống vỏ** (chỉ dựa vào không khí trong hộp):
- Hộp nhựa kín cỡ 100 × 60 × 35 mm (diện tích ~0,03 m², h ≈ 6–8 W/m²·K): R ≈ 4–5 °C/W → không khí trong hộp nóng hơn ngoài **~15–20 °C** → 55–60 °C.
- FET nóng nhất 1,2 W × R_θJA 35–40 °C/W ≈ +45 °C → **Tj ≈ 100–105 °C**, shunt và tụ bulk ~80–90 °C. Quá sát giới hạn tin cậy (mục tiêu Tj ≤ 100 °C để FET, tụ, EG2104 sống lâu) → **không đạt 15 A liên tục**.

**Có đường tản nhiệt xuống vỏ nhôm / tấm nhôm** (phương án chọn):
- Mặt dưới board: **mở mask đồng hở** dưới 4 FET, Q5 và shunt; mỗi pad drain 9–12 via 0,3 mm (via-in-pad lấp keo nếu JLCPCB hỗ trợ miễn phí cho 4 lớp — kiểm lại; nếu không thì via sát mép pad, tented mặt trên).
- Pad silicone cách điện 1 mm, 3 W/m·K, vùng 25 × 40 mm: R ≈ 0,001 / (3 × 0,001) ≈ **0,3 °C/W** cho cả vùng; R hiệu dụng từ junction một FET xuống vỏ ≈ **12–18 °C/W** (R_θJC + đồng + via + pad).
- Vỏ nhôm cầm nhiệt ra ngoài: vỏ ≈ ngoài hộp + ~10 °C → 50 °C.
- **Tj FET nóng nhất ≈ 50 + 1,2 × 15 ≈ 70 °C** ✔; shunt ~65 °C ✔; không khí trong hộp ~50 °C → tụ, MCU, EG2104 thoải mái.
- Lưu ý: đồng hở mặt dưới thuộc các net khác nhau (VM+, M+, M−, GND) → **pad tản nhiệt bắt buộc cách điện**, phủ kín vùng đồng hở; ốc M2 4 góc ép board xuống, cột đỡ thấp hơn bề dày pad ~20–30 % để pad bị nén.
- Nếu vỏ là nhựa: thêm tấm nhôm 2 mm làm tấm trải nhiệt (giảm nhưng không bằng vỏ nhôm) → định mức thực tế ~10–12 A; xác nhận ở bài thử nhiệt.

**Giảm công suất theo NTC** (NTC đặt giữa các FET, đọc nhiệt board):
- < 85 °C: 100 % dòng; 85–100 °C: giảm tuyến tính về 50 %; > 105 °C: ngắt, báo `OVERTEMP`, tự chạy lại khi < 80 °C (cấu hình được).

**Giảm tổn hao thêm nếu đo thực tế còn nóng**: nối tắt R gate 10 Ω (thêm mã 0 Ω, kiểm ringing), PWM 20 → 16 kHz (chuyển mạch −20 %), chọn FET Q_gd thấp hơn.

### 6.4 Đường đồng 15 A

- IPC-2221 lớp ngoài: 5 mm đồng 2 oz ≈ 17 A (ΔT 20 °C). Trong hộp kín dùng ΔT 10 °C → **pour ≥ 7 mm song song L1 và L4**, khâu ≥ 20 via 0,3 mm mỗi đường.
- Đường pad VM → bulk → cầu H → shunt → Q5 → pad VM− ngắn và thẳng.

### 6.5 TVS và dải VM

- **SMCJ26A**: V_WM 26 V ≥ 25,2 V, V_C ≈ 42,1 V < 60 V FET ✔. Lưu ý V_BR của TVS giảm nhẹ khi nóng — vẫn > 25,2 V.
- Firmware: giảm dốc giảm tốc khi VM > 26,5 V; quá áp > 27,5 V; thấp áp < 9,0 V.

### 6.6 Nguồn logic

- Tải 5 V: F103 ~40 mA + CH340C ~15 mA + encoder ≤ 50 mA + boost 12VG ~25 mA + logic/LED ~10 mA → ~**140 mA**. MP2459 0,5 A ✔.
- LDO 3,3 V tải ~70 mA: ~0,12 W ✔.

### 6.7 Đo VM, nhiệt, analog

- Chia áp 100 kΩ / 10 kΩ → thang 36,3 V; tụ 100 nF tại PA5.
- NTC 10 kΩ B3950 giữa 4 FET, pull-up 10 kΩ → PA3.
- AIN (PA2): R 10 kΩ + 100 nF, dải 0–3,3 V; quá áp 5 V chỉ bơm ~0,14 mA vào diode kẹp nội (không cần BAT54S). ESD qua SRV05-4.

### 6.8 I/O 3,3–5 V

- Vào 5 V qua R 1 kΩ vào chân FT: không có dòng bơm vào MCU (FT chịu tới VDD + 4 V); R 1 kΩ giới hạn dòng khi ESD/gai.
- Ra push-pull 3,3 V: đủ cho ngưỡng TTL/HCT/LVC của mạch 5 V (V_IH ≤ 2 V); **không** đủ cho CMOS 5 V thuần (V_IH 3,5 V) → khi đó cấu hình open-drain, pull-up 4,7–10 kΩ lên 5 V phía ngoài và tháo pull-up 3V3 trên board (nếu để cả hai, dòng 0,17 mA chảy ngược vào 3V3 — vô hại nhưng không cần).
- ESD SRV05-4 với VDD = +5V: kẹp tín hiệu 5 V mà không kéo xuống.

### 6.9 Encoder

- Xấu nhất 150 kHz/kênh; RC 1 kΩ × 100 pF = 100 ns; pull-up 1 kΩ với cáp ~200 pF → τ ≈ 0,2 µs (encoder phải sink ≥ 5 mA ở 5 V). Bật ICF của TIM2.

## 7. Checklist schematic (trước ERC)

- [ ] **Trạng thái an toàn khi reset**: gate Q8 kéo **lên** 3V3 (PB14 Hi-Z → Q8 dẫn → SD = 0); SD kéo xuống 100 kΩ; PA8, PB13 kéo xuống 10 kΩ tại chân IN; nút ENA kéo xuống 10 kΩ → **4 FET tắt** khi MCU Hi-Z. Khi rail 3V3 chưa lên mà ENA đã cao: Q8 không được bật → SD có thể lên → IN_A = IN_B = 0 → hai low-side dẫn (phanh, không shoot-through); nếu cần "tắt hẳn" ở trạng thái này thì lấy pull-up gate Q8 từ +5V qua 10 kΩ / 10 kΩ chia đôi (2,5 V > V_GS(th) max 1,45 V của AO3400, < V_GS max 12 V ✔).
- [ ] **Rail 12VG chưa lên hoặc sụt**: UVLO EG2104 giữ HO/LO thấp; R 10 kΩ G–S giữ gate tắt; Q5 cũng tắt (gate lấy từ 12VG).
- [ ] **ENA phần cứng**: SD chỉ được kéo lên từ nút ENA (qua 10 kΩ) → ENA thấp/đứt dây thì SD = 0. JP1 nối nút ENA lên +5V qua 1 kΩ nếu không dùng ENA.
- [ ] **Không** dùng mạch AND bằng diode (PB14 → R → SD, diode SD → ENA): khi đứt dây ENA, SD vẫn lên ~3 V.
- [ ] PA8 và PB13 nối **thẳng** IN_A, IN_B (không R nối tiếp lớn); kiểm V_IH ≤ 2,5 V của EG2104.
- [ ] Comparator: SO → IN−, VREF_OC → IN+ → quá dòng kéo **thấp** PB12. Pull-up 10 kΩ lên 3V3.
- [ ] Bootstrap: anode → +12VG, cathode → VB; tụ VB–VS sát chân driver; diode ≥ 60 V.
- [ ] COM mỗi EG2104 nối source low-side **của nửa cầu đó**.
- [ ] VBAT → 3V3 + 100 nF; PB2/BOOT1 → 10 kΩ GND; BOOT0 → 10 kΩ GND.
- [ ] Không mạch auto-boot: DTR/RTS của CH340C để trống; BOOT0 kéo xuống 10 kΩ + nút lên 3V3 qua 1 kΩ.
- [ ] Mỗi LDO/buck/boost có tụ vào **và** ra đúng datasheet; tụ ra MT3608 ≥ 25 V.
- [ ] **Không tín hiệu ngoài nào vào PA0–PA7, PB0, PB1, PB5** (không FT). 4 I/O ở PA15, PB3, PB4, PA11.
- [ ] Họ logic ghi đúng: **74LVC2G14** (không phải 74HC/74LS).
- [ ] VDD của SRV05-4 nối **+5V** (không phải 3V3, nếu không sẽ kẹp tín hiệu 5 V).
- [ ] Điện trở nối VM đủ công suất; LED PWR nối 3V3.
- [ ] Tụ VM ghi rõ `50V`, tụ điện phân ghi `105°C`.
- [ ] Q5: **drain → VM− (pin), source → GND board** (diode thân dẫn từ GND board về cực âm pin khi đúng cực, chặn khi ngược cực; V_GS tính so với GND board); gate ← +12VG qua 1 kΩ, 100 kΩ G–S.
- [ ] PWR_FLAG cho VM, +12VG, +5V, +3V3, GND; No-Connect cho chân bỏ trống (PA6, PA7, PA12, PB0, PB5, PC15).

## 8. Gán chân MCU (STM32F103CBT6, LQFP48)

> Kiểm lại bằng STM32CubeMX. **SWJ = SWD only** để giải phóng PA15, PB3, PB4. **I2C1 remap** sang PB8/PB9. TIM1 dùng CH1 (PA8) + CH1N (PB13) + BKIN (PB12), không remap.

| Chức năng | Chân | Ngoại vi | 5 V? | Ghi chú |
| --- | --- | --- | --- | --- |
| Encoder A / B | PA0 / PA1 | TIM2_CH1/CH2 | ✗ | Qua 74LVC2G14 |
| AIN (lệnh analog) | PA2 | ADC_IN2 | ✗ | |
| NTC | PA3 | ADC_IN3 | ✗ | |
| SO (dòng) | PA4 | ADC_IN4 | ✗ | Từ INA181; injected, TIM1 kích giữa chu kỳ |
| Đo VM | PA5 | ADC_IN5 | ✗ | |
| Dự phòng analog | PA6, PA7 | ADC_IN6/7 | ✗ | Pad test |
| IN_A (PWM thuận) | PA8 | TIM1_CH1 | | → IN EG2104 #A; idle = 0, kéo xuống 10 kΩ |
| USB TX / RX | PA9 / PA10 | USART1 ↔ CH340C | | ROM bootloader |
| **IO4** | **PA11** | GPIO/EXTI | ✔ | Giải phóng do bỏ CAN |
| Dự phòng | PA12 | GPIO | ✔ | Pad test (có thể làm IO5) |
| SWDIO / SWCLK | PA13 / PA14 | SWD | | |
| IO1 | PA15 | GPIO/EXTI | ✔ | Mặc định HOME |
| Dự phòng | PB0 | ADC_IN8 | ✗ | Pad test (giải phóng do bỏ 74HCT08) |
| VREF_OC PWM | PB1 | TIM3_CH4 | ✗ | Lọc RC 2 bậc → comparator |
| BOOT1 | PB2 | — | | 10 kΩ GND |
| IO2 / IO3 | PB3 / PB4 | GPIO/EXTI | ✔ | Hành trình −/+ hoặc ngõ ra |
| Dự phòng | PB5 | GPIO | ✗ | LED thứ hai hoặc NC |
| STEP / RC | PB6 | TIM4_CH1 | ✔ | Clock ngoài TIM4; RC dùng capture |
| DIR (lệnh ngoài) | PB7 | GPIO (TIM4_CH2) | ✔ | |
| I2C SCL / SDA | PB8 / PB9 | I2C1 (remap) | ✔ | |
| UART TX / RX | PB10 / PB11 | USART3 | ✔ | |
| nOC (quá dòng) | PB12 | TIM1_BKIN | ✔ | Cực tính thấp, AOE = 1 |
| IN_B (PWM nghịch) | PB13 | TIM1_CH1N | | → IN EG2104 #B; idle = 0, kéo xuống 10 kΩ |
| nDIS | PB14 | GPIO | | → gate Q8 (1 = tắt cầu); gate kéo lên 3V3 |
| ENA đọc lại | PB15 | GPIO/EXTI | ✔ | |
| LED STATUS | PC13 | GPIO | | R 1 kΩ |
| Nút USER | PC14 | GPIO | | |
| Dự phòng | PC15 | GPIO | | |
| Resonator | PD0 / PD1 | HSE 8 MHz | | Chân giữa resonator → GND |
| BOOT0, NRST | chân 44, 7 | | | Nút |

## 9. Layout PCB

### 9.1 Stackup

**4 lớp**: L1 linh kiện + công suất (2 oz), L2 GND liền, L3 3V3/5V/12VG + tín hiệu chậm, L4 pour công suất + **vùng đồng hở tản nhiệt** (2 oz). **Không đặt linh kiện mặt dưới** (mặt dưới ép pad tản nhiệt).

### 9.2 Bố trí (nhìn từ trên, ~40 × 62 mm)

```
 ┌○───────────────────────────────── 62 mm ────────────────────────────────○┐
 │ [VM+][VM−]  TVS  [C][C][C]bulk   EG2104A ─ Q1 Q2 ─ [M+]                  │  ← mép công suất
 │   Q5          4,7µF×4  shunt+INA181          4,7µF×2                     │
 │  (ngược cực)  ← GND sao          EG2104B ─ Q3 Q4 ─ [M−]   NTC  buck  boost│ 40 mm
 │ ·············· không tín hiệu nào chạy ngang ranh giới này ············· │
 │ [USB-C] ESD CH340C   STM32F103 LQFP48  8MHz   Q8  comp  LDO   ESD        │
 │ ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ●  pad 2,54       │  ← mép tín hiệu
 └○────────────────────────────────────────────────────────────────────────○┘
   ○ = lỗ M2 NPTH (Ø2,2 mm, vùng cấm đồng Ø5 mm, tâm cách mép 3 mm)
   Mặt dưới: vùng đồng hở ~25 × 40 mm dưới Q1–Q5 + shunt → pad silicone → vỏ nhôm
```

Hàng pad tín hiệu (mỗi nhóm có GND riêng): `ENC: 5V GND A B` · `CMD: STEP DIR ENA GND` · `UART: TX RX GND` · `I2C: SCL SDA 3V3 GND` · `IO: 1 2 3 4 AIN GND` · `SWD: 3V3 DIO CLK RST GND`.

### 9.3 Quy tắc

1. **Vòng commutation trước tiên**: 2 × 4,7 µF + 100 nF giữa drain high-side và source low-side mỗi nửa cầu, < 3 mm; bulk ngay cạnh cầu H.
2. **Mỗi EG2104 sát nửa cầu của nó** (≤ 10 mm, không bắt chéo): HO song song VS, LO song song source/COM; R gate + diode sát chân G; tụ bootstrap và VCC sát chân driver.
3. **Shunt Kelvin**: đường cảm biến đi cặp về INA181 ngay cạnh; SO → PA4 và comparator qua RC sát chân nhận.
4. **GND sao** tại đầu âm shunt; COM driver về source low-side.
5. **Nhiệt**: FET, Q5, shunt gom một vùng để pad tản nhiệt phủ hết; via nhiệt dày dưới pad drain; vùng đồng hở mặt dưới có khe ≥ 0,5 mm giữa các net, không có via hở/lỗ sắc cạnh làm thủng pad; **NTC đặt giữa các FET**, không đặt cạnh tụ bulk. Tụ bulk, MCU, tụ điện phân đặt **xa** FET.
6. Nút SH nhỏ, xa encoder, resonator, analog và boost.
7. Net class: `Power` ≥ 3 mm / clearance 0,3 mm; `Gate` 0,3 mm; `Default` 0,15–0,2 mm.
8. 4 lỗ M2 **NPTH**, vùng cấm đồng Ø5 mm; pad cách mép ≥ 0,5 mm.
9. Silkscreen: nhãn mọi pad ở cả hai mặt; `9–25V 3S–6S`, `15A WITH HEATSINK`, `FUSE 20A ON BATTERY WIRE`.
10. Trước khi xuất: **Refill zones → DRC 0 lỗi thật → Gerber mới hơn file PCB** → `python -I scripts/kicad_review.py`.

## 10. Firmware

Giữ **giao thức JSON + lệnh `describe`**, GUI Web Serial và MCP server của V2; lớp điều khiển fixed-point cho Cortex-M3; tầng HAL/LL tách riêng để thay chip (mục 3.1).

- **Nền tảng**: HSE 8 MHz → 72 MHz; TIM1 CH1 20 kHz center-aligned, BKIN cực tính thấp, **AOE = 1**, idle = 0; ADC1 injected (SO) giữa chu kỳ + regular (VM, NTC, AIN) qua DMA; TIM3_CH4 cho VREF_OC; TIM2 encoder (mở rộng 32 bit); TIM4 Step/RC; IWDG; PVD.
- **Điều khiển cầu H**: TIM1 DTG = 0, OSSR = OSSI = 1, OIS1 = OIS1N = 0; thuận = CC1E, nghịch = CC1NE + CC1NP (mục 4.3); đổi chiều = duty về 0 → đổi CCER tại update event → chạy lại; nDIS (PB14) = 1 khi lỗi; duty ≤ 97 %; nạp bootstrap ≥ 1 ms trước khi bật; đếm số lần BKIN mỗi giây → `OVERCURRENT`.
- **Vòng điều khiển** (Q15/Q31): dòng PI 10–20 kHz → tốc độ PI 1 kHz → vị trí PID 500 Hz–1 kHz, profile hình thang.
- **Nguồn lệnh** (mỗi lúc một nguồn): USB (USART1), UART (USART3), I2C slave, Step/Dir, RC, analog.
- **I/O**: IO1–IO4 gán chức năng qua cấu hình (HOME, hành trình −/+, ngõ vào enable phụ, ngõ ra "đến vị trí"/lỗi); mỗi ngõ ra chọn push-pull hoặc open-drain.
- **Bảo vệ**: kẹt rotor, I²t, thấp/quá áp VM, **giảm công suất theo NTC (85/100/105 °C)**, timeout lệnh, ENA thấp → `ENA_OFF`, nghi ngắn mạch high-side → tắt SD. Phanh ngắn mạch chỉ dưới ~20 % tốc độ.
- **Nạp firmware**: GUI gửi lệnh `bootloader` → firmware tắt cầu, tắt ngắt, đặt lại MSP và nhảy vào ROM bootloader (F103: 0x1FFF F000 — kiểm địa chỉ cho GD32/APM32) → nạp qua USART1 bằng `stm32flash` / STM32CubeProgrammer. Dự phòng: giữ BOOT0 + nhấn RST.

## 11. BOM & chi phí ước tính (lô 10 board)

| Khối | Linh kiện | Ước tính (đ) |
| --- | --- | --- |
| MCU | STM32F103CBT6 + resonator 8 MHz | 55–90k |
| Gate driver | 2× EG2104S + bootstrap + mạch gate + Q8 AO3400 | 5–9k |
| Rail gate | MT3608 + cuộn + diode + tụ | 3–4k |
| Đo/cắt dòng | Shunt 2512 3 W + INA181A1 + comparator | 9–14k |
| MOSFET | 5× 60 V PDFN 5×6 ≤ 1,6 mΩ (nội địa; TPH1R306PL: ~100–150k) | 35–60k |
| Bảo vệ VM | SMCJ26A, R | 3–5k |
| Tụ VM | 3× 220 µF/50 V 105 °C + 4× 4,7 µF/50 V | 15–28k |
| Nguồn logic | MP2459 + cuộn + Schottky ×3 + LDO | 8–12k |
| USB | USB-C, SRV05-4, CH340C | 8–12k |
| ESD I/O | 2× SRV05-4 | 2–4k |
| Encoder | NTC, 74LVC2G14, ferrite | 2–4k |
| Thụ động, nút, LED | | 12–18k |
| PCB | 4 lớp 40 × 62 mm, đồng ngoài 2 oz | 30–60k |
| SMT | Setup + linh kiện Extended chia 10 board | 40–80k |
| **Tổng board** | | **~235–405k** |
| Ngoài board | Pad silicone tản nhiệt ~25 × 40 mm (+ tấm nhôm nếu vỏ nhựa) | 10–30k |

Bỏ CAN giảm ~5–10k; FET thấp R_DS hơn và tụ 105 °C cộng thêm ~13–18k. Muốn ≤ 320k: MOSFET nội địa đạt R_DS max, lô 20 board, đổi linh kiện Extended sang Basic. Khi chuyển sang GD32/APM32F103CBT6 tiết kiệm thêm 25–50k. Giá là ước tính LCSC/JLCPCB, cần báo giá thật khi có BOM.

## 12. Kiểm thử & tiêu chí nghiệm thu

Cấp nguồn qua nguồn có giới hạn dòng trước khi dùng pin, theo thứ tự:

| Bài thử | Cách làm | Tiêu chí đạt |
| --- | --- | --- |
| Kiểm thiết kế | ERC, DRC sau refill, `kicad_review.py` | 0 lỗi thật; Gerber mới hơn PCB |
| Chỉ USB | Cắm USB, chưa có VM | MCU chạy; nạp qua CH340 bằng lệnh `bootloader` từ GUI và bằng nút BOOT0 + RST; VM ≈ 0 V; 12VG = 12,6 V ±5 %; +5V ≈ 4,6 V |
| Nguồn | VM 9 V, 16,8 V, 25,2 V, giới hạn 100 mA | +5V ≈ 5,25 V (sau Schottky), 3,3 V ±3 %, 12VG = 12,6 V; gợn 3,3 V < 20 mVpp; V_GS Q5 ≈ 12,5 V |
| Ngược cực | Cấp ngược 25 V (giới hạn dòng) | Không hỏng, dòng ≈ 0 |
| Trạng thái an toàn | Reset MCU, rút ENA, tắt boost 12VG, xem gate | Cả 4 FET tắt trong mọi trường hợp; tắt 12VG thì Q5 tắt và cầu không chạy |
| Dead-time / PWM | 10/50/97 % trên tải trở | Không shoot-through; overshoot SH < 20 % VM; bootstrap không sụt ở 97 % |
| Encoder | Quay tay, rồi tốc độ max | Không mất xung, chiều đúng |
| I/O | Mỗi IO: vào 3,3 V và 5 V; ra push-pull và open-drain kéo lên 5 V | Đọc đúng mức; không có dòng ngược vào 3V3 khi tháo pull-up |
| Đo dòng | So đồng hồ 0–15 A | Sai số < 5 % toàn thang |
| Cắt dòng phần cứng | Kẹt rotor ở 25,2 V | Dòng chặt ~30 A mỗi chu kỳ; trễ BKIN < 3 µs; STALL ≤ 0,5 s; lặp 10 lần |
| ENA | Rút dây ENA khi đang chạy | SD = 0 bằng phần cứng, `ENA_OFF` |
| Hãm | Dừng/đảo chiều ở 97 % tốc độ, pin 6S đầy | VM < 27,5 V, không hỏng |
| Truyền thông | I2C từ MCU chủ (kể cả bus 5 V); UART 3,3 V và 5 V; Step/Dir 200 kHz | Không mất lệnh; I2C tự hồi phục |
| Nhiễu / USB | 15 A đảo chiều liên tục 30 phút, USB + encoder cùng nối | Không reset, không mất USB, không sai xung |
| **Nhiệt trong hộp kín** | Board lắp trong hộp thật, có pad tản nhiệt + vỏ/tấm nhôm, **môi trường 40 °C** (hoặc 25 °C + cộng bù 15 °C), 15 A liên tục **đến khi ổn định nhiệt (≥ 60 phút)**; đo NTC + cặp nhiệt trên FET, shunt, tụ bulk | FET (vỏ) < 90 °C, shunt < 100 °C, tụ bulk < 75 °C, EG2104 < 90 °C; firmware không giảm công suất trước 60 phút |
| Giảm công suất nhiệt | Chặn đường tản nhiệt (bỏ pad), chạy 15 A | Firmware giảm dòng từ 85 °C, ngắt ở 105 °C, không hỏng |
| Điều khiển | Bước tốc độ và vị trí | Không dao động; sai vị trí ≤ 1 xung |

## 13. Rủi ro

| # | Rủi ro | Biểu hiện | Cách phòng |
| --- | --- | --- | --- |
| R1 | **Nhiệt 15 A trong hộp kín** | FET/tụ quá nóng, tuổi thọ giảm, giảm công suất sớm | Đường tản nhiệt xuống vỏ nhôm qua pad cách điện, FET ≤ 1,6 mΩ max, 2 oz, tụ 105 °C, NTC giảm công suất; vỏ nhựa → hạ định mức 10–12 A |
| R2 | Pad tản nhiệt bị thủng / lệch → chạm đồng hở xuống vỏ | Ngắn mạch VM/M± xuống vỏ | Pad cách điện ≥ 1 kV phủ dư 2 mm mỗi phía; không via hở sắc cạnh; kiểm cách điện bằng đồng hồ sau lắp |
| R3 | Thông số EG2104 khác IR2104; **V_IH > 2,5 V** khi không còn 74HCT08 | Chuyển mạch chậm; driver không nhận mức 3,3 V, chập chờn theo nhiệt | Đọc datasheet EG2104 trước khi vẽ; footprint cùng chân FD2104; bài thử PWM ở 85 °C |
| R3b | Firmware cấu hình sai CCER (CH1/CH1N) | Động cơ chạy sai chiều / không chạy | Mọi tổ hợp đều không shoot-through (mục 4.3); test đơn vị cho hàm đổi chiều; ENA cứng vẫn ngắt được |
| R4 | Ngắn mạch M± xuống GND không bị phát hiện | Cháy FET high-side | Cầu chì 20 A trên dây pin; firmware phát hiện VM sụt |
| R5 | Nhiễu chuyển mạch kích comparator giả | PWM bị cắt sớm | RC 1 kΩ/100 pF trước comparator, BKIN chốt đến chu kỳ sau, INA181 sát shunt |
| R6 | USB không cách ly | Mất USB khi chạy tải | GND sao, USB ở vùng logic, ESD; tune tải nặng bằng laptop pin |
| R7 | Nhiễu reset MCU (chung GND) | Động cơ giật | 4 lớp + L2 GND liền, lọc π buck, VDDA ferrite, PVD, IWDG, SD mặc định tắt |
| R8 | Gai VM khi hãm | Hỏng FET/tụ | SMCJ26A, Q5 cho dòng hãm về pin, giới hạn dốc giảm tốc; không chạy bằng nguồn adapter không nhận dòng ngược |
| R9 | Tụ bulk quá tải dòng gợn + nóng | Tụ phồng | Tụ 105 °C polymer/low-ESR, đặt xa FET, kiểm ripple current |
| R10 | Thay chip GD32/APM32 khác hành vi | Firmware lỗi clock/ADC/flash | Tầng HAL riêng; chạy lại bài thử clock, ADC, PWM/BKIN, bootloader |
| R11 | CPU F103 không đủ cho vòng dòng 20 kHz + JSON | Jitter | Fixed-point; hạ vòng dòng xuống 10 kHz |
| R12 | I2C F103 treo | Mất lệnh | Timeout + reset ngoại vi |
| R13 | Lệnh nhảy ROM bootloader lỗi (firmware hỏng, chip tương đương khác địa chỉ) | Không nạp được qua GUI | Dự phòng nút BOOT0 + RST; kiểm địa chỉ system memory khi đổi chip |

## 14. Câu hỏi mở (chốt trước giai đoạn 1)

- [x] **Lỗ bắt ốc**: 4 lỗ M2 ở 4 góc, NPTH (tâm cách mép 3 mm, chỉnh nếu phải khớp khung).
- [x] **Gate driver**: 2× EG2104. Còn phải đọc datasheet EG2104 (dòng ra, dead-time, ngưỡng IN/SD, UVLO).
- [x] **15 A liên tục trong hộp kín** → thiết kế tản nhiệt xuống vỏ (mục 6.3).
- [x] **MCU**: STM32F103CBT6, giữ khả năng thay GD32/APM32F103CBT6 (mục 3.1).
- [x] **CAN**: bỏ.
- [x] **4 I/O**: mức logic 3,3–5 V, chân FT (PA15, PB3, PB4, PA11).
- [x] **Buck 5 V**: chốt MP2459 (MPS), nguồn thứ hai MSKSEMI MP2459GJ-Z-MS sau khi kiểm datasheet.
- [x] **74HCT08**: bỏ (mục 4.3). Còn phải xác nhận **V_IH IN/SD của EG2104 ≤ 2,5 V** trước khi vẽ; nếu không đạt → chọn driver cùng chân đạt, hoặc thêm bộ đệm HCT.
- [ ] **Vỏ hộp**: nhôm hay nhựa? Kích thước? Mặt dưới board có ép được xuống vỏ/tấm nhôm không? (quyết định 15 A có đạt thật hay phải hạ 10–12 A)
- [ ] **Nhiệt độ môi trường ngoài hộp** tối đa bao nhiêu? (tài liệu giả định 40 °C)
- [ ] 4 I/O dùng làm ngõ ra có cần kéo tải (LED, relay) không, hay chỉ mức logic? (nếu kéo tải > 8 mA → cần transistor đệm)

---

## Phụ lục A — BOM dự kiến (đã gom mã linh kiện)

Mục tiêu: **ít mã linh kiện nhất** để giảm phí linh kiện Extended ở JLCPCB (~3 USD mỗi mã Extended mỗi đơn) và đơn giản hóa mua hàng. Từ ~69 mã (bản trước) giảm còn **33 mã**, tổng ~170 linh kiện. Ký hiệu RefDes là đề xuất. Giá trị chia áp FB và dung lượng hiệu dụng của tụ ceramic dưới DC bias phải kiểm lại theo datasheet.

### A0. Bảng mã linh kiện hợp nhất (33 mã)

Mọi điện trở 0603 1 %; mọi tụ ceramic nhỏ 0603 50 V.

| Mã | Linh kiện | Gói | SL | Dùng ở |
| --- | --- | --- | --- | --- |
| M01 | Điện trở 10 Ω 1 % | 0603 | 7 | R gate ×4, nối tiếp diode bootstrap ×2, lọc 12VG ×1 |
| M02 | Điện trở 1 kΩ 1 % | 0603 | 26 | FB buck/boost, gate Q5, gate Q8, ENA, lọc SO, LED, nút BOOT0, encoder (nối tiếp + pull-up), STEP/DIR, IO, UART |
| M03 | Điện trở 10 kΩ 1 % | 0603 | 30 | G–S FET, kéo xuống IN, NTC, FB, Q8, ENA, nOC, VREF_OC RC, BOOT0/BOOT1, CC (2 × 10 kΩ song song), AIN, pull-up IO, pull-up I2C, chia áp VM |
| M04 | Điện trở 100 kΩ 1 % | 0603 | 4 | Chia áp VM, G–S Q5, kéo xuống SD, kéo xuống VREF_OC |
| M05 | Shunt 2 mΩ 1 % 3 W, TCR ≤ 100 ppm | 2512 | 1 | Đo dòng |
| M06 | Tụ 100 pF 50 V C0G | 0603 | 11 | Lọc SO ×2, ENA, encoder ×2, STEP/DIR ×2, IO ×4 |
| M07 | Tụ 100 nF 50 V X7R | 0603 | 17 | Tách nguồn mọi IC, VM nửa cầu, BST buck, NRST, AIN, đo VM |
| M08 | Tụ 1 µF 50 V X7R | 0603 | 11 | Bootstrap (2 tụ/nửa cầu), VCC EG2104, VREF_OC RC, LDO, VDDA |
| M09 | Tụ 10 µF 25 V X5R | 0805 | 8 | Vào/ra boost, lọc 12VG, ra buck ×2, bulk 5 V, bulk 3V3 |
| M10 | Tụ 4,7 µF 50 V X7R | 1210 | 6 | Commutation 2/nửa cầu ×4, lọc π vào buck ×2 |
| M11 | Tụ điện phân 220 µF 50 V 105 °C low-ESR (hoặc polymer) | SMD 10 × 10,5 | 3 | Bulk VM |
| M12 | N-MOSFET 60 V, R_DS max ≤ 1,6 mΩ @10 V, Q_gd ≤ 15 nC (tham chiếu TPH1R306PL) | PDFN 5×6 | 5 | Q1–Q4 cầu H, Q5 chống ngược cực |
| M13 | AO3400 (N-ch 30 V, V_GS(th) 0,65–1,45 V, V_GS max ±12 V) | SOT-23 | 1 | Q8 tắt SD; cùng chân G-S-D với 2N7002 |
| M14 | 1N4148W (100 V, t_rr 4 ns) | SOD-123 | 6 | Bootstrap ×2, tắt nhanh gate ×4 |
| M15 | Schottky SS36 (60 V, 3 A) | SMA | 4 | Boost, buck, OR nguồn 5 V ×2 |
| M16 | TVS SMCJ26A | SMC | 1 | VM |
| M17 | LED xanh lá | 0603 | 2 | PWR, STATUS |
| M18 | **EG2104S** (V_IH ≤ 2,5 V — kiểm datasheet) | SOP-8 | 2 | Gate driver |
| M19 | **MT3608** | SOT-23-6 | 1 | Boost 12VG |
| M20 | **INA181A1** | SOT-23-6 | 1 | Khuếch đại dòng ×20 |
| M21 | **TLV7011** (hoặc LMV331), open-drain | SOT-23-5 | 1 | Comparator quá dòng |
| M22 | **MP2459GJ** (MPS; nguồn thứ hai: MSKSEMI MP2459GJ-Z-MS, LCSC C45367195) | TSOT23-6 | 1 | Buck 5 V, 4,5–55 V, 480 kHz, V_FB 0,81 V |
| M23 | **AP2112K-3.3** | SOT-23-5 | 1 | LDO 3,3 V |
| M24 | **STM32F103CBT6** | LQFP-48 | 1 | MCU |
| M25 | **CH340C** | SOP-16 | 1 | USB–UART |
| M26 | **74LVC2G14** | SOT-23-6 | 1 | Schmitt encoder |
| M27 | **SRV05-4** | SOT-23-6 | 5 | ESD: USB, encoder + STEP/DIR, IO, UART + I2C, ENA + AIN |
| M28 | Cuộn 22 µH, I_sat ≥ 1 A | 4 × 4 mm | 2 | Boost, buck (kiểm datasheet MP2459) |
| M29 | Ferrite 600 Ω @100 MHz, ≥ 1 A | 0805 | 4 | Lọc π buck, VDDA, VBUS, nguồn 5 V encoder |
| M30 | Resonator gốm 8 MHz có tụ tích hợp (3 chân, vd Murata CSTNE8M00G55) | SMD 3,2 × 1,3 | 1 | HSE (thay thạch anh + 2 tụ tải) |
| M31 | NTC 10 kΩ B3950 | 0603 | 1 | Nhiệt FET |
| M32 | Nút nhấn SMD 3 × 4 mm | SMD | 3 | RST, BOOT0, USER |
| M33 | USB-C 16P (USB 2.0) | SMD | 1 | Cổng USB |

Không phải linh kiện SMT: pad VM±/M± và pad tín hiệu (đồng trên PCB), 4 lỗ M2, jumper hàn JP1/JP3/JP4 (đồng trên PCB), cầu chì 20 A trên dây pin, pad silicone tản nhiệt, ốc M2.

### A0.1 Các thay đổi để gom mã (so với bản trước)

| Trước | Sau | Kiểm tra |
| --- | --- | --- |
| R gate 2,2 Ω, lọc 10 Ω | **10 Ω** cho cả hai | Họ IR2104 có điện trở nội ~45–90 Ω → thêm 10 Ω chỉ chậm thêm ~10–15 %; tắt nhanh qua 1N4148W. Nếu FET nóng vì chuyển mạch: nối tắt R gate (thêm mã 0 Ω) |
| G–S 47 kΩ, kéo xuống IN 47 kΩ | **10 kΩ** | Tải driver 1,2 mA/FET ở 12 V, MCU 0,33 mA — không đáng kể |
| ES1D bootstrap | **1N4148W** + **10 Ω nối tiếp** | V_R 100 V ≥ VM + 12VG + gai; 10 Ω giới hạn dòng nạp lần đầu ~1,2 A |
| Tụ bootstrap 1 µF/25 V 0805 | **2 × 1 µF/50 V 0603** song song | ~0,6–0,7 µF hiệu dụng mỗi tụ ở 12 V → ~1,3 µF ≥ 0,7 µF yêu cầu |
| SS14, SS24, B5819W | **SS36** cho cả 4 vị trí | 60 V cho diode buck (VM tới 42 V khi TVS kẹp); V_F ~0,4 V ở 0,15 A |
| Zener 12 V cho gate Q5 | **Gate Q5 lấy từ +12VG** qua 1 kΩ, 100 kΩ G–S | Q5 là FET low-side nên +12VG (so với GND board) cho V_GS ≈ 12,5 V ở mọi mức pin. Ngược cực: board không có nguồn → Q5 tắt. Mất 12VG: EG2104 UVLO cũng tắt cầu → không có dòng qua thân diode Q5 |
| Thạch anh + 2 × 18 pF | **Resonator có tụ tích hợp** | ±0,5 % đủ cho UART (không còn CAN) |
| USBLC6-2SC6 | **SRV05-4** (VDD nối VBUS) | SRV05-4 là mảng TVS điện dung thấp dùng được cho USB 2.0 |
| 2 × MMBT3904 + R + jumper (auto-boot) | **Bỏ** | Firmware nhận lệnh `bootloader` từ GUI rồi nhảy vào ROM bootloader (F103: 0x1FFF F000 — kiểm cho GD32/APM32); dự phòng nút BOOT0 + RST. Bỏ luôn rủi ro reset khi mở cổng COM |
| CC 5,1 kΩ | **2 × 10 kΩ song song** = 5 kΩ | Nằm trong dung sai R_d của USB-C (5,1 kΩ ±20 %) |
| BAT54S + 1 kΩ cho AIN | **10 kΩ + 100 nF**, ESD qua SRV05-4 | Dải 0–3,3 V; quá áp 5 V chỉ bơm ~0,14 mA vào diode kẹp nội — trong giới hạn, nhưng có thể làm lệch nhẹ ADC kênh khác khi đang quá áp |
| PESD5V0S1BL cho ENA | Kênh của **SRV05-4** thứ 5 (ENA + AIN) | |
| 2,2 kΩ pull-up encoder | **1 kΩ** | Encoder open-collector sink 5 mA ở 5 V (Hall thường ≥ 10 mA) |
| 2,2 kΩ pull-up I2C + 33 Ω nối tiếp | **10 kΩ**, bỏ R nối tiếp | Đủ cho 100 kHz dây ngắn; 400 kHz → master thêm pull-up mạnh hơn. Không đặt R nối tiếp lớn (tạo phân áp mức thấp) |
| 100 Ω nối tiếp UART/encoder/gate Q8 | **1 kΩ** | UART 1 Mbaud với ~30 pF: τ 30 ns ✔; encoder 1 kΩ × 100 pF = 100 ns ✔ |
| Lọc SO 100 Ω + 1 nF | **1 kΩ + 100 pF** | Cùng τ = 100 ns |
| VREF_OC RC 4,7 kΩ / 1 µF | **10 kΩ / 1 µF** ×2 | fc ≈ 16 Hz; hệ số DC 100k / 120k = 0,83 → firmware bù (VREF_OC 1,2 V ↔ duty 43,6 %) |
| R trễ comparator 1 MΩ | **Bỏ** | BKIN chốt PWM tắt đến chu kỳ sau (AOE), comparator dao động ở ngưỡng không ảnh hưởng |
| FB boost 191 k / 10 k | **(10 k + 10 k) / 1 k** | 0,6 × 21 = **12,6 V** |
| FB buck 56 k / 10 k | **(10 k + 1 k + 1 k) / (1 k + 1 k)** | 0,81 × 7 = **5,67 V** trước Schottky → ~5,25 V sau (kiểm V_FB MP2459) |
| Chia áp VM 100 k / 12 k | **100 k / 10 k** | Thang 36,3 V |
| Tụ 2,2 µF/50 V 1206, 22 µF, 4,7 µF/10 V, 10 nF, 1 nF | Gom về **4,7 µF/50 V 1210, 2 × 10 µF, 10 µF, 100 nF, 100 pF** | BST buck 100 nF (kiểm datasheet MP2459) |
| Polyfuse encoder | **Ferrite** (cùng mã M29) | Không giới hạn dòng: chập nguồn encoder → buck tự giới hạn dòng → MCU brown-out → SD tắt (an toàn, phải cấp nguồn lại) |
| Ferrite 0603 + 0805 | **0805 ≥ 1 A** cho cả 4 vị trí | |

### A1. Nguồn vào & bảo vệ VM

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| Q5 | M12 | MOSFET 60 V | 1 | Chống ngược cực low-side: drain → VM− (pin), source → GND board |
| R1 | M02 | 1 kΩ | 1 | +12VG → gate Q5 |
| R2 | M04 | 100 kΩ | 1 | G–S Q5 |
| D2 | M16 | SMCJ26A | 1 | |
| C1–C3 | M11 | 220 µF / 50 V | 3 | Kiểm ripple current |
| C4–C7 | M10 | 4,7 µF / 50 V | 4 | 2 tụ mỗi nửa cầu, sát FET |
| C8, C9 | M07 | 100 nF | 2 | 1 tụ mỗi nửa cầu |
| R3, R4 | M04, M03 | 100 kΩ / 10 kΩ | 2 | Chia áp VM → PA5 |
| C10 | M07 | 100 nF | 1 | Tại PA5 |

### A2. Cầu H & gate driver

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| Q1–Q4 | M12 | MOSFET 60 V | 4 | |
| U1, U2 | M18 | EG2104S | 2 | 1 chip mỗi nửa cầu |
| D3, D4 | M14 | 1N4148W | 2 | Bootstrap, anode → +12VG |
| R5, R6 | M01 | 10 Ω | 2 | Nối tiếp diode bootstrap |
| C11–C14 | M08 | 1 µF / 50 V | 4 | 2 tụ song song VB–VS mỗi nửa cầu |
| C15, C16 | M08 | 1 µF / 50 V | 2 | VCC mỗi EG2104 |
| C17, C18 | M07 | 100 nF | 2 | VCC mỗi EG2104 |
| R7–R10 | M01 | 10 Ω | 4 | R gate |
| D5–D8 | M14 | 1N4148W | 4 | Song song R gate (cathode về phía driver), tắt nhanh |
| R11–R14 | M03 | 10 kΩ | 4 | G–S mỗi FET |
| R15, R16 | M03 | 10 kΩ | 2 | Kéo xuống IN_A (PA8), IN_B (PB13) |
| TH1 | M31 | NTC 10 kΩ | 1 | Giữa 4 FET → PA3 |
| R17 | M03 | 10 kΩ | 1 | Pull-up NTC lên 3V3 |

### A3. Rail gate +12VG

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| U3 | M19 | MT3608 | 1 | EN nối VIN |
| L1 | M28 | 22 µH | 1 | |
| D9 | M15 | SS36 | 1 | |
| C19 | M09 | 10 µF | 1 | Vào |
| C20, C21 | M09 | 10 µF / 25 V | 2 | Ra (~3 µF hiệu dụng mỗi tụ ở 12,6 V; kiểm gợn) |
| R18, R19 | M03 | 10 kΩ + 10 kΩ nối tiếp | 2 | FB phía trên |
| R20 | M02 | 1 kΩ | 1 | FB phía dưới → **12,6 V** |
| R21 | M01 | 10 Ω | 1 | Lọc RC trước VCC driver |
| C22 | M09 | 10 µF | 1 | Lọc RC |

### A4. Mạch SD / ENA

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| Q8 | M13 | AO3400 | 1 | Drain → SD, source → GND |
| R22 | M03 | 10 kΩ | 1 | Kéo **lên** 3V3 tại gate Q8 |
| R23 | M02 | 1 kΩ | 1 | PB14 (nDIS) → gate Q8 |
| R24 | M04 | 100 kΩ | 1 | Kéo SD xuống GND |
| R25 | M03 | 10 kΩ | 1 | Nút ENA → SD |
| R26 | M03 | 10 kΩ | 1 | Kéo nút ENA xuống GND |
| R27 | M02 | 1 kΩ | 1 | Pad ENA → nút ENA (nút ENA nối PB15) |
| C23 | M06 | 100 pF | 1 | Lọc ENA |
| JP1 + R28 | — + M02 | Jumper hàn + 1 kΩ lên +5V | 1 | Bỏ qua ENA khi không dùng |

### A5. Đo & cắt dòng

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| R29 | M05 | Shunt 2 mΩ | 1 | Đo Kelvin |
| U5 | M20 | INA181A1 | 1 | REF → GND |
| C24 | M07 | 100 nF | 1 | |
| R30, C25 | M02 + M06 | 1 kΩ + 100 pF | 2 | Lọc SO → comparator |
| R31, C26 | M02 + M06 | 1 kΩ + 100 pF | 2 | Lọc SO → PA4 |
| U6 | M21 | TLV7011 | 1 | Nuôi 3V3; không R trễ |
| C27 | M07 | 100 nF | 1 | |
| R32 | M03 | 10 kΩ | 1 | Pull-up nOC → PB12 |
| R33, R34 | M03 | 10 kΩ | 2 | RC 2 bậc VREF_OC từ PB1 |
| C28, C29 | M08 | 1 µF | 2 | RC 2 bậc VREF_OC |
| R35 | M04 | 100 kΩ | 1 | Kéo VREF_OC xuống khi MCU reset |

### A6. Nguồn logic 5 V / 3,3 V

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| FB1 | M29 | Ferrite | 1 | Lọc π vào buck |
| C30, C31 | M10 | 4,7 µF / 50 V | 2 | Hai phía FB1 |
| U7 | M22 | MP2459 | 1 | |
| C32 | M07 | 100 nF | 1 | Tụ BST |
| D10 | M15 | SS36 | 1 | Diode bắt |
| L2 | M28 | 22 µH | 1 | |
| R36–R38 | M03, M02, M02 | 10 kΩ + 1 kΩ + 1 kΩ nối tiếp | 3 | FB phía trên (12 kΩ) |
| R39, R40 | M02 | 1 kΩ + 1 kΩ nối tiếp | 2 | FB phía dưới (2 kΩ) → **5,67 V** |
| C33, C34 | M09 | 10 µF | 2 | Ra buck |
| D11, D12 | M15 | SS36 | 2 | OR: buck → +5V, VBUS → +5V |
| C35 | M09 | 10 µF | 1 | Bulk +5V |
| U8 | M23 | AP2112K-3.3 | 1 | |
| C36, C37 | M08 | 1 µF | 2 | Vào/ra LDO |
| C38 | M09 | 10 µF | 1 | Bulk 3V3 (gần MCU) |
| LED1, R41 | M17 + M02 | LED + 1 kΩ | 2 | PWR, nối 3V3 |
| FB4 | M29 | Ferrite | 1 | +5V → nguồn encoder |

### A7. MCU

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| U9 | M24 | STM32F103CBT6 | 1 | |
| Y1 | M30 | Resonator 8 MHz | 1 | Chân giữa → GND |
| C39–C42 | M07 | 100 nF | 4 | VDD1/2/3 + VBAT |
| FB2 | M29 | Ferrite | 1 | VDDA |
| C43, C44 | M08 + M07 | 1 µF + 100 nF | 2 | VDDA |
| R42, R43 | M03 | 10 kΩ | 2 | BOOT0, PB2/BOOT1 kéo xuống |
| C45 | M07 | 100 nF | 1 | NRST |
| SW1–SW3 | M32 | Nút | 3 | RST, BOOT0 (lên 3V3 qua R44), USER (PC14) |
| R44 | M02 | 1 kΩ | 1 | Nối tiếp nút BOOT0 |
| LED2, R45 | M17 + M02 | LED + 1 kΩ | 2 | STATUS (PC13) |

### A8. USB (CH340C, không cách ly)

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| J1 | M33 | USB-C 16P | 1 | |
| R46–R49 | M03 | 10 kΩ | 4 | 2 × 10 kΩ song song cho mỗi CC (5 kΩ) |
| FB3 | M29 | Ferrite | 1 | VBUS |
| U10 | M27 | SRV05-4 | 1 | ESD D+/D−, VDD → VBUS |
| U11 | M25 | CH340C | 1 | VCC = V3 = 3V3 |
| C46 | M07 | 100 nF | 1 | |

### A9. Encoder

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| U12 | M26 | 74LVC2G14 | 1 | Nuôi 3V3 |
| C47 | M07 | 100 nF | 1 | |
| R50, R51 | M02 | 1 kΩ | 2 | Pull-up A, B (phía connector), JP3 chọn 3V3/5V |
| R52, R53 | M02 | 1 kΩ | 2 | Nối tiếp A, B |
| C48, C49 | M06 | 100 pF | 2 | |
| U13 | M27 | SRV05-4 | 1 | ESD: ENC A, ENC B, STEP, DIR (VDD → +5V) |

### A10. Lệnh, I/O, UART, I2C

| Ref | Mã | Giá trị | SL | Ghi chú |
| --- | --- | --- | --- | --- |
| R54, R55 | M02 | 1 kΩ | 2 | Nối tiếp STEP, DIR |
| C50, C51 | M06 | 100 pF | 2 | |
| R56, C52 | M03 + M07 | 10 kΩ + 100 nF | 2 | AIN (PA2), dải 0–3,3 V |
| R57–R60 | M02 | 1 kΩ | 4 | Nối tiếp IO1–IO4 |
| C53–C56 | M06 | 100 pF | 4 | |
| R61–R64 | M03 | 10 kΩ (DNP được) | 4 | Pull-up IO lên 3V3 |
| U14 | M27 | SRV05-4 | 1 | ESD IO1–IO4 |
| R65, R66 | M02 | 1 kΩ | 2 | Nối tiếp UART TX/RX |
| R67, R68 | M03 | 10 kΩ | 2 | Pull-up I2C lên 3V3, JP4 ngắt |
| U15 | M27 | SRV05-4 | 1 | ESD UART TX/RX, SCL, SDA |
| U16 | M27 | SRV05-4 | 1 | ESD ENA, AIN (2 kênh còn lại để trống) |

### A11. Tổng hợp

| Nhóm | Số mã | SL |
| --- | --- | --- |
| Điện trở + shunt | 5 | 68 |
| Tụ | 6 | 56 |
| Bán dẫn rời (MOSFET, AO3400, diode, TVS, LED) | 6 | 19 |
| IC | 10 | 15 |
| Cuộn, ferrite, resonator, NTC, nút, USB-C | 6 | 12 |
| **Tổng** | **33** | **~170** |

Gợi ý khi đặt JLCPCB: các mã M01–M04, M06–M09, M14, M17 thường có bản **Basic** (không mất phí Extended); kiểm lại từng mã khi lên BOM thật. Các mã chắc chắn Extended: MOSFET, EG2104, MT3608, INA181, TLV7011, MP2459, CH340C, 74LVC2G14, SRV05-4, resonator, tụ 220 µF, shunt 3 W.
