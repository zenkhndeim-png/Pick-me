# XAUUSD SMC Pro v9.5 ICT — โค้ดเต็มสำหรับคัดลอก

v9.4 LIVE ทุกบรรทัด บวกระบบสัญญาณแบบ ICT (liquidity sweep → MSS → FVG/OB →
premium/discount → killzone) อ่านวิธีใช้ที่ [ICT_GUIDE.md](ICT_GUIDE.md)

⚠️ ฝั่ง SMC ผ่าน backtest แล้ว ฝั่ง ICT **ยังไม่ผ่าน** — เลือกโหมด
"SMC เท่านั้น" ถ้าต้องการผลเท่ากับ v9.4 เป๊ะ ๆ

กดปุ่มคัดลอก (ไอคอนมุมขวาบนของกล่องโค้ด) แล้ววางใน Pine Editor
บรรทัดแรกต้องเป็น `//@version=5`

```pine
//@version=5
// ══════════════════════════════════════════════════════════════════════════
// XAUUSD SMC Pro v9.5 ICT — v9.4 LIVE + โมเดล ICT เต็มรูปแบบ
//
// ทุกอย่างของ v9.4 ยังอยู่ครบไม่แตะต้อง (คิด lot, เตือนทุนไม่พอ, ติดตาม
// สถานะ limit → ติด → ปิด, alert ครบทุกจังหวะ)
//
// ที่เพิ่มเข้ามาคือ "ระบบสัญญาณที่สอง" ที่เข้าไม้ตามลำดับของ ICT จริง ๆ:
//
//   1) Liquidity sweep  — ราคาแทงทะลุ swing เดิมไปเก็บ stop แล้วปิดกลับ
//   2) MSS              — พลิกโครงสร้างสวนทางที่กวาดไป (พร้อม displacement)
//   3) โซนเข้า          — FVG ของขาที่พลิก ถ้าไม่มีใช้ Order Block
//   4) Premium/Discount — Long ต้องอยู่ครึ่งล่างของกรอบ / Short ครึ่งบน
//   5) Killzone         — เข้าเฉพาะช่วงเวลาที่เงินใหญ่ทำงาน
//   6) SL หลังจุดกวาด   — ไม่ใช่ ATR ลอย ๆ / TP เลือกได้ R คงที่ หรือไปหา
//                          liquidity ฝั่งตรงข้าม
//
// ⚠️ อ่านก่อนใช้เงินจริง
//   ส่วน SMC (v9.4) ผ่าน backtest H1 20 เดือน 568 records PF 1.241 มาแล้ว
//   ส่วน ICT ในไฟล์นี้ "ยังไม่ผ่าน backtest" — เป็นตรรกะที่เขียนตามตำรา ICT
//   ตรง ๆ แต่ยังไม่มีตัวเลขยืนยันว่าทำเงินได้บนทอง
//   → เอาไปรันเดโมเก็บสถิติก่อน ถ้าจะกลับไปใช้ชุดที่พิสูจน์แล้วให้เลือก
//     โหมด "SMC เท่านั้น (= v9.4 เดิม)"
// ══════════════════════════════════════════════════════════════════════════
indicator("XAUUSD SMC Pro v9.5 ICT", overlay=true, max_labels_count=500, max_lines_count=500, max_boxes_count=500, max_bars_back=500)

// ══════════════════════════════════════════
// 1. บัญชีและความเสี่ยง
// ══════════════════════════════════════════
grp_acc    = "1. บัญชีและความเสี่ยง"
acctSize   = input.float(30.0, "ทุนในบัญชี ($)", minval=1, step=1, group=grp_acc)
riskPct    = input.float(5.0, "เสี่ยงต่อไม้ (% ของทุน)", minval=0.1, maxval=50, step=0.1, group=grp_acc, tooltip="มาตรฐานคือ 1-2% / ทุนน้อยมากอาจต้องยอม 5% แต่รู้ไว้ว่าแพ้ 5 ไม้ติด = ทุนหาย 25%")
ozPerLot   = input.float(100, "1 lot = กี่ oz", options=[100, 10, 1], group=grp_acc, tooltip="บัญชีมาตรฐาน = 100 / micro = 10 / cent มักเป็น 1 — เช็คกับโบรก")
minLot     = input.float(0.01, "lot เล็กสุดที่โบรกรับ", minval=0.001, step=0.001, group=grp_acc)
lotStep    = input.float(0.01, "ขั้นการเพิ่ม lot", minval=0.001, step=0.001, group=grp_acc)
blockOver  = input.bool(true, "ไม่ยิงสัญญาณถ้าทุนไม่พอ", group=grp_acc, tooltip="เปิดไว้ = ปลอดภัย ถ้าไม่มีสัญญาณเลยแปลว่า TF นี้ SL ใหญ่เกินทุน")

// ══════════════════════════════════════════
// 2. เลือกระบบสัญญาณ
// ══════════════════════════════════════════
grp_mode   = "2. เลือกระบบสัญญาณ"
MODE_SMC   = "SMC เท่านั้น (= v9.4 ที่ผ่าน backtest)"
MODE_ICT   = "ICT เท่านั้น"
MODE_ANY   = "ทั้งคู่ — อันไหนมาก่อนก็เข้า"
MODE_BOTH  = "ทั้งคู่ — ICT ต้องได้คะแนน SMC ด้วย"
sigMode    = input.string(MODE_ANY, "ระบบที่ใช้ยิงสัญญาณ", options=[MODE_SMC, MODE_ICT, MODE_ANY, MODE_BOTH], group=grp_mode, tooltip="SMC = ชุดที่ผ่าน backtest มาแล้ว / ICT = ตรรกะใหม่ ยังไม่ผ่าน backtest — เริ่มที่เดโมก่อน")

modeSMConly = sigMode == MODE_SMC
modeICTonly = sigMode == MODE_ICT
modeBothAny = sigMode == MODE_ANY
modeBothCfm = sigMode == MODE_BOTH
useSMCsig   = modeSMConly or modeBothAny
useICTsig   = modeICTonly or modeBothAny or modeBothCfm

// ══════════════════════════════════════════
// 3. โครงสร้าง / MTF
// ══════════════════════════════════════════
grp_struct = "3. Structure / MTF"
swingLen   = input.int(5, "Swing Length", minval=2, maxval=20, group=grp_struct)
useMTF     = input.bool(true, "เปิด MTF Filter", group=grp_struct)
mtfConfirm = input.string("H1", "ยืนยันด้วย TF", options=["M15","M30","H1","H4"], group=grp_struct, tooltip="ต้องสูงกว่า TF ของกราฟ — เล่น M5 ใช้ H1 / เล่น H1 ใช้ H4")
strictMTF  = input.bool(true, "บังคับเข้าตามเทรนด์ MTF", group=grp_struct)
noRepaint  = input.bool(true, "MTF ไม่ repaint (ใช้ค่าแท่งที่ปิดแล้ว)", group=grp_struct, tooltip="เปิดไว้เสมอตอนเทรดจริง ปิดแล้วสัญญาณจะโผล่แล้วหายระหว่างแท่ง")

// ══════════════════════════════════════════
// 4. สัญญาณ SMC (ชุดเดิม v9.4)
// ══════════════════════════════════════════
grp_sig    = "4. สัญญาณ SMC"
emaFast    = input.int(21, "EMA Fast", minval=5, group=grp_sig)
emaSlow    = input.int(50, "EMA Slow", minval=10, group=grp_sig)
rsiLen     = input.int(14, "RSI Length", minval=2, group=grp_sig)
rsiOB      = input.int(70, "RSI OB", minval=60, group=grp_sig)
rsiOS      = input.int(30, "RSI OS", maxval=40, group=grp_sig)
rsiUseInp  = input.bool(false, "ให้ RSI OB/OS มีผลจริง (แก้บั๊ก B2)", group=grp_sig, tooltip="ปิดไว้ = ใช้ตรรกะเดิมที่ผ่าน backtest (เลข 35/65 ตายตัว ช่อง RSI OB/OS แทบไม่มีผล)\nเปิด = ช่อง RSI OB/OS ใช้งานได้จริง แต่จำนวนไม้จะเปลี่ยนจากที่เคยทดสอบ")
volFilter  = input.bool(true, "Volume Filter", group=grp_sig)
minScore   = input.int(4, "Min Score", minval=2, maxval=7, group=grp_sig)
useMomentum= input.bool(true, "ยอมรับ Momentum candle", group=grp_sig)
signalGap  = input.int(15, "ระยะห่างสัญญาณขั้นต่ำ (แท่ง)", minval=1, maxval=50, group=grp_sig)
noChaseHi  = input.int(62, "ห้าม Long ถ้า RSI เกิน", minval=50, maxval=75, group=grp_sig)
noChaseLo  = input.int(38, "ห้าม Short ถ้า RSI ต่ำกว่า", minval=25, maxval=50, group=grp_sig)

// ══════════════════════════════════════════
// 5. สัญญาณ ICT  ← ของใหม่
// ══════════════════════════════════════════
grp_ict    = "5. สัญญาณ ICT"
ictPivLen  = input.int(3, "ความยาว swing สำหรับหา liquidity", minval=2, maxval=10, group=grp_ict, tooltip="ICT ใช้ short-term high/low — 2-3 พอ ยิ่งมากยิ่งได้จุดใหญ่แต่สัญญาณน้อย")
ictLiqAge  = input.int(120, "liquidity เก่าได้ไม่เกิน (แท่ง)", minval=10, maxval=500, group=grp_ict, tooltip="กันไม่ให้ไปกวาด swing ที่เกิดเมื่อหลายวันก่อนแล้วนับเป็น setup")
ictDrLook  = input.int(40, "กรอบวัด Premium/Discount ย้อนหลัง (แท่ง)", minval=10, maxval=200, group=grp_ict, tooltip="ต้องครอบขาที่ลงมากวาด ไม่ใช่แค่ขาที่เพิ่งพลิก — ถ้าตั้งสั้นเกิน FVG จะตกฝั่ง premium ตลอดแล้วไม่มีสัญญาณเลย")
ictMssLook = input.int(10, "หา high/low ที่ต้องเบรก (MSS) ย้อนหลัง", minval=3, maxval=50, group=grp_ict)
ictArmAge  = input.int(12, "หลังกวาดแล้วต้อง MSS ภายใน (แท่ง)", minval=2, maxval=50, group=grp_ict, tooltip="กวาดแล้วไม่พลิกโครงสร้างในเวลานี้ = ทิ้ง setup")
ictNeedDsp = input.bool(true, "แท่ง MSS ต้องเป็น displacement", group=grp_ict, tooltip="แท่งที่เบรกต้องตัวใหญ่จริง ไม่ใช่คลานข้าม")
ictDspAtr  = input.float(0.5, "displacement = ตัวเทียน > ATR ×", minval=0.1, step=0.1, group=grp_ict)
ictZoneScan= input.int(12, "หาโซนเข้า (FVG/OB) ย้อนหลัง (แท่ง)", minval=3, maxval=30, group=grp_ict)
ictUseOB   = input.bool(true, "ไม่มี FVG ให้ใช้ Order Block แทน", group=grp_ict, tooltip="OB = แท่งตรงข้ามแท่งสุดท้ายก่อนขาที่พลิก")
ENT_CE     = "กลางโซน (CE) — สมดุลสุด"
ENT_FAR    = "ขอบไกล — รอย่อลึก ติดยากแต่ SL แคบ"
ENT_NEAR   = "ขอบใกล้ — เข้าไว ติดง่ายแต่ SL กว้าง"
ictEntMode = input.string(ENT_CE, "วาง limit ตรงไหนของโซน", options=[ENT_CE, ENT_FAR, ENT_NEAR], group=grp_ict)
ictSlBuf   = input.float(0.3, "SL เลยจุดกวาดไปอีก (× ATR)", minval=0.0, step=0.1, group=grp_ict, tooltip="SL วางหลัง low/high ที่ไปกวาดมา ไม่ใช่ ATR ลอย ๆ")
ictWideSkip= input.bool(true, "SL กว้างเกินเพดาน → ข้ามสัญญาณ", group=grp_ict, tooltip="เปิด = ไม่บีบ SL ให้แคบกว่าที่โครงสร้างต้องการ (บทเรียนจากบั๊ก A3)")
TP_R       = "R คงที่ (ตามช่อง TP = SL ×)"
TP_LIQ     = "ไปหา liquidity ฝั่งตรงข้าม"
ictTpMode  = input.string(TP_R, "TP ของไม้ ICT", options=[TP_R, TP_LIQ], group=grp_ict, tooltip="R คงที่ = ชุดที่มีข้อมูลรองรับจาก 568 ไม้ / liquidity = ตำรา ICT แต่ระยะไม่แน่นอน")
ictMinRR   = input.float(1.0, "โหมด liquidity: R ขั้นต่ำ ไม่ถึงข้าม", minval=0.3, step=0.1, group=grp_ict)
ictNeedDsc = input.bool(true, "Long ต้องอยู่ Discount / Short ต้อง Premium", group=grp_ict, tooltip="วัดจากกรอบ [จุดที่กวาด ↔ ปลายขาที่พลิก] — ห้ามซื้อของแพง")
ictNeedOTE = input.bool(false, "บังคับเข้าในโซน OTE 62-79% เท่านั้น", group=grp_ict, tooltip="เข้มขึ้นอีกชั้น สัญญาณจะน้อยลงมาก")
ictUseRng  = input.bool(true, "ใช้ฟิลเตอร์ Sideway กับ ICT ด้วย", group=grp_ict)
ictGap     = input.int(5, "ระยะห่างสัญญาณ ICT ขั้นต่ำ (แท่ง)", minval=1, maxval=50, group=grp_ict)
ictExpiry  = input.int(8, "ยกเลิก limit ของ ICT ถ้าไม่ติดใน (แท่ง)", minval=1, maxval=50, group=grp_ict)

// ══════════════════════════════════════════
// 6. Killzone (ช่วงเวลาที่ยอมเข้า)
// ══════════════════════════════════════════
grp_kz     = "6. Killzone"
useKZ      = input.bool(true, "เข้าเฉพาะช่วง Killzone (ใช้กับ ICT)", group=grp_kz)
kzTz       = input.string("America/New_York", "โซนเวลาที่ใช้อ้างอิง", options=["America/New_York","Europe/London","UTC","Asia/Bangkok"], group=grp_kz, tooltip="ค่าเริ่มต้นเป็นเวลานิวยอร์กตามตำรา ICT (ระบบจัดการ DST ให้เอง)")
kzUseLD    = input.bool(true,  "London Killzone", group=grp_kz)
kzLD       = input.session("0200-0500", "ช่วง London", group=grp_kz)
kzUseNY    = input.bool(true,  "New York AM Killzone", group=grp_kz)
kzNY       = input.session("0700-1000", "ช่วง New York AM", group=grp_kz)
kzUsePM    = input.bool(false, "PM / London Close", group=grp_kz)
kzPM       = input.session("1300-1600", "ช่วง PM", group=grp_kz)
kzShade    = input.bool(true, "ระบายพื้นหลังช่วง Killzone", group=grp_kz)

// ══════════════════════════════════════════
// 7. TP / SL
// ══════════════════════════════════════════
grp_tpsl   = "7. TP / SL"
slAtrMult  = input.float(1.2, "SL = ATR ×", minval=0.3, step=0.1, group=grp_tpsl)
maxSLusd   = input.float(12.0, "SL สูงสุด ($ ต่อ oz)", minval=1.0, step=0.5, group=grp_tpsl)
useSwingSL = input.bool(true, "ใช้ swing ถ้าอยู่ในระยะ", group=grp_tpsl)
rrTP       = input.float(1.2, "TP = SL ×", minval=0.5, step=0.1, group=grp_tpsl, tooltip="เป้าเดียว ไม่แบ่งไม้ — จากข้อมูล H1 568 ไม้: 77% แตะ 1R ได้ แต่มีแค่ 26% ที่ถึง 1.8R ยิ่งเป้าไกลยิ่งพลาดเยอะ")
atrLen     = input.int(14, "ATR Length", minval=1, group=grp_tpsl)

// ══════════════════════════════════════════
// 8. Limit Entry (ของฝั่ง SMC)
// ══════════════════════════════════════════
grp_lim    = "8. Limit Entry (SMC)"
useLimit   = input.bool(true, "รอราคาย่อก่อนเข้า (วาง limit)", group=grp_lim)
pullbackAtr= input.float(0.4, "รอราคาย่อกลับ (× ATR)", minval=0.1, step=0.1, group=grp_lim)
limitExpiry= input.int(6, "ยกเลิก limit ถ้าไม่ติดใน (แท่ง)", minval=1, maxval=50, group=grp_lim)

// ══════════════════════════════════════════
// 9. Sideway Filter
// ══════════════════════════════════════════
grp_rng    = "9. Sideway Filter"
useRangeFlt= input.bool(true, "ไม่เทรดตอนตลาด Sideway", group=grp_rng)
adxLen     = input.int(14, "ADX Length", minval=5, group=grp_rng)
adxMin     = input.float(20, "ADX ขั้นต่ำ", minval=10, maxval=40, step=1, group=grp_rng)
emaSlopeFlt= input.bool(true, "กรอง EMA แบนด้วย", group=grp_rng)
emaSepMin  = input.float(0.15, "EMA21-50 ต้องห่างกัน ≥ (× ATR)", minval=0, step=0.05, group=grp_rng)
showRange  = input.bool(true, "แสดงโซน Sideway", group=grp_rng)

// ══════════════════════════════════════════
// 10. หน้าตา
// ══════════════════════════════════════════
grp_st     = "10. หน้าตา"
buyColor   = input.color(color.new(#00E676, 0), "Buy", group=grp_st)
sellColor  = input.color(color.new(#FF1744, 0), "Sell", group=grp_st)
rangeClr   = input.color(color.new(#787878, 88), "โซน Sideway", group=grp_st)
kzClr      = input.color(color.new(#2962FF, 93), "โซน Killzone", group=grp_st)
zoneBullClr= input.color(color.new(#00E676, 80), "โซนเข้า ICT ฝั่ง Buy", group=grp_st)
zoneBearClr= input.color(color.new(#FF1744, 80), "โซนเข้า ICT ฝั่ง Sell", group=grp_st)
showStruct = input.bool(true, "แสดง BOS/CHoCH", group=grp_st)
showICTmk  = input.bool(true, "แสดงจุดกวาด liquidity + MSS", group=grp_st, tooltip="แสดงเสมอแม้เลือกโหมด SMC — เอาไว้ดูว่าโมเดล ICT มองเห็นอะไร")
showPanel  = input.bool(true, "แสดงตารางสรุป", group=grp_st)

// ══════════════════════════════════════════
// CORE
// ══════════════════════════════════════════
ema21  = ta.ema(close, emaFast)
ema50  = ta.ema(close, emaSlow)
rsiVal = ta.rsi(close, rsiLen)
atrVal = ta.atr(atrLen)
avgVol = ta.sma(volume, 20)

trendUp = ema21 > ema50
trendDn = ema21 < ema50

[diP_, diM_, adxVal] = ta.dmi(adxLen, adxLen)
emaSep   = math.abs(ema21 - ema50)
isRange  = adxVal < adxMin or (emaSlopeFlt and emaSep < emaSepMin * atrVal)
trending = not useRangeFlt or not isRange

chartSec = timeframe.in_seconds(timeframe.period)

f_htfBias(_tf) =>
    [a21, a50] = request.security(syminfo.tickerid, _tf, [ta.ema(close, emaFast), ta.ema(close, emaSlow)], lookahead=barmerge.lookahead_off)
    [b21, b50] = request.security(syminfo.tickerid, _tf, [ta.ema(close, emaFast)[1], ta.ema(close, emaSlow)[1]], lookahead=barmerge.lookahead_on)
    e21 = noRepaint ? b21 : a21
    e50 = noRepaint ? b50 : a50
    e21 > e50 ? 1 : e21 < e50 ? -1 : 0

raw15  = f_htfBias("15")
raw30  = f_htfBias("30")
raw60  = f_htfBias("60")
raw240 = f_htfBias("240")

confSec   = mtfConfirm == "M15" ? timeframe.in_seconds("15") : mtfConfirm == "M30" ? timeframe.in_seconds("30") : mtfConfirm == "H1" ? timeframe.in_seconds("60") : timeframe.in_seconds("240")
confRaw   = mtfConfirm == "M15" ? raw15 : mtfConfirm == "M30" ? raw30 : mtfConfirm == "H1" ? raw60 : raw240
confIsHTF = confSec > chartSec
localBias = trendUp ? 1 : trendDn ? -1 : 0
confirmBias = confIsHTF ? confRaw : localBias

// ── เครื่องคิด lot (ยกมาไว้ข้างบนเพราะทั้ง SMC และ ICT ใช้ร่วมกัน) ──
f_lot(_slDist) =>
    riskUsd = acctSize * riskPct / 100.0
    perLot  = _slDist * ozPerLot
    raw     = perLot > 0 ? riskUsd / perLot : 0.0
    stepped = lotStep > 0 ? math.floor(raw / lotStep + 1e-9) * lotStep : raw
    stepped

f_riskOf(_slDist, _lot) => _slDist * ozPerLot * _lot

// ══════════════════════════════════════════
// SWING / BOS / CHoCH  (ฝั่ง SMC)
// ══════════════════════════════════════════
swingHigh = ta.pivothigh(high, swingLen, swingLen)
swingLow  = ta.pivotlow(low,  swingLen, swingLen)

var float swH1 = na
var float swL1 = na
if not na(swingHigh)
    swH1 := swingHigh
if not na(swingLow)
    swL1 := swingLow

var int  marketBias = 0
var bool brokeH = false
var bool brokeL = false
bosUp   = false
bosDn   = false
chochUp = false
chochDn = false

if not na(swingHigh)
    brokeH := false
if not na(swingLow)
    brokeL := false

if not na(swH1) and close > swH1 and not brokeH
    if marketBias == 1
        bosUp := true
    else
        chochUp := true
        marketBias := 1
        brokeL := false
    brokeH := true

if not na(swL1) and close < swL1 and not brokeL
    if marketBias == -1
        bosDn := true
    else
        chochDn := true
        marketBias := -1
        brokeH := false
    brokeL := true

if showStruct and barstate.isconfirmed
    if bosUp
        label.new(bar_index, high, "BOS", style=label.style_label_down, color=color.new(buyColor, 30), textcolor=color.white, size=size.tiny)
    if bosDn
        label.new(bar_index, low, "BOS", style=label.style_label_up, color=color.new(sellColor, 30), textcolor=color.white, size=size.tiny)
    if chochUp
        label.new(bar_index, high, "CHoCH", style=label.style_label_down, color=buyColor, textcolor=color.white, size=size.small)
    if chochDn
        label.new(bar_index, low, "CHoCH", style=label.style_label_up, color=sellColor, textcolor=color.white, size=size.small)

// ══════════════════════════════════════════
// ENTRY CONDITIONS (ฝั่ง SMC)
// ══════════════════════════════════════════
// rsiUseInp = false  → ตรรกะเดิมที่ผ่าน backtest (เลข 35/65 ตายตัว)
// rsiUseInp = true   → ช่อง RSI OB/OS ใช้งานได้จริง (แก้บั๊ก B2 ใน REVIEW_v9)
rsiOK_buy  = rsiUseInp ? (rsiVal > rsiOS and rsiVal < math.min(rsiOB, noChaseHi)) : (rsiVal > 35 and rsiVal < rsiOB and rsiVal < noChaseHi)
rsiOK_sell = rsiUseInp ? (rsiVal < rsiOB and rsiVal > math.max(rsiOS, noChaseLo)) : (rsiVal < 65 and rsiVal > rsiOS and rsiVal > noChaseLo)
volOK      = not volFilter or na(volume) or na(avgVol) or volume > avgVol * 0.7

bullEngulf  = close[1] < open[1] and close > open and close > open[1] and open <= close[1]
bearEngulf  = close[1] > open[1] and close < open and close < open[1] and open >= close[1]
bodySize    = math.abs(close - open)
candleRange = high - low
bullPinBar  = candleRange > 0 and (math.min(open, close) - low) > bodySize * 2 and close > open
bearPinBar  = candleRange > 0 and (high - math.max(open, close)) > bodySize * 2 and close < open

bullMomentum = useMomentum and close > open and bodySize > atrVal * 0.4 and rsiVal > rsiVal[1] and rsiVal < noChaseHi
bearMomentum = useMomentum and close < open and bodySize > atrVal * 0.4 and rsiVal < rsiVal[1] and rsiVal > noChaseLo

bullTrigger = bullEngulf or bullPinBar or bullMomentum
bearTrigger = bearEngulf or bearPinBar or bearMomentum

mtfBuyOK  = not useMTF or (strictMTF ? confirmBias == 1  : confirmBias >= 0)
mtfSellOK = not useMTF or (strictMTF ? confirmBias == -1 : confirmBias <= 0)

buyScore  = (trendUp ? 1 : 0) + (rsiOK_buy ? 1 : 0) + (bullTrigger ? 1 : 0) + (volOK ? 1 : 0) + (marketBias == 1 ? 1 : 0) + (chochUp ? 1 : 0) + (bosUp ? 1 : 0)
sellScore = (trendDn ? 1 : 0) + (rsiOK_sell ? 1 : 0) + (bearTrigger ? 1 : 0) + (volOK ? 1 : 0) + (marketBias == -1 ? 1 : 0) + (chochDn ? 1 : 0) + (bosDn ? 1 : 0)

// ══════════════════════════════════════════
// ICT — 1. Killzone
// ══════════════════════════════════════════
// เขียนตรง ๆ ไม่ห่อเป็นฟังก์ชัน — time() ต้องการ session/timezone เป็น simple string
// ถ้าส่งผ่าน parameter ของฟังก์ชัน Pine อาจ infer เป็น series แล้ว compile ไม่ผ่าน
sessOK = chartSec < 86400
inLD = kzUseLD and sessOK and not na(time(timeframe.period, kzLD, kzTz))
inNY = kzUseNY and sessOK and not na(time(timeframe.period, kzNY, kzTz))
inPM = kzUsePM and sessOK and not na(time(timeframe.period, kzPM, kzTz))
anyKZon = kzUseLD or kzUseNY or kzUsePM
// TF รายวันขึ้นไปไม่มีความหมายเรื่อง session → ปล่อยผ่าน ไม่บล็อกทิ้ง
inKZ = not useKZ or not anyKZon or chartSec >= 86400 or inLD or inNY or inPM
kzNow = inLD or inNY or inPM

// ══════════════════════════════════════════
// ICT — 2. Liquidity (short-term high / low)
// ══════════════════════════════════════════
ictPH = ta.pivothigh(high, ictPivLen, ictPivLen)
ictPL = ta.pivotlow(low,  ictPivLen, ictPivLen)

var float liqHi    = na
var int   liqHiBar = na
var float liqLo    = na
var int   liqLoBar = na
if not na(ictPH)
    liqHi := ictPH
    liqHiBar := bar_index - ictPivLen
if not na(ictPL)
    liqLo := ictPL
    liqLoBar := bar_index - ictPivLen

recentHigh = ta.highest(high, ictMssLook)
recentLow  = ta.lowest(low,  ictMssLook)
// กรอบราคาที่ใช้วัด premium / discount — ต้องยาวพอจะครอบขาที่ลงมากวาด
drHi = ta.highest(high, ictDrLook)
drLo = ta.lowest(low,  ictDrLook)
enoughBars = bar_index > math.max(60, ictDrLook + ictZoneScan + 5)

// ══════════════════════════════════════════
// ICT — 3. State machine: sweep → MSS
// bull setup = กวาด sell-side liquidity (ใต้ low) แล้วพลิกขึ้น
// bear setup = กวาด buy-side liquidity (เหนือ high) แล้วพลิกลง
// ══════════════════════════════════════════
var bool  bArm = false
var int   bArmBar = na
var float bLow = na          // จุดต่ำสุดของการกวาด → ที่วาง SL
var float bMss = na          // ระดับที่ต้องปิดเหนือถึงจะนับว่าพลิกโครงสร้าง
var float bTaken = na        // ระดับ liquidity ที่กวาดไปแล้ว (กันยิงซ้ำจุดเดิม)
var float bDrHi = na         // ยอดของกรอบตอนที่กวาด — ใช้วัด discount และเป็นเป้า liquidity

var bool  sArm = false
var int   sArmBar = na
var float sHigh = na
var float sMss = na
var float sTaken = na
var float sDrLo = na

// ผลลัพธ์ของแท่งนี้ (ไม่ใช่ var — รีเซ็ตทุกแท่ง)
bool  sweptSSL   = false     // เพิ่งกวาดฝั่งล่าง
bool  sweptBSL   = false
bool  mssBuyRaw  = false
bool  mssSellRaw = false
float mssBuyLow  = na        // snapshot ค่าตอน MSS (กันโดนทับถ้ามี sweep ใหม่ในแท่งเดียวกัน)
int   mssBuyArm  = na
float mssBuyDr   = na
float mssSellHi  = na
int   mssSellArm = na
float mssSellDr  = na

dispOK = not ictNeedDsp or bodySize > ictDspAtr * atrVal

if barstate.isconfirmed and enoughBars
    // ── 3.1 ตรวจว่า setup ที่ถืออยู่ยังใช้ได้มั้ย (ใช้ค่าเดิมก่อนอัปเดต) ──
    if bArm and (bar_index - bArmBar > ictArmAge or close < bLow)
        bArm := false
    if sArm and (bar_index - sArmBar > ictArmAge or close > sHigh)
        sArm := false

    // ── 3.2 ระหว่างรอ MSS ถ้าลงลึกกว่าเดิมแต่ยังปิดเหนือ ให้ขยับจุด SL ตาม ──
    if bArm
        bLow := math.min(bLow, low)
    if sArm
        sHigh := math.max(sHigh, high)

    // ── 3.3 MSS: ปิดทะลุ high/low ระยะสั้นฝั่งตรงข้ามที่กวาดมา ──
    if bArm and bar_index > bArmBar and close > bMss and dispOK
        mssBuyRaw := true
        mssBuyLow := bLow
        mssBuyArm := bArmBar
        mssBuyDr  := bDrHi
        bArm := false
    if sArm and bar_index > sArmBar and close < sMss and dispOK
        mssSellRaw := true
        mssSellHi  := sHigh
        mssSellArm := sArmBar
        mssSellDr  := sDrLo
        sArm := false

    // ── 3.4 หา sweep ใหม่ (ทำหลัง MSS เสมอ เพื่อไม่ให้เกิดสองอย่างในแท่งเดียว) ──
    liqLoFresh = not na(liqLo) and not na(liqLoBar) and (bar_index - liqLoBar) <= ictLiqAge
    liqHiFresh = not na(liqHi) and not na(liqHiBar) and (bar_index - liqHiBar) <= ictLiqAge
    if not bArm and liqLoFresh and low < liqLo and close > liqLo and (na(bTaken) or bTaken != liqLo)
        bArm := true
        bArmBar := bar_index
        bLow := low
        bMss := recentHigh
        bTaken := liqLo
        bDrHi := drHi
        sweptSSL := true
    if not sArm and liqHiFresh and high > liqHi and close < liqHi and (na(sTaken) or sTaken != liqHi)
        sArm := true
        sArmBar := bar_index
        sHigh := high
        sMss := recentLow
        sTaken := liqHi
        sDrLo := drLo
        sweptBSL := true

// ══════════════════════════════════════════
// ICT — 4. หาโซนเข้า (FVG ก่อน ถ้าไม่มีค่อย OB)
// เรียกทุกแท่งที่ scope นอกสุด เพื่อไม่ให้ Pine งงเรื่อง history buffer
// คืนค่า: [ขอบบน, ขอบล่าง, offset ของแท่งซ้ายสุด, ชนิดโซน]
// ══════════════════════════════════════════
f_bullZone(_scan, _useOB) =>
    float zt = na
    float zb = na
    int   zi = na
    string zk = ""
    for i = 0 to _scan
        if na(zt) and low[i] > high[i + 2] and close[i + 1] > open[i + 1]
            zt := low[i]
            zb := high[i + 2]
            zi := i + 2
            zk := "FVG"
    if na(zt) and _useOB
        for i = 0 to _scan
            if na(zt) and close[i] < open[i]
                zt := high[i]
                zb := low[i]
                zi := i
                zk := "OB"
    [zt, zb, zi, zk]

f_bearZone(_scan, _useOB) =>
    float zt = na
    float zb = na
    int   zi = na
    string zk = ""
    for i = 0 to _scan
        if na(zt) and high[i] < low[i + 2] and close[i + 1] < open[i + 1]
            zt := low[i + 2]
            zb := high[i]
            zi := i + 2
            zk := "FVG"
    if na(zt) and _useOB
        for i = 0 to _scan
            if na(zt) and close[i] > open[i]
                zt := high[i]
                zb := low[i]
                zi := i
                zk := "OB"
    [zt, zb, zi, zk]

scanB = mssBuyRaw  ? math.max(0, math.min(ictZoneScan, bar_index - mssBuyArm))  : 0
scanS = mssSellRaw ? math.max(0, math.min(ictZoneScan, bar_index - mssSellArm)) : 0
[bzTop, bzBot, bzOff, bzKind] = f_bullZone(scanB, ictUseOB)
[szTop, szBot, szOff, szKind] = f_bearZone(scanS, ictUseOB)


// ══════════════════════════════════════════
// ICT — 5. ประกอบเป็นไม้ (entry / SL / TP)
// ══════════════════════════════════════════
bzMid = (bzTop + bzBot) / 2
szMid = (szTop + szBot) / 2
ictBuyEntry  = ictEntMode == ENT_CE ? bzMid : ictEntMode == ENT_FAR ? bzBot : bzTop
ictSellEntry = ictEntMode == ENT_CE ? szMid : ictEntMode == ENT_FAR ? szTop : szBot

ictBuySL   = mssBuyLow  - ictSlBuf * atrVal
ictSellSL  = mssSellHi  + ictSlBuf * atrVal
ictBuyDist = ictBuyEntry - ictBuySL
ictSellDist= ictSellSL - ictSellEntry

// TP โหมด liquidity = ยอด/ก้นของกรอบ ซึ่งเป็นที่ที่ stop ฝั่งตรงข้ามกองอยู่
buyLiqTgt  = mssBuyDr
sellLiqTgt = mssSellDr
ictBuyTP   = ictTpMode == TP_R ? ictBuyEntry  + ictBuyDist  * rrTP : buyLiqTgt
ictSellTP  = ictTpMode == TP_R ? ictSellEntry - ictSellDist * rrTP : sellLiqTgt
ictBuyRR   = ictBuyDist  > 0 ? (ictBuyTP - ictBuyEntry) / ictBuyDist   : na
ictSellRR  = ictSellDist > 0 ? (ictSellEntry - ictSellTP) / ictSellDist : na
buyRRok    = ictTpMode == TP_R or (not na(ictBuyRR)  and ictBuyRR  >= ictMinRR)
sellRRok   = ictTpMode == TP_R or (not na(ictSellRR) and ictSellRR >= ictMinRR)

// premium / discount วัดจากกรอบ [จุดที่กวาด ↔ ยอดกรอบก่อนหน้า]
eqB = (mssBuyDr + mssBuyLow) / 2
eqS = (mssSellHi + mssSellDr) / 2
buyDscOK  = not ictNeedDsc or ictBuyEntry  <= eqB
sellPrmOK = not ictNeedDsc or ictSellEntry >= eqS

// OTE 62-79% ของกรอบเดียวกัน
legB   = mssBuyDr - mssBuyLow
legS   = mssSellHi - mssSellDr
oteBhi = mssBuyDr - 0.62 * legB
oteBlo = mssBuyDr - 0.79 * legB
oteShi = mssSellDr + 0.79 * legS
oteSlo = mssSellDr + 0.62 * legS
buyOteOK  = not ictNeedOTE or (legB > 0 and ictBuyEntry  >= oteBlo and ictBuyEntry  <= oteBhi)
sellOteOK = not ictNeedOTE or (legS > 0 and ictSellEntry >= oteSlo and ictSellEntry <= oteShi)

ictBuyLotIdeal  = f_lot(ictBuyDist)
ictSellLotIdeal = f_lot(ictSellDist)
ictBuyLot  = math.max(ictBuyLotIdeal,  minLot)
ictSellLot = math.max(ictSellLotIdeal, minLot)
ictBuyAfford  = not blockOver or ictBuyLotIdeal  >= minLot
ictSellAfford = not blockOver or ictSellLotIdeal >= minLot

ictBuySLok  = ictBuyDist  > 0 and (not ictWideSkip or ictBuyDist  <= maxSLusd)
ictSellSLok = ictSellDist > 0 and (not ictWideSkip or ictSellDist <= maxSLusd)

ictTrendOK = not ictUseRng or trending
ictScoreBuyOK  = not modeBothCfm or buyScore  >= minScore
ictScoreSellOK = not modeBothCfm or sellScore >= minScore

ictBuyValid = useICTsig and mssBuyRaw and not na(bzTop) and ictBuyEntry < close and
     ictBuySLok and buyRRok and buyDscOK and buyOteOK and ictBuyAfford and
     inKZ and ictTrendOK and mtfBuyOK and ictScoreBuyOK
ictSellValid = useICTsig and mssSellRaw and not na(szTop) and ictSellEntry > close and
     ictSellSLok and sellRRok and sellPrmOK and sellOteOK and ictSellAfford and
     inKZ and ictTrendOK and mtfSellOK and ictScoreSellOK

// ══════════════════════════════════════════
// TP / SL ฝั่ง SMC (เหมือน v9.4 ทุกบรรทัด)
// ══════════════════════════════════════════
slDistBase = math.min(atrVal * slAtrMult, maxSLusd)

buyEntry  = useLimit ? close - pullbackAtr * atrVal : close
sellEntry = useLimit ? close + pullbackAtr * atrVal : close

buySwingDist = not na(swL1) and swL1 < buyEntry ? buyEntry - swL1 : na
buyUseSwing  = useSwingSL and not na(buySwingDist) and buySwingDist <= maxSLusd
buySLdist    = buyUseSwing ? math.min(buySwingDist + atrVal * 0.1, maxSLusd) : slDistBase
finalBuySL   = buyEntry - buySLdist
finalBuyTP   = buyEntry + buySLdist * rrTP

sellSwingDist = not na(swH1) and swH1 > sellEntry ? swH1 - sellEntry : na
sellUseSwing  = useSwingSL and not na(sellSwingDist) and sellSwingDist <= maxSLusd
sellSLdist    = sellUseSwing ? math.min(sellSwingDist + atrVal * 0.1, maxSLusd) : slDistBase
finalSellSL   = sellEntry + sellSLdist
finalSellTP   = sellEntry - sellSLdist * rrTP

buyLotIdeal  = f_lot(buySLdist)
sellLotIdeal = f_lot(sellSLdist)
buyLotUse    = math.max(buyLotIdeal, minLot)
sellLotUse   = math.max(sellLotIdeal, minLot)
buyAffordOK  = not blockOver or buyLotIdeal  >= minLot
sellAffordOK = not blockOver or sellLotIdeal >= minLot
buyRiskUsd_  = f_riskOf(buySLdist,  buyLotUse)
sellRiskUsd_ = f_riskOf(sellSLdist, sellLotUse)
buyRiskPct_  = acctSize > 0 ? buyRiskUsd_  / acctSize * 100 : 0.0
sellRiskPct_ = acctSize > 0 ? sellRiskUsd_ / acctSize * 100 : 0.0

// ══════════════════════════════════════════
// สถานะไม้ — ว่าง / รอ limit / เปิดอยู่
// ══════════════════════════════════════════
var int    stState  = 0        // 0 ว่าง, 1 รอ limit ติด, 2 เปิดอยู่
var bool   stIsBuy  = false
var string stKind   = ""
var float  stEntry  = na
var float  stSL     = na
var float  stTP     = na
var float  stLot    = na
var int    stBar    = na
var int    stExp    = na
var int    lastSigBar = na

canSignal    = na(lastSigBar) or (bar_index - lastSigBar) >= signalGap
canSignalIct = na(lastSigBar) or (bar_index - lastSigBar) >= ictGap
free         = stState == 0

baseBuy  = bullTrigger and mtfBuyOK  and trending and buySLdist  > 0 and buyAffordOK
baseSell = bearTrigger and mtfSellOK and trending and sellSLdist > 0 and sellAffordOK

smcBuySig  = useSMCsig and barstate.isconfirmed and buyScore  >= minScore and baseBuy  and canSignal and free
smcSellSig = useSMCsig and barstate.isconfirmed and sellScore >= minScore and baseSell and canSignal and free
ictBuySig  = ictBuyValid  and canSignalIct and free
ictSellSig = ictSellValid and canSignalIct and free

// ลำดับความสำคัญ: ICT ก่อน แล้วค่อย SMC — และห้ามยิงสองทางในแท่งเดียว
takeIctBuy  = ictBuySig
takeIctSell = ictSellSig and not takeIctBuy
takeSmcBuy  = smcBuySig  and not takeIctBuy and not takeIctSell
takeSmcSell = smcSellSig and not takeIctBuy and not takeIctSell and not takeSmcBuy

buySignal  = takeIctBuy or takeSmcBuy
sellSignal = takeIctSell or takeSmcSell
anySignal  = buySignal or sellSignal
sigIsICT   = takeIctBuy or takeIctSell

// ค่าที่จะใช้จริงของไม้นี้
selEntry = takeIctBuy ? ictBuyEntry : takeIctSell ? ictSellEntry : takeSmcBuy ? buyEntry : sellEntry
selSL    = takeIctBuy ? ictBuySL    : takeIctSell ? ictSellSL    : takeSmcBuy ? finalBuySL : finalSellSL
selTP    = takeIctBuy ? ictBuyTP    : takeIctSell ? ictSellTP    : takeSmcBuy ? finalBuyTP : finalSellTP
selDist  = takeIctBuy ? ictBuyDist  : takeIctSell ? ictSellDist  : takeSmcBuy ? buySLdist  : sellSLdist
selLot   = takeIctBuy ? ictBuyLot   : takeIctSell ? ictSellLot   : takeSmcBuy ? buyLotUse  : sellLotUse
selKind  = takeIctBuy ? "ICT " + bzKind : takeIctSell ? "ICT " + szKind : "SMC"
selExp   = sigIsICT ? ictExpiry : limitExpiry
selIsLim = sigIsICT or useLimit
selRiskUsd = f_riskOf(selDist, selLot)
selRiskPct = acctSize > 0 ? selRiskUsd / acctSize * 100 : 0.0
selRewUsd  = f_riskOf(math.abs(selTP - selEntry), selLot)

// ── เปิดสัญญาณใหม่ ──
if anySignal
    stState := selIsLim ? 1 : 2
    stIsBuy := buySignal
    stKind  := selKind
    stEntry := selEntry
    stSL    := selSL
    stTP    := selTP
    stLot   := selLot
    stBar   := bar_index
    stExp   := selExp
    lastSigBar := bar_index

// ── เดินสถานะ ──
filledNow  = false
expiredNow = false
tpNow      = false
slNow      = false

if barstate.isconfirmed and stState == 1 and bar_index > stBar
    touched = stIsBuy ? low <= stEntry : high >= stEntry
    if touched
        stState := 2
        filledNow := true
    else if bar_index - stBar >= stExp
        stState := 0
        expiredNow := true

// หมายเหตุ: ถ้าแท่งเดียวกันทั้งติดทั้งโดน เราให้ SL ชนะไว้ก่อน (นับแบบระวัง)
if barstate.isconfirmed and stState == 2 and bar_index > stBar
    hitSL = stIsBuy ? low  <= stSL : high >= stSL
    hitTP = stIsBuy ? high >= stTP : low  <= stTP
    if hitSL
        stState := 0
        slNow := true
    else if hitTP
        stState := 0
        tpNow := true

// ══════════════════════════════════════════
// วาดบนกราฟ
// ══════════════════════════════════════════
f_p(_x) => str.tostring(_x, "#.##")
f_m(_x) => str.tostring(_x, "#.##")

// ── เครื่องหมายของโมเดล ICT (แสดงแม้จะไม่ได้ใช้ยิงสัญญาณ) ──
if showICTmk and barstate.isconfirmed
    if sweptSSL
        label.new(bar_index, low, "✕ SSL", style=label.style_label_up, color=color.new(#FFB300, 25), textcolor=color.black, size=size.tiny, tooltip="กวาด sell-side liquidity แล้วปิดกลับเหนือ low เดิม")
    if sweptBSL
        label.new(bar_index, high, "✕ BSL", style=label.style_label_down, color=color.new(#FFB300, 25), textcolor=color.black, size=size.tiny, tooltip="กวาด buy-side liquidity แล้วปิดกลับใต้ high เดิม")
    if mssBuyRaw
        label.new(bar_index, high, "MSS ↑", style=label.style_label_down, color=color.new(#2962FF, 20), textcolor=color.white, size=size.tiny)
        line.new(mssBuyArm, bMss[1], bar_index, bMss[1], color=color.new(#2962FF, 30), style=line.style_dotted, width=1)
    if mssSellRaw
        label.new(bar_index, low, "MSS ↓", style=label.style_label_up, color=color.new(#2962FF, 20), textcolor=color.white, size=size.tiny)
        line.new(mssSellArm, sMss[1], bar_index, sMss[1], color=color.new(#2962FF, 30), style=line.style_dotted, width=1)

// ── โซนเข้าของ ICT ──
if takeIctBuy
    box.new(bar_index - bzOff, bzTop, bar_index + stExp, bzBot, bgcolor=zoneBullClr, border_color=color.new(#00E676, 40), border_width=1)
if takeIctSell
    box.new(bar_index - szOff, szTop, bar_index + stExp, szBot, bgcolor=zoneBearClr, border_color=color.new(#FF1744, 40), border_width=1)

// ── ป้ายสัญญาณ ──
if anySignal
    dirTxt = buySignal ? "BUY" : "SELL"
    rrTxt  = selDist > 0 ? str.tostring(math.abs(selTP - selEntry) / selDist, "#.##") : "-"
    txt = dirTxt + (selIsLim ? " LIMIT" : "") + "  [" + selKind + "]" +
         "\nเข้า " + f_p(selEntry) +
         "\nTP  " + f_p(selTP) + "   +$" + f_m(selRewUsd) + "  (" + rrTxt + "R)" +
         "\nSL  " + f_p(selSL) + "   -$" + f_m(selRiskUsd) +
         "\nLot " + str.tostring(selLot, "#.###") + "   เสี่ยง " + str.tostring(selRiskPct, "#.#") + "%" +
         (selIsLim ? "\nยกเลิกถ้าไม่ติดใน " + str.tostring(selExp) + " แท่ง" : "")
    label.new(bar_index, buySignal ? low : high, txt, style=buySignal ? label.style_label_up : label.style_label_down, color=buySignal ? buyColor : sellColor, textcolor=color.white, size=size.normal)
    line.new(bar_index, selTP,    bar_index + 12, selTP,    color=color.new(buySignal ? #00E676 : #FF1744, 20), style=line.style_dashed, width=2)
    line.new(bar_index, selEntry, bar_index + 12, selEntry, color=color.new(#FFEB3B, 20), style=line.style_dotted, width=1)
    line.new(bar_index, selSL,    bar_index + 12, selSL,    color=color.new(buySignal ? #FF1744 : #00E676, 20), style=line.style_dashed, width=2)

if filledNow
    label.new(bar_index, stIsBuy ? low : high, "ติดแล้ว", style=stIsBuy ? label.style_label_up : label.style_label_down, color=color.new(#FFEB3B, 20), textcolor=color.black, size=size.tiny)
if expiredNow
    label.new(bar_index, close, "ยกเลิก limit", style=label.style_label_left, color=color.new(color.gray, 20), textcolor=color.white, size=size.tiny)
if tpNow
    label.new(bar_index, close, "TP", style=label.style_label_left, color=color.new(#00E676, 20), textcolor=color.white, size=size.tiny)
if slNow
    label.new(bar_index, close, "SL", style=label.style_label_left, color=color.new(#FF1744, 20), textcolor=color.white, size=size.tiny)

// ══════════════════════════════════════════
// ทำไมถึงยังไม่ยิงสัญญาณ — ไล่หาตัวที่บล็อกอยู่จริง
// เพิ่มเพราะอาการ "เปิดอินดี้แล้วไม่มีอะไรขึ้นเลย" แยกไม่ออกว่าเงื่อนไข
// ยังไม่ครบ หรือโดน risk gate ตัดทิ้งตั้งแต่ต้น
// ══════════════════════════════════════════
// ทุนขั้นต่ำที่ต้องมี ถึงจะเข้าไม้ด้วย lot ขั้นต่ำของโบรกแล้วยังอยู่ในเพดานเสี่ยง
refSLdist   = math.min(buySLdist, sellSLdist)
minAcctNeed = riskPct > 0 ? refSLdist * ozPerLot * minLot / (riskPct / 100) : na
affordAny   = buyAffordOK or sellAffordOK
bestScore   = math.max(buyScore, sellScore)
gapLeft     = signalGap - (bar_index - nz(lastSigBar, bar_index))

TXT_READY = "เงื่อนไขครบ — รอแท่งปิด"
blockTxt =
     stState != 0        ? "กำลังถือไม้ (1 ไม้ต่อครั้ง)" :
     not affordAny       ? "ทุนไม่พอ — ต้องมี ≥ $" + str.tostring(minAcctNeed, "#") :
     not trending        ? "Sideway — ADX " + str.tostring(adxVal, "#.#") + " < " + str.tostring(adxMin, "#") :
     modeICTonly and not inKZ ? "นอกเวลา Killzone" :
     not canSignal       ? "เว้นระยะสัญญาณ อีก " + str.tostring(gapLeft) + " แท่ง" :
     not (mtfBuyOK or mtfSellOK) ? "MTF สวนทาง (" + mtfConfirm + " = " + (confirmBias == 1 ? "UP" : confirmBias == -1 ? "DOWN" : "-") + ")" :
     useSMCsig and bestScore < minScore ? "คะแนนได้ " + str.tostring(bestScore) + " / ต้องการ " + str.tostring(minScore) :
     useSMCsig and not (bullTrigger or bearTrigger) ? "รอแท่งเทียน trigger" :
     modeICTonly        ? "รอ sweep แล้ว MSS" :
     TXT_READY
blockCol = stState != 0 ? color.orange : not affordAny ? color.new(#FF1744, 0) : blockTxt == TXT_READY ? color.new(#00E676, 0) : color.gray

// ══════════════════════════════════════════
// ตารางสรุป
// ══════════════════════════════════════════
f_cell(_b) => _b == 1 ? "UP" : _b == -1 ? "DOWN" : "-"
f_clr(_b)  => _b == 1 ? buyColor : _b == -1 ? sellColor : color.gray

// premium / discount ของกรอบล่าสุด (ดูภาพรวม ไม่เกี่ยวกับ setup ที่ถืออยู่)
drRange = drHi - drLo
pdPct   = drRange > 0 ? (close - drLo) / drRange * 100 : na
pdTxt   = na(pdPct) ? "-" : (pdPct > 55 ? "Premium " : pdPct < 45 ? "Discount " : "EQ ") + str.tostring(pdPct, "#") + "%"
pdCol   = na(pdPct) ? color.gray : pdPct > 55 ? sellColor : pdPct < 45 ? buyColor : color.orange

ictStTxt = bArm ? "กวาด SSL แล้ว รอ MSS (" + str.tostring(ictArmAge - (bar_index - bArmBar)) + ")" : sArm ? "กวาด BSL แล้ว รอ MSS (" + str.tostring(ictArmAge - (bar_index - sArmBar)) + ")" : "รอ liquidity sweep"
ictStCol = bArm ? buyColor : sArm ? sellColor : color.gray

var table t = table.new(position.top_right, 2, 16, bgcolor=color.new(#1a1a2e, 8), border_width=1, border_color=color.new(color.gray, 70))
if showPanel and barstate.islast
    table.cell(t, 0, 0, "SMC v9.5", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#0f3460, 20))
    table.cell(t, 1, 0, "ICT", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#0f3460, 20))

    stTxt = stState == 0 ? "รอสัญญาณ" : stState == 1 ? "รอ limit ติด (" + str.tostring(stExp - (bar_index - stBar)) + " แท่ง)" : (stIsBuy ? "ถือ BUY" : "ถือ SELL")
    stCol = stState == 0 ? color.gray : stState == 1 ? color.orange : (stIsBuy ? buyColor : sellColor)
    table.cell(t, 0, 1, "สถานะ", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 1, stTxt, text_color=stCol, text_size=size.tiny)

    table.cell(t, 0, 2, "ไม้นี้มาจาก", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 2, stState == 0 ? "-" : stKind, text_color=stState == 0 ? color.gray : color.white, text_size=size.tiny)

    table.cell(t, 0, 3, "เข้า / SL / TP", text_color=color.gray, text_size=size.tiny)
    lvlTxt = stState == 0 ? "-" : f_p(stEntry) + " / " + f_p(stSL) + " / " + f_p(stTP)
    table.cell(t, 1, 3, lvlTxt, text_color=color.white, text_size=size.tiny)

    table.cell(t, 0, 4, "Lot ที่ใช้", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 4, stState == 0 ? "-" : str.tostring(stLot, "#.###"), text_color=color.white, text_size=size.tiny)

    table.cell(t, 0, 5, "ระบบที่ใช้", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 5, modeSMConly ? "SMC (ผ่าน backtest)" : modeICTonly ? "ICT (ยังไม่ผ่าน)" : modeBothCfm ? "ICT+SMC ยืนยันกัน" : "ICT + SMC", text_color=modeSMConly ? color.new(#00E676,0) : color.orange, text_size=size.tiny)

    table.cell(t, 0, 6, "ICT setup", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 6, ictStTxt, text_color=ictStCol, text_size=size.tiny)

    table.cell(t, 0, 7, "Killzone", text_color=color.gray, text_size=size.tiny)
    kzTxt = not useKZ ? "ปิดฟิลเตอร์" : chartSec >= 86400 ? "n/a (TF ใหญ่)" : kzNow ? (inLD ? "London" : inNY ? "New York" : "PM") : "นอกเวลา — ICT ไม่เข้า"
    table.cell(t, 1, 7, kzTxt, text_color=not useKZ ? color.gray : kzNow or chartSec >= 86400 ? color.new(#00E676,0) : color.orange, text_size=size.tiny)

    table.cell(t, 0, 8, "Premium/Discount", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 8, pdTxt, text_color=pdCol, text_size=size.tiny)

    nextRisk = math.min(buyRiskPct_, sellRiskPct_)
    affordOK = math.min(buyLotIdeal, sellLotIdeal) >= minLot
    riskCol  = affordOK ? color.new(#00E676,0) : color.new(#FF1744,0)
    table.cell(t, 0, 9, "ถ้าเข้าตอนนี้ (SMC)", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 9, affordOK ? "เสี่ยง " + str.tostring(nextRisk, "#.#") + "%" : "เสี่ยง " + str.tostring(nextRisk, "#.#") + "% — ต้องมีทุน ≥ $" + str.tostring(minAcctNeed, "#"), text_color=riskCol, text_size=size.tiny)

    table.cell(t, 0, 10, "lot ที่ควรใช้", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 10, affordOK ? str.tostring(math.min(buyLotIdeal, sellLotIdeal), "#.###") : "ต่ำกว่า " + str.tostring(minLot, "#.###"), text_color=riskCol, text_size=size.tiny)

    table.cell(t, 0, 11, "ทุน / เสี่ยง", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 11, "$" + f_m(acctSize) + " / " + str.tostring(riskPct, "#.#") + "%", text_color=color.white, text_size=size.tiny)

    table.cell(t, 0, 12, "ยืนยันด้วย " + mtfConfirm, text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 12, confIsHTF ? f_cell(confirmBias) : f_cell(confirmBias) + " (ใช้กราฟ)", text_color=confIsHTF ? f_clr(confirmBias) : color.orange, text_size=size.tiny)

    table.cell(t, 0, 13, "ATR / ADX", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 13, f_m(atrVal) + " / " + str.tostring(adxVal, "#.#"), text_color=trending ? color.white : color.orange, text_size=size.tiny)

    table.cell(t, 0, 14, "MTF repaint", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 14, noRepaint ? "กันแล้ว" : "⚠ ยังไม่กัน", text_color=noRepaint ? color.new(#00E676,0) : color.new(#FF1744,0), text_size=size.tiny)

    table.cell(t, 0, 15, "ทำไมยังไม่ยิง", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 15, blockTxt, text_color=blockCol, text_size=size.tiny)

// ══════════════════════════════════════════
// ALERTS
// ══════════════════════════════════════════
if anySignal
    alert("XAUUSD " + (buySignal ? "BUY" : "SELL") + (selIsLim ? " LIMIT" : " ตลาด") +
         " [" + selKind + "]" +
         " | เข้า " + f_p(selEntry) +
         " | SL " + f_p(selSL) +
         " | TP " + f_p(selTP) +
         " | Lot " + str.tostring(selLot, "#.###") +
         " | เสี่ยง $" + f_m(selRiskUsd) + " (" + str.tostring(selRiskPct, "#.#") + "%)" +
         (selIsLim ? " | ยกเลิกถ้าไม่ติดใน " + str.tostring(selExp) + " แท่ง" : ""), alert.freq_once_per_bar_close)

if expiredNow
    alert("XAUUSD — limit ไม่ติดตามเวลา ยกเลิกออเดอร์ที่ค้างได้เลย", alert.freq_once_per_bar_close)
if filledNow
    alert("XAUUSD — ออเดอร์ติดแล้วที่ " + f_p(stEntry) + " เช็คว่า SL " + f_p(stSL) + " กับ TP " + f_p(stTP) + " ตั้งครบ", alert.freq_once_per_bar_close)
if tpNow
    alert("XAUUSD — ถึง TP " + f_p(stTP), alert.freq_once_per_bar_close)
if slNow
    alert("XAUUSD — โดน SL " + f_p(stSL), alert.freq_once_per_bar_close)

alertcondition(buySignal,  title="Buy",  message="XAUUSD BUY — ดูตัวเลขในป้ายบนกราฟ")
alertcondition(sellSignal, title="Sell", message="XAUUSD SELL — ดูตัวเลขในป้ายบนกราฟ")
alertcondition(sweptSSL,   title="ICT — กวาด SSL", message="XAUUSD — กวาด sell-side liquidity แล้ว จับตารอ MSS ขึ้น")
alertcondition(sweptBSL,   title="ICT — กวาด BSL", message="XAUUSD — กวาด buy-side liquidity แล้ว จับตารอ MSS ลง")

bgcolor(showRange and useRangeFlt and isRange ? rangeClr : na, title="Sideway")
bgcolor(kzShade and kzNow ? kzClr : na, title="Killzone")
```
