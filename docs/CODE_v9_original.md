# XAUUSD SMC Pro v9 (ต้นฉบับ) — โค้ดเต็มสำหรับคัดลอก

เวอร์ชันเดิมตามที่ส่งมา เก็บไว้เทียบเท่านั้น — ยังมีบั๊กตามที่ระบุใน [REVIEW_v9.md](REVIEW_v9.md)

กดปุ่มคัดลอก (ไอคอนมุมขวาบนของกล่องโค้ด) แล้ววางใน Pine Editor ได้เลย
บรรทัดแรกต้องเป็น `//@version=5`

```pine
//@version=5
indicator("XAUUSD SMC Pro v9", overlay=true, max_labels_count=500, max_lines_count=500, max_boxes_count=500)

// ══════════════════════════════════════════
// INPUTS
// ══════════════════════════════════════════
grp_struct  = "Structure"
swingLen    = input.int(5, "Swing Length", minval=2, maxval=20, group=grp_struct)

grp_mtf     = "Multi-Timeframe"
useMTF      = input.bool(true, "เปิด MTF Filter", group=grp_mtf)
mtfConfirm  = input.string("H1", "ยืนยันด้วย TF", options=["M15","M30","H1","H4"], group=grp_mtf)

grp_rt      = "Realtime Signal"
useRT       = input.bool(true, "เปิด Realtime (early warning)", group=grp_rt)
rtMinScore  = input.int(5, "Realtime Min Score", minval=4, maxval=7, group=grp_rt)
rtNeedAlign = input.bool(true, "Realtime ต้อง MTF aligned", group=grp_rt)
rtUseGap    = input.bool(true, "Realtime เว้นระยะเหมือน confirmed", group=grp_rt)

grp_fvg     = "FVG"
showFVG     = input.bool(true, "Show FVG", group=grp_fvg)
fvgMitig    = input.bool(true, "ลบ FVG ที่ถูกเติมแล้ว", group=grp_fvg)
fvgLen      = input.int(12, "ความยาวกล่อง FVG (แท่ง)", minval=3, maxval=50, group=grp_fvg, tooltip="กล่องยาวคงที่ ไม่ลากยาวข้ามจอ")
fvgBullClr  = input.color(color.new(#00E676, 82), "Bullish FVG", group=grp_fvg)
fvgBearClr  = input.color(color.new(#FF1744, 82), "Bearish FVG", group=grp_fvg)

grp_signal  = "Signal"
emaFast     = input.int(21, "EMA Fast", minval=5,  group=grp_signal)
emaSlow     = input.int(50, "EMA Slow", minval=10, group=grp_signal)
rsiLen      = input.int(14, "RSI Length", minval=2, group=grp_signal)
rsiOB       = input.int(70, "RSI OB", minval=60, group=grp_signal)
rsiOS       = input.int(30, "RSI OS", maxval=40, group=grp_signal)
volFilter   = input.bool(true, "Volume Filter", group=grp_signal)
minScore    = input.int(4, "Min Score (แท่งปิด)", minval=2, maxval=5, group=grp_signal)
useMomentum = input.bool(true, "ยอมรับ Momentum candle", group=grp_signal)
signalGap   = input.int(15, "ระยะห่างสัญญาณขั้นต่ำ (แท่ง)", minval=1, maxval=50, group=grp_signal)
// ── ค่าที่ผ่านการ backtest แล้วว่าดีขึ้น (sync จากตัว strategy) ──
noChaseHi   = input.int(62, "ห้าม Long ถ้า RSI เกิน (กันไล่ยอด)", minval=50, maxval=70, group=grp_signal)
noChaseLo   = input.int(38, "ห้าม Short ถ้า RSI ต่ำกว่า (กันไล่ก้น)", minval=30, maxval=50, group=grp_signal)
strictMTF   = input.bool(true, "บังคับเข้าตามเทรนด์ MTF (Long=ขาขึ้น, Short=ขาลง)", group=grp_signal)

grp_tpsl    = "TP / SL (0.01 lot)"
slAtrMult   = input.float(1.2, "SL = ATR ×", minval=0.3, step=0.1, group=grp_tpsl, tooltip="ตัวคูณ ATR สำหรับ SL (M15-H1 แนะนำ 1.2-1.5)")
maxSLusd    = input.float(12.0, "SL สูงสุด ($)", minval=1.0, step=0.5, group=grp_tpsl, tooltip="เพดาน SL — ตั้ง 12 ให้พอดี H1 / ถ้าเล่น M15 ล้วนลดเหลือ 6-8 ได้")
useSwingSL  = input.bool(true, "ใช้ swing ถ้าอยู่ในระยะ", group=grp_tpsl, tooltip="ใช้ swing เป็น SL เฉพาะถ้าไม่ไกลเกินเพดาน")
rr1         = input.float(1.0, "TP1 = SL ×", minval=0.5, step=0.1, group=grp_tpsl, tooltip="TP แรก ปิดครึ่งไม้")
rr2         = input.float(1.8, "TP2 = SL ×", minval=1.0, step=0.1, group=grp_tpsl, tooltip="TP สอง ปิดที่เหลือ")
atrLen      = input.int(14, "ATR Length", minval=1, group=grp_tpsl)
lotSize     = input.float(0.01, "Lot Size", minval=0.01, step=0.01, group=grp_tpsl)

grp_limit   = "Limit Entry (แนะนำสำหรับ H1)"
useLimit    = input.bool(true, "โหมด Limit — รอราคาย่อก่อนเข้า (ผ่าน backtest H1/H4)", group=grp_limit, tooltip="เปิดสำหรับ H1/H4 (ดีขึ้นมาก) / ปิดถ้าเล่น M15")
pullbackAtr = input.float(0.4, "รอราคาย่อกลับ (× ATR)", minval=0.1, step=0.1, group=grp_limit, tooltip="Long วาง Buy Limit ต่ำกว่าราคาปิด X×ATR / Short วาง Sell Limit สูงกว่า")

grp_range   = "Sideway Filter (กรอง ADX)"
useRangeFlt = input.bool(true, "ไม่เทรดตอนตลาด Sideway (ผ่าน backtest — DD ลดครึ่ง)", group=grp_range, tooltip="กรองด้วย ADX ต่ำ + EMA แบน = ตลาดออกข้าง")
adxLen      = input.int(14, "ADX Length", minval=5, group=grp_range)
adxMin      = input.float(20, "ADX ขั้นต่ำ (ต่ำกว่านี้ = sideway ไม่เทรด)", minval=10, maxval=40, step=1, group=grp_range)
emaSlopeFlt = input.bool(true, "กรอง EMA แบนด้วย", group=grp_range)
emaSepMin   = input.float(0.15, "EMA21-50 ต้องห่างกัน ≥ (× ATR)", minval=0, step=0.05, group=grp_range)
showRange   = input.bool(true, "แสดงโซน Sideway บนกราฟ (พื้นเทา)", group=grp_range)

grp_style   = "Style"
buyColor    = input.color(color.new(#00E676, 0), "Buy", group=grp_style)
sellColor   = input.color(color.new(#FF1744, 0), "Sell", group=grp_style)
rtBuyColor  = input.color(color.new(#00E676, 45), "Buy (realtime)", group=grp_style)
rtSellColor = input.color(color.new(#FF1744, 45), "Sell (realtime)", group=grp_style)
structUpClr = input.color(color.new(#00E676, 0), "BOS/CHoCH ขาขึ้น (เขียว)", group=grp_style)
structDnClr = input.color(color.new(#FF1744, 0), "BOS/CHoCH ขาลง (แดง)", group=grp_style)
rangeClr    = input.color(color.new(#787878, 88), "สีโซน Sideway", group=grp_style)
showStruct  = input.bool(true, "Show BOS/CHoCH", group=grp_style)

// ══════════════════════════════════════════
// CORE
// ══════════════════════════════════════════
ema21  = ta.ema(close, emaFast)
ema50  = ta.ema(close, emaSlow)
rsiVal = ta.rsi(close, rsiLen)
atrVal = ta.atr(atrLen)
avgVol = ta.sma(volume, 20)

// ══════════════════════════════════════════
// MTF BIAS  — FIX #1: no repaint (lookahead_off + gaps_off, ค่าที่ปิดแล้ว)
// ══════════════════════════════════════════
f_htfBias(_tf) =>
    [e21, e50] = request.security(syminfo.tickerid, _tf, [ta.ema(close, emaFast), ta.ema(close, emaSlow)], lookahead=barmerge.lookahead_off)
    e21 > e50 ? 1 : e21 < e50 ? -1 : 0

bias_m15 = f_htfBias("15")
bias_m30 = f_htfBias("30")
bias_h1  = f_htfBias("60")
bias_h4  = f_htfBias("240")
bias_d1  = f_htfBias("D")

// FIX #2: เขียน branch ให้ครบ อ่านง่าย
confirmBias = mtfConfirm == "M15" ? bias_m15 : mtfConfirm == "M30" ? bias_m30 : mtfConfirm == "H1" ? bias_h1 : mtfConfirm == "H4" ? bias_h4 : 0
mtfSum = bias_m15 + bias_m30 + bias_h1 + bias_h4 + bias_d1
// FIX #5: รวม D1 เข้า allBull/allBear ให้สอดคล้องกับ rtNeedAlign
allBull = bias_m15 == 1 and bias_m30 == 1 and bias_h1 == 1 and bias_h4 == 1 and bias_d1 == 1
allBear = bias_m15 == -1 and bias_m30 == -1 and bias_h1 == -1 and bias_h4 == -1 and bias_d1 == -1

// ══════════════════════════════════════════
// SWING HIGH / LOW
// ══════════════════════════════════════════
swingHigh = ta.pivothigh(high, swingLen, swingLen)
swingLow  = ta.pivotlow(low, swingLen, swingLen)

var float swH1 = na, var float swL1 = na
if not na(swingHigh)
    swH1 := swingHigh
if not na(swingLow)
    swL1 := swingLow

// ══════════════════════════════════════════
// BOS & CHoCH  — FIX #3: reset ฝั่งตรงข้ามเมื่อ bias เปลี่ยน (symmetric)
// ══════════════════════════════════════════
var int  marketBias = 0
var bool brokeH = false, var bool brokeL = false
var bool bosUp = false, var bool bosDn = false
var bool chochUp = false, var bool chochDn = false

bosUp := false, bosDn := false, chochUp := false, chochDn := false

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
        brokeL := false  // bias พลิกขึ้น → เปิดโอกาสให้จับ break ฝั่งล่างใหม่
    brokeH := true

if not na(swL1) and close < swL1 and not brokeL
    if marketBias == -1
        bosDn := true
    else
        chochDn := true
        marketBias := -1
        brokeH := false  // bias พลิกลง → เปิดโอกาสให้จับ break ฝั่งบนใหม่
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
// FVG + Mitigation
// ══════════════════════════════════════════
var box[] bullBoxes = array.new_box()
var box[] bearBoxes = array.new_box()

bullFVG = low > high[2] and close[1] > open[1]
bearFVG = high < low[2] and close[1] < open[1]

if showFVG and barstate.isconfirmed
    if bullFVG
        b = box.new(bar_index - 2, low, bar_index + fvgLen, high[2], bgcolor=fvgBullClr, border_color=color.new(#00E676, 55), border_width=1)
        array.push(bullBoxes, b)
    if bearFVG
        b = box.new(bar_index - 2, low[2], bar_index + fvgLen, high, bgcolor=fvgBearClr, border_color=color.new(#FF1744, 55), border_width=1)
        array.push(bearBoxes, b)
    // เช็ค mitigation อย่างเดียว — ไม่ยืดกล่อง (กล่องยาวคงที่ตอนสร้าง)
    if array.size(bullBoxes) > 0
        for i = array.size(bullBoxes) - 1 to 0
            bx = array.get(bullBoxes, i)
            if fvgMitig and low <= box.get_bottom(bx)
                box.delete(bx)
                array.remove(bullBoxes, i)
    if array.size(bearBoxes) > 0
        for i = array.size(bearBoxes) - 1 to 0
            bx = array.get(bearBoxes, i)
            if fvgMitig and high >= box.get_top(bx)
                box.delete(bx)
                array.remove(bearBoxes, i)

// ══════════════════════════════════════════
// ENTRY CONDITIONS
// ══════════════════════════════════════════
trendUp = ema21 > ema50
trendDn = ema21 < ema50
// กันไล่ยอด/ก้น — Long ห้ามซื้อตอน RSI สูงเกิน, Short ห้ามขายตอน RSI ต่ำเกิน (ผ่าน backtest)
rsiOK_buy  = rsiVal > 35 and rsiVal < rsiOB and rsiVal < noChaseHi
rsiOK_sell = rsiVal < 65 and rsiVal > rsiOS and rsiVal > noChaseLo
volOK = not volFilter or volume > avgVol * 0.7

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

// strictMTF = เข้าตามเทรนด์ MTF จริงๆ ไม่ใช่แค่ "ไม่สวน" (ผ่าน backtest)
mtfBuyOK  = not useMTF or (strictMTF ? confirmBias == 1  : confirmBias >= 0)
mtfSellOK = not useMTF or (strictMTF ? confirmBias == -1 : confirmBias <= 0)

// Sideway filter — ADX ต่ำ + EMA แบน = ตลาดออกข้าง → ไม่เทรด (ผ่าน backtest: DD ลดครึ่ง)
[diP_, diM_, adxVal] = ta.dmi(adxLen, adxLen)
emaSep   = math.abs(ema21 - ema50)
isRange  = adxVal < adxMin or (emaSlopeFlt and emaSep < emaSepMin * atrVal)
trending = not useRangeFlt or not isRange

buyScore  = (trendUp ? 1 : 0) + (rsiOK_buy ? 1 : 0) + (bullTrigger ? 1 : 0) + (volOK ? 1 : 0) + (marketBias == 1 ? 1 : 0) + (chochUp ? 1 : 0) + (bosUp ? 1 : 0)
sellScore = (trendDn ? 1 : 0) + (rsiOK_sell ? 1 : 0) + (bearTrigger ? 1 : 0) + (volOK ? 1 : 0) + (marketBias == -1 ? 1 : 0) + (chochDn ? 1 : 0) + (bosDn ? 1 : 0)

// ══════════════════════════════════════════
// TP / SL — แคบ + cap เพดาน + TP1/TP2
// ══════════════════════════════════════════
slDistBase = math.min(atrVal * slAtrMult, maxSLusd)

// ราคาเข้าอ้างอิง: โหมด limit = รอราคาย่อ (Buy ต่ำลง / Sell สูงขึ้น) ไม่งั้นใช้ราคาปิด
buyEntry  = useLimit ? close - pullbackAtr * atrVal : close
sellEntry = useLimit ? close + pullbackAtr * atrVal : close

buySwingDist = not na(swL1) and swL1 < buyEntry ? buyEntry - swL1 : na
buyUseSwing  = useSwingSL and not na(buySwingDist) and buySwingDist <= maxSLusd
buySLdist    = buyUseSwing ? math.min(buySwingDist + atrVal * 0.1, maxSLusd) : slDistBase
finalBuySL   = buyEntry - buySLdist
finalBuyTP1  = buyEntry + buySLdist * rr1
finalBuyTP2  = buyEntry + buySLdist * rr2

sellSwingDist = not na(swH1) and swH1 > sellEntry ? swH1 - sellEntry : na
sellUseSwing  = useSwingSL and not na(sellSwingDist) and sellSwingDist <= maxSLusd
sellSLdist    = sellUseSwing ? math.min(sellSwingDist + atrVal * 0.1, maxSLusd) : slDistBase
finalSellSL   = sellEntry + sellSLdist
finalSellTP1  = sellEntry - sellSLdist * rr1
finalSellTP2  = sellEntry - sellSLdist * rr2

buyQualOK  = buySLdist > 0
sellQualOK = sellSLdist > 0

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
// SIGNAL LOGIC — Confirmed vs Realtime
// ══════════════════════════════════════════
baseBuy  = bullTrigger and mtfBuyOK and buyQualOK and trending
baseSell = bearTrigger and mtfSellOK and sellQualOK and trending

confirmedBuy  = barstate.isconfirmed and buyScore >= minScore and baseBuy
confirmedSell = barstate.isconfirmed and sellScore >= minScore and baseSell

rtAlignBuy  = not rtNeedAlign or allBull
rtAlignSell = not rtNeedAlign or allBear

var int lastSignalBar = na
canSignal   = na(lastSignalBar) or (bar_index - lastSignalBar) >= signalGap

// FIX #6: realtime เว้นระยะได้เหมือน confirmed (กัน spam alert)
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
var line  rtL1 = na, var line rtL2 = na, var line rtL3 = na

if useRT
    label.delete(rtLabel)
    line.delete(rtL1), line.delete(rtL2), line.delete(rtL3)
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
// DRAW — CONFIRMED
// ══════════════════════════════════════════
// เก็บเส้นของแต่ละไม้ + ระดับ TP2/SL เพื่อยืดเส้นตามกราฟจนกว่าจะโดน
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
    bool  active

var TradeViz[] trades = array.new<TradeViz>()

if buySignal
    hdr = useLimit ? "BUY LIMIT\nเข้า " + str.tostring(buyEntry, "#.##") : "BUY " + str.tostring(close, "#.##")
    lblText = hdr +
         "\nTP1 " + str.tostring(finalBuyTP1, "#.##") + "  (+" + str.tostring(buyTP1Pts) + " / $" + str.tostring(buyTP1usd) + ")" +
         "\nTP2 " + str.tostring(finalBuyTP2, "#.##") + "  (+" + str.tostring(buyTP2Pts) + " / $" + str.tostring(buyTP2usd) + ")" +
         "\nSL  " + str.tostring(finalBuySL, "#.##") + "  (-" + str.tostring(buySLPts) + " / -$" + str.tostring(buySLusd) + ")"
    label.new(bar_index, low, lblText, style=label.style_label_up, color=buyColor, textcolor=color.white, size=size.normal)
    bLnTP2   = line.new(bar_index, finalBuyTP2, bar_index, finalBuyTP2, color=color.new(#00E676, 20), style=line.style_dashed, width=2)
    bLnTP1   = line.new(bar_index, finalBuyTP1, bar_index, finalBuyTP1, color=color.new(#00E676, 45), style=line.style_dashed, width=1)
    bLnSL    = line.new(bar_index, finalBuySL,  bar_index, finalBuySL,  color=color.new(#FF1744, 20), style=line.style_dashed, width=2)
    bLnEntry = line.new(bar_index, buyEntry,    bar_index, buyEntry,    color=color.new(#FFEB3B, 20), style=line.style_dotted, width=1)
    array.push(trades, TradeViz.new(bLnTP2, bLnTP1, bLnSL, bLnEntry, finalBuyTP2, finalBuyTP1, finalBuySL, buyEntry, true, true))

if sellSignal
    hdr = useLimit ? "SELL LIMIT\nเข้า " + str.tostring(sellEntry, "#.##") : "SELL " + str.tostring(close, "#.##")
    lblText = hdr +
         "\nTP1 " + str.tostring(finalSellTP1, "#.##") + "  (+" + str.tostring(sellTP1Pts) + " / $" + str.tostring(sellTP1usd) + ")" +
         "\nTP2 " + str.tostring(finalSellTP2, "#.##") + "  (+" + str.tostring(sellTP2Pts) + " / $" + str.tostring(sellTP2usd) + ")" +
         "\nSL  " + str.tostring(finalSellSL, "#.##") + "  (-" + str.tostring(sellSLPts) + " / -$" + str.tostring(sellSLusd) + ")"
    label.new(bar_index, high, lblText, style=label.style_label_down, color=sellColor, textcolor=color.white, size=size.normal)
    sLnTP2   = line.new(bar_index, finalSellTP2, bar_index, finalSellTP2, color=color.new(#FF1744, 20), style=line.style_dashed, width=2)
    sLnTP1   = line.new(bar_index, finalSellTP1, bar_index, finalSellTP1, color=color.new(#FF1744, 45), style=line.style_dashed, width=1)
    sLnSL    = line.new(bar_index, finalSellSL,  bar_index, finalSellSL,  color=color.new(#00E676, 20), style=line.style_dashed, width=2)
    sLnEntry = line.new(bar_index, sellEntry,    bar_index, sellEntry,    color=color.new(#FFEB3B, 20), style=line.style_dotted, width=1)
    array.push(trades, TradeViz.new(sLnTP2, sLnTP1, sLnSL, sLnEntry, finalSellTP2, finalSellTP1, finalSellSL, sellEntry, false, true))

// ── ยืดเส้นตามกราฟทุกแท่ง จนกว่าจะโดน TP2 หรือ SL แล้วหยุด (ขนานกับราคา) ──
if array.size(trades) > 0
    for i = 0 to array.size(trades) - 1
        tr = array.get(trades, i)
        if tr.active
            line.set_x2(tr.lnTP2, bar_index)
            line.set_x2(tr.lnTP1, bar_index)
            line.set_x2(tr.lnSL, bar_index)
            line.set_x2(tr.lnEntry, bar_index)
            hitTP2 = tr.isBuy ? high >= tr.tp2 : low <= tr.tp2
            hitSL  = tr.isBuy ? low <= tr.sl  : high >= tr.sl
            if hitTP2 or hitSL
                tr.active := false  // เทรดจบ → หยุดยืดเส้นตรงจุดที่โดน

// ══════════════════════════════════════════
// MTF INFO TABLE
// ══════════════════════════════════════════
f_biasCell(_b) => _b == 1 ? "UP" : _b == -1 ? "DOWN" : "-"
f_biasClr(_b)  => _b == 1 ? buyColor : _b == -1 ? sellColor : color.gray

var table t = table.new(position.top_right, 2, 10, bgcolor=color.new(#1a1a2e, 10), border_width=1, border_color=color.new(color.gray, 70))
if barstate.islast
    table.cell(t, 0, 0, "TIMEFRAME", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#0f3460, 20))
    table.cell(t, 1, 0, "BIAS", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#0f3460, 20))
    table.cell(t, 0, 1, "M15", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 1, f_biasCell(bias_m15), text_color=f_biasClr(bias_m15), text_size=size.tiny)
    table.cell(t, 0, 2, "M30", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 2, f_biasCell(bias_m30), text_color=f_biasClr(bias_m30), text_size=size.tiny)
    table.cell(t, 0, 3, "H1", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 3, f_biasCell(bias_h1), text_color=f_biasClr(bias_h1), text_size=size.tiny)
    table.cell(t, 0, 4, "H4", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 4, f_biasCell(bias_h4), text_color=f_biasClr(bias_h4), text_size=size.tiny)
    table.cell(t, 0, 5, "D1", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 5, f_biasCell(bias_d1), text_color=f_biasClr(bias_d1), text_size=size.tiny)
    alignTxt = mtfSum >= 3 ? "STRONG UP" : mtfSum <= -3 ? "STRONG DOWN" : "MIXED"
    table.cell(t, 0, 6, "ALIGN", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 6, alignTxt, text_color=mtfSum >= 3 ? buyColor : mtfSum <= -3 ? sellColor : color.gray, text_size=size.tiny)
    table.cell(t, 0, 7, "RSI", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 7, str.tostring(rsiVal, "#.#"), text_color=color.white, text_size=size.tiny)
    table.cell(t, 0, 8, "ATR", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 8, str.tostring(atrVal, "#.##"), text_color=color.white, text_size=size.tiny)
    rtStatus = not useRT ? "OFF" : realtimeBuy ? "BUY?" : realtimeSell ? "SELL?" : "รอ"
    table.cell(t, 0, 9, "REALTIME", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 9, rtStatus, text_color=realtimeBuy ? buyColor : realtimeSell ? sellColor : color.gray, text_size=size.tiny)

// ══════════════════════════════════════════
// ALERTS
// ══════════════════════════════════════════
alertcondition(buySignal,  title="Buy (ยืนยัน)",  message="XAUUSD BUY — วาง Buy Limit ตามราคาในป้าย (ดู TP/SL)")
alertcondition(sellSignal, title="Sell (ยืนยัน)", message="XAUUSD SELL — วาง Sell Limit ตามราคาในป้าย (ดู TP/SL)")
alertcondition(realtimeBuy,  title="Buy (realtime)",  message="XAUUSD BUY กำลังก่อตัว — ยังไม่ยืนยัน")
alertcondition(realtimeSell, title="Sell (realtime)", message="XAUUSD SELL กำลังก่อตัว — ยังไม่ยืนยัน")

// ── สำหรับ TradingView ฟรี: alert เดียวครอบทั้ง Buy+Sell ──
// สร้าง alert 1 อัน → เงื่อนไข = "Any alert() function call"
if buySignal
    alert("XAUUSD BUY (ยืนยัน) @ " + str.tostring(buyEntry, "#.##") + " | วาง Buy Limit | TP1 " + str.tostring(finalBuyTP1, "#.##") + " TP2 " + str.tostring(finalBuyTP2, "#.##") + " SL " + str.tostring(finalBuySL, "#.##"), alert.freq_once_per_bar_close)
if sellSignal
    alert("XAUUSD SELL (ยืนยัน) @ " + str.tostring(sellEntry, "#.##") + " | วาง Sell Limit | TP1 " + str.tostring(finalSellTP1, "#.##") + " TP2 " + str.tostring(finalSellTP2, "#.##") + " SL " + str.tostring(finalSellSL, "#.##"), alert.freq_once_per_bar_close)

// โซน Sideway (พื้นเทา) — ช่วงที่ระบบไม่เทรด
bgcolor(showRange and useRangeFlt and isRange ? rangeClr : na, title="Sideway zone")
bgcolor(buySignal ? color.new(buyColor, 92) : na)
bgcolor(sellSignal ? color.new(sellColor, 92) : na)
```
