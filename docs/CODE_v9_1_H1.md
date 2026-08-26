# XAUUSD SMC Pro v9.1 (H1) — โค้ดเต็มสำหรับคัดลอก

เวอร์ชันแก้บั๊กแล้ว ดูรายละเอียดสิ่งที่แก้ใน [REVIEW_v9.md](REVIEW_v9.md)

กดปุ่มคัดลอก (ไอคอนมุมขวาบนของกล่องโค้ด) แล้ววางใน Pine Editor ได้เลย
บรรทัดแรกต้องเป็น `//@version=5`

```pine
//@version=5
// ══════════════════════════════════════════════════════════════════════════
// XAUUSD SMC Pro v9.1 — เวอร์ชันแก้บั๊ก + ปรับสำหรับ H1
// อ้างอิงรายการแก้ทั้งหมดใน docs/REVIEW_v9.md
// ══════════════════════════════════════════════════════════════════════════
indicator("XAUUSD SMC Pro v9.1 (H1)", overlay=true, max_labels_count=500, max_lines_count=500, max_boxes_count=500)

// ══════════════════════════════════════════
// INPUTS
// ══════════════════════════════════════════
grp_struct  = "Structure"
swingLen    = input.int(5, "Swing Length", minval=2, maxval=20, group=grp_struct)
swMaxAge    = input.int(60, "อายุ swing สูงสุดที่ใช้เป็น SL ได้ (แท่ง)", minval=5, maxval=300, group=grp_struct, tooltip="FIX: กัน swing เก่าค้างเป็นสิบวันแล้วเอามาคิด SL")

grp_mtf     = "Multi-Timeframe"
useMTF      = input.bool(true, "เปิด MTF Filter", group=grp_mtf)
mtfConfirm  = input.string("H4", "ยืนยันด้วย TF", options=["M15","M30","H1","H4"], group=grp_mtf, tooltip="ต้องสูงกว่า TF ของกราฟ ถ้าเท่ากับ/ต่ำกว่า ระบบจะใช้เทรนด์ของกราฟเองแทน (เล่น H1 → เลือก H4)")

grp_rt      = "Realtime Signal"
useRT       = input.bool(true, "เปิด Realtime (early warning)", group=grp_rt)
rtMinScore  = input.int(6, "Realtime Min Score", minval=4, maxval=8, group=grp_rt)
rtNeedAlign = input.bool(true, "Realtime ต้อง MTF aligned", group=grp_rt)
rtUseGap    = input.bool(true, "Realtime เว้นระยะเหมือน confirmed", group=grp_rt)

grp_fvg     = "FVG"
showFVG     = input.bool(true, "Show FVG (แสดงกล่อง)", group=grp_fvg)
fvgMitig    = input.bool(true, "ลบ FVG ที่ถูกเติมแล้ว", group=grp_fvg)
fvgLen      = input.int(12, "ความยาวกล่อง FVG (แท่ง)", minval=3, maxval=50, group=grp_fvg)
maxFvgKeep  = input.int(40, "เก็บ FVG ไว้สูงสุด (กล่อง)", minval=5, maxval=200, group=grp_fvg, tooltip="FIX: กัน array บวมจนเกิน max_boxes_count")
useFvgScore = input.bool(true, "ให้คะแนนถ้าราคาอยู่ในโซน FVG", group=grp_fvg, tooltip="เพิ่ม: เดิม FVG วาดอย่างเดียว ไม่ถูกใช้ตัดสินใจเลย")
fvgBullClr  = input.color(color.new(#00E676, 82), "Bullish FVG", group=grp_fvg)
fvgBearClr  = input.color(color.new(#FF1744, 82), "Bearish FVG", group=grp_fvg)

grp_signal  = "Signal"
emaFast     = input.int(21, "EMA Fast", minval=5,  group=grp_signal)
emaSlow     = input.int(50, "EMA Slow", minval=10, group=grp_signal)
rsiLen      = input.int(14, "RSI Length", minval=2, group=grp_signal)
rsiOB       = input.int(70, "RSI OB", minval=60, group=grp_signal)
rsiOS       = input.int(30, "RSI OS", maxval=40, group=grp_signal)
volFilter   = input.bool(true, "Volume Filter", group=grp_signal)
minScore    = input.int(5, "Min Score (แท่งปิด)", minval=2, maxval=8, group=grp_signal, tooltip="คะแนนเต็ม 8 — H1 แนะนำ 5 ขึ้นไป")
reqStruct   = input.bool(true, "บังคับให้โครงสร้าง SMC ตรงทาง (marketBias)", group=grp_signal, tooltip="FIX: เดิม score 4 ผ่านได้โดยไม่ใช้ BOS/CHoCH เลย = ไม่ใช่ SMC")
useMomentum = input.bool(true, "ยอมรับ Momentum candle", group=grp_signal)
signalGap   = input.int(15, "ระยะห่างสัญญาณขั้นต่ำ (แท่ง)", minval=1, maxval=50, group=grp_signal)
noChaseHi   = input.int(62, "ห้าม Long ถ้า RSI เกิน (กันไล่ยอด)", minval=50, maxval=70, group=grp_signal)
noChaseLo   = input.int(38, "ห้าม Short ถ้า RSI ต่ำกว่า (กันไล่ก้น)", minval=30, maxval=50, group=grp_signal)
strictMTF   = input.bool(true, "บังคับเข้าตามเทรนด์ MTF (Long=ขาขึ้น, Short=ขาลง)", group=grp_signal)

grp_sess    = "Session Filter (เพิ่มใหม่)"
useSession  = input.bool(true, "เทรดเฉพาะช่วงที่มีสภาพคล่อง", group=grp_sess, tooltip="ทองแกว่งจริงช่วง London + NY / ช่วง Asia มัก sideway และ spread กว้าง")
sessStr     = input.session("0800-2300", "ช่วงเวลาเทรด", group=grp_sess)
sessTz      = input.string("Europe/London", "โซนเวลาของช่วงเทรด", options=["Europe/London","America/New_York","Asia/Bangkok","UTC"], group=grp_sess)

grp_tpsl    = "TP / SL (0.01 lot)"
slAtrMult   = input.float(1.5, "SL = ATR ×", minval=0.3, step=0.1, group=grp_tpsl, tooltip="H1 แนะนำ 1.5-2.0 (ทอง H1 ATR ราว $5-15)")
maxSLusd    = input.float(20.0, "SL สูงสุด ($ ต่อ 1 oz)", minval=1.0, step=0.5, group=grp_tpsl, tooltip="ค่าเดิม 12 แคบเกินสำหรับ H1 ยุคทองผันผวน — โดน cap เกือบทุกไม้")
wideSLskip  = input.bool(true, "ถ้า ATR กว้างเกินเพดาน → ข้ามสัญญาณ (ไม่บีบ SL)", group=grp_tpsl, tooltip="FIX สำคัญ: เดิมจะ cap SL ให้แคบลงเฉยๆ = เข้าไม้ที่ SL แคบกว่าความผันผวนจริง โดนกวาดทิ้ง")
useSwingSL  = input.bool(true, "ใช้ swing ถ้าอยู่ในระยะ", group=grp_tpsl)
slBufAtr    = input.float(0.15, "บัฟเฟอร์เลย swing (× ATR)", minval=0.0, step=0.05, group=grp_tpsl)
rr1         = input.float(1.0, "TP1 = SL ×", minval=0.5, step=0.1, group=grp_tpsl)
rr2         = input.float(2.0, "TP2 = SL ×", minval=1.0, step=0.1, group=grp_tpsl)
atrLen      = input.int(14, "ATR Length", minval=1, group=grp_tpsl)
lotSize     = input.float(0.01, "Lot Size", minval=0.01, step=0.01, group=grp_tpsl)

grp_limit   = "Limit Entry"
useLimit    = input.bool(true, "โหมด Limit — รอราคาย่อก่อนเข้า", group=grp_limit)
pullbackAtr = input.float(0.4, "รอราคาย่อกลับ (× ATR)", minval=0.1, step=0.1, group=grp_limit)
limitExpiry = input.int(6, "ยกเลิก Limit ถ้าไม่ติดใน (แท่ง)", minval=1, maxval=50, group=grp_limit, tooltip="FIX: เดิมไม่มีการยกเลิก — ไม้ที่ไม่เคยติดก็ถูกนับ TP/SL ไปด้วย")
maxTradeKeep= input.int(60, "เก็บไม้ย้อนหลังบนกราฟสูงสุด", minval=10, maxval=120, group=grp_limit, tooltip="FIX: กัน array/line บวมจนเกิน max_lines_count")

grp_range   = "Sideway Filter (กรอง ADX)"
useRangeFlt = input.bool(true, "ไม่เทรดตอนตลาด Sideway", group=grp_range)
adxLen      = input.int(14, "ADX Length", minval=5, group=grp_range)
adxMin      = input.float(20, "ADX ขั้นต่ำ", minval=10, maxval=40, step=1, group=grp_range)
emaSlopeFlt = input.bool(true, "กรอง EMA แบนด้วย", group=grp_range)
emaSepMin   = input.float(0.15, "EMA21-50 ต้องห่างกัน ≥ (× ATR)", minval=0, step=0.05, group=grp_range)
showRange   = input.bool(true, "แสดงโซน Sideway บนกราฟ (พื้นเทา)", group=grp_range)

grp_style   = "Style"
buyColor    = input.color(color.new(#00E676, 0), "Buy", group=grp_style)
sellColor   = input.color(color.new(#FF1744, 0), "Sell", group=grp_style)
rtBuyColor  = input.color(color.new(#00E676, 45), "Buy (realtime)", group=grp_style)
rtSellColor = input.color(color.new(#FF1744, 45), "Sell (realtime)", group=grp_style)
structUpClr = input.color(color.new(#00E676, 0), "BOS/CHoCH ขาขึ้น", group=grp_style)
structDnClr = input.color(color.new(#FF1744, 0), "BOS/CHoCH ขาลง", group=grp_style)
rangeClr    = input.color(color.new(#787878, 88), "สีโซน Sideway", group=grp_style)
showStruct  = input.bool(true, "Show BOS/CHoCH", group=grp_style)
showStats   = input.bool(true, "แสดงสถิติผลลัพธ์ในตาราง", group=grp_style)

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

// เพิ่ม: session filter
inSession = not useSession or not na(time(timeframe.period, sessStr, sessTz))

// ══════════════════════════════════════════
// MTF BIAS
// FIX #1 (repaint): ค่าเดิม lookahead_off ไม่ใช่ "ค่าที่ปิดแล้ว" — มันคือค่าของแท่ง HTF
//   ที่กำลังก่อตัวอยู่ จึงเปลี่ยนไปมาระหว่างแท่ง = repaint
//   วิธีมาตรฐานที่ไม่ repaint คือ lookahead_on + ดึงค่า [1] (แท่ง HTF ที่ปิดแล้วเท่านั้น)
// FIX #2 (lower TF): บนกราฟ H1 การขอ M15/M30 เป็นการขอ TF ที่ต่ำกว่ากราฟ
//   ค่าที่ได้ไม่น่าเชื่อถือ → ตัดออกอัตโนมัติ และแสดง n/a ในตาราง
// ══════════════════════════════════════════
chartSec = timeframe.in_seconds(timeframe.period)

f_htfBias(_tf) =>
    [e21, e50] = request.security(syminfo.tickerid, _tf, [ta.ema(close, emaFast)[1], ta.ema(close, emaSlow)[1]], gaps=barmerge.gaps_off, lookahead=barmerge.lookahead_on)
    e21 > e50 ? 1 : e21 < e50 ? -1 : 0

raw15  = f_htfBias("15")
raw30  = f_htfBias("30")
raw60  = f_htfBias("60")
raw240 = f_htfBias("240")
rawD   = f_htfBias("D")

ok15  = timeframe.in_seconds("15")  >= chartSec
ok30  = timeframe.in_seconds("30")  >= chartSec
ok60  = timeframe.in_seconds("60")  >= chartSec
ok240 = timeframe.in_seconds("240") >= chartSec
okD   = timeframe.in_seconds("D")   >= chartSec

bias_m15 = ok15  ? raw15  : 0
bias_m30 = ok30  ? raw30  : 0
bias_h1  = ok60  ? raw60  : 0
bias_h4  = ok240 ? raw240 : 0
bias_d1  = okD   ? rawD   : 0

validCount = (ok15 ? 1 : 0) + (ok30 ? 1 : 0) + (ok60 ? 1 : 0) + (ok240 ? 1 : 0) + (okD ? 1 : 0)
mtfSum = bias_m15 + bias_m30 + bias_h1 + bias_h4 + bias_d1

// FIX #3: ถ้า TF ยืนยัน <= TF กราฟ (เช่น เล่น H1 แล้วเลือกยืนยันด้วย H1)
//   confirmBias จะซ้ำกับ trendUp/trendDn ทุกประการ = คะแนนซ้ำซ้อน + ฟิลเตอร์ไม่ทำงาน
//   → ใช้เทรนด์ของกราฟเองแทนแบบตรงไปตรงมา และเตือนในตาราง
localBias  = trendUp ? 1 : trendDn ? -1 : 0
confSec    = mtfConfirm == "M15" ? timeframe.in_seconds("15") : mtfConfirm == "M30" ? timeframe.in_seconds("30") : mtfConfirm == "H1" ? timeframe.in_seconds("60") : timeframe.in_seconds("240")
confRaw    = mtfConfirm == "M15" ? raw15 : mtfConfirm == "M30" ? raw30 : mtfConfirm == "H1" ? raw60 : raw240
confIsHTF  = confSec > chartSec
confirmBias = confIsHTF ? confRaw : localBias

// FIX #4: allBull/allBear เดิมต้องครบ 5 TF รวม M15/M30 ที่ต่ำกว่ากราฟ → แทบไม่มีทางเป็นจริง
//   ตอนนี้นับเฉพาะ TF ที่ใช้ได้จริง
allBull = validCount > 0 and (not ok15 or bias_m15 == 1) and (not ok30 or bias_m30 == 1) and (not ok60 or bias_h1 == 1) and (not ok240 or bias_h4 == 1) and (not okD or bias_d1 == 1)
allBear = validCount > 0 and (not ok15 or bias_m15 == -1) and (not ok30 or bias_m30 == -1) and (not ok60 or bias_h1 == -1) and (not ok240 or bias_h4 == -1) and (not okD or bias_d1 == -1)
strongUp = validCount > 0 and mtfSum > 0 and mtfSum >= validCount - 1
strongDn = validCount > 0 and mtfSum < 0 and mtfSum <= -(validCount - 1)

// ══════════════════════════════════════════
// SWING HIGH / LOW  (+ อายุของ swing)
// ══════════════════════════════════════════
swingHigh = ta.pivothigh(high, swingLen, swingLen)
swingLow  = ta.pivotlow(low, swingLen, swingLen)

var float swH1 = na
var float swL1 = na
var int   swHBar = na
var int   swLBar = na

if not na(swingHigh)
    swH1 := swingHigh
    swHBar := bar_index
if not na(swingLow)
    swL1 := swingLow
    swLBar := bar_index

swHFresh = not na(swHBar) and (bar_index - swHBar) <= swMaxAge
swLFresh = not na(swLBar) and (bar_index - swLBar) <= swMaxAge

// ══════════════════════════════════════════
// BOS & CHoCH
// ══════════════════════════════════════════
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
        label.new(bar_index, high, "BOS", style=label.style_label_down, color=color.new(structUpClr, 20), textcolor=color.white, size=size.tiny)
        line.new(bar_index - 8, swH1, bar_index, swH1, color=structUpClr, width=1)
    if bosDn
        label.new(bar_index, low, "BOS", style=label.style_label_up, color=color.new(structDnClr, 20), textcolor=color.white, size=size.tiny)
        line.new(bar_index - 8, swL1, bar_index, swL1, color=structDnClr, width=1)
    if chochUp
        label.new(bar_index, high, "CHoCH", style=label.style_label_down, color=color.new(structUpClr, 0), textcolor=color.white, size=size.small)
        line.new(bar_index - 8, swH1, bar_index, swH1, color=structUpClr, style=line.style_dashed, width=2)
    if chochDn
        label.new(bar_index, low, "CHoCH", style=label.style_label_up, color=color.new(structDnClr, 0), textcolor=color.white, size=size.small)
        line.new(bar_index - 8, swL1, bar_index, swL1, color=structDnClr, style=line.style_dashed, width=2)

// ══════════════════════════════════════════
// FVG
// FIX #5: เดิมสร้าง/ตรวจ FVG อยู่ใน `if showFVG` → ปิดการแสดงผล = logic หายไปด้วย
//   ตอนนี้แยกส่วนคำนวณออกจากส่วนวาด และเก็บระดับโซนไว้ใช้ให้คะแนนจริง
// ══════════════════════════════════════════
type FVG
    box   bx
    float top
    float bot
    bool  isBull

var FVG[] fvgs = array.new<FVG>()

bullFVG = low > high[2] and close[1] > open[1]
bearFVG = high < low[2] and close[1] < open[1]

if barstate.isconfirmed
    if bullFVG
        box nb = na
        if showFVG
            nb := box.new(bar_index - 2, low, bar_index + fvgLen, high[2], bgcolor=fvgBullClr, border_color=color.new(#00E676, 55), border_width=1)
        array.push(fvgs, FVG.new(nb, low, high[2], true))
    if bearFVG
        box nb2 = na
        if showFVG
            nb2 := box.new(bar_index - 2, low[2], bar_index + fvgLen, high, bgcolor=fvgBearClr, border_color=color.new(#FF1744, 55), border_width=1)
        array.push(fvgs, FVG.new(nb2, low[2], high, false))
    if fvgMitig and array.size(fvgs) > 0
        for i = array.size(fvgs) - 1 to 0
            f = array.get(fvgs, i)
            mit = f.isBull ? low <= f.bot : high >= f.top
            if mit
                if not na(f.bx)
                    box.delete(f.bx)
                array.remove(fvgs, i)
    while array.size(fvgs) > maxFvgKeep
        oldF = array.shift(fvgs)
        if not na(oldF.bx)
            box.delete(oldF.bx)

inBullFVG = false
inBearFVG = false
if array.size(fvgs) > 0
    for i = 0 to array.size(fvgs) - 1
        f = array.get(fvgs, i)
        touching = high >= f.bot and low <= f.top
        if touching and f.isBull
            inBullFVG := true
        if touching and not f.isBull
            inBearFVG := true

// ══════════════════════════════════════════
// ENTRY CONDITIONS
// ══════════════════════════════════════════
// FIX #6: เดิม `rsiVal < rsiOB and rsiVal < noChaseHi` — noChaseHi(62) < rsiOB(70) เสมอ
//   ทำให้ input RSI OB/OS เป็นค่าตายไม่มีผล ตอนนี้ผูกให้ทั้งสองค่ามีความหมาย
rsiOK_buy  = rsiVal > rsiOS and rsiVal < math.min(rsiOB, noChaseHi)
rsiOK_sell = rsiVal < rsiOB and rsiVal > math.max(rsiOS, noChaseLo)

// FIX #7: บางฟีด XAUUSD ไม่มี volume (na) → เงื่อนไขเดิมเป็น false ตลอด = เสียคะแนนฟรี 1 แต้ม
volOK = not volFilter or na(volume) or na(avgVol) or volume > avgVol * 0.7

bullEngulf = close[1] < open[1] and close > open and close > open[1] and open <= close[1]
bearEngulf = close[1] > open[1] and close < open and close < open[1] and open >= close[1]
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

fvgBuyOK  = useFvgScore and inBullFVG
fvgSellOK = useFvgScore and inBearFVG

buyScore  = (trendUp ? 1 : 0) + (rsiOK_buy ? 1 : 0) + (bullTrigger ? 1 : 0) + (volOK ? 1 : 0) + (marketBias == 1 ? 1 : 0) + (chochUp ? 1 : 0) + (bosUp ? 1 : 0) + (fvgBuyOK ? 1 : 0)
sellScore = (trendDn ? 1 : 0) + (rsiOK_sell ? 1 : 0) + (bearTrigger ? 1 : 0) + (volOK ? 1 : 0) + (marketBias == -1 ? 1 : 0) + (chochDn ? 1 : 0) + (bosDn ? 1 : 0) + (fvgSellOK ? 1 : 0)

// ══════════════════════════════════════════
// TP / SL
// ══════════════════════════════════════════
slTooWide  = atrVal * slAtrMult > maxSLusd
slDistBase = math.min(atrVal * slAtrMult, maxSLusd)

buyEntry  = useLimit ? close - pullbackAtr * atrVal : close
sellEntry = useLimit ? close + pullbackAtr * atrVal : close

buySwingDist = swLFresh and not na(swL1) and swL1 < buyEntry ? buyEntry - swL1 + atrVal * slBufAtr : na
buyUseSwing  = useSwingSL and not na(buySwingDist) and buySwingDist <= maxSLusd
buySLdist    = buyUseSwing ? buySwingDist : slDistBase
finalBuySL   = buyEntry - buySLdist
finalBuyTP1  = buyEntry + buySLdist * rr1
finalBuyTP2  = buyEntry + buySLdist * rr2

sellSwingDist = swHFresh and not na(swH1) and swH1 > sellEntry ? swH1 - sellEntry + atrVal * slBufAtr : na
sellUseSwing  = useSwingSL and not na(sellSwingDist) and sellSwingDist <= maxSLusd
sellSLdist    = sellUseSwing ? sellSwingDist : slDistBase
finalSellSL   = sellEntry + sellSLdist
finalSellTP1  = sellEntry - sellSLdist * rr1
finalSellTP2  = sellEntry - sellSLdist * rr2

// FIX #8: เดิมเมื่อ ATR กว้างเกินเพดาน ระบบจะ "บีบ SL ให้แคบ" แล้วเข้าไม้อยู่ดี
//   = SL แคบกว่าความผันผวนจริงของ H1 โดนกวาดทิ้งเป็นเรื่องปกติ
//   ตอนนี้เลือกได้ว่าจะข้ามสัญญาณนั้นไปเลย
buySLcapped  = not buyUseSwing and slTooWide
sellSLcapped = not sellUseSwing and slTooWide
buyQualOK  = not na(atrVal) and buySLdist > 0 and not (wideSLskip and buySLcapped)
sellQualOK = not na(atrVal) and sellSLdist > 0 and not (wideSLskip and sellSLcapped)

buyTP1Pts  = math.round(buySLdist * rr1 / 0.01)
buyTP2Pts  = math.round(buySLdist * rr2 / 0.01)
buySLPts   = math.round(buySLdist / 0.01)
buyTP1usd  = math.round(buySLdist * rr1 * lotSize * 100, 2)
buyTP2usd  = math.round(buySLdist * rr2 * lotSize * 100, 2)
buySLusd   = math.round(buySLdist * lotSize * 100, 2)

sellTP1Pts = math.round(sellSLdist * rr1 / 0.01)
sellTP2Pts = math.round(sellSLdist * rr2 / 0.01)
sellSLPts  = math.round(sellSLdist / 0.01)
sellTP1usd = math.round(sellSLdist * rr1 * lotSize * 100, 2)
sellTP2usd = math.round(sellSLdist * rr2 * lotSize * 100, 2)
sellSLusd  = math.round(sellSLdist * lotSize * 100, 2)

// ══════════════════════════════════════════
// SIGNAL LOGIC
// ══════════════════════════════════════════
structBuyOK  = not reqStruct or marketBias == 1
structSellOK = not reqStruct or marketBias == -1

baseBuy  = bullTrigger and mtfBuyOK and buyQualOK and trending and inSession and structBuyOK
baseSell = bearTrigger and mtfSellOK and sellQualOK and trending and inSession and structSellOK

confirmedBuy  = barstate.isconfirmed and buyScore >= minScore and baseBuy
confirmedSell = barstate.isconfirmed and sellScore >= minScore and baseSell

rtAlignBuy  = not rtNeedAlign or allBull
rtAlignSell = not rtNeedAlign or allBear

var int lastSignalBar = na
canSignal   = na(lastSignalBar) or (bar_index - lastSignalBar) >= signalGap
rtCanSignal = not rtUseGap or canSignal

realtimeBuy  = useRT and not barstate.isconfirmed and buyScore >= rtMinScore and baseBuy and rtAlignBuy and rtCanSignal
realtimeSell = useRT and not barstate.isconfirmed and sellScore >= rtMinScore and baseSell and rtAlignSell and rtCanSignal

buySignal  = confirmedBuy and canSignal
sellSignal = confirmedSell and canSignal
if buySignal or sellSignal
    lastSignalBar := bar_index

// ══════════════════════════════════════════
// DRAW — REALTIME
// ══════════════════════════════════════════
var label rtLabel = na
var line  rtL1 = na
var line  rtL2 = na
var line  rtL3 = na

if useRT
    label.delete(rtLabel)
    line.delete(rtL1)
    line.delete(rtL2)
    line.delete(rtL3)
    if realtimeBuy and not confirmedBuy
        rtLabel := label.new(bar_index, low, "BUY (RT) " + str.tostring(close, "#.##") + "\nTP1 " + str.tostring(finalBuyTP1, "#.##") + "  TP2 " + str.tostring(finalBuyTP2, "#.##") + "\nSL " + str.tostring(finalBuySL, "#.##") + "\nรอยืนยันจนแท่งปิด", style=label.style_label_up, color=rtBuyColor, textcolor=color.white, size=size.normal)
        rtL1 := line.new(bar_index, finalBuyTP2, bar_index + 10, finalBuyTP2, color=color.new(#00E676, 40), style=line.style_dotted, width=1)
        rtL2 := line.new(bar_index, finalBuyTP1, bar_index + 10, finalBuyTP1, color=color.new(#00E676, 55), style=line.style_dotted, width=1)
        rtL3 := line.new(bar_index, finalBuySL, bar_index + 10, finalBuySL, color=color.new(#FF1744, 40), style=line.style_dotted, width=1)
    if realtimeSell and not confirmedSell
        rtLabel := label.new(bar_index, high, "SELL (RT) " + str.tostring(close, "#.##") + "\nTP1 " + str.tostring(finalSellTP1, "#.##") + "  TP2 " + str.tostring(finalSellTP2, "#.##") + "\nSL " + str.tostring(finalSellSL, "#.##") + "\nรอยืนยันจนแท่งปิด", style=label.style_label_down, color=rtSellColor, textcolor=color.white, size=size.normal)
        rtL1 := line.new(bar_index, finalSellTP2, bar_index + 10, finalSellTP2, color=color.new(#FF1744, 40), style=line.style_dotted, width=1)
        rtL2 := line.new(bar_index, finalSellTP1, bar_index + 10, finalSellTP1, color=color.new(#FF1744, 55), style=line.style_dotted, width=1)
        rtL3 := line.new(bar_index, finalSellSL, bar_index + 10, finalSellSL, color=color.new(#00E676, 40), style=line.style_dotted, width=1)

// ══════════════════════════════════════════
// DRAW — CONFIRMED + ติดตามผลของแต่ละไม้
// FIX #9: เดิมโหมด Limit วาดเส้นแล้วนับ TP/SL ทันที ทั้งที่ราคาอาจไม่เคยย่อลงมาติดเลย
//   ตอนนี้มีสถานะ pending → ติด (filled) → จบ (TP2/SL) → หรือหมดอายุ (cancel)
// FIX #10: array ไม้เดิมโตไม่จำกัด + วน loop ทุกไม้ทุกแท่ง และเส้นเกิน 500 เส้น
//   จะถูก TradingView ลบทิ้งเงียบๆ ทำให้ set_x2 ไม่มีผล → จำกัดจำนวนที่เก็บ
// ══════════════════════════════════════════
type TradeViz
    line  lnTP2
    line  lnTP1
    line  lnSL
    line  lnEntry
    float tp2
    float tp1
    float sl
    float entry
    bool  isBuy
    int   barCreated
    int   barFilled
    bool  pending
    bool  active
    bool  tp1Hit

var TradeViz[] trades = array.new<TradeViz>()
var int cntTP1    = 0
var int cntTP2    = 0
var int cntSL     = 0
var int cntCancel = 0

if buySignal
    hdr = useLimit ? "BUY LIMIT\nเข้า " + str.tostring(buyEntry, "#.##") : "BUY " + str.tostring(close, "#.##")
    lblText = hdr +
         "\nTP1 " + str.tostring(finalBuyTP1, "#.##") + "  (+" + str.tostring(buyTP1Pts) + " / $" + str.tostring(buyTP1usd) + ")" +
         "\nTP2 " + str.tostring(finalBuyTP2, "#.##") + "  (+" + str.tostring(buyTP2Pts) + " / $" + str.tostring(buyTP2usd) + ")" +
         "\nSL  " + str.tostring(finalBuySL, "#.##") + "  (-" + str.tostring(buySLPts) + " / -$" + str.tostring(buySLusd) + ")" +
         (useLimit ? "\nยกเลิกถ้าไม่ติดใน " + str.tostring(limitExpiry) + " แท่ง" : "") +
         "\nโดน TP1 → เลื่อน SL มา BE"
    label.new(bar_index, low, lblText, style=label.style_label_up, color=buyColor, textcolor=color.white, size=size.normal)
    bLnTP2   = line.new(bar_index, finalBuyTP2, bar_index, finalBuyTP2, color=color.new(#00E676, 20), style=line.style_dashed, width=2)
    bLnTP1   = line.new(bar_index, finalBuyTP1, bar_index, finalBuyTP1, color=color.new(#00E676, 45), style=line.style_dashed, width=1)
    bLnSL    = line.new(bar_index, finalBuySL,  bar_index, finalBuySL,  color=color.new(#FF1744, 20), style=line.style_dashed, width=2)
    bLnEntry = line.new(bar_index, buyEntry,    bar_index, buyEntry,    color=color.new(#FFEB3B, 20), style=line.style_dotted, width=1)
    array.push(trades, TradeViz.new(bLnTP2, bLnTP1, bLnSL, bLnEntry, finalBuyTP2, finalBuyTP1, finalBuySL, buyEntry, true, bar_index, useLimit ? int(na) : bar_index, useLimit, true, false))

if sellSignal
    hdr = useLimit ? "SELL LIMIT\nเข้า " + str.tostring(sellEntry, "#.##") : "SELL " + str.tostring(close, "#.##")
    lblText = hdr +
         "\nTP1 " + str.tostring(finalSellTP1, "#.##") + "  (+" + str.tostring(sellTP1Pts) + " / $" + str.tostring(sellTP1usd) + ")" +
         "\nTP2 " + str.tostring(finalSellTP2, "#.##") + "  (+" + str.tostring(sellTP2Pts) + " / $" + str.tostring(sellTP2usd) + ")" +
         "\nSL  " + str.tostring(finalSellSL, "#.##") + "  (-" + str.tostring(sellSLPts) + " / -$" + str.tostring(sellSLusd) + ")" +
         (useLimit ? "\nยกเลิกถ้าไม่ติดใน " + str.tostring(limitExpiry) + " แท่ง" : "") +
         "\nโดน TP1 → เลื่อน SL มา BE"
    label.new(bar_index, high, lblText, style=label.style_label_down, color=sellColor, textcolor=color.white, size=size.normal)
    sLnTP2   = line.new(bar_index, finalSellTP2, bar_index, finalSellTP2, color=color.new(#FF1744, 20), style=line.style_dashed, width=2)
    sLnTP1   = line.new(bar_index, finalSellTP1, bar_index, finalSellTP1, color=color.new(#FF1744, 45), style=line.style_dashed, width=1)
    sLnSL    = line.new(bar_index, finalSellSL,  bar_index, finalSellSL,  color=color.new(#00E676, 20), style=line.style_dashed, width=2)
    sLnEntry = line.new(bar_index, sellEntry,    bar_index, sellEntry,    color=color.new(#FFEB3B, 20), style=line.style_dotted, width=1)
    array.push(trades, TradeViz.new(sLnTP2, sLnTP1, sLnSL, sLnEntry, finalSellTP2, finalSellTP1, finalSellSL, sellEntry, false, bar_index, useLimit ? int(na) : bar_index, useLimit, true, false))

// จำกัดจำนวนไม้ที่เก็บ — ลบไม้เก่าสุดพร้อมเส้นของมัน
while array.size(trades) > maxTradeKeep
    oldT = array.shift(trades)
    line.delete(oldT.lnTP2)
    line.delete(oldT.lnTP1)
    line.delete(oldT.lnSL)
    line.delete(oldT.lnEntry)

pendingCnt = 0
if array.size(trades) > 0
    for i = array.size(trades) - 1 to 0
        tr = array.get(trades, i)
        if tr.active
            line.set_x2(tr.lnTP2, bar_index)
            line.set_x2(tr.lnTP1, bar_index)
            line.set_x2(tr.lnSL, bar_index)
            line.set_x2(tr.lnEntry, bar_index)
            if tr.pending
                filled = tr.isBuy ? low <= tr.entry : high >= tr.entry
                if filled
                    tr.pending := false
                    tr.barFilled := bar_index
                else if bar_index - tr.barCreated >= limitExpiry
                    // Limit ไม่ติดในเวลาที่กำหนด → ยกเลิก
                    tr.active := false
                    cntCancel := cntCancel + 1
                    line.delete(tr.lnTP2)
                    line.delete(tr.lnTP1)
                    line.delete(tr.lnSL)
                else
                    pendingCnt := pendingCnt + 1
            if tr.active and not tr.pending and bar_index > tr.barFilled
                hitSL  = tr.isBuy ? low <= tr.sl : high >= tr.sl
                hitTP2 = tr.isBuy ? high >= tr.tp2 : low <= tr.tp2
                hitTP1 = tr.isBuy ? high >= tr.tp1 : low <= tr.tp1
                if not tr.tp1Hit and hitTP1 and not hitSL
                    tr.tp1Hit := true
                    cntTP1 := cntTP1 + 1
                if hitSL or hitTP2
                    tr.active := false
                    // แท่งเดียวโดนทั้งคู่ = ไม่รู้ลำดับจริง → นับเป็น SL (มองแบบระวังไว้ก่อน)
                    if hitSL
                        cntSL := cntSL + 1
                    else
                        cntTP2 := cntTP2 + 1

// ══════════════════════════════════════════
// INFO TABLE
// ══════════════════════════════════════════
f_biasCell(_b, _ok) => not _ok ? "n/a" : _b == 1 ? "UP" : _b == -1 ? "DOWN" : "-"
f_biasClr(_b, _ok)  => not _ok ? color.new(color.gray, 50) : _b == 1 ? buyColor : _b == -1 ? sellColor : color.gray

var table t = table.new(position.top_right, 2, 17, bgcolor=color.new(#1a1a2e, 10), border_width=1, border_color=color.new(color.gray, 70))
if barstate.islast
    table.cell(t, 0, 0, "TIMEFRAME", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#0f3460, 20))
    table.cell(t, 1, 0, "BIAS", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#0f3460, 20))
    table.cell(t, 0, 1, "M15", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 1, f_biasCell(bias_m15, ok15), text_color=f_biasClr(bias_m15, ok15), text_size=size.tiny)
    table.cell(t, 0, 2, "M30", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 2, f_biasCell(bias_m30, ok30), text_color=f_biasClr(bias_m30, ok30), text_size=size.tiny)
    table.cell(t, 0, 3, "H1", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 3, f_biasCell(bias_h1, ok60), text_color=f_biasClr(bias_h1, ok60), text_size=size.tiny)
    table.cell(t, 0, 4, "H4", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 4, f_biasCell(bias_h4, ok240), text_color=f_biasClr(bias_h4, ok240), text_size=size.tiny)
    table.cell(t, 0, 5, "D1", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 5, f_biasCell(bias_d1, okD), text_color=f_biasClr(bias_d1, okD), text_size=size.tiny)
    alignTxt = strongUp ? "STRONG UP" : strongDn ? "STRONG DOWN" : "MIXED"
    table.cell(t, 0, 6, "ALIGN", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 6, alignTxt, text_color=strongUp ? buyColor : strongDn ? sellColor : color.gray, text_size=size.tiny)
    table.cell(t, 0, 7, "CONFIRM TF", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 7, confIsHTF ? mtfConfirm : mtfConfirm + " → ใช้กราฟ", text_color=confIsHTF ? color.white : color.orange, text_size=size.tiny)
    table.cell(t, 0, 8, "RSI", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 8, str.tostring(rsiVal, "#.#"), text_color=color.white, text_size=size.tiny)
    table.cell(t, 0, 9, "ATR", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 9, str.tostring(atrVal, "#.##"), text_color=color.white, text_size=size.tiny)
    table.cell(t, 0, 10, "ADX", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 10, str.tostring(adxVal, "#.#"), text_color=adxVal >= adxMin ? color.white : color.orange, text_size=size.tiny)
    table.cell(t, 0, 11, "SL ที่จะใช้", text_color=color.gray, text_size=size.tiny)
    slTxt = slTooWide ? (wideSLskip ? "กว้างเกิน → ข้าม" : "โดน cap " + str.tostring(maxSLusd, "#.#")) : "$" + str.tostring(slDistBase, "#.##")
    table.cell(t, 1, 11, slTxt, text_color=slTooWide ? color.orange : color.white, text_size=size.tiny)
    table.cell(t, 0, 12, "SESSION", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 12, not useSession ? "OFF" : inSession ? "เปิด" : "ปิด", text_color=inSession ? buyColor : color.gray, text_size=size.tiny)
    table.cell(t, 0, 13, "REGIME", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 13, trending ? "TREND" : "SIDEWAY", text_color=trending ? color.white : color.orange, text_size=size.tiny)
    rtStatus = not useRT ? "OFF" : realtimeBuy ? "BUY?" : realtimeSell ? "SELL?" : "รอ"
    table.cell(t, 0, 14, "REALTIME", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 14, rtStatus, text_color=realtimeBuy ? buyColor : realtimeSell ? sellColor : color.gray, text_size=size.tiny)
    if showStats
        closedN = cntTP2 + cntSL
        wrTxt = closedN > 0 ? str.tostring(math.round(cntTP2 * 100.0 / closedN, 1)) + "%" : "-"
        table.cell(t, 0, 15, "TP2 / SL", text_color=color.gray, text_size=size.tiny)
        table.cell(t, 1, 15, str.tostring(cntTP2) + " / " + str.tostring(cntSL) + "  (" + wrTxt + ")", text_color=color.white, text_size=size.tiny)
        table.cell(t, 0, 16, "TP1 / ยกเลิก / รอ", text_color=color.gray, text_size=size.tiny)
        table.cell(t, 1, 16, str.tostring(cntTP1) + " / " + str.tostring(cntCancel) + " / " + str.tostring(pendingCnt), text_color=color.white, text_size=size.tiny)

// ══════════════════════════════════════════
// ALERTS
// ══════════════════════════════════════════
alertcondition(buySignal,  title="Buy (ยืนยัน)",  message="XAUUSD BUY — วาง Buy Limit ตามราคาในป้าย (ดู TP/SL)")
alertcondition(sellSignal, title="Sell (ยืนยัน)", message="XAUUSD SELL — วาง Sell Limit ตามราคาในป้าย (ดู TP/SL)")
alertcondition(realtimeBuy,  title="Buy (realtime)",  message="XAUUSD BUY กำลังก่อตัว — ยังไม่ยืนยัน")
alertcondition(realtimeSell, title="Sell (realtime)", message="XAUUSD SELL กำลังก่อตัว — ยังไม่ยืนยัน")

if buySignal
    alert("XAUUSD BUY (ยืนยัน) @ " + str.tostring(buyEntry, "#.##") + " | " + (useLimit ? "วาง Buy Limit (หมดอายุ " + str.tostring(limitExpiry) + " แท่ง)" : "เข้าตลาด") + " | TP1 " + str.tostring(finalBuyTP1, "#.##") + " TP2 " + str.tostring(finalBuyTP2, "#.##") + " SL " + str.tostring(finalBuySL, "#.##"), alert.freq_once_per_bar_close)
if sellSignal
    alert("XAUUSD SELL (ยืนยัน) @ " + str.tostring(sellEntry, "#.##") + " | " + (useLimit ? "วาง Sell Limit (หมดอายุ " + str.tostring(limitExpiry) + " แท่ง)" : "เข้าตลาด") + " | TP1 " + str.tostring(finalSellTP1, "#.##") + " TP2 " + str.tostring(finalSellTP2, "#.##") + " SL " + str.tostring(finalSellSL, "#.##"), alert.freq_once_per_bar_close)

// ══════════════════════════════════════════
// BACKGROUND
// ══════════════════════════════════════════
bgcolor(showRange and useRangeFlt and isRange ? rangeClr : na, title="Sideway zone")
bgcolor(buySignal ? color.new(buyColor, 92) : na, title="Buy bar")
bgcolor(sellSignal ? color.new(sellColor, 92) : na, title="Sell bar")
```
