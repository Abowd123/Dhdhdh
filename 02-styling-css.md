# CivilDraft — Styling (CSS)

> كل ملفات الأنماط الخاصة بالواجهة.

**عدد الملفات:** 10

---

## `css/base.css`

```css
*{box-sizing:border-box}
[hidden]{display:none!important}

html,body{height:100%;margin:0}
body{
 background:var(--bg); color:var(--fg);
 font:13px/1.5 var(--ui);
 display:flex; flex-direction:column; overflow:hidden;
 -webkit-font-smoothing:antialiased; text-rendering:optimizeLegibility;
}
button,input,select,textarea{font-family:inherit;font-size:12.5px}
button{transition:background var(--t) var(--ease),
 color var(--t) var(--ease),border-color var(--t) var(--ease),
 box-shadow var(--t) var(--ease)}
:focus-visible{outline:2px solid color-mix(in srgb,var(--ac) 72%,
 transparent);outline-offset:1px}
@media (prefers-reduced-motion:reduce){
 *{transition:none!important;animation:none!important;
   scroll-behavior:auto!important}
}
.gap{flex:1}
.dv{width:1px;height:18px;display:inline-block;
 background:linear-gradient(transparent,var(--ln) 22%,
 var(--ln) 78%,transparent)}
.mono{font-family:var(--mono)}

#top{
 flex:none; display:flex; align-items:center; gap:11px;
 padding:7px 14px;
 background:linear-gradient(180deg,
  color-mix(in srgb,var(--bg3) 58%,var(--bg2)),var(--bg2));
 border-bottom:1px solid var(--ln);
 box-shadow:var(--sh1),var(--edge);
}
.brand{color:var(--brand);letter-spacing:.6px;font-size:15px;
 font-weight:600}
#top .ver,#top .tip{color:var(--fg3);font-size:11.5px;
 letter-spacing:.15px}
#top .tip{opacity:.8}

#main{flex:1;display:flex;min-height:0}
#work{flex:1;display:flex;flex-direction:column;min-width:0}
#stage{flex:1;position:relative;min-height:0;background:var(--bg);
 direction:ltr}
#cv{position:absolute;inset:0;display:block;outline:none;
 cursor:crosshair;touch-action:none}

.num,input.num{direction:ltr;unicode-bidi:isolate;text-align:start}
.num,input.num,#stPos,#clLive,#log{font-variant-numeric:tabular-nums}

#icoSheet{position:absolute;width:0;height:0;overflow:hidden}
.ic{flex:none;stroke:currentColor;fill:none;stroke-width:1.6;
 stroke-linecap:round;stroke-linejoin:round}

#cmdline{
 position:relative; flex:none;
 display:flex; align-items:center; gap:9px;
 padding:8px 12px; background:var(--bg2);
 border-top:1px solid var(--ln);
}
#clSug{
 position:absolute; inset-inline:10px; bottom:calc(100% + 4px);
 z-index:25; background:var(--bg3);
 border:1px solid var(--ln2); border-radius:var(--r3);
 overflow:hidden; box-shadow:var(--sh3),var(--edge);
 font-family:var(--mono); font-size:12px;
}
#clSug .it{padding:6px 11px;color:var(--fg2);cursor:pointer}
#clSug .it:hover{background:var(--hov);color:var(--fg)}
#clSug .it.sel{background:var(--acq);color:var(--acf);
 box-shadow:inset 2px 0 0 var(--ac)}
#clPrompt{
 color:var(--brand); font-family:var(--mono); font-size:12px;
 white-space:nowrap; min-width:200px; letter-spacing:.2px;
}
#clIn{
 flex:1; min-width:80px; background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd); border-radius:var(--r2);
 padding:7px 11px; font-family:var(--mono);
 box-shadow:inset 0 1px 2px rgba(0,0,0,.25);
}
#clIn:focus{outline:none;border-color:var(--ac);
 box-shadow:var(--focus)}
#clLive{
 color:var(--ok); font-family:var(--mono); font-size:12px;
 white-space:nowrap; min-width:130px;
 direction:ltr; unicode-bidi:isolate; text-align:start;
}

#log{
 flex:none; height:120px; min-height:32px; max-height:60vh;
 overflow-y:auto; resize:vertical;
 padding:6px 10px;
 background:var(--bg4); border-top:1px solid var(--ln);
 font-family:var(--mono); font-size:11.5px; line-height:1.6;
}
#log::-webkit-scrollbar{width:9px}
#log::-webkit-scrollbar-thumb{background:var(--thumb);
 border-radius:5px}
#log .ln{white-space:pre-wrap;word-break:break-word;
 padding-inline-start:7px;border-inline-start:2px solid transparent}
#log .ok{color:var(--ok);border-inline-start-color:
 color-mix(in srgb,var(--ok) 45%,transparent)}
#log .wr{color:var(--wr);border-inline-start-color:
 color-mix(in srgb,var(--wr) 45%,transparent)}
#log .er{color:var(--er);border-inline-start-color:
 color-mix(in srgb,var(--er) 55%,transparent)}
#log .in{color:var(--fg3)}

#status{
 flex:none; position:relative; display:flex; align-items:center;
 gap:8px; padding:5px 12px; background:var(--bg2);
 border-top:1px solid var(--ln); color:var(--fg3); font-size:11.5px;
}
#status button{
 background:transparent; color:var(--fg3);
 border:1px solid transparent; border-radius:999px;
 padding:3px 10px; cursor:pointer;
}
#status button:hover{background:var(--hov);color:var(--fg2)}
#status button.on{background:var(--acq);color:var(--acf);
 border-color:var(--acb)}
#status #osBtn{padding:2px 5px}
#stPos{color:var(--fg2);min-width:132px;
 direction:ltr;unicode-bidi:isolate;text-align:start}
#stInfo{color:var(--ok);max-width:38vw;overflow:hidden;
 text-overflow:ellipsis;white-space:nowrap}
#stSel{color:var(--wr)}

#osPop{
 position:absolute; bottom:calc(100% + 6px); z-index:30;
 background:var(--bg3); border:1px solid var(--ln2);
 border-radius:var(--r3); padding:10px 12px; min-width:190px;
 box-shadow:var(--sh3),var(--edge);
}
#osPop h5{margin:0 0 7px;color:var(--fg2);font-size:12px;
 font-weight:600;letter-spacing:.2px}
#osPop label{display:flex;align-items:center;gap:6px;
 padding:3px 0;color:var(--fg2);cursor:pointer;border-radius:var(--r1)}
#osPop label:hover{color:var(--fg)}
#osPop .pi{margin-top:8px;padding-top:8px;
 border-top:1px solid var(--ln);display:flex;align-items:center;
 gap:6px;color:var(--fg2)}
#osPop .pi input{width:54px;background:var(--bg4);color:var(--fg);
 border:1px solid var(--fldbd);border-radius:var(--r1);padding:2px 5px}
#osPop .fr{display:flex;gap:5px;margin-top:8px}
#osPop .fr button{flex:1;background:var(--bg2);color:var(--fg2);
 border:1px solid var(--ln);border-radius:var(--r1);padding:4px 0;
 cursor:pointer}
#osPop .fr button:hover{background:var(--hov);color:var(--fg);
 border-color:var(--ln2)}

#helpBox{
 position:fixed; inset-block-start:6vh; inset-inline:0;
 width:min(720px,92vw); margin-inline:auto;
 max-height:88vh; z-index:60;
 background:var(--bg3); border:1px solid var(--ln2);
 border-radius:var(--r4); box-shadow:var(--sh3),var(--edge);
 display:flex; flex-direction:column; overflow:hidden;
}
#helpBox .hd{
 flex:none; display:flex; align-items:center;
 justify-content:space-between; gap:10px; padding:11px 16px;
 background:linear-gradient(180deg,
  color-mix(in srgb,var(--bg2) 88%,var(--brand) 4%),var(--bg2));
 border-bottom:1px solid var(--ln); color:var(--brand);
 font-weight:600; letter-spacing:.2px;
}
#helpBox .hd button{
 background:var(--bg4); color:var(--fg2);
 border:1px solid var(--ln); border-radius:var(--r2);
 padding:4px 11px; cursor:pointer;
}
#helpBox .hd button:hover{background:var(--hov);color:var(--fg);
 border-color:var(--ln2)}
#helpBox .bd{flex:1;overflow-y:auto;padding:12px 18px 20px}
#helpBox .bd h4{color:var(--ac);font-size:12.5px;margin:18px 0 7px;
 letter-spacing:.2px}
#helpBox .bd h4:first-child{margin-top:0}
#helpBox .bd ul{margin:0 0 11px;padding-inline-start:20px}
#helpBox .bd li{margin:5px 0;color:var(--fg2)}
#helpBox .bd code{
 font-family:var(--mono); background:var(--bg4); color:var(--fg);
 padding:1px 6px; border-radius:var(--r1);
 border:1px solid var(--ln);
}

table.tools{width:100%;border-collapse:collapse;font-size:12px}
table.tools th{
 text-align:start; color:var(--fg3); font-weight:600;
 padding:6px 9px; border-bottom:1px solid var(--ln2);
 letter-spacing:.3px;
}
table.tools td{
 padding:6px 9px; border-bottom:1px solid var(--ln);
 color:var(--fg2); vertical-align:top;
}
table.tools tr:hover td{background:var(--hov);color:var(--fg)}
```

<a id="f-css-cmd-css"></a>

---

## `css/cmd.css`

```css
/* ═══ سطر الأوامر · الإدخال الحركي · القوائم · الخصائص السريعة ═══ */
#cmdWrap{
 flex:none; display:flex; flex-direction:column; min-inline-size:0;
 opacity:var(--cmdOpa,1);
}
#cmdWrap[data-mode="top"]{order:-1}
#cmdWrap[data-mode="top"] #cmdline{
 border-block-start:none; border-block-end:1px solid var(--ln)}
#cmdWrap[data-mode="top"] #log{
 border-block-start:none; border-block-end:1px solid var(--ln);
 order:-1}
#cmdFloat{position:fixed;inset:0;z-index:30;pointer-events:none}
#cmdWrap[data-mode="float"]{
 position:fixed; pointer-events:auto; z-index:30;
 min-inline-size:320px; resize:horizontal; overflow:hidden;
 border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge);
}
#cmdWrap[data-mode="float"] #cmdline{
 cursor:grab; border-block-start:none}
:root.dragging #cmdWrap[data-mode="float"] #cmdline{cursor:grabbing}
#cmdWrap[data-mode="float"] #log{border-radius:0 0 var(--r3) var(--r3)}
#clMenuBtn{
 flex:none; background:transparent; color:var(--fg3);
 border:1px solid transparent; border-radius:var(--r1);
 padding:1px 5px; cursor:pointer; line-height:1.4;
}
#clMenuBtn:hover{background:var(--hov);color:var(--fg)}
#cMenu{
 position:fixed; z-index:76; min-inline-size:230px; max-block-size:80vh;
 overflow-y:auto;
 background:var(--bg3); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge); padding:4px 0;
}

/* ═══ الإدخال الحركي ═══ داخل #stage المعزول ═══ */
#dynBox{
 position:absolute; inset-inline-start:0; inset-block-start:0;
 z-index:10; display:flex; align-items:center; gap:5px;
 padding:4px 6px; border-radius:var(--r2);
 background:color-mix(in srgb,var(--bg3) 94%,transparent);
 border:1px solid var(--brandb);
 box-shadow:var(--sh2),var(--edge);
 pointer-events:auto; will-change:transform;
}
.dF{display:inline-flex;align-items:center;gap:3px}
.dF label{color:var(--fg3);font-size:10.5px;margin:0}
.dF input{
 inline-size:62px; background:var(--bg4); color:var(--ok);
 border:1px solid var(--fldbd); border-radius:var(--r1); padding:2px 4px;
 font-family:var(--mono); font-size:11.5px;
}
.dF input:focus{outline:none;border-color:var(--ac);color:var(--fg)}
.du{color:var(--fg3);font-size:10px}
.dK{color:var(--brand);font-size:10.5px;font-family:var(--mono)}

/* ═══ قائمة السياق ═══ */
#ctxMenu{
 position:fixed; z-index:78; min-inline-size:214px; max-block-size:82vh;
 overflow-y:auto;
 background:var(--bg3); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge); padding:4px 0;
}

/* ═══ الخصائص السريعة ═══ */
#qpCard{
 position:absolute; z-index:11; inline-size:238px;
 background:color-mix(in srgb,var(--bg2) 95%,transparent);
 border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh2),var(--edge);
 overflow:hidden;
}
.qpH{
 display:flex; align-items:center; gap:4px;
 padding:4px 8px; background:var(--bg3);
 border-block-end:1px solid var(--ln);
 color:var(--brand); font-size:11.5px; cursor:grab;
}
:root.dragging .qpH{cursor:grabbing}
.qpH span{flex:1;overflow:hidden;text-overflow:ellipsis;
 white-space:nowrap}
.qpH button{
 flex:none; background:transparent; color:var(--fg3);
 border:1px solid transparent; border-radius:var(--r1);
 padding:0 4px; cursor:pointer; font-size:11px; line-height:1.5;
}
.qpH button:hover{background:var(--hov);color:var(--fg)}
#qpBody{padding:5px 8px 7px;display:flex;flex-direction:column;
 gap:4px}
.qf{display:flex;align-items:center;gap:5px}
.qf label{
 flex:none; inline-size:74px; color:var(--fg3); font-size:11px;
 margin:0; overflow:hidden; text-overflow:ellipsis;
 white-space:nowrap;
}
.qf input,.qf select{
 flex:1; min-inline-size:0; background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd); border-radius:var(--r1); padding:2px 5px;
 font-size:11.5px;
}
.qf input.num{direction:ltr;unicode-bidi:isolate;text-align:start}
.qf input:focus,.qf select:focus{outline:none;border-color:var(--ac);
 box-shadow:var(--focus)}
#qpBody .chk{font-size:11.5px}
#qpBody .hint{margin:2px 0;font-size:11px;color:var(--fg3)}

@media (max-height:700px){
 #cmdWrap[data-mode="float"]{max-block-size:44vh}
}
```

<a id="f-css-dock-css"></a>

---

## `css/dock.css`

```css
/* ═══ الإرساء: أعمدة · عائمة · شارات · قوائم ═══
   خصائص منطقية بحتة — dom.js يحرس ذلك. */

#main{position:relative}

/* ═══ الأعمدة ═══ */
.dock{
 flex:none; overflow-y:auto; overflow-x:hidden;
 background:var(--bg2); padding:0 0 26px;
 min-inline-size:0;
}
#side{border-inline-end:1px solid var(--ln)}
#sideE{border-inline-start:1px solid var(--ln)}
.dock::-webkit-scrollbar{width:9px}
.dock::-webkit-scrollbar-thumb{background:var(--thumb);
 border-radius:5px}
.dock::-webkit-scrollbar-thumb:hover{background:var(--ln2)}

/* العمود المُخفى تلقائياً يطفو فوق اللوحة عند انكشافه */
.dock.auto{
 position:absolute; inset-block:0; z-index:22;
 box-shadow:var(--sh3);
}
#side.auto{inset-inline-start:27px;
 border-inline-start:1px solid var(--ln2)}
#sideE.auto{inset-inline-end:27px;
 border-inline-end:1px solid var(--ln2)}

/* ═══ الفاصل ═══ */
.dsz{
 flex:none; inline-size:5px; cursor:col-resize;
 background:transparent; border:none; padding:0;
 align-self:stretch; transition:background var(--t) var(--ease);
}
.dsz:hover,.dsz:focus-visible{background:var(--acq);outline:none}
:root.dragging{cursor:col-resize;user-select:none}
:root.dragging *{cursor:col-resize!important}

/* ═══ شارات الإخفاء التلقائي ═══ */
.strip{
 flex:none; inline-size:27px; display:flex; flex-direction:column;
 gap:3px; padding:5px 0; align-items:center;
 background:var(--bg2); border-inline:1px solid var(--ln);
}
.strip button{
 display:flex; flex-direction:column; align-items:center; gap:3px;
 background:var(--bg3); color:var(--fg2);
 border:1px solid var(--ln); border-radius:var(--r1);
 padding:5px 2px; cursor:pointer; inline-size:21px;
}
.strip button span{
 writing-mode:vertical-rl; font-size:10.5px;
 max-block-size:120px; overflow:hidden; text-overflow:ellipsis;
 white-space:nowrap;
}
.strip button:hover{background:var(--hov);color:var(--fg)}

/* ═══ ترويسة اللوحة ═══ */
details.sec>summary{display:flex;align-items:center;gap:6px}
details.sec>summary::before{flex:none;color:var(--fg3)}
.pT{margin-inline-start:auto;display:inline-flex;gap:1px;flex:none}
.pB{
 background:transparent; color:var(--fg3);
 border:1px solid transparent; border-radius:var(--r1);
 padding:0 4px; cursor:pointer; font-size:12px; line-height:1.5;
}
.pB:hover{background:var(--hov);color:var(--fg)}
.pB.pX:hover{color:var(--er)}
details.sec>summary:not(:hover) .pB{opacity:.45}

/* ═══ وضع التبويبات ═══ */
.zTabs{
 position:sticky; inset-block-start:0; z-index:3;
 display:flex; flex-wrap:wrap; gap:2px;
 padding:5px 6px; background:var(--bg2);
 border-block-end:1px solid var(--ln);
}
.zTabs button{
 display:inline-flex; align-items:center; gap:4px;
 background:var(--bg3); color:var(--fg2);
 border:1px solid var(--ln); border-radius:var(--r1);
 padding:3px 7px; cursor:pointer; font-size:11.5px;
 max-inline-size:132px;
}
.zTabs button span{overflow:hidden;text-overflow:ellipsis;
 white-space:nowrap}
.zTabs button:hover{background:var(--hov);color:var(--fg)}
.zTabs button.on{background:var(--acq);color:var(--acf);
 border-color:var(--acb)}
.zTabs button:focus-visible{outline:2px solid var(--ac);
 outline-offset:-2px}
/* في التبويبات لا طيّ: اللوح الجاري مفتوحٌ دائماً */
details.sec.inTab>summary{cursor:default}
details.sec.inTab>summary::before{content:"";margin:0}

/* ═══ النوافذ العائمة ═══ */
#floats{position:fixed;inset:0;z-index:34;pointer-events:none}
.flt{
 position:fixed; pointer-events:auto;
 display:flex; flex-direction:column;
 background:var(--bg2); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge);
 overflow:hidden; resize:both; min-inline-size:220px;
 min-block-size:120px;
}
.flt.max{
 inset-inline:auto; inset-block:auto;
 inset-inline-start:8px; inset-block-start:44px;
 inline-size:calc(100vw - 16px); block-size:calc(100vh - 96px);
 resize:none;
}
.flt>details.sec{
 flex:1; display:flex; flex-direction:column;
 border-block-end:none; min-block-size:0;
}
.flt>details.sec>summary{
 flex:none; cursor:grab; background:var(--bg3);
 border-block-end:1px solid var(--ln);
}
:root.dragging .flt>details.sec>summary{cursor:grabbing}
.flt>details.sec>*:not(summary){overflow-y:auto}
.flt>details.sec{overflow:hidden}

/* ═══ القوائم المنبثقة ═══ */
#pMenu,#wsMenu{
 position:fixed; z-index:72; min-inline-size:206px;
 background:var(--bg3); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge);
 padding:4px 0; overflow:hidden;
}
.pmH{
 padding:5px 11px 6px; color:var(--wr); font-size:11.5px;
 border-block-end:1px solid var(--ln); margin-block-end:3px;
}
.pmI{
 display:flex; align-items:center; gap:8px; inline-size:100%;
 background:transparent; color:var(--fg2);
 border:none; padding:5px 11px; cursor:pointer; text-align:start;
 font-size:12px;
}
.pmI span:not(.ky){flex:1}
.pmI .ky{color:var(--ok);font-size:11px}
.pmI:hover{background:var(--hov);color:var(--fg)}
.pmI:disabled{opacity:.34;cursor:default}
.pmI:disabled:hover{background:transparent}
.pmI:focus-visible{outline:2px solid var(--ac);outline-offset:-2px}
.pmI.pmDel{
 margin-block-start:-30px; margin-inline-start:auto;
 inline-size:30px; padding:5px 0; justify-content:center;
 position:relative; color:var(--fg3);
}
.pmI.pmDel:hover{color:var(--er);background:transparent}
.pmS{height:1px;background:var(--ln);margin:4px 9px}

#pPark{display:none}

@media (max-width:1080px){
 #sideE{display:none}
}
```

<a id="f-css-helpbot-css"></a>

---

## `css/helpbot.css`

```css
/* ═══ روبوت الإرشاد — واجهة عصرية ═══ يرث ألوان theme.css */

/* الزرّ العائم */
.hb-fab{
 position:fixed; inset-inline-end:20px; bottom:20px; z-index:59;
 width:52px; height:52px; border-radius:50%; cursor:pointer;
 display:grid; place-items:center; color:var(--acf,#fff);
 background:linear-gradient(135deg,var(--ac),
   color-mix(in srgb,var(--ac) 70%,#000 8%));
 border:none; box-shadow:0 6px 18px rgba(0,0,0,.28);
 transition:transform .18s var(--ease,ease), box-shadow .18s;
}
.hb-fab:hover{transform:translateY(-2px) scale(1.05);
 box-shadow:0 10px 24px rgba(0,0,0,.34)}
.hb-fab:active{transform:scale(.96)}
.hb-fab.hb-open{transform:scale(.9); opacity:.85}
.hb-fab:focus-visible{outline:2px solid var(--ac); outline-offset:3px}

/* النافذة */
.helpbot{
 position:fixed; inset-inline-end:20px; bottom:84px; width:360px;
 max-width:calc(100vw - 32px); height:min(560px,72vh);
 display:flex; flex-direction:column; z-index:60; overflow:hidden;
 background:var(--bg2); color:var(--fg);
 border:1px solid var(--ln); border-radius:16px;
 box-shadow:0 16px 48px rgba(0,0,0,.32), var(--edge);
 font:13px/1.6 var(--ui);
 animation:hb-in .22s var(--ease,cubic-bezier(.2,.8,.2,1));
}
.helpbot[hidden]{display:none}
@keyframes hb-in{from{opacity:0; transform:translateY(12px) scale(.98)}
 to{opacity:1; transform:none}}

/* الرأس */
.hb-head{
 display:flex; align-items:center; justify-content:space-between;
 padding:12px 14px; flex:none;
 background:linear-gradient(180deg,
   color-mix(in srgb,var(--bg3) 60%,var(--bg2)),var(--bg2));
 border-bottom:1px solid var(--ln);
}
.hb-brand{display:flex; align-items:center; gap:10px}
.hb-logo{
 width:32px; height:32px; border-radius:9px; flex:none;
 display:grid; place-items:center; font-weight:700; font-size:16px;
 color:var(--acf,#fff);
 background:linear-gradient(135deg,var(--ac),
   color-mix(in srgb,var(--ac) 70%,#000 8%));
}
.hb-titles{display:flex; flex-direction:column; line-height:1.25}
.hb-title{color:var(--fg); font-weight:600; font-size:13.5px}
.hb-sub{display:flex; align-items:center; gap:5px;
 color:var(--fg3); font-size:11px}
.hb-dot{width:7px; height:7px; border-radius:50%;
 background:var(--fg3); transition:background .2s}
.hb-dot.on{background:var(--ok,#5cd98e);
 box-shadow:0 0 0 3px color-mix(in srgb,var(--ok,#5cd98e) 25%,transparent)}
.hb-actions{display:flex; gap:4px}
.hb-icon{
 width:30px; height:30px; border-radius:8px; cursor:pointer;
 display:grid; place-items:center;
 background:transparent; border:none; color:var(--fg3);
 transition:background .15s, color .15s;
}
.hb-icon:hover{background:var(--hov); color:var(--fg)}
.hb-icon:focus-visible{outline:2px solid var(--ac); outline-offset:1px}

/* سجلّ المحادثة */
.hb-log{flex:1; overflow-y:auto; padding:14px; display:flex;
 flex-direction:column; gap:12px; scroll-behavior:smooth}
.hb-log::-webkit-scrollbar{width:8px}
.hb-log::-webkit-scrollbar-thumb{background:var(--ln2,var(--ln));
 border-radius:8px}

.hb-row{display:flex; gap:8px; align-items:flex-end; max-width:100%}
.hb-row.hb-user{flex-direction:row-reverse}
.hb-avatar{
 width:26px; height:26px; border-radius:50%; flex:none;
 display:grid; place-items:center; font-size:10px; font-weight:600;
 background:var(--bg4); color:var(--fg3);
}
.hb-user .hb-avatar{background:var(--ac); color:var(--acf,#fff)}
.hb-msg{
 max-width:78%; padding:9px 12px; border-radius:14px;
 white-space:pre-line; word-break:break-word;
}
.hb-bot .hb-msg{background:var(--bg4); color:var(--fg);
 border-end-start-radius:4px}
.hb-user .hb-msg{background:var(--ac); color:var(--acf,#fff);
 border-end-end-radius:4px}
.hb-ans-text{white-space:pre-line}
.hb-warn{margin-top:8px; padding:6px 9px; border-radius:var(--r2);
 background:color-mix(in srgb,var(--wr) 15%,transparent);
 color:var(--wr); font-size:12px}

/* مؤشّر الكتابة */
.hb-typing .hb-msg{display:flex; gap:4px; padding:12px}
.hb-typing .hb-msg span{width:6px; height:6px; border-radius:50%;
 background:var(--fg3); animation:hb-bounce 1s infinite}
.hb-typing .hb-msg span:nth-child(2){animation-delay:.15s}
.hb-typing .hb-msg span:nth-child(3){animation-delay:.3s}
@keyframes hb-bounce{0%,60%,100%{opacity:.3; transform:translateY(0)}
 30%{opacity:1; transform:translateY(-4px)}}

/* أزرار الإجراء */
.hb-actbar{display:flex; flex-wrap:wrap; gap:6px; margin-top:10px}
.hb-act{
 padding:7px 13px; border:none; border-radius:9px; cursor:pointer;
 font:inherit; font-weight:500; color:var(--acf,#fff);
 background:linear-gradient(135deg,var(--ac),
   color-mix(in srgb,var(--ac) 72%,#000 6%));
 transition:filter .15s, transform .1s;
}
.hb-act:hover{filter:brightness(1.1)}
.hb-act:active{transform:scale(.97)}

/* روابط ذات صلة */
.hb-related{display:flex; flex-wrap:wrap; align-items:center; gap:6px;
 margin-top:10px; padding-top:8px;
 border-top:1px dashed var(--ln)}
.hb-related-lbl{font-size:11.5px; color:var(--fg3)}

/* الرقائق */
.hb-chips{display:flex; flex-wrap:wrap; gap:8px; padding:0 14px 12px;
 flex:none}
.hb-chips.hb-hidden{display:none}
.hb-chip{
 background:var(--bg3); color:var(--fg2);
 border:1px solid var(--ln2,var(--ln)); border-radius:20px;
 padding:7px 13px; font:inherit; font-size:12px; cursor:pointer;
 transition:background .15s, color .15s, border-color .15s,
   transform .1s;
}
.hb-chip:hover{background:var(--hov); color:var(--fg);
 border-color:var(--ac); transform:translateY(-1px)}
.hb-chip:active{transform:none}
.hb-chip.hb-sm{padding:4px 10px; font-size:11.5px; border-radius:14px}

/* نموذج الإدخال */
.hb-form{display:flex; gap:8px; padding:12px 14px; flex:none;
 border-top:1px solid var(--ln); background:var(--bg2)}
.hb-in{flex:1; background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd,var(--ln)); border-radius:22px;
 padding:9px 15px; font:inherit;
 transition:border-color .15s, box-shadow .15s}
.hb-in:focus{outline:none; border-color:var(--ac); box-shadow:var(--focus)}
.hb-send{
 width:40px; height:40px; flex:none; border-radius:50%; cursor:pointer;
 display:grid; place-items:center; color:var(--acf,#fff); border:none;
 background:linear-gradient(135deg,var(--ac),
   color-mix(in srgb,var(--ac) 72%,#000 6%));
 transition:filter .15s, transform .1s;
}
.hb-send:hover{filter:brightness(1.1)}
.hb-send:active{transform:scale(.94)}
.hb-send:focus-visible{outline:2px solid var(--ac); outline-offset:2px}

/* زرّ فتح المساعدة في الشريط (إن استُخدم بدل الـ FAB) */
.hb-open-btn{background:var(--bg4); color:var(--fg2);
 border:1px solid var(--ln2,var(--ln)); border-radius:8px;
 padding:5px 12px; cursor:pointer; font:inherit}
.hb-open-btn:hover{background:var(--hov); color:var(--fg);
 border-color:var(--ac)}
.hb-open-btn:focus-visible{outline:2px solid var(--ac); outline-offset:1px}

/* استجابة للشاشات الصغيرة */
@media (max-width:480px){
 .helpbot{inset-inline:12px; width:auto; bottom:80px;
  height:min(70vh,520px)}
 .hb-fab{inset-inline-end:14px; bottom:14px}
}
@media (prefers-reduced-motion:reduce){
 .helpbot{animation:none}
 .hb-fab,.hb-chip,.hb-act,.hb-send,.hb-typing .hb-msg span{
  transition:none!important; animation:none!important}
}

/* حوار استعادة ما بعد التعطّل */
.rc-overlay{position:fixed;inset:0;z-index:100;display:grid;
 place-items:center;background:rgba(0,0,0,.45)}
.rc-box{background:var(--bg2);color:var(--fg);border:1px solid var(--ln);
 border-radius:14px;padding:20px 22px;max-width:360px;width:90%;
 box-shadow:0 16px 48px rgba(0,0,0,.35)}
.rc-title{margin:0 0 8px;color:var(--fg);font-size:15px}
.rc-msg{margin:0 0 16px;color:var(--fg2);font-size:13px;line-height:1.6}
.rc-btns{display:flex;gap:8px;justify-content:flex-start}
.rc-yes{background:var(--ac);color:var(--acf,#fff);border:none;
 border-radius:8px;padding:8px 16px;cursor:pointer;font:inherit}
.rc-no{background:var(--bg4);color:var(--fg2);
 border:1px solid var(--ln);border-radius:8px;padding:8px 16px;
 cursor:pointer;font:inherit}
.rc-yes:hover{filter:brightness(1.1)}
.rc-no:hover{background:var(--hov);color:var(--fg)}

/* ═══ معرض القوالب والكتل الذكية ═══ */
.gal{
 position:fixed; inset-inline-end:20px; bottom:84px; width:380px;
 max-width:calc(100vw - 32px); max-height:72vh; display:flex;
 flex-direction:column; z-index:61; overflow:hidden; background:var(--bg2);
 color:var(--fg); border:1px solid var(--ln); border-radius:16px;
 box-shadow:0 16px 48px rgba(0,0,0,.32), var(--edge); font:13px/1.5 var(--ui);
 animation:hb-in .2s var(--ease,cubic-bezier(.2,.8,.2,1));
}
.gal[hidden]{display:none}
.gal-head{display:flex; align-items:center; justify-content:space-between;
 padding:10px 14px; border-bottom:1px solid var(--ln); background:var(--bg3)}
.gal-tabs{display:flex; gap:4px}
.gal-tab{background:transparent; border:none; color:var(--fg3);
 padding:6px 12px; border-radius:8px; cursor:pointer; font:inherit}
.gal-tab.is-on{background:var(--acq); color:var(--ac)}
.gal-tab:hover{color:var(--fg)}
.gal-close{background:none; border:none; color:var(--fg3); cursor:pointer;
 font-size:14px; padding:2px 6px; border-radius:6px}
.gal-close:hover{background:var(--hov); color:var(--fg)}
.gal-grid{flex:1; overflow-y:auto; padding:14px; display:grid;
 grid-template-columns:1fr 1fr; gap:10px}
.gal-card{display:flex; flex-direction:column; gap:4px; text-align:start;
 padding:12px; border-radius:12px; cursor:pointer; background:var(--bg3);
 border:1px solid var(--ln2,var(--ln));
 transition:border-color .15s, transform .1s, background .15s;
 font:inherit; color:inherit}
.gal-card:hover{border-color:var(--ac); transform:translateY(-2px); background:var(--hov)}
.gal-card:active{transform:none}
.gal-card:focus-visible{outline:2px solid var(--ac); outline-offset:1px}
.gal-card-title{font-weight:600; color:var(--fg)}
.gal-card-cat{font-size:11px; color:var(--ac)}
.gal-card-desc{font-size:11.5px; color:var(--fg3); line-height:1.6}
@media (max-width:480px){
 .gal{inset-inline:12px; width:auto; bottom:80px}
 .gal-grid{grid-template-columns:1fr}
}
@media (prefers-reduced-motion:reduce){
 .gal{animation:none}
 .gal-card{transition:none!important}
}
```

<a id="f-css-palette-css"></a>

---

## `css/palette.css`

```css
/* ═══ لوحة الأوامر (Ctrl+K) ═══ */
#palette{position:fixed;inset:0;z-index:90;
 background:color-mix(in srgb,#000 46%,transparent);
 backdrop-filter:blur(2px);
 display:flex;justify-content:center;align-items:flex-start}
#palette[hidden]{display:none}
#palette .pw{margin-top:12vh;width:min(560px,92vw);background:var(--bg3);
 border:1px solid var(--ln2);border-radius:var(--r4);
 box-shadow:var(--sh3),var(--edge);overflow:hidden}
#palette input{width:100%;box-sizing:border-box;border:0;
 border-bottom:1px solid var(--ln);
 background:transparent;color:var(--fg);font:inherit;font-size:15px;
 padding:13px 16px;outline:none}
#palette input::placeholder{color:var(--fg3)}
#palette .pl{max-height:46vh;overflow:auto;padding:4px 0}
#palette .pl::-webkit-scrollbar{width:9px}
#palette .pl::-webkit-scrollbar-thumb{background:var(--thumb);border-radius:5px}
#palette .it{display:flex;justify-content:space-between;align-items:center;gap:10px;
 padding:8px 16px;cursor:pointer;border-inline-start:2px solid transparent}
#palette .it:hover{background:var(--hov)}
#palette .it.sel{background:var(--acq);
 border-inline-start-color:var(--ac)}
#palette .it.sel .lb{color:var(--acf)}
#palette .it .lb{color:var(--fg)}
#palette .it .sb{color:var(--fg3);font-size:11.5px;
 font-family:var(--mono);direction:ltr}
#palette .it .dg{color:var(--wr);font-size:10.5px;font-weight:400}
#palette .nm{padding:16px;color:var(--fg3);font-size:12.5px;text-align:center}
#palette .ft{padding:8px 16px;border-top:1px solid var(--ln);
 color:var(--fg3);font-size:11px}
```

<a id="f-css-ribbon-css"></a>

---

## `css/ribbon.css`

```css
/* ═══ الشريط الرئيسي — التنسيق ═══
   أربعة أسطر تُبنى عليها كل الأزرار:
   .rbTab تبويب · .rbBig زرّ كبير · .rbSm زرّ صغير · .rbCol عمود
   والفاصل يُرسَم قبل كل عمودٍ صغير لا بين كل عنصرين. */

/* ─── زرّ التطبيق ─── */
#appBtn{
 position:relative; flex:none;
 display:inline-flex; align-items:center; gap:6px;
 background:linear-gradient(180deg,
  color-mix(in srgb,var(--bg3) 86%,var(--brand) 14%),var(--bg3));
 color:var(--brandf);
 border:1px solid var(--brandb); border-radius:var(--r3);
 padding:5px 14px; cursor:pointer; font-weight:600;
 letter-spacing:.3px; box-shadow:var(--sh1),var(--edge);
 transition:border-color .14s,background .14s,transform .14s;
}
#appBtn:hover{transform:translateY(-1px)}
#appBtn:hover{border-color:var(--brand);
 background:linear-gradient(180deg,
  color-mix(in srgb,var(--bg3) 76%,var(--brand) 24%),var(--bg3))}
#appBtn[aria-expanded="true"]{
 background:var(--brandq); color:var(--brandf);
 border-color:var(--brand); box-shadow:inset 0 2px 6px rgba(0,0,0,.35)}

/* ─── الوصول السريع ─── */
#qat{flex:none;display:inline-flex;align-items:center;gap:2px}
#qat button{
 display:inline-flex; align-items:center; justify-content:center;
 width:28px; height:26px;
 background:transparent; color:var(--fg2);
 border:1px solid transparent; border-radius:var(--r1);
 cursor:pointer; transition:background .12s,color .12s;
}
#qat button:hover{background:var(--hov);color:var(--fg)}
#qat button:focus-visible{outline:2px solid var(--ac);outline-offset:1px}
#qat button:disabled{opacity:.3;cursor:default}
#qat button:disabled:hover{background:transparent}
#qat .dv{flex:none;width:1px;height:15px;margin:0 3px;background:var(--ln)}

/* ─── قائمة التطبيق ─── */
#appMenu{
 position:fixed; inset-block-start:36px; z-index:70;
 inset-inline-start:8px; width:min(340px,94vw);
 background:var(--bg3); border:1px solid var(--ln2);
 border-radius:var(--r4); box-shadow:var(--sh3),var(--edge);
 max-height:80vh; display:flex; flex-direction:column;
 overflow:hidden;
}
:root[dir="rtl"] #appMenu{inset-inline-start:auto;inset-inline-end:8px}
.amHead{flex:none;padding:9px;border-bottom:1px solid var(--ln);
 position:relative}
#amSearch{
 width:100%; background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd); border-radius:var(--r2);
 padding:7px 9px; box-shadow:inset 0 1px 2px rgba(0,0,0,.22);
}
#amSearch:focus{outline:none;border-color:var(--ac);
 box-shadow:var(--focus)}
#amRes{
 position:absolute; inset-inline:9px;
 inset-block-start:calc(100% - 2px); z-index:2;
 background:var(--bg4); border:1px solid var(--ln2);
 border-block-start:none; border-radius:0 0 var(--r3) var(--r3);
 overflow:hidden; box-shadow:var(--sh2);
}
.amR{padding:6px 10px;cursor:pointer;display:flex;gap:8px;
 align-items:baseline}
.amR .mono{font-family:var(--mono);font-size:11px;color:var(--fg3)}
.amR:hover{background:var(--hov)}
.amR.sel{background:var(--acq);box-shadow:inset 2px 0 0 var(--ac)}
.amR.sel b{color:var(--acf)}
.amBody{flex:1;overflow-y:auto;padding:6px 0}
.amBody::-webkit-scrollbar{width:9px}
.amBody::-webkit-scrollbar-thumb{background:var(--thumb);
 border-radius:5px}
.amI{
 display:flex; align-items:center; gap:9px; width:100%;
 background:transparent; color:var(--fg2);
 border:none; padding:7px 13px; cursor:pointer; text-align:start;
}
.amI:hover{background:var(--hov);color:var(--fg)}
.amI:focus-visible{outline:2px solid var(--ac);outline-offset:-2px}
.amI .lb{flex:1}
.amI .ky{color:var(--fg3);font-size:11px;font-family:var(--mono)}
.amSep{height:1px;background:var(--ln);margin:6px 11px}

/* ─── هيكل الشريط ─── */
#ribbon{
 flex:none; display:flex; flex-direction:column;
 background:var(--bg2); border-block-end:1px solid var(--ln);
}

/* ─── صفّ التبويبات ─── يتصدّر أفقياً ولا ينضغط ─── */
#rbTabs{
 flex:none; display:flex; align-items:stretch; gap:2px;
 padding:0 8px; background:var(--bg2);
 border-block-end:1px solid var(--ln);
 overflow-x:auto; scrollbar-width:none;
}
#rbTabs::-webkit-scrollbar{display:none}
#rbTabs .gap{flex:1}
.rbTab{
 position:relative; flex:none;
 background:transparent; color:var(--fg2);
 border:1px solid transparent; border-block-end:none;
 border-radius:var(--r2) var(--r2) 0 0;
 padding:7px 16px 6px; cursor:pointer; white-space:nowrap;
 font-size:12.5px; letter-spacing:.2px; font-weight:500;
 transition:background .14s,color .14s;
}
.rbTab:hover{background:var(--hov);color:var(--fg)}
.rbTab.on{
 background:var(--bg3); color:var(--fg); font-weight:600;
 border-color:var(--ln); border-block-end-color:var(--bg3);
 margin-block-end:-1px;
}
.rbTab.on::after{
 content:""; position:absolute; inset-inline:0;
 inset-block-start:0; height:2px;
 background:var(--brand); border-radius:2px 2px 0 0;
}
.rbTab.ctx{color:var(--wr)}
.rbTab.ctx.on{color:var(--brandf);background:var(--brandq);
 border-color:var(--brandb);border-block-end-color:var(--brandq)}
.rbTab.ctx.on::after{background:var(--wr)}
.rbTab:focus-visible{outline:2px solid var(--ac);outline-offset:-2px}
.rbTab .kt{
 position:absolute; inset-block-start:-2px; inset-inline-end:1px;
 background:var(--wr); color:#1a1206;
 font-size:9.5px; line-height:1.35;
 padding:0 3px; border-radius:3px; font-weight:700;
}
.rbTgl{
 background:transparent; color:var(--fg3);
 border:1px solid transparent; border-radius:var(--r1);
 padding:2px 9px; cursor:pointer; align-self:center;
 transition:background .12s,color .12s;
}
.rbTgl:hover{background:var(--hov);color:var(--fg2)}

/* ─── اللوحات ─── */
#rbPanes{flex:none;background:var(--bg3);box-shadow:var(--edge)}
.rbPane{
 display:flex; align-items:stretch;
 overflow-x:auto; overflow-y:hidden;
 min-height:96px; scroll-behavior:smooth;
}
.rbPane::-webkit-scrollbar{height:8px}
.rbPane::-webkit-scrollbar-thumb{background:var(--thumb);
 border-radius:5px}
#ribbon.min #rbPanes{display:none}

.rbp{
 flex:none; display:flex; flex-direction:column;
 padding:7px 12px 0; margin:6px 3px 6px 0;
 border-inline-end:1px solid var(--ln);
}
.rbpBody{
 flex:1; display:flex; align-items:stretch; gap:5px;
 padding-block-end:4px;
}
/* فاصلٌ قبل كل عمودٍ صغير — والأول بلا فاصل */
.rbpBody > .rbCol{
 border-inline-start:1px solid var(--ln);
 padding-inline-start:5px;
}
.rbpBody > .rbCol:first-child{
 border-inline-start:none; padding-inline-start:0;
}
/* ذيل اللوح: التسمية في الوسط والمُفتتِح بطرف اللوح */
.rbpFoot{
 flex:none; position:relative;
 display:flex; align-items:center; justify-content:center;
 padding:3px 20px 5px;
 color:var(--fg3); font-size:10.5px; white-space:nowrap;
 letter-spacing:.35px; font-weight:500;
}
.rbDlg{
 position:absolute; inset-inline-end:2px; top:50%;
 transform:translateY(-50%);
 display:inline-flex; align-items:center; justify-content:center;
 width:16px; height:16px;
 background:transparent; color:var(--fg3);
 border:none; border-radius:var(--r1);
 padding:0; cursor:pointer; font-size:9px; line-height:1;
 transition:background .12s,color .12s;
}
.rbDlg:hover{background:var(--hov);color:var(--ac)}
.rbDlg:focus-visible{outline:2px solid var(--ac);outline-offset:0}

/* ─── الأزرار ─── */
.rbBig{
 display:flex; flex-direction:column; align-items:center;
 justify-content:flex-start; gap:5px;
 width:62px; padding:7px 3px 4px;
 background:transparent; color:var(--fg2);
 border:1px solid transparent; border-radius:var(--r2);
 cursor:pointer;
 transition:background .14s,color .14s,border-color .14s,transform .14s;
}
.rbBig .lb{font-size:11px;line-height:1.3;text-align:center;
 word-break:break-word}
.rbCol{display:flex;flex-direction:column;gap:2px;
 justify-content:flex-start}
.rbSm{
 display:flex; align-items:center; gap:6px;
 background:transparent; color:var(--fg2);
 border:1px solid transparent; border-radius:var(--r1);
 padding:3px 9px; cursor:pointer; white-space:nowrap;
 font-size:11.5px; min-height:23px;
 transition:background .14s,color .14s,border-color .14s;
}
.rbSm .lb{max-width:118px;overflow:hidden;text-overflow:ellipsis}

.rbBig:hover{background:var(--hov);color:var(--fg);transform:translateY(-1px)}
.rbSm:hover{background:var(--hov);color:var(--fg)}
.rbBig:active{transform:translateY(0);box-shadow:inset 0 1px 4px rgba(0,0,0,.18)}
.rbSm:active{box-shadow:inset 0 1px 4px rgba(0,0,0,.18)}
.rbBig.on,.rbSm.on{
 background:var(--acq); color:var(--acf); border-color:var(--acb);
 box-shadow:var(--edge);
}
.rbBig:disabled,.rbSm:disabled{opacity:.3;cursor:default}
.rbBig:disabled:hover,.rbSm:disabled:hover{background:transparent;
 box-shadow:none}
.rbBig:focus-visible,.rbSm:focus-visible{
 outline:2px solid var(--ac);outline-offset:-2px}
.noic{display:block;width:16px;height:16px}
.rbBig .noic{width:24px;height:24px}

:root.clean #optbar{display:none}

/* ─── الشاشات الضيّقة ─── */
@media (max-height:800px){
 .rbPane{min-height:88px}
 .rbBig{width:54px;padding:4px 2px 2px}
 .rbp{padding:5px 8px 0}
}
@media (max-width:1100px){
 .rbTab{padding:5px 10px;font-size:12px}
 .rbSm{padding:2px 6px}
 .rbSm .lb{max-width:84px}
}
@media (prefers-reduced-motion:reduce){
 #ribbon *,#qat button,#appBtn{transition:none!important}
}
```

<a id="f-css-status-css"></a>

---

## `css/status.css`

```css
/* ═══ شريط الحالة · التنقّل · التركيبات ═══ */
#status{flex-wrap:nowrap}
#stItems{
 flex:1; display:flex; align-items:center; gap:5px;
 min-inline-size:0; overflow:hidden;
}
#stItems>button{
 display:inline-flex; align-items:center; gap:4px; flex:none;
 background:transparent; color:var(--fg3);
 border:1px solid transparent; border-radius:999px;
 padding:3px 9px; cursor:pointer; white-space:nowrap;
 transition:background var(--t) var(--ease),color var(--t) var(--ease);
}
#stItems>button:hover{background:var(--hov);color:var(--fg2)}
#stItems>button.on{background:var(--acq);color:var(--acf);
 border-color:var(--acb)}
#stItems>button.bad{color:var(--er)}
#stItems>button.bad.on{background:color-mix(in srgb,var(--er) 18%,transparent);
 border-color:color-mix(in srgb,var(--er) 45%,transparent);color:var(--dng)}
#stItems>button:disabled{opacity:.34;cursor:default}
#stItems>button:focus-visible{outline:2px solid var(--ac);
 outline-offset:-2px}
#stItems>button.stPop{padding:2px 3px;margin-inline-start:-4px}
#stItems>button .lb{font-size:11.5px}
#stItems>button.wide .lb{min-inline-size:44px;text-align:start}

#stSel{color:var(--wr);flex:none;white-space:nowrap}
#stHint{flex:0 1 auto;min-inline-size:0;overflow:hidden;
 text-overflow:ellipsis;white-space:nowrap}
#stInfo{color:var(--ok);max-inline-size:26vw;flex:none}
#stMenu{
 position:fixed; z-index:74; min-inline-size:226px; max-block-size:78vh;
 overflow-y:auto;
 background:var(--bg3); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge); padding:4px 0;
}

/* ═══ شريط التنقّل ═══ داخل #stage المعزول ═══ */
#navbar{
 position:absolute; inset-inline-end:10px; inset-block-start:10px;
 z-index:8; display:flex; flex-direction:column; gap:2px;
 padding:4px; border-radius:var(--r3);
 background:color-mix(in srgb,var(--bg2) 85%,transparent);
 backdrop-filter:blur(6px);
 border:1px solid var(--ln2);
 box-shadow:var(--sh2),var(--edge);
}
#navbar button{
 display:flex; align-items:center; justify-content:center;
 inline-size:29px; block-size:27px;
 background:transparent; color:var(--fg2);
 border:1px solid transparent; border-radius:var(--r2); cursor:pointer;
}
#navbar button:hover{background:var(--hov);color:var(--fg)}
#navbar button.on{background:var(--acq);color:var(--acf);
 border-color:var(--acb)}
#navbar button:disabled{opacity:.3;cursor:default}
#navbar button:disabled:hover{background:transparent}
#navbar button:focus-visible{outline:2px solid var(--ac);
 outline-offset:-2px}
#vMenu{
 position:fixed; z-index:74; min-inline-size:234px;
 background:var(--bg3); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge); padding:4px 0;
}
.pmE{padding:6px 12px;color:var(--fg3);font-size:11.5px}

/* ═══ البوصلة ═══ */
#compass{
 position:absolute; inset-inline-end:9px; inset-block-end:38px;
 z-index:8;
}
#cmpBtn{
 background:color-mix(in srgb,var(--bg2) 74%,transparent);
 border:1px solid var(--ln2); border-radius:50%;
 padding:2px; cursor:pointer; display:block; line-height:0;
}
#cmpBtn:hover{border-color:var(--ac)}
#cmpBtn:focus-visible{outline:2px solid var(--ac);outline-offset:2px}
.cmpR{fill:none;stroke:var(--fg3);stroke-width:1.1;opacity:.65}
.cmpN{fill:var(--er);stroke:none}
.cmpT{stroke:var(--fg2);stroke-width:1.6;fill:none;
 stroke-linecap:round}
.cmpL{fill:var(--fg2);font:600 9px var(--ui)}
#cmpPop{
 position:absolute; inset-inline-end:0; inset-block-end:52px;
 inline-size:198px; z-index:9;
 background:var(--bg3); border:1px solid var(--ln2); border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge); padding:0 9px 8px;
}
.cmpRow{display:flex;align-items:center;gap:5px;margin:7px 0}
.cmpRow input{
 flex:1; background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd); border-radius:var(--r1); padding:4px 6px;
}
.cmpRow input:focus{outline:none;border-color:var(--ac);box-shadow:var(--focus)}
.cmpPre{display:flex;gap:3px}
.cmpPre button{
 flex:1; background:var(--bg2); color:var(--fg2);
 border:1px solid var(--ln); border-radius:var(--r1);
 padding:3px 0; cursor:pointer; font-size:11px;
}
.cmpPre button:hover{background:var(--hov);color:var(--fg)}

/* ═══ بطاقة المنظور ═══ */
#vpLabel{
 position:absolute; inset-inline-start:9px; inset-block-start:7px;
 z-index:8; display:flex; align-items:center; gap:4px;
 pointer-events:none;
}
.vpI{
 background:color-mix(in srgb,var(--bg2) 76%,transparent);
 color:var(--fg3); border:1px solid var(--ln2); border-radius:var(--r1);
 padding:1px 6px; font-size:11px; white-space:nowrap;
 pointer-events:auto;
}
button.vpI{cursor:pointer}
button.vpI:hover{color:var(--fg);border-color:var(--ac)}
#vpW{color:var(--wr)}
#vpW.bad{color:var(--er)}

@media (max-width:1080px){
 #stInfo{max-inline-size:16vw}
 #stItems>button .lb{display:none}
 #stItems>button.wide .lb{display:inline}
}
```

<a id="f-css-theme-css"></a>

---

## `css/theme.css`

```css
:root{
 color-scheme:dark;
 --bg:#0e1116; --bg2:#151a20; --bg3:#1c222a; --bg4:#0a0c10;
 --ln:#272e37; --ln2:#38414c;
 --fg:#edf1f5; --fg2:#aab4bf; --fg3:#76808c;
 --ac:#5b9dff; --acq:#17263c; --acb:#345885; --acf:#d3e6ff;
 --brand:#d1a85f; --brandq:#251f14; --brandb:#4c3e24; --brandf:#f3ddab;
 --ok:#5fc98a; --wr:#e6b768; --er:#ec746f; --dng:#ffabab;
 --hov:#232a33; --fldbd:#333e4a; --thumb:#333f4c;
 --r1:6px; --r2:9px; --r3:13px; --r4:18px;
 --sh1:0 1px 2px rgba(0,0,0,.38);
 --sh2:0 10px 24px -8px rgba(0,0,0,.5),0 2px 7px rgba(0,0,0,.3);
 --sh3:0 22px 54px -14px rgba(0,0,0,.62),0 5px 14px rgba(0,0,0,.38);
 --edge:inset 0 1px 0 rgba(255,255,255,.05);
 --focus:0 0 0 3px color-mix(in srgb,var(--ac) 45%,transparent);
 --ui:"Segoe UI","Noto Sans Arabic","IBM Plex Sans Arabic","Dubai",
      Tahoma,Arial,sans-serif;
 --mono:"Cascadia Mono","JetBrains Mono",Consolas,"Courier New",monospace;
 --t:140ms; --ease:cubic-bezier(.2,.7,.25,1);
}
:root[data-theme="light"]{
 color-scheme:light;
 --bg:#f5f6f9; --bg2:#ececf2; --bg3:#e2e4eb; --bg4:#ffffff;
 --ln:#d7dbe3; --ln2:#bac1cd;
 --fg:#161a20; --fg2:#4a525d; --fg3:#767f8b;
 --ac:#1c66d0; --acq:#e0ebfb; --acb:#a3c5f0; --acf:#0d4590;
 --brand:#8a6a20; --brandq:#f6ecd9; --brandb:#dac697; --brandf:#5c4713;
 --ok:#0f7a45; --wr:#8a5a00; --er:#b3261e; --dng:#b3261e;
 --hov:#e1e5ec; --fldbd:#c4cbd6; --thumb:#c6cdd8;
 --sh1:0 1px 2px rgba(16,24,40,.07);
 --sh2:0 10px 24px -8px rgba(16,24,40,.14),0 2px 7px rgba(16,24,40,.07);
 --sh3:0 22px 54px -14px rgba(16,24,40,.2),0 5px 14px rgba(16,24,40,.09);
 --edge:inset 0 1px 0 rgba(255,255,255,.9);
}
```

<a id="f-css-tools-css"></a>

---

## `css/tools.css`

```css
/* ═══ شريط الأدوات · شريط الخيارات · اللوحة الجانبية ═══ */
#tools{
 flex:none; display:flex; align-items:center; flex-wrap:wrap; gap:4px;
 padding:6px 12px; background:var(--bg2);
 border-bottom:1px solid var(--ln);
}
#tools .sp{width:10px}
#tools button{
 display:inline-flex; align-items:center; gap:5px;
 background:var(--bg3); color:var(--fg2);
 border:1px solid var(--ln); border-radius:var(--r2);
 padding:4px 11px; cursor:pointer; white-space:nowrap;
 transition:background var(--t) var(--ease),color var(--t) var(--ease),
  border-color var(--t) var(--ease);
}
#tools button.ico{padding:4px 8px}
#tools button:hover{background:var(--hov);color:var(--fg)}
#tools button.on{background:var(--acq);color:var(--acf);border-color:var(--acb)}
#tools button.del{color:var(--er)}
#tools button:disabled{opacity:.35;cursor:default}

#optbar{
 flex:none; display:flex; align-items:center; gap:9px; flex-wrap:wrap;
 padding:5px 10px; background:var(--bg2);
 border-bottom:1px solid var(--ln); min-height:34px;
}
#optbar .tl{color:var(--brand);font-size:12px;white-space:nowrap;font-weight:600}
#optbar .of{display:flex;align-items:center;gap:5px}
#optbar .of>span{color:var(--fg3);font-size:11.5px;white-space:nowrap}
#optbar input[type=text],#optbar input[type=number],#optbar select{
 background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd); border-radius:var(--r1); padding:3px 6px;
}
#optbar input[type=text],#optbar input[type=number]{width:64px}
#optbar input:focus,#optbar select:focus{
 outline:none;border-color:var(--ac);box-shadow:var(--focus)}
#optbar label.chk{
 display:inline-flex;align-items:center;gap:5px;color:var(--fg2);
 font-size:11.5px;cursor:pointer}
#optbar .hint{color:var(--fg3);font-size:11.5px}

#side{
 flex:none; width:312px; overflow-y:auto; overflow-x:hidden;
 background:var(--bg2); border-inline-end:1px solid var(--ln);
 padding:0 0 26px;
}
#side::-webkit-scrollbar,#log::-webkit-scrollbar{width:9px}
#side::-webkit-scrollbar-thumb,#log::-webkit-scrollbar-thumb{
 background:var(--thumb);border-radius:5px}
#side::-webkit-scrollbar-thumb:hover,#log::-webkit-scrollbar-thumb:hover{
 background:var(--ln2)}

details.sec{border-bottom:1px solid var(--ln)}
details.sec>summary{
 padding:8px 12px; cursor:pointer; color:var(--fg2);
 font-weight:600; font-size:12.5px; letter-spacing:.15px;
 background:var(--bg2);
 position:sticky; top:0; z-index:2; list-style:none;
}
details.sec>summary::-webkit-details-marker{display:none}
details.sec>summary::before{content:"▸ ";color:var(--fg3)}
details.sec[open]>summary::before{content:"▾ "}
details.sec>summary:hover{color:var(--fg)}
details.sec>*:not(summary){padding-inline:11px}
details.sec>*:not(summary):last-child{padding-bottom:9px}

details.sub{border:1px solid var(--ln);border-radius:var(--r1);
 margin:4px 0;background:var(--bg4)}
details.sub>summary{padding:4px 8px;cursor:pointer;
 color:var(--fg2);font-size:11.5px;list-style:none}
details.sub>summary::-webkit-details-marker{display:none}
details.sub>summary::before{content:"▸ ";color:var(--fg3)}
details.sub[open]>summary::before{content:"▾ "}
details.sub>div{padding:0 8px 6px}

.row{margin:5px 0}
.row2{display:flex;gap:7px;margin:5px 0}
.row2>.f{flex:1;min-width:0}
label{display:block;color:var(--fg3);font-size:11px;margin-bottom:2px}
#side input[type=text],#side input[type=number],#side select{
 width:100%; background:var(--bg4); color:var(--fg);
 border:1px solid var(--fldbd); border-radius:var(--r2); padding:5px 8px;
}
#side input:focus,#side select:focus{outline:none;border-color:var(--ac);
 box-shadow:var(--focus)}
#side input[type=color]{
 width:100%; height:24px; padding:1px 2px; cursor:pointer;
 background:var(--bg4); border:1px solid var(--fldbd); border-radius:var(--r1);
}
#side input[type=color]:focus{outline:none;border-color:var(--ac)}
#side input.num,#optbar input.num{
 direction:ltr;unicode-bidi:isolate;text-align:start}
label.chk{display:inline-flex;align-items:center;gap:5px;margin:0;
 color:var(--fg2);font-size:11.5px;cursor:pointer}
label.chk input{width:auto}
.btnrow{display:flex;gap:6px;flex-wrap:wrap;margin:7px 0}
#side button{
 background:var(--bg3); color:var(--fg2);
 border:1px solid var(--ln); border-radius:var(--r2);
 padding:5px 10px; cursor:pointer; flex:1; white-space:nowrap;
 transition:background var(--t) var(--ease),color var(--t) var(--ease);
}
#side button:hover{background:var(--hov);color:var(--fg)}
#side button.pri{background:var(--acq);color:var(--acf);border-color:var(--acb)}
#side button.del{color:var(--er)}
.hint{color:var(--fg3);font-size:11px;line-height:1.55;margin:6px 0}
.hint.warn{color:var(--wr)}
.phead{
 color:var(--brand); font-family:var(--mono); font-size:12px;
 padding:3px 0 5px; border-bottom:1px solid var(--ln); margin-bottom:5px;
}
.ro{
 display:block; padding:4px 6px; color:var(--fg2);
 background:var(--bg4); border:1px solid var(--ln); border-radius:var(--r1);
 font-family:var(--mono); font-size:11.5px;
}
@media (max-width:1080px){
 #side{width:262px}
 #clPrompt{min-width:150px}
}
```

<a id="f-css-tour-css"></a>

---

## `css/tour.css`

```css
/* ═══ الجولة التعريفية ═══ */
#tour{position:fixed;inset-inline-start:16px;bottom:82px;z-index:70;
 width:min(330px,88vw)}
#tour[hidden]{display:none}
#tour .tc{background:var(--bg3);border:1px solid var(--ln2);
 border-radius:var(--r3);
 box-shadow:var(--sh3),var(--edge);overflow:hidden}
#tour .th{display:flex;justify-content:space-between;align-items:center;gap:8px;
 padding:10px 13px;border-bottom:1px solid var(--ln);
 color:var(--brand);font-weight:600;font-size:12px}
#tour .tn{color:var(--fg3);font-size:11px}
#tour .tb{padding:12px 13px;color:var(--fg2);font-size:12.5px;line-height:1.75}
#tour .tb code{font-family:var(--mono);direction:ltr;display:inline-block;
 background:var(--bg4);color:var(--fg);border:1px solid var(--ln);
 padding:0 5px;border-radius:var(--r1)}
#tour .tf{display:flex;gap:7px;padding:9px 13px;border-top:1px solid var(--ln)}
#tour .tf button{background:var(--acq);color:var(--acf);
 border:1px solid var(--acb);border-radius:var(--r2);
 padding:5px 12px;cursor:pointer;font-size:12px}
#tour .tf button:hover{filter:brightness(1.1)}
#tour .tf .gh{background:transparent;color:var(--fg3);border-color:transparent}
#tour .tf .gh:hover{background:var(--hov);color:var(--fg2);filter:none}
```

---

# النواة — الهندسة والحالة (`js/core/`)

<a id="f-js-core-areas-js"></a>

---

