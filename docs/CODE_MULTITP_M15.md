# XAUUSD Multi-TP M15 — โค้ดเต็มสำหรับคัดลอก

indicator แยกตัวใหม่สำหรับ M15 ทำหน้าตาตามภาพที่ส่งมา —
ป้ายราคา TP1-TP5 / ไม้เสริม / SL เรียงทางขวา พร้อมเลข point และบรรทัดเทรนด์ TF ใหญ่

⚠️ โครงสร้างระดับราคาถอดจากภาพได้ครบ แต่ตรรกะสัญญาณของต้นทางไม่มีใครเห็น
ตัวนี้ใช้ Supertrend พลิกทางซึ่งผมเขียนเอง **ยังไม่ผ่าน backtest** —
อ่านรายละเอียดที่ [MULTITP_M15.md](MULTITP_M15.md)

กดปุ่มคัดลอก (ไอคอนมุมขวาบนของกล่องโค้ด) แล้ววางใน Pine Editor
บรรทัดแรกต้องเป็น `//@version=5`

```pine
//@version=5
// ══════════════════════════════════════════════════════════════════════════
// XAUUSD Multi-TP M15
//
// indicator แยกต่างหาก ไม่เกี่ยวกับตระกูล SMC Pro ในโฟลเดอร์เดียวกันเลย
// ทำหน้าตาตามภาพที่ส่งมา: ป้ายราคาเรียงกันทางขวา TP1-TP5 / ไม้เสริม / SL
// พร้อมเลข point ในวงเล็บ + บรรทัดอ่านเทรนด์ TF ใหญ่ + คลาวด์สีตามเทรนด์
//
// โครงสร้างระดับราคา — ถอดมาจากตัวเลขในภาพตรง ๆ
//   ขั้น (step) = ATR × ตัวคูณ
//   TP1..TP5    = ราคาเข้า ± (1,2,3,4,5) × step      ← ห่างเท่ากันทุกขั้น
//   SL          = ราคาเข้า ∓ 1.30 × step
//   ไม้เสริม     = ราคาเข้า ∓ 0.50 × step
//   เลข point   = ระยะราคา ÷ syminfo.mintick (ทอง 3 ทศนิยม = 0.001)
//
// ⚠️ อ่านก่อนใช้
//   ภาพต้นทางเป็นโพสต์แจก EA ที่โชว์ไม้ที่ชนะ 1 ไม้ ตรรกะข้างในเขาไม่มีใครเห็น
//   ไฟล์นี้ลอกได้แค่ "หน้าตาและโครงสร้างระดับราคา" ส่วนเครื่องยนต์สัญญาณเป็น
//   Supertrend ธรรมดาที่ผมเขียนเอง อธิบายได้ทุกบรรทัด แต่ยังไม่ผ่าน backtest
//   ใด ๆ ทั้งสิ้น — เอาไปรันเดโมเก็บสถิติก่อนเสมอ
//
//   และรู้ไว้ว่าโครงสร้าง TP ห่างเท่ากัน 5 ชั้นแบบนี้ TP5 อยู่ไกลถึง 3.85R
//   ภาพเดียวที่ทะลุ TP5 ไม่ได้บอกว่ามันทะลุบ่อยแค่ไหน ตารางในโค้ดนี้จึงนับ
//   ให้ดูว่าจริง ๆ แล้วแต่ละชั้นโดนกี่ครั้ง
// ══════════════════════════════════════════════════════════════════════════
indicator("XAUUSD Multi-TP M15", overlay=true, max_labels_count=500, max_lines_count=500)

// ══════════════════════════════════════════
// 1. สัญญาณ
// ══════════════════════════════════════════
g1        = "1. สัญญาณ"
stFactor  = input.float(3.0, "Supertrend Factor", minval=0.5, step=0.1, group=g1, tooltip="ยิ่งมากยิ่งกรองแน่น สัญญาณน้อยลงแต่ทนการแกว่งได้ดีขึ้น (M15 ทอง แนะนำ 2.5-4)")
stAtrLen  = input.int(10, "Supertrend ATR", minval=2, group=g1)
useHtfFlt = input.bool(true, "เข้าตามเทรนด์ TF ที่ 1 เท่านั้น", group=g1, tooltip="เปิด = Buy เฉพาะตอน TF ที่ 1 เป็นบวก / Sell เฉพาะตอนเป็นลบ")
minBarGap = input.int(4, "เว้นระยะสัญญาณขั้นต่ำ (แท่ง)", minval=1, maxval=50, group=g1)

// ══════════════════════════════════════════
// 2. TP / SL  (ตัวเลขตามภาพ)
// ══════════════════════════════════════════
g2        = "2. TP / SL"
atrLen    = input.int(14, "ATR Length", minval=1, group=g2)
stepMult  = input.float(1.0, "ระยะ 1 ขั้น = ATR ×", minval=0.1, step=0.1, group=g2, tooltip="TP แต่ละชั้นห่างกันเท่านี้ ในภาพต้นทางขั้นละ 3.915 ซึ่งใกล้เคียง ATR(14) ของ M15 พอดี")
tpCount   = input.int(5, "จำนวน TP", minval=1, maxval=5, group=g2)
slMult    = input.float(1.3, "SL = ระยะขั้น ×", minval=0.2, step=0.1, group=g2, tooltip="ถอดจากภาพ: SL ห่าง 5.089 = 1.30 เท่าของขั้น 3.915")
useAdd    = input.bool(true, "แสดงไม้เสริม (+Buy.1 / +Sell.1)", group=g2, tooltip="ไม้ที่สองเผื่อราคาย่อกลับมาอีกนิด — ในภาพห่างจากไม้แรก 0.50 เท่าของขั้น")
addMult   = input.float(0.5, "ไม้เสริมห่างจากไม้แรก = ระยะขั้น ×", minval=0.1, step=0.1, group=g2)
closeOnSL = input.bool(true, "โดน SL แล้วจบไม้", group=g2)

// ══════════════════════════════════════════
// 3. อ่านเทรนด์ TF ใหญ่
// ══════════════════════════════════════════
g3      = "3. เทรนด์ TF ใหญ่"
htf1    = input.timeframe("60",  "TF ที่ 1", group=g3)
htf2    = input.timeframe("240", "TF ที่ 2", group=g3)
htfEma  = input.int(50, "วัดจากระยะห่าง close กับ EMA", minval=5, group=g3, tooltip="ค่าที่แสดง = (close ของ TF นั้น − EMA) แปลงเป็น point / ติดลบ = อยู่ใต้ EMA = ขาลง")
noRepnt = input.bool(true, "ไม่ repaint (ใช้ค่าแท่งที่ปิดแล้ว)", group=g3, tooltip="เปิดไว้เสมอตอนใช้จริง ไม่งั้นค่าจะขยับไปมาระหว่างแท่ง")

// ══════════════════════════════════════════
// 4. หน้าตา
// ══════════════════════════════════════════
g4        = "4. หน้าตา"
showCloud = input.bool(true, "แสดงคลาวด์เทรนด์", group=g4)
emaFast   = input.int(20, "EMA เร็ว (ขอบคลาวด์)", minval=2, group=g4)
emaSlow   = input.int(50, "EMA ช้า (ขอบคลาวด์)", minval=3, group=g4)
emaLong   = input.int(200, "EMA ยาว (เส้นเทา)", minval=10, group=g4)
lblOffset = input.int(4, "ป้ายห่างจากแท่งล่าสุด (แท่ง)", minval=0, maxval=30, group=g4)
showTrLbl = input.bool(true, "แสดงบรรทัดเทรนด์ TF ใหญ่บนกราฟ", group=g4)
showPanel = input.bool(true, "แสดงตารางสรุป", group=g4)
upClr     = input.color(color.new(#26A69A, 0),  "สีขาขึ้น", group=g4)
dnClr     = input.color(color.new(#EF5350, 0),  "สีขาลง", group=g4)
cloudUp   = input.color(color.new(#26A69A, 80), "คลาวด์ขาขึ้น", group=g4)
cloudDn   = input.color(color.new(#EF5350, 80), "คลาวด์ขาลง", group=g4)
tpClr     = input.color(color.new(#2196F3, 0),  "ป้าย TP ที่ยังไม่โดน", group=g4)
tpHitClr  = input.color(color.new(#00E676, 0),  "ป้าย TP ที่โดนแล้ว", group=g4)
slClr     = input.color(color.new(#AB47BC, 0),  "ป้าย SL", group=g4)

// ══════════════════════════════════════════
// CORE
// ══════════════════════════════════════════
atrVal = ta.atr(atrLen)
eFast  = ta.ema(close, emaFast)
eSlow  = ta.ema(close, emaSlow)
eLong  = ta.ema(close, emaLong)

[stLine, stDir] = ta.supertrend(stFactor, stAtrLen)
// Pine: stDir = -1 คือขาขึ้น, 1 คือขาลง
stUp = stDir == -1

// ── ตัวช่วยจัดรูปแบบตัวเลข ──
ptSize = syminfo.mintick

// ใส่คอมมาคั่นหลักพันเอง ไม่พึ่ง format string ของ str.tostring
f_comma(_v) =>
    neg = _v < 0
    s   = str.tostring(math.round(math.abs(_v)), "#")
    n   = str.length(s)
    out = ""
    for i = 0 to n - 1
        out := str.substring(s, n - 1 - i, n - i) + out
        if (i + 1) % 3 == 0 and i < n - 1
            out := "," + out
    (neg ? "-" : "") + out

f_px(_p)   => str.tostring(_p, format.mintick)
f_pts(_d)  => f_comma(_d / ptSize) + " pts"

// ══════════════════════════════════════════
// เทรนด์ TF ใหญ่ — ระยะห่างจาก EMA แปลงเป็น point
// ══════════════════════════════════════════
f_htfGap(_tf) =>
    live = request.security(syminfo.tickerid, _tf, close - ta.ema(close, htfEma), lookahead=barmerge.lookahead_off)
    done = request.security(syminfo.tickerid, _tf, (close - ta.ema(close, htfEma))[1], gaps=barmerge.gaps_off, lookahead=barmerge.lookahead_on)
    noRepnt ? done : live

gap1 = f_htfGap(htf1)
gap2 = f_htfGap(htf2)
htf1OK_buy  = not useHtfFlt or gap1 > 0
htf1OK_sell = not useHtfFlt or gap1 < 0

// ══════════════════════════════════════════
// สัญญาณ — Supertrend พลิกทาง
// ══════════════════════════════════════════
var int lastSigBar = na
gapOK = na(lastSigBar) or (bar_index - lastSigBar) >= minBarGap

flipUp = stUp and not stUp[1]
flipDn = not stUp and stUp[1]

buySig  = barstate.isconfirmed and flipUp and gapOK and htf1OK_buy  and not na(atrVal)
sellSig = barstate.isconfirmed and flipDn and gapOK and htf1OK_sell and not na(atrVal)

// ══════════════════════════════════════════
// สถานะไม้ที่ถืออยู่
// ══════════════════════════════════════════
var int    tDir    = 0        // 1 = buy, -1 = sell, 0 = ว่าง
var float  tEntry  = na
var float  tAdd    = na
var float  tSL     = na
var float  tStep   = na
var int    tBar    = na
var float[] tpLv   = array.new_float(5, na)
var bool[]  tpHit  = array.new_bool(5, false)

// นับสถิติว่าจริง ๆ แล้วแต่ละชั้นโดนกี่ครั้ง
var int[] tpCnt   = array.new_int(5, 0)
var int   nTrades = 0
var int   nSL     = 0

newSig = buySig or sellSig

if newSig
    tDir   := buySig ? 1 : -1
    tEntry := close
    tStep  := atrVal * stepMult
    tAdd   := tEntry - tDir * addMult * tStep
    tSL    := tEntry - tDir * slMult  * tStep
    tBar   := bar_index
    lastSigBar := bar_index
    nTrades := nTrades + 1
    for i = 0 to 4
        array.set(tpLv,  i, tEntry + tDir * (i + 1) * tStep)
        array.set(tpHit, i, false)

// ── เดินสถานะ ──
hitSLnow = false
int hitTPnow = 0

if barstate.isconfirmed and tDir != 0 and bar_index > tBar
    for i = 0 to tpCount - 1
        if not array.get(tpHit, i)
            lv = array.get(tpLv, i)
            if tDir == 1 ? high >= lv : low <= lv
                array.set(tpHit, i, true)
                array.set(tpCnt, i, array.get(tpCnt, i) + 1)
                hitTPnow := i + 1
    if (tDir == 1 ? low <= tSL : high >= tSL)
        hitSLnow := true
        nSL := nSL + 1
        if closeOnSL
            tDir := 0

// ══════════════════════════════════════════
// วาด — คลาวด์ + เส้น
// ══════════════════════════════════════════
pFast = plot(showCloud ? eFast : na, "EMA เร็ว",  color=color.new(color.gray, 40), linewidth=1)
pSlow = plot(showCloud ? eSlow : na, "EMA ช้า",   color=color.new(color.gray, 40), linewidth=1)
fill(pFast, pSlow, color=eFast > eSlow ? cloudUp : cloudDn, title="คลาวด์เทรนด์")
plot(eLong, "EMA ยาว", color=color.new(color.silver, 30), linewidth=2)
plot(stLine, "Supertrend", color=stUp ? color.new(upClr, 40) : color.new(dnClr, 40), linewidth=1, style=plot.style_linebr)

plotshape(buySig,  "Buy",  shape.triangleup,   location.belowbar, color=upClr, text="Buy",  textcolor=upClr, size=size.tiny)
plotshape(sellSig, "Sell", shape.triangledown, location.abovebar, color=dnClr, text="Sell", textcolor=dnClr, size=size.tiny)

// ══════════════════════════════════════════
// วาด — ป้ายราคาทางขวา (สร้างครั้งเดียว แล้วอัปเดตทุกแท่ง)
// ไม่ push เข้า array ไม่สร้างใหม่ทุกแท่ง จึงไม่มีทางชนลิมิต drawing
// ══════════════════════════════════════════
var label lbEntry = na
var label lbAdd   = na
var label lbSL    = na
var label lbTP1 = na, var label lbTP2 = na, var label lbTP3 = na, var label lbTP4 = na, var label lbTP5 = na
var line  lnEntry = na
var line  lnAdd   = na
var line  lnSL    = na
var line  lnTP1 = na, var line lnTP2 = na, var line lnTP3 = na, var line lnTP4 = na, var line lnTP5 = na
var label lbTr1 = na
var label lbTr2 = na

f_mkLabel(_clr) => label.new(bar_index, close, "", xloc=xloc.bar_index, style=label.style_label_left, color=_clr, textcolor=color.white, size=size.small)
f_mkLine(_clr)  => line.new(bar_index, close, bar_index, close, color=_clr, style=line.style_dashed, width=1)

if newSig and na(lbEntry)
    lbEntry := f_mkLabel(upClr)
    lbAdd   := f_mkLabel(upClr)
    lbSL    := f_mkLabel(slClr)
    lbTP1   := f_mkLabel(tpClr)
    lbTP2   := f_mkLabel(tpClr)
    lbTP3   := f_mkLabel(tpClr)
    lbTP4   := f_mkLabel(tpClr)
    lbTP5   := f_mkLabel(tpClr)
    lnEntry := f_mkLine(upClr)
    lnAdd   := f_mkLine(upClr)
    lnSL    := f_mkLine(slClr)
    lnTP1   := f_mkLine(tpClr)
    lnTP2   := f_mkLine(tpClr)
    lnTP3   := f_mkLine(tpClr)
    lnTP4   := f_mkLine(tpClr)
    lnTP5   := f_mkLine(tpClr)

// จบด้วยค่า bool เพื่อให้ Pine รู้ชนิดที่ฟังก์ชันคืนแน่นอน
// (ฟังก์ชันที่ตัวสุดท้ายเป็น if เปล่า ๆ Pine จะเดาชนิดไม่ออก)
f_place(_lb, _ln, _px, _txt, _clr, _on) =>
    y = _on ? _px : na
    label.set_xy(_lb, bar_index + lblOffset, y)
    label.set_text(_lb, _on ? _txt : "")
    label.set_color(_lb, _clr)
    line.set_xy1(_ln, tBar, y)
    line.set_xy2(_ln, bar_index + lblOffset, y)
    line.set_color(_ln, _clr)
    _on

// อัปเดตเฉพาะแท่งล่าสุด — ป้ายกับเส้นเห็นผลแค่ขอบขวาอยู่แล้ว
// และการต่อสตริงทุกแท่งย้อนหลังหลายพันแท่งเสี่ยงชนเพดานเวลาประมวลผลของ Pine
if not na(lbEntry) and barstate.islast
    on   = tDir != 0
    dirT = tDir == 1 ? "Buy ▲" : "Sell ▼"
    eClr = tDir == 1 ? upClr : dnClr
    f_place(lbEntry, lnEntry, tEntry, dirT + " : " + f_px(tEntry), eClr, on)
    f_place(lbAdd,   lnAdd,   tAdd,   "+" + (tDir == 1 ? "Buy" : "Sell") + ".1 : " + f_px(tAdd), color.new(eClr, 35), on and useAdd)
    f_place(lbSL,    lnSL,    tSL,    "SL : " + f_px(tSL) + "  (" + f_pts(math.abs(tEntry - tSL)) + ")", slClr, on)
    for i = 0 to 4
        lv  = array.get(tpLv, i)
        hit = array.get(tpHit, i)
        txt = "TP" + str.tostring(i + 1) + " : " + f_px(lv) + "  (" + f_pts(math.abs(lv - tEntry)) + ")" + (hit ? "  ✔" : "")
        clr = hit ? tpHitClr : tpClr
        shown = on and (i < tpCount)
        if i == 0
            f_place(lbTP1, lnTP1, lv, txt, clr, shown)
        if i == 1
            f_place(lbTP2, lnTP2, lv, txt, clr, shown)
        if i == 2
            f_place(lbTP3, lnTP3, lv, txt, clr, shown)
        if i == 3
            f_place(lbTP4, lnTP4, lv, txt, clr, shown)
        if i == 4
            f_place(lbTP5, lnTP5, lv, txt, clr, shown)

// ── บรรทัดอ่านเทรนด์ TF ใหญ่ วางไว้ที่ระดับ EMA ของกราฟ ──
if showTrLbl and barstate.islast
    if na(lbTr1)
        lbTr1 := label.new(bar_index, close, "", xloc=xloc.bar_index, style=label.style_label_left, color=color.new(color.gray, 20), textcolor=color.white, size=size.small)
        lbTr2 := label.new(bar_index, close, "", xloc=xloc.bar_index, style=label.style_label_left, color=color.new(color.gray, 20), textcolor=color.white, size=size.small)
    label.set_xy(lbTr1, bar_index + lblOffset, eSlow)
    label.set_text(lbTr1, htf1 + " Trend : " + f_pts(gap1))
    label.set_textcolor(lbTr1, gap1 > 0 ? upClr : dnClr)
    label.set_xy(lbTr2, bar_index + lblOffset, eLong)
    label.set_text(lbTr2, htf2 + " Trend : " + f_pts(gap2))
    label.set_textcolor(lbTr2, gap2 > 0 ? upClr : dnClr)

// ══════════════════════════════════════════
// ตารางสรุป — ตอบคำถามว่าแต่ละชั้นโดนจริงกี่ครั้ง
// ══════════════════════════════════════════
var table t = table.new(position.top_right, 2, 10, bgcolor=color.new(#141823, 8), border_width=1, border_color=color.new(color.gray, 70))
if showPanel and barstate.islast
    table.cell(t, 0, 0, "Multi-TP", text_color=color.white, text_size=size.tiny, bgcolor=color.new(#1E3A5F, 20))
    table.cell(t, 1, 0, timeframe.period, text_color=color.white, text_size=size.tiny, bgcolor=color.new(#1E3A5F, 20))
    table.cell(t, 0, 1, "สถานะ", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 1, tDir == 0 ? "รอสัญญาณ" : tDir == 1 ? "ถือ Buy" : "ถือ Sell", text_color=tDir == 0 ? color.gray : tDir == 1 ? upClr : dnClr, text_size=size.tiny)
    table.cell(t, 0, 2, "ระยะ 1 ขั้น", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 2, str.tostring(atrVal * stepMult, "#.###") + "  (" + f_pts(atrVal * stepMult) + ")", text_color=color.white, text_size=size.tiny)
    table.cell(t, 0, 3, htf1 + " Trend", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 3, f_pts(gap1), text_color=gap1 > 0 ? upClr : dnClr, text_size=size.tiny)
    table.cell(t, 0, 4, htf2 + " Trend", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 4, f_pts(gap2), text_color=gap2 > 0 ? upClr : dnClr, text_size=size.tiny)
    table.cell(t, 0, 5, "ไม้ทั้งหมด", text_color=color.gray, text_size=size.tiny)
    table.cell(t, 1, 5, str.tostring(nTrades) + "  (โดน SL " + str.tostring(nSL) + ")", text_color=color.white, text_size=size.tiny)
    for i = 0 to 4
        pctTxt = nTrades > 0 ? str.tostring(array.get(tpCnt, i) * 100.0 / nTrades, "#") + "%" : "-"
        table.cell(t, 0, 6 + i, "ถึง TP" + str.tostring(i + 1), text_color=i < tpCount ? color.gray : color.new(color.gray, 60), text_size=size.tiny)
        table.cell(t, 1, 6 + i, str.tostring(array.get(tpCnt, i)) + " / " + str.tostring(nTrades) + "  " + pctTxt, text_color=i < tpCount ? color.white : color.new(color.gray, 60), text_size=size.tiny)

// ══════════════════════════════════════════
// ALERTS
// ══════════════════════════════════════════
alertcondition(buySig,  title="Buy",  message="XAUUSD M15 — Buy (Supertrend พลิกขึ้น) ดูราคาในป้ายบนกราฟ")
alertcondition(sellSig, title="Sell", message="XAUUSD M15 — Sell (Supertrend พลิกลง) ดูราคาในป้ายบนกราฟ")

if newSig
    alert("XAUUSD M15 " + (buySig ? "BUY" : "SELL") +
         " @ " + f_px(tEntry) +
         (useAdd ? " | ไม้เสริม " + f_px(tAdd) : "") +
         " | SL " + f_px(tSL) + " (" + f_pts(math.abs(tEntry - tSL)) + ")" +
         " | TP1 " + f_px(array.get(tpLv, 0)) +
         " TP" + str.tostring(tpCount) + " " + f_px(array.get(tpLv, tpCount - 1)) +
         " | ขั้นละ " + f_pts(tStep), alert.freq_once_per_bar_close)
if hitTPnow > 0
    alert("XAUUSD M15 — ถึง TP" + str.tostring(hitTPnow) + " ที่ " + f_px(array.get(tpLv, hitTPnow - 1)), alert.freq_once_per_bar_close)
if hitSLnow
    alert("XAUUSD M15 — โดน SL ที่ " + f_px(tSL), alert.freq_once_per_bar_close)
```
