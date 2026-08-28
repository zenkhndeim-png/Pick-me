# Pick-me — XAUUSD SMC Pro

Pine Script indicator สำหรับเทรด XAUUSD ตามแนวคิด Smart Money Concepts

| ไฟล์ | คำอธิบาย |
|---|---|
| `src/XAUUSD_SMC_Pro_v9.pine` | เวอร์ชันต้นฉบับ (เก็บไว้เทียบ) |
| `src/XAUUSD_SMC_Pro_v9_FIXED.pine` | **v9 ต้นฉบับ แก้เฉพาะบั๊ก 11 จุด ไม่เพิ่มอะไรเลย** |
| `docs/V9_FIXED.md` | แก้อะไรบ้าง + อะไรที่ตั้งใจไม่แก้ และเพราะอะไร |
| `docs/CODE_v9_FIXED.md` | โค้ด v9 (fixed) พร้อมปุ่มคัดลอก |
| `src/XAUUSD_SMC_Pro_v9_1_H1.pine` | เวอร์ชันแก้บั๊ก + ปรับค่าสำหรับ H1 |
| `docs/REVIEW_v9.md` | รายละเอียดบั๊กที่พบและวิธีแก้ |
| `docs/CODE_v9_1_H1.md` | โค้ด v9.1 ในรูปแบบ markdown — มีปุ่มคัดลอกในตัว |
| `docs/CODE_v9_original.md` | โค้ดต้นฉบับในรูปแบบ markdown |
| `src/XAUUSD_SMC_Pro_v9_2_STRATEGY.pine` | strategy v9.2 — แก้ 3 บั๊ก + break-even |
| `docs/CODE_v9_2_STRATEGY.md` | โค้ด v9.2 พร้อมปุ่มคัดลอก |
| `docs/TEST_PLAN_v9_2.md` | แผนทดสอบทีละข้อ + ผลที่วัดจริงแล้ว |
| `src/XAUUSD_SMC_Pro_v9_3_MICRO.pine` | v9.3 สำหรับทุนน้อย — risk gate + เบรกฉุกเฉิน |
| `docs/CODE_v9_3_MICRO.md` | โค้ด v9.3 พร้อมปุ่มคัดลอก |
| `docs/MICRO_30USD.md` | คู่มือเทรดด้วยทุน $30 + ผล M5 |
| `src/XAUUSD_SMC_Pro_v9_4_LIVE.pine` | indicator สำหรับเทรดมือจริง |
| `docs/CODE_v9_4_LIVE.md` | โค้ด v9.4 พร้อมปุ่มคัดลอก |
| `docs/LIVE_GUIDE.md` | คู่มือเทรดมือ |
| `src/XAUUSD_SMC_Pro_v9_5_ICT.pine` | **v9.4 + ระบบเข้าไม้แบบ ICT** (sweep → MSS → FVG/OB → P/D → killzone) |
| `docs/CODE_v9_5_ICT.md` | โค้ด v9.5 พร้อมปุ่มคัดลอก |
| `docs/ICT_GUIDE.md` | คู่มือส่วน ICT + ค่าที่แนะนำ |
| `src/XAUUSD_SMC_Pro_v10_ICT.pine` | **สายตรงจาก v9 — แก้บั๊ก 12 ข้อ + ICT** (แนะนำถ้ายึด v9 เป็นหลัก) |
| `docs/CODE_v10_ICT.md` | โค้ด v10 พร้อมปุ่มคัดลอก |
| `docs/V10_GUIDE.md` | คู่มือ v10 + ตารางบั๊กที่แก้ทั้งหมด |

## เริ่มตรงไหนดี

อยากได้ v9 เดิมที่ไม่มีอะไรงอกออกมา แค่บั๊กหาย → [`docs/V9_FIXED.md`](docs/V9_FIXED.md)

## สายอื่นที่แยกออกไป

- **สาย v9 → v10** — 0.01 lot คงที่ TP1/TP2 realtime ตาราง MTF เหมือนเดิม
  เริ่มที่ [`docs/V10_GUIDE.md`](docs/V10_GUIDE.md)
- **สาย v9.2 → v9.5** — คิด lot จากขนาดทุน มีเบรกกันทุนไม่พอ ถือทีละไม้
  เริ่มที่ [`docs/LIVE_GUIDE.md`](docs/LIVE_GUIDE.md)

รายละเอียดบั๊กของต้นฉบับอยู่ที่ [`docs/REVIEW_v9.md`](docs/REVIEW_v9.md)
