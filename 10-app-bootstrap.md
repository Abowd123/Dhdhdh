# CivilDraft — App Bootstrap

> نقطة تشغيل التطبيق والحارس الأولي.

**عدد الملفات:** 2

---

## `js/app.js`

```javascript
/* ═══ نقطة الدخول ═══
   يوصّل الطبقات، يستعيد آخر مشروع، ويربط المفاتيح.
   لا نظام سكربت: سطر الإدخال يقبل إحداثيات وأسماء أدوات فقط. */

import {SAFE,fatal,bootOk} from "./bootguard.js";

import "./tools/draw.js";
import "./tools/sketch.js";
import "./tools/openings.js";
import "./tools/parts.js";
import "./tools/areas.js";
import "./tools/modify.js";
import "./tools/annotate.js";
import "./tools/ref.js";
import "./tools/boq.js";
import "./tools/elev.js";
import "./tools/section.js";
import {installDefaults as installBlockDefaults,makeInstance}
 from "./core/blocks.js";
import {initBlockTool,isInserting,onKey as onBlockKey}
 from "./tools/blocks.js";
import {initBlockPanel,toggleBlockPanel}
 from "./ui/blockpanel.js";
import {installDefaults as installTemplateDefaults,apply as applyTemplate}
 from "./core/templates.js";
import {setImage as setUnderlayImage,onChange as onUnderlayChange}
 from "./core/underlay.js";
import {addWall} from "./core/walls.js";
import {loadCode} from "./core/code.js";
import {jrTaint} from "./core/journal.js";
import {snapsLoad,snapAutoStart} from "./io/snaps.js";
import {wirePalette,paletteToggle,paletteClose,paletteIsOpen} from "./ui/palette.js";
import {wireTour,tourMaybe} from "./ui/tour.js";
import {createHelpBot} from "./ui/helpbot.js";
import {mountHelpButton} from "./ui/helpbutton.js";
import {createAutosave} from "./core/persist.js";
import {maybeOfferRecovery} from "./ui/recoverdialog.js";

import {S,restore,setAfterEdit,setSaveError,setRefLost,saveNow,
        saveResume,saveMode,undo,redo,touch,autosave,ensureShape,
        edit,editFailed,pack,loadState} from "./core/state.js";
import {clamp,m2} from "./core/units.js";
import {osSummary,MODES} from "./core/osnap.js";
import {showAll} from "./core/layers.js";
import * as E from "./core/ents.js";
import * as R from "./tools/registry.js";
import {resize,draw,fit,cv,setShift,selectAll,delSel,setSel,
        selList,hitTest,UI,V} from "./ui/canvas.js";
import {buildTools,buildOptbar,syncOptbar,syncTools} from "./ui/optbar.js";
import {buildSide,loadForms,wireForms,refresh,renderProps,
        rep,eInfo,syncToggles} from "./ui/props.js";
import {wireInspector,runInspect,clearFindings} from "./ui/inspector.js";
import {wireDefaults,renderDefaults,syncDefaults} from "./ui/defaults.js";
import {wireAI} from "./ui/ai.js";
import {HOOK} from "./ui/bus.js";
import {loadUI,uiSet,UIS,setUiError} from "./ui/store.js";
import {mountIcons} from "./ui/icons.js";
import {buildRibbon,buildQAT,setTab,setMin,syncRibbon,
        syncRibbonTogs,invalidateSync} from "./ui/ribbon/render.js";
import {wireRibbon,setShell,setClean,setTheme,ribbonSel,
        autoFit,ktOn,revealSec,runSpec} from "./ui/ribbon/wire.js";
import {wireAppMenu,close as closeAppMenu,isOpen as appMenuOpen}
 from "./ui/appmenu.js";
import {initDock,wsApply,dockStats} from "./ui/dock.js";
import {wireStatus,syncStatus,buildStatus} from "./ui/statusbar.js";
import {wireNav,syncNav,runNav,viewSave} from "./ui/navbar.js";
import {wireOverlay,syncOverlay} from "./ui/overlay.js";
import {setTheme as setCanvasTheme,navSet,navMode} from "./ui/canvas.js";
import {wireCmd,applyCmd,cmdMenuClose} from "./ui/cmdline.js";
import {wireDyn,syncDyn,dynRoute,hideDyn} from "./ui/dyninput.js";
import {wireCtx,ctxOpen,ctxClose,ctxIsOpen} from "./ui/ctxmenu.js";
import {wireQuick,quickSel,qpToggle} from "./ui/quickprops.js";
import {initHistoryPanel,refreshHistoryPanel,toggleHistoryPanel,
        historyPanelOpen,closeHistoryPanel} from "./ui/historypanel.js";
import {initCmdPalette,toggleCmdPalette,closeCmdPalette,cmdPaletteOpen}
 from "./ui/cmdpalette.js";

const $=s=>document.querySelector(s);
const $$=s=>[...document.querySelectorAll(s)];
let phase1Autosave=null;   /* تُنشأ في build()، تُستعمل في boot() */

/* ═══ البناء ═══
   مخزن الواجهة أوّلاً: wirePanels يقرأ منه حالة الأقسام، والقشرة
   تُبنى بعد اللوحة الجانبية لأن الوكالة تنقر أزرارها.
   مرحلة البناء ملفوفةٌ بمصيدة: عطبٌ في أي نداءٍ هنا يُقال بسببه
   المرئيّ بدل أن يُسقط تقييم الوحدة كلّها فتبقى شاشةٌ بيضاء صامتة. */
function build(){
 const bm=document.getElementById("bootMsg");
 if(bm)bm.remove();

 loadUI(SAFE);
 loadCode();
 snapsLoad();
 mountIcons();

 /* ═══ الناقل ═══ أوّلاً — التوصيل ينادي report وprompt وtoggles،
    فلا يجوز أن تُملأ الخطّافات بعده. */
 HOOK.props=()=>{renderProps(); ribbonSel(); quickSel()};
 HOOK.refresh=r=>refresh(!!r);
 HOOK.status=m=>eInfo(m);
 HOOK.report=(c,m)=>rep(c,m);
 HOOK.prompt=()=>syncPrompt();
 HOOK.toggles=()=>{syncToggles();renderOsPop();syncRibbonTogs();
  syncStatus();syncNav();syncOverlay()};
 HOOK.help=()=>help();
 HOOK.defs=()=>syncDefaults();
 HOOK.ctx=(x,y)=>ctxOpen(x,y);
 /* سطح العمل يحمل القشرة والتبويب، وdock.js لا يعرفهما — فيُبلّغ */
 HOOK.ws=w=>{
  if(w.shell&&w.shell!==UIS.shell)setShell(w.shell);
  if(w.tab)setTab(w.tab);
  if((w.clean?1:0)!==(UIS.clean?1:0))setClean(w.clean?1:0);
  syncRibbonTogs();
 };
 HOOK.clean=v=>setClean(!!v);      /* مالكٌ واحد للشاشة النظيفة */

 R.H.draw=draw;
 R.H.rep=(c,m)=>rep(c,m);
 R.H.prompt=()=>syncPrompt();
 R.H.refresh=()=>refresh(false);
 R.H.hit=(x,y)=>hitTest(x,y);
 R.H.sel=()=>selList();
 R.H.setSel=l=>setSel(l||[],null);
 R.H.del=()=>delSel();

 /* ═══ نسخةٌ احتياطية ثانوية ═══ مخزنٌ منفصل (IndexedDB مختلفة عن
    io/store.js) كشبكة أمانٍ إضافية فقط — لا تحلّ محلّ نظام الجلسة
    الأساسي (S ← restore/autosave/saveNow أعلاه)، ولذا حوارُها لا
    يُعرَض إلّا حين يعجز الأساسي عن استعادة شيء (انظر boot() أدناه). */
 phase1Autosave=createAutosave({
  serialize:()=>pack(),
  restore:d=>{ loadState(d,true); refresh(true); },
  meta:()=>({name:S.meta&&S.meta.name}),
  interval:8000
 });
 phase1Autosave.start();

 setAfterEdit(reload=>{
  try{refresh(!!reload)}
  catch(e){rep("er","تحديث الواجهة: "+e.message)}
  refreshHistoryPanel();
  phase1Autosave.markDirty();
 });
 setSaveError(onSaveFail);          /* دالّةٌ مُعرَّفة أدناه */
 /* المرجع خارج التاريخ: نسخةٌ تجاوزت الحدّ فزالت — يُقال ولا
    تُخترَع كياناتٌ، والرسم سليم */
 setRefLost(()=>rep("wr","المرجع المستورد لم يعد في متناول "
  +"التراجع — أعِد استيراده إن احتجتَه. رسمك سليم."));
 /* تفضيلات الواجهة: الفشل يُقال لحظةَ وقوعه لا في الإقلاع التالي */
 setUiError(n=>rep("wr","تعذّر حفظ تفضيلات الواجهة"
  +((n==="QuotaExceededError")?" — التخزين ممتلئ":(n?` — ${n}`:""))
  +" · تخطيط اللوحات والمناظر لن يبقى بعد إغلاق الصفحة. "
  +"امسح ما لا تحتاجه من «CivilDraft ← امسح كل ما هو محفوظ محلّياً»."));

 /* ═══ التوصيل ═══ */
 setCanvasTheme(UIS.theme);   /* يضبط data-theme ويُبطل كاش النقوش */
 document.documentElement.dataset.shell=UIS.shell;
 buildTools();
 buildSide();
 wireForms();
 wireInspector();
 R.loadOpts();
 wireDefaults();
 initDock();      /* بعد buildSide: يحصد الأقسام ويوزّعها بالتخطيط */
 wireAI();
 buildOptbar();
 buildQAT();
 wireAppMenu();
 wireRibbon();
 wireStatus();
 wireNav();
 wireOverlay();
 wireCmd();
 wireDyn();
 wireCtx();
 wireQuick();
  wirePalette();
  wireTour();

 /* ═══ روبوت الإرشاد ═══ إضافيٌّ ومستقلٌّ عن لوحة المرجع الثابتة
    (#helpBox / F1 / #bHelp أعلاه) — زرٌّ منفصل في الشريط يفتح
    دردشةً تشرح الأدوات وتُفعِّلها. R.begin يقبل id الأداة مباشرةً،
    فلا حاجة لتخمين اسم دالّة الدخول. */
 const helpBot=createHelpBot({activate:toolId=>R.begin(toolId)});
 mountHelpButton({onClick:()=>helpBot.toggle()});

 initHistoryPanel();
 installBlockDefaults();
 installTemplateDefaults();
 initBlockPanel();
 initBlockTool({
  addInstance:inst=>{(S.blocks||(S.blocks=[])).push(inst)},
  redraw:()=>draw(),
  snap:w=>[Math.round(w[0]),Math.round(w[1])],
  defaultLayer:"0"
 });
 onUnderlayChange(()=>draw());
 initCmdPalette({extra:[
  {id:"undo",label:"تراجع",aliases:["undo","u"],
   hint:"Ctrl+Z", run:()=>{rep(undo()?"in":"wr","تراجع");}},
  {id:"redo",label:"إعادة",aliases:["redo"],
   hint:"Ctrl+Shift+Z", run:()=>{rep(redo()?"in":"wr","إعادة");}},
  {id:"save",label:"حفظ",aliases:["save","حفظ ملف"],
   hint:"Ctrl+S", run:()=>{const b=$("#xSave"); if(b)b.click();}},
  {id:"history",label:"لوحة السجل",aliases:["history","سجل"],
   hint:"Ctrl+Shift+H", run:()=>toggleHistoryPanel()},
  {id:"clean",label:"شاشة نظيفة",aliases:["clean screen","نظافة"],
   hint:"Ctrl+0", run:()=>setClean(!UIS.clean)},
  {id:"theme",label:"تبديل السمة",aliases:["theme","سمة داكنة فاتحة"],
   hint:"Ctrl+Shift+T", run:()=>
    setTheme(UIS.theme==="dark"?"light":"dark")},
  {id:"fit",label:"ملاءمة العرض",aliases:["fit","zoom fit"],
    hint:"", run:()=>{fit(); eInfo("مُلوئم");}},
   {id:"blocks",label:"لوحة العناصر",aliases:["blocks","عناصر"],
    hint:"B",run:()=>toggleBlockPanel()},
   {id:"template-room",label:"قالب غرفة مستطيلة",aliases:["room","قالب غرفة"],
    hint:"",run:()=>useTemplate("room")},
   {id:"template-studio",label:"قالب استوديو",aliases:["studio","قالب استوديو"],
    hint:"",run:()=>useTemplate("studio")},
   {id:"template-office",label:"قالب مكتب",aliases:["office","قالب مكتب"],
    hint:"",run:()=>useTemplate("office")},
   {id:"underlay",label:"تحميل صورة مرجعية",aliases:["underlay","مرجع صورة"],
    hint:"",run:()=>chooseUnderlay()}
 ]});
 /* القشرة آخراً */
 setShell(UIS.shell);
 if(UIS.clean)setClean(1);
 autoFit();
}
function useTemplate(name){
 const r=edit(()=>applyTemplate(name,{
  addWall:w=>addWall(w.a,w.b,w.th,"int","c"),
  addBlock:b=>(S.blocks||(S.blocks=[])).push(makeInstance(b.block,b)),
  setMeta:m=>{if(m.scale)S.meta.scale=m.scale}
 }),"تطبيق قالب "+name);
 if(!editFailed()&&r){rep("ok","طُبّق القالب");draw();refresh(false)}
}
function chooseUnderlay(){
 const i=document.createElement("input"); i.type="file"; i.accept="image/*";
 i.onchange=()=>{
  const f=i.files&&i.files[0]; if(!f)return;
  const r=new FileReader();
  r.onload=()=>{setUnderlayImage(String(r.result||""));eInfo("حُمّلت الصورة المرجعية")};
  r.readAsDataURL(f);
 };
 i.click();
}
try{build()}
catch(e){
 fatal((e&&e.stack)?String(e.stack).split("\n").slice(0,4).join("\n")
  :String(e),"بناء الواجهة");
}
/* الحفظ التلقائي يُبلّغ عن فشله مرّةً — الصمت هو ما كان يفقد
   المستخدمَ جلستَه بلا كلمة */
let saveWarned=false;
function onSaveFail(r){
 if(saveWarned)return;
 saveWarned=true;
 rep("er","تعذّر الحفظ التلقائي"
  +((r.err==="QuotaExceededError")?" — التخزين ممتلئ":
    (r.err?` — ${r.err}`:""))
  +(r.refs?` · المرجع ${r.refs} كياناً`:"")
  +" · احفظ المشروع ملفّاً (Ctrl+S)، ورسمُك سليم.");
}
/* ═══ سطر الإدخال ═══ */
const cl=$("#clIn"), clP=$("#clPrompt"), clL=$("#clLive"),
      clSug=$("#clSug");
let hist=-1;
export function syncPrompt(){
 const q=R.promptText();
 const p=(q.tool?q.tool+" · ":"")+q.p;
 if(clP.textContent!==p)clP.textContent=p;
 const lv=q.live||"";
 if(clL.textContent!==lv)clL.textContent=lv;
 const hn=$("#stHint");
 const h=R.active()
  ? (R.T.def.hint||"")+" · Esc يلغي"
  : "لا أداة نشطة · انقر لتحديد · اسحب إطاراً على الفراغ";
 if(hn&&hn.textContent!==h)hn.textContent=h;
 syncTools();
 syncRibbon();
 syncDyn();
 syncOptbar();   /* لا buildOptbar: تُنادى مع كل حركة مؤشّر */
}
/* ═══ اقتراح الأدوات ═══
   قائمة بسيطة من أسماء الأدوات المسجَّلة تُبنى أثناء الكتابة،
   تُدوَّر بـ Tab وتُنفَّذ بالنقر أو Enter (حين تكون مُبرَزة فعلاً). */
let sugList=[], sugIdx=-1, sugNav=false;
function sugHide(){sugList=[];sugIdx=-1;sugNav=false;if(clSug){clSug.hidden=true;clSug.innerHTML=""}}
function sugRender(){
 if(!clSug)return;
 clSug.innerHTML=sugList.map((k,i)=>
  `<div class="it${i===sugIdx?" sel":""}" data-i="${i}">${esc(k)}</div>`
 ).join("");
}
function sugBuild(){
 if(!clSug)return;
 sugNav=false;
 const v=cl.value.trim().toLowerCase();
 if(!v||R.active()){sugHide();return}
 sugList=[...new Set(Object.keys(R.TOOLS))]
  .filter(k=>k.startsWith(v)).sort().slice(0,8);
 if(!sugList.length){sugHide();return}
 sugIdx=0;
 sugRender();
 clSug.hidden=false;
}
function sugRotate(dir){
 if(!sugList.length)return;
 sugNav=true;
 sugIdx=(sugIdx+dir+sugList.length)%sugList.length;
 sugRender();
}
function sugCommit(v){
 cl.value=""; hist=-1; sugHide();
 if(R.active()){
  if(v)R.feedText(v); else R.enter();
 }else if(v){
  /* ؟ يُحوّل السطر إلى المساعد — بادئةٌ صريحة فلا يُلتبَس
     بأسماء الأدوات ولا يقع طلبٌ شبكيّ صامت */
  if(/^[?؟]/.test(v)){
   const q=v.replace(/^[?؟]\s*/,"").trim();
   if(!q){
    rep("in","اكتب سؤالك بعد ؟ — مثل: ؟ ارسم غرفة 4×5");
    syncPrompt(); return;
   }
   R.histAdd(v);
   import("./ui/ai.js").then(A=>A.askFromLine(q));
   syncPrompt(); return;
  }
  R.histAdd(v);
  let d=R.findTool(v), arg=null;
  if(!d){
   const i=v.search(/\s/);
   if(i>0){
    const t=R.findTool(v.slice(0,i));
    if(t&&t.arg){d=t; arg=v.slice(i+1).trim()}
   }
  }
  if(d)R.begin(d,arg);
  else rep("er",`لا أداة بهذا الاسم: ${v} — F1 للقائمة`);
 }else if(R.T.last)R.begin(R.T.last,R.T.lastArg);
 syncPrompt(); draw();
}
if(clSug)clSug.addEventListener("mousedown",e=>{
 const it=e.target.closest("[data-i]");
 if(!it)return;
 e.preventDefault();
 sugCommit(sugList[+it.dataset.i]);
});
cl.addEventListener("keydown",e=>{
 const k=e.key, ck=e.ctrlKey||e.metaKey;
 if(k==="Escape"){
  e.preventDefault();
  if(sugList.length)sugHide();
  else if(cl.value)cl.value="";
  else if(R.active())R.cancel();
  else setSel([],null);
  syncPrompt(); draw(); return;
 }
 if(k==="ArrowUp"||k==="ArrowDown"){
  if(!cl.value&&selList().length)return;  /* اترك الأمر يصعد للنُدج */
  if(!R.T.hist.length)return;
  e.preventDefault();
  if(hist<0)hist=R.T.hist.length;
  hist=clamp(hist+(k==="ArrowUp"?-1:1),0,R.T.hist.length);
  cl.value=(hist<R.T.hist.length)?R.T.hist[hist]:"";
  sugHide();
  return;
 }
 if((k==="ArrowLeft"||k==="ArrowRight")&&!cl.value)return;
 /* المسافة تُنفّذ السطر (كأوتوكاد)، إلّا في سطرٍ يحتمل المسافات:
    سؤالُ المساعد (؟) وقيمةٌ نصّية تنتظرها خطوةٌ حرّة. */
 const argTool=()=>{
  if(R.active())return false;
  const h=cl.value.trim().split(/\s+/)[0];
  const d=h?R.findTool(h):null;
  return !!(d&&d.arg);
 };
 const freeText=/^[?؟]/.test(cl.value)
  ||(R.active()&&R.step()&&R.step().text)
  ||argTool();
 if(k==="Enter"||(k===" "&&!freeText&&!cl.value.includes(" "))){
  e.preventDefault();
  if(sugNav&&sugList.length){sugCommit(sugList[sugIdx]);return}
  const v=cl.value.trim();
  sugCommit(v);
  return;
 }
 if(k==="F1"){e.preventDefault();help();return}
 if(k==="Tab"){
  if(!sugList.length)return;
  e.preventDefault();
  sugRotate(e.shiftKey?-1:1);
  return;
 }
 if(ck||/^F\d+$/.test(k))return;   /* لا نوقف: تصعد إلى المعالج العامّ */
 if(!cl.value&&(k==="Delete"||k==="Backspace"))return;
 e.stopPropagation();
});
cl.addEventListener("input",()=>{sugBuild()});
cl.addEventListener("blur",()=>sugHide());
/* ═══ الأدوات والأزرار ═══ */
$("#tools").addEventListener("click",e=>{
 const b=e.target.closest("button");
 if(!b)return;
 if(b.id==="bUndo"){
  rep(undo()?"in":"wr","تراجع"); return;
 }
 if(b.id==="bRedo"){
  rep(redo()?"in":"wr","إعادة"); return;
 }
 if(b.id==="bFit"){fit();eInfo("مُلوئم");return}
 if(b.id==="bHelp"){help();return}
 if(b.id==="bLall"){
  const n=edit(()=>showAll(),"إظهار كل الطبقات");
  if(editFailed())return;
  rep(n?"ok":"in",n?`أُظهرت ${n} طبقة`:"كل الطبقات ظاهرة");
  refresh(false); return;
 }
 const cmd=b.dataset.cmd;
 if(cmd===undefined)return;
 if(cmd==="@insp"){runInspect();return}
 if(!cmd)R.cancel(true); else R.begin(cmd);
 syncPrompt(); draw(); cl.focus();
});
/* ═══ مفاتيح الحالة ولوحة الالتقاط ═══ */
function rbToggle(k){
 S.rb[k]=S.rb[k]?0:1;
 if(k==="ortho"&&S.rb.ortho)S.rb.polar=0;
 if(k==="polar"&&S.rb.polar)S.rb.ortho=0;
 touch(); syncToggles(); autosave(); draw();
 if(k==="dyn"&&!S.rb.dyn)hideDyn();
 const AR={snap:"التقاط الكائنات",ortho:"التعامد",
  polar:"التتبّع القطبي",grips:"المقابض",ends:"علامات الأطراف",
  grid:"الشبكة",gsnap:"الالتقاط على الخطوة",
  paths:"مسارات الجدران",dyn:"الإدخال الحركي"};
 eInfo(`${AR[k]||k}: ${S.rb[k]?"مُشغّل":"مُوقف"}`
  +(k==="snap"&&S.rb[k]?" · "+osSummary():"")
  +(k==="polar"&&S.rb[k]?` · كل ${S.pol.inc}°`:"")
  +(k==="gsnap"&&S.rb[k]?` · كل ${m2(S.meta.snap)} م`:""));
}
/* ═══ التفويض بدل الربط المباشر ═══
   الشريط يُبنى ويُعاد بناؤه بالتخصيص، فالربط المباشر يتوقّف صامتاً
   بعد أول إعادة. التفويض لا يتوقّف. */
$("#status").addEventListener("click",e=>{
 /* زرّا الالتقاط والقطبي يفتحان اللوح نفسه — فيه أنماطُه وزاويته */
 const ob=e.target.closest("#osBtn,#polBtn");
 if(ob){
  pop.hidden=!pop.hidden;
  renderOsPop();
  placeOsPop(ob);
  if(!pop.hidden&&ob.id==="polBtn"){
   const i=$("#polInc");
   if(i){i.focus(); i.select()}
  }
  return;
 }
 const b=e.target.closest("[data-rb]");
 if(b){rbToggle(b.dataset.rb); return}
 const a=e.target.closest("[data-act]");
 if(!a)return;
 if(a.dataset.act==="wsMenu")a.dataset.wsx="1";
 runSpec({act:a.dataset.act});      /* مُنفِّذٌ واحد — يُبلِّغ المجهول */
});
/* بطاقة المنظور تشترك في الأفعال نفسها */
$("#stage").addEventListener("click",e=>{
 const a=e.target.closest("[data-act]");
 if(a)runSpec({act:a.dataset.act});
});

const pop=$("#osPop");
function placeOsPop(btn){
 if(pop.hidden||!btn)return;
 const r=btn.getBoundingClientRect();
 const rtl=getComputedStyle(document.documentElement)
  .direction==="rtl";
 const w=pop.offsetWidth||190;
 const x=rtl?(innerWidth-r.right):r.left;
 pop.style.insetInlineStart=Math.round(
  clamp(x,4,Math.max(4,innerWidth-w-4)))+"px";
}
function renderOsPop(){
 if(pop.hidden)return;
 pop.innerHTML=`<h5>أنماط الالتقاط</h5>`
  +MODES.map(m=>`<label><input type="checkbox" data-os="${m.k}"`
   +`${+S.os[m.k]?" checked":""}> ${m.n}</label>`).join("")
  +`<div class="pi">زاوية القطبي <input type="number" id="polInc"
    class="num" min="1" max="90" step="1"
    value="${clamp(parseInt(S.pol.inc,10)||15,1,90)}"> °</div>
   <div class="fr"><button data-osa="all">الكل</button>
    <button data-osa="none">لا شيء</button></div>`;
}
pop.addEventListener("change",e=>{
 const t=e.target;
 if(t.dataset.os){
  S.os[t.dataset.os]=t.checked?1:0;
  touch(); autosave(); draw();
  eInfo("الالتقاط: "+osSummary());
  return;
 }
 if(t.id==="polInc"){
  S.pol.inc=clamp(parseInt(t.value,10)||15,1,90);
  touch(); autosave(); draw();      /* الخطوط تتبع الزاوية */
  eInfo(`التتبّع القطبي كل ${S.pol.inc}°`);
 }
});
pop.addEventListener("click",e=>{
 const a=e.target.dataset.osa;
 if(!a)return;
 MODES.forEach(m=>{S.os[m.k]=(a==="all")?1:0});
 touch(); renderOsPop(); autosave(); draw();
 eInfo("الالتقاط: "+osSummary());
});
addEventListener("mousedown",e=>{
 if(pop.hidden)return;
 if(!pop.contains(e.target)&&!e.target.closest("#osBtn,#polBtn"))
  pop.hidden=true;
},true);

/* ═══ المفاتيح العامّة ═══ */
const ARR={ArrowLeft:[-1,0],ArrowRight:[1,0],
 ArrowUp:[0,1],ArrowDown:[0,-1]};
function nudge(dx,dy){
 edit(()=>selList().forEach(s=>{
  const o=E.grabOf(s);
  if(o)E.moveEnt(s,o,dx,dy);
 }),"تحريك بالمفاتيح");
}
addEventListener("keydown",e=>{
 setShift(e.shiftKey);
 const tg=(e.target.tagName||"").toLowerCase();
 const inCl=(e.target===cl);
 const typing=!inCl&&(tg==="input"||tg==="select"||tg==="textarea");
 const ck=e.ctrlKey||e.metaKey;
 const k=(e.key||"").toLowerCase();

 if(e.key==="Escape"){
  const hb=$("#helpBox");
  if(hb&&!hb.hidden){e.preventDefault();hb.hidden=true;return}
 }
 if(e.key==="Escape"&&cmdPaletteOpen()){
  e.preventDefault(); closeCmdPalette(); return;
 }
 if(e.key==="Escape"&&historyPanelOpen()){
  e.preventDefault(); closeHistoryPanel(); return;
 }
 if(e.key==="Escape"&&appMenuOpen()){
  e.preventDefault(); closeAppMenu(); return;
 }
 if(e.key==="Escape"&&navMode()){
  e.preventDefault(); navSet(null); eInfo(""); return;
 }
 if(e.key==="Escape"&&ctxIsOpen()){
  e.preventDefault(); ctxClose(); return;
 }
  if(isInserting()&&onBlockKey(e)){e.preventDefault();return}
 if(ck&&e.key==="F1"){e.preventDefault();
  setMin(!UIS.ribbonMin); dispatchEvent(new Event("resize")); return}
 /* لكلٍّ بديلٌ لا يحتجزه المتصفّح، والأصل يبقى لمن يعمل عنده */
 if((ck&&e.key==="0")||(ck&&e.shiftKey&&k==="c")){e.preventDefault();
  setClean(!UIS.clean); return}
 if(ck&&e.shiftKey&&(k==="t"||k==="m")){e.preventDefault();
  setTheme(UIS.theme==="dark"?"light":"dark"); return}
 if(e.key==="F1"){e.preventDefault();help();return}
 if(e.key==="F3"){e.preventDefault();rbToggle("snap");return}
 if(e.key==="F7"){e.preventDefault();runInspect();return}
 if(e.key==="F8"){e.preventDefault();rbToggle("ortho");return}
 if(e.key==="F10"){e.preventDefault();rbToggle("polar");return}
 if(e.key==="F11"){e.preventDefault();rbToggle("grips");return}
 if(e.key==="F12"||(ck&&e.shiftKey&&k==="e")){
  e.preventDefault(); rbToggle("ends"); return;
 }
 if(e.key==="F6"){e.preventDefault();rbToggle("grid");return}
 if(e.key==="F9"){e.preventDefault();rbToggle("gsnap");return}
 if(ck&&e.shiftKey&&k==="l"){
  e.preventDefault();
  const n=edit(()=>showAll(),"إظهار كل الطبقات");
  if(editFailed())return;
  rep(n?"ok":"in",n?`أُظهرت ${n} طبقة`:"كل الطبقات ظاهرة");
  refresh(false); return;
 }
 if(ck&&k==="d"&&!e.shiftKey){e.preventDefault();
  rbToggle("dyn"); return}
 if(ck&&e.shiftKey&&k==="q"){e.preventDefault(); qpToggle(); return}
  if(ck&&(k==="k"||(e.shiftKey&&k==="p"))){
   e.preventDefault(); paletteToggle(); return;
  }
  if(e.key==="Escape"&&paletteIsOpen()){
   e.preventDefault(); paletteClose(); return;
  }
 if(ck&&e.shiftKey&&k==="h"){e.preventDefault(); toggleHistoryPanel(); return}
 if(ck&&k==="z"){
  e.preventDefault();
  const ok=e.shiftKey?redo():undo();
   if(ok)jrTaint(e.shiftKey?"إعادة":"تراجع");
  eInfo(ok?(e.shiftKey?"إعادة":"تراجع"):"لا شيء");
  return;
 }
 if(inCl){
  const passS=ck&&k==="s";
  const passEmpty=!cl.value
   &&(e.key==="Delete"||e.key==="Backspace"||ARR[e.key]);
  if(!passS&&!passEmpty)return;
 }
 if(typing){if(e.key==="Escape")e.target.blur();return}
  if(!typing&&!inCl&&!ck&&!e.altKey&&k==="b"){
   e.preventDefault();toggleBlockPanel();return;
  }
 if(ck&&k==="a"){e.preventDefault();eInfo(`${selectAll()} محدد`);return}
 if(ck&&k==="s"){
  e.preventDefault();
  const b=$("#xSave");
  if(b)b.click();
  return;
 }
 if(e.key==="Escape"){
  hideDyn();
  if(R.active())R.cancel(); else setSel([],null);
  syncPrompt(); draw(); return;
 }
 if(e.key==="Delete"||e.key==="Backspace"){
  if(!R.active()){
   e.preventDefault();
   const r=delSel();
   if(r)eInfo("حُذف "+E.delSay(r)
    +(r.skipped?` · تُخطّي ${r.skipped}`:""));
  }
  return;
 }
 if(e.key==="Enter"||e.key===" "){
  e.preventDefault();
  if(R.active())R.enter();
  else if(R.T.last)R.begin(R.T.last,R.T.lastArg);
  syncPrompt(); draw(); return;
 }
 if(ARR[e.key]&&!R.active()&&selList().length){
  e.preventDefault();
  const st=Math.max(1,S.meta.snap)*(e.shiftKey?10:1);
  nudge(ARR[e.key][0]*st, ARR[e.key][1]*st);
  return;
 }
  /* أي حرف مطبوع يذهب إلى سطر الإدخال — كما في أوتوكاد:
    لا اختصار حرفٍ مفردٍ يسرق المفتاح. والأرقام تذهب إلى حقول
    الإدخال الحركي إن كانت ظاهرة: قاعدةٌ واحدة تُفسَّر — تلك
    حقولُ قيَمٍ، وهذا سطرُ صيَغٍ وأسماء أدوات. */
 if(!ck&&!e.altKey&&e.key.length===1){
  if(dynRoute(e.key)){e.preventDefault(); return}
  cl.focus();
  return;
 }
});
addEventListener("keyup",e=>setShift(e.shiftKey));

/* ═══ المساعدة ═══
   تُبنى من السجلّ نفسه، فلا تتخلّف عن الأدوات. */
function help(){
 const B=$("#helpBox");
 if(!B)return;
 if(!B.hidden){B.hidden=true;return}
 const rows=R.toolList()
  .filter(d=>d&&d.id)
  .sort((a,b)=>a.label.localeCompare(b.label,"ar"))
  .map(d=>{
   const al=Object.keys(R.TOOLS)
    .filter(k=>R.TOOLS[k]===d&&k!==d.id)
    .slice(0,4).join(" · ");
   return `<tr><td>${esc(d.label)}</td>`
    +`<td class="mono">${esc(d.id)}${al?" · "+esc(al):""}</td>`
    +`<td>${esc(d.hint||"")}</td></tr>`;
  }).join("");
 B.innerHTML=`
<div class="hd"><b>CivilDraft — المساعدة</b>
 <button id="helpX">إغلاق</button></div>
<div class="bd">
 <h4>الإدخال</h4>
 <ul>
  <li><b>الإحداثي المطلق</b> <code>3,4</code> — بالمتر من الأصل</li>
  <li><b>النسبي</b> <code>@5,0</code> — من النقطة السابقة</li>
  <li><b>القطبي</b> <code>@5&lt;45</code> — طول وزاوية</li>
  <li><b>الطول وحده</b> <code>5</code> — على اتجاه المؤشّر الجاري</li>
  <li><b>قفل الزاوية</b> <code>&lt;30</code> — يقيّد الاتجاه حتى النقرة</li>
  <li><b>المقاس</b> <code>9x14</code> — للمستطيل</li>
 </ul>
 <h4>المفاتيح</h4>
 <ul>
  <li><b>Esc</b> يلغي الأداة أو التحديد ·
      <b>Enter / مسافة</b> يؤكّد أو يعيد آخر أداة</li>
  <li><b>Ctrl+Z</b> تراجع · <b>Ctrl+Shift+Z</b> إعادة ·
      <b>Ctrl+A</b> تحديد المرئيّ · <b>Ctrl+S</b> حفظ</li>
  <li><b>Ctrl+Shift+L</b> أظهر كل الطبقات</li>
  <li><b>F3</b> التقاط الكائنات · <b>F8</b> تعامد ·
      <b>F10</b> قطبي · <b>F11</b> مقابض ·
      <b>Ctrl+Shift+E</b> علامات الأطراف</li>
  <li><b>F7</b> الفاحص · <b>F1</b> هذه اللوحة</li>
  <li><b>F6</b> الشبكة · <b>F9</b> الالتقاط على الخطوة</li>
  <li><b>الأسهم</b> تُزحزح المحدَّد خطوةَ التقاط (Shift ×10)</li>
  <li><b>وسط الفأرة</b> تحريك · <b>العجلة</b> تكبير ·
      <b>الزرّ الأيمن</b> Enter</li>
  <li><b>Alt</b> يُظهر دلائل التبويبات ثم رقمها ·
      <b>Ctrl+F1</b> يطوي الشريط · <b>Ctrl+0</b> أو
      <b>Ctrl+Shift+C</b> شاشة نظيفة ·
      <b>Ctrl+Shift+T</b> أو <b>Ctrl+Shift+M</b> يبدّل السِّمة</li>
  <li><b>الإدخال الحركي</b> Ctrl+D · <b>Tab</b> يقفل الحقل وينتقل ·
      <b>Ctrl+Shift+Q</b> الخصائص السريعة</li>
  <li><b>الزرّ الأيمن</b> Enter مع أداةٍ نشطة · قائمةُ سياق في
      السكون — يُبدَّل من «عرض ← الإدخال»</li>
  <li><b>Ctrl+K</b> لوحة الأوامر (بحثٌ عربي/إنجليزي عن أداة) ·
      <b>Ctrl+Shift+H</b> لوحة السجل (قفزٌ لأي خطوة تراجع)</li>
 </ul>
 <p class="hint" style="color:var(--fg3);margin:10px 0 0;font-size:11.5px">
  أربعة اختصاراتٍ أوتوكادية يحتجزها المتصفّح ولا يمكن ردُّها:
  <b>F12</b> و<b>Ctrl+Shift+I</b> (أدوات المطوّر) و
  <b>Ctrl+Shift+T</b> (تبويب) و<b>Ctrl+0</b> (تكبير). كلٌّ منها
  له بديلٌ أعلاه، والأصل يعمل حيث لا يحتجزه المتصفّح.
 </p>
 <h4>العقد — ما يفعله البرنامج وما لا يفعله</h4>
 <ul>
  <li>لا يتحرّك إحداثيٌّ إلا بأمرك. لا لحم تلقائي ولا تقريب صامت
      ولا زحف فتحةٍ ولا تقليم بُعد.</li>
  <li>دمج الأركان وطرح الفتحات ودمج الأعمدة: <b>عرضٌ لا تعديل</b> —
      البيانات تبقى كما رسمتها، وحذف الفتحة يعيد الجدار كاملاً.</li>
  <li>المنطقة تُخبَز مرّةً بأمرك فتصير كائناً مستقلّاً. تغيّر جدارٍ
      يجعلها «قديمة» بحدٍّ متقطّع، وأنت تحدّثها أو تثبّتها أو تتركها.</li>
  <li>المخفيّ لا يُرسَم ولا يُحدَّد ولا يُصدَّر. المقفل يُرى ولا يُلمَس.
      والهندسة لا تُخفى: خبز المناطق يقرأ الجدران كلّها.</li>
  <li>المرجع المستورد جامد: تقيس عليه وتلتقط نقاطه، ولا يُستنتَج
      منه جدار.</li>
  <li>الفاحص يخبرك ولا يصلح. كل تحذير قابل للنقر يقفز إلى موضعه.</li>
 </ul>
 <h4>الأدوات</h4>
 <table class="tools"><thead><tr><th>الأداة</th><th>الاسم</th>
  <th>ملاحظة</th></tr></thead><tbody>${rows}</tbody></table>
 <h4>التصدير</h4>
 <ul>
  <li><b>DXF R12</b> — الشرطة قطعٌ حقيقية والهاشور خطوطٌ مولَّدة
      (لا HATCH في R12) · الترميز ANSI_1256</li>
  <li><b>SVG</b> — الأصدق للعربية: النصّ متّجه بخطّ النظام</li>
  <li><b>PDF</b> — الهندسة متّجهة، والنصّ العربي صورةٌ لكل نصٍّ فريد
      لأن الخطوط القياسية لا تحمل العربية</li>
  <li><b>PNG</b> — بأي دقّة، ورسّامه مستقلّ عن قماش الشاشة</li>
 </ul>
</div>`;
 B.hidden=false;
 $("#helpX").onclick=()=>{B.hidden=true; cl.focus()};
 $("#helpX").focus();          /* التركيز داخل الحوار */
}
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");

/* ═══ الإقلاع ═══ */
addEventListener("resize",resize);
/* IndexedDB لا يُعتمَد عليه عند الإغلاق — كتابةٌ متزامنة هنا */
document.addEventListener("visibilitychange",()=>{
 if(document.hidden)saveNow();
 else saveResume();          /* عادت الصفحة: الحفظ الآجل يعمل */
});
addEventListener("beforeunload",()=>saveNow());

(async function boot(){
 let had=false;
 if(SAFE)rep("wr","وضع الإنقاذ: لم تُستعَد الجلسة ولا التفضيلات. "
  +"ما هو محفوظ باقٍ — أزِل ?safe للعودة.");
 else{
  try{had=await restore()}
  catch(e){rep("er","تعذّر استعادة الجلسة: "+e.message)}
 }
 ensureShape();
 loadForms();
 clearFindings();
  if(!SAFE)snapAutoStart(10);
 refresh(true);
 resize();
 if(had){
  fit();
  rep("in",`استُعيدت الجلسة: ${S.walls.length} جدار · `
   +`${S.opens.length} فتحة · ${S.areas.length} منطقة`
   +((S.ref&&S.ref.ents&&S.ref.ents.length)
     ?` · مرجع ${S.ref.ents.length} كياناً`:""));
  /* العائد قد يكون كائناً — الترميم يحمل عدد ما رُمِّم */
  const via=(had&&had.via)||had;
  if(via==="migrate")rep("ok","نُقلت الجلسة إلى IndexedDB — "
   +"لا حدَّ ٥ م.ب بعد الآن");
  if(via==="healed")rep("wr","آخر إغلاقٍ لم يتّسع للمرجع في "
   +`الحفظ السريع، فرُمِّم من النسخة الكاملة (${had.refs} كياناً). `
   +"رسمك من الأحدث والمرجع من الأسبق — راجعه إن كنت حاذيتَه "
   +"قُبيل الإغلاق.");
 }else{
  V.k=0.05; V.cx=6000; V.cy=4000;
  draw();
  rep("in","CivilDraft — ابدأ بأداة «جدار» أو استورد DXF مرجعاً. "
   +"F1 للمساعدة.");
  /* الجلسة الأساسية لم تُعِد شيئاً — نسأل عن النسخة الاحتياطية
     الثانوية إن وُجدت، بدل عرض حوارين معاً. */
  if(!SAFE&&phase1Autosave){
   const got=await maybeOfferRecovery(phase1Autosave).catch(()=>false);
   if(got){ fit(); rep("ok","استُعيدت نسخةٌ احتياطية سابقة."); }
  }
 }
 if(saveMode()==="ls")rep("wr","IndexedDB غير متاح — الحفظ "
  +"التلقائي في localStorage بحدّ ٥ م.ب، ومرجعٌ كبير قد لا يُحفَظ. "
  +"احفظ ملفّاً بين حينٍ وحين.");
 ribbonSel();
 syncRibbonTogs();
 if(UIS.wsCur)rep("in",`سطح العمل: ${UIS.wsCur}`);
 syncPrompt();
 cl.focus();
  tourMaybe(SAFE,had);
  bootOk();
})().catch(e=>fatal((e&&e.message)||String(e),"الإقلاع"));
```

<a id="f-js-bootguard-js"></a>

---

## `js/bootguard.js`

```javascript
/* ═══ حرس الإقلاع ═══
   أوّل ما يُحمَّل، وورقةٌ في شجرة الاعتماد. يمسك عطب التحميل وعطب
   البناء وأيَّ وعدٍ مرفوض، ويكتب سبباً مرئياً — فالشاشة البيضاء
   الصامتة أسوأ ما قد يقع في برنامجٍ يحمل عمل المستخدم.

   ومنفذ إنقاذ: ?safe يتجاهل تفضيلات الواجهة والجلسة المحفوظة ولا
   يمحوهما — فتُفتَح النسخة السليمة ويُحفَظ الملفّ ثم يُصلَح ما فسد. */

export const SAFE=/[?&]safe\b/.test(location.search);
let DONE=false, SHOWN=false;

const box=()=>{
 let b=document.getElementById("bootErr");
 if(b)return b;
 b=document.createElement("div");
 b.id="bootErr";
 b.setAttribute("dir","rtl");
 b.style.cssText="position:fixed;inset-inline:0;inset-block-start:0;"
  +"z-index:9999;background:#3a1c1c;color:#ffd9d9;"
  +"border-block-end:1px solid #6b2b2b;padding:10px 14px;"
  +"font:13px/1.6 Tahoma,Arial,sans-serif;max-block-size:60vh;"
  +"overflow:auto;white-space:pre-wrap";
 (document.body||document.documentElement).appendChild(b);
 return b;
};
export function fatal(msg,where){
 SHOWN=true;
 const b=box();
 const line=(where?`[${where}] `:"")+String(msg==null?"":msg);
 b.appendChild(document.createTextNode(line+"\n"));
 if(!b.dataset.tail){
  b.dataset.tail="1";
  const a=document.createElement("div");
  a.style.cssText="margin-block-start:8px;color:#ffb3b3";
  a.textContent=SAFE
   ? "أنت في وضع الإنقاذ سلفاً — احفظ ملفّاً إن ظهر رسمك."
   : "جرّب وضع الإنقاذ: أضِف ?safe إلى العنوان — "
     +"يتجاهل التفضيلات المحفوظة ولا يمحوها.";
  b.appendChild(a);
 }
 return false;
}
/* يُنادى من آخر boot() فيُلغي المرقب */
export const bootOk=()=>{DONE=true};

addEventListener("error",e=>{
 if(DONE&&SHOWN)return;
 const m=e.error&&e.error.stack
  ? String(e.error.stack).split("\n").slice(0,3).join("\n")
  : (e.message||"عطبٌ غير موصوف");
 fatal(m,"تحميل");
},true);
addEventListener("unhandledrejection",e=>{
 const r=e.reason;
 fatal((r&&r.message)||String(r),"وعدٌ مرفوض");
});
/* مرقب: يمسك فشل جلب وحدةٍ — لا يفير error على النافذة */
setTimeout(()=>{
 if(DONE||SHOWN)return;
 fatal("لم يكتمل الإقلاع خلال عشر ثوان — تعذّر تحميل وحدةٍ؟ "
  +"افتح وحدة التحكّم لترى الملفّ المفقود.","مرقب");
},10000);
```

---

---

