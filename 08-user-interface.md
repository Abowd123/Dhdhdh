# CivilDraft — User Interface (UI)

> مكوّنات الواجهة: اللوحات، الشريط، القوائم، الحوارات، السمة.

**عدد الملفات:** 34

---

## `js/ui/ai.js`

````javascript
/* ═══ لوحة المساعد ═══
   ثلاث حالات: خاملة · خطّةٌ منتظرةٌ للموافقة · مُنفَّذةٌ تحت
   المعاينة. لا شيء يُثبَّت بلا نقرةٍ منك. */
import {S} from "../core/state.js";
import {inspect} from "../core/inspect.js";
import {sceneBBoxAll,scene} from "../core/render.js";
import {renderCanvas} from "../io/png.js";
import {digest,digestSize} from "../ai/ctx.js";
import {SYS} from "../ai/lang.js";
import {AI,loadAI,saveAI,ready,isLocal,hostOf,ask,
        abort} from "../ai/net.js";
import {parsePlan,planReady} from "../ai/plan.js";
import {trial,commit,rollback} from "../ai/run.js";
import {validate as validateOps} from "../ai/ops.js";
import {runOps} from "../ai/opsrun.js";
import {editFailed} from "../core/state.js";
import {mnum} from "../core/units.js";
import {HOOK} from "./bus.js";
import {generateFromText} from "../ai/generate.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");

/* كتلةُ ‹ops› بديلٌ لكتلة ‹plan› — راجع ai/lang.js#SYS. تُقرأ هنا
   وحدها (لا في ai/plan.js) فلا يفترق ما يُقبَل عمّا يُعرَض. */
function extractOps(txt){
 const T=String(txt||"");
 const m=T.match(/```ops[ \t]*\r?\n([\s\S]*?)```/i);
 if(!m)return null;
 let j=null;
 try{j=JSON.parse(m[1])}catch(e){return null}
 if(!j||!Array.isArray(j.ops))return null;
 return {ops:j.ops, why:String(j.why||""),
  prose:T.slice(0,m.index).trim()||T.replace(m[0],"").trim()};
}

/* سطرٌ مختصر لعرض عمليةٍ مُصدَّقة — لا يُنفَّذ، عرضٌ فقط */
function opLine(o){
 const P=p=>`${mnum(p[0])},${mnum(p[1])}`;
 if(o.op==="wall")return `جدار ${P(o.a)}→${P(o.b)}`
  +(o.t!=null?` س${mnum(o.t)}`:"")+(o.type?` ${o.type}`:"");
 if(o.op==="open")return `فتحة ${o.kind} على ${o.wall} `
  +`عند ${mnum(o.at)} ع${mnum(o.w)}`;
 if(o.op==="area")return `منطقة عند ${P(o.at)}`
  +(o.name?` «${o.name}»`:"");
 if(o.op==="text")return `نصّ عند ${P(o.at)}: ${o.s}`;
 if(o.op==="field")return `حقل ${o.field}=${o.value} `
  +`(${o.ids.length} عنصراً)`;
 if(o.op==="note")return `ملاحظة: ${o.s}`;
 return o.op;
}
let PLAN=null, TRIAL=null, OPS=null, BUSY=0;
const wantIns=()=>($("#aiIns")?$("#aiIns").checked:!!+AI.wantIns);
const wantTtl=()=>($("#aiTtl")?$("#aiTtl").checked:!!+AI.wantTtl);
const wantDes=()=>($("#aiDes")?$("#aiDes").checked:0);

export function renderAI(){
 const b=$("#aiBox");
 if(!b)return;
 let h="";
 if(!ready())
  h+=`<p class="hint warn">المساعد غير مُهيَّأ — املأ العنوان `
   +`والموديل وشغّله.</p>`;
 else h+=`<p class="hint">${esc(AI.model)} · `
  +(isLocal()?`محلّي — لا يخرج شيء من جهازك`
   :`<span class="warn">خارجي — تُرسَل خلاصة مشروعك إلى `
    +`${esc(hostOf())}</span>`)
  +(AI.keep?"":` · المفتاح لهذه الجلسة وحدها`)+`</p>`;
 if(BUSY)h+=`<p class="hint">يُفكّر… <button id="aiStop">`
  +`ألغِ</button></p>`;
 if(PLAN&&PLAN.prose)
  h+=`<div class="ro" style="white-space:pre-wrap;margin:5px 0">`
   +`${esc(PLAN.prose)}</div>`;
 (PLAN?PLAN.notes:[]).forEach(n=>{
  h+=`<p class="hint">${esc(n)}</p>`});
 if(PLAN&&PLAN.hasPlan&&!TRIAL){
  h+=`<p class="hint">${PLAN.lines.length} سطراً`
   +(PLAN.destruct.length?` · <span class="warn">يحوي `
    +`${esc([...new Set(PLAN.destruct)].join(" · "))}</span>`:"")
   +`</p>`;
  /* المعروض هو المُنفَّذ: السطر المطبَّع لا الخام — فلا يفترق ما
     تراه عمّا يُغذّى إلى feedText */
  h+=PLAN.lines.map(L=>`<div class="ro" style="margin:1px 0`
   +`${L.bad?";color:var(--er)":""}"><span class="mono">`
   +`${esc(L.s)}</span> <span style="color:var(--fg3)">`
   +`${esc(L.bad||L.note)}</span></div>`).join("");
  h+=`<div class="btnrow">`
   +`<button id="aiRun" class="pri"${planReady(PLAN)?"":" disabled"}>`
   +`نفّذ للمعاينة</button>`
   +`<button id="aiDrop">أهمِل</button></div>`;
  if(PLAN.errs.length)
   h+=`<p class="hint warn">${PLAN.errs.length} سطراً مرفوضاً — `
    +`عدّل الطلب أو صرّح بالأوامر الهادمة</p>`;
 }
 if(OPS&&OPS.prose)
  h+=`<div class="ro" style="white-space:pre-wrap;margin:5px 0">`
   +`${esc(OPS.prose)}</div>`;
 if(OPS){
  h+=`<p class="hint">${OPS.V.ok.length} عمليةً صالحة`
   +(OPS.V.bad.length?` · <span class="warn">`
    +`${OPS.V.bad.length} مرفوضة</span>`:"")+`</p>`;
  /* المعروض معاينةُ التصديق (validate) وحدها — بلا كتابة، فلا
     يُنفَّذ شيءٌ إلّا بالنقر على «نفّذ» */
  h+=OPS.V.ok.map(o=>`<div class="ro" style="margin:1px 0">`
   +`<span class="mono">${esc(opLine(o))}</span></div>`).join("");
  h+=OPS.V.bad.map(b=>`<div class="ro" style="margin:1px 0;`
   +`color:var(--er)">#${b.i}: ${esc(b.why)}</div>`).join("");
  h+=`<div class="btnrow">`
   +`<button id="aiOpsRun" class="pri"`
   +`${OPS.V.ok.length?"":" disabled"}>نفّذ</button>`
   +`<button id="aiOpsDrop">أهمِل</button></div>`
   +`<p class="hint">لا تنقل ولا تحذف — إنشاءٌ وتعديلُ حقولٍ `
   +`فقط، فتُثبَّت مباشرةً في خطوة تراجعٍ واحدة (Ctrl+Z يتراجع `
   +`عنها).</p>`;
 }
 if(TRIAL){
  h+=`<p class="hint" style="color:var(--wr)">مُنفَّذة تحت `
   +`المعاينة: ${TRIAL.ran} سطراً · ${TRIAL.made} كياناً جديداً`
   +(TRIAL.errs?` · ${TRIAL.errs} رفضاً`:"")+`</p>`;
  h+=TRIAL.res.filter(x=>x.err).slice(0,8).map(x=>
   `<div class="ro" style="color:var(--er);margin:1px 0">`
   +`${esc(x.s)} — ${esc(x.err)}</div>`).join("");
  h+=`<div class="btnrow">`
   +`<button id="aiKeep2" class="pri">ثبّت</button>`
   +`<button id="aiBack" class="del">أرجِع</button></div>`
   +`<p class="hint">انظر اللوحة قبل التثبيت. الإرجاع يعيد كل `
   +`شيء إلى ما قبل الخطة.</p>`;
 }
 b.innerHTML=h;
 wireDyn();
}
function wireDyn(){
 const s=$("#aiStop"); if(s)s.onclick=()=>{abort()};
 const r=$("#aiRun"); if(r)r.onclick=()=>runPlan();
 const d=$("#aiDrop");
 if(d)d.onclick=()=>{PLAN=null; renderAI()};
 const or=$("#aiOpsRun"); if(or)or.onclick=()=>runOpsPlan();
 const od=$("#aiOpsDrop");
 if(od)od.onclick=()=>{OPS=null; renderAI()};
 const k=$("#aiKeep2");
 if(k)k.onclick=()=>{
  const n=commit(TRIAL);
  HOOK.report("ok",`ثُبّتت الخطة · ${n} كياناً · Ctrl+Z يتراجع `
   +`عنها كلّها`);
  TRIAL=null; PLAN=null;
  HOOK.refresh(1); renderAI();
 };
 const b=$("#aiBack");
 if(b)b.onclick=()=>{
  rollback(TRIAL);
  HOOK.report("in","أُرجِعت الخطة — لا أثر");
  TRIAL=null;
  HOOK.refresh(1); renderAI();
 };
}
function runPlan(){
 if(!planReady(PLAN))return;
 TRIAL=trial(PLAN.lines,{stopOnError:1});
 HOOK.refresh(0);
 HOOK.report(TRIAL.errs?"wr":"ok",
  `معاينة: ${TRIAL.ran} سطراً · ${TRIAL.made} كياناً`
  +(TRIAL.errs?` · ${TRIAL.errs} رفضاً`:"")
  +" — ثبّت أو أرجِع");
 renderAI();
}
/* لا معاينةَ حيّةً هنا (راجع opsrun.js) — التنفيذ ذرّيٌّ فوراً،
   وedit() نفسه يرجع تلقائياً إن أخفق. */
function runOpsPlan(){
 if(!OPS||!OPS.V.ok.length)return;
 const r=runOps(OPS.ops);
 if(editFailed()||!r){
  HOOK.report("er","تعذّر التنفيذ"); return;
 }
 HOOK.report(r.refused.length?"wr":"ok",
  `${r.say} · Ctrl+Z يتراجع عنها`);
 OPS=null;
 HOOK.refresh(1); renderAI();
}
export async function askFromLine(q){
 if(TRIAL){HOOK.report("wr","ثبّت المعاينة أو أرجِعها أوّلاً");return}
 if(!q)return;
 if(!ready()){HOOK.report("er","المساعد غير مُهيَّأ");return}
 if(!isLocal()&&!AI.__ok){
  /* الموافقة تذكر الوجهة وما يُرسَل بعدده — لا سؤالاً مبهماً.
     وهي موافقةُ جلسةٍ لا تُحفَظ على القرص. */
  const D=digest({inspect:0,title:wantTtl()?1:0});
  if(!confirm(`سيُرسَل وصفُ مشروعك إلى ${hostOf()}.\n\n`
   +`الحجم: ${digestSize(D)}\n`
   +`المحتوى: أبعاد الجدران والفتحات وأسماء المناطق`
   +(wantTtl()?"\nوبلوك العنوان — فيه اسم المالك والموقع":"")
   +(AI.vision&&S.walls.length?"\nوصورةُ اللوحة":"")
   +`\n\nلخصوصيةٍ كاملة استعمل Ollama محلّياً.\n`
   +`ويُسأل مرّةً واحدة في هذه الجلسة.`))return;
  AI.__ok=1;
 }
 const dst=$("#aiAsk");
 if(dst)dst.value=q;
 BUSY=1; renderAI();
 try{
  const ins=wantIns()?inspect(sceneBBoxAll()):0;
  const D=digest({inspect:ins,title:wantTtl()?1:0});
  let img=null;
  if(AI.vision&&S.walls.length){
   const r=renderCanvas(scene().P,sceneBBoxAll(),
    {dpi:96,max:1400,pad:Math.max(1,S.meta.scale)*8});
   img=r.canvas.toDataURL("image/png");
  }
  HOOK.status(`يُرسَل ${digestSize(D)}`);
  const r=await ask(SYS(),`${D}\n\n## الطلب\n${q}`,img);
  /* ‹ops› تُقرأ أوّلاً: extract() في plan.js يقع على أوّل سياجٍ
     حين لا يجد «plan» موسومةً، فكتلةُ ops وحدها كانت تُقرأ
     أسطرَ أوامرَ فتُرفَض كلّها كنصٍّ مجهول. */
  const eo=extractOps(r.txt);
  if(eo){
   PLAN=null;
   OPS={ops:eo.ops, prose:eo.prose, V:validateOps(eo.ops)};
   HOOK.report(OPS.V.bad.length?"wr":"in",
    `المساعد: ${r.ms} مس`
    +(r.usage?` · ${r.usage.total_tokens||"?"} رمزاً`:"")
    +` · ${OPS.V.ok.length} عمليةً`
    +(OPS.V.bad.length?` · ${OPS.V.bad.length} مرفوضة`:""));
  }else{
   OPS=null;
   PLAN=parsePlan(r.txt,wantDes()?1:0,AI.maxLines);
   HOOK.report(PLAN.hasPlan?"in":"ok",
    `المساعد: ${r.ms} مس`
    +(r.usage?` · ${r.usage.total_tokens||"?"} رمزاً`:"")
    +(PLAN.hasPlan?` · خطّةٌ من ${PLAN.lines.length} سطراً`
      :" · إجابة"));
   if(!PLAN.hasPlan&&PLAN.prose)HOOK.report("ok",PLAN.prose);
  }
 }catch(e){HOOK.report("er",e.message)}
 BUSY=0; renderAI();
}
export function wireAI(){
 loadAI();
 /* aiIns وaiTtl صارا محفوظَين: كانا يُقرآن من DOM ولا يُخزَّنان،
    فيعودان مُطفأَين في كل جلسة. و«اسمح بالأوامر الهادمة» يبقى
    بلا حفظٍ بقصد: تصريحٌ لا يليق به اللزوم. */
 const F=["aiUrl","aiKey","aiModel","aiTemp","aiVis","aiOn",
  "aiKeep","aiIns","aiTtl"];
 const put=()=>{
  const g=id=>$("#"+id);
  if(g("aiUrl"))g("aiUrl").value=AI.url;
  if(g("aiKey"))g("aiKey").value=AI.key?"••••••••":"";
  if(g("aiModel"))g("aiModel").value=AI.model;
  if(g("aiTemp"))g("aiTemp").value=AI.temp;
  if(g("aiVis"))g("aiVis").checked=!!+AI.vision;
  if(g("aiOn"))g("aiOn").checked=!!+AI.on;
  if(g("aiKeep"))g("aiKeep").checked=!!+AI.keep;
  if(g("aiIns"))g("aiIns").checked=!!+AI.wantIns;
  if(g("aiTtl"))g("aiTtl").checked=!!+AI.wantTtl;
 };
 put();
 F.forEach(id=>{
  const el=$("#"+id);
  if(!el)return;
  el.onchange=()=>{
   AI.url=$("#aiUrl").value.trim();
   const k=$("#aiKey").value;
   if(k&&!/^•+$/.test(k))AI.key=k.trim();
   AI.model=$("#aiModel").value.trim();
   AI.temp=Math.max(0,Math.min(1,parseFloat($("#aiTemp").value)||0));
   AI.vision=$("#aiVis").checked?1:0;
   AI.on=$("#aiOn").checked?1:0;
   if($("#aiKeep"))AI.keep=$("#aiKeep").checked?1:0;
   if($("#aiIns"))AI.wantIns=$("#aiIns").checked?1:0;
   if($("#aiTtl"))AI.wantTtl=$("#aiTtl").checked?1:0;
   AI.__ok=0;                    /* الوجهة قد تكون تغيّرت */
   saveAI(); put(); renderAI();
   if(id==="aiKeep"&&!AI.keep)
    HOOK.report("in","المفتاح لهذه الجلسة وحدها — لن يُكتَب "
     +"على القرص");
  };
  el.onkeydown=e=>{
   if(e.key==="Escape"){el.blur();return}
   e.stopPropagation();
  };
 });
 const s=$("#aiSend");
 if(s)s.onclick=()=>askFromLine(($("#aiAsk").value||"").trim());
 const a=$("#aiAsk");
 if(a)a.onkeydown=e=>{
  if(e.key==="Escape"){a.blur();return}
  if(e.key==="Enter"&&(e.ctrlKey||e.metaKey)){
   e.preventDefault();
   askFromLine((a.value||"").trim());
   return;
  }
  e.stopPropagation();
 };
 const c=$("#aiKeyClr");
 if(c)c.onclick=()=>{AI.key=""; saveAI(); put();
  HOOK.report("in","مُسح المفتاح")};
 wireLocalGen();
 renderAI();
}

/* ═══ توليد محلّي بلا اتصال ═══ ai/generate.js فقط — لا AI.on ولا
   شبكة. نفس عقد لوحة ops أعلاه: معاينةٌ (validate) بلا كتابة، ثم
   تنفيذٌ صريح بنقرة يعبر runOps فخطوة تراجعٍ واحدة. */
function renderGenReport(r,committed){
 const b=$("#genBox");
 if(!b)return;
 if(!r){b.innerHTML="";return}
 let h=`<p class="hint">${r.valid} عمليةً صالحة`
  +(r.rejected.length?` · <span class="warn">`
   +`${r.rejected.length} مرفوضة</span>`:"")
  +(r.unmatched.length?` · ${r.unmatched.length} لم يُفهَم`:"")+`</p>`;
 h+=r.rejected.map(x=>`<div class="ro" style="margin:1px 0;`
  +`color:var(--er)">#${x.i}: ${esc(x.why)}</div>`).join("");
 r.unmatched.forEach(u=>{h+=`<div class="ro" style="margin:1px 0;`
  +`color:var(--fg3)">لم يُفهَم: ${esc(u)}</div>`});
 if(committed&&r.result)
  h+=`<p class="hint" style="color:var(--wr)">${esc(r.result.say)} `
   +`· Ctrl+Z يتراجع عنها</p>`;
 else if(r.valid)
  h+=`<p class="hint">معاينةٌ فقط — لم يُكتَب شيء. اضغط `
   +`«توليد ونفّذ» للتثبيت.</p>`;
 b.innerHTML=h;
}
function runLocalGen(commit){
 const a=$("#genAsk");
 const text=(a?a.value:"").trim();
 if(!text){HOOK.report("wr","اكتب وصفاً أوّلاً");return}
 const r=generateFromText(text,{commit});
 renderGenReport(r,commit);
 if(commit&&r.committed){
  if(editFailed()){HOOK.report("er","تعذّر التنفيذ");return}
  HOOK.report(r.rejected.length?"wr":"ok",
   `${r.result.say} · Ctrl+Z يتراجع عنها`);
  HOOK.refresh(1);
 }else{
  HOOK.report(r.valid?"in":"wr",
   `${r.valid} من ${r.generated} عمليةً صالحة`
   +(r.unmatched.length?` · ${r.unmatched.length} لم يُفهَم`:""));
 }
}
function wireLocalGen(){
 const run=$("#genRun"); if(run)run.onclick=()=>runLocalGen(true);
 const prev=$("#genPrev"); if(prev)prev.onclick=()=>runLocalGen(false);
}
````

<a id="f-js-ui-appmenu-js"></a>

---

## `js/ui/appmenu.js`

```javascript
/* ═══ قائمة «مِسطَر» ═══
   بديل زرّ التطبيق في أوتوكاد. وفيها بحثُ الأوامر الذي يقابل
   صندوق «Type a keyword»، إلّا أنه يقرأ سجلّ الأدوات نفسه
   بأسمائها ومختصراتها المعلَنة في alias — فلا قائمةَ ثانية
   تتخلّف عن الأولى.

   «الملفّات الحديثة» مؤجَّلة إلى م٨: لا سجلَّ ملفّاتٍ اليوم،
   وقائمةٌ فارغة تَعِد بما لا تُنجز. */
import {icon} from "./icons.js";
import {UIS} from "./store.js";
import {ACT,runItem,setTheme,setShell,setClean,revealSec}
 from "./ribbon/wire.js";
import * as R from "../tools/registry.js";
import {draw} from "./canvas.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||16):"";

const ROWS=[
 {act:"fnew", n:"مشروع جديد", ico:"fnew", k:""},
 {act:"xOpen",n:"افتح مشروعاً",ico:"open", k:""},
 {act:"xSave",n:"احفظ المشروع",ico:"save", k:"Ctrl+S"},
 {sep:1},
 {act:"xDxf",n:"تصدير DXF",ico:"dxf",k:""},
 {act:"xSvg",n:"تصدير SVG",ico:"svg",k:""},
 {act:"xPng",n:"تصدير PNG",ico:"png",k:""},
 {act:"xPdf",n:"تصدير PDF",ico:"pdf",k:""},
 {sep:1},
 {act:"rImp",   n:"استورد DXF مرجعاً",ico:"ref",    k:""},
 {act:"inspect",n:"افحص المخطط",      ico:"inspect",k:"F7"},
 {sep:1},
 {act:"shell",n:"بدّل القشرة",ico:"shell",k:""},
 {act:"theme",n:"بدّل السِّمة",  ico:"theme",k:""},
 {act:"clean",n:"شاشة نظيفة", ico:"clean",k:"Ctrl+0"},
 {sep:1},
 {act:"perfShow",n:"القياس",ico:"info",k:""},
 {act:"purgeAll",n:"امسح كل ما هو محفوظ محلّياً",ico:"del",k:""},
 {sep:1},
 {act:"help",n:"المساعدة",ico:"help",k:"F1"}
];
/* ═══ بحث الأوامر ═══ */
function search(q){
 const s=String(q||"").trim().toLowerCase();
 if(!s)return [];
 const out=[];
 R.toolList().forEach(d=>{
  if(!d||!d.id)return;
  const al=Object.keys(R.TOOLS).filter(k=>R.TOOLS[k]===d&&k!==d.id);
  const hay=[d.label,d.id].concat(al).join(" ").toLowerCase();
  if(!hay.includes(s))return;
  const rank=(d.label.toLowerCase().startsWith(s)||d.id.startsWith(s))
   ?0:1;
  out.push({d,al,rank});
 });
 out.sort((a,b)=>a.rank-b.rank
  ||a.d.label.localeCompare(b.d.label,"ar"));
 return out.slice(0,9);
}
let SR=[], SI=-1;

function renderSearch(){
 const box=$("#amRes");
 if(!box)return;
 if(!SR.length){box.hidden=true; box.innerHTML=""; return}
 box.innerHTML=SR.map((r,i)=>
  `<div class="amR${i===SI?" sel":""}" data-i="${i}">`
  +`<b>${esc(r.d.label)}</b>`
  +`<span class="mono">${esc(r.d.id)}`
  +(r.al.length?" · "+esc(r.al.slice(0,3).join(" ")):"")+`</span>`
  +`</div>`).join("");
 box.hidden=false;
}
function runSearch(i){
 const r=SR[i];
 if(!r)return;
 close();
 R.begin(r.d);
 HOOK.prompt(); draw();
 const c=$("#clIn");
 if(c)c.focus();
}
/* ═══ الفتح والإغلاق ═══ */
export function open(){
 const p=$("#appMenu");
 if(!p)return;
 p.hidden=false;
 $("#appBtn").setAttribute("aria-expanded","true");
 const s=$("#amSearch");
 if(s){s.value=""; SR=[]; SI=-1; renderSearch(); s.focus()}
}
export function close(){
 const p=$("#appMenu");
 if(!p||p.hidden)return;
 p.hidden=true;
 const b=$("#appBtn");
 if(b)b.setAttribute("aria-expanded","false");
 SR=[]; SI=-1;
}
export const isOpen=()=>{
 const p=$("#appMenu");
 return !!p&&!p.hidden;
};
export function buildAppMenu(){
 const p=$("#appMenu");
 if(!p)return 0;
 p.innerHTML=`
<div class="amHead">
 <input id="amSearch" type="text" spellcheck="false"
  autocomplete="off" placeholder="ابحث عن أداة — جدار · باب · بُعد">
 <div id="amRes" hidden></div>
</div>
<div class="amBody" role="menu">
${ROWS.map(r=>r.sep?`<div class="amSep"></div>`
 :`<button type="button" class="amI" role="menuitem" `
  +`data-act="${esc(r.act)}">${ic(r.ico)}`
  +`<span class="lb">${esc(r.n)}</span>`
  +`<span class="ky mono">${esc(r.k||"")}</span></button>`).join("")}
</div>`;
 return ROWS.filter(r=>!r.sep).length;
}
export function wireAppMenu(){
 const b=$("#appBtn"), p=$("#appMenu");
 if(!b||!p)return false;
 buildAppMenu();
 b.onclick=e=>{e.stopPropagation(); isOpen()?close():open()};

 p.addEventListener("click",e=>{
  const r=e.target.closest("[data-i]");
  if(r){runSearch(+r.dataset.i); return}
  const it=e.target.closest("[data-act]");
  if(!it)return;
  const act=it.dataset.act;
  close();
  runItem(it);
  if(!ACT[act])HOOK.report("wr",`فعلٌ غير معروف: ${act}`);
 });
 const s=$("#amSearch");
 if(s){
  s.addEventListener("input",()=>{
   SR=search(s.value); SI=SR.length?0:-1; renderSearch();
  });
  s.addEventListener("keydown",e=>{
   if(e.key==="Escape"){e.preventDefault(); close(); return}
   if(e.key==="ArrowDown"||e.key==="ArrowUp"){
    if(!SR.length)return;
    e.preventDefault();
    SI=(SI+(e.key==="ArrowDown"?1:-1)+SR.length)%SR.length;
    renderSearch();
    return;
   }
   if(e.key==="Enter"){
    e.preventDefault();
    if(SI>=0)runSearch(SI);
    return;
   }
   e.stopPropagation();
  });
 }
 addEventListener("mousedown",e=>{
  if(!isOpen())return;
  if(!p.contains(e.target)&&e.target!==b)close();
 },true);
 return true;
}
```

<a id="f-js-ui-blockdraw-js"></a>

---

## `js/ui/blockdraw.js`

```javascript
/* ═══ رسم مثيل كتلة على القماش أو في المصغّرة ═══ */
import { explode } from "../core/blocks.js";
import { pal } from "./theme.js";

export function drawInstance(ctx, inst, toScreen, opts = {}) {
  const prims = explode(inst);
  if (!prims.length) return;
  const P = pal();
  ctx.save();
  ctx.lineWidth = opts.lineWidth || 1.4;
  ctx.strokeStyle = opts.color || (opts.ghost ? (P.ghost || P.pre) : (P.dflt || "#e9edf2"));
  if (opts.ghost) { ctx.globalAlpha = 0.6; ctx.setLineDash([5, 4]); }
  for (const p of prims) {
    ctx.beginPath();
    if (p.t === "line") {
      const a = toScreen(p.a), b = toScreen(p.b);
      ctx.moveTo(a[0], a[1]); ctx.lineTo(b[0], b[1]);
    } else {
      p.pts.forEach((pt, i) => {
        const s = toScreen(pt);
        if (i) ctx.lineTo(s[0], s[1]); else ctx.moveTo(s[0], s[1]);
      });
      if (p.closed) ctx.closePath();
    }
    ctx.stroke();
  }
  ctx.restore();
}
```

<a id="f-js-ui-blockpanel-js"></a>

---

## `js/ui/blockpanel.js`

```javascript
/* ═══ لوحة العناصر: نقر أو سحب لبدء إدراج كتلة ═══ */
import { blockList, makeInstance, explode, bbox } from "../core/blocks.js";
import { startInsert } from "../tools/blocks.js";
import { pal } from "./theme.js";

const ID = "blkPanel";
let root = null, listEl = null, mounted = false;

function injectCss() {
  if (document.getElementById("blk-css")) return;
  const css = `
#${ID}{position:fixed;inset-block:56px auto;inset-inline-start:12px;z-index:38;
 width:194px;max-height:min(70vh,560px);display:flex;flex-direction:column;
 background:var(--bg2,#1a1d21);color:var(--fg,#e9edf2);
 border:1px solid var(--ln,#2a2f36);border-radius:8px;
 box-shadow:0 12px 34px rgba(0,0,0,.42);overflow:hidden;font:12.5px/1.4 var(--ui,system-ui,sans-serif)}
#${ID}[hidden]{display:none}#${ID} .blk-h{display:flex;align-items:center;padding:8px 10px;
 background:var(--bg3,#20242a);border-bottom:1px solid var(--ln,#2a2f36)}
#${ID} .blk-title{font-weight:600}#${ID} .blk-x{margin-inline-start:auto;background:none;border:0;
 cursor:pointer;color:var(--fg3,#8a929c);font-size:14px;padding:2px 5px}
#${ID} .blk-body{overflow-y:auto;padding:8px;display:grid;grid-template-columns:repeat(2,1fr);
 gap:6px;align-content:start}#${ID} .blk-it{display:flex;flex-direction:column;align-items:center;
 gap:4px;padding:6px 4px;cursor:pointer;border-radius:7px;color:var(--fg2,#c4ccd6);
 background:var(--bg2,#1a1d21);border:1px solid var(--ln2,#333a42)}
#${ID} .blk-it:hover{border-color:var(--ac,#6ea8fe);color:var(--fg,#e9edf2)}
#${ID} .blk-th{width:100%;height:38px;border-radius:5px;background:var(--bg3,#20242a)}
#${ID} .blk-lbl{font-size:11px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;max-width:100%}`;
  const st = document.createElement("style");
  st.id = "blk-css"; st.textContent = css; document.head.appendChild(st);
}

function thumb(name) {
  const w = 46, h = 38, pad = 6, cv = document.createElement("canvas");
  cv.width = w; cv.height = h; cv.className = "blk-th";
  const ctx = cv.getContext("2d"), inst = makeInstance(name), prims = explode(inst), bb = bbox(inst);
  const bw = Math.max(1e-6, bb.maxX - bb.minX), bh = Math.max(1e-6, bb.maxY - bb.minY);
  const s = Math.min((w - 2 * pad) / bw, (h - 2 * pad) / bh);
  const ox = (w - bw * s) / 2 - bb.minX * s, oy = (h - bh * s) / 2 - bb.minY * s;
  const T = ([x, y]) => [ox + x * s, h - (oy + y * s)];
  ctx.strokeStyle = pal().dflt || "#e9edf2"; ctx.lineWidth = 1.2;
  for (const p of prims) {
    ctx.beginPath();
    if (p.t === "line") { const a = T(p.a), b = T(p.b); ctx.moveTo(a[0], a[1]); ctx.lineTo(b[0], b[1]); }
    else { p.pts.forEach((pt, i) => { const q = T(pt); i ? ctx.lineTo(q[0], q[1]) : ctx.moveTo(q[0], q[1]); }); if (p.closed) ctx.closePath(); }
    ctx.stroke();
  }
  return cv;
}

function build() {
  injectCss();
  root = document.getElementById(ID);
  if (!root) {
    root = document.createElement("aside");
    root.id = ID; root.hidden = true; root.dir = "rtl";
    root.innerHTML = `<header class="blk-h"><span class="blk-title">العناصر</span>
      <button class="blk-x" data-act="close" title="إغلاق">✕</button></header><div class="blk-body"></div>`;
    document.body.appendChild(root);
  }
  listEl = root.querySelector(".blk-body");
  root.addEventListener("click", onClick);
}
function render() {
  if (!listEl) return;
  listEl.innerHTML = "";
  for (const b of blockList()) {
    const it = document.createElement("button");
    it.className = "blk-it"; it.dataset.name = b.name; it.title = b.title; it.draggable = true;
    it.appendChild(thumb(b.name));
    const lbl = document.createElement("span"); lbl.className = "blk-lbl"; lbl.textContent = b.title;
    it.appendChild(lbl);
    it.addEventListener("dragstart", e => e.dataTransfer.setData("application/x-mistar-block", b.name));
    listEl.appendChild(it);
  }
}
function onClick(e) {
  const act = e.target.closest("[data-act]");
  if (act?.dataset.act === "close") return closeBlockPanel();
  const it = e.target.closest(".blk-it");
  if (it) startInsert(it.dataset.name);
}
export function initBlockPanel() { if (mounted) return; build(); mounted = true; render(); }
export function openBlockPanel() { if (!mounted) initBlockPanel(); root.hidden = false; render(); }
export function closeBlockPanel() { if (root) root.hidden = true; }
export function toggleBlockPanel() { if (!mounted) initBlockPanel(); root.hidden = !root.hidden; if (!root.hidden) render(); }
```

<a id="f-js-ui-bus-js"></a>

---

## `js/ui/bus.js`

```javascript
/* ═══ ناقل الأحداث — يكسر الاعتماد الدائري بين القماش واللوحات ═══ */
export const HOOK={
 props:()=>{}, refresh:()=>{}, status:()=>{},
 report:()=>{}, prompt:()=>{}, toggles:()=>{}, help:()=>{},
 defs:()=>{}, ws:()=>{}, ctx:()=>false,
 layCtl:()=>{}, ctxPanel:()=>{},
 clean:()=>{}          /* مالكه wire.setClean — والإرساء يطلبه */
};
```

<a id="f-js-ui-canvas-js"></a>

---

## `js/ui/canvas.js`

```javascript
/* ═══ القماش: العرض والتفاعل ═══
   يقرأ قائمة أوّليات المشهد نفسها التي تُصدَّر، ويُفوّض كل ما يتعلّق
   بالكيانات إلى ents.js — فإضافة نوعٍ لا تعني تعديل هذا الملفّ.
   العالم بالمليمتر و Y للأعلى · الشاشة بالبكسل و Y للأسفل. */

import {S,txtH,edit,touch,snapshot,pushHistory,
        autosave} from "../core/state.js";
import {m2,mm,clamp,deg,D2R,scl} from "../core/units.js";
import {scene,sceneBBox,sceneBBoxAll} from "../core/render.js";
import {looseEnds,isLow} from "../core/walls.js";
import {osnap,osOn,MODES,MNAME} from "../core/osnap.js";
import {constrain} from "../core/coords.js";
import {vis,pickable,resolve} from "../core/layers.js";
import * as E from "../core/ents.js";
import * as R from "../tools/registry.js";
import {HOOK} from "./bus.js";
import {pal,layCss,primCss,setPal,themeName} from "./theme.js";
import {UIS} from "./store.js";
import {bump} from "../core/perf.js";
import {isInserting, onMove as onBlockMove, onClick as onBlockClick,
        ghost as blockGhost} from "../tools/blocks.js";
import {drawInstance} from "./blockdraw.js";
import {draw as drawUnderlay} from "../core/underlay.js";
import {jrTaint} from "../core/journal.js";

export const cv=document.getElementById("cv");
const ctx=cv.getContext("2d");

export const V={k:0.05,cx:0,cy:0,w:0,h:0,dpr:1};
export const UI={SELS:[],SEL:null,guides:[],track:null,osnap:null,
 marq:null,grips:[],hot:null,hov:null,pre:null,preP:null,free:null};

export const W2S=(x,y)=>[(x-V.cx)*V.k+V.w/2, V.h/2-(y-V.cy)*V.k];
export const S2W=(px,py)=>[V.cx+(px-V.w/2)/V.k, V.cy-(py-V.h/2)/V.k];

export function resize(){
 const r=cv.parentElement.getBoundingClientRect();
 V.dpr=Math.min(2,window.devicePixelRatio||1);
 V.w=Math.max(200,Math.round(r.width));
 V.h=Math.max(160,Math.round(r.height));
 cv.width=Math.round(V.w*V.dpr);
 cv.height=Math.round(V.h*V.dpr);
 cv.style.width=V.w+"px";
 cv.style.height=V.h+"px";
 ctx.setTransform(V.dpr,0,0,V.dpr,0,0);
 draw();
}
export function fitBox(B,pad){
 vPush();
 if(!B){V.k=0.05;V.cx=0;V.cy=0;draw();return}
 const w=Math.max(1000,B.x1-B.x0), h=Math.max(1000,B.y1-B.y0);
 const p=(pad==null?0.07:pad);
 V.k=clamp(Math.min(V.w/(w*(1+p*2)), V.h/(h*(1+p*2))),1e-5,3);
 V.cx=(B.x0+B.x1)/2; V.cy=(B.y0+B.y1)/2;
 draw();
}
export const fit=()=>fitBox(sceneBBoxAll());
export function zoomAt(px,py,f){
 vEpoch();
 const b=S2W(px,py);
 V.k=clamp(V.k*f,1e-5,3);
 const a=S2W(px,py);
 V.cx+=b[0]-a[0]; V.cy+=b[1]-a[1];
 draw();
}
export function panBy(dx,dy){vEpoch(); V.cx-=dx/V.k; V.cy+=dy/V.k; draw()}

/* ═══ تاريخ المنظر ═══
   عهدٌ لكل حركةٍ متّصلة: العجلة والتحريك تُسجّلان مرّةً عند بدء
   الحركة لا مع كل بكسل، والعمليات المنفصلة تُسجّل لحظتها. */
const VH=[];
let vLast=0;
function vPush(){
 const c={k:V.k,cx:V.cx,cy:V.cy};
 const p=VH[VH.length-1];
 if(p&&Math.abs(p.k-c.k)<1e-12&&p.cx===c.cx&&p.cy===c.cy)return;
 VH.push(c);
 if(VH.length>40)VH.shift();
}
function vEpoch(){
 const t=Date.now();
 if(t-vLast>500)vPush();
 vLast=t;
}
export const canPrev=()=>VH.length>0;
export function zoomPrev(){
 const v=VH.pop();
 if(!v)return false;
 V.k=v.k; V.cx=v.cx; V.cy=v.cy;
 draw();
 return true;
}
/* ═══ أوضاع التنقّل ═══ لا تُنشئ شيئاً ولا تعدّل بياناتٍ ═══ */
export const NAV={mode:null};
export const navMode=()=>NAV.mode;
export function navSet(m){
 NAV.mode=(m==="zw"||m==="pan")?m:null;
 cv.style.cursor=NAV.mode==="pan"?"grab"
  :(NAV.mode==="zw"?"crosshair":"crosshair");
 if(!NAV.mode)UI.zwin=null;
 draw();
 return NAV.mode;
}
export function setTheme(t){
 const v=setPal(t);
 Object.keys(PAT).forEach(k=>{delete PAT[k]});
 draw();
 return v;
}

/* ═══ الالتقاط والتقييد ═══ */
let shift=false;
export const setShift=v=>{shift=!!v};
export const snapMode=()=>{
 if((!!S.rb.ortho)!==(!!shift))return "ortho";
 return (+S.rb.polar)?"polar":null;
};
export function snap(x,y,from){
 UI.guides=[]; UI.osnap=null; UI.track=null;
 const st=Math.max(1,S.meta.snap);
 const tol=16/V.k;
 if(+S.rb.snap&&osOn()){
  const o=osnap(x,y,tol,from);
  if(o){UI.osnap=o;return o.p}
 }
 let p=(+S.rb.gsnap)
  ? [Math.round(x/st)*st, Math.round(y/st)*st]
  : [Math.round(x), Math.round(y)];
 /* ═══ القفل ═══
    زاويةٌ مقفلة أو طولٌ مقفل أو كلاهما. والطول وحده يتبع اتجاه
    المؤشّر بعد تقييده بالتعامد أو القطبي إن كانا مُشغَّلين. */
 const AL=R.T.lock, LL=R.T.lenLock;
 if(from&&(AL!=null||LL!=null)){
  let ux,uy;
  if(AL!=null){ux=Math.cos(AL*D2R); uy=Math.sin(AL*D2R)}
  else{
   const dx=x-from[0], dy=y-from[1], Ld=Math.hypot(dx,dy);
   if(Ld<1)return [from[0],from[1]];
   ux=dx/Ld; uy=dy/Ld;
   const md0=snapMode();
   if(md0){
    const c0=constrain(from,[x,y],md0,S.pol.inc,S.pol.extra);
    if(c0){ux=Math.cos(c0.a*D2R); uy=Math.sin(c0.a*D2R)}
   }
  }
  const t=(LL!=null)?LL:((x-from[0])*ux+(y-from[1])*uy);
  p=[Math.round(from[0]+ux*t),Math.round(from[1]+uy*t)];
  UI.track={from,a:deg(Math.atan2(uy,ux)*180/Math.PI),p};
  return p;
 }
 const md=snapMode();
 if(from&&md){
  const c=constrain(from,[x,y],md,S.pol.inc,S.pol.extra);
  if(c){UI.track={from,a:c.a,p:c.p}; return c.p}
 }
 return p;
}
/* ═══ التحديد — يُفوّض إلى ents ═══ */
export const hitTest=(x,y)=>E.hitTest(x,y,14/V.k);
/* موضعٌ تمثيليّ للكيان — لتثبيت البطاقات قربه */
export function shapeOfSel(s){
 const poly=E.outlineOf(s);
 if(poly&&poly.length){
  let x=0,y=0;
  poly.forEach(p=>{x+=p[0];y+=p[1]});
  return [x/poly.length,y/poly.length];
 }
 const sh=E.shapeOf(s);
 if(!sh)return null;
 if(sh.t==="pt")return sh.p.slice();
 if(sh.t==="seg")return [(sh.a[0]+sh.b[0])/2,(sh.a[1]+sh.b[1])/2];
 const P=sh.pts||[];
 if(!P.length)return null;
 let x=0,y=0;
 P.forEach(p=>{x+=p[0];y+=p[1]});
 return [x/P.length,y/P.length];
}
const same=(a,b)=>a&&b&&a.k===b.k&&a.id===b.id;
const inSel=s=>UI.SELS.some(x=>same(x,s));
export function setSel(list,primary){
 UI.SELS=list||[];
 UI.SEL=primary||(UI.SELS.length?UI.SELS[UI.SELS.length-1]:null);
 rebuildGrips();
 HOOK.props();
}
export const selList=()=>UI.SELS.slice();
export function toggleSel(s){
 if(inSel(s)){
  const L=UI.SELS.filter(x=>!same(x,s));
  setSel(L,L[L.length-1]);
 }else setSel(UI.SELS.concat([s]),s);
}
export function selectAll(){
 setSel(E.pickEnts());
 draw();
 return UI.SELS.length;
}
/* يُنادى بعد أي تبديل طبقة: ما صار مخفيّاً أو مقفلاً يخرج */
export function pruneSel(){
 const before=UI.SELS.length;
 const keep=UI.SELS.filter(pickable);
 if(keep.length!==before)
  setSel(keep,keep[keep.length-1]||null);
 return before-keep.length;
}
export function delSel(){
 const L=selList();
 if(!L.length)return null;
 const r=edit(()=>E.delEnts(L),"حذف");
 setSel([],null);
 draw();
 return r||null;
}
export function rebuildGrips(){
 UI.grips=[];
 if(!+S.rb.grips)return;
 if(!UI.SELS.length||UI.SELS.length>30)return;
 UI.SELS.forEach(s=>E.gripsOf(s).forEach(g=>UI.grips.push({...g,s})));
}
export function gripAt(px,py){
 let best=null,bd=8;
 UI.grips.forEach(g=>{
  const q=W2S(g.p[0],g.p[1]);
  const d=Math.hypot(q[0]-px,q[1]-py);
  if(d<bd){bd=d;best=g}
 });
 return best;
}
/* ═══ الرسم ═══ */
function drawGrid(){
 const st=Math.max(1,S.meta.snap);
 let s=st;
 while(s*V.k<9)s*=2;
 if(s*V.k<9)return;
 const a=S2W(0,V.h), b=S2W(V.w,0);
 const x0=Math.floor(a[0]/s)*s, x1=Math.ceil(b[0]/s)*s;
 const y0=Math.floor(a[1]/s)*s, y1=Math.ceil(b[1]/s)*s;
 if((x1-x0)/s>4000||(y1-y0)/s>4000)return;
 const P=pal();
 const big=s*10;
 ctx.save(); ctx.lineWidth=1;
 for(let x=x0;x<=x1;x+=s){
  const p=W2S(x,0);
  ctx.strokeStyle=(Math.abs(x%big)<1)?P.gMajor:P.gMinor;
  ctx.beginPath();ctx.moveTo(p[0],0);ctx.lineTo(p[0],V.h);ctx.stroke();
 }
 for(let y=y0;y<=y1;y+=s){
  const p=W2S(0,y);
  ctx.strokeStyle=(Math.abs(y%big)<1)?P.gMajor:P.gMinor;
  ctx.beginPath();ctx.moveTo(0,p[1]);ctx.lineTo(V.w,p[1]);ctx.stroke();
 }
 const o=W2S(0,0);
 ctx.strokeStyle=P.gAxis; ctx.lineWidth=1.4;
 ctx.beginPath();ctx.moveTo(0,o[1]);ctx.lineTo(V.w,o[1]);
 ctx.moveTo(o[0],0);ctx.lineTo(o[0],V.h);ctx.stroke();
 ctx.restore();
}
const PAT={};
function patFor(scr){
 const key=themeName()+"|"+Math.round(scr);
 if(PAT[key])return PAT[key];
 const s=clamp(Math.round(scr),4,90);
 const c=document.createElement("canvas");
 c.width=c.height=s;
 const x=c.getContext("2d");
 x.strokeStyle=pal().hatch; x.lineWidth=1;
 x.beginPath();x.moveTo(0,s);x.lineTo(s,0);x.stroke();
 PAT[key]=ctx.createPattern(c,"repeat");
 return PAT[key];
}
function lwOf(L){
 const w=resolve(L,themeName()).lw;
 return clamp((w||25)/100*V.k*S.meta.scale*0.5,0.7,6);
}
function paint(P){
 /* ١ — صبغة المناطق أسفل كل شيء */
 P.forEach(g=>{
  if(g.t!=="fill")return;
  const r=g.ring;
  if(!r||r.length<3)return;
  ctx.save(); ctx.beginPath();
  r.forEach((q,i)=>{
   const p=W2S(q[0],q[1]);
   if(i)ctx.lineTo(p[0],p[1]); else ctx.moveTo(p[0],p[1]);
  });
  ctx.closePath();
  ctx.fillStyle=(g.style==="hatch")
   ?patFor(Math.max(5,txtH()*2*V.k))
   :pal().tint;
  ctx.fill();
  ctx.restore();
 });
 /* ٢ — الهاشور */
 P.forEach(g=>{
  if(g.t!=="hatch")return;
  const loops=(g.loops||[]).filter(l=>l&&l.length>2);
  if(!loops.length)return;
  ctx.save(); ctx.beginPath();
  loops.forEach(lp=>{
   lp.forEach((q,i)=>{
    const p=W2S(q[0],q[1]);
    if(i)ctx.lineTo(p[0],p[1]); else ctx.moveTo(p[0],p[1]);
   });
   ctx.closePath();
  });
  ctx.fillStyle=(g.pat==="SOLID")?pal().solid
   :patFor(Math.max(4,(g.sc||300)*V.k));
  ctx.fill("evenodd");
  ctx.restore();
 });
 /* ٣ — الخطوط والنصوص */
 P.forEach(g=>{
  if(g.t==="hatch"||g.t==="fill")return;
  const L=g.L||"0";
  const R=resolve(L,themeName());
  const css=primCss(g);
  ctx.save();
  ctx.strokeStyle=css; ctx.fillStyle=css;
  ctx.lineWidth=(g.bad||g.warn)?1.6:lwOf(L);
  if(R.a<1)ctx.globalAlpha=R.a;
  /* دَشّ الأوّلية الصريح (تحذيرٌ · معطوبٌ · هامشٌ خفيف) يسبق دَشّ
     الطبقة — وإلّا فشُرَط الطبقة بالمليمتر الورقي، كارتفاع النصّ
     تماماً، تُضرَب بالمقياس ثم بمقياس الشاشة. */
  const dash=g.dash
   ? g.dash.map(v=>Math.max(1,v*V.k))
   : (R.dash.length
      ? R.dash.map(v=>Math.max(0.5,v*S.meta.scale*V.k))
      : []);
  ctx.setLineDash(dash);
  if(g.t==="line"){
   const a=W2S(g.a[0],g.a[1]), b=W2S(g.b[0],g.b[1]);
   ctx.beginPath();ctx.moveTo(a[0],a[1]);ctx.lineTo(b[0],b[1]);
   ctx.stroke();
  }else if(g.t==="poly"){
   if(!g.pts||g.pts.length<2){ctx.restore();return}
   ctx.beginPath();
   g.pts.forEach((q,i)=>{
    const p=W2S(q[0],q[1]);
    if(i)ctx.lineTo(p[0],p[1]); else ctx.moveTo(p[0],p[1]);
   });
   if(g.cl!==0)ctx.closePath();
   ctx.stroke();
  }else if(g.t==="arc"){
   const c=W2S(g.cx,g.cy), r=g.r*V.k;
   if(r<0.4){ctx.restore();return}
   /* Y مقلوب: الزوايا تُعكس */
   ctx.beginPath();
   ctx.arc(c[0],c[1],r,-g.a1*Math.PI/180,-g.a0*Math.PI/180);
   ctx.stroke();
  }else if(g.t==="text"){
   const px=g.h*V.k;
   if(px<4.5){ctx.restore();return}
   const p=W2S(g.x,g.y);
   ctx.translate(p[0],p[1]);
   const rt=deg(g.rot||0);
   if(rt)ctx.rotate(-rt*Math.PI/180);
   ctx.font=`${px.toFixed(1)}px Tahoma,Arial`;
   ctx.direction="rtl";
   ctx.textAlign=/l$/.test(g.al||"")?"left"
    :(/r$/.test(g.al||"")?"right":"center");
   ctx.textBaseline=/^m/.test(g.al||"")?"middle":"alphabetic";
   ctx.setLineDash([]);
   ctx.fillText(String(g.s),0,0);
  }
  ctx.restore();
 });
 ctx.setLineDash([]);
}
/* المسار المرسوم — خفيفاً ليُرى الفرق بينه وبين الجسم */
function drawPaths(){
 if(V.k<0.006)return;
 if(!vis("A-WALL")&&!vis("A-WALL-LOW"))return;
 ctx.save();
 ctx.setLineDash([4,4]); ctx.lineWidth=1;
 ctx.strokeStyle=pal().path;
 S.walls.forEach(w=>{
  if(w.align==="c")return;
  if(!vis(isLow(w)?"A-WALL-LOW":"A-WALL"))return;
  const a=W2S(w.a[0],w.a[1]), b=W2S(w.b[0],w.b[1]);
  ctx.beginPath();ctx.moveTo(a[0],a[1]);ctx.lineTo(b[0],b[1]);
  ctx.stroke();
 });
 ctx.restore();
}
/* معيَّن أحمر على الطرف غير المتّصل — علامة لا تعديل */
function drawEnds(){
 if(!+S.rb.ends)return;
 if(!vis("A-WALL")&&!vis("A-WALL-LOW"))return;
 const L=looseEnds(2);
 if(!L.length||L.length>400)return;
 ctx.save();
 ctx.strokeStyle=pal().loose; ctx.lineWidth=1.4;
 L.forEach(e=>{
  const p=W2S(e.p[0],e.p[1]), r=4.5;
  ctx.beginPath();
  ctx.moveTo(p[0],p[1]-r);ctx.lineTo(p[0]+r,p[1]);
  ctx.lineTo(p[0],p[1]+r);ctx.lineTo(p[0]-r,p[1]);
  ctx.closePath();ctx.stroke();
 });
 ctx.restore();
}
/* الإبراز قبل النقر — يوفّر نقرةً خاطئة، خصوصاً أن الإصابة تفضّل
   الأصغر فقد لا يكون ما تظنّه. لا يُرسَم لما هو محدَّد سلفاً. */
function drawPre(){
 const s=UI.pre;
 if(!s)return;
 if(UI.SELS.some(x=>x.k===s.k&&x.id===s.id))return;
 const poly=E.outlineOf(s), sh=poly?null:E.shapeOf(s);
 const PC=pal();
 ctx.save(); ctx.setLineDash([]);
 ctx.strokeStyle=PC.pre; ctx.lineWidth=2.6;
 const path=pts=>{
  ctx.beginPath();
  pts.forEach((q,i)=>{
   const t=W2S(q[0],q[1]);
   if(i)ctx.lineTo(t[0],t[1]); else ctx.moveTo(t[0],t[1]);
  });
 };
 if(poly&&poly.length>2){path(poly); ctx.closePath(); ctx.stroke()}
 else if(sh&&sh.t==="seg"){path([sh.a,sh.b]); ctx.stroke()}
 else if(sh&&sh.t==="poly"&&sh.pts&&sh.pts.length>2){
  path(sh.pts); ctx.closePath(); ctx.stroke();
 }else if(sh&&sh.t==="pt"){
  const c=W2S(sh.p[0],sh.p[1]);
  ctx.beginPath(); ctx.arc(c[0],c[1],9,0,7); ctx.stroke();
 }
 const P=UI.preP;
 if(P){
  const nm=`${s.id} ${E.NAME[s.k]||s.k}`;
  ctx.font="11px Tahoma"; ctx.direction="rtl";
  ctx.textAlign="left"; ctx.textBaseline="top";
  const w=ctx.measureText(nm).width+9;
  ctx.fillStyle=PC.chip;
  ctx.fillRect(P[0]+13,P[1]-19,w,16);
  ctx.strokeStyle=PC.preLn; ctx.lineWidth=1;
  ctx.strokeRect(P[0]+13,P[1]-19,w,16);
  ctx.fillStyle=PC.preTx;
  ctx.fillText(nm,P[0]+17,P[1]-16);
 }
 ctx.restore();
}
function drawSel(){
 if(!UI.SELS.length)return;
 const P=pal();
 ctx.save(); ctx.setLineDash([]);
 UI.SELS.forEach(s=>{
  const prim=same(s,UI.SEL);
  ctx.strokeStyle=prim?P.sel:P.sel2;
  ctx.lineWidth=prim?2.2:1.5;
  const poly=E.outlineOf(s);
  if(poly&&poly.length>2){
   ctx.beginPath();
   poly.forEach((q,i)=>{
    const t=W2S(q[0],q[1]);
    if(i)ctx.lineTo(t[0],t[1]); else ctx.moveTo(t[0],t[1]);
   });
   ctx.closePath(); ctx.stroke();
   return;
  }
  const sh=E.shapeOf(s);
  if(!sh)return;
  if(sh.t==="seg"){
   const a=W2S(sh.a[0],sh.a[1]), b=W2S(sh.b[0],sh.b[1]);
   ctx.beginPath();ctx.moveTo(a[0],a[1]);ctx.lineTo(b[0],b[1]);
   ctx.stroke();
  }else if(sh.t==="pt"){
   const c=W2S(sh.p[0],sh.p[1]);
   ctx.beginPath();ctx.arc(c[0],c[1],10,0,7);ctx.stroke();
  }else if(sh.t==="poly"&&sh.pts&&sh.pts.length>2){
   ctx.beginPath();
   sh.pts.forEach((q,i)=>{
    const t=W2S(q[0],q[1]);
    if(i)ctx.lineTo(t[0],t[1]); else ctx.moveTo(t[0],t[1]);
   });
   ctx.closePath(); ctx.stroke();
  }
 });
 ctx.restore();
 drawGrips();
}
function drawGrips(){
 if(!UI.grips.length)return;
 const P=pal();
 ctx.save(); ctx.setLineDash([]); ctx.lineWidth=1.2;
 UI.grips.forEach(g=>{
  const q=W2S(g.p[0],g.p[1]);
  const hot=UI.hot&&UI.hot.s.id===g.s.id&&UI.hot.k===g.k;
  const hov=!hot&&UI.hov&&UI.hov.s.id===g.s.id&&UI.hov.k===g.k;
  const r=(hot||hov)?5:3.5;
  ctx.fillStyle=hot?P.gripHot:(hov?P.gripHov:P.grip);
  ctx.strokeStyle=P.gripLn;
  ctx.fillRect(q[0]-r,q[1]-r,r*2,r*2);
  ctx.strokeRect(q[0]-r,q[1]-r,r*2,r*2);
 });
 ctx.restore();
}
function drawTrack(){
 const t=UI.track;
 if(!t)return;
 const L=9e5, r=t.a*Math.PI/180;
 const b0=W2S(t.from[0]-Math.cos(r)*L, t.from[1]-Math.sin(r)*L);
 const b1=W2S(t.from[0]+Math.cos(r)*L, t.from[1]+Math.sin(r)*L);
 const P=pal();
 ctx.save();
 ctx.strokeStyle=(Math.abs(t.a%90)<0.01)?P.trkO:P.trkP;
 ctx.setLineDash([5,5]); ctx.lineWidth=1;
 ctx.beginPath();ctx.moveTo(b0[0],b0[1]);ctx.lineTo(b1[0],b1[1]);
 ctx.stroke();
 ctx.restore();
}
function drawOsnap(){
 const o=UI.osnap;
 if(!o)return;
 const s=W2S(o.p[0],o.p[1]), r=6.5;
 const mk=(MODES.find(m=>m.k===o.m)||{}).mk;
 const P=pal();
 ctx.save();
 ctx.strokeStyle=P.snap; ctx.fillStyle=P.snap;
 ctx.lineWidth=1.8; ctx.setLineDash([]);
 ctx.beginPath();
 if(mk==="sq")ctx.rect(s[0]-r,s[1]-r,r*2,r*2);
 else if(mk==="tri"){
  ctx.moveTo(s[0],s[1]-r-1);ctx.lineTo(s[0]+r+1,s[1]+r);
  ctx.lineTo(s[0]-r-1,s[1]+r);ctx.closePath();
 }else if(mk==="x"){
  ctx.moveTo(s[0]-r,s[1]-r);ctx.lineTo(s[0]+r,s[1]+r);
  ctx.moveTo(s[0]-r,s[1]+r);ctx.lineTo(s[0]+r,s[1]-r);
 }else if(mk==="plus"){
  ctx.rect(s[0]-r,s[1]-r,r*2,r*2);
  ctx.moveTo(s[0]-r,s[1]);ctx.lineTo(s[0]+r,s[1]);
  ctx.moveTo(s[0],s[1]-r);ctx.lineTo(s[0],s[1]+r);
 }else if(mk==="per"){
  ctx.moveTo(s[0]-r,s[1]-r);ctx.lineTo(s[0]-r,s[1]+r);
  ctx.lineTo(s[0]+r,s[1]+r);
  ctx.moveTo(s[0]-r,s[1]);ctx.lineTo(s[0],s[1]);
  ctx.lineTo(s[0],s[1]+r);
 }else if(mk==="ref"){
  /* المرجع: معيَّن مجوَّف — يُميّزه عن نقاط رسمك */
  ctx.moveTo(s[0],s[1]-r);ctx.lineTo(s[0]+r,s[1]);
  ctx.lineTo(s[0],s[1]+r);ctx.lineTo(s[0]-r,s[1]);
  ctx.closePath();
 }else{
  ctx.moveTo(s[0]-r,s[1]-r);ctx.lineTo(s[0]+r,s[1]-r);
  ctx.lineTo(s[0]-r,s[1]+r);ctx.lineTo(s[0]+r,s[1]+r);
  ctx.closePath();
 }
 ctx.stroke();
 const nm=MNAME[o.m]||"";
 ctx.font="11px Tahoma";
 ctx.direction="rtl";
 ctx.textAlign="left"; ctx.textBaseline="top";
 const w=ctx.measureText(nm).width+9;
 ctx.fillStyle=P.chip;
 ctx.fillRect(s[0]+r+4,s[1]+r+2,w,16);
 ctx.strokeStyle=P.snapLn; ctx.lineWidth=1;
 ctx.strokeRect(s[0]+r+4,s[1]+r+2,w,16);
 ctx.fillStyle=P.snap;
 ctx.fillText(nm,s[0]+r+8,s[1]+r+5);
 ctx.restore();
}
function drawPreview(){
 const PL=pal();
 if(UI.free&&UI.free.length>1){
  ctx.save(); ctx.setLineDash([]); ctx.lineWidth=1.8;
  ctx.strokeStyle=PL.grip;
  ctx.beginPath();
  UI.free.forEach((q,i)=>{
   const t=W2S(q[0],q[1]);
   if(i)ctx.lineTo(t[0],t[1]); else ctx.moveTo(t[0],t[1]);
  });
  ctx.stroke(); ctx.restore();
 }
 (UI.guides||[]).forEach(g=>{
  const a=W2S(g[0][0],g[0][1]), b=W2S(g[1][0],g[1][1]);
  ctx.save(); ctx.setLineDash([3,4]); ctx.lineWidth=1;
  ctx.strokeStyle=PL.guide;
  ctx.beginPath();ctx.moveTo(a[0],a[1]);ctx.lineTo(b[0],b[1]);
  ctx.stroke();
  ctx.restore();
 });
 drawTrack();
 if(UI.marq){
  const a=W2S(UI.marq.a[0],UI.marq.a[1]);
  const b=W2S(UI.marq.b[0],UI.marq.b[1]);
  const win=UI.marq.win;
  ctx.save();
  ctx.setLineDash(win?[]:[6,4]);
  ctx.strokeStyle=win?PL.win:PL.cross;
  ctx.fillStyle=win?PL.winF:PL.crossF;
  ctx.lineWidth=1.2;
  const x=Math.min(a[0],b[0]), y=Math.min(a[1],b[1]);
  ctx.fillRect(x,y,Math.abs(b[0]-a[0]),Math.abs(b[1]-a[1]));
  ctx.strokeRect(x,y,Math.abs(b[0]-a[0]),Math.abs(b[1]-a[1]));
  ctx.setLineDash([]);
  ctx.font="11px Tahoma"; ctx.direction="rtl";
  ctx.fillStyle=ctx.strokeStyle;
  ctx.textAlign="left"; ctx.textBaseline="top";
  ctx.fillText(win?"نافذة — احتواء كامل":"قطع — تلامس",x+5,y+4);
  ctx.restore();
 }
 const P=R.preview();
 if(P.length){
  ctx.save(); ctx.lineWidth=1.6;
  P.forEach(g=>{
   const col=g.c||PL.grip;
   ctx.strokeStyle=col; ctx.setLineDash([6,4]);
   if(g.t==="l"){
    const a=W2S(g.a[0],g.a[1]), b=W2S(g.b[0],g.b[1]);
    ctx.beginPath();ctx.moveTo(a[0],a[1]);ctx.lineTo(b[0],b[1]);
    ctx.stroke();
    const L=Math.hypot(g.b[0]-g.a[0],g.b[1]-g.a[1]);
    if(L*V.k>30){
     const t=(L/1000).toFixed(2);
     ctx.setLineDash([]);
     ctx.font="12px Consolas,monospace";
     ctx.direction="ltr";
     ctx.textAlign="center"; ctx.textBaseline="bottom";
     const mx=(a[0]+b[0])/2, my=(a[1]+b[1])/2;
     const w=ctx.measureText(t).width+8;
     ctx.fillStyle=PL.chip2;
     ctx.fillRect(mx-w/2,my-17,w,16);
     ctx.fillStyle=col;
     ctx.fillText(t,mx,my-4);
    }
   }else if(g.t==="r"){
    const a=W2S(g.a[0],g.a[1]), b=W2S(g.b[0],g.b[1]);
    ctx.strokeRect(Math.min(a[0],b[0]),Math.min(a[1],b[1]),
     Math.abs(b[0]-a[0]),Math.abs(b[1]-a[1]));
    ctx.setLineDash([]);
    ctx.font="12px Consolas,monospace";
    ctx.direction="ltr";
    ctx.textAlign="left"; ctx.fillStyle=col;
    ctx.fillText(
     `${(Math.abs(g.b[0]-g.a[0])/1000).toFixed(2)} × `
     +`${(Math.abs(g.b[1]-g.a[1])/1000).toFixed(2)} م`,
     Math.min(a[0],b[0])+6, Math.min(a[1],b[1])-7);
   }else if(g.t==="b"){
    /* معاينة جسم الجدار بسماكته */
    const dx=g.b[0]-g.a[0], dy=g.b[1]-g.a[1];
    const L=Math.hypot(dx,dy);
    if(L<1)return;
    const nx=-dy/L*(g.w/2), ny=dx/L*(g.w/2);
    const Q=[[g.a[0]+nx,g.a[1]+ny],[g.b[0]+nx,g.b[1]+ny],
             [g.b[0]-nx,g.b[1]-ny],[g.a[0]-nx,g.a[1]-ny]];
    ctx.setLineDash([]); ctx.lineWidth=1.1;
    ctx.strokeStyle=PL.cross;
    ctx.beginPath();
    Q.forEach((q,i)=>{
     const t=W2S(q[0],q[1]);
     if(i)ctx.lineTo(t[0],t[1]); else ctx.moveTo(t[0],t[1]);
    });
    ctx.closePath(); ctx.stroke();
   }else if(g.t==="pg"){
    if(!g.pts||g.pts.length<2)return;
    ctx.beginPath();
    g.pts.forEach((q,i)=>{
     const t=W2S(q[0],q[1]);
     if(i)ctx.lineTo(t[0],t[1]); else ctx.moveTo(t[0],t[1]);
    });
    if(g.cl!==0)ctx.closePath();
    ctx.stroke();
   }else if(g.t==="tx"){
    const t=W2S(g.p[0],g.p[1]);
    ctx.setLineDash([]);
    ctx.font="12px Consolas,monospace";
    ctx.direction="rtl"; ctx.textAlign="center";
    ctx.textBaseline="middle";
    const w=ctx.measureText(g.s).width+10;
    ctx.fillStyle=PL.chip2;
    ctx.fillRect(t[0]-w/2,t[1]-9,w,18);
    ctx.fillStyle=col;
    ctx.fillText(String(g.s),t[0],t[1]);
   }
  });
  ctx.restore();
 }
 if(R.T.ghost){
  const s=W2S(R.T.ghost[0],R.T.ghost[1]);
  ctx.save(); ctx.setLineDash([]);
  ctx.strokeStyle=PL.ghost; ctx.lineWidth=1.5;
  ctx.beginPath();ctx.arc(s[0],s[1],5,0,7);ctx.stroke();
  ctx.restore();
 }
 if(UI.zwin){
  const a=W2S(UI.zwin.a[0],UI.zwin.a[1]);
  const b=W2S(UI.zwin.b[0],UI.zwin.b[1]);
  ctx.save();
  ctx.setLineDash([5,4]); ctx.lineWidth=1.3;
  ctx.strokeStyle=PL.win; ctx.fillStyle=PL.winF;
  const x=Math.min(a[0],b[0]), y=Math.min(a[1],b[1]);
  ctx.fillRect(x,y,Math.abs(b[0]-a[0]),Math.abs(b[1]-a[1]));
  ctx.strokeRect(x,y,Math.abs(b[0]-a[0]),Math.abs(b[1]-a[1]));
  ctx.restore();
 }
 drawOsnap();
}
function drawScaleBar(){
 const want=110/V.k;
 const pow=Math.pow(10,Math.floor(Math.log10(want)));
 let s=pow;
 [1,2,5,10].some(f=>{if(pow*f>=want){s=pow*f;return true}return false});
 const px=s*V.k;
 if(px<20||px>V.w*0.6)return;
 const x=V.w-px-16, y=V.h-16;
 const P=pal();
 ctx.save();
 ctx.strokeStyle=P.bar; ctx.fillStyle=P.bar;
 ctx.lineWidth=1.4; ctx.setLineDash([]);
 ctx.beginPath();
 ctx.moveTo(x,y-5);ctx.lineTo(x,y);ctx.lineTo(x+px,y);
 ctx.lineTo(x+px,y-5);
 ctx.stroke();
 ctx.font="10.5px Tahoma";
 ctx.textAlign="center"; ctx.textBaseline="bottom";
 ctx.direction="rtl";
 ctx.fillText((s/1000)+" م",x+px/2,y-7);
 ctx.direction="ltr";
 ctx.textAlign="right";
 ctx.fillText(scl(S.meta.scale),V.w-16,y-22);
 ctx.direction="rtl";
 ctx.restore();
}
let RQ=false;
export function draw(){
 bump("frame");
 if(RQ)return;
 RQ=true;
 requestAnimationFrame(()=>{RQ=false;paintAll()});
}
function paintAll(){
 const P=pal();
 ctx.clearRect(0,0,V.w,V.h);
 ctx.fillStyle=P.bg; ctx.fillRect(0,0,V.w,V.h);
  drawUnderlay(ctx,p=>W2S(p[0],p[1]),V.k);
 if(+S.rb.grid)drawGrid();
 let sc=null;
 try{sc=scene()}
 catch(e){
  ctx.fillStyle=P.er; ctx.font="13px Tahoma";
  ctx.direction="rtl";
  ctx.textAlign="left"; ctx.textBaseline="top";
  ctx.fillText("تعذّر بناء المشهد: "+e.message,12,12);
  return;
 }
 paint(sc.P);
 const bg=blockGhost();
 if(bg)drawInstance(ctx,bg,p=>W2S(p[0],p[1]),{ghost:true});
 if(+S.rb.paths)drawPaths();
 drawEnds();
 drawPre();
 rebuildGrips();
 drawSel();
 drawPreview();
 drawScaleBar();
}
/* ═══ الفأرة ═══ */
let panning=null, drag=null;
const GN={a:"البداية",b:"النهاية",mid:"الجسم",
 c:"المركز",e0:"الحدّ الأول",e1:"الحدّ الثاني",
 L:"التسمية",pos:"موضع الخطّ",base:"الأساس",end:"النهاية",
 p:"الموضع",r:"القطر",sz:"المقاس",rot:"الدوران",w:"العرض"};
const gname=k=>GN[k]||(/^v\d+$/.test(k)?"رأس":
 (/^p\d+$/.test(k)?"نقطة":k));

function onDown(e){
 cv.focus(); UI.pre=null;
 /* الزرّ الأيمن يحرّك في وضع «Enter» وحده — وفي وضع القائمة لا
    يجوز التمييز بين نقرةٍ وسحبةٍ لأن contextmenu يقع قبل الحركة
    أو بعدها بحسب النظام. */
 if(e.button===1||(e.button===2&&UIS.rclick==="enter")){
  panning=[e.clientX,e.clientY];
  return;
 }
 if(e.button!==0)return;
 if(NAV.mode==="pan"){
  panning=[e.clientX,e.clientY];
  cv.style.cursor="grabbing";
  return;
 }
 if(NAV.mode==="zw"){
  const p=S2W(e.offsetX,e.offsetY);
  drag={zwin:true,a:p};
  UI.zwin={a:p,b:p};
  draw();
  return;
 }
 const raw=S2W(e.offsetX,e.offsetY);
  if(isInserting()){
   onBlockClick(raw);
   return;
  }
 if(R.active()){
  const stp=R.step();
  if(stp&&stp.freehand){
   drag={free:[[Math.round(raw[0]),Math.round(raw[1])]]};
   UI.free=drag.free;
   draw(); return;
  }
  R.feedPoint(snap(raw[0],raw[1],R.baseOf()));
  HOOK.prompt();
  return;
 }
 const gr=gripAt(e.offsetX,e.offsetY);
 if(gr){
  UI.hot=gr;
  drag={grip:gr,start:snap(raw[0],raw[1],null),moved:false,
   snap:snapshot(),o:E.grabOf(gr.s)};
  HOOK.status(`مقبض ${gname(gr.k)} من ${gr.s.id}`);
  draw(); return;
 }
 const hit=hitTest(raw[0],raw[1]);
 if(!hit){
  const p=snap(raw[0],raw[1],null);
  drag={marq:true,a:p,add:e.shiftKey};
  UI.marq={a:p,b:p,win:true};
  if(!e.shiftKey)setSel([],null);
  draw(); return;
 }
 if(e.shiftKey){toggleSel(hit);draw();return}
 const multi=inSel(hit)&&UI.SELS.length>1;
 if(!multi)setSel([hit],hit);
 else{UI.SEL=hit;HOOK.props()}
 const p=snap(raw[0],raw[1],null);
 /* الحرس الأخير قبل التحويل: ما لا يُحدَّد لا يُعدَّل ولا يُحرَّك،
    ولو وصل إلى التحديد من مسارٍ لا يصفّي. */
 const L2=UI.SELS.filter(pickable);
 if(L2.length<UI.SELS.length)
  HOOK.status(`${UI.SELS.length-L2.length} عنصراً مخفيّاً أو `
   +`مقفلاً لن يتحرّك`);
 drag={move:1,start:p,moved:false,snap:snapshot(),
  list:L2.map(s=>({s,o:E.grabOf(s)})).filter(x=>x.o)};
 draw();
}
function onMove(e){
 if(panning){
  panBy(e.clientX-panning[0], e.clientY-panning[1]);
  panning=[e.clientX,e.clientY];
  return;
 }
 const raw=S2W(e.offsetX,e.offsetY);
 const st=document.getElementById("stPos");
 if(st)st.textContent=`${m2(raw[0])} , ${m2(raw[1])} م`;
 UI.pre=null; UI.preP=[e.offsetX,e.offsetY];
  if(isInserting()){
   onBlockMove(raw);
   return;
  }
 if(drag&&drag.zwin){
  UI.zwin={a:drag.a,b:raw};
  draw(); return;
 }
 if(drag&&drag.free){
  const q=drag.free[drag.free.length-1];
  if(Math.hypot(raw[0]-q[0],raw[1]-q[1])>Math.max(15,2.5/V.k))
   drag.free.push([Math.round(raw[0]),Math.round(raw[1])]);
  draw(); return;
 }

 if(drag&&drag.marq){
  const p=snap(raw[0],raw[1],null);
  UI.marq={a:drag.a,b:p,win:p[0]>=drag.a[0]};
  draw(); return;
 }
 if(drag&&drag.grip){
  const base=/^(mid|c|p|L|base)$/.test(drag.grip.k)
   ? drag.start : null;
  const p=snap(raw[0],raw[1],base);
  if(!drag.moved){
   pushHistory(drag.snap,"تعديل مقبض"); drag.moved=true;
   /* النسخة تُحسَب مرّةً عند بدء السحب لا في كل إطار: سحبُ بُعدٍ
      أو نصٍّ لا يُبطِل اتحاد الأجسام ولا الحلقات ولا البصمات */
   drag.tf=E.touchFn(E.bumpOf([drag.grip.s]));
  }
  E.dragGrip(drag.grip,drag.o,p,
   p[0]-drag.start[0],p[1]-drag.start[1]);
  (drag.tf||touch)();
  draw(); return;
 }
 if(drag&&drag.move){
  const p=snap(raw[0],raw[1],null);
  const dx=p[0]-drag.start[0], dy=p[1]-drag.start[1];
  if(!drag.moved){
   if(Math.hypot(dx,dy)<Math.max(2,4/V.k))return;
   pushHistory(drag.snap,"تحريك"); drag.moved=true;
   drag.tf=E.touchFn(E.bumpOf(drag.list.map(x=>x.s)));
  }
  drag.list.forEach(({s,o})=>E.moveEnt(s,o,dx,dy));
  (drag.tf||touch)();
  draw(); return;
 }
 if(R.active()){
  const st=R.step();
  /* خطوة تنتظر عنصراً ⇒ أبرِز المرشَّح تحت المؤشّر قبل النقر */
  if(st&&st.ent)UI.pre=E.hitTest(raw[0],raw[1],14/V.k);
  R.T.ghost=snap(raw[0],raw[1],R.baseOf());
  HOOK.prompt(); draw(); return;
 }
 UI.hov=gripAt(e.offsetX,e.offsetY);
 if(!UI.hov)UI.pre=hitTest(raw[0],raw[1]);
 R.T.ghost=null;
 snap(raw[0],raw[1],null);
 cv.style.cursor=NAV.mode?(NAV.mode==="pan"?"grab":"crosshair")
  :(UI.hov?"pointer":(UI.pre?"pointer":"crosshair"));
 draw();
}
function onUp(){
 if(drag&&drag.zwin){
  const a=drag.a, b=UI.zwin?UI.zwin.b:a;
  drag=null; UI.zwin=null;
  const r={x0:Math.min(a[0],b[0]),y0:Math.min(a[1],b[1]),
           x1:Math.max(a[0],b[0]),y1:Math.max(a[1],b[1])};
  if((r.x1-r.x0)*V.k>8&&(r.y1-r.y0)*V.k>8)fitBox(r,0.02);
  navSet(null);
  HOOK.status("");
  draw(); return;
 }
 if(NAV.mode==="pan"&&panning){
  panning=null; cv.style.cursor="grab"; return;
 }
 if(drag&&drag.free){
  const P=drag.free;
  drag=null; UI.free=null;
  if(P.length>1)R.feedStroke(P);
  HOOK.prompt(); draw(); return;
 }
 if(drag&&drag.marq&&UI.marq){
  const a=UI.marq.a, b=UI.marq.b;
  const r={x0:Math.min(a[0],b[0]),y0:Math.min(a[1],b[1]),
           x1:Math.max(a[0],b[0]),y1:Math.max(a[1],b[1])};
  if((r.x1-r.x0)*V.k>5||(r.y1-r.y0)*V.k>5){
   const out=E.pickInRect(r,UI.marq.win,drag.add,UI.SELS);
   setSel(out,out[out.length-1]);
   HOOK.status(out.length?`${out.length} عنصر محدد`
    :"لا عنصر داخل الإطار");
  }
  UI.marq=null; drag=null; draw(); return;
 }
 if(drag&&(drag.grip||drag.move)){
  UI.hot=null;
  if(drag.moved){
   jrTaint(drag.grip?"سحب مقبض":"سحب مباشر");
   touch();HOOK.refresh();autosave();
  }
  drag=null; rebuildGrips(); draw(); return;
 }
 drag=null; panning=null;
}
/* التسجيل بعد التعريف — مسارٌ واحد للفأرة واللمس والقلم */
cv.addEventListener("mousedown",onDown);
cv.addEventListener("mousemove",onMove);
addEventListener("mouseup",onUp);

/* ═══ اللمس والقلم ═══
   اللوح هو الجهاز الطبيعي لرسم مخطّطٍ في الموقع، ولم يكن يعمل:
   كل التفاعل أحداث فأرة. وPointer Events تُغذّي المعالِجات نفسها
   فلا منطق ثانٍ يتخلّف عن الأول.

   الإصبع الواحد يرسم ويحدّد كالفأرة تماماً. والإصبعان تنقّلٌ لا
   رسم: يُلغيان ما بدأه الأول لأن نيّتهما التكبير والتحريك.
   والضغط المطوّل بديلُ الزرّ الأيمن — فلا قائمة سياق بلا فأرة. */
const PT=new Map();
let gest=null, lp=null;
const pdist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);
const pmid=(a,b)=>[(a.x+b.x)/2,(a.y+b.y)/2];

function abortDrag(){
 if(drag&&drag.moved){onUp(); return}   /* ما تحرّك يُنهى نظامياً */
 drag=null; panning=null;
 UI.marq=null; UI.free=null; UI.zwin=null;
 draw();
}
const lpClear=()=>{if(lp){clearTimeout(lp.t); lp=null}};
function lpStart(e){
 lpClear();
 const x=e.clientX, y=e.clientY;
 lp={x,y,t:setTimeout(()=>{
  lp=null;
  if(drag&&drag.moved)return;          /* سحبٌ جارٍ لا ضغطة */
  abortDrag();
  if(HOOK.ctx)HOOK.ctx(x,y);
 },550)};
}

cv.addEventListener("pointerdown",e=>{
 if(e.pointerType==="mouse")return;     /* للفأرة معالجُها */
 e.preventDefault();
 if(cv.setPointerCapture)cv.setPointerCapture(e.pointerId);
 PT.set(e.pointerId,{x:e.offsetX,y:e.offsetY});
 if(PT.size===1){
  cv.focus();
  if(!R.active())lpStart(e);            /* لا قائمة وسط أداة */
  onDown(e);
  return;
 }
 if(PT.size===2){
  lpClear();
  abortDrag();
  const [a,b]=[...PT.values()];
  gest={d:pdist(a,b), k:V.k, m:pmid(a,b)};
 }
},{passive:false});

cv.addEventListener("pointermove",e=>{
 if(e.pointerType==="mouse")return;
 const p=PT.get(e.pointerId);
 if(!p)return;
 e.preventDefault();
 p.x=e.offsetX; p.y=e.offsetY;
 if(lp&&Math.hypot(e.clientX-lp.x,e.clientY-lp.y)>8)lpClear();
 if(PT.size>=2&&gest){
  const [a,b]=[...PT.values()];
  const d=pdist(a,b), m=pmid(a,b);
  /* نسبةٌ مطلقة من بداية الإيماءة لا تراكمية: القرص ثم الفتح
     يعود إلى المقياس نفسه بلا انجراف */
  if(gest.d>8&&d>8){
   const f=(d/gest.d)*(gest.k/V.k);
   if(f>0.01&&Math.abs(f-1)>1e-4)zoomAt(m[0],m[1],f);
  }
  panBy(m[0]-gest.m[0], m[1]-gest.m[1]);
  gest.m=m;
  return;
 }
 onMove(e);
},{passive:false});

function ptEnd(e){
 if(e.pointerType==="mouse")return;
 PT.delete(e.pointerId);
 if(PT.size<2)gest=null;
 if(PT.size===0){lpClear(); onUp()}
}
cv.addEventListener("pointerup",ptEnd);
cv.addEventListener("pointercancel",ptEnd);

cv.addEventListener("contextmenu",e=>{
 e.preventDefault();
 const m=UIS.rclick||"auto";
 const wantMenu=(m==="menu")||(m==="auto"&&!R.active());
 if(wantMenu&&HOOK.ctx&&HOOK.ctx(e.clientX,e.clientY))return;
 if(R.active())R.enter();
 else if(R.T.last)R.begin(R.T.last);
 HOOK.prompt(); draw();
});
cv.addEventListener("wheel",e=>{
 e.preventDefault();
 /* قرص لوحة اللمس يصل ctrl+wheel — خطوةٌ أنعم من عجلة الفأرة */
 const f=e.ctrlKey?(1+Math.min(0.25,Math.abs(e.deltaY)*0.01))
                  :1.12;
 zoomAt(e.offsetX,e.offsetY,e.deltaY<0?f:1/f);
},{passive:false});
cv.addEventListener("dragover",e=>e.preventDefault());
cv.addEventListener("drop",e=>{
 e.preventDefault();
 const name=e.dataTransfer?.getData("application/x-mistar-block");
 if(!name)return;
 const p=S2W(e.offsetX,e.offsetY);
 import("../tools/blocks.js").then(({startInsert})=>{
  startInsert(name); onBlockMove(p); onBlockClick(p);
 });
});
cv.addEventListener("mouseleave",()=>{
 UI.pre=null; UI.hov=null; draw();
});
```

<a id="f-js-ui-cmdline-js"></a>

---

## `js/ui/cmdline.js`

```javascript
/* ═══ سطر الأوامر: الموضع والسجل ═══
   العُقَد تُنقَل ولا تُبنى — كعقد dock.js نفسه: #cmdWrap ينتقل بين
   أسفل اللوحة وأعلاها ونافذةٍ عائمة، فيحفظ مستمعيه وقيمة الحقل
   وموضع تمرير السجل. وإعادة بنائه كانت ستُفقِد ما يكتبه المستخدم
   وسط أمر. */
import {UIS,uiSet,saveUI} from "./store.js";
import {icon} from "./icons.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||14):"";
const rtl=()=>getComputedStyle(document.documentElement)
 .direction==="rtl";
const cl=(v,a,b)=>v<a?a:(v>b?b:v);

export const CMODES={bottom:"أسفل اللوحة",top:"أعلى اللوحة",
 float:"نافذة عائمة"};
export const LOGH={0:"بلا سجل",84:"منخفض",120:"متوسط",200:"مرتفع"};
export const OPAS={100:"معتم",88:"شفافية خفيفة",72:"شفافية أوسع"};

/* ═══ التطبيق ═══ */
export function applyCmd(){
 const w=$("#cmdWrap"), work=$("#work"), fl=$("#cmdFloat"),
       stage=$("#stage"), lg=$("#log");
 if(!w||!work)return;
 const m=CMODES[UIS.cmdMode]?UIS.cmdMode:"bottom";
 w.dataset.mode=m;
 if(m==="float"){
  fl.appendChild(w);
  placeCmd();
 }else if(m==="top"){
  work.insertBefore(w,stage);
 }else{
  work.appendChild(w);
 }
 if(lg){
  const h=LOGH[UIS.logH]!==undefined?+UIS.logH:120;
  lg.hidden=(h===0)||!!UIS.clean;
  if(h>0)lg.style.height=h+"px";
 }
 w.style.setProperty("--cmdOpa",String((+UIS.cmdOpa||100)/100));
 w.hidden=!!UIS.clean;
 dispatchEvent(new Event("resize"));
}
function placeCmd(){
 const w=$("#cmdWrap");
 if(!w||UIS.cmdMode!=="float")return;
 const W=innerWidth, H=innerHeight;
 UIS.cmdW=Math.round(cl(+UIS.cmdW||620,320,Math.max(360,W-40)));
 UIS.cmdX=Math.round(cl(+UIS.cmdX||24,-UIS.cmdW+120,Math.max(0,W-140)));
 UIS.cmdY=Math.round(cl(+UIS.cmdY||(H-220),40,Math.max(60,H-90)));
 w.style.insetInlineStart=UIS.cmdX+"px";
 w.style.insetBlockStart=UIS.cmdY+"px";
 w.style.inlineSize=UIS.cmdW+"px";
}
export function setCmdMode(m){
 if(!CMODES[m])return false;
 uiSet("cmdMode",m);
 applyCmd();
 HOOK.report("in",`سطر الأوامر: ${CMODES[m]}`);
 const c=$("#clIn");
 if(c)c.focus();
 return true;
}
export function setLogH(h){
 uiSet("logH",+h||0);
 applyCmd();
 HOOK.report("in",`السجل: ${LOGH[+h]||h+" بكسل"}`);
}
export function setCmdOpa(v){
 uiSet("cmdOpa",cl(Math.round(+v||100),40,100));
 applyCmd();
}
export function clearLog(){
 const lg=$("#log");
 if(lg)lg.innerHTML="";
 HOOK.report("in","فُرّغ السجل");
}
/* ═══ القائمة ═══ */
export function cmdMenu(x,y){
 const m=$("#cMenu");
 if(!m)return;
 const row=(a,n,i,on)=>`<button type="button" class="pmI" `
  +`data-cma="${esc(a)}">${ic(i)}<span>${esc(n)}</span>`
  +`<span class="ky">${on?"●":""}</span></button>`;
 m.innerHTML=`<div class="pmH">سطر الأوامر</div>`
  +Object.keys(CMODES).map(k=>row("m:"+k,CMODES[k],
   k==="float"?"float":(k==="top"?"up":"down"),
   UIS.cmdMode===k)).join("")
  +`<div class="pmS"></div>`
  +Object.keys(LOGH).map(k=>row("h:"+k,LOGH[k],"cmdl",
   String(UIS.logH)===k)).join("")
  +`<div class="pmS"></div>`
  +Object.keys(OPAS).map(k=>row("o:"+k,OPAS[k],"opa",
   String(UIS.cmdOpa)===k)).join("")
  +`<div class="pmS"></div>`
  +`<button type="button" class="pmI" data-cma="clr">`
  +`${ic("clear")}<span>فرّغ السجل</span></button>`;
 m.hidden=false;
 const w=m.offsetWidth||230, h=m.offsetHeight||330;
 m.style.insetInlineStart=Math.round(
  cl(rtl()?(innerWidth-x-4):(x-w+4),4,innerWidth-w-4))+"px";
 m.style.insetBlockStart=Math.round(cl(y-h-6,4,innerHeight-h-8))+"px";
}
export const cmdMenuClose=()=>{
 const m=$("#cMenu");
 if(m)m.hidden=true;
};
function cmdRun(a){
 cmdMenuClose();
 const m=/^m:(\w+)$/.exec(a);
 if(m)return setCmdMode(m[1]);
 const h=/^h:(\d+)$/.exec(a);
 if(h)return setLogH(h[1]);
 const o=/^o:(\d+)$/.exec(a);
 if(o)return setCmdOpa(o[1]);
 if(a==="clr")return clearLog();
}
/* ═══ التوصيل ═══ */
let DRG=null;
export function wireCmd(){
 const bar=$("#cmdline");
 if(!bar)return false;
 if(!$("#clMenuBtn"))bar.insertAdjacentHTML("beforeend",
  `<button type="button" id="clMenuBtn" title="موضع سطر الأوامر"`
  +` aria-label="خيارات سطر الأوامر">⋮</button>`);
 document.addEventListener("click",e=>{
  if(e.target.closest("#clMenuBtn")){
   const r=e.target.closest("#clMenuBtn").getBoundingClientRect();
   cmdMenu(rtl()?r.left:r.right,r.top);
   return;
  }
  const a=e.target.closest("[data-cma]");
  if(a)cmdRun(a.dataset.cma);
 });
 addEventListener("mousedown",e=>{
  const m=$("#cMenu");
  if(m&&!m.hidden&&!m.contains(e.target)
   &&!e.target.closest("#clMenuBtn"))cmdMenuClose();
  /* السحب في الوضع العائم: من الشريط لا من الحقل */
  if(UIS.cmdMode!=="float"||UIS.lockUI)return;
  const b=e.target.closest("#cmdline");
  if(!b||e.target.closest("input,button"))return;
  DRG={px:e.clientX,py:e.clientY,x:+UIS.cmdX||24,y:+UIS.cmdY||0,on:0};
  document.documentElement.classList.add("dragging");
 },true);
 addEventListener("mousemove",e=>{
  if(!DRG)return;
  const dx=e.clientX-DRG.px, dy=e.clientY-DRG.py;
  if(!DRG.on&&Math.hypot(dx,dy)<4)return;
  DRG.on=1;
  UIS.cmdX=Math.round(DRG.x+(rtl()?-dx:dx));
  UIS.cmdY=Math.round(DRG.y+dy);
  placeCmd();
 });
 addEventListener("mouseup",()=>{
  if(!DRG)return;
  const on=DRG.on;
  DRG=null;
  document.documentElement.classList.remove("dragging");
  if(on)saveUI();
 });
 addEventListener("keydown",e=>{
  const m=$("#cMenu");
  if(e.key==="Escape"&&m&&!m.hidden){
   e.preventDefault(); e.stopPropagation(); cmdMenuClose();
  }
 },true);
 addEventListener("resize",()=>placeCmd());
 if(typeof ResizeObserver!=="undefined"){
  const w=$("#cmdWrap");
  let t=null;
  new ResizeObserver(()=>{
   if(UIS.cmdMode!=="float")return;
   UIS.cmdW=Math.round(w.offsetWidth);
   if(t)clearTimeout(t);
   t=setTimeout(()=>{t=null; saveUI()},400);
  }).observe(w);
 }
 applyCmd();
 return true;
}
```

<a id="f-js-ui-cmdpalette-js"></a>

---

## `js/ui/cmdpalette.js`

```javascript
/* ═══ لوحة الأوامر ═══ بحثٌ ضبابيّ عربي/إنجليزي فوق سجلّ الأدوات
   الحقيقي (tools/registry.js: toolList/begin) — لا قائمة أوامر
   وهميّة موازية. تقبل أيضاً أفعالاً على مستوى التطبيق (تراجع،
   حفظ...) تُحقَن من app.js عبر initCmdPalette({extra}). */
import {toolList,begin,active,cancel} from "../tools/registry.js";
import {HOOK} from "./bus.js";

const ID="cmdPal";
let root=null,inputEl=null,listEl=null,mounted=false,extra=[],items=[],sel=0;

const nz=s=>String(s||"").toLowerCase().trim();
function score(q,name){
 q=nz(q); name=nz(name); if(!q)return 0;
 let i=0,sc=0,run=0;
 for(let j=0;j<name.length&&i<q.length;j++){
  if(name[j]===q[i]){i++; run++; sc+=2+run; if(j===0)sc+=4;}
  else run=0;
 }
 return i===q.length?sc-(name.length-q.length):-1;
}
function buildIndex(){
 const fromTools=toolList().map(d=>({
  id:"tool:"+d.id, label:d.label||d.id, hint:d.hint||"",
  names:[d.label,...(String(d.alias||"").split(/\s+/).filter(Boolean))],
  run:()=>{ if(active())cancel(true); begin(d.id); }
 }));
 items=fromTools.concat(extra.map(x=>({
  id:"ext:"+x.id, label:x.label, hint:x.hint||"",
  names:[x.label,...(x.aliases||[])], run:x.run
 })));
}
function suggest(q,limit=9){
 if(!nz(q))return items.slice(0,limit);
 const out=[];
 for(const it of items){
  let best=-1;
  for(const n of it.names)best=Math.max(best,score(q,n));
  if(best>=0)out.push({...it,score:best});
 }
 return out.sort((a,b)=>b.score-a.score).slice(0,limit);
}

function injectCss(){
 if(document.getElementById("cp-css"))return;
 const css=`
#${ID}-ov{position:fixed; inset:0; z-index:70; display:flex; align-items:flex-start;
 justify-content:center; padding-block-start:min(14vh,140px); background:rgba(0,0,0,.4)}
#${ID}-ov[hidden]{display:none}
#${ID}{width:min(440px,92vw); max-height:60vh; display:flex; flex-direction:column;
 background:var(--bg2,#1a1d21); color:var(--fg,#e9edf2);
 border:1px solid var(--ln,#2a2f36); border-radius:var(--r3,10px);
 box-shadow:var(--sh3,0 18px 48px rgba(0,0,0,.5)); overflow:hidden;
 font:13px/1.5 var(--ui,system-ui,sans-serif)}
#${ID} input{width:100%; box-sizing:border-box; padding:11px 12px; font-size:14px;
 background:var(--bg3,#20242a); color:var(--fg,#e9edf2); border:0;
 border-bottom:1px solid var(--ln,#2a2f36); outline:0}
#${ID} .cp-list{list-style:none; margin:0; padding:4px; overflow-y:auto}
#${ID} .cp-it{display:flex; align-items:baseline; gap:8px; padding:7px 10px;
 border-radius:6px; cursor:pointer; color:var(--fg2,#c4ccd6)}
#${ID} .cp-it .lbl{font-weight:600; color:inherit}
#${ID} .cp-it .hint{flex:1; text-align:end; color:var(--fg3,#8a929c); font-size:11.5px;
 white-space:nowrap; overflow:hidden; text-overflow:ellipsis}
#${ID} .cp-it.sel{background:var(--acq,rgba(110,168,254,.18)); color:var(--acf,#fff)}
#${ID} .cp-empty{padding:18px 12px; text-align:center; color:var(--fg3,#8a929c)}`;
 const st=document.createElement("style"); st.id="cp-css"; st.textContent=css;
 document.head.appendChild(st);
}
function build(){
 injectCss();
 let ov=document.getElementById(ID+"-ov");
 if(!ov){
  ov=document.createElement("div"); ov.id=ID+"-ov"; ov.hidden=true; ov.setAttribute("dir","rtl");
  ov.innerHTML=
   `<div id="${ID}" role="dialog" aria-label="لوحة الأوامر">
      <input type="text" placeholder="اكتب اسم أداة أو أمراً… (خط، جدار، move)"
       spellcheck="false" autocomplete="off">
      <ul class="cp-list"></ul>
    </div>`;
  document.body.appendChild(ov);
  ov.addEventListener("mousedown",e=>{ if(e.target===ov)close(); });
 }
 root=ov; inputEl=root.querySelector("input"); listEl=root.querySelector(".cp-list");
 inputEl.addEventListener("input",()=>render(inputEl.value));
 inputEl.addEventListener("keydown",onKey);
 listEl.addEventListener("click",e=>{
  const it=e.target.closest("[data-i]"); if(it)run(+it.dataset.i);
 });
 mounted=true;
}
let shown=[];
function render(q){
 shown=suggest(q); sel=0;
 listEl.innerHTML=shown.length? shown.map((it,i)=>
  `<li class="cp-it${i===0?" sel":""}" data-i="${i}">
    <span class="lbl">${esc(it.label)}</span>
    <span class="hint">${esc(it.hint)}</span></li>`).join("")
  : `<li class="cp-empty">لا نتائج</li>`;
}
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
function run(i){
 const it=shown[i]; if(!it)return;
 close();
 try{ it.run(); }
 catch(e){ if(HOOK&&HOOK.report)HOOK.report("er",e.message||String(e)); }
}
function move(d){
 if(!shown.length)return;
 sel=(sel+d+shown.length)%shown.length;
 [...listEl.children].forEach((li,i)=>li.classList.toggle("sel",i===sel));
 listEl.children[sel]?.scrollIntoView({block:"nearest"});
}
function onKey(e){
 if(e.key==="Escape"){ e.preventDefault(); close(); return; }
 if(e.key==="ArrowDown"){ e.preventDefault(); move(1); return; }
 if(e.key==="ArrowUp"){ e.preventDefault(); move(-1); return; }
 if(e.key==="Enter"){ e.preventDefault(); run(sel); return; }
}
function close(){ if(root)root.hidden=true; }

export function initCmdPalette(opts){
 extra=(opts&&opts.extra)||[];
 buildIndex();
}
export function openCmdPalette(){
 if(!mounted)build();
 buildIndex();
 root.hidden=false; inputEl.value=""; render(""); inputEl.focus();
}
export function closeCmdPalette(){ close(); }
export const cmdPaletteOpen=()=>!!(root&&!root.hidden);
export function toggleCmdPalette(){
 if(!mounted)build();
 if(root.hidden)openCmdPalette(); else close();
}
```

<a id="f-js-ui-ctxmenu-js"></a>

---

## `js/ui/ctxmenu.js`

```javascript
/* ═══ قائمة السياق على اللوحة ═══
   تُبنى من الحال: أداةٌ نشطة ⇒ تأكيدٌ وخياراتُ خطوتها وتراجعٌ
   وإلغاء · تحديدٌ قائم ⇒ أوامر نوعه من مخطّط الشريط نفسه (فلا
   قائمتان تتخلّف إحداهما) · سكونٌ ⇒ إعادةُ آخر أداة وأشيعُ الأوامر.

   والتنفيذ عبر runSpec نفسه الذي ينفّذ به الشريط. */
import {CTX} from "./ribbon/schema.js";
import {icon} from "./icons.js";
import {UIS} from "./store.js";
import * as R from "../tools/registry.js";
import {selList,draw} from "./canvas.js";
import {NAME} from "../core/ents.js";
import {runSpec} from "./ribbon/wire.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||14):"";
const rtl=()=>getComputedStyle(document.documentElement)
 .direction==="rtl";
const cl=(v,a,b)=>v<a?a:(v>b?b:v);

let ROWS=[];

const flat=(items,out)=>{
 (items||[]).forEach(it=>{
  if(it.group)flat(it.group,out); else out.push(it);
 });
 return out;
};
function rowsFor(){
 const out=[];
 const H=n=>out.push({h:n});
 const S2=()=>out.push({s:1});
 if(R.active()){
  const st=R.step();
  H(R.T.def.label+(st?" · "+st.p:""));
  out.push({fn:"enter",n:"تأكيد (Enter)",ico:"select"});
  if(st&&st.opts)Object.keys(st.opts).forEach(k=>out.push({
   fn:"opt:"+k,n:st.opts[k].n+` (${k.toUpperCase()})`,ico:"ctx"}));
  out.push({fn:"ustep",n:"تراجع خطوة",ico:"undo"});
  S2();
  out.push({fn:"cancel",n:"إلغاء الأداة (Esc)",ico:"close"});
  return out;
 }
 const L=selList();
 let kind=null;
 if(L.length){
  kind=L[0].k;
  for(const s of L)if(s.k!==kind){kind=null; break}
 }
 if(L.length){
  H(L.length>1?`${L.length} عنصر`
   :`${L[0].id} ${NAME[L[0].k]||L[0].k}`);
  ["move","copy","rotate","mirror"].forEach(c=>out.push(
   {cmd:c,n:{move:"نقل",copy:"نسخ",rotate:"دوران",
    mirror:"مرآة"}[c],ico:c}));
  if(kind&&CTX[kind]){
   S2();
   const seen=new Set();
   (CTX[kind].panels||[]).forEach(p=>flat(p.items,[])
    .forEach(it=>{
     const id=it.cmd||it.act;
     if(!id||seen.has(id))return;
     if(["move","copy","rotate","mirror"].includes(it.cmd))return;
     seen.add(id);
     out.push({cmd:it.cmd,act:it.act,n:it.n,ico:it.ico});
    }));
  }
  S2();
  out.push({fn:"clear",n:"ألغِ التحديد",ico:"close"});
  return out;
 }
 H("لا تحديد");
 if(R.T.last){
  const d=R.TOOLS[R.T.last];
  out.push({cmd:R.T.last,
   n:`أعِد «${d?d.label:R.T.last}»`,ico:"redo"});
  S2();
 }
 [["wall","جدار","wall"],["rect","مستطيل","rect"],
  ["dim","بُعد","dim"],["text","نصّ","text"]]
  .forEach(([c,n,i])=>out.push({cmd:c,n,ico:i}));
 S2();
 out.push({fn:"all",n:"تحديد المرئيّ (Ctrl+A)",ico:"select"});
 out.push({act:"fit",n:"ملاءمة",ico:"fit"});
 out.push({act:"undo",n:"تراجع",ico:"undo"});
 out.push({act:"inspect",n:"افحص",ico:"inspect"});
 return out;
}
export function ctxOpen(x,y){
 const m=$("#ctxMenu");
 if(!m)return false;
 ROWS=rowsFor();
 m.innerHTML=ROWS.map((r,i)=>r.s?`<div class="pmS"></div>`
  :(r.h?`<div class="pmH">${esc(r.h)}</div>`
  :`<button type="button" class="pmI" data-ctx="${i}">`
   +`${ic(r.ico||"ctx")}<span>${esc(r.n)}</span></button>`)).join("");
 m.hidden=false;
 const w=m.offsetWidth||220, h=m.offsetHeight||260;
 m.style.insetInlineStart=Math.round(
  cl(rtl()?(innerWidth-x-4):(x-w+4),4,innerWidth-w-4))+"px";
 m.style.insetBlockStart=Math.round(cl(y+2,4,innerHeight-h-8))+"px";
 const f=m.querySelector(".pmI");
 if(f)f.focus();
 return true;
}
export const ctxClose=()=>{
 const m=$("#ctxMenu");
 if(m&&!m.hidden){m.hidden=true; ROWS=[]}
};
export const ctxIsOpen=()=>{
 const m=$("#ctxMenu");
 return !!m&&!m.hidden;
};
function run(i){
 const r=ROWS[i];
 ctxClose();
 if(!r)return;
 if(r.cmd||r.act){runSpec(r); return}
 const f=r.fn||"";
 if(f==="enter"){R.enter(); HOOK.prompt(); draw(); return}
 if(f==="cancel"){R.cancel(); HOOK.prompt(); draw(); return}
 if(f==="ustep"){R.undoStep(); return}
 if(f.startsWith("opt:")){
  R.feedText(f.slice(4));
  HOOK.prompt(); draw();
  return;
 }
 if(f==="clear"){
  import("./canvas.js").then(C=>{C.setSel([],null); C.draw()});
  return;
 }
 if(f==="all"){
  import("./canvas.js").then(C=>{
   HOOK.status(`${C.selectAll()} محدد`);
  });
 }
}
export function wireCtx(){
 const m=$("#ctxMenu");
 if(!m)return false;
 m.addEventListener("click",e=>{
  const b=e.target.closest("[data-ctx]");
  if(b)run(+b.dataset.ctx);
 });
 addEventListener("mousedown",e=>{
  if(ctxIsOpen()&&!m.contains(e.target))ctxClose();
 },true);
 addEventListener("keydown",e=>{
  if(!ctxIsOpen())return;
  if(e.key==="Escape"){
   e.preventDefault(); e.stopPropagation(); ctxClose();
   const cv=document.getElementById("cv");
   if(cv)cv.focus();
   return;
  }
  if(e.key==="ArrowDown"||e.key==="ArrowUp"){
   e.preventDefault(); e.stopPropagation();
   const B=[...m.querySelectorAll(".pmI")];
   const i=B.indexOf(document.activeElement);
   const j=(i+(e.key==="ArrowDown"?1:-1)+B.length)%B.length;
   if(B[j])B[j].focus();
  }
 },true);
 return true;
}
```

<a id="f-js-ui-defaults-js"></a>

---

## `js/ui/defaults.js`

```javascript
/* ═══ لوحة الإعدادات الافتراضية ═══
   تُبنى من سجلّ الأدوات نفسه فلا تتخلّف عنه: كل حقلٍ هنا هو الحقل
   الذي يظهر في شريط الخيارات، والقيمة واحدة في الموضعين لأن
   المصدر واحد (OPT في registry).
   الافتراضات لزجة بين الجلسات سلفاً؛ الناقص كان العرض لا الحفظ.

   وتُعرَض الحقول كلّها بلا شرط when: تضبط ارتفاع السترة قبل أن
   تختار «سترة». ولأن الشرط ملغى فمجموعة الحقول ثابتة، فينقسم
   العمل قسمين كما في شريط الخيارات:
     renderDefaults  يبني — مرّةً في الإقلاع، وعند إعادة المصنع.
     syncDefaults    يحدّث القيَم بمقارنة، ويتخطّى المركَّز عليه.
   وكان البناء الكامل يقع مع كل تغيير خيار، فتُغلَق الأقسام
   الفرعية المفتوحة ويُفقَد موضع التمرير. */
import * as R from "../tools/registry.js";
import {reg,renderPanel,markDirty} from "./panels.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");

const withOpts=()=>R.toolList()
 .filter(d=>d&&d.id&&(d.opts||[]).length)
 .sort((a,b)=>a.label.localeCompare(b.label,"ar"));

function field(d,f){
 const o=R.OPT[d.id]||{};
 const v=(o[f.k]===undefined)?f.def:o[f.k];
 const tag=`data-dt="${esc(d.id)}" data-dk="${esc(f.k)}"`;
 if(f.type==="chk")
  return `<div class="row"><label class="chk">`
   +`<input type="checkbox" ${tag}`
   +`${(v===1||v===true||v==="1")?" checked":""}> `
   +`${esc(f.label)}</label></div>`;
 let ctl;
 if(f.type==="sel")
  ctl=`<select ${tag}>`+f.items.map(([iv,it])=>
   `<option value="${esc(iv)}"`
   +`${String(iv)===String(v)?" selected":""}>${esc(it)}</option>`)
   .join("")+`</select>`;
 else
  ctl=`<input class="num" type="${f.type==="num"?"number":"text"}" `
   +`${tag} value="${esc(v)}"${f.type==="num"?' step="0.1"':""}>`;
 return `<div class="row"><label>${esc(f.label)}</label>${ctl}</div>`;
}
/* box يأتي من سجلّ اللوحات · و$ احتياطٌ للنداء المباشر */
export function renderDefaults(box){
 const el=box||$("#defs");
 if(!el)return;
 const T=withOpts();
 el.innerHTML=`<p class="hint">${T.length} أداة لها خيارات — `
  +`تُقرأ عند إنشاء العنصر لا قبله، فتغييرها وسط سلسلة يسري على `
  +`القطعة التالية وحدها.</p>`
  +T.map(d=>`<details class="sub"><summary>${esc(d.label)}`
   +` <span class="mono" style="color:var(--fg3)">`
   +`${esc(d.id)}</span></summary><div>`
   +(d.opts||[]).map(f=>field(d,f)).join("")
   +`</div></details>`).join("");
}
export function syncDefaults(){
 const box=$("#defs");
 if(!box)return;
 const A=document.activeElement;
 box.querySelectorAll("[data-dt]").forEach(el=>{
  if(el===A)return;              /* لا نكتب فوق ما يكتبه المستخدم */
  const o=R.OPT[el.dataset.dt]||{};
  const v=o[el.dataset.dk];
  if(el.type==="checkbox"){
   const on=(v===1||v===true||v==="1");
   if(el.checked!==on)el.checked=on;
   return;
  }
  const s=String(v==null?"":v);
  if(el.value!==s)el.value=s;
 });
}
export function wireDefaults(){
 const box=$("#defs");
 if(!box)return;
 reg("defs","#defs",renderDefaults,"الإعدادات الافتراضية");
 box.addEventListener("change",e=>{
  const dt=e.target.dataset.dt, dk=e.target.dataset.dk;
  if(!dt||!dk)return;
  const v=(e.target.type==="checkbox")?(e.target.checked?1:0)
   :e.target.value;
  R.setOpt(dt,dk,v);
  const d=R.TOOLS[dt];
  const f=d&&(d.opts||[]).find(x=>x.k===dk);
  HOOK.report("in",`${d?d.label:dt} · ${f?f.label:dk}: `
   +((f&&f.type==="chk")?(v?"مُشغّل":"مُوقف"):v));
  HOOK.prompt();          /* شريط الخيارات يُبنى من OPT نفسه */
 });
 box.addEventListener("keydown",e=>{
  if(e.key==="Escape"){e.target.blur();return}
  e.stopPropagation();
 });
 const rst=$("#dRst");
 if(rst)rst.onclick=()=>{
  if(!confirm("إعادة كل خيارات الأدوات إلى مصنعها؟"))return;
  let n=0;
  withOpts().forEach(d=>(d.opts||[]).forEach(f=>{
   const o=R.OPT[d.id]||{};
   if(String(o[f.k])!==String(f.def))n++;
   R.setOpt(d.id,f.k,f.def);
  }));
  markDirty("defs");
  renderPanel("defs");
  HOOK.report(n?"ok":"in",n?`أُعيد ${n} خياراً إلى مصنعه`
   :"كلّها على المصنع سلفاً");
  HOOK.prompt();
 };
 renderPanel("defs");
}
```

<a id="f-js-ui-dock-js"></a>

---

## `js/ui/dock.js`

```javascript
/* ═══ قشرة الإرساء ═══
   قاعدةٌ واحدة تحكم الملفّ كلّه: العُقَد تُنقَل ولا تُعاد بناءً.
   appendChild ينقل العنصر بمستمعيه وقيَم حقوله وموضع تمريره،
   فلوحةٌ تنتقل من عمودٍ إلى نافذةٍ عائمة تحفظ كل ذلك. ولذلك
   انتقل تفويض props.js إلى document: لوحةٌ خارج #side لا يصلها
   مستمعٌ مربوطٌ عليه.

   ولا حالةَ موازية: الموضع في UIS.layout، والانفتاح في نفس
   المكان (secSet في store.js تكتب فيه)، والمرسوم يُقرأ من DOM.
   فلا يفترق ما تراه عمّا يُحفَظ. */
import {PANELS,ZONES,ZN,MODES,WMIN,WMAX,WS,DEFLAY,normLay,
        wsNorm,pDef,pIds,isPanel} from "./layout.js";
import {UIS,saveUI,uiSet} from "./store.js";
import {icon} from "./icons.js";
import {renderVisible,renderPanel,markDirty} from "./panels.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||14):"";
const rtl=()=>getComputedStyle(document.documentElement)
 .direction==="rtl";
const clamp=(v,a,b)=>v<a?a:(v>b?b:v);

let L=null;                    /* مرجعٌ إلى UIS.layout */
const TITLE={};                /* المعرّف ⇒ التسمية · تُقرأ من DOM */
const save=()=>saveUI();

export const zoneEl=z=>$(z==="s"?"#side":"#sideE");
export const stripEl=z=>$(z==="s"?"#stripS":"#stripE");
export const secEl=id=>document.querySelector(
 `details.sec[data-sec="${id}"]`);
export const fltEl=id=>document.querySelector(`.flt[data-flt="${id}"]`);
export const zoneOf=id=>(L&&L.p[id])?L.p[id].z:null;
export const titleOf=id=>TITLE[id]||id;
export const layout=()=>L;

/* الترتيب من DOM لا من الرقم — المرسوم هو المرجع.
   ‏.sec بلا data-sec ليست لوحة: النقصُ لا يجوز أن يُدخِل undefined
   في القائمة فيرمي لاحقاً في buildTabs أو pDef(null). */
export function order(z){
 const el=zoneEl(z);
 if(!el)return [];
 return [...el.querySelectorAll(":scope > details.sec")]
  .map(d=>d.dataset.sec).filter(isPanel);
}
function reindex(){
 ZONES.forEach(z=>order(z).forEach((id,i)=>{
  if(L.p[id])L.p[id].i=i;
 }));
}
/* ═══ الحصاد ═══
   يُنادى مرّةً بعد buildSide: يقرأ التسميات، ويحقن أدوات الترويسة،
   ويُودِع الجميع في المرآب ثم يوزّعهم بالتخطيط. */
function harvest(){
 const src=$("#side");
 if(!src)return 0;
 const park=$("#pPark");
 let n=0;
 PANELS.forEach(p=>{
  const d=secEl(p.id);
  if(!d)return;
  const sum=d.querySelector(":scope > summary");
  if(sum&&!sum.dataset.wired){
   TITLE[p.id]=sum.textContent.trim();
   sum.dataset.wired="1";
   sum.insertAdjacentHTML("beforeend",
    `<span class="pT">`
    +`<button type="button" class="pB" data-pm="${esc(p.id)}"`
    +` title="خيارات اللوحة" aria-label="خيارات ${esc(TITLE[p.id])}"`
    +`>⋮</button>`
    +`<button type="button" class="pB pX" data-pc="${esc(p.id)}"`
    +` title="إغلاق اللوحة" aria-label="إغلاق ${esc(TITLE[p.id])}"`
    +`>✕</button></span>`);
  }
  park.appendChild(d);
  n++;
 });
 return n;
}
/* ═══ التوزيع ═══ */
function applyLayout(){
 const park=$("#pPark"), fl=$("#floats");
 /* بالترتيب المخزَّن لا بترتيب الإعلان */
 const byZ={s:[],e:[],f:[],x:[]};
 PANELS.forEach(p=>{
  const st=L.p[p.id];
  (byZ[st.z]||byZ.x).push(p.id);
 });
 ZONES.forEach(z=>{
  byZ[z].sort((a,b)=>L.p[a].i-L.p[b].i);
  const host=zoneEl(z);
  byZ[z].forEach(id=>{
   const d=secEl(id);
   if(d)host.appendChild(d);
  });
 });
 byZ.f.forEach(id=>mkFloat(id));
 byZ.x.forEach(id=>{
  const d=secEl(id);
  if(d)park.appendChild(d);
 });
 PANELS.forEach(p=>{
  const d=secEl(p.id);
  if(d)d.open=!!L.p[p.id].o;
 });
 syncZones();
}
/* ═══ النوافذ العائمة ═══ */
function mkFloat(id){
 const d=secEl(id);
 if(!d)return null;
 let f=fltEl(id);
 if(!f){
  f=document.createElement("div");
  f.className="flt";
  f.dataset.flt=id;
  $("#floats").appendChild(f);
 }
 f.appendChild(d);
 d.open=true;                  /* العائمة لا تُطوى — تُغلَق */
 placeFloat(id);
 return f;
}
function placeFloat(id){
 const f=fltEl(id), st=L.p[id];
 if(!f||!st)return;
 f.classList.toggle("max",!!st.max);
 if(st.max){f.style.inlineSize="";f.style.blockSize="";return}
 const W=innerWidth, H=innerHeight;
 st.w=clamp(st.w,220,Math.max(240,W-40));
 st.h=clamp(st.h,120,Math.max(160,H-60));
 st.x=clamp(st.x,-st.w+80,Math.max(0,W-80));
 st.y=clamp(st.y,0,Math.max(0,H-60));
 f.style.insetInlineStart=st.x+"px";
 f.style.insetBlockStart=st.y+"px";
 f.style.inlineSize=st.w+"px";
 f.style.blockSize=st.h+"px";
}
export function toFront(id){
 const f=fltEl(id);
 if(!f)return;
 $("#floats").appendChild(f);
}
/* ═══ النقل ═══ */
export function dockTo(id,z,before){
 if(!isPanel(id)||!ZONES.includes(z))return false;
 const d=secEl(id);
 if(!d)return false;
 const f=fltEl(id);
 const host=zoneEl(z);
 if(before&&before.parentElement===host)host.insertBefore(d,before);
 else host.appendChild(d);
 if(f)f.remove();
 L.p[id].z=z; L.p[id].max=0;
 reindex(); syncZones(); save();
 markDirty(id); renderVisible(1);
 return true;
}
export function toFloat(id,x,y){
 if(!isPanel(id))return false;
 const d=secEl(id);
 if(!d)return false;
 const st=L.p[id];
 if(x!=null){st.x=Math.round(x); st.y=Math.round(y)}
 else{
  const r=d.getBoundingClientRect();
  if(r.width>8){
   st.x=Math.round(rtl()?(innerWidth-r.right):r.left);
   st.y=Math.round(r.top);
   st.w=Math.round(clamp(r.width,240,720));
  }
 }
 st.z="f"; st.o=1;
 mkFloat(id);
 reindex(); syncZones(); save();
 markDirty(id); renderVisible(1);
 return true;
}
export function closePanel(id){
 if(!isPanel(id))return false;
 const d=secEl(id);
 if(!d)return false;
 const f=fltEl(id);
 $("#pPark").appendChild(d);
 if(f)f.remove();
 L.p[id].z="x";
 reindex(); syncZones(); save();
 HOOK.report("in",`أُغلقت لوحة «${titleOf(id)}» — `
  +`تُعاد من «عرض ← لوحات»`);
 return true;
}
export function openPanel(id,z){
 if(!isPanel(id))return false;
 const st=L.p[id];
 if(st.z==="x")return dockTo(id,z||pDef(id).z||"s");
 if(st.z==="f"){toFront(id); return true}
 return true;
}
export function movePanel(id,dir){
 const st=L.p[id];
 if(!st||st.z==="f"||st.z==="x")return false;
 const host=zoneEl(st.z);
 const d=secEl(id);
 const sib=(dir<0)?d.previousElementSibling:d.nextElementSibling;
 const tgt=(sib&&sib.matches("details.sec"))?sib:null;
 if(!tgt)return false;
 if(dir<0)host.insertBefore(d,tgt);
 else host.insertBefore(tgt,d);
 reindex(); syncZones(); save();
 return true;
}
export function setOpen(id,v){
 const d=secEl(id);
 if(!d)return false;
 d.open=!!v;                   /* toggle يُنصَت في panels.js فيُرسَم */
 return true;
}
/* ═══ العمود: الوضع · العرض · الإخفاء التلقائي ═══ */
export function setMode(z,m){
 if(!MODES[m])return false;
 L.mode[z]=m;
 if(m==="tab"&&!order(z).includes(L.cur[z]))L.cur[z]=order(z)[0]||null;
 syncZones(); save();
 renderVisible(1);
 HOOK.report("in",`${ZN[z]}: ${MODES[m]}`);
 return true;
}
export function setZoneW(z,w){
 L.zw[z]=Math.round(clamp(w,WMIN,WMAX));
 const el=zoneEl(z);
 if(el)el.style.inlineSize=L.zw[z]+"px";
 dispatchEvent(new Event("resize"));
 return L.zw[z];
}
export function setAuto(z,v){
 L.auto[z]=v?1:0;
 if(!L.auto[z])unpeek();
 syncZones(); save();
 HOOK.report("in",`${ZN[z]}: ${L.auto[z]
  ?"إخفاء تلقائي — انقر شارته لتنكشف":"مثبَّت"}`);
 return L.auto[z];
}
let PEEK=null;
export function peek(z,id){
 unpeek();
 const el=zoneEl(z);
 if(!el)return;
 PEEK=z;
 el.classList.add("peek");
 el.hidden=false;
 if(L.mode[z]==="tab"&&id)L.cur[z]=id;
 else if(id)setOpen(id,1);
 syncZones();
 renderVisible(1);
}
export function unpeek(){
 if(!PEEK)return;
 const el=zoneEl(PEEK);
 if(el)el.classList.remove("peek");
 PEEK=null;
 syncZones();
}
export const peeking=()=>PEEK;

/* ═══ المزامنة ═══ حالة الأعمدة والشارات والتبويبات ═══ */
export function syncZones(){
 ZONES.forEach(z=>{
  const el=zoneEl(z), st=stripEl(z);
  const ids=order(z);
  const has=ids.length>0;
  const auto=!!L.auto[z]&&has;
  el.dataset.mode=L.mode[z];
  el.classList.toggle("auto",auto);
  el.hidden=!has||(auto&&PEEK!==z)||!!UIS.clean;
  el.style.inlineSize=L.zw[z]+"px";
  const rs=document.querySelector(`.dsz[data-rsz="${z}"]`);
  if(rs)rs.hidden=el.hidden;
  if(st){st.hidden=!auto||!!UIS.clean; if(auto)buildStrip(z,ids)}
  if(L.mode[z]==="tab"){
   if(!ids.includes(L.cur[z]))L.cur[z]=ids[0]||null;
   buildTabs(z,ids);
  }else{
   const t=el.querySelector(":scope > .zTabs");
   if(t)t.remove();
  }
  ids.forEach(id=>{
   const d=secEl(id);
   if(!d)return;
   const tabMode=(L.mode[z]==="tab");
   d.classList.toggle("inTab",tabMode);
   d.hidden=tabMode&&(id!==L.cur[z]);
   if(tabMode&&id===L.cur[z])d.open=true;
  });
 });
 const fl=$("#floats");
 if(fl)fl.hidden=!!UIS.clean;
}
function buildTabs(z,ids){
 const el=zoneEl(z);
 let t=el.querySelector(":scope > .zTabs");
 if(!t){
  t=document.createElement("div");
  t.className="zTabs";
  t.setAttribute("role","tablist");
  t.dataset.zone=z;
  el.insertBefore(t,el.firstChild);
 }
 t.innerHTML=ids.map(id=>
  `<button type="button" role="tab" data-ztab="${esc(id)}"`
  +` data-zone="${esc(z)}" aria-selected="${id===L.cur[z]}"`
  +` tabindex="${id===L.cur[z]?0:-1}"`
  +` class="${id===L.cur[z]?"on":""}" title="${esc(titleOf(id))}">`
  +`${ic(pDef(id).ico,14)}<span>${esc(titleOf(id))}</span>`
  +`</button>`).join("");
}
function buildStrip(z,ids){
 const st=stripEl(z);
 st.innerHTML=ids.map(id=>
  `<button type="button" data-peek="${esc(id)}"`
  +` data-zone="${esc(z)}" title="${esc(titleOf(id))}">`
  +`${ic(pDef(id).ico,14)}<span>${esc(titleOf(id))}</span>`
  +`</button>`).join("");
}
/* ═══ الإظهار ═══ يستعمله الشريط بدلاً من فتح القسم مباشرةً ═══ */
export function revealPanel(id){
 if(!isPanel(id))return null;
 if(UIS.clean)HOOK.clean(0);      /* الخروج كاملٌ لا نصفه */
 const st=L.p[id];
 if(st.z==="x")dockTo(id,pDef(id).z||"s");
 const z=L.p[id].z;
 if(z==="f"){
  toFront(id);
  setOpen(id,1);
  renderVisible(1);
  return secEl(id);
 }
 if(L.auto[z]&&PEEK!==z)peek(z,id);
 if(L.mode[z]==="tab"){L.cur[z]=id; syncZones()}
 else setOpen(id,1);
 renderVisible(1);
 const d=secEl(id);
 if(d)d.scrollIntoView({block:"nearest",behavior:"smooth"});
 save();
 return d;
}
/* مزامنةُ الأعمدة مع علَمٍ يملكه غيرُها — لا كتابةَ للعلَم هنا */
export function dockClean(){ syncZones() }
/* ═══ قائمة اللوحة ═══ أوامرٌ صريحة لا سحبٌ يُخمَّن ═══ */
function pmHtml(id){
 const st=L.p[id], z=st.z;
 const R=[];
 const it=(a,n,i,dis)=>R.push(
  `<button type="button" class="pmI" data-pma="${a}"`
  +`${dis?" disabled":""}>${ic(i)}<span>${esc(n)}</span></button>`);
 ZONES.forEach(k=>it("dock:"+k,"أرسِ في "+ZN[k],
  k==="s"?"dockS":"dockE",z===k));
 it("float","نافذة عائمة","float",z==="f");
 if(z==="f")it("max",st.max?"أعِد الحجم":"كبّر","maxi");
 if(z!=="f"&&z!=="x"){
  R.push(`<div class="pmS"></div>`);
  it("up","إلى الأعلى","up",!secEl(id).previousElementSibling
   ||!secEl(id).previousElementSibling.matches("details.sec"));
  it("down","إلى الأسفل","down",!secEl(id).nextElementSibling);
  R.push(`<div class="pmS"></div>`);
  it("mode","وضع العمود: "
   +MODES[L.mode[z]==="acc"?"tab":"acc"],
   L.mode[z]==="acc"?"tabs":"acc");
  it("auto",L.auto[z]?"ثبّت العمود":"إخفاء تلقائي للعمود","autohide");
 }
 R.push(`<div class="pmS"></div>`);
 it("close","إغلاق اللوحة","close");
 return R.join("");
}
export function pmOpen(id,x,y){
 const m=$("#pMenu");
 if(!m)return;
 m.dataset.for=id;
 m.innerHTML=`<div class="pmH">${esc(titleOf(id))}</div>`+pmHtml(id);
 m.hidden=false;
 const w=m.offsetWidth||220, h=m.offsetHeight||240;
 m.style.insetInlineStart=Math.round(
  clamp(rtl()?(innerWidth-x-4):(x-w+4),4,innerWidth-w-4))+"px";
 m.style.insetBlockStart=Math.round(
  clamp(y+4,4,innerHeight-h-8))+"px";
 const f=m.querySelector(".pmI:not([disabled])");
 if(f)f.focus();
}
export const pmClose=()=>{
 const m=$("#pMenu");
 if(m&&!m.hidden){m.hidden=true; m.dataset.for=""}
};
function pmRun(id,a){
 pmClose();
 if(a==="close")return closePanel(id);
 if(a==="float")return toFloat(id);
 if(a==="up")return movePanel(id,-1);
 if(a==="down")return movePanel(id,1);
 if(a==="max"){
  L.p[id].max=L.p[id].max?0:1;
  placeFloat(id); save();
  return true;
 }
 const d=/^dock:(s|e)$/.exec(a);
 if(d)return dockTo(id,d[1]);
 const z=L.p[id].z;
 if(a==="mode")return setMode(z,L.mode[z]==="acc"?"tab":"acc");
 if(a==="auto")return setAuto(z,!L.auto[z]);
 return false;
}
/* ═══ أسطح العمل ═══ */
export const wsAll=()=>{
 const o={};
 Object.keys(WS).forEach(k=>{o[k]={...WS[k],built:1}});
 Object.keys(UIS.ws||{}).forEach(k=>{
  const w=wsNorm(UIS.ws[k]);
  if(w)o[k]={...w,built:0};
 });
 return o;
};
export function wsApply(key,quiet){
 const A=wsAll(), w=A[key];
 if(!w){HOOK.report("wr",`لا سطح عمل «${key}»`); return false}
 ZONES.forEach(z=>{
  L.zw[z]=clamp(+((w.zw||{})[z])||DEFLAY().zw[z],WMIN,WMAX);
  L.mode[z]=MODES[(w.mode||{})[z]]?w.mode[z]:"acc";
  L.auto[z]=((w.auto||{})[z])?1:0;
  L.cur[z]=null;
 });
 const open=new Set(w.open||[]);
 PANELS.forEach(p=>{L.p[p.id].z="x"; L.p[p.id].max=0});
 ZONES.forEach(z=>(w[z]||[]).forEach((id,i)=>{
  if(!L.p[id])return;
  L.p[id].z=z; L.p[id].i=i; L.p[id].o=open.has(id)?1:0;
 }));
 UIS.wsCur=key;
 applyLayout();
 markDirtyAll();
 renderVisible(1);
 save();
 /* القشرة والتبويب — يتولّاهما app.js عبر الخطّاف فلا دورة */
 if(HOOK.ws)HOOK.ws(w);
 if(!quiet)HOOK.report("ok",`سطح العمل: ${w.n}`);
 return true;
}
export function wsSave(name){
 const n=String(name||"").trim().slice(0,32);
 if(!n){HOOK.report("wr","الاسم فارغ"); return false}
 if(WS[n]){HOOK.report("wr",`«${n}» اسمٌ مدمج — اختر غيره`);
  return false}
 UIS.ws=UIS.ws||{};
 if(!UIS.ws[n]&&Object.keys(UIS.ws).length>=24){
  HOOK.report("wr","٢٤ سطح عملٍ محفوظاً — احذف واحداً أوّلاً");
  return false;
 }
 reindex();
 UIS.ws[n]={n,
  shell:UIS.shell, tab:UIS.tab, clean:UIS.clean?1:0,
  zw:{s:L.zw.s,e:L.zw.e}, mode:{s:L.mode.s,e:L.mode.e},
  auto:{s:L.auto.s,e:L.auto.e},
  s:order("s"), e:order("e"),
  open:pIds().filter(id=>L.p[id].o&&L.p[id].z!=="x")};
 UIS.wsCur=n;
 save();
 HOOK.report("ok",`حُفظ سطح العمل «${n}»`);
 return true;
}
export function wsDel(name){
 if(WS[name]){HOOK.report("wr","السطح المدمج لا يُحذَف"); return false}
 if(!UIS.ws||!UIS.ws[name])return false;
 delete UIS.ws[name];
 if(UIS.wsCur===name)UIS.wsCur="";
 save();
 HOOK.report("ok",`حُذف سطح العمل «${name}»`);
 return true;
}
export function wsReset(){
 const d=DEFLAY();
 L.zw=d.zw; L.mode=d.mode; L.auto=d.auto; L.cur=d.cur;
 PANELS.forEach(p=>{L.p[p.id]={...d.p[p.id]}});
 UIS.wsCur="";
 applyLayout(); markDirtyAll(); renderVisible(1); save();
 HOOK.report("ok","أُعيد تخطيط اللوحات إلى مصنعه");
}
const markDirtyAll=()=>PANELS.forEach(p=>markDirty(p.id));

/* ═══ قائمة أسطح العمل ═══ */
export function wsMenuOpen(x,y){
 const m=$("#wsMenu");
 if(!m)return;
 const A=wsAll();
 m.innerHTML=`<div class="pmH">أسطح العمل</div>`
  +Object.keys(A).map(k=>
   `<button type="button" class="pmI" data-wsa="use:${esc(k)}">`
   +`${ic("ws")}<span>${esc(A[k].n)}</span>`
   +`<span class="ky">${UIS.wsCur===k?"●":""}</span></button>`
   +(A[k].built?"":`<button type="button" class="pmI pmDel"`
    +` data-wsa="del:${esc(k)}" title="حذف">${ic("close")}</button>`))
   .join("")
  +`<div class="pmS"></div>`
  +`<button type="button" class="pmI" data-wsa="save">`
  +`${ic("save")}<span>احفظ الحالي…</span></button>`
  +`<button type="button" class="pmI" data-wsa="reset">`
  +`${ic("undo")}<span>أعِد تخطيط المصنع</span></button>`;
 m.hidden=false;
 const w=m.offsetWidth||230, h=m.offsetHeight||260;
 m.style.insetInlineStart=Math.round(
  clamp(rtl()?(innerWidth-x-4):(x-w+4),4,innerWidth-w-4))+"px";
 m.style.insetBlockStart=Math.round(
  clamp(y+4,4,innerHeight-h-8))+"px";
}
export const wsMenuClose=()=>{
 const m=$("#wsMenu");
 if(m)m.hidden=true;
};
function wsRun(a){
 const u=/^use:(.+)$/.exec(a);
 if(u){wsMenuClose(); return wsApply(u[1])}
 const d=/^del:(.+)$/.exec(a);
 if(d){
  const A=wsAll();
  if(!confirm(`حذف سطح العمل «${(A[d[1]]||{}).n||d[1]}»؟`))return false;
  wsDel(d[1]);
  wsMenuOpen(innerWidth/2,120);
  return true;
 }
 if(a==="save"){
  wsMenuClose();
  const n=prompt("اسم سطح العمل:",UIS.wsCur||"سطحي");
  return n==null?false:wsSave(n);
 }
 if(a==="reset"){
  wsMenuClose();
  if(!confirm("إعادة تخطيط اللوحات إلى مصنعه؟"))return false;
  wsReset();
  return true;
 }
 return false;
}
/* ═══ السحب: العائمة والفواصل ═══ */
let DRAG=null;

function onMove(e){
 if(!DRAG)return;
 const dx=e.clientX-DRAG.px, dy=e.clientY-DRAG.py;
 if(!DRAG.on&&Math.hypot(dx,dy)<4)return;
 DRAG.on=true;
 if(DRAG.k==="flt"){
  const st=L.p[DRAG.id];
  if(st.max)return;
  const sx=rtl()?-dx:dx;
  st.x=Math.round(DRAG.x0+sx);
  st.y=Math.round(DRAG.y0+dy);
  placeFloat(DRAG.id);
  return;
 }
 /* الفاصل: العرض يُقاس من حافة العمود الثابتة لا من الإزاحة */
 const el=zoneEl(DRAG.z);
 const r=el.getBoundingClientRect();
 const anchorRight=(DRAG.z==="s")===rtl();
 setZoneW(DRAG.z, anchorRight?(r.right-e.clientX)
  :(e.clientX-r.left));
}
function onUp(){
 if(!DRAG)return;
 const d=DRAG;
 DRAG=null;
 document.documentElement.classList.remove("dragging");
 if(!d.on)return;
 save();
 if(d.k==="rsz")dispatchEvent(new Event("resize"));
}
/* ═══ التوصيل ═══ */
export function wireDock(){
 /* أدوات الترويسة داخل summary: النقر لا يجوز أن يطوي اللوحة */
 document.addEventListener("click",e=>{
  const pm=e.target.closest("[data-pm]");
  if(pm){
   e.preventDefault(); e.stopPropagation();
   const r=pm.getBoundingClientRect();
   pmOpen(pm.dataset.pm,rtl()?r.left:r.right,r.bottom);
   return;
  }
  const pc=e.target.closest("[data-pc]");
  if(pc){
   e.preventDefault(); e.stopPropagation();
   closePanel(pc.dataset.pc);
   return;
  }
  const a=e.target.closest("[data-pma]");
  if(a&&!a.disabled){
   const m=$("#pMenu");
   pmRun(m.dataset.for,a.dataset.pma);
   return;
  }
  const w=e.target.closest("[data-wsa]");
  if(w){wsRun(w.dataset.wsa); return}
  const zt=e.target.closest("[data-ztab]");
  if(zt){
   L.cur[zt.dataset.zone]=zt.dataset.ztab;
   syncZones(); renderVisible(1); save();
   return;
  }
  const pk=e.target.closest("[data-peek]");
  if(pk){
   const z=pk.dataset.zone;
   if(PEEK===z&&(L.mode[z]!=="tab"||L.cur[z]===pk.dataset.peek))
    unpeek();
   else peek(z,pk.dataset.peek);
   return;
  }
 });
 /* الطيّ بمفتاح المسافة على الترويسة يعمل أصلاً — details أصيل.
    والأسهم بين تبويبات العمود بمنطق القراءة العربية. */
 document.addEventListener("keydown",e=>{
  const m=$("#pMenu"), wm=$("#wsMenu");
  if(e.key==="Escape"){
   if(m&&!m.hidden){e.preventDefault();e.stopPropagation();
    pmClose();return}
   if(wm&&!wm.hidden){e.preventDefault();e.stopPropagation();
    wsMenuClose();return}
   if(PEEK){e.preventDefault();e.stopPropagation();unpeek();return}
  }
  const zt=e.target.closest&&e.target.closest("[data-ztab]");
  if(zt){
   const step=(e.key==="ArrowLeft")?1:((e.key==="ArrowRight")?-1:0);
   if(!step)return;
   e.preventDefault();
   const z=zt.dataset.zone, ids=order(z);
   const i=ids.indexOf(zt.dataset.ztab);
   const j=(i+step+ids.length)%ids.length;
   L.cur[z]=ids[j];
   syncZones(); renderVisible(1); save();
   const nb=document.querySelector(`[data-ztab="${ids[j]}"]`);
   if(nb)nb.focus();
   return;
  }
  const rs=e.target.closest&&e.target.closest("[data-rsz]");
  if(rs){
   const step=(e.key==="ArrowLeft")?-16
    :((e.key==="ArrowRight")?16:0);
   if(!step)return;
   e.preventDefault();
   const z=rs.dataset.rsz;
   const sg=((z==="s")===rtl())?-1:1;
   setZoneW(z,L.zw[z]+step*sg);
   save();
  }
 },true);
 /* سحب العائمة من ترويستها · والطيّ يُمنَع إن حدث نقل */
 document.addEventListener("mousedown",e=>{
  const m=$("#pMenu"), wm=$("#wsMenu");
  if(m&&!m.hidden&&!m.contains(e.target)
   &&!e.target.closest("[data-pm]"))pmClose();
  if(wm&&!wm.hidden&&!wm.contains(e.target)
   &&!e.target.closest("[data-wsx]"))wsMenuClose();
  if(PEEK){
   const el=zoneEl(PEEK);
   if(el&&!el.contains(e.target)&&!e.target.closest(".strip"))
    unpeek();
  }
  const rs=e.target.closest(".dsz");
  if(rs){
   e.preventDefault();
   DRAG={k:"rsz",z:rs.dataset.rsz,px:e.clientX,py:e.clientY,on:0};
   document.documentElement.classList.add("dragging");
   return;
  }
  const sum=e.target.closest(".flt > details.sec > summary");
  if(!sum||e.target.closest(".pB"))return;
  const f=sum.closest(".flt");
  const id=f.dataset.flt;
  toFront(id);
  DRAG={k:"flt",id,px:e.clientX,py:e.clientY,
   x0:L.p[id].x,y0:L.p[id].y,on:0};
  document.documentElement.classList.add("dragging");
 },true);
 /* السحب لا يطوي: النقرة التالية للنقل تُبطَل */
 document.addEventListener("click",e=>{
  const sum=e.target.closest(".flt > details.sec > summary");
  if(sum&&sum.dataset.moved){delete sum.dataset.moved;
   e.preventDefault()}
 },true);
 addEventListener("mousemove",e=>{
  if(DRAG&&DRAG.k==="flt"&&!DRAG.on){
   const s=secEl(DRAG.id);
   const sm=s&&s.querySelector(":scope > summary");
   if(sm&&Math.hypot(e.clientX-DRAG.px,e.clientY-DRAG.py)>=4)
    sm.dataset.moved="1";
  }
  onMove(e);
 });
 addEventListener("mouseup",onUp);
 addEventListener("blur",onUp);
 /* تغيير الحجم يُحفَظ بعد سكون · والعائمة تُقيَّد داخل النافذة */
 addEventListener("resize",()=>{
  PANELS.forEach(p=>{if(L.p[p.id].z==="f")placeFloat(p.id)});
 });
 if(typeof ResizeObserver!=="undefined"){
  let t=null;
  const ro=new ResizeObserver(es=>{
   es.forEach(x=>{
    const id=x.target.dataset.flt;
    if(!id||L.p[id].max)return;
    L.p[id].w=Math.round(x.target.offsetWidth);
    L.p[id].h=Math.round(x.target.offsetHeight);
   });
   if(t)clearTimeout(t);
   t=setTimeout(()=>{t=null; save()},400);
  });
  const mo=new MutationObserver(()=>{
   document.querySelectorAll(".flt").forEach(f=>{
    if(f.dataset.ro)return;
    f.dataset.ro="1"; ro.observe(f);
   });
  });
  mo.observe($("#floats"),{childList:true});
  document.querySelectorAll(".flt").forEach(f=>{
   f.dataset.ro="1"; ro.observe(f);
  });
 }
}
/* ═══ الإقلاع ═══ يُنادى بعد buildSide وقبل أي رسم ═══ */
export function initDock(){
 UIS.layout=normLay(UIS.layout);
 L=UIS.layout;
 const n=harvest();
 applyLayout();
 wireDock();
 return n;
}
export const dockStats=()=>({
 s:order("s"), e:order("e"),
 f:pIds().filter(id=>L.p[id].z==="f"),
 x:pIds().filter(id=>L.p[id].z==="x"),
 mode:{...L.mode}, auto:{...L.auto}, zw:{...L.zw},
 ws:UIS.wsCur||""
});
```

<a id="f-js-ui-dyninput-js"></a>

---

## `js/ui/dyninput.js`

```javascript
/* ═══ الإدخال الحركي ═══
   حقولٌ عند المؤشّر تُظهر الطول والزاوية وتقبلهما. ولا مُحلِّل ثانٍ:
   ما تكتبه يُركَّب سلسلةً بصيغة سطر الإدخال نفسها ويمرّ بـ
   feedText — فالمعنى واحد في الموضعين، ولا تتفرّق قراءةُ «5» عن
   قراءتها هناك.

   وTab يقفل الحقل: الطول يُقيّد المؤشّر على مسافةٍ ثابتة، والزاوية
   على اتجاهٍ ثابت. القفل مساعدةُ إدخال لا تعديل — يزول بوقوع
   النقطة كما يزول قفل الزاوية القائم. */
import {S} from "../core/state.js";
import {M,m3,deg,clamp} from "../core/units.js";
import {angOf,lenOf} from "../core/coords.js";
import * as R from "../tools/registry.js";
import {V,W2S,draw} from "./canvas.js";
import {UIS} from "./store.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");

let SIG=null, IDX=0, EDIT=[0,0];
export const dynOn=()=>!!+S.rb.dyn;

/* ═══ أي حقولٍ تليق بالخطوة الجارية ═══
   الكيان والنصّ والتأكيد لا تقبل إحداثياً، فلا حقولَ لها. */
function fieldsOf(st){
 if(!st||st.ent||st.text||st.confirm)return null;
 if(st.ang)return [{k:"a",n:"الزاوية",u:"°"}];
 return R.baseOf()
  ? [{k:"L",n:"الطول",u:"م"},{k:"a",n:"الزاوية",u:"°"}]
  : [{k:"x",n:"س",u:"م"},{k:"y",n:"ص",u:"م"}];
}
function build(F,sig){
 const box=$("#dynBox");
 box.innerHTML=F.map((f,i)=>
  `<span class="dF"><label for="dyn${i}">${esc(f.n)}</label>`
  +`<input id="dyn${i}" class="num" type="text" data-df="${i}"`
  +` spellcheck="false" autocomplete="off" inputmode="decimal">`
  +`<span class="du">${esc(f.u)}</span></span>`).join("")
 +`<span class="dK" hidden>⟨مقفل⟩</span>`;
 SIG=sig; IDX=0; EDIT=F.map(()=>0);
}
const inp=i=>$(`#dynBox [data-df="${i}"]`);

function place(g){
 const box=$("#dynBox");
 const s=W2S(g[0],g[1]);
 const w=box.offsetWidth||190, h=box.offsetHeight||30;
 const x=clamp(s[0]+18,2,Math.max(2,V.w-w-4));
 const y=clamp(s[1]+14,2,Math.max(2,V.h-h-4));
 box.style.transform=`translate(${Math.round(x)}px,`
  +`${Math.round(y)}px)`;
}
function fill(F,g){
 const b=R.baseOf();
 F.forEach((f,i)=>{
  const el=inp(i);
  if(!el||el===document.activeElement||EDIT[i])return;
  let v="";
  if(f.k==="L")v=b?((lenOf(b,g)/1000).toFixed(3)):"";
  else if(f.k==="a")v=b?(angOf(b,g).toFixed(1)):"";
  else if(f.k==="x")v=(g[0]/1000).toFixed(3);
  else if(f.k==="y")v=(g[1]/1000).toFixed(3);
  if(el.value!==v)el.value=v;
 });
 const kk=$("#dynBox .dK");
 if(kk){
  const L=(R.T.lenLock!=null), A=(R.T.lock!=null);
  kk.hidden=!(L||A);
  if(!kk.hidden)kk.textContent="⟨"
   +(L?`طول ${m3(R.T.lenLock)}`:"")+(L&&A?" · ":"")
   +(A?`زاوية ${R.T.lock}°`:"")+"⟩";
 }
}
export function hideDyn(){
 const box=$("#dynBox");
 if(box&&!box.hidden){
  box.hidden=true;
  EDIT=EDIT.map(()=>0);
 }
}
/* تُنادى من syncPrompt مع كل حركة مؤشّر — تُقارِن ولا تبني */
export function syncDyn(){
 const box=$("#dynBox");
 if(!box)return;
 if(!dynOn()||!R.active()||UIS.clean){hideDyn(); return}
 const F=fieldsOf(R.step());
 const g=R.T.ghost;
 if(!F||!g){hideDyn(); return}
 const sig=F.map(f=>f.k).join(",");
 /* القياس بعد الإظهار: المخفيّ عرضه صفر */
 if(sig!==SIG)build(F,sig);
 box.hidden=false;
 place(g);
 fill(F,g);
}
/* ═══ التركيب والتغذية ═══ */
const val=i=>{
 const el=inp(i);
 return el?String(el.value||"").trim():"";
};
function compose(F){
 if(F.length===1&&F[0].k==="a")return val(0);
 if(F[0].k==="x"){
  const x=val(0), y=val(1);
  if(!x&&!y)return "";
  return `${x||"0"},${y||"0"}`;
 }
 const L=val(0), a=val(1);
 if(L&&a)return `@${L}<${a}`;
 if(L)return L;                 /* الطول على الاتجاه الجاري */
 if(a)return `<${a}`;           /* قفل زاوية */
 return "";
}
function commit(){
 const F=fieldsOf(R.step());
 if(!F)return false;
 const s=compose(F);
 if(!s){R.enter(); return true}
 R.lockLen(null);
 R.T.lock=null;
 EDIT=EDIT.map(()=>0);
 R.feedText(s);
 HOOK.prompt(); draw();
 return true;
}
function lockField(i){
 const F=fieldsOf(R.step());
 if(!F||!F[i])return;
 const v=val(i);
 if(!v)return;
 if(F[i].k==="L")R.lockLen(M(v));
 else if(F[i].k==="a")R.lockAng(deg(parseFloat(v)||0));
}
function clearAll(){
 const F=fieldsOf(R.step())||[];
 F.forEach((f,i)=>{const el=inp(i); if(el)el.value=""});
 EDIT=EDIT.map(()=>0);
 R.lockLen(null);
 R.lockAng(null);
}
/* ═══ التوجيه ═══
   الأرقام إلى هنا، والحروف وصيغُ @ و < إلى سطر الإدخال — قاعدةٌ
   واحدة تُفسَّر: هذه حقولُ قيَمٍ، وذاك سطرُ صيَغٍ وأسماء أدوات. */
export function dynRoute(ch){
 if(!dynOn()||!R.active())return false;
 const box=$("#dynBox");
 if(!box||box.hidden)return false;
 if(!/^[0-9.\-٠-٩۰-۹]$/.test(ch))return false;
 const el=inp(IDX)||inp(0);
 if(!el)return false;
 el.focus();
 el.value=ch;
 EDIT[IDX]=1;
 try{el.setSelectionRange(ch.length,ch.length)}catch(e){}
 return true;
}
export function wireDyn(){
 const box=$("#dynBox");
 if(!box)return false;
 box.addEventListener("input",e=>{
  const i=+e.target.dataset.df;
  if(isFinite(i))EDIT[i]=e.target.value?1:0;
 });
 box.addEventListener("keydown",e=>{
  const i=+e.target.dataset.df;
  const F=fieldsOf(R.step())||[];
  if(e.key==="Enter"){
   e.preventDefault(); e.stopPropagation();
   commit();
   return;
  }
  if(e.key==="Tab"){
   e.preventDefault(); e.stopPropagation();
   if(F.length<2)return;
   if(!e.shiftKey)lockField(i);
   const j=(i+(e.shiftKey?-1:1)+F.length)%F.length;
   IDX=j;
   const n=inp(j);
   if(n){n.focus(); n.select()}
   return;
  }
  if(e.key==="Escape"){
   e.stopPropagation();
   const dirty=val(0)||val(1)||R.T.lock!=null||R.T.lenLock!=null;
   if(dirty){
    e.preventDefault();
    clearAll();
    HOOK.prompt(); draw();
    return;
   }
   /* نظيفٌ ⇒ نُسلّم للمعالج العامّ ليلغي الأداة */
   const cv=document.getElementById("cv");
   if(cv)cv.focus();
   return;
  }
  if(e.key==="ArrowUp"||e.key==="ArrowDown")return;
  e.stopPropagation();
 });
 box.addEventListener("focusin",e=>{
  const i=+e.target.dataset.df;
  if(isFinite(i))IDX=i;
 });
 return true;
}
```

<a id="f-js-ui-gallery-js"></a>

---

## `js/ui/gallery.js`

```javascript
/* ═══ واجهة معرض القوالب والكتل الذكية ═══
   نافذةٌ عائمة بتبويبين: قوالب جاهزة (مبانٍ كاملة) وكتل بارامترية
   (عناصر مفردة). كل بطاقة تُدرِج عند النقر مباشرة عند نقطة الأصل
   [0,0] وتمرّ عبر المدقّق (validate) قبل أي كتابة في الحالة. */
import {templateList, placeTemplate} from "../ai/templates_lib.js";
import {blockList, placeBlock} from "../ai/smartblocks.js";

export function createGallery(opts={}) {
  const onDone = typeof opts.onDone === "function" ? opts.onDone : ()=>{};

  const root = document.createElement("div");
  root.className = "gal"; root.setAttribute("role", "dialog");
  root.setAttribute("aria-label", "معرض القوالب والكتل");
  root.dir = "rtl"; root.hidden = true;

  root.innerHTML = `
    <div class="gal-head">
      <div class="gal-tabs" role="tablist">
        <button class="gal-tab is-on" data-tab="tpl" role="tab">قوالب جاهزة</button>
        <button class="gal-tab" data-tab="blk" role="tab">كتل بارامترية</button>
      </div>
      <button class="gal-close" type="button" aria-label="إغلاق">×</button>
    </div>
    <div class="gal-grid" role="tabpanel"></div>
  `;

  (opts.mount || document.body).appendChild(root);
  const grid = root.querySelector(".gal-grid");
  let tab = "tpl";

  function card(item, kind) {
    const c = document.createElement("button");
    c.className = "gal-card"; c.type = "button";
    c.innerHTML = `<span class="gal-card-title">${item.label}</span>
      <span class="gal-card-cat">${item.cat || ""}</span>
      <span class="gal-card-desc">${item.desc || (item.blocks ? item.blocks + " كتل" : "")}</span>`;

    c.addEventListener("click", () => {
      const rep = kind === "tpl"
        ? placeTemplate(item.id, {commit:true})
        : placeBlock(item.id, [0,0], null, {commit:true});
      onDone(item, rep);
      close();
    });
    return c;
  }

  function render() {
    grid.innerHTML = "";
    const items = tab === "tpl" ? templateList : blockList;
    items.forEach(it => grid.appendChild(card(it, tab)));

    root.querySelectorAll(".gal-tab").forEach(t => {
      t.classList.toggle("is-on", t.dataset.tab === tab);
      t.setAttribute("aria-selected", t.dataset.tab === tab ? "true" : "false");
    });
  }

  root.querySelectorAll(".gal-tab").forEach(t => {
    t.addEventListener("click", () => { tab = t.dataset.tab; render(); });
  });

  root.querySelector(".gal-close").addEventListener("click", () => close());
  window.addEventListener("keydown", e => { if (e.key === "Escape" && !root.hidden) close(); });

  function open() { root.hidden = false; render(); }
  function close() { root.hidden = true; }

  return { root, open, close, toggle: () => root.hidden ? open() : close() };
}
```

<a id="f-js-ui-helpbot-js"></a>

---

## `js/ui/helpbot.js`

```javascript
/* ═══ نافذة روبوت الإرشاد — واجهة عصرية ═══
   شاتٌ يجيب على «كيف أفعل كذا؟» مع زرّ عائم، رأسٍ مرتّب، وحالة
   اتصالٍ للمزوّد. يحقن دالة تفعيل الأداة. a11y كامل. */
import {answerQuestion, suggestTopics, TOPIC_Q} from "../ai/help/answer.js";
import {providerReady} from "../ai/help/provider.js";
import {createGallery} from "./gallery.js";

export function createHelpBot(opts={}){
 const activate=typeof opts.activate==="function"?opts.activate:()=>{};

 /* ── الزرّ العائم (FAB) ── */
 const fab=document.createElement("button");
 fab.className="hb-fab"; fab.type="button";
 fab.setAttribute("aria-label","فتح المساعدة");
 fab.title="المساعدة — اسأل: كيف أفعل كذا؟";
 fab.innerHTML=`<svg viewBox="0 0 24 24" width="22" height="22"
   fill="none" stroke="currentColor" stroke-width="2"
   stroke-linecap="round" stroke-linejoin="round">
   <path d="M21 11.5a8.38 8.38 0 0 1-8.5 8.5 8.5 8.5 0 0 1-3.8-.9L3 21l1.9-5.7
    a8.5 8.5 0 0 1 3.5-11.3 8.38 8.38 0 0 1 11.3 3.5 8.38 8.38 0 0 1 1.3 4z"/>
   <path d="M9.5 9a2.5 2.5 0 0 1 4.9.7c0 1.7-2.4 2.3-2.4 2.3"/>
   <line x1="12" y1="16" x2="12" y2="16"/></svg>`;

 /* ── النافذة ── */
 const root=document.createElement("div");
 root.className="helpbot"; root.setAttribute("role","dialog");
 root.setAttribute("aria-label","روبوت المساعدة");
 root.setAttribute("aria-modal","false"); root.dir="rtl"; root.hidden=true;
 root.innerHTML=`
  <div class="hb-head">
    <div class="hb-brand">
      <span class="hb-logo" aria-hidden="true">؟</span>
      <div class="hb-titles">
        <span class="hb-title">مساعد CivilDraft</span>
        <span class="hb-sub"><i class="hb-dot"></i><span class="hb-status">محلي</span></span>
      </div>
    </div>
    <div class="hb-actions">
      <button class="hb-icon hb-clear" type="button"
        aria-label="مسح المحادثة" title="مسح المحادثة">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none"
         stroke="currentColor" stroke-width="2" stroke-linecap="round">
         <path d="M3 6h18M8 6V4h8v2M6 6l1 14h10l1-14"/></svg></button>
      <button class="hb-icon hb-close" type="button"
        aria-label="إغلاق" title="إغلاق">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none"
         stroke="currentColor" stroke-width="2" stroke-linecap="round">
         <path d="M18 6 6 18M6 6l12 12"/></svg></button>
    </div>
  </div>
  <div class="hb-log" role="log" aria-live="polite"></div>
  <div class="hb-chips" role="group" aria-label="أسئلة شائعة"></div>
  <form class="hb-form">
    <input class="hb-in" type="text" autocomplete="off"
      placeholder="اكتب سؤالك… مثل: كيف أبني غرفة؟" aria-label="اكتب سؤالك">
    <button class="hb-send" type="submit" aria-label="إرسال" title="إرسال">
      <svg viewBox="0 0 24 24" width="18" height="18" fill="none"
       stroke="currentColor" stroke-width="2" stroke-linecap="round"
       stroke-linejoin="round"><path d="M22 2 11 13M22 2l-7 20-4-9-9-4z"/></svg>
    </button>
  </form>`;

 const mount=opts.mount||document.body;
 mount.appendChild(fab); mount.appendChild(root);

 /* ── معرض القوالب والكتل ── يفتح فوق نافذة الشات عبر زرّ إجراء ── */
 const gallery=createGallery({ mount,
  onDone:(item,rep)=>{
   if(rep&&rep.committed){
    let msg=`تم إدراج «${item.label}» (${rep.valid} عنصر)`;
    if(rep.rejected&&rep.rejected.length) msg+=` · رُفض ${rep.rejected.length}`;
    if(rep.doorRejected) msg+=" · تعذّر إدراج الباب";
    bubble("bot",msg);
   } else if(rep&&rep.error){
    bubble("bot",rep.error);
   } else {
    bubble("bot",`تعذّر إدراج «${item.label}» — راجع القياسات.`);
   }
  }
 });

 const log=root.querySelector(".hb-log");
 const chips=root.querySelector(".hb-chips");
 const form=root.querySelector(".hb-form");
 const input=root.querySelector(".hb-in");
 const statusEl=root.querySelector(".hb-status");
 const dotEl=root.querySelector(".hb-dot");

 function refreshStatus(){
  const on=providerReady();
  statusEl.textContent=on?"مزوّد متصل":"محلي";
  dotEl.classList.toggle("on",on);
 }

 function bubble(who,text){
  const row=document.createElement("div"); row.className="hb-row hb-"+who;
  const av=document.createElement("span"); av.className="hb-avatar";
  av.textContent=who==="user"?"أنت":"م";
  const b=document.createElement("div"); b.className="hb-msg";
  b.textContent=text;
  row.appendChild(av); row.appendChild(b);
  log.appendChild(row); log.scrollTop=log.scrollHeight; return b;
 }

 function renderAnswer(ans){
  const row=document.createElement("div"); row.className="hb-row hb-bot";
  const av=document.createElement("span"); av.className="hb-avatar";
  av.textContent="م";
  const wrap=document.createElement("div"); wrap.className="hb-msg";
  const t=document.createElement("div"); t.className="hb-ans-text";
  t.textContent=ans.text; wrap.appendChild(t);
  if(ans.warn){ const w=document.createElement("div");
   w.className="hb-warn"; w.textContent="⚠ "+ans.warn; wrap.appendChild(w); }
  if(ans.actions&&ans.actions.length){
   const bar=document.createElement("div"); bar.className="hb-actbar";
   ans.actions.forEach(a=>{ const btn=document.createElement("button");
    btn.className="hb-act"; btn.type="button"; btn.textContent=a.label;
    btn.addEventListener("click",()=>{
     if(a.type==="activateTool"&&a.toolId){
      activate(a.toolId);
      bubble("bot","تم تفعيل الأداة — اتبع الخطوات على اللوحة.");
     } else if(a.type==="runLint"){
      ask("افحص رسمي");
     } else if(a.type==="openGallery"){
      gallery.open();
     }
    });
    bar.appendChild(btn); });
   wrap.appendChild(bar); }
  if(ans.related&&ans.related.length){
   const r=document.createElement("div"); r.className="hb-related";
   const lbl=document.createElement("span"); lbl.className="hb-related-lbl";
   lbl.textContent="قد يهمّك:"; r.appendChild(lbl);
   ans.related.forEach(rel=>{ const link=document.createElement("button");
    link.className="hb-chip hb-sm"; link.type="button";
    link.textContent=rel.title;
    link.addEventListener("click",()=>ask(rel.title)); r.appendChild(link); });
   wrap.appendChild(r); }
  row.appendChild(av); row.appendChild(wrap);
  log.appendChild(row); log.scrollTop=log.scrollHeight;
 }

 async function ask(q){ const text=String(q||"").trim(); if(!text) return;
  chips.classList.add("hb-hidden");            // أخفِ الرقائق بعد أول سؤال
  bubble("user",text);
  const thinking=bubble("bot","");
  thinking.parentElement.classList.add("hb-typing");
  thinking.innerHTML=`<span></span><span></span><span></span>`;
  try{ const ans=await answerQuestion(text);
   thinking.parentElement.remove(); renderAnswer(ans); refreshStatus(); }
  catch(e){ thinking.textContent="حدث خطأ: "+e.message;
   thinking.parentElement.classList.remove("hb-typing"); } }

 function renderChips(){ chips.innerHTML=""; chips.classList.remove("hb-hidden");
  suggestTopics().forEach(t=>{ const c=document.createElement("button");
   c.className="hb-chip"; c.type="button"; c.textContent=t.title;
   c.addEventListener("click",()=>ask(TOPIC_Q[t.id]||t.title));
   chips.appendChild(c); }); }

 function greet(){ if(!log.childElementCount)
   bubble("bot","مرحباً! أنا مساعد CivilDraft. اسألني كيف تعمل أي أداة"
    +" أو مهمة، وسأشرح الخطوات لك."); }

 form.addEventListener("submit",e=>{ e.preventDefault();
  ask(input.value); input.value=""; input.focus(); });
 root.querySelector(".hb-close").addEventListener("click",()=>api.close());
 root.querySelector(".hb-clear").addEventListener("click",()=>{
  log.innerHTML=""; greet(); renderChips(); input.focus(); });
 fab.addEventListener("click",()=>api.toggle());
 window.addEventListener("keydown",e=>{
  if(e.key==="Escape"&&!root.hidden) api.close(); });

 const api={ el:root, fab, gallery,
  open(){ root.hidden=false; fab.classList.add("hb-open");
   refreshStatus(); greet(); renderChips(); input.focus(); },
  close(){ root.hidden=true; fab.classList.remove("hb-open");
   if(opts.onClose) opts.onClose(); },
  toggle(){ root.hidden?api.open():api.close(); }, ask };
 return api;
}
```

<a id="f-js-ui-helpbutton-js"></a>

---

## `js/ui/helpbutton.js`

```javascript
/* ═══ زرّ روبوت المساعدة في الشريط ═══ يحقن زرّاً في #top.
   يستقلّ عمداً عن HOOK.help و#bHelp وF1 (تلك للوحة المرجع
   الثابتة الموجودة مسبقاً في app.js) فلا يتصادم معها — زرٌّ إضافيٌّ
   لروبوت المحادثة، لا بديلٌ لها. onClick صريحٌ يحدّد من يُفتَح. */
export function mountHelpButton(opts={}){
 const bar=document.querySelector(opts.mountSelector||"#top");
 if(!bar) return null;
 if(bar.querySelector(".hb-open-btn")) return null;
 const btn=document.createElement("button");
 btn.className="hb-open-btn"; btn.type="button";
 btn.setAttribute("aria-label", opts.ariaLabel||"فتح مساعد الدردشة");
 btn.title=opts.title||"مساعد CivilDraft (اسأل: كيف أفعل كذا؟)";
 btn.textContent=opts.label||"؟ مساعد";
 const onClick=typeof opts.onClick==="function"?opts.onClick:()=>{};
 btn.addEventListener("click",onClick);
 const gap=bar.querySelector(".gap");
 if(gap && gap.nextSibling) bar.insertBefore(btn, gap.nextSibling);
 else bar.appendChild(btn);
 return btn;
}
```

<a id="f-js-ui-historypanel-js"></a>

---

## `js/ui/historypanel.js`

```javascript
/* ═══ لوحة السجل ═══ سجلّ تراجع/إعادة مرئي — قفزٌ لأي خطوة.
   تعتمد على core/state.js الحقيقي (historyTimeline/historyJumpTo/
   canUndo/canRedo/undo/redo) — لا سجلّ ثانٍ، ولا لقطة مستقلّة.
   تُستدعى refreshHistoryPanel() من نفس معاودة setAfterEdit في
   app.js، فتبقى متزامنةً مع كل تعديلٍ أو تراجعٍ أو إعادة. */
import {historyTimeline,historyJumpTo,canUndo,canRedo,undo,redo}
 from "../core/state.js";
import {HOOK} from "./bus.js";

const ID="histPanel";
let root=null,listEl=null,countEl=null,mounted=false;
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");

function injectCss(){
 if(document.getElementById("hp-css"))return;
 const css=`
#${ID}{position:fixed; inset-block:56px auto; inset-inline-end:12px; z-index:40;
 width:250px; max-height:min(62vh,540px); display:flex; flex-direction:column;
 background:var(--bg2,#1a1d21); color:var(--fg,#e9edf2);
 border:1px solid var(--ln,#2a2f36); border-radius:var(--r3,8px);
 box-shadow:var(--sh3,0 12px 34px rgba(0,0,0,.42)); overflow:hidden;
 font:12.5px/1.5 var(--ui,system-ui,sans-serif)}
#${ID}[hidden]{display:none}
#${ID} .hp-h{display:flex; align-items:center; gap:8px; padding:8px 10px;
 background:var(--bg3,#20242a); border-bottom:1px solid var(--ln,#2a2f36)}
#${ID} .hp-title{font-weight:600}
#${ID} .hp-count{color:var(--fg3,#8a929c); font:11px var(--mono,monospace)}
#${ID} .hp-x{margin-inline-start:auto; background:none; border:0; cursor:pointer;
 color:var(--fg3,#8a929c); font-size:14px; line-height:1; padding:2px 5px; border-radius:4px}
#${ID} .hp-x:hover{background:var(--hov,rgba(255,255,255,.07)); color:var(--fg,#e9edf2)}
#${ID} .hp-body{overflow-y:auto; flex:1}
#${ID} .hp-list{list-style:none; margin:0; padding:4px}
#${ID} .hp-it{display:flex; align-items:center; gap:8px; padding:6px 9px;
 border-radius:6px; cursor:pointer; color:var(--fg2,#c4ccd6)}
#${ID} .hp-it:hover{background:var(--hov,rgba(255,255,255,.07)); color:var(--fg,#e9edf2)}
#${ID} .hp-it .dot{width:8px; height:8px; border-radius:50%; flex:none;
 border:1.5px solid var(--fg3,#8a929c)}
#${ID} .hp-it.cur{background:var(--acq,rgba(110,168,254,.15)); color:var(--acf,#fff)}
#${ID} .hp-it.cur .dot{background:var(--ac,#6ea8fe); border-color:var(--ac,#6ea8fe)}
#${ID} .hp-it.future{opacity:.6}
#${ID} .hp-it .lbl{flex:1; white-space:nowrap; overflow:hidden; text-overflow:ellipsis}
#${ID} .hp-it .n{color:var(--fg3,#8a929c); font:11px var(--mono,monospace)}
#${ID} .hp-empty{padding:20px 12px; text-align:center; color:var(--fg3,#8a929c)}
#${ID} .hp-f{display:flex; gap:6px; padding:8px; background:var(--bg3,#20242a);
 border-top:1px solid var(--ln,#2a2f36)}
#${ID} .hp-f button{flex:1; padding:5px 6px; cursor:pointer; font-size:11.5px;
 color:var(--fg2,#c4ccd6); background:var(--bg2,#1a1d21);
 border:1px solid var(--ln2,#333a42); border-radius:6px}
#${ID} .hp-f button:hover:not(:disabled){background:var(--hov,rgba(255,255,255,.07)); color:var(--fg,#e9edf2)}
#${ID} .hp-f button:disabled{opacity:.4; cursor:default}`;
 const st=document.createElement("style"); st.id="hp-css"; st.textContent=css;
 document.head.appendChild(st);
}
function build(){
 injectCss();
 root=document.getElementById(ID);
 if(!root){
  root=document.createElement("aside");
  root.id=ID; root.hidden=true; root.setAttribute("dir","rtl");
  root.innerHTML=
   `<header class="hp-h"><span class="hp-title">السجل</span>
      <span class="hp-count"></span>
      <button type="button" class="hp-x" data-act="close" title="إغلاق">✕</button></header>
    <div class="hp-body"><ul class="hp-list"></ul></div>
    <footer class="hp-f">
      <button type="button" data-act="undo" title="تراجع">↶ تراجع</button>
      <button type="button" data-act="redo" title="إعادة">↷ إعادة</button></footer>`;
  document.body.appendChild(root);
 }
 listEl=root.querySelector(".hp-list"); countEl=root.querySelector(".hp-count");
 root.addEventListener("click",onClick);
}
export function refreshHistoryPanel(){
 if(!mounted||!root||root.hidden)return;
 const {past,future,current}=historyTimeline();
 const steps=past.concat(future); const total=steps.length;
 countEl.textContent=total?`${current}/${total}`:"";
 root.querySelector('[data-act="undo"]').disabled=!canUndo();
 root.querySelector('[data-act="redo"]').disabled=!canRedo();
 if(!total){listEl.innerHTML=`<li class="hp-empty">لا خطوات بعد — ابدأ الرسم</li>`; return}
 const rows=[];
 for(let a=total;a>=1;a--){
  const cls=a===current?"cur":(a>current?"future":"");
  rows.push(`<li class="hp-it ${cls}" data-jump="${a}" title="${esc(steps[a-1])}">
   <span class="dot"></span><span class="lbl">${esc(steps[a-1])}</span><span class="n">${a}</span></li>`);
 }
 rows.push(`<li class="hp-it ${current===0?"cur":""}" data-jump="0" title="الحالة الأولية">
  <span class="dot"></span><span class="lbl">البداية</span><span class="n">0</span></li>`);
 listEl.innerHTML=rows.join("");
}
function onClick(e){
 const act=e.target.closest("[data-act]");
 if(act){
  if(act.dataset.act==="close")return closeHistoryPanel();
  if(act.dataset.act==="undo"){undo(); refreshHistoryPanel(); return}
  if(act.dataset.act==="redo"){redo(); refreshHistoryPanel(); return}
 }
 const it=e.target.closest("[data-jump]");
 if(it){
  const n=+it.dataset.jump; historyJumpTo(n);
  if(HOOK&&HOOK.report)HOOK.report("in",`السجل: انتقال إلى الخطوة ${n}`);
  refreshHistoryPanel();
 }
}
export function initHistoryPanel(){
 if(mounted)return;
 build(); mounted=true;
}
export function openHistoryPanel(){
 if(!mounted)initHistoryPanel();
 root.hidden=false; refreshHistoryPanel();
}
export function closeHistoryPanel(){ if(root)root.hidden=true; }
export function toggleHistoryPanel(){
 if(!mounted)initHistoryPanel();
 root.hidden=!root.hidden;
 if(!root.hidden)refreshHistoryPanel();
}
export const historyPanelOpen=()=>!!(root&&!root.hidden);
```

<a id="f-js-ui-icons-js"></a>

---

## `js/ui/icons.js`

```javascript
/* ═══ سبرايت الأيقونات ═══
   SVG مضمَّن في الشفرة: لا ملفّات صور ولا اعتماديات ولا طلبات شبكة.
   كلّها 24×24 خطّية بـ currentColor فتتبع السِّمة والحالة (مُشغّل ·
   مُعطَّل · محدَّد) بلا نسخٍ ثانية.

   القيمة هي المحتوى الداخلي للرمز — فيُقبل path وcircle وrect.
   والمفقود يعيد نصّاً فارغاً، فالزرّ يظهر بتسميته ولا ينكسر. */

export const ICONS={
 /* الرسم */
 line:'<path d="M4 20 20 4"/>',
 pline:'<path d="M3 18 9 8l6 6 6-9"/>',
 rect:'<path d="M4 6h16v12H4z"/>',
 circle:'<circle cx="12" cy="12" r="8"/>',
 arc:'<path d="M4 18a8 8 0 0 1 16 0"/>'
  +'<circle cx="4" cy="18" r="1.1"/><circle cx="20" cy="18" r="1.1"/>',
 wall:'<path d="M3 8h18M3 16h18"/>'
  +'<path d="M7 8l-3 8M13 8l-3 8M19 8l-3 8" opacity=".45"/>',
 col:'<path d="M8 8h8v8H8z"/>'
  +'<path d="M12 4.5v3M12 16.5v3M4.5 12h3M16.5 12h3" opacity=".6"/>',
 gridcols:'<path d="M4 4h4v4H4zM16 4h4v4h-4zM4 16h4v4H4zM16 16h4v4h-4z"/>'
  +'<path d="M6 8v8M18 8v8M8 6h8M8 18h8" opacity=".4"/>',

 /* الفتحات */
 door:'<path d="M3 19h4M17 19h4"/><path d="M7 19V7"/>'
  +'<path d="M7 7a12 12 0 0 1 10 12" stroke-dasharray="2.5 2.5"/>',
 window:'<path d="M3 8v8M21 8v8"/><path d="M3 9.5h18M3 14.5h18"/>'
  +'<path d="M12 9.5v5"/>',
 opening:'<path d="M3 8v8M21 8v8"/>'
  +'<path d="M3 9.5h5M16 9.5h5M3 14.5h5M16 14.5h5"/>',
 niche:'<path d="M3 8h18M3 16h18"/><path d="M9 16v-4.5h6V16"/>',

 /* الأجزاء */
 stair:'<path d="M3 20h4v-4h4v-4h4V8h4V4"/>'
  +'<path d="M6 17l10-10" opacity=".45"/>',
 wc:'<rect x="8" y="4" width="8" height="3" rx="1"/>'
  +'<ellipse cx="12" cy="13.5" rx="4" ry="6"/>',
 lav:'<rect x="5" y="5" width="14" height="14" rx="2"/>'
  +'<ellipse cx="12" cy="13" rx="5" ry="3.5"/><path d="M12 5v3"/>',
 shower:'<rect x="5" y="5" width="14" height="14" rx="1"/>'
  +'<path d="M5 5l14 14M19 5L5 19" opacity=".45"/>'
  +'<circle cx="12" cy="12" r="1.6"/>',
 sink:'<rect x="3" y="6" width="18" height="12" rx="1"/>'
  +'<rect x="5.5" y="8.5" width="13" height="7"/>'
  +'<circle cx="9" cy="12" r="1"/><circle cx="15" cy="12" r="1"/>',
 tub:'<rect x="3" y="7" width="18" height="10" rx="2"/>'
  +'<rect x="5.5" y="9" width="13" height="6" rx="2"/>'
  +'<circle cx="7.5" cy="12" r=".9"/>',

 /* المناطق */
 area:'<path d="M4 5h16v14H4z"/>'
  +'<path d="M6 17l10-10M10 17l6-6M6 13l6-6" opacity=".4"/>',
 arearef:'<path d="M4 6h11v12H4z" opacity=".6"/>'
  +'<path d="M20 13a4.5 4.5 0 1 1-2-3.7"/><path d="M18 6v3.5h-3.2"/>',

 /* التعديل */
 move:'<path d="M12 3v18M3 12h18"/>'
  +'<path d="M9.6 5.4 12 3l2.4 2.4M9.6 18.6 12 21l2.4-2.4'
  +'M5.4 9.6 3 12l2.4 2.4M18.6 9.6 21 12l-2.4 2.4"/>',
 copy:'<rect x="4" y="4" width="11" height="11"/>'
  +'<rect x="9" y="9" width="11" height="11"/>',
 rotate:'<path d="M20 12a8 8 0 1 1-3.2-6.4"/><path d="M17 2.5v4h-4"/>'
  +'<circle cx="12" cy="12" r="1.2"/>',
 mirror:'<path d="M12 3v18" stroke-dasharray="3 2"/>'
  +'<path d="M9 7 4 12l5 5z"/><path d="M15 7l5 5-5 5z"/>',
 offset:'<path d="M5 4v16"/><path d="M11 4v16" stroke-dasharray="3 2"/>'
  +'<path d="M14.5 12h5.5M20 12l-2.2-2.2M20 12l-2.2 2.2"/>',
 brk:'<path d="M3 12h6M15 12h6"/>'
  +'<path d="M11 6v12M13 6v12" opacity=".6"/>',
 divide:'<path d="M3 12h18"/><path d="M9 8v8M15 8v8"/>',
 trim:'<path d="M8 3v18M16 3v18" opacity=".55"/>'
  +'<path d="M2 12h6M16 12h6"/><path d="M9.5 9.5 14.5 14.5M14.5 9.5 9.5 14.5"/>',
 extend:'<path d="M18.5 3v18" opacity=".55"/>'
  +'<path d="M3 12h11"/><path d="M14 12h4M18 12l-2.2-2.2M18 12l-2.2 2.2"/>',
 stretch:'<path d="M4 8h8v8H4z" stroke-dasharray="3 2"/>'
  +'<path d="M13 12h7M20 12l-2.4-2.4M20 12l-2.4 2.4"/>',
 weld:'<path d="M3 16h7M14 16v-8"/><circle cx="12" cy="16" r="2.4"/>',
 match:'<path d="M6 14l6-6 4 4-6 6H6z"/><path d="M14 6l2-2 4 4-2 2z"/>',
 scale:'<path d="M5 19V5h14" opacity=".55"/>'
  +'<path d="M9 19v-6h6v6z"/><path d="M15 13 20 8M20 8h-3.5M20 8v3.5"/>',
 array:'<path d="M4 4h5v5H4zM15 4h5v5h-5zM4 15h5v5H4zM15 15h5v5h-5z"/>',
 info:'<circle cx="12" cy="12" r="8.5"/>'
  +'<path d="M12 11.2v5.6"/><path d="M12 7.4v.4"/>',

 /* التأشير */
 dim:'<path d="M4 7v10M20 7v10"/><path d="M4 12h16"/>'
  +'<path d="M7 9.6 4 12l3 2.4M17 9.6 20 12l-3 2.4"/>',
 chain:'<path d="M3 9v6M9 9v6M15 9v6M21 9v6"/><path d="M3 12h18"/>',
 text:'<path d="M5 6h14M12 6v12"/>',
 lead:'<path d="M4 19l8-8h8"/><path d="M4 19l.6-3.6L8.2 15z"/>',
 level:'<path d="M12 6l-4 5h8z"/><path d="M5 11h14"/><path d="M8.5 15h7"/>',
 axis:'<path d="M12 2v16" stroke-dasharray="6 2 1.4 2"/>'
  +'<circle cx="12" cy="20" r="2.4"/>',

 /* الأدوات والحالة */
 measure:'<path d="M3 15L15 3l6 6L9 21z"/>'
  +'<path d="M7 11l2 2M10 8l2 2M13 5l2 2" opacity=".7"/>',
 inspect:'<circle cx="10.5" cy="10.5" r="6.5"/><path d="M15.2 15.2 21 21"/>'
  +'<path d="M10.5 7v4.2M10.5 13.6v.4"/>',
 select:'<path d="M5 4l6 15 2-6 6-2z"/>',
 selid:'<path d="M4 3l5 12 1.6-4.8L15.4 9z"/>'
  +'<path d="M14 15h6M14 19h6M16.4 13v8M19 13v8" opacity=".65"/>',
 layers:'<path d="M12 4l8 4-8 4-8-4z"/><path d="M4 12l8 4 8-4"/>'
  +'<path d="M4 16l8 4 8-4"/>',
 grid:'<path d="M4 4h16v16H4z"/>'
  +'<path d="M9.3 4v16M14.6 4v16M4 9.3h16M4 14.6h16" opacity=".55"/>',
 ortho:'<path d="M5 19V5h14"/><path d="M5 12h7v7" opacity=".5"/>',
 polar:'<circle cx="12" cy="12" r="8"/>'
  +'<path d="M12 12l6-4.5M12 12h8M12 12l5 5.5" opacity=".65"/>',
 osnap:'<rect x="7.5" y="7.5" width="9" height="9"/>'
  +'<path d="M12 2v4M12 18v4M2 12h4M18 12h4"/>',
 grips:'<path d="M4 12h16"/>'
  +'<rect x="2.6" y="10.6" width="2.8" height="2.8"/>'
  +'<rect x="10.6" y="10.6" width="2.8" height="2.8"/>'
  +'<rect x="18.6" y="10.6" width="2.8" height="2.8"/>',
 ends:'<path d="M3 16h9"/><path d="M12 16l3.2-3.2L18.4 16l-3.2 3.2z"/>',

 /* الملفّ والإخراج */
 undo:'<path d="M4.5 9h9a5 5 0 0 1 0 10H8"/><path d="M8 5L4.5 9 8 13"/>',
 redo:'<path d="M19.5 9h-9a5 5 0 0 0 0 10H16"/><path d="M16 5l3.5 4L16 13"/>',
 fit:'<path d="M4 9V4h5M20 9V4h-5M4 15v5h5M20 15v5h-5"/>'
  +'<rect x="9" y="9" width="6" height="6" opacity=".55"/>',
 help:'<circle cx="12" cy="12" r="8.5"/>'
  +'<path d="M9.6 9.6a2.4 2.4 0 1 1 4.8 0c0 1.8-2.4 2-2.4 3.9"/>'
  +'<path d="M12 17.2v.3"/>',
 save:'<path d="M5 4h11l3 3v13H5z"/><path d="M9 4v5h6V4"/>'
  +'<rect x="8" y="13" width="8" height="6"/>',
 open:'<path d="M3 7h6l2 2h10v9H3z"/>',
 fnew:'<path d="M6 3h8l4 4v14H6z"/><path d="M14 3v4h4"/>'
  +'<path d="M12 11v6M9 14h6"/>',
 print:'<path d="M7 9V4h10v5"/><rect x="4" y="9" width="16" height="7" rx="1"/>'
  +'<path d="M7 16h10v5H7z"/>',
 xport:'<path d="M12 3v11"/><path d="M8 10l4 4 4-4"/>'
  +'<path d="M4 18h16v3H4z"/>',
 sheet:'<rect x="3" y="5" width="18" height="14"/>'
  +'<rect x="5.5" y="7" width="13" height="10" stroke-dasharray="2 2"/>'
  +'<path d="M13 17v-4h5.5"/>',
 ref:'<rect x="3" y="5" width="18" height="14" stroke-dasharray="4 3"/>'
  +'<path d="M7.5 15.5L12 9l4.5 6.5"/>',
 table:'<rect x="3" y="5" width="18" height="14"/>'
  +'<path d="M3 10h18M3 15h18M9 5v14M15 5v14" opacity=".7"/>'
 ,
 /* ═══ و١ — الفتحات والأدوات الناقصة ═══ */
 fixed:'<path d="M3 8v8M21 8v8"/><path d="M3 9.5h18M3 14.5h18"/>'
  +'<path d="M6 9.5 18 14.5" opacity=".45"/>',
 arch:'<path d="M3 8v8M21 8v8"/>'
  +'<path d="M6 16a6 6 0 0 1 12 0" stroke-dasharray="2.5 2"/>',
 bidet:'<rect x="8" y="4" width="8" height="2.6" rx="1"/>'
  +'<ellipse cx="12" cy="13.5" rx="4" ry="6"/>'
  +'<circle cx="12" cy="13.5" r="1"/>',
 ur:'<path d="M6 5h12v5l-2.5 5.5L12 19l-3.5-3.5L6 10z"/>'
  +'<ellipse cx="12" cy="11.5" rx="2.6" ry="3.4"/>',
 wm:'<rect x="5" y="4" width="14" height="16" rx="1"/>'
  +'<circle cx="12" cy="13" r="4.2"/><path d="M5 8h14"/>',
 fd:'<rect x="7" y="7" width="10" height="10"/>'
  +'<path d="M7 7l10 10M17 7L7 17" opacity=".5"/>',
 chaincmp:'<path d="M3 8h18M3 8v3M9 8v3M15 8v3M21 8v3"/>'
  +'<path d="M4 17h6M14 17h6"/><path d="M11 15.5 13 17l-2 1.5"/>',
 refalign:'<rect x="3" y="6" width="8" height="12" '
  +'stroke-dasharray="3 2"/><rect x="13" y="6" width="8" height="12"/>'
  +'<path d="M11 12h2"/>',
 refcal:'<path d="M4 16 16 4l4 4L8 20z"/>'
  +'<path d="M8 12l2 2M11 9l2 2" opacity=".65"/>',
 refmove:'<rect x="3" y="7" width="10" height="10" '
  +'stroke-dasharray="3 2"/>'
  +'<path d="M15 12h6M21 12l-2.4-2.4M21 12l-2.4 2.4"/>',

 /* ═══ و١ — الملفّ والقشرة ═══ */
 menu:'<path d="M4 7h16M4 12h16M4 17h16"/>',
 dxf:'<path d="M6 3h8l4 4v14H6z"/><path d="M14 3v4h4"/>'
  +'<path d="M8.5 12l3 6M11.5 12l-3 6M13.5 12v6M13.5 12h2.4'
  +'M13.5 15h2" opacity=".8"/>',
 svg:'<path d="M6 3h8l4 4v14H6z"/><path d="M14 3v4h4"/>'
  +'<path d="M9 13.5a5 5 0 0 1 6 0" opacity=".8"/>'
  +'<circle cx="9" cy="16" r="1"/><circle cx="15" cy="16" r="1"/>',
 png:'<rect x="3" y="5" width="18" height="14" rx="1"/>'
  +'<circle cx="8.5" cy="10" r="1.6"/><path d="M3 17l5-4 4 3 4-4 5 5"/>',
 pdf:'<path d="M6 3h8l4 4v14H6z"/><path d="M14 3v4h4"/>'
  +'<path d="M9 12v6M9 12h2a1.6 1.6 0 0 1 0 3.2H9M14 12v6M14 12h2.4'
  +'M14 15h2" opacity=".8"/>',
 csv:'<rect x="3" y="5" width="18" height="14"/>'
  +'<path d="M3 10h18M9 5v14M15 5v14" opacity=".55"/>'
  +'<path d="M5.5 13.5h1.6M11.5 13.5h1.6M17.5 13.5h1"/>',
 clean:'<path d="M4 4h5M4 4v5M20 4h-5M20 4v5'
  +'M4 20h5M4 20v-5M20 20h-5M20 20v-5"/>',
 rbmin:'<path d="M3 5h18"/><path d="M8 11l4 4 4-4" />'
  +'<path d="M4 20h16" opacity=".4"/>',
 theme:'<circle cx="12" cy="12" r="7.5"/>'
  +'<path d="M12 4.5a7.5 7.5 0 0 0 0 15z" fill="currentColor"'
  +' stroke="none"/>',
 shell:'<rect x="3" y="4" width="18" height="16" rx="1"/>'
  +'<path d="M3 9h18"/><path d="M8 4v5M13 4v5" opacity=".55"/>',
 /* لوحةُ المساعد — شرارةٌ رباعيّة الرؤوس، بلا صلةٍ بأيقونةٍ أخرى */
 ai:'<path d="M12 3l1.8 5.2L19 10l-5.2 1.8L12 17l-1.8-5.2L5 10'
  +'l5.2-1.8z"/><path d="M18.5 15.5l.8 2.1 2.1.8-2.1.8-.8 2.1'
  +'-.8-2.1-2.1-.8 2.1-.8z" opacity=".6"/>',
 del:'<path d="M5 7h14"/><path d="M9 7V4h6v3"/>'
  +'<path d="M6.5 7l1 13h9l1-13"/><path d="M10 11v6M14 11v6"'
  +' opacity=".6"/>',
 props:'<rect x="3" y="4" width="18" height="16" rx="1"/>'
  +'<path d="M3 8h18"/><path d="M6 12h5M6 16h5M13 12h5M13 16h5"'
  +' opacity=".65"/>',
 unlock:'<rect x="5" y="11" width="14" height="9" rx="1"/>'
  +'<path d="M8.5 11V8a3.5 3.5 0 0 1 7 0"/>',
 renum:'<path d="M4 6h3M5.5 6v5M4 11h3"/>'
  +'<path d="M4 15h3v2.5H4V20h3" opacity=".8"/>'
  +'<path d="M11 8h9M11 13h9M11 18h9" opacity=".55"/>'
 ,
 /* ═══ و٢ — الإرساء ═══ */
 dockS:'<rect x="3" y="4" width="18" height="16" rx="1"/>'
  +'<path d="M14 4v16"/><path d="M16.5 8h2M16.5 11h2" opacity=".6"/>',
 dockE:'<rect x="3" y="4" width="18" height="16" rx="1"/>'
  +'<path d="M10 4v16"/><path d="M5.5 8h2M5.5 11h2" opacity=".6"/>',
 float:'<rect x="3" y="6" width="13" height="12" rx="1"'
  +' stroke-dasharray="3 2"/>'
  +'<rect x="8" y="3" width="13" height="12" rx="1"/>'
  +'<path d="M8 6.6h13" opacity=".6"/>',
 close:'<path d="M6 6l12 12M18 6L6 18"/>',
 maxi:'<rect x="4" y="5" width="16" height="14" rx="1"/>'
  +'<path d="M4 9h16" opacity=".6"/>',
 autohide:'<rect x="3" y="4" width="4" height="16" rx="1"/>'
  +'<path d="M11 12h9M20 12l-2.6-2.6M20 12l-2.6 2.6"/>',
 tabs:'<rect x="3" y="7" width="18" height="13" rx="1"/>'
  +'<path d="M3 7h6V4h6v3" opacity=".8"/>',
 acc:'<path d="M3 5h18M3 12h18M3 19h18"/>'
  +'<path d="M5.6 8.4 7 7l1.4 1.4" opacity=".55"/>',
 up:'<path d="M12 19V5"/><path d="M6.6 10.4 12 5l5.4 5.4"/>',
 down:'<path d="M12 5v14"/><path d="M6.6 13.6 12 19l5.4-5.4"/>',
 ws:'<rect x="3" y="4" width="18" height="16" rx="1"/>'
  +'<path d="M3 9h18M9 9v11" opacity=".65"/>'
  +'<path d="M12 13h6M12 16h6" opacity=".45"/>',
 panel:'<rect x="3" y="4" width="18" height="16" rx="1"/>'
  +'<path d="M15 4v16M3 8h12" opacity=".65"/>'
 ,
 /* ═══ و٣ — شريط الحالة والتنقّل ═══ */
 gsnap:'<path d="M4 4h16v16H4z" opacity=".35"/>'
  +'<path d="M9.3 4v16M14.6 4v16M4 9.3h16M4 14.6h16" opacity=".35"/>'
  +'<circle cx="9.3" cy="14.6" r="2.2"/>',
 paths:'<path d="M3 8h18M3 16h18" opacity=".4"/>'
  +'<path d="M3 12h18" stroke-dasharray="4 3"/>',
 lock:'<rect x="5" y="11" width="14" height="9" rx="1"/>'
  +'<path d="M8.5 11V8a3.5 3.5 0 0 1 7 0v3"/>',
 warn:'<path d="M12 3.5 21 19.5H3z"/>'
  +'<path d="M12 9.5v4.4M12 16.4v.3"/>',
 zwin:'<rect x="3" y="5" width="12" height="10" stroke-dasharray="3 2"/>'
  +'<circle cx="15" cy="15" r="4.6"/><path d="M18.4 18.4 21.5 21.5"/>',
 zprev:'<circle cx="11" cy="11" r="6.4"/><path d="M15.6 15.6 21 21"/>'
  +'<path d="M15.4 5.6a7 7 0 0 0-9.8.8"/><path d="M5 3.6v3.2h3.2"/>',
 zin:'<circle cx="10.5" cy="10.5" r="6.4"/><path d="M15.2 15.2 21 21"/>'
  +'<path d="M8 10.5h5M10.5 8v5"/>',
 zout:'<circle cx="10.5" cy="10.5" r="6.4"/><path d="M15.2 15.2 21 21"/>'
  +'<path d="M8 10.5h5"/>',
 pan:'<path d="M12 3v18M3 12h18" opacity=".45"/>'
  +'<path d="M8.6 6.4 12 3l3.4 3.4M8.6 17.6 12 21l3.4-3.4'
  +'M6.4 8.6 3 12l3.4 3.4M17.6 8.6 21 12l-3.4 3.4"/>',
 view:'<path d="M2 12s3.6-6 10-6 10 6 10 6-3.6 6-10 6-10-6-10-6z"/>'
  +'<circle cx="12" cy="12" r="2.8"/>',
 north:'<circle cx="12" cy="12" r="8.6" opacity=".45"/>'
  +'<path d="M12 3.4 15 12h-6z"/><path d="M12 12v8.6" opacity=".6"/>'
 ,
 /* ═══ و٤ — الإدخال والقوائم ═══ */
 dyn:'<path d="M4 12h5M15 12h5M12 4v5M12 15v5" opacity=".5"/>'
  +'<rect x="9" y="9" width="11" height="6" rx="1"/>'
  +'<path d="M11.4 11v2" opacity=".8"/>',
 cmdl:'<rect x="3" y="6" width="18" height="12" rx="1"/>'
  +'<path d="M6.5 10l2 2-2 2"/><path d="M10.5 14h5"/>',
 ctx:'<rect x="4" y="4" width="16" height="16" rx="1"/>'
  +'<path d="M7.5 9h9M7.5 12h9M7.5 15h5" opacity=".7"/>',
 chamfer:'<path d="M5 20V9l4-4h11"/>'
  +'<path d="M5 9V5h4" opacity=".4" stroke-dasharray="2 2"/>',
 arraypol:'<circle cx="12" cy="12" r="7" opacity=".4"'
  +' stroke-dasharray="3 2"/>'
  +'<path d="M10.5 3h3v3h-3zM18 10.5h3v3h-3z'
  +'M10.5 18h3v3h-3zM3 10.5h3v3H3z"/>',
 qp:'<rect x="3" y="5" width="18" height="9" rx="1"/>'
  +'<path d="M3 9.5h18" opacity=".5"/>'
  +'<path d="M6 7.2h3M13 7.2h5M6 11.8h4M14 11.8h4" opacity=".7"/>'
  +'<path d="M8 17l2 2 2-2" opacity=".5"/>',
 opa:'<circle cx="12" cy="12" r="8"/>'
  +'<path d="M12 4a8 8 0 0 0 0 16z" fill="currentColor"'
  +' stroke="none" opacity=".38"/>',
 clear:'<path d="M4 7h16"/><path d="M9 7V4.5h6V7"/>'
  +'<path d="M6.5 7l1 12.5h9L17.5 7"/>'
};
export const iconNames=()=>Object.keys(ICONS);
export const hasIcon=n=>!!ICONS[n];

/* يُنادى مرّةً في الإقلاع قبل بناء أي شريط */
export function mountIcons(){
 if(document.getElementById("icoSheet"))return 0;
 const d=document.createElement("div");
 d.id="icoSheet";
 d.setAttribute("aria-hidden","true");
 d.innerHTML=`<svg xmlns="http://www.w3.org/2000/svg" width="0" height="0">`
  +iconNames().map(k=>`<symbol id="i-${k}" viewBox="0 0 24 24">`
   +`${ICONS[k]}</symbol>`).join("")
  +`</svg>`;
 document.body.appendChild(d);
 return iconNames().length;
}
export function icon(n,size){
 if(!ICONS[n])return "";
 const s=size||16;
 return `<svg class="ic" width="${s}" height="${s}" viewBox="0 0 24 24"`
  +` aria-hidden="true" focusable="false"><use href="#i-${n}"/></svg>`;
}
```

<a id="f-js-ui-inspector-js"></a>

---

## `js/ui/inspector.js`

```javascript
/* ═══ لوحة الفاحص والورقة والمرجع والتصدير ═══
   الفاحص يُنفَّذ بزرّ. كل سطر قابل للنقر يحدّد العنصر ويقفز إليه،
   ولا يعدّل شيئاً. */
import {S,edit,editFailed} from "../core/state.js";
import {m2,m3,clamp,dim2,dm2,scl} from "../core/units.js";
import {inspect,SEV} from "../core/inspect.js";
import {openSchedule,okName,OK} from "../core/opens.js";
import {schedule,netArea,netPerim,isStale} from "../core/areas.js";
/* ═══ التصدير ═══ الفاحص لا يعرف صيغةً: المصدِّر يعرفها وحده ═══ */
import * as EX from "../io/export.js";
import {scene,sceneBBox} from "../core/render.js";
import {paperMM,SNAMES,fitsSheet} from "../core/sheet.js";
import {anyHidden,pickable} from "../core/layers.js";
import {hasRef,refCount,clearRef,resetRef,srcList,srcOn,srcSet,
        srcShown,isIdent,refTr,refSnapCount,refStats,setRef,
        KIND} from "../core/ref.js";
import {parseDXF,decodeDXF,skipSummary,MAXENT,MAXPTS,
        MAXCO} from "../io/dxfin.js";
import {toJSON,fromJSON,dl,pickFile,pickBin,
        humanSize} from "../io/project.js";
import {setSel,fitBox,draw,V} from "./canvas.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");

let LAST=null;

export function runInspect(){
 LAST=inspect(sceneBBox());
 renderFindings();
 const {er,wr}=LAST;
 HOOK.report(er?"er":(wr?"wr":"ok"),
  (er||wr||LAST.in)
   ? `الفاحص: ${er} خطأ · ${wr} تنبيه · ${LAST.in} ملاحظة`
   : "الفاحص: لا ملاحظات");
 return LAST;
}
export const clearFindings=()=>{LAST=null; renderFindings()};

function renderFindings(){
 const box=$("#insp");
 if(!box)return;
 if(!LAST){
  box.innerHTML=`<p class="hint">اضغط «افحص» — لا شيء يُفحَص `
   +`تلقائياً مع كل رسمة</p>`;
  return;
 }
 if(!LAST.list.length){
  box.innerHTML=`<p class="hint" style="color:var(--ok)">`
   +`لا ملاحظات · المخطط سليم بحسب الفحوص المتاحة</p>`;
  return;
 }
 const CL={er:"var(--er)",wr:"var(--wr)",in:"var(--fg2)"};
 box.innerHTML=`<p class="hint">${LAST.er} خطأ · ${LAST.wr} تنبيه `
  +`· ${LAST.in} ملاحظة — انقر السطر للقفز</p>`
  +LAST.list.slice(0,120).map((f,i)=>
   `<div class="ro" data-ins="${i}" style="cursor:pointer;`
   +`margin:2px 0;color:${CL[f.sev]}">`
   +`${esc(SEV[f.sev])}: ${esc(f.msg)}</div>`).join("")
  +(LAST.list.length>120
   ? `<p class="hint">… و ${LAST.list.length-120} ملاحظة أخرى</p>`
   : "")
  +`<p class="hint">الفاحص لا يصلح شيئاً — يخبرك وأنت تقرّر</p>`;
}
/* ═══ الورقة ═══ */
export function syncSheet(){
 if(!$("#shOn"))return;
 $("#shOn").checked=!!+S.sheet.on;
 $("#shSize").value=S.sheet.size;
 $("#shOr").value=S.sheet.orient;
 $("#shMar").value=S.sheet.margin;
 $("#shTb").checked=!!+S.sheet.tb;
 const t=S.title||{};
 $("#tProj").value=t.proj||"";
 $("#tOwner").value=t.owner||"";
 $("#tLoc").value=t.loc||"";
 $("#tSheet").value=t.sheet||"";
 $("#tRev").value=t.rev||"";
 $("#tBy").value=t.by||"";
 const p=paperMM();
 const f=fitsSheet(sceneBBox());
 const el=$("#shInfo");
 if(el)el.innerHTML=`${esc(dim2(p[0],p[1],"مم"))} · ${esc(scl(S.meta.scale))} `
  +`⇒ ${esc(dim2((p[0]*S.meta.scale/1000).toFixed(1),
      (p[1]*S.meta.scale/1000).toFixed(1),"م"))} نموذجياً`
  +(+S.sheet.on
   ? (f.ok?`<br><span style="color:var(--ok)">الرسم داخل الإطار`
     +`</span>`
     :`<br><span class="warn">يتجاوز الإطار بـ ${m2(f.over)} م`
     +`</span>`)
   : "");
}
/* ═══ المرجع ═══ */
export function renderRef(){
 const box=$("#rInfo"), lay=$("#rLays");
 if(!box)return;
 if(!hasRef()){
  box.innerHTML=`<span class="hint">لا مرجع — «استورد DXF» يضع `
   +`الملفّ خلفيةً للقياس والالتقاط</span>`;
  if(lay)lay.innerHTML="";
  return;
 }
 const r=refStats(), t=refTr();
 const by=Object.keys(r.by).map(k=>`${r.by[k]} ${KIND[k]||k}`)
  .join(" · ");
 box.innerHTML=`<b>${esc(r.name||"بلا اسم")}</b><br>`
  +`${r.n} كياناً (${by})<br>`
  +`الوحدة: ${esc(r.units)}`
  +(r.guessed?` <span class="warn">(مفترضة)</span>`:"")
  +` · الترميز ${esc(r.enc)}<br>`
  +`التحويل: ×${t.k.toFixed(5)} · ${t.rot.toFixed(2)}° · `
  +`${m2(t.dx)},${m2(t.dy)} م`
  +(isIdent()?` <span class="hint">(لم يُحاذَ بعد)</span>`:"")
  +`<br>${refSnapCount()} نقطة التقاط`
  +(r.trunc?`<br><span class="warn">استُثني ${r.trunc} كياناً `
   +`لتجاوز الحدّ</span>`:"");
 if(!lay)return;
 const L=srcList();
 lay.innerHTML=`<p class="hint">طبقات الملفّ `
  +`(${srcShown()}/${L.length}) — الإخفاء هنا داخل المرجع وحده`
  +`</p>`
  +L.slice(0,60).map(n=>{
   const on=srcOn(n);
   return `<div style="display:flex;gap:5px;align-items:center;`
    +`margin:1px 0;opacity:${on?1:.45}">`
    +`<button data-rsrc="${esc(n)}" style="width:22px;`
    +`padding:1px 0;flex:none">${on?"◉":"○"}</button>`
    +`<span style="flex:1;font-size:11px;overflow:hidden;`
    +`text-overflow:ellipsis;white-space:nowrap">${esc(n)}</span>`
    +`<span class="ro" style="font-size:10.5px">`
    +`${S.ref.src[n]}</span></div>`;
  }).join("")
  +(L.length>60?`<p class="hint">… و ${L.length-60} طبقة</p>`:"");
}
/* ═══ التوصيل ═══ */
export function wireInspector(){
 const box=$("#insp");
 if(box)box.addEventListener("click",e=>{
  const t=e.target.closest("[data-ins]");
  if(!t||!LAST)return;
  const f=LAST.list[+t.dataset.ins];
  if(!f)return;
  /* القفز يقع دائماً — ترى موضع الملاحظة ولو كان مقفلاً.
     والتحديد يمرّ بالحرس نفسه: ما لا يُحدَّد لا يُسحَب. */
  if(f.k&&f.id){
   const s={k:f.k,id:f.id};
   if(pickable(s))setSel([s],s);
   else HOOK.report("in",`${f.id} على طبقةٍ مخفيّة أو مقفلة — `
    +`قُفِز إلى موضعه ولم يُحدَّد`);
  }
  if(f.p){
   const r=Math.max(1500,2200/Math.max(1e-6,V.k));
   fitBox({x0:f.p[0]-r,y0:f.p[1]-r,x1:f.p[0]+r,y1:f.p[1]+r},0.1);
  }
  draw();
  HOOK.status(f.msg);
 });
 $("#bInsp").onclick=()=>runInspect();

 /* ═══ CSV ═══ BOM لأجل إكسل العربي · CRLF لأجل ويندوز ═══ */
 const csv=rows=>"\uFEFF"+rows.map(r=>r.map(c=>{
  const s=String(c==null?"":c);
  return /[",\n\r]/.test(s)?`"${s.replace(/"/g,'""')}"`:s;
 }).join(",")).join("\r\n")+"\r\n";

 $("#bOsCsv").onclick=()=>{
  const d=openSchedule();
  if(!d.rows.length){HOOK.report("in","لا فتحات");return}
  const R2=[["الرمز","النوع","العرض م","الارتفاع م","الجلسة م",
   "المصاريع","عمق الكوّة م","العدد","المعرّفات"]];
  d.rows.forEach(r=>R2.push([r.mark,okName(r.kind),m2(r.w),m2(r.h),
   m2(r.sill),r.pan,r.dep?m2(r.dep):"",r.n,r.ids.join(" ")]));
  R2.push(["","المجموع","","","","","",d.total,""]);
  const n=dl(EX.safeName(S.meta.name+"-فتحات","csv"),csv(R2),
   "text/csv;charset=utf-8");
  HOOK.report("ok",`جدول الفتحات · ${d.rows.length} نوعاً · `
   +`${humanSize(n)}`);
 };
 $("#bAsCsv").onclick=()=>{
  const d=schedule();
  if(!d.rows.length){HOOK.report("in","لا مناطق");return}
  const R2=[["المنطقة","المساحة م²","المحيط م","الحالة"]];
  d.rows.forEach(r=>R2.push([r.name,(r.ar/1e6).toFixed(2),
   m3(r.pr),r.stale?"قديمة":"مطابقة"]));
  R2.push(["المجموع",(d.total/1e6).toFixed(2),"",""]);
  const n=dl(EX.safeName(S.meta.name+"-مساحات","csv"),csv(R2),
   "text/csv;charset=utf-8");
  HOOK.report("ok",`جدول المساحات · ${d.rows.length} منطقة · `
   +`${humanSize(n)}`);
 };
 /* ═══ التصدير ═══
    زرٌّ واحدٌ أربع مرّات: النطاق والتسمية والحصيلة والتحذيرات في
    io/export.js، وهنا التنزيل وحده. وكانت خمسةُ أشياءَ مكرّرةً
    أربع مرّات، فانجرفت: warnClip في الأربعة وsayNotes في اثنين،
    وزرّان غير متزامنَين واثنان متزامنان.
    والخيارات قراءةٌ واحدة: «ألوان التنبيه» مُطفأٌ افتراضاً لأن
    التسليم للعميل لا يحمل ألوان تشخيص. */
 const wantWarn=()=>!!($("#xWarn")&&$("#xWarn").checked);
 const wantDark=()=>!!($("#xDark")&&$("#xDark").checked);
 const dpiOf=()=>clamp(parseInt(($("#xDpi")||{}).value,10)||300,
  72,1200);
 const syncExport=()=>{
  const el=$("#xInfo");
  if(el)el.textContent=EX.summary();
 };
 const BTN={xDxf:"dxf",xSvg:"svg",xPng:"png",xPdf:"pdf"};
 const busy=v=>Object.keys(BTN).forEach(id=>{
  const b=$("#"+id);
  if(b)b.disabled=!!v;
 });
 Object.keys(BTN).forEach(id=>{
  const el=$("#"+id);
  if(!el)return;
  el.onclick=async()=>{
   const fmt=BTN[id];
   busy(1);
   HOOK.status(`يُبنى ${EX.FMT[fmt].n}…`);
   try{
    const r=await EX.run(fmt,{dark:wantDark(),
     showWarn:wantWarn(), dpi:dpiOf()});
    if(r.ok)dl(r.name, r.blob||r.raw, r.mime);
    r.report.forEach(m=>HOOK.report(m.lv,m.s));
   }catch(e){
    HOOK.report("er",`${EX.FMT[fmt].n}: `+String(e.message||e));
   }
   busy(0);
   syncExport();
  };
 });
 /* السطر يُحدَّث عند فتح القسم وبعد كل تصدير وبعد تغيّر الورقة:
    مقاسُها واتجاهُها ورقمُ اللوحة كلُّها فيه. */
 const sec=document.querySelector('details[data-sec="export"]');
 if(sec)sec.addEventListener("toggle",syncExport);
 syncExport();

 $("#xSave").onclick=()=>{
  const n=dl(EX.safeName(S.meta.name,"json"),toJSON(),
   "application/json");
  HOOK.report("ok",`حُفظ المشروع · ${humanSize(n)}`);
  if(anyHidden()||hasRef())HOOK.report("in",
   "الملفّ يحمل كل شيء بما فيه المخفيّ والمرجع — الإخفاء عرضٌ "
   +"لا حذف. وهو ليس مخرَجاً: بلا هيئةٍ ولا نطاقٍ ولا مقياس.");
 };
 $("#xOpen").onclick=()=>{
  pickFile((txt,name)=>{
   if(txt==null){HOOK.report("wr","لم يُقرأ ملفّ");return}
   if((S.walls.length||S.cols.length)
    &&!confirm("فتح ملفّ؟ سيُفقد غير المحفوظ."))return;
   try{
    const r=fromJSON(txt);
    LAST=null;
    HOOK.refresh(1);
    HOOK.report("ok",`فُتح ${name} · ${r.walls} جدار · `
     +`${r.opens} فتحة · ${r.cols} عمود · ${r.areas} منطقة · `
     +`${r.dims} بُعد`+(r.ref?` · مرجع ${r.ref} كياناً`:""));
    syncExport();
    import("./canvas.js").then(C=>C.fit());
   }catch(e){HOOK.report("er",e.message)}
  });
 };
 /* ═══ الورقة ═══ */
 $("#shOn").onchange=()=>{
  edit(()=>{S.sheet.on=$("#shOn").checked?1:0});
  HOOK.refresh(0);
  syncExport();
 };
 ["shSize","shOr","shMar","shTb"].forEach(id=>{
  const el=$("#"+id);
  if(!el)return;
  el.onchange=()=>{
   edit(()=>{
    S.sheet.size=SNAMES.includes($("#shSize").value)
     ? $("#shSize").value : "A3";
    S.sheet.orient=($("#shOr").value==="p")?"p":"l";
    S.sheet.margin=clamp(parseFloat($("#shMar").value)||12,0,60);
    S.sheet.tb=$("#shTb").checked?1:0;
   });
   HOOK.refresh(0);
   syncExport();
  };
 });
 $("#shCenter").onclick=()=>{
  edit(()=>{S.sheet.cx=null; S.sheet.cy=null});
  HOOK.report("in","الورقة تُتَمركَز على الرسم");
  HOOK.refresh(0);
  syncExport();
 };
 ["tProj","tOwner","tLoc","tSheet","tRev","tBy"].forEach(id=>{
  const el=$("#"+id);
  if(!el)return;
  el.onchange=()=>{
   edit(()=>{
    S.title.proj=$("#tProj").value.slice(0,60);
    S.title.owner=$("#tOwner").value.slice(0,60);
    S.title.loc=$("#tLoc").value.slice(0,60);
    S.title.sheet=$("#tSheet").value.slice(0,16);
    S.title.rev=$("#tRev").value.slice(0,8);
    S.title.by=$("#tBy").value.slice(0,30);
   });
   HOOK.refresh(0);
   syncExport();      /* رقمُ اللوحة والمراجعة في اسم الملفّ */
  };
  el.onkeydown=e=>{
   if(e.key==="Escape"){el.blur();return}
   e.stopPropagation();
  };
 });
 /* ═══ المرجع ═══ */
 $("#rImp").onclick=()=>{
  pickBin(".dxf",(buf,name)=>{
   if(!buf){HOOK.report("wr","لم يُقرأ ملفّ");return}
   try{
    const {txt,enc}=decodeDXF(buf);
    const u=$("#rUnit").value;
    const res=parseDXF(txt,{unit:u?parseFloat(u):null,
     cap:MAXENT});
    res.enc=enc;
    if(!res.ents.length)
     throw new Error("لم يُنتج الملفّ كياناً واحداً مقروءاً");
    edit(()=>setRef(res,name));
    HOOK.report("ok",`استُورد ${name} · ${res.ents.length} كياناً `
     +`على ${Object.keys(res.src).length} طبقة · الوحدة `
     +`${res.units.name}${res.guessed?" (مفترضة)":""} · `
     +`الترميز ${enc} · ${humanSize(buf.byteLength||0)}`);
    /* التوقّف يُعلَن أوّلاً: المرجع منقوصٌ بقدرٍ لا يُعرَف، فلا
       يُقاس عليه — وأخطر ما في القراءة أن تمضي وأنت تحسبها تمّت */
    if(res.stop)HOOK.report("er",
     res.stop==="ops"
      ? `تُوقّف التحليل عند ${res.ops.toLocaleString("en")} عملية — `
        +`الملفّ يحوي بلوكاتٍ متعشّقة عميقاً أو مصفوفاتٍ ضخمة. `
        +`المرجع منقوصٌ بقدرٍ لا يُعرَف — لا تقِس عليه.`
      : `تُوقّف التحليل بعد ${Math.round(res.ms/1000)} ثانية — `
        +`الملفّ أكبر مما يُحلَّل في المتصفّح. جرّب حفظه من `
        +`برنامجك بعد حذف ما لا تحتاجه.`);
    if(res.clipped)HOOK.report("wr",
     `${res.clipped} كياناً قُصَّت رؤوسه عند ${MAXPTS} — `
     +`مضلّعاتٌ بعشرات الآلاف من النقاط لا تُقاس عليها`);
    const sk2=res.skip["قيمة خارج المدى"];
    if(sk2)HOOK.report("wr",`${sk2} كياناً نُبِذ لقيَمٍ خارج `
     +`المدى (±${(MAXCO/1e6).toFixed(0)} كم) — وحدةٌ خاطئة أو `
     +`ملفٌّ معطوب`);
    const sk=skipSummary(res.skip,["قيمة خارج المدى"]);
    if(sk)HOOK.report("in",`تُخطّي: ${sk}`);
    if(res.trunc)HOOK.report("wr",`استُثني ${res.trunc} كياناً `
     +`لتجاوز الحدّ (${MAXENT}) — المرجع منقوص`);
    if(res.approx.spline)HOOK.report("in",
     `${res.approx.spline} منحنى SPLINE رُسم متقطّعاً — `
     +`التقطيع علامةُ التقريب`);
    if(res.guessed)HOOK.report("wr",
     "وحدة الملفّ مجهولة وفُرضت مليمتراً — عاير المرجع بمسافةٍ "
     +"تعرفها قبل أن تقيس عليه");
    HOOK.report("in","المرجع جامد: لا يُحدَّد ولا يدخل المساحات. "
     +"استعمل «محاذاة» لتضعه في موضعه.");
    HOOK.refresh(1);
    import("./canvas.js").then(C=>C.fit());
   }catch(e){HOOK.report("er","DXF: "+e.message)}
  });
 };
 $("#rClr").onclick=()=>{
  if(!hasRef()){HOOK.report("in","لا مرجع");return}
  if(!confirm(`إزالة المرجع (${refCount()} كياناً)؟ `
   +`رسمك لا يتأثّر.`))return;
  const n=edit(()=>clearRef());
  if(editFailed())return;
  HOOK.report("ok",`أُزيل المرجع · ${n} كياناً · الملفّ يصغر`);
  HOOK.refresh(1);
 };
 $("#rRst").onclick=()=>{
  if(!hasRef())return;
  edit(()=>resetRef());
  HOOK.report("in","صُفّر تحويل المرجع — عاد إلى إحداثياته "
   +"المستوردة");
  HOOK.refresh(0);
 };
 const rl=$("#rLays");
 if(rl)rl.addEventListener("click",e=>{
  const n=e.target.dataset.rsrc;
  if(!n)return;
  const on=edit(()=>srcSet(n,srcOn(n)));
  if(editFailed())return;
  HOOK.report("in",`طبقة المرجع «${n}»: ${on?"ظاهرة":"مخفيّة"}`);
  HOOK.refresh(0);
 });
}
```

<a id="f-js-ui-layout-js"></a>

---

## `js/ui/layout.js`

```javascript
/* ═══ مواضع اللوحات وأسطح العمل ═══
   سجلٌّ للموضع وحده. الرسم في panels.js، والتسمية في اللوحة نفسها
   (نصّ summary) فلا تُكتَب مرّتين ولا تتخلّف إحداهما عن الأخرى.
   المعرّفات هي data-sec القائمة في props.js — وdom.js يقابل
   القائمتين فلا تنفرد إحداهما بعنصر.

   الأعمدة: s بداية السطر (يمين في العربية) · e نهايته.
   وz للوحة: s | e | f عائمة | x مغلقة. */

export const PANELS=[
 {id:"proj",   ico:"props",   z:"s", o:1},
 {id:"lays",   ico:"layers",  z:"s", o:1},
 {id:"props",  ico:"panel",   z:"s", o:1},
 {id:"sched",  ico:"table",   z:"s", o:0},
 {id:"osched", ico:"table",   z:"s", o:0},
 {id:"axes",   ico:"axis",    z:"s", o:0},
 {id:"ref",    ico:"ref",     z:"s", o:0},
 {id:"sheet",  ico:"sheet",   z:"s", o:0},
 {id:"insp",   ico:"inspect", z:"s", o:1},
 {id:"export", ico:"xport",   z:"s", o:0},
 {id:"defs",   ico:"shell",   z:"s", o:0},
 {id:"ai",     ico:"ai",      z:"s", o:0},
 {id:"state",  ico:"grid",    z:"s", o:1}
];
export const pIds=()=>PANELS.map(p=>p.id);
export const pDef=id=>PANELS.find(p=>p.id===id)||null;
export const isPanel=id=>!!pDef(id);

export const ZONES=["s","e"];
export const ZN={s:"العمود الأيمن",e:"العمود الأيسر",
 f:"عائمة",x:"مغلقة"};
export const MODES={acc:"أقسام",tab:"تبويبات"};
export const WMIN=210, WMAX=560;

/* ═══ أسطح العمل المدمجة ═══
   s و e قائمتان مرتَّبتان · ما لم يُذكَر فيهما مغلق · open ما يُفتَح.
   والقصد أن كل سطحٍ يعرض ما يخصّ عمله ويُغلق ما عداه — لا لوحةً
   تُبنى لتُخفى. */
export const WS={
 arch:{n:"معماري", shell:"ribbon", tab:"arch", clean:0,
  zw:{s:312,e:270}, mode:{s:"acc",e:"acc"}, auto:{s:0,e:0},
  s:["proj","lays","props","insp","ai","state"], e:[],
  open:["lays","props","insp"]},

 annot:{n:"تأشير", shell:"ribbon", tab:"annt", clean:0,
  zw:{s:300,e:288}, mode:{s:"acc",e:"tab"}, auto:{s:0,e:0},
  s:["props","lays","state"], e:["sched","osched","axes"],
  open:["props","sched","osched","axes"]},

 out:{n:"إخراج", shell:"ribbon", tab:"out", clean:0,
  zw:{s:312,e:264}, mode:{s:"acc",e:"acc"}, auto:{s:0,e:0},
  s:["sheet","export","state"], e:["insp","lays"],
  open:["sheet","export","insp"]},

 full:{n:"كامل", shell:"ribbon", tab:"home", clean:0,
  zw:{s:312,e:270}, mode:{s:"acc",e:"acc"}, auto:{s:0,e:0},
  s:["proj","lays","props","sched","osched","axes","ai"],
  e:["ref","sheet","insp","export","defs","state"],
  open:["lays","props","insp","state"]},

 bare:{n:"بلا شريط", shell:"classic", tab:"home", clean:0,
  zw:{s:288,e:264}, mode:{s:"acc",e:"acc"}, auto:{s:1,e:0},
  s:["props","lays","insp","state"], e:[],
  open:["props","lays"]}
};

/* ═══ التخطيط الافتراضي ═══ يوافق ما كان قبل و٢ حرفياً ═══ */
export const DEFLAY=()=>({
 zw:{s:312,e:270},
 mode:{s:"acc",e:"acc"},
 auto:{s:0,e:0},
 cur:{s:null,e:null},          /* اللوحة الجارية في وضع التبويبات */
 p:PANELS.reduce((a,p)=>{
  a[p.id]={z:p.z,i:PANELS.indexOf(p),o:p.o,
   x:120,y:110,w:320,h:300,max:0};
  return a;
 },{})
});
/* ═══ التطبيع الدفاعي ═══
   تخطيطٌ محرَّر يدوياً أو من إصدارٍ أقدم يُصلَح شكلاً: لوحةٌ مجهولة
   تُنبذ، وناقصةٌ تُستكمل من مصنعها. */
export function normLay(L){
 const d=DEFLAY();
 const o={zw:{},mode:{},auto:{},cur:{},p:{}};
 const num=(v,a,b,f)=>{
  const n=Math.round(+v);
  return isFinite(n)?Math.max(a,Math.min(b,n)):f;
 };
 ZONES.forEach(z=>{
  o.zw[z]=num(L&&L.zw&&L.zw[z],WMIN,WMAX,d.zw[z]);
  const m=L&&L.mode&&L.mode[z];
  o.mode[z]=MODES[m]?m:"acc";
  o.auto[z]=(L&&L.auto&&L.auto[z])?1:0;
  const c=L&&L.cur&&L.cur[z];
  o.cur[z]=isPanel(c)?c:null;
 });
 PANELS.forEach(p=>{
  const s=(L&&L.p&&L.p[p.id])||{};
  const z=/^(s|e|f|x)$/.test(s.z)?s.z:p.z;
  o.p[p.id]={z,
   i:num(s.i,0,999,PANELS.indexOf(p)),
   o:(s.o===undefined)?p.o:(s.o?1:0),
   x:num(s.x,-4000,8000,120), y:num(s.y,0,8000,110),
   w:num(s.w,220,1200,320), h:num(s.h,120,2000,300),
   max:s.max?1:0};
 });
 return o;
}
export const wsNorm=w=>{
 if(!w||typeof w!=="object")return null;
 const F=id=>isPanel(id);
 return {n:String(w.n||"سطح").slice(0,32),
  shell:(w.shell==="classic")?"classic":"ribbon",
  tab:String(w.tab||"home").slice(0,16),
  clean:w.clean?1:0,
  zw:{s:+w.zw?.s||312, e:+w.zw?.e||270},
  mode:{s:MODES[w.mode?.s]?w.mode.s:"acc",
        e:MODES[w.mode?.e]?w.mode.e:"acc"},
  auto:{s:w.auto?.s?1:0, e:w.auto?.e?1:0},
  s:(Array.isArray(w.s)?w.s:[]).filter(F),
  e:(Array.isArray(w.e)?w.e:[]).filter(F),
  open:(Array.isArray(w.open)?w.open:[]).filter(F)};
};
```

<a id="f-js-ui-navbar-js"></a>

---

## `js/ui/navbar.js`

```javascript
/* ═══ شريط التنقّل والمناظر المسمّاة ═══
   بديل عجلة أوتوكاد: أزرارٌ صريحة على حافة اللوحة.
   «تكبير نافذة» و«تحريك» وضعان مؤقّتان تُلغيهما Esc، ولا يدخلان
   سجلّ الأدوات لأنهما لا يُنشئان شيئاً ولا يعدّلان بياناتٍ — تنقّلٌ
   محض. ولهذا لا يظهران في المساعدة كأداتين.

   والمنظر إحداثيُّ عرضٍ لا بياناتُ رسم، فيسكن مخزن الواجهة: يبقى
   بين الجلسات ولا يُحمَل في ملفّ المشروع إلى حاسبٍ آخر. */
import {icon} from "./icons.js";
import {UIS,saveUI} from "./store.js";
import {V,fit,fitBox,zoomAt,navSet,navMode,zoomPrev,canPrev,
        draw} from "./canvas.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||16):"";

const BTN=[
 {act:"fit",   n:"ملاءمة",       ico:"fit"},
 {act:"zwin",  n:"تكبير نافذة",  ico:"zwin",  mode:"zw"},
 {act:"zprev", n:"المنظر السابق",ico:"zprev"},
 {act:"zin",   n:"تكبير",        ico:"zin"},
 {act:"zout",  n:"تصغير",        ico:"zout"},
 {act:"pan",   n:"تحريك",        ico:"pan",   mode:"pan"},
 {act:"views", n:"مناظر مسمّاة", ico:"view"}
];
export function buildNav(){
 const n=$("#navbar");
 if(!n)return 0;
 n.innerHTML=BTN.map(b=>
  `<button type="button" data-nav="${esc(b.act)}" `
  +`title="${esc(b.n)}" aria-label="${esc(b.n)}">`
  +`${ic(b.ico)}</button>`).join("");
 return BTN.length;
}
export function syncNav(){
 const n=$("#navbar");
 if(!n)return;
 const m=navMode();
 n.querySelectorAll("[data-nav]").forEach(b=>{
  const d=BTN.find(x=>x.act===b.dataset.nav);
  b.classList.toggle("on",!!(d&&d.mode&&d.mode===m));
 });
 const p=n.querySelector('[data-nav="zprev"]');
 if(p)p.disabled=!canPrev();
}
/* ═══ المناظر ═══ */
const views=()=>{
 if(!Array.isArray(UIS.views))UIS.views=[];
 return UIS.views;
};
export function viewSave(name){
 const nm=String(name||"").trim().slice(0,28);
 if(!nm){HOOK.report("wr","الاسم فارغ"); return false}
 const A=views();
 const v={n:nm,k:V.k,cx:Math.round(V.cx),cy:Math.round(V.cy)};
 const i=A.findIndex(x=>x.n===nm);
 if(i>=0)A[i]=v; else A.push(v);
 if(A.length>24)A.shift();
 saveUI();
 HOOK.report("ok",`حُفظ المنظر «${nm}»`);
 return true;
}
export function viewGo(name){
 const v=views().find(x=>x.n===name);
 if(!v)return false;
 navSet(null);
 const w=V.w/Math.max(1e-6,v.k), h=V.h/Math.max(1e-6,v.k);
 fitBox({x0:v.cx-w/2,y0:v.cy-h/2,x1:v.cx+w/2,y1:v.cy+h/2},0);
 HOOK.report("in",`المنظر «${name}»`);
 return true;
}
export function viewDel(name){
 const A=views();
 const i=A.findIndex(x=>x.n===name);
 if(i<0)return false;
 A.splice(i,1);
 saveUI();
 HOOK.report("ok",`حُذف المنظر «${name}»`);
 return true;
}
function vmOpen(x,y){
 const m=$("#vMenu");
 if(!m)return;
 const A=views();
 m.innerHTML=`<div class="pmH">مناظر مسمّاة</div>`
  +(A.length?A.map(v=>
   `<button type="button" class="pmI" data-vma="go:${esc(v.n)}">`
   +`${ic("view",14)}<span>${esc(v.n)}</span>`
   +`<span class="ky mono num">1:${Math.round(1/Math.max(1e-9,v.k))}`
   +`</span></button>`
   +`<button type="button" class="pmI pmDel" `
   +`data-vma="del:${esc(v.n)}" title="حذف">${ic("close",14)}`
   +`</button>`).join("")
   :`<div class="pmE">لا مناظر محفوظة</div>`)
  +`<div class="pmS"></div>`
  +`<button type="button" class="pmI" data-vma="save">`
  +`${ic("save",14)}<span>احفظ المنظر الحالي…</span></button>`;
 m.hidden=false;
 const rtl=getComputedStyle(document.documentElement)
  .direction==="rtl";
 const w=m.offsetWidth||234, h=m.offsetHeight||220;
 const cl=(v,a,b)=>v<a?a:(v>b?b:v);
 m.style.insetInlineStart=Math.round(
  cl(rtl?(innerWidth-x+4):(x-w-4),4,innerWidth-w-4))+"px";
 m.style.insetBlockStart=Math.round(cl(y,4,innerHeight-h-8))+"px";
}
const vmClose=()=>{const m=$("#vMenu"); if(m)m.hidden=true};

export function runNav(act){
 if(act==="fit"){navSet(null); fit(); HOOK.status("مُلوئم"); return}
 if(act==="zin"){zoomAt(V.w/2,V.h/2,1.25); return}
 if(act==="zout"){zoomAt(V.w/2,V.h/2,1/1.25); return}
 if(act==="zprev"){
  if(!zoomPrev())HOOK.report("in","لا منظر سابق");
  syncNav(); return;
 }
 if(act==="zwin"||act==="pan"){
  const m=(act==="zwin")?"zw":"pan";
  const on=navSet(navMode()===m?null:m);
  HOOK.status(on==="zw"?"اسحب إطار التكبير · Esc يلغي"
   :(on==="pan"?"اسحب لتحريك المنظر · Esc يلغي":""));
  syncNav(); draw(); return;
 }
 if(act==="views"){
  const b=document.querySelector('[data-nav="views"]');
  const r=b?b.getBoundingClientRect():{left:innerWidth-60,top:120};
  vmOpen(r.left,r.top);
 }
}
export function wireNav(){
 buildNav();
 const n=$("#navbar");
 if(!n)return false;
 n.addEventListener("click",e=>{
  const b=e.target.closest("[data-nav]");
  if(b&&!b.disabled)runNav(b.dataset.nav);
 });
 document.addEventListener("click",e=>{
  const a=e.target.closest("[data-vma]");
  if(!a)return;
  const v=a.dataset.vma;
  const g=/^go:(.+)$/.exec(v);
  if(g){vmClose(); viewGo(g[1]); return}
  const d=/^del:(.+)$/.exec(v);
  if(d){
   if(!confirm(`حذف المنظر «${d[1]}»؟`))return;
   viewDel(d[1]);
   const b=document.querySelector('[data-nav="views"]');
   const r=b?b.getBoundingClientRect():{left:innerWidth-60,top:120};
   vmOpen(r.left,r.top);
   return;
  }
  if(v==="save"){
   vmClose();
   const nm=prompt("اسم المنظر:","منظر "+(views().length+1));
   if(nm!=null)viewSave(nm);
  }
 });
 addEventListener("mousedown",e=>{
  const m=$("#vMenu");
  if(m&&!m.hidden&&!m.contains(e.target)
   &&!e.target.closest('[data-nav="views"]'))vmClose();
 },true);
 addEventListener("keydown",e=>{
  const m=$("#vMenu");
  if(e.key==="Escape"&&m&&!m.hidden){
   e.preventDefault(); e.stopPropagation(); vmClose();
  }
 },true);
 return true;
}
export {vmClose};
```

<a id="f-js-ui-optbar-js"></a>

---

## `js/ui/optbar.js`

```javascript
/* ═══ شريط الأدوات وشريط خياراتها ═══
   الخيارات تُقرأ عند إنشاء العنصر لا قبله ولا بعده،
   فتغييرها وسط سلسلة يسري على القطعة التالية وحدها.

   ملاحظة أداء وسلوك: HOOK.prompt() تُنادى من mousemove مع كل حركة
   مؤشّر أثناء أي أداة. فكان الشريط يُهدَم ويُبنى ستّين مرّةً في
   الثانية، ويسرق التركيز من حقلٍ تكتب فيه إن حرّكتَ الفأرة.
   فانقسم قسمين:
     buildOptbar  يهدم ويبني — عند تغيّر الأداة أو تغيّر الحقول
                  الظاهرة (بصمةٌ تُقارَن، لا تخمين).
     syncOptbar   يحدّث القيَم بمقارنةٍ فلا يهدم شيئاً، ويتخطّى
                  العنصر المركَّز عليه فلا يُكتَب فوق ما تكتبه. */
import {S} from "../core/state.js";
import * as R from "../tools/registry.js";
import {icon} from "./icons.js";
import {UIS} from "./store.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||15):"";

/* ترتيب الظهور · @ يعني أمراً لا أداة */
const BAR=[
 {cmd:"",       label:"تحديد", title:"Esc", ico:"select"},
 {cmd:"sel",    label:"تحديد بالمعرّف",title:"SE", ico:"selid"},
 {sp:1},
 {cmd:"wall",   label:"جدار",   title:"W",  ico:"wall"},
 {cmd:"rect",   label:"مستطيل", title:"R",  ico:"rect"},
 {cmd:"col",    label:"عمود",   title:"K",  ico:"col"},
 {cmd:"gridcols",label:"أعمدة المحاور",title:"GK", ico:"gridcols"},
 {sp:1},
 {cmd:"door",   label:"باب",    title:"D",  ico:"door"},
 {cmd:"win",    label:"شباك",   title:"N",  ico:"window"},
 {cmd:"opening",label:"فتحة",   title:"OP", ico:"opening"},
 {cmd:"niche",  label:"كوّة",    title:"",   ico:"niche"},
 {sp:1},
 {cmd:"stair",  label:"درج",    title:"ST", ico:"stair"},
 {cmd:"wc",     label:"كرسي",   title:"",   ico:"wc"},
 {cmd:"lav",    label:"مغسلة",  title:"",   ico:"lav"},
 {cmd:"shower", label:"دُش",     title:"",   ico:"shower"},
 {cmd:"sink",   label:"مجلى",   title:"",   ico:"sink"},
 {sp:1},
 {cmd:"area",   label:"منطقة",  title:"A",  ico:"area"},
 {cmd:"arearef",label:"حدّث",    title:"AR", ico:"arearef"},
 {sp:1},
 {cmd:"move",   label:"نقل",    title:"M",  ico:"move"},
 {cmd:"copy",   label:"نسخ",    title:"CP", ico:"copy"},
 {cmd:"rotate", label:"دوران",  title:"RO", ico:"rotate"},
 {cmd:"mirror", label:"مرآة",   title:"MR", ico:"mirror"},
 {cmd:"offset", label:"إزاحة",  title:"OF", ico:"offset"},
 {sp:1},
 {cmd:"break",  label:"قطع",    title:"BR", ico:"brk"},
 {cmd:"divide", label:"قسمة",   title:"DV", ico:"divide"},
 {cmd:"trim",   label:"قصّ",     title:"TR", ico:"trim"},
 {cmd:"extend", label:"تمديد",  title:"EX", ico:"extend"},
 {cmd:"stretch",label:"شدّ",     title:"STR",ico:"stretch"},
 {cmd:"weld",   label:"لحم",    title:"WL", ico:"weld"},
 {cmd:"match",  label:"مطابقة", title:"MA", ico:"match"},
 {sp:1},
 {cmd:"dim",    label:"بُعد",    title:"D1", ico:"dim"},
 {cmd:"chain",  label:"سلسلة",  title:"CH", ico:"chain"},
 {cmd:"text",   label:"نصّ",     title:"T",  ico:"text"},
 {cmd:"lead",   label:"قائد",   title:"LE", ico:"lead"},
 {cmd:"level",  label:"منسوب",  title:"LV", ico:"level"},
 {cmd:"axis",   label:"محور",   title:"AX", ico:"axis"},
 {sp:1},
 {cmd:"measure",label:"قياس",   title:"MI", ico:"measure"},
 {cmd:"@insp",  label:"افحص",   title:"F7", ico:"inspect"}
];

let TBTN=null, lastCur=null;

export function buildTools(){
 $("#tools").innerHTML=BAR.map(b=>b.sp?`<span class="sp"></span>`
  :`<button data-cmd="${esc(b.cmd)}" title="${esc(b.title||"")}">`
   +`${ic(b.ico)}<span>${esc(b.label)}</span></button>`).join("")
  +`<span class="gap"></span>`
  +`<button id="bLall" title="Ctrl+Shift+L">${ic("layers")}`
   +`<span>أظهر الكل</span></button>`
  +`<button id="bUndo" title="تراجع · Ctrl+Z" aria-label="تراجع"`
   +` class="ico">${ic("undo")||"↶"}</button>`
  +`<button id="bRedo" title="إعادة · Ctrl+Shift+Z" aria-label="إعادة"`
   +` class="ico">${ic("redo")||"↷"}</button>`
  +`<button id="bFit" title="ملاءمة العرض">${ic("fit")}`
   +`<span>ملاءمة</span></button>`
  +`<button id="bHelp" title="المساعدة · F1" aria-label="المساعدة"`
   +` class="ico">${ic("help")||"؟"}</button>`;
 TBTN=null; lastCur=null;
}
/* ═══ إبراز الأداة النشطة — لا يعمل إلّا إن تغيّرت ═══ */
export function syncTools(){
 if(!TBTN)TBTN=[...document.querySelectorAll("#tools [data-cmd]")];
 const cur=R.active()?R.T.def.id:"";
 if(cur===lastCur)return;
 lastCur=cur;
 TBTN.forEach(b=>b.classList.toggle("on",b.dataset.cmd===cur));
}
/* ═══ شريط الخيارات ═══ */
const visFields=d=>{
 const o=R.OPT[d.id]||{};
 return (d.opts||[]).filter(f=>!f.when||f.when(o));
};
/* البصمة: الأداة + مفاتيح الحقول الظاهرة. تغيّرها وحده يوجب البناء */
const sigOf=d=>d?(d.id+"|"+visFields(d).map(f=>f.k).join(",")):"";
let barSig=null;

function ctlOf(f,v){
 const tag=`data-ok="${esc(f.k)}"`;
 if(f.type==="sel")
  return `<span class="of"><span>${esc(f.label)}</span>`
   +`<select ${tag}>`
   +f.items.map(([iv,it])=>`<option value="${esc(iv)}"`
    +`${String(iv)===String(v)?" selected":""}>${esc(it)}</option>`)
    .join("")+`</select></span>`;
 if(f.type==="chk")
  return `<label class="chk"><input type="checkbox" ${tag}`
   +`${(v===1||v===true||v==="1")?" checked":""}> `
   +`${esc(f.label)}</label>`;
 if(f.type==="num")
  return `<span class="of"><span>${esc(f.label)}</span>`
   +`<input class="num" type="number" ${tag} `
   +`value="${esc(v)}" step="0.1"></span>`;
 return `<span class="of"><span>${esc(f.label)}</span>`
  +`<input class="num" type="text" ${tag} value="${esc(v)}"></span>`;
}
export function buildOptbar(){
 const box=$("#optbar");
 if(!box)return;
 const d=R.T.def;
 barSig=sigOf(d);
 if(!d){
  box.innerHTML=`<span class="tl">تحديد</span>`
   +`<span class="hint">انقر عنصراً · اسحب إطاراً على الفراغ · `
   +`Shift+نقر يضيف · اسحب المقابض · Ctrl+A يحدّد المرئيّ</span>`;
  return;
 }
 const o=R.OPT[d.id]||{};
 box.innerHTML=`<span class="tl">${esc(d.label)}</span>`
  +visFields(d).map(f=>ctlOf(f,o[f.k])).join("")
  +(d.hint?`<span class="hint">${esc(d.hint)}</span>`:"");
}
/* تحديث القيَم بلا هدم · يُنادى مع كل حركة مؤشّر فلا يجوز أن يبني */
export function syncOptbar(){
 const d=R.T.def;
 if(sigOf(d)!==barSig){buildOptbar(); return}
 if(!d)return;
 const box=$("#optbar");
 if(!box)return;
 const o=R.OPT[d.id]||{};
 const A=document.activeElement;
 box.querySelectorAll("[data-ok]").forEach(el=>{
  if(el===A)return;              /* لا نكتب فوق ما يكتبه المستخدم */
  const v=o[el.dataset.ok];
  if(el.type==="checkbox"){
   const on=(v===1||v===true||v==="1");
   if(el.checked!==on)el.checked=on;
   return;
  }
  const s=String(v==null?"":v);
  if(el.value!==s)el.value=s;
 });
}
$("#optbar").addEventListener("change",e=>{
 const k=e.target.dataset.ok;
 if(!k||!R.T.def)return;
 const v=(e.target.type==="checkbox")?(e.target.checked?1:0)
  :e.target.value;
 R.setOpt(R.T.def.id,k,v);
 syncOptbar();      /* يعيد البناء وحده إن ظهر حقلٌ شرطيّ أو اختفى */
 HOOK.prompt();
 HOOK.defs();       /* لوحة الافتراضات تُظهر القيمة نفسها */
});
/* منع تسرّب المفاتيح من حقول الخيارات إلى الاختصارات */
$("#optbar").addEventListener("keydown",e=>{
 if(e.key==="Escape"){e.target.blur();return}
 e.stopPropagation();
});
```

<a id="f-js-ui-overlay-js"></a>

---

## `js/ui/overlay.js`

```javascript
/* ═══ تركيبات فوق اللوحة ═══
   بوصلة الشمال وبطاقة المنظور. كلتاهما داخل #stage المعزول بـ
   dir=ltr، فلا يتسرّب اتجاه الواجهة إلى ما يُركَّب على الرسم —
   وهذا ما أعدّته و٠ لهذه اللحظة.

   البوصلة مؤشّرٌ على S.meta.north وناقلٌ إليه: تعرض الزاوية
   وتحرّرها. والسهم المُصدَّر يخرج من sheet.js لأنه رمزٌ على اللوحة
   لا شارةُ شاشة. */
import {S,edit} from "../core/state.js";
import {deg,clamp,scl} from "../core/units.js";
import {UIS,saveUI} from "./store.js";
import {V,draw} from "./canvas.js";
import {scene} from "../core/render.js";
import {anyHidden,hiddenCount} from "../core/layers.js";
import {HOOK} from "./bus.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");

/* ═══ البوصلة ═══ */
export function buildCompass(){
 const c=$("#compass");
 if(!c)return 0;
 c.innerHTML=`
<button type="button" id="cmpBtn" title="زاوية الشمال — انقر للتعديل"
 aria-label="زاوية الشمال">
 <svg viewBox="0 0 48 48" width="44" height="44" aria-hidden="true">
  <circle cx="24" cy="24" r="20.5" class="cmpR"/>
  <g id="cmpG">
   <path d="M24 5 L30 26 L24 22 L18 26 Z" class="cmpN"/>
   <path d="M24 22 L24 43" class="cmpT"/>
  </g>
  <text x="24" y="47" class="cmpL" text-anchor="middle">ش</text>
 </svg>
</button>
<div id="cmpPop" hidden>
 <div class="pmH">زاوية الشمال</div>
 <div class="cmpRow">
  <input id="cmpIn" type="number" class="num" min="0" max="359.9"
   step="0.5" aria-label="زاوية الشمال بالدرجات"> <span>°</span>
 </div>
 <div class="cmpPre">
  <button type="button" data-cmp="0">0</button>
  <button type="button" data-cmp="90">90</button>
  <button type="button" data-cmp="180">180</button>
  <button type="button" data-cmp="270">270</button>
 </div>
 <p class="hint">تُقاس عكس عقارب الساعة. تُصدَّر سهماً في زاوية
  الإطار حين تكون الورقة ظاهرة.</p>
</div>`;
 return 1;
}
export function syncCompass(){
 const g=$("#cmpG");
 if(g)g.setAttribute("transform",
  `rotate(${-(+S.meta.north||0)} 24 24)`);
 const b=$("#cmpBtn");
 if(b)b.title=`زاوية الشمال ${(+S.meta.north||0).toFixed(1)}° — `
  +`انقر للتعديل`;
 const i=$("#cmpIn");
 if(i&&document.activeElement!==i)
  i.value=String(+S.meta.north||0);
 const c=$("#compass");
 if(c)c.hidden=!!UIS.clean||!UIS.compass;
}
function setNorth(v){
 const a=deg(parseFloat(v)||0);
 edit(()=>{S.meta.north=a});
 syncCompass();
 HOOK.refresh(0);
 HOOK.report("in",`زاوية الشمال ${a.toFixed(1)}°`);
}
/* ═══ بطاقة المنظور ═══
   لا «الطبقة الحالية» فيها: لا مفهومَ لها اليوم — الطبقة تُشتَقّ من
   نوع الكيان، وم٠ يجعلها جدولاً حيّاً فتدخل حينها. */
export function buildVp(){
 const v=$("#vpLabel");
 if(!v)return 0;
 v.innerHTML=`<span class="vpI" id="vpMode" title="فضاء النموذج — `
  +`فضاء الورقة يدخل مع التخطيطات">[النموذج]</span>`
  +`<button type="button" class="vpI" data-act="stScale" `
  +`title="المقياس — انقر لإعداد المشروع"><span id="vpSc" `
  +`class="mono num">[1:100]</span></button>`
  +`<button type="button" class="vpI" data-act="stWarn" id="vpW" `
  +`hidden></button>`;
 return 1;
}
export function syncVp(){
 const box=$("#vpLabel");
 if(!box)return;
 box.hidden=!!UIS.clean||!UIS.vpLabel;
 const sc=$("#vpSc");
 if(sc)sc.textContent=`[${scl(S.meta.scale)}]`;
 const w=$("#vpW");
 if(!w)return;
 let n=0, P=[];
 try{
  const s=scene();
  if(s.bad)P.push(`${s.bad} فتحة معطوبة`);
  if(s.stale)P.push(`${s.stale} منطقة قديمة`);
  if(s.loose)P.push(`${s.loose} بُعداً معلَّقاً`);
  n=s.bad+s.stale+s.loose;
 }catch(e){}
 const hid=anyHidden()?hiddenCount():0;
 if(hid)P.push(`${hid} كياناً مخفيّاً`);
 const tot=n+hid;
 w.hidden=!tot;
 if(!tot)return;
 w.textContent=`[${tot} تنبيه]`;
 w.title=P.join(" · ")+" — انقر للفاحص";
 w.classList.toggle("bad",n>0);
}
export function wireOverlay(){
 buildCompass();
 buildVp();
 const b=$("#cmpBtn"), p=$("#cmpPop");
 if(b&&p){
  b.onclick=e=>{
   e.stopPropagation();
   p.hidden=!p.hidden;
   if(!p.hidden){syncCompass(); const i=$("#cmpIn"); if(i)i.focus()}
  };
  p.addEventListener("click",e=>{
   const q=e.target.closest("[data-cmp]");
   if(q){setNorth(q.dataset.cmp); p.hidden=true}
  });
  p.addEventListener("change",e=>{
   if(e.target.id==="cmpIn")setNorth(e.target.value);
  });
  p.addEventListener("keydown",e=>{
   if(e.key==="Escape"){e.preventDefault(); p.hidden=true;
    b.focus(); return}
   if(e.key==="Enter"&&e.target.id==="cmpIn"){
    e.preventDefault(); setNorth(e.target.value); p.hidden=true;
    return;
   }
   e.stopPropagation();
  });
  addEventListener("mousedown",e=>{
   if(p.hidden)return;
   if(!p.contains(e.target)&&e.target!==b&&!b.contains(e.target))
    p.hidden=true;
  },true);
 }
 syncCompass(); syncVp();
 return true;
}
export const syncOverlay=()=>{syncCompass(); syncVp()};
```

<a id="f-js-ui-palette-js"></a>

---

## `js/ui/palette.js`

````javascript
/* ═══ لوحة الأوامر ═══
   فهرس واحد للأدوات والأفعال والمقاسات والقوالب وسجلّ الأوامر. */
import * as R from "../tools/registry.js";
import {allItems} from "./ribbon/schema.js";
import {runSpec} from "./ribbon/wire.js";
import {HOOK} from "./bus.js";
import {SIZES,TPL,applySize} from "../tools/presets.js";
import {parsePlan,planReady} from "../ai/plan.js";
import {trial,commit,rollback} from "../ai/run.js";
import {V} from "./canvas.js";
import {JR,jrText,jrPlan,jrClear,jrCount,jrTainted,jrMute} from "../core/journal.js";
import {snapTake,snapList,snapRestore,snapsLoad} from "../io/snaps.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s).replace(/&/g,"&amp;")
 .replace(/</g,"&lt;").replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const KB={ض:"q",ص:"w",ث:"e",ق:"r",ف:"t",غ:"y",ع:"u",ه:"i",
 خ:"o",ح:"p",ش:"a",س:"s",ي:"d",ب:"f",ل:"g",ا:"h",ت:"j",
 ن:"k",م:"l",ئ:"z",ء:"x",ؤ:"c",ر:"v",ى:"n",ة:"m",
 ج:"[",د:"]",ك:";",ط:"'",و:",",ز:".",ظ:"/"};
const toLat=s=>[...String(s||"")].map(c=>KB[c]===undefined?c:KB[c]).join("");
const norm=s=>String(s==null?"":s).toLowerCase()
 .replace(/[\u064B-\u0652\u0670\u0640]/g,"")
 .replace(/[أإآٱ]/g,"ا").replace(/ى/g,"ي").replace(/ة/g,"ه")
 .replace(/\s+/g," ").trim();
let BODY=null,NT=-1,LIST=[],IDX=0,OPEN=0;
const box=()=>$("#palette");
function score(hay,q){
 if(!q)return 0;
 if(hay.startsWith(q))return 1000-hay.length;
 const i=hay.indexOf(q);
 if(i>=0)return 700-i-hay.length*.1;
 let s=0,j=0,run=0;
 for(const ch of q){
  const k=hay.indexOf(ch,j); if(k<0)return 0;
  run=k===j?run+1:0; s+=10+run*6-Math.min(k-j,20)*.5; j=k+1;
 }
 return s;
}
function build(){
 const n=R.toolList().length;
 if(BODY&&n===NT)return BODY;
 NT=n; const out=[];
 R.toolList().filter(d=>d&&d.id).forEach(d=>{
  const al=Object.keys(R.TOOLS).filter(k=>R.TOOLS[k]===d&&k!==d.id);
  out.push({kind:"cmd",key:d.id,label:d.label,sub:[d.id,...al].join(" · "),
   hay:norm([d.id,d.label,...al,d.hint||""].join(" ")),
   destruct:!!d.destruct});
 });
 const seen=new Set(["delSel"]);
 allItems().filter(i=>(i.act||i.cmd)&&!seen.has(i.act||i.cmd)).forEach(i=>{
  const key=i.act||i.cmd; seen.add(key);
  out.push({kind:i.act?"act":"cmd",key,label:i.n||key,sub:"أمر",
   hay:norm(`${i.n||""} ${key}`)});
 });
 SIZES.forEach(p=>out.push({kind:"size",key:p.id,ref:p,label:p.label,
  sub:`${p.sub} · ${p.tool}`,hay:norm(`${p.label} ${p.cat} ${p.sub} ${p.tool}`)}));
 TPL.forEach(t=>out.push({kind:"tpl",key:t.id,ref:t,label:t.label,sub:t.sub,
  hay:norm(`${t.label} ${t.sub} قالب`)}));
 const fn=[
  {key:"jr.copy",label:"سجلّ الأوامر: انسخه",sub:"نصّ أسطرٍ بنحو سطر الإدخال",fn:jrCopy},
  {key:"jr.replay",label:"سجلّ الأوامر: أعِد تشغيله",sub:"يُنفَّذ على الحالة الجارية",fn:jrReplay},
  {key:"jr.clear",label:"سجلّ الأوامر: امسحه",sub:"يبدأ التسجيل من الآن",fn:()=>{
   jrClear();HOOK.report("in","مُسح سجلّ الأوامر");
  }},
  {key:"snap.now",label:"لقطة الآن",sub:"نسخة كاملة تعبر إغلاق الصفحة",fn:snapNow},
  {key:"snap.list",label:"استعادة لقطة…",sub:"آخر ١٢ لقطة · الاستعادة خطوة تراجع",fn:paletteSnaps}
 ];
 fn.forEach(f=>out.push({kind:"fn",key:f.key,ref:f,label:f.label,sub:f.sub,hay:norm(f.label+" "+f.sub)}));
 BODY=out; return BODY;
}
function rank(raw){
 const q=norm(raw),qa=norm(toLat(raw));
 return build().map(it=>({it,s:Math.max(score(it.hay,q),qa!==q?score(it.hay,qa):0)}))
  .filter(x=>x.s>0).sort((a,b)=>b.s-a.s).slice(0,12);
}
function render(){
 const B=box(); if(!B)return;
 B.querySelector(".pl").innerHTML=LIST.length
  ?LIST.map((x,i)=>`<div class="it${i===IDX?" sel":""}" data-i="${i}">
    <span class="lb">${esc(x.it.label)}${x.it.destruct?' <b class="dg">هادم</b>':""}</span>
    <span class="sb">${esc(x.it.sub)}</span></div>`).join("")
  :`<div class="nm">لا نتيجة</div>`;
}
export function paletteClose(){
 const B=box(); if(!B||!OPEN)return;
 OPEN=0; B.hidden=true; $("#clIn")?.focus();
}
export function paletteOpen(){
 const B=box(); if(!B)return;
 OPEN=1; B.hidden=false; const inp=B.querySelector("input");
 inp.value=""; IDX=0; LIST=build().filter(x=>["wall","rect","door","win","area","dim",
  "move","copy","offset","measure"].includes(x.key)).map(it=>({it,s:1}));
 render(); inp.focus();
}
export const paletteToggle=()=>OPEN?paletteClose():paletteOpen();
export const paletteIsOpen=()=>!!OPEN;
function runTpl(t){
 const at=[Math.round(V.cx/100)/10,Math.round(V.cy/100)/10];
 const p=parsePlan("```plan\n"+t.lines(at).join("\n")+"\n```",1,400);
 if(!planReady(p)){HOOK.report("er",`«${t.label}» مرفوض: `+(p.errs[0]||"سطر غير مفهوم"));return}
 const r=trial(p.lines,{stopOnError:1});
 if(r.errs){rollback(r);HOOK.report("er",`تعذّر «${t.label}» — أُرجع كل شيء`);return}
 const n=commit(r);HOOK.report("ok",`${t.label} · ${n} كياناً · Ctrl+Z يتراجع عنها`);
 HOOK.refresh(1);
}
function jrCopy(){
 const n=jrCount();
 if(!n){HOOK.report("in","السجلّ فارغ");return}
 if(navigator.clipboard?.writeText)
  navigator.clipboard.writeText(jrText()).then(
   ()=>HOOK.report("ok",`نُسخ ${n} سطراً${jrTainted()?" · مشوب":""}`),
   ()=>HOOK.report("er","تعذّر النسخ — المتصفّح منع الحافظة"));
 else HOOK.report("er","الحافظة غير متاحة");
}
function jrReplay(){
 const n=jrCount();
 if(!n){HOOK.report("in","السجلّ فارغ");return}
 if(jrTainted()&&!confirm(`السجلّ مشوب بـ ${jrTainted()} عملية لا يُعبّر عنها بسطر. الإعادة ستختلف. متابعة؟`))return;
 const p=parsePlan(jrPlan(),1,JR.max);
 if(!planReady(p)){HOOK.report("er",`السجلّ غير قابل للإعادة: `+(p.errs[0]||"سطر غير مفهوم"));return}
 jrMute(1); let r=null; try{r=trial(p.lines,{stopOnError:1})}finally{jrMute(0)}
 if(!r||r.errs){if(r)rollback(r);HOOK.report("er","تعذّرت الإعادة — أُرجع كل شيء");return}
 const made=commit(r);HOOK.report("ok",`أُعيد ${r.ran} سطراً · ${made} كياناً`);
 HOOK.refresh(1);
}
async function snapNow(){
 const r=await snapTake("يدوية");
 HOOK.report(r.ok?"ok":"wr",r.ok?`أُخذت لقطة · ${snapList().length} من 12 محفوظة`
  :(r.err==="فارغ"?"لا شيء يُلتقط — المشروع فارغ":"تعذّرت اللقطة — IndexedDB غير متاح"));
}
export function paletteSnaps(){
 paletteOpen(); const L=snapList();
 if(!L.length){HOOK.report("in","لا لقطات بعد — «لقطة الآن» تأخذ أولها");paletteClose();return}
 LIST=L.map(m=>({s:1,it:{kind:"snapr",key:m.id,ref:m,
  label:`${new Date(m.t).toLocaleString("ar")} · ${m.why}`,
  sub:`${m.w} جدار · ${m.o} فتحة · ${m.a} منطقة`}})); IDX=0; render();
}
export function wirePalette(){
 const B=box(); if(!B)return;
 B.innerHTML=`<div class="pw" role="dialog" aria-modal="true" aria-label="لوحة الأوامر">
  <input type="text" spellcheck="false" autocomplete="off" placeholder="اكتب اسم أداة أو أمر…">
  <div class="pl"></div><div class="ft">↑↓ تنقّل · Enter ينفّذ · Esc يغلق</div></div>`;
 B.hidden=true; const inp=B.querySelector("input");
 inp.addEventListener("input",()=>{LIST=rank(inp.value);IDX=0;render()});
 inp.addEventListener("keydown",e=>{
  e.stopPropagation();
  if(e.key==="Escape"){e.preventDefault();paletteClose();return}
  if(e.key==="ArrowDown"||e.key==="ArrowUp"){e.preventDefault();if(LIST.length)
   {IDX=(IDX+(e.key==="ArrowDown"?1:-1)+LIST.length)%LIST.length;render()}return}
  if(e.key==="Enter"){e.preventDefault();run(LIST[IDX])}
 });
 B.querySelector(".pl").addEventListener("mousedown",e=>{
  const it=e.target.closest("[data-i]");if(it){e.preventDefault();run(LIST[+it.dataset.i])}
 });
 B.addEventListener("mousedown",e=>{if(e.target===B)paletteClose()});
 snapsLoad();
}
function run(x){
 if(!x)return; paletteClose();
 if(x.it.kind==="cmd"){R.histAdd(x.it.key);R.begin(x.it.key);HOOK.prompt();return}
 if(x.it.kind==="act"){runSpec({act:x.it.key});return}
 if(x.it.kind==="size"){applySize(x.it.ref);HOOK.report("in",`${x.it.label} — انقر الموضع`);HOOK.prompt();return}
 if(x.it.kind==="tpl"){runTpl(x.it.ref);return}
 if(x.it.kind==="fn"){x.it.ref.fn();return}
 if(x.it.kind==="snapr"){
  const m=x.it.ref;
  if(!confirm(`استعادة لقطة ${new Date(m.t).toLocaleString("ar")}؟\nعملك الحالي يُستبدل، و Ctrl+Z يعيده.`))return;
  snapRestore(m.id).then(r=>{if(r.ok){HOOK.report("ok",`استُعيدت اللقطة · ${r.w} جدار`);HOOK.refresh(1)}
   else HOOK.report("er",r.err)});
 }
}
````

<a id="f-js-ui-panels-js"></a>

---

## `js/ui/panels.js`

```javascript
/* ═══ سجلّ اللوحات ═══
   قبله: refresh() تعيد بناء ثماني لوحات مع كل تعديل حقل، ومنها
   لوحاتٌ في أقسامٍ مغلقة لا يراها أحد — جدول الفتحات والمساحات
   والورقة والمرجع تُبنى كلّها لتُخفى.
   بعده: كل لوحة تُعلَن مرّة، ولا تُرسَم إلّا إن كانت مرئيّة.
   والمغلقة تُوسَم «متّسخة» فتُرسَم لحظةَ فتحها.

   وهذا نصف ما يحتاجه الإرساء٢: بقيّته قشرةٌ حول الأجسام
   نفسها، لا تعديلٌ فيها. */
import {secSet,secOpen} from "./store.js";

const REG=new Map();

export function reg(id,sel,render,label){
 REG.set(id,{id,sel,render,label:label||id,dirty:1,n:0});
 return id;
}
export const panelIds=()=>[...REG.keys()];
export const panelSel=id=>{
 const p=REG.get(id);
 return p?p.sel:null;
};
export const markDirty=id=>{
 const p=REG.get(id);
 if(p)p.dirty=1;
};
export const markAllDirty=()=>{REG.forEach(p=>{p.dirty=1})};

/* ═══ الرؤية ═══
   اللوحة قد تسكن عموداً أو نافذةً عائمة أو مرآباً مغلقاً، فلا
   يكفي فحص details.sec: نصعد الأسلاف حتى الجذر.
   نتخطّى offsetParent بقصد — يفرض إعادة تخطيطٍ ويكذب على
   العناصر الثابتة الموضع. */
const shown=el=>{
 if(!el||!el.isConnected)return false;
 for(let n=el;n&&n.nodeType===1;n=n.parentElement){
  if(n.hidden)return false;
  if(n.tagName==="DETAILS"&&n.classList.contains("sec")&&!n.open)
   return false;
 }
 return true;
};
export function visible(id){
 const p=REG.get(id);
 if(!p)return false;
 return shown(document.querySelector(p.sel));
}
/* الرسم يعزل خطأ لوحةٍ عن بقيّتها: عطبٌ في جدولٍ لا يُسقط الخصائص */
export function renderPanel(id,force){
 const p=REG.get(id);
 if(!p)return false;
 const el=document.querySelector(p.sel);
 if(!el)return false;
 if(!force&&!shown(el)){p.dirty=1; return false}
 try{p.render(el); p.dirty=0; p.n++; return true}
 catch(e){
  p.dirty=1;
  el.innerHTML=`<p class="hint warn">تعذّر بناء اللوحة: `
   +`${String(e.message||e)}</p>`;
  return false;
 }
}
/* dirtyOnly=1 يرسم المتّسخ المرئيّ وحده */
export function renderVisible(dirtyOnly){
 let n=0;
 REG.forEach(p=>{
  if(dirtyOnly&&!p.dirty)return;
  if(renderPanel(p.id))n++;
 });
 return n;
}
/* ═══ حالة الأقسام ═══
   toggle لا يصعد، فنُنصت في طور الالتقاط على document: اللوحة قد
   تسكن عموداً ثانياً أو نافذةً عائمة، ومستمعٌ مربوطٌ على #side
   يتوقّف صامتاً عند أول نقل. */
let WIRED=false;
export function wirePanels(){
 const D=[...document.querySelectorAll("details.sec[data-sec]")];
 D.forEach(d=>{d.open=secOpen(d.dataset.sec,d.open)});
 if(WIRED)return D.length;
 WIRED=true;
 document.addEventListener("toggle",e=>{
  const d=e.target;
  if(!d.dataset||!d.dataset.sec)return;
  if(!d.classList||!d.classList.contains("sec"))return;
  secSet(d.dataset.sec,d.open);
  if(d.open)renderVisible(1);
 },true);
 return D.length;
}
export const stats=()=>[...REG.values()]
 .map(p=>({id:p.id,dirty:p.dirty,n:p.n,vis:visible(p.id)}));
```

<a id="f-js-ui-props-js"></a>

---

## `js/ui/props.js`

```javascript
/* ═══ اللوحة الجانبية: المشروع · الطبقات · الخصائص · الجدول ═══
   في التحديد المتعدّد: الحقل المختلف يظهر فارغاً بعلامة «متعدّد»
   ولا يُكتب من تلقائه. */
import {S,LAYERS,edit,editFailed,touch,undo,redo,canUndo,canRedo,
        newState,ensureShape,setEditError,autosave} from "../core/state.js";
import {M,m2,m3,mm,mnum,clamp,isLen,sqm,deg,
 rng2,dm2,pt2,scl,ltr} from "../core/units.js";
import {ALIGN,wallById,wallLen,dir,looseEnds,isLow,
        lowH} from "../core/walls.js";
import {openById,okName,OKINDS,OK,openState,allowed,saySpans,span,
        panOf,depOf,openSchedule} from "../core/opens.js";
import {areaById,netArea,netPerim,isStale,restamp,rebake,
        schedule,FILLS} from "../core/areas.js";
import {dimById,chainById,annoById,dimValue,fmtLen,DK,AK,
        dimLoose,chainVals,chainSum,levelStr,
        isOverridden,parseVals} from "../core/dims.js";
import {colById,CK,CT,colArea,colOnWall} from "../core/cols.js";
import {fixById,FK,FKINDS,fixName,fixOnWall,
        snapToWall} from "../core/fixt.js";
import {stById,stCheck,stGeom} from "../core/stairs.js";
import {scene,regionLoops} from "../core/render.js";
/* من layers.js — يُضاف: layOf · hasLay · LT · LWS · plotAll
   · noPlotCount · وحالاتُ الطبقات · resetLays */
import {LAYS,layOf,hasLay,LNAME,AUX,LT,LWS,vis,locked,plots,
        setLay,pickable,toggleOff,toggleLock,isolate,showAll,
        unlockAll,plotAll,resetLays,layCounts,hiddenCount,
        noPlotCount,anyHidden,anyLocked,layStates,stateSave,
        stateApply,stateDel,layOfEnt} from "../core/layers.js";
/* ومن batch.js — يُضاف clearAreaNames */
import {FLD,fldOf,fldName,fieldVal,groupSel,groupOrder,readField,
        applyField,applyOne,sayApply,groupForced,summary,
        renumberCols,clearDimTxt,clearAreaNames,
        nameAreasSeq} from "../core/batch.js";
import {NAME,delSay} from "../core/ents.js";
import {selList,delSel,setSel,fit,draw,pruneSel,UI} from "./canvas.js";
import {syncSheet,renderRef} from "./inspector.js";
import {syncStatus} from "./statusbar.js";
import {syncOverlay} from "./overlay.js";
import {HOOK} from "./bus.js";
import {reg,renderPanel,renderVisible,markAllDirty,
        wirePanels} from "./panels.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const H_sel=()=>selList();

/* ═══ السجل ═══ */
const LOG=$("#log");
const CLS={ok:"ok",er:"er",wr:"wr",in:"in"};
export function rep(cls,msg){
 if(!LOG)return;
 const d=document.createElement("div");
 d.className="ln "+(CLS[cls]||"in");
 d.textContent=String(msg==null?"":msg);
 LOG.appendChild(d);
 while(LOG.childElementCount>400)LOG.removeChild(LOG.firstChild);
 LOG.scrollTop=LOG.scrollHeight;
}
setEditError(m=>rep("er",m));
export const eInfo=m=>{const e=$("#stInfo"); if(e)e.textContent=m||""};
const eSel=m=>{const e=$("#stSel"); if(e)e.textContent=m||""};

const OPT=(arr,cur)=>arr.map(([v,t])=>
 `<option value="${esc(v)}"${String(v)===String(cur)?" selected":""}>`
 +`${esc(t)}</option>`).join("");

/* ═══ بناء اللوحة ═══ */
export function buildSide(){
 $("#side").innerHTML=`
<details class="sec" data-sec="proj" open><summary>المشروع</summary>
 <div class="row"><label>اسم اللوحة</label>
  <input id="mName" type="text"></div>
 <div class="row2">
  <div class="f"><label>المقياس 1:</label>
   <input id="mScale" type="number" min="1" max="5000" step="1"></div>
  <div class="f"><label>النص مم</label>
   <input id="mTxt" type="number" min="0.5" max="20" step="0.1"></div>
 </div>
 <div class="row2">
  <div class="f"><label>خارجي م</label><input id="mExt" type="text"></div>
  <div class="f"><label>داخلي م</label><input id="mInt" type="text"></div>
 </div>
 <div class="row2">
  <div class="f"><label>ارتفاع الدور م</label>
   <input id="mH" type="text"></div>
  <div class="f"><label>خطوة الالتقاط م</label>
   <input id="mSnap" type="text"></div>
 </div>
 <div class="row2">
  <div class="f"><label>تعبئة الجدران</label>
   <select id="mFill">${OPT([["none","بلا"],["hatch","هاشور"],
    ["solid","مصمّت"]],"none")}</select></div>
  <div class="f"><label>عشرية الأبعاد</label>
   <select id="mDec">${OPT([["0","0"],["1","1"],["2","2"],
    ["3","3"]],"2")}</select></div>
 </div>
 <div class="row"><label>علامة البُعد</label>
  <select id="mTick">${OPT([["slash","شرطة"],["arrow","سهم"]],
   "slash")}</select></div>
 <label class="chk" style="margin:6px 0">
  <input type="checkbox" id="oJoins">
  دمج الأركان ووصلات T عند العرض</label>
 <label class="chk" style="margin:4px 0">
  <input type="checkbox" id="oSolo">
  أظهر الأعمدة مستقلّة (بلا دمج)</label>
 <p class="hint">الدمج وطرح الفتحات عرضٌ لا تعديل: البيانات تبقى كما
  رسمتها. والحلقات تشمل الأعمدة دائماً.</p>
</details>

<details class="sec" data-sec="lays" open><summary>الطبقات</summary>
 <div id="lays"></div>
 <div class="btnrow">
  <button id="lAll">أظهر الكل</button>
  <button id="lUnlock">افتح المقفل</button>
 </div>
 <div class="btnrow">
  <button id="lPlotAll">أعِد الطبع</button>
  <button id="lReset" class="del">أعِد المصنع</button>
 </div>
 <p class="hint">المخفيّ لا يُرسَم ولا يُحدَّد ولا يُصدَّر.
  والمقفل يُرى ولا يُلمَس. الهندسة لا تُخفى: خبز المناطق يقرأ
  الجدران كلّها.</p>
</details>

<details class="sec" data-sec="props" open><summary>الخصائص</summary>
 <div id="props"><p class="hint">لا تحديد</p></div>
</details>

<details class="sec" data-sec="sched"><summary>جدول المساحات</summary>
 <div id="sched"></div>
 <div class="btnrow"><button id="bRefAll">حدّث القديمة</button>
  <button id="bAsCsv">CSV</button></div>
</details>

<details class="sec" data-sec="osched"><summary>جدول الفتحات</summary>
 <div id="osched"></div>
 <div class="btnrow"><button id="bOsCsv">CSV</button></div>
</details>

<details class="sec" data-sec="axes"><summary>المحاور</summary>
 <div id="axInfo" class="hint"></div>
 <div class="btnrow"><button id="bAxClr" class="del">
  امسح المحاور</button></div>
</details>

<details class="sec" data-sec="ref"><summary>المرجع المستورد</summary>
 <div class="btnrow">
  <button id="rImp" class="pri">استورد DXF</button>
  <button id="rClr" class="del">أزِل</button>
 </div>
 <div class="row2">
  <div class="f"><label>الوحدة عند الاستيراد</label>
   <select id="rUnit">
    <option value="">من الملفّ</option>
    <option value="1">مليمتر</option>
    <option value="10">سنتيمتر</option>
    <option value="1000">متر</option>
    <option value="25.4">بوصة</option>
    <option value="304.8">قدم</option>
   </select></div>
  <div class="f" style="display:flex;align-items:flex-end">
   <button id="rRst" style="flex:1">صفّر التحويل</button></div>
 </div>
 <div class="btnrow">
  <button data-run="refalign">محاذاة</button>
  <button data-run="refcal">معايرة</button>
  <button data-run="refmove">نقل</button>
 </div>
 <div id="rInfo" class="hint"></div>
 <div id="rLays"></div>
 <p class="hint">المرجع جامد: تراه وتقيس عليه وتلتقط نقاطه، ولا
  يُحدَّد ولا يدخل الاتحاد ولا الحلقات ولا المساحات. ارسم جدرانك
  فوقه بيدك — لا يُستنتَج منه جدار.</p>
</details>

<details class="sec" data-sec="sheet"><summary>الورقة وبلوك العنوان</summary>
 <label class="chk" style="margin:6px 0">
  <input type="checkbox" id="shOn"> أظهر الورقة</label>
 <div class="row2">
  <div class="f"><label>المقاس</label>
   <select id="shSize"><option>A4</option><option>A3</option>
    <option>A2</option><option>A1</option><option>A0</option>
   </select></div>
  <div class="f"><label>الاتجاه</label>
   <select id="shOr"><option value="l">أفقي</option>
    <option value="p">رأسي</option></select></div>
 </div>
 <div class="row2">
  <div class="f"><label>الهامش مم</label>
   <input id="shMar" type="number" min="0" max="60" step="1"></div>
  <div class="f" style="display:flex;align-items:flex-end">
   <label class="chk"><input type="checkbox" id="shTb">
    بلوك العنوان</label></div>
 </div>
 <div class="btnrow"><button id="shCenter">تمركز على الرسم</button></div>
 <div id="shInfo" class="hint"></div>
 <div class="row"><label>المشروع</label>
  <input id="tProj" type="text"></div>
 <div class="row2">
  <div class="f"><label>المالك</label><input id="tOwner" type="text">
  </div>
  <div class="f"><label>الموقع</label><input id="tLoc" type="text">
  </div>
 </div>
 <div class="row2">
  <div class="f"><label>اللوحة</label><input id="tSheet" type="text">
  </div>
  <div class="f"><label>المراجعة</label><input id="tRev" type="text">
  </div>
 </div>
 <div class="row"><label>الرسم بواسطة</label>
  <input id="tBy" type="text"></div>
</details>

<details class="sec" data-sec="insp" open><summary>الفاحص</summary>
 <div class="btnrow"><button id="bInsp" class="pri">افحص (F7)</button>
 </div>
 <div id="insp"></div>
</details>

<details class="sec" data-sec="export"><summary>التصدير والملفّ</summary>
 <div class="row2">
  <div class="f"><label>دقّة PNG</label>
   <input id="xDpi" type="number" min="72" max="1200" step="50"
    value="300"></div>
  <div class="f" style="display:flex;align-items:flex-end">
   <label class="chk"><input type="checkbox" id="xDark">
    خلفية داكنة</label></div>
 </div>
 <div class="row2">
  <div class="f" style="display:flex;align-items:flex-end">
   <label class="chk"><input type="checkbox" id="xWarn">
    ألوان التنبيه في المخرَج</label></div>
  <div class="f"></div>
 </div>
 <p class="hint" id="xInfo"></p>
 <div class="btnrow">
  <button id="xDxf">DXF</button>
  <button id="xSvg">SVG</button>
 </div>
 <div class="btnrow">
  <button id="xPng">PNG</button>
  <button id="xPdf">PDF</button>
 </div>
 <div class="btnrow">
  <button id="xSave" class="pri">احفظ المشروع</button>
  <button id="xOpen">افتح</button>
 </div>
 <p class="hint">النطاق: الورقة إن كانت مُشغّلة، وإلا الرسم بهامش.
  الأربعة يقرأون أوّليات المشهد نفسها.</p>
</details>

<details class="sec" data-sec="defs"><summary>الإعدادات الافتراضية</summary>
 <div id="defs"></div>
 <div class="btnrow"><button id="dRst" class="del">
  أعِد المصنع</button></div>
 <p class="hint">هذه هي خيارات شريط الأدوات نفسها مجموعةً في
  موضع واحد — لزجة بين الجلسات.</p>
</details>

<details class="sec" data-sec="ai"><summary>المساعد</summary>
 <div class="row"><label>الطلب (Ctrl+Enter يرسل · أو ؟ في سطر
  الإدخال)</label>
  <textarea id="aiAsk" rows="3" style="width:100%"></textarea></div>
 <div class="btnrow"><button id="aiSend" class="pri">أرسِل</button>
 </div>
 <label class="chk"><input type="checkbox" id="aiIns">
  أرفِق تقرير الفاحص</label>
 <label class="chk"><input type="checkbox" id="aiDes">
  اسمح بالأوامر الهادمة</label>
 <label class="chk"><input type="checkbox" id="aiTtl">
  أرفِق بلوك العنوان</label>
 <div id="aiBox"></div>
 <details class="sub"><summary>توليد محلّي (بلا اتصال)</summary><div>
  <p class="hint">أنماطٌ ثابتة معدودة — لا تحتاج إعداد مزوّدٍ ولا
   اتصالاً بالشبكة. مثال: «غرفة 4 في 5 عند 0 0، باب على W1 عند 1.2»</p>
  <div class="row"><label>الوصف</label>
   <textarea id="genAsk" rows="2" style="width:100%"></textarea></div>
  <div class="btnrow"><button id="genRun" class="pri">توليد ونفّذ</button>
   <button id="genPrev">معاينة فقط</button></div>
  <div id="genBox"></div>
 </div></details>
 <details class="sub"><summary>الإعداد</summary><div>
  <label class="chk"><input type="checkbox" id="aiOn"> مُشغّل</label>
  <div class="row"><label>العنوان</label>
   <input id="aiUrl" type="text"></div>
  <div class="row"><label>الموديل</label>
   <input id="aiModel" type="text"></div>
  <div class="row2">
   <div class="f"><label>المفتاح</label>
    <input id="aiKey" type="password"></div>
   <div class="f"><label>الحرارة</label>
    <input id="aiTemp" type="number" min="0" max="1" step="0.1">
   </div>
  </div>
  <label class="chk"><input type="checkbox" id="aiVis">
   أرسِل صورة اللوحة (يحتاج موديلاً بصرياً)</label>
  <label class="chk"><input type="checkbox" id="aiKeep">
   احفظ المفتاح في هذا المتصفّح</label>
  <div class="btnrow"><button id="aiKeyClr" class="del">
   امسح المفتاح</button></div>
  <p class="hint">الإعداد في المتصفّح لا في ملفّ المشروع — لا
   يُحفَظ ولا يُصدَّر. لخصوصيةٍ كاملة استعمل Ollama محلّياً:
   <span class="mono">localhost:11434/v1/chat/completions</span>
  </p>
 </div></details>
</details>

<details class="sec" data-sec="state" open><summary>الحالة</summary>
 <div class="btnrow">
  <button id="bNew" class="del">مشروع جديد</button>
  <button id="bFit2">ملاءمة</button>
 </div>
 <div id="sumInfo" class="hint"></div>
</details>`;
}
/* ═══ النماذج ═══ */
const FORMIDS=["#mName","#mScale","#mTxt","#mExt","#mInt","#mH",
 "#mSnap","#mFill","#mDec","#mTick","#oJoins","#oSolo"];
export function loadForms(){
 const m=S.meta;
 $("#mName").value=m.name||"";
 $("#mScale").value=m.scale;
 $("#mTxt").value=m.txtMM;
 $("#mExt").value=mnum(m.tExt);
 $("#mInt").value=mnum(m.tInt);
 $("#mH").value=mnum(m.wallH);
 $("#mSnap").value=mnum(m.snap);
 $("#mFill").value=S.opt.fill;
 $("#mDec").value=String(m.dimDec);
 $("#mTick").value=m.dimTick;
 $("#oJoins").checked=!!+S.opt.joins;
 $("#oSolo").checked=!!+S.opt.colSolo;
}
function readForms(){
 const m=S.meta;
 m.name=$("#mName").value.trim()||m.name;
 m.scale=clamp(parseInt($("#mScale").value,10)||100,1,5000);
 m.txtMM=clamp(parseFloat($("#mTxt").value)||2.2,0.5,20);
 if(isLen($("#mExt").value))m.tExt=Math.max(50,M($("#mExt").value));
 if(isLen($("#mInt").value))m.tInt=Math.max(50,M($("#mInt").value));
 if(isLen($("#mH").value))m.wallH=clamp(M($("#mH").value),1500,8000);
 if(isLen($("#mSnap").value))m.snap=clamp(M($("#mSnap").value),1,5000);
 m.dimDec=clamp(parseInt($("#mDec").value,10)||0,0,3);
 m.dimTick=($("#mTick").value==="arrow")?"arrow":"slash";
 S.opt.fill=$("#mFill").value;
 S.opt.joins=$("#oJoins").checked?1:0;
 S.opt.colSolo=$("#oSolo").checked?1:0;
 ensureShape();
}
export function wireForms(){
 FORMIDS.forEach(id=>{
  const el=$(id);
  if(!el)return;
  el.addEventListener("change",()=>{edit(()=>readForms());
   refresh(false)});
  el.addEventListener("keydown",e=>{
   if(e.key==="Escape"){el.blur();return}
   e.stopPropagation();
  });
 });
 $("#bNew").onclick=()=>{
  if((S.walls.length||S.cols.length)
   &&!confirm("مشروع جديد؟ سيُفقد غير المحفوظ."))return;
  newState(); loadForms();
  import("./inspector.js").then(M=>M.clearFindings());
  refresh(true); fit();
  rep("ok","مشروع جديد");
 };
 $("#bFit2").onclick=()=>fit();
 $("#lAll").onclick=()=>{
  const n=edit(()=>showAll());
  if(editFailed())return;
  rep(n?"ok":"in",n?`أُظهرت ${n} طبقة`:"كل الطبقات ظاهرة");
  refresh(false);
 };
 $("#lUnlock").onclick=()=>{
  const n=edit(()=>unlockAll());
  if(editFailed())return;
  rep(n?"ok":"in",n?`فُتحت ${n} طبقة`:"لا طبقة مقفلة");
  refresh(false);
 };
 $("#lPlotAll").onclick=()=>{
  const n=edit(()=>plotAll());
  if(editFailed())return;
  rep(n?"ok":"in",n?`أُعيد طبع ${n} طبقة`:"كلُّها تُطبَع");
  refresh(false);
 };
 $("#lReset").onclick=()=>{
  if(!confirm("إعادة جدول الطبقات إلى مصنعه؟ الألوان والأوزان "
   +"والأنواع والشفافية والرؤية والقفل والطبع كلُّها تُصفَّر. "
   +"وحالاتُ الطبقات المحفوظة تبقى."))return;
  edit(()=>resetLays());
  if(editFailed())return;
  LCUR=null;
  rep("ok","أُعيد جدول الطبقات إلى مصنعه");
  refresh(false);
 };
 $("#bAxClr").onclick=()=>{
  if(!S.grid.xs.length&&!S.grid.ys.length)return;
  if(!confirm("مسح كل المحاور؟"))return;
  edit(()=>{S.grid.xs=[];S.grid.ys=[]});
  rep("ok","مُسحت المحاور");
  refresh(false);
 };
 $("#bRefAll").onclick=()=>{
  import("../tools/registry.js").then(R=>R.begin("arearef"));
 };
 /* ═══ سجلّ اللوحات ═══
    كلٌّ تُعلَن مرّة، ولا تُرسَم إلّا مرئيّة. جداول المساحات
    والفتحات والورقة والمرجع كانت تُبنى مع كل تعديل حقلٍ ثم تُخفى. */
 reg("lays",  "#lays",   renderLays,   "الطبقات");
 reg("props", "#props",  paintProps,   "الخصائص");
 reg("sched", "#sched",  renderSched,  "جدول المساحات");
 reg("osched","#osched", renderOSched, "جدول الفتحات");
 reg("axes",  "#axInfo", renderAxes,   "المحاور");
 reg("sheet", "#shInfo", ()=>syncSheet(),   "الورقة");
 reg("ref",   "#rInfo",  ()=>renderRef(),   "المرجع");
 reg("state", "#sumInfo",renderSummary,"الحالة");
 wirePanels();
}
/* ═══ الطبقات ═══
   الترتيبُ والتسميةُ واللونُ من الجدول الحيّ: صفٌّ يُضاف إلى
   laydef يظهر في اللوحة بلا لمسِ هذا الملفّ. */
const LORD=()=>LAYS().map(l=>({L:l.n,n:l.d||l.n,
 col:l.col,aux:AUX.has(l.n)}));
/* الطبقةُ المفتوحةُ للتحرير — حالُ عرضٍ عابرٌ لا تفضيلَ يُحفَظ */
let LCUR=null;
const LFN={col:"لون الشاشة",pcol:"لون الورق",lw:"وزن الخطّ",
 lt:"نوع الخطّ",op:"الشفافية",aci:"رمز ACI",d:"الوصف"};

/* ═══ محرِّرُ الطبقة ═══
   سبعةُ حقولٍ كانت مكتوبةً في الجدول ومقروءةً في المصدِّرين
   الأربعة، وبلا مدخلٍ من الواجهة. */
function laysEditor(nm){
 const l=layOf(nm);
 if(!l)return "";
 const aux=AUX.has(nm);
 return `<details class="sub" open><summary>${esc(l.d||nm)} `
  +`<span class="mono" style="color:var(--fg3)">${esc(nm)}</span>`
  +`</summary><div>`
  +F2(FF("لون الشاشة",
      `<input type="color" data-lf="col" value="${esc(l.col)}">`),
     FF("لون الورق",
      `<input type="color" data-lf="pcol" value="${esc(l.pcol)}">`))
  +F2(FF("وزن الخطّ مم",
      `<select data-lf="lw">`+LWS.map(([w,t])=>
       `<option value="${w}"${w===l.lw?" selected":""}>`
       +`${esc(t)}</option>`).join("")+`</select>`),
     FF("نوع الخطّ",
      `<select data-lf="lt">`+Object.keys(LT).map(k=>
       `<option value="${esc(k)}"${k===l.lt?" selected":""}>`
       +`${esc(LT[k].n)}</option>`).join("")+`</select>`))
  +F2(FF("الشفافية ٪",
      `<input type="number" class="num" data-lf="op" min="0" `
      +`max="90" step="5" value="${l.op}">`),
     FF("رمز ACI",
      `<input type="number" class="num" data-lf="aci" min="0" `
      +`max="256" step="1" value="${l.aci}">`))
  +F("الوصف",`<input type="text" data-lf="d" `
    +`value="${esc(l.d||"")}">`)
  +`<p class="hint">لونان لا لونٌ واحد: الجدار على شاشةٍ داكنة `
  +`قريبٌ من الأبيض وعلى الورق أسود — عكسُ خلفيةٍ لا انجراف. `
  +`ولونُ الورق هو لونُ الشاشة الفاتحة نفسه، وهو ما يُكتَب في `
  +`DXF لوناً حقيقياً (420) ورمزَ ACI احتياطاً.`
  +(aux?`<br>طبقةٌ مساعدة: تُرسَم ولا كياناتَ تُحدَّد عليها، `
    +`فلا قفلَ لها.`:"")
  +`</p></div></details>`;
}
/* ═══ حالاتُ الطبقات ═══ لقطةٌ مسمّاة — هيئةٌ لا هويّة ═══ */
function laysStates(){
 const A=layStates();
 return `<details class="sub"><summary>حالات الطبقات`
  +(A.length?` · ${A.length}`:"")+`</summary><div>`
  +(A.length
   ? `<div class="row2"><div class="f">`
     +`<select id="lStName">`
     +A.map(n=>`<option value="${esc(n)}">${esc(n)}</option>`)
      .join("")+`</select></div>`
     +`<div class="f" style="display:flex;gap:4px;`
     +`align-items:flex-end">`
     +`<button id="lStGo" class="pri" style="flex:1">طبّق</button>`
     +`<button id="lStDel" class="del" `
     +`style="flex:none;width:32px">✕</button></div></div>`
   : `<p class="hint">لا حالاتٍ محفوظة</p>`)
  +`<div class="btnrow"><button id="lStSave">`
  +`احفظ الحالة الجارية…</button></div>`
  +`<p class="hint">تحمل الرؤية والقفل والطبع واللونين والوزن `
  +`والنوع والشفافية — ولا تحمل الاسم ولا الوصف: هيئةٌ لا هويّة. `
  +`وهي بياناتُ مشروعٍ تُحفَظ في الملفّ وتدخل التاريخ.</p>`
  +`</div></details>`;
}
function renderLays(box){
 if(LCUR&&!hasLay(LCUR))LCUR=null;
 const C=layCounts();
 box.innerHTML=LORD().map(x=>{
  const on=vis(x.L), lk=locked(x.L), pl=plots(x.L);
  const n=C[x.L]||0, cur=(LCUR===x.L);
  /* عنوانُ القفل يُحسَب هنا: كسرُ السطر داخل ${…} كان يفتح قالباً
     ثانياً فيصير النصُّ وسماً له — رميٌ يُسقِط بناءَ الجدول كلّه. */
  const ltip=x.aux?"طبقةٌ مساعدة — لا كياناتَ تُقفَل"
   :(lk?"افتح":"اقفل");
  return `<div style="display:flex;gap:4px;align-items:center;`
   +`margin:2px 0;opacity:${on?1:0.45}`
   +`${cur?";background:var(--bg4);border-radius:3px":""}">`
   +`<button data-loff="${esc(x.L)}" title="${on?"أخفِ":"أظهر"}" `
   +`style="flex:none;width:22px;padding:2px 0">`
   +`${on?"◉":"○"}</button>`
   /* الطبعُ مستقلٌّ عن الرؤية: A-REFR مصنعُه «لا يُطبَع»، ولا
      سبيلَ إلى طبعه قبل هذا الزرّ. */
   +`<button data-lplot="${esc(x.L)}" `
   +`title="${pl?"تُطبَع":"لا تُطبَع"}" `
   +`style="flex:none;width:22px;padding:2px 0`
   +`${pl?"":";color:var(--fg3)"}">${pl?"⎙":"⌀"}</button>`
   +`<button data-llock="${esc(x.L)}" title="${esc(ltip)}"`
   +`${x.aux?" disabled":""} `
   +`style="flex:none;width:22px;padding:2px 0`
   +`${lk?";color:var(--wr)":""}">${lk?"▣":"▢"}</button>`
   +`<button data-lsel="${esc(x.L)}" title="خصائص الطبقة" `
   +`style="flex:none;width:14px;height:14px;padding:0;`
   +`background:${esc(x.col||"#888")};border-radius:2px;`
   +`border:1px solid var(--ln)"></button>`
   +`<button data-lsel="${esc(x.L)}" title="خصائص الطبقة" `
   +`style="flex:1;min-width:0;text-align:start;`
   +`background:transparent;border:none;padding:1px 2px;`
   +`font-size:11.5px;color:${cur?"var(--ac)":"var(--fg2)"};`
   +`overflow:hidden;text-overflow:ellipsis;white-space:nowrap">`
   +`${esc(x.n)}</button>`
   +`<span class="ro" style="flex:none;font-size:10.5px">`
   +`${n}</span>`
   +`<button data-liso="${esc(x.L)}" title="انفراد" `
   +`style="flex:none;width:22px;padding:2px 0">◧</button>`
   +`</div>`;
 }).join("")
 +(LCUR?laysEditor(LCUR):"")
 +laysStates();
 const W=[];
 if(anyHidden())W.push(`${hiddenCount()} كياناً مخفيّاً — لن `
  +`يُصدَّر ولن يُحدَّد`);
 const np=noPlotCount();
 if(np)W.push(`${np} كياناً يُرى ولا يُطبَع`);
 if(anyLocked())W.push(`طبقاتٌ مقفلة — تُرى ولا تُلمَس`);
 if(W.length)box.innerHTML+=`<p class="hint warn">`
  +W.map(esc).join("<br>")+`</p>`;
}
/* ═══ لوحات الخصائص ═══ */
const F=(l,i)=>`<div class="row"><label>${l}</label>${i}</div>`;
const F2=(a,b)=>`<div class="row2">${a}${b}</div>`;
const FF=(l,i)=>`<div class="f"><label>${l}</label>${i}</div>`;
const IN=(p,k,v,x)=>`<input data-p="${p}" data-k="${k}" `
 +`type="${k==="num"?"number":"text"}" value="${esc(v)}" ${x||""}>`;
const SE=(p,arr,cur)=>`<select data-p="${p}" data-k="sel">`
 +OPT(arr,cur)+`</select>`;
const CK2=(p,on,lbl)=>`<div class="row"><label class="chk">`
 +`<input type="checkbox" data-p="${p}" data-k="chk"`
 +`${on?" checked":""}> ${lbl}</label></div>`;
const RO=v=>`<span class="ro num">${esc(v)}</span>`;

/* ═══ متحكِّمُ الحقل ═══ للوحاتِ الثلاث ═══
   المُوحَّدُ هو المتحكِّم لا الصفّ: props تلفّه في .row وquickprops
   في .qf، فالغلافُ هيئةُ لوحةٍ والمتحكِّمُ عقدُ حقل — اسمُ السمة
   وقائمةُ العناصر والحدُّ والقيمةُ وحالُ التعدّد.

   C سياقُ العرض: kind (جماعيّ ⇒ data-bk/bf) · val · mixed · own
   · num (‏len رقمياً لا نصّاً) */
const HTML_LIMITS=1;   /* سِمَتا min/max على العدديّ — أطفئها إن
                          نازعت CSS الخاصّةَ بـ:invalid */
const fldTag=(d,C)=>C.kind
 ? `data-bk="${esc(C.kind)}" data-bf="${esc(d.k)}" `
   +`data-k="${esc(d.t)}"`
 : `data-p="${esc(d.k)}" data-k="${esc(d.t)}"`;

/* الثابتُ يصل الحقلَ سِمةً · والتابعُ لا: قيمتُه تتبدّل بحاضنه،
   وسِمةٌ كاذبةٌ أسوأ من غيابها. وparseVal يفحصهما جميعاً. */
const fldAttr=d=>{
 if(d.t!=="num")return "";
 let a=` step="${d.step||(d.int?1:0.1)}"`;
 if(HTML_LIMITS){
  if(typeof d.min==="number")a+=` min="${d.min}"`;
  if(typeof d.max==="number")a+=` max="${d.max}"`;
 }
 return a;
};
const fldShow=(d,v)=>(v==null||v==="")?""
 :((d.t==="len")?mnum(v):String(v));

export function fldCtl(d,val,ctx){
 const C=ctx||{};
 const tag=fldTag(d,C), mix=!!C.mixed;
 if(d.t==="chk")
  return `<input type="checkbox" ${tag}`
   +`${(!mix&&+val)?" checked":""}${mix?' data-mix="1"':""}>`;
 if(d.t==="sel")
  return `<select ${tag}>`
   +(mix?`<option value="" selected>— متعدّد`
     +`${C.own?` (${C.own})`:""} —</option>`:"")
   +(d.items||[]).map(([v,n])=>`<option value="${esc(v)}"`
    +`${(!mix&&String(val)===String(v))?" selected":""}>`
    +`${esc(n)}</option>`).join("")
   +`</select>`;
 /* len نصٌّ في الثلاث: الأرقامُ الهندية والفاصلةُ العربية تُقبَل،
    وclass="num" يعزل الاتجاه. وكان رقمياً في الجماعية والسريعة
    فتُرفَض «٢٫٥» فيهما وتُقبَل في المفردة. */
 const asNum=(d.t==="num");
 return `<input type="${asNum?"number":"text"}" ${tag} class="num" `
  +`value="${esc(mix?"":fldShow(d,val))}"`
  +(mix?` placeholder="متعدّد${C.own?` (${C.own})`:""}"`:"")
  +fldAttr(d)+`>`;
}
const fldLab=(d,C)=>esc(d.n)
 +((C&&C.own!=null&&C.all!=null&&C.own<C.all)
  ? ` <span class="ro" style="font-size:10px;display:inline">`
    +`${C.own}/${C.all}</span>` : "");
const fldTip=(d,C)=>(d.hint&&(!C||C.hint!==0))
 ? `<p class="hint">${esc(d.hint)}</p>` : "";

/* القيمةُ من ctx.val إن أُعطيت (الجماعية) وإلّا من الكيان */
const fldVal=(kind,e,d,C)=>(C&&C.val!==undefined)
 ? C.val : fieldVal(kind,e,d.k);

/* صفٌّ كامل */
function fldRow(kind,e,key,ctx){
 const d=fldOf(kind,key);
 if(!d)return "";
 const C=ctx||{};
 const v=fldVal(kind,e,d,C);
 if(d.t==="chk")
  return `<div class="row"><label class="chk">`
   +fldCtl(d,v,C)+` ${fldLab(d,C)}</label></div>`+fldTip(d,C);
 return F(fldLab(d,C),fldCtl(d,v,C))+fldTip(d,C);
}
/* نصفُ صفّ — يُركَّب يدوياً مع حقلٍ غيرِ مولَّد.
   وhint لا يُرسَم هنا: <p> داخل row2 يكسر التخطيط. */
function fldHalf(kind,e,key,ctx){
 const d=fldOf(kind,key);
 if(!d)return "";
 const C=ctx||{};
 return FF(fldLab(d,C),fldCtl(d,fldVal(kind,e,d,C),C));
}

function wallProps(w){
 const d=dir(w);
 let h=`<div class="phead">${esc(w.id)} · جدار</div>`;
 h+=F2(FF("الطول م",RO(m3(wallLen(w)))),
       FF("الزاوية",RO(d?d.ang.toFixed(2)+"°":"—")));
 h+=F2(FF("البداية",RO(`${m2(w.a[0])} , ${m2(w.a[1])}`)),
       FF("النهاية",RO(`${m2(w.b[0])} , ${m2(w.b[1])}`)));
 h+=F2(fldHalf("wall",w,"t"), fldHalf("wall",w,"type"));
 h+=fldRow("wall",w,"align",{hint:0});
 /* ارتفاعُ السترة ليس في الجدول: لا مقابلَ له جماعياً */
 if(isLow(w))h+=F("ارتفاع السترة م",IN("h","len",mnum(lowH(w))));
 const O=S.opens.filter(o=>o.wall===w.id);
 h+=`<p class="hint">${O.length} فتحة عليه`
  +(O.length?`: ${O.map(o=>o.id).join(" · ")}`:"")+`</p>`;
 return h;
}
function openProps(o){
 const w=wallById(o.wall);
 const K=OK[o.kind]||OK.door;
 const st=openState(o);
 let h=`<div class="phead">${esc(o.id)} · `
  +`${esc(okName(o.kind))}</div>`;
 /* ثلاثةٌ من أربعةِ صفوفٍ تُزوّج مولَّداً بغيرِ مولَّد، فالتخطيطُ
    يبقى مُركَّباً بيدٍ وfldHalf تُعطي النصف. */
 h+=fldRow("open",o,"kind",{hint:0});
 h+=F2(fldHalf("open",o,"w"), fldHalf("open",o,"h"));
 h+=F2(fldHalf("open",o,"sill"),
       FF("الموضع م",IN("s","len",mnum(o.s))));
 if(K.sw)
  h+=F2(FF("المفصّلة",SE("hinge",[["start","البداية"],
        ["end","النهاية"]],o.hinge)),
        fldHalf("open",o,"swing"));
 if(K.pan)h+=F("المصاريع",IN("pan","num",panOf(o),
  'min="1" max="6"'));
 if(o.kind==="niche")
  h+=F2(fldHalf("open",o,"dep"),
        FF("الوجه",SE("face",[["l","يسار المسار"],
         ["r","يمينه"]],o.face||"l")));
 /* ═══ وما بعده عرضٌ محضٌ لا إدخال — لا يُولَّد من جدولٍ عامّ ═══ */
 if(w){
  const A=allowed(w,o.w,o);
  const [a,b]=span(o);
  h+=`<p class="hint">على ${esc(w.id)} · طوله ${m2(wallLen(w))} م · `
   +`المدى ${esc(rng2(a,b,"م"))}`
   +(A.fits?`<br>المواضع الحرّة ${esc(saySpans(A))} م`
     +(A.split?` <span class="warn">(متقطّعة)</span>`:"")
    :`<br><span class="warn">لا موضع حرٌّ بهذا العرض</span>`)
   +`</p>`;
 }
 if(st==="over")
  h+=`<p class="hint warn">تخرج عن مدى جدارها — الطرح مقصوص على
   المدى، والبيانات لم تُمَس. عدّل الموضع أو العرض.</p>`;
 else if(st==="clash")
  h+=`<p class="hint warn">تتراكب مع فتحة أخرى على الجدار نفسه.</p>`;
 return h;
}
function areaProps(a){
 const st=isStale(a);
 let h=`<div class="phead">${esc(a.id)} · منطقة`
  +(st?` · <span style="color:var(--wr)">قديمة</span>`:"")+`</div>`;
 h+=fldRow("area",a,"name",{hint:0});
 h+=F2(FF("المساحة م²",RO(sqm(netArea(a)))),
       FF("المحيط م",RO(m3(netPerim(a)))));
 h+=F2(FF("الأضلاع",RO(a.ring.length)),
       fldHalf("area",a,"fill"));
 h+=fldRow("area",a,"showArea",{hint:0});
 h+=`<p class="hint">التسمية ${a.lp?"في موضع صريح — لا تزحف"
  :"في القطب المحسوب · اسحب مقبضها لتثبيتها"}</p>`;
 if(st){
  h+=`<p class="hint warn">تغيّر جدار يجاورها. الحلقة المخزَّنة لم
   تُمَسّ — «تحديث» يعيد خبزها، و«تثبيت» يقبل الوضع الحالي.</p>`;
  h+=`<div class="btnrow">
   <button data-aref="${esc(a.id)}" class="pri">تحديث</button>
   <button data-astm="${esc(a.id)}">تثبيت البصمة</button></div>`;
 }
 return h;
}
function dimProps(d){
 const loose=dimLoose(d,30);
 let h=`<div class="phead">${esc(d.id)} · بُعد ${DK[d.kind]}`
  +(loose?` · <span style="color:var(--wr)">معلَّق</span>`:"")
  +`</div>`;
 h+=F2(FF("المقاس م",RO(fmtLen(dimValue(d)))),
       fldHalf("dim",d,"kind"));
 h+=F2(FF("الطرف الأول",RO(`${m2(d.a[0])} , ${m2(d.a[1])}`)),
       FF("الطرف الثاني",RO(`${m2(d.b[0])} , ${m2(d.b[1])}`)));
 h+=fldRow("dim",d,"txt",{hint:0});
 if(isOverridden(d))
  h+=`<p class="hint warn">النصّ البديل يُعرَض بدل المقاس الحقيقي
   (${fmtLen(dimValue(d))} م) وعليه علامة *. أفرغ الحقل ليعود
   المقاس.</p>`;
 if(loose)
  h+=`<p class="hint warn">طرفٌ لا يصادف هندسةً — البُعد لم يُزحَف
   ولم يُحذَف. اسحب مقبضه إلى الموضع الصحيح، أو اتركه.</p>`;
 h+=`<p class="hint">النقطتان وموضع الخطّ مخزَّنة صريحةً — لا ترتبط
  بجدار فلا تزحف معه</p>`;
 return h;
}
function chainProps(c){
 const V=chainVals(c);
 let h=`<div class="phead">${esc(c.id)} · سلسلة `
  +`${c.axis==="h"?"أفقية":"رأسية"}</div>`;
 /* القيَمُ ليست في الجدول: نصٌّ مركَّبٌ يُحلَّل بـparseVals */
 h+=F("القيَم م",IN("vals","text",V.map(v=>mnum(v)).join(" ")));
 h+=F2(FF("العدد",RO(V.length)),
       FF("المجموع م",RO(fmtLen(chainSum(c)))));
 h+=fldRow("chain",c,"total",{hint:0});
 h+=`<div class="btnrow">`
  +`<button data-ccmp="${esc(c.id)}">قارن بالهندسة</button></div>`;
 h+=`<p class="hint">القيَم كما كتبتها. المقارنة تقريرٌ لا تصحيح —
  لا تُعدَّل قيمة إلا بيدك في هذا الحقل.</p>`;
 return h;
}
function annoProps(a){
 let h=`<div class="phead">${esc(a.id)} · ${esc(AK[a.kind])}</div>`;
 if(a.kind==="level"){
  h+=F("المنسوب م",IN("z","len",mnum(a.z)));
  h+=F2(fldHalf("anno",a,"pre"),
        FF("المعروض",RO(levelStr(a))));
 }else{
  h+=F("النصّ",IN("s","text",a.s));
  h+=F2(fldHalf("anno",a,"hm"),
        a.kind==="text"
         ?FF("الدوران °",IN("rot","num",a.rot||0))
         :FF("النقاط",RO(a.pts.length)));
  if(a.kind==="text")h+=fldRow("anno",a,"al",{hint:0});
 }
 return h;
}
function colProps(c){
 const on=colOnWall(c,2);
 let h=`<div class="phead">${esc(c.id)} · عمود`
  +(c.tag?` ${esc(c.tag)}`:"")+`</div>`;
 h+=F2(fldHalf("col",c,"kind"), fldHalf("col",c,"type"));
 if(c.kind==="circ")h+=fldRow("col",c,"w",{hint:0});
 else h+=F2(fldHalf("col",c,"w"), fldHalf("col",c,"h"));
 h+=F2(fldHalf("col",c,"rot"),
       FF("المساحة م²",RO((colArea(c)/1e6).toFixed(3))));
 /* القطبُ والوسمُ ليسا في الجدول: لا مقابلَ لهما جماعياً */
 h+=F2(FF("المركز x",IN("x","len",mnum(c.x))),
       FF("المركز y",IN("y","len",mnum(c.y))));
 h+=F("الوسم",IN("tag","text",c.tag||""));
 h+=`<p class="hint">${on
  ?`يُدمَج مع ${esc(on)} في الجسم المصمَّت — دمجٌ وقت العرض، `
   +`والبيانات مستقلّة`
  :`منفرد — لا يلامس جداراً. المساحة لا تخصمه.`}</p>`;
 return h;
}
function fixProps(f){
 const on=fixOnWall(f,150);
 let h=`<div class="phead">${esc(f.id)} · `
  +`${esc(fixName(f))}</div>`;
 h+=fldRow("fix",f,"kind",{hint:0});
 h+=F2(fldHalf("fix",f,"w"), fldHalf("fix",f,"d"));
 h+=F2(fldHalf("fix",f,"rot"),
       FF("الموضع",RO(`${m2(f.x)} , ${m2(f.y)}`)));
 h+=fldRow("fix",f,"mir",{hint:0});
 h+=`<div class="btnrow"><button data-fsnap="${esc(f.id)}">`
  +`ألصِق بأقرب جدار</button></div>`;
 h+=`<p class="hint">${on?`ظهرها على ${esc(on)}`
  :"ظهرها حرّ"} — الإحداثيات صريحة، والإلصاق أمرٌ لا رابطة</p>`;
 return h;
}
function stairProps(t){
 const c=stCheck(t), g=stGeom(t);
 let h=`<div class="phead">${esc(t.id)} · درج`
  +(c.ok?"":` · <span style="color:var(--wr)">خارج المدى</span>`)
  +`</div>`;
 h+=F2(fldHalf("stair",t,"w"), fldHalf("stair",t,"n"));
 h+=F2(FF("طول القِلعة م",RO(g?m3(g.L):"—")),
       fldHalf("stair",t,"h"));
 h+=F2(FF("القائمة م",RO(m3(c.rise))),
       FF("النائمة م",RO(m3(c.tread))));
 h+=F2(FF("2ق + ن م",RO(m3(c.rule))),
       fldHalf("stair",t,"up"));
 h+=fldRow("stair",t,"cut",{hint:0});
 if(!c.ok)
  h+=`<p class="hint warn">${c.msgs.join("<br>")}<br>`
   +`القياسات كما رسمتها — عدّل الطول أو عدد القوائم. لا شيء `
   +`يُصحَّح تلقائياً.</p>`;
 else
  h+=`<p class="hint" style="color:var(--ok)">القياسات داخل المدى `
   +`المريح</p>`;
 return h;
}
/* ═══ التحديد المتعدّد ═══ */
function multiProps(L){
 const G=groupSel(L);
 const KS=groupOrder(G);
 const out=(L.length-KS.reduce((s,k)=>s+G[k].length,0));
 let h=`<p class="hint">${L.length} عنصر: ${esc(summary(G))}`
  +(out?` · ${out} خارج التعديل (مخفيّ أو مقفل)`:"")+`</p>`;
 h+=`<div class="btnrow">
  <button data-run="move">نقل</button>
  <button data-run="copy">نسخ</button>
  <button data-run="rotate">دوران</button></div>
 <div class="btnrow">
  <button data-run="mirror">مرآة</button>
  <button data-run="weld">لحم</button></div>`;
 KS.forEach(k=>{
  const arr=G[k];
  h+=`<div class="phead" style="margin-top:8px">`
   +`${esc(NAME[k]||k)} · ${arr.length}</div>`;
  FLD[k].forEach(f=>{
   const r=readField(k,arr,f.k);
   if(!r.own)return;                 /* لا أحد يملكه فلا يُعرَض */
   /* arr مصفّاةٌ من groupSel فطولُها هو القابلُ للتعديل فعلاً،
      لا طولُ التحديد الخام (r.n) — والنسبةُ صادقة.
      وhint:0 فالتخطيطُ يبقى كما هو. */
   h+=fldRow(k,null,f.k,{kind:k,
    val:r.mixed?null:r.value, mixed:!!r.mixed,
    own:r.own, all:arr.length, hint:0});
  });
  /* الأوامر الجماعية الصريحة */
  if(k==="col")h+=`<div class="btnrow">`
   +`<button data-brenum="1">أعد الترقيم من أعلى اليمين</button>`
   +`</div>`;
  if(k==="dim")h+=`<div class="btnrow">`
   +`<button data-bcleartxt="1">امسح النصّ البديل</button></div>`;
  if(k==="area")h+=`<div class="btnrow">`
   +`<input id="bASeq" type="text" placeholder="سابقة الاسم" `
   +`style="flex:1">`
   +`<button data-bname="1">سمِّ بالتسلسل</button></div>`
   +`<div class="btnrow"><button data-bclrname="1">`
   +`امسح الأسماء</button></div>`;
 });
 h+=`<p class="hint">الحقل الفارغ بعلامة «متعدّد» لا يُكتب من `
  +`تلقائه — اكتب فيه لتطبّقه على الجميع.</p>`;
 h+=`<div class="btnrow"><button id="pDel" class="del">`
  +`حذف المحدد</button></div>`;
 return h;
}
function paintProps(box){
 const L=selList();
 if(!L.length){
  box.innerHTML=`<p class="hint">لا تحديد — انقر عنصراً، أو اسحب `
   +`إطاراً على الفراغ</p>`;
  eSel(""); return;
 }
 if(L.length>1){
  box.innerHTML=multiProps(L);
  /* المربّع المختلف: ثلاثيّ الحالة حتى تحسمه */
  box.querySelectorAll('[data-mix="1"]').forEach(el=>{
   el.indeterminate=true;
  });
  eSel(`${L.length} عنصر`);
  return;
 }
 const s=L[0];
 let h="";
 if(s.k==="wall"){
  const w=wallById(s.id);
  if(!w){box.innerHTML="";return}
  h=wallProps(w); eSel(`جدار ${w.id}`);
 }else if(s.k==="open"){
  const o=openById(s.id);
  if(!o){box.innerHTML="";return}
  h=openProps(o); eSel(`فتحة ${o.id}`);
 }else if(s.k==="area"){
  const a=areaById(s.id);
  if(!a){box.innerHTML="";return}
  h=areaProps(a); eSel(`منطقة ${a.id}`+(isStale(a)?" (قديمة)":""));
 }else if(s.k==="dim"){
  const d=dimById(s.id);
  if(!d){box.innerHTML="";return}
  h=dimProps(d); eSel(`بُعد ${d.id}`);
 }else if(s.k==="chain"){
  const c=chainById(s.id);
  if(!c){box.innerHTML="";return}
  h=chainProps(c); eSel(`سلسلة ${c.id}`);
 }else if(s.k==="anno"){
  const a=annoById(s.id);
  if(!a){box.innerHTML="";return}
  h=annoProps(a); eSel(`${AK[a.kind]} ${a.id}`);
 }else if(s.k==="col"){
  const c=colById(s.id);
  if(!c){box.innerHTML="";return}
  h=colProps(c); eSel(`عمود ${c.id}`);
 }else if(s.k==="fix"){
  const f=fixById(s.id);
  if(!f){box.innerHTML="";return}
  h=fixProps(f); eSel(`${fixName(f)} ${f.id}`);
 }else if(s.k==="stair"){
  const t=stById(s.id);
  if(!t){box.innerHTML="";return}
  h=stairProps(t); eSel(`درج ${t.id}`);
 }else{box.innerHTML="";return}

 const LY=layOfEnt(s);
 if(LY)h+=`<p class="hint">الطبقة: ${esc(LNAME(LY))}`
  +`${locked(LY)?" · مقفلة":""}</p>`;
 h+=`<div class="btnrow"><button id="pDel" class="del">حذف</button>`
  +`</div>`;
 h+=`<p class="hint">الإحداثيات تُعدَّل بالمقابض على اللوحة — `
  +`لا شيء يتحرّك دونك</p>`;
 box.innerHTML=h;
}
/* شريط الحالة يتحدّث دائماً · وجسم اللوحة إن كان مرئيّاً وحده */
const selBrief=()=>{
 const L=selList();
 if(!L.length)return "";
 return (L.length>1)?`${L.length} عنصر`
  :`${NAME[L[0].k]||L[0].k} ${L[0].id}`;
};
export function renderProps(){
 if(!renderPanel("props"))eSel(selBrief());
}
/* SETFIELDS تُحذَف: كانت قائمةً يدويّةً موازيةً لـFLD، ومفاتيحُ
   SET كانت ≡ مفاتيحَ FLD في كل نوع — فالسؤالُ من المصدر. */
const owned=(k,f)=>!!fldOf(k,f);

document.addEventListener("change",e=>{
 /* ═══ محرِّرُ الطبقة ═══ يسبق كل شيء: سماتُه مستقلّة ═══ */
 const lf=e.target.dataset.lf;
 if(lf){
  if(!LCUR){rep("wr","لا طبقةَ مفتوحة");return}
  const num=/^(lw|op|aci)$/.test(lf);
  const v=num?parseInt(e.target.value,10):e.target.value;
  const done=edit(()=>setLay(LCUR,lf,v));
  if(editFailed())return;
  if(!done)rep("er",`${LFN[lf]||lf}: قيمةٌ لا تُقبَل — `
   +`لم يُكتَب شيء`);
  else rep("in",`${LNAME(LCUR)} · ${LFN[lf]||lf}: `
   +`${esc(String(e.target.value))}`);
  refresh(false);
  return;
 }
 /* ═══ الجماعيُّ يسبق المفرد ═══ */
 const bk=e.target.dataset.bk, bf=e.target.dataset.bf;
 if(bk&&bf){
  const kd=e.target.dataset.k;
  const arr=(groupSel(H_sel())[bk])||[];
  if(!arr.length){rep("wr","لا عنصر قابل للتعديل");return}
  let raw;
  if(kd==="chk"){
   e.target.indeterminate=false;
   raw=e.target.checked?1:0;
  }else raw=e.target.value;
  /* والسياسةُ صريحةٌ في موضع النداء: الفراغُ هنا «لا تكتب» */
  const r=edit(()=>applyField(bk,arr,bf,raw,{skipBlank:1}));
  if(r){
   if(r.blank)
    rep("in",`${fldName(bk,bf)}: تُرك فارغاً — لم يُكتب شيء`);
   else{
    rep(r.refused.length?"wr":"ok",sayApply(r));
    r.refused.slice(0,8).forEach(x=>
     rep("er",`  ${x.id}: ${x.msg}`));
    if(r.refused.length>8)
     rep("in",`  … و ${r.refused.length-8} رفضاً آخر`);
    /* والقسرُ مجموعٌ بسببه — كان يقع صامتاً */
    groupForced(r.forced).forEach(m=>rep("in","  "+m));
   }
  }
  refresh(false);
  return;
 }
 const p=e.target.dataset.p;
 if(!p)return;
 const L=selList();
 if(L.length!==1)return;
 const s=L[0];
 const kind=e.target.dataset.k, raw=e.target.value;
 /* ═══ ما يملكه الجدول يمرّ بمُثبِّته ═══
    parseVal يحلّل ويفحص المدى، فلا فحصَ مسبقٌ هنا ولا رسالةَ
    ثانية — والفراغُ في المفردة قيمةٌ (مسحُ اسمٍ أو نصٍّ بديل). */
 if(owned(s.k,p)){
  const v=(kind==="chk")?(e.target.checked?1:0):raw;
  const r=edit(()=>applyOne(s,p,v));
  if(r){
   if(!r.ok)rep("er",r.msg);
   else if(r.msg)rep("in",r.msg);   /* القسرُ يُقال */
   if(s.k==="open"){
    const o=openById(s.id);
    if(o){
     const st=openState(o);
     if(st==="over")rep("wr",`${o.id} صارت تخرج عن مدى جدارها`);
     else if(st==="clash")
      rep("wr",`${o.id} صارت تتراكب مع فتحة أخرى`);
    }
   }else if(s.k==="stair"){
    const t=stById(s.id);
    if(t){
     const c=stCheck(t);
     if(!c.ok)rep("wr",`${t.id}: `+c.msgs.join(" · "));
     else rep("ok",`${t.id}: ق ${m3(c.rise)} · ن ${m3(c.tread)} م`);
    }
   }
  }
  refresh(false);
  return;
 }
 /* ═══ وما بقي: حقولُ اللوحة المفردة وحدها ═══
    موضعُها وقطبها ومفصّلتها ونصُّها — لا مقابلَ لها جماعياً،
    فتُحلَّل هنا. */
 let v=raw;
 if(kind==="len"){
  if(!isLen(raw)){rep("er",`«${raw}» ليس طولاً`);renderProps();return}
  v=M(raw);
 }else if(kind==="num"){
  v=parseFloat(raw);
  if(!isFinite(v)){rep("er",`«${raw}» ليس رقماً`);renderProps();return}
 }else if(kind==="chk")v=e.target.checked?1:0;

 if(s.k==="wall"){
  const w=wallById(s.id);
  if(!w)return;
  edit(()=>{if(p==="h")w.h=Math.max(200,v)});
 }else if(s.k==="open"){
  const o=openById(s.id);
  if(!o)return;
  edit(()=>{
   if(p==="s")o.s=Math.round(v);
   else if(p==="hinge")o.hinge=(v==="end")?"end":"start";
   else if(p==="pan")o.pan=clamp(Math.round(v),1,6);
   else if(p==="face")o.face=(v==="r")?"r":"l";
  });
  const st=openState(o);
  if(st==="over")rep("wr",`${o.id} صارت تخرج عن مدى جدارها`);
  else if(st==="clash")rep("wr",`${o.id} صارت تتراكب مع فتحة أخرى`);
 }else if(s.k==="chain"){
  const c=chainById(s.id);
  if(!c)return;
  if(p==="vals"){
   try{
    const V=parseVals(raw);
    edit(()=>{c.vals=V});
    rep("ok",`${c.id}: ${V.length} قيمة · `
     +`المجموع ${fmtLen(V.reduce((x,y)=>x+y,0))} م`);
   }catch(er){rep("er",er.message)}
   refresh(false);
   return;
  }
 }else if(s.k==="anno"){
  const a=annoById(s.id);
  if(!a)return;
  edit(()=>{
   if(p==="s")a.s=String(raw).slice(0,120);
   else if(p==="rot")a.rot=deg(v);
   else if(p==="z")a.z=v;
  });
 }else if(s.k==="col"){
  const c=colById(s.id);
  if(!c)return;
  edit(()=>{
   if(p==="x")c.x=Math.round(v);
   else if(p==="y")c.y=Math.round(v);
   else if(p==="tag"){
    const t=String(raw).slice(0,10);
    if(t)c.tag=t; else delete c.tag;
   }
  });
 }
 refresh(false);
});
/* ═══ مستمع النقر ═══ */
document.addEventListener("click",e=>{
 const t=e.target;
 /* صفّ جدول الفتحات ⇒ تحديد فتحاته */
 const osr=t.closest("[data-osel]");
 if(osr){
  const L=String(osr.dataset.osel||"").split(" ").filter(Boolean)
   .map(id=>({k:"open",id})).filter(pickable);
  setSel(L,L[L.length-1]||null);
  rep(L.length?"in":"wr",L.length
   ?`${L.length} فتحة محدَّدة — عدّلها جماعياً من «الخصائص»`
   :"فتحات هذا الصفّ مخفيّة أو مقفلة");
  draw(); return;
 }
 /* الطبقات */
 const lo=t.dataset.loff;
 if(lo){
  edit(()=>toggleOff(lo));
  const d=pruneSel();
  rep(vis(lo)?"in":"wr",
   `${LNAME(lo)}: ${vis(lo)?"ظاهرة":"مخفيّة"}`
   +(d?` · خرج ${d} عنصراً من التحديد`:""));
  refresh(false); return;
 }
 const lp=t.dataset.lplot;
 if(lp){
  edit(()=>setLay(lp,"plot",plots(lp)?0:1));
  rep("in",`${LNAME(lp)}: ${plots(lp)?"تُطبَع"
   :"لا تُطبَع — تُرى على الشاشة وتُستثنى من المخرَج"}`);
  refresh(false); return;
 }
 const ll=t.dataset.llock;
 if(ll){
  edit(()=>toggleLock(ll));
  const d=pruneSel();
  rep("in",`${LNAME(ll)}: ${locked(ll)?"مقفلة — تُرى ولا تُلمَس"
   :"مفتوحة"}`+(d?` · خرج ${d} عنصراً من التحديد`:""));
  refresh(false); return;
 }
 const li=t.dataset.liso;
 if(li){
  edit(()=>isolate(li));
  pruneSel();
  rep("in",`انفراد ${LNAME(li)} — «أظهر الكل» يعيد الجميع`);
  refresh(false); return;
 }
 /* ═══ فتحُ الطبقة للتحرير ═══ نقرةٌ ثانيةٌ تُغلقه ═══ */
 const lsl=t.closest("[data-lsel]");
 if(lsl){
  const n=lsl.dataset.lsel;
  LCUR=(LCUR===n)?null:n;
  renderPanel("lays",1);
  return;
 }
 if(t.id==="lStSave"){
  const nm=prompt("اسم حالة الطبقات:",
   layStates()[0]||"تسليم");
  if(nm==null)return;
  const r=edit(()=>stateSave(nm));
  if(editFailed())return;
  rep(r?"ok":"wr",r?`حُفظت الحالة «${r}»`:"الاسم فارغ");
  renderPanel("lays",1);
  return;
 }
 if(t.id==="lStGo"){
  const sel=$("#lStName");
  if(!sel||!sel.value)return;
  const n=edit(()=>stateApply(sel.value));
  if(editFailed())return;
  const d=pruneSel();
  rep(n?"ok":"in",n?`طُبّقت «${sel.value}» · ${n} حقلاً`
   :`«${sel.value}» مطابقةٌ للوضع الجاري`
   +(d?` · خرج ${d} عنصراً من التحديد`:""));
  refresh(false);
  return;
 }
 if(t.id==="lStDel"){
  const sel=$("#lStName");
  if(!sel||!sel.value)return;
  if(!confirm(`حذف حالة الطبقات «${sel.value}»؟`))return;
  edit(()=>stateDel(sel.value));
  if(editFailed())return;
  rep("ok",`حُذفت «${sel.value}»`);
  renderPanel("lays",1);
  return;
 }
 /* الأدوات */
 const run=t.dataset.run;
 if(run){
  import("../tools/registry.js").then(R=>{
   R.begin(run);
   const cl=$("#clIn");
   if(cl)cl.focus();
  });
  return;
 }
 /* المناطق */
 const ar=t.dataset.aref;
 if(ar){
  const a=areaById(ar);
  if(!a)return;
  const r=edit(()=>rebake(a,regionLoops()));
  if(r)rep("ok",`${a.id}: ${sqm(r.before)} → ${sqm(r.after)} م²`);
  refresh(false); return;
 }
 const as=t.dataset.astm;
 if(as){
  const a=areaById(as);
  if(!a)return;
  edit(()=>restamp(a));
  rep("in",`${a.id}: ثُبّتت البصمة — الحلقة كما هي`);
  refresh(false); return;
 }
 /* السلسلة */
 if(t.dataset.ccmp){
  import("../tools/registry.js").then(R=>R.begin("chaincmp"));
  return;
 }
 /* الأداة الصحية */
 const fs=t.dataset.fsnap;
 if(fs){
  const f=fixById(fs);
  if(!f)return;
  const r=snapToWall([f.x,f.y],3000);
  if(!r){rep("wr","لا جدار قريب");return}
  edit(()=>{f.x=r.p[0]; f.y=r.p[1]; f.rot=r.rot});
  rep("ok",`أُلصِقت بـ ${r.wall} — الإحداثيات صريحة بعدها`);
  refresh(false); return;
 }
 /* الأوامر الجماعية */
 if(t.dataset.brenum){
  const arr=(groupSel(H_sel()).col)||[];
  if(!arr.length){rep("wr","لا أعمدة محدَّدة");return}
  const r=edit(()=>renumberCols(arr,"C"));
  if(r)rep("ok",`رُقّم ${r.n} عموداً · ${r.first} … ${r.last}`);
  refresh(false); return;
 }
 if(t.dataset.bcleartxt){
  const arr=(groupSel(H_sel()).dim)||[];
  const n=edit(()=>clearDimTxt(arr));
  if(editFailed())return;
  rep(n?"ok":"in", n?`مُسح النصّ البديل من ${n} بُعد — عادت `
   +`المقاسات الحقيقية`:"لا نصّ بديل في المحدَّد");
  refresh(false); return;
 }
 if(t.dataset.bname){
  const arr=(groupSel(H_sel()).area)||[];
  if(!arr.length){rep("wr","لا مناطق محدَّدة");return}
  const p=($("#bASeq")&&$("#bASeq").value)||"";
  const r=edit(()=>nameAreasSeq(arr,p));
  if(r)rep("ok",`سُمّيت ${r.n} منطقة من الأكبر · `
   +`أوّلها «${r.first}»`);
  refresh(false); return;
 }
 if(t.dataset.bclrname){
  const arr=(groupSel(H_sel()).area)||[];
  if(!arr.length){rep("wr","لا مناطق محدَّدة");return}
  const n=edit(()=>clearAreaNames(arr));
  if(editFailed())return;
  /* الفراغُ في الحقل الجماعيّ يعني «لا تكتب»، فالمسحُ زرٌّ صريح
     — لا معنىً ثانٍ للفراغ يقع خلسة. */
  rep(n?"ok":"in",n?`مُسح اسمُ ${n} منطقة`
   :"لا أسماءَ في المحدَّد");
  refresh(false); return;
 }
 /* الحذف */
 if(t.id==="pDel"){
  const r=delSel();
  if(!r)return;
  rep("ok","حُذف "+delSay(r)
   +(r.skipped?` · تُخطّي ${r.skipped} مخفيّ أو مقفل`:""));
 }
});
/* ═══ الجدول والمحاور والملخّص ═══ */
function renderSched(box){
 const S2=schedule();
 if(!S2.rows.length){
  box.innerHTML=`<p class="hint">لا مناطق — استعمل أداة «منطقة»</p>`;
  return;
 }
 box.innerHTML=S2.rows.map(r=>
  `<div class="ro" style="display:flex;gap:6px;margin:2px 0`
  +`${r.stale?";color:var(--wr)":""}">`
  +`<span style="flex:1;overflow:hidden;text-overflow:ellipsis">`
  +`${esc(r.name)}${r.stale?" ⚠":""}</span>`
  +`<span>${sqm(r.ar)}</span></div>`).join("")
  +`<div class="ro" style="display:flex;gap:6px;margin-top:5px;`
  +`color:var(--ok)"><span style="flex:1">المجموع `
  +`(${S2.rows.length})</span><span>${sqm(S2.total)} م²</span></div>`;
}
function renderOSched(box){
 const D2=openSchedule();
 if(!D2.rows.length){
  box.innerHTML=`<p class="hint">لا فتحات — استعمل «باب» أو `
   +`«شباك»</p>`;
  return;
 }
 box.innerHTML=D2.rows.map(r=>{
  const hid=!vis(OK[r.kind].lay);
  return `<div class="ro" data-osel="${esc(r.ids.join(" "))}" `
   +`title="${esc(r.ids.join(" · "))}" `
   +`style="display:flex;gap:6px;margin:2px 0;cursor:pointer`
   +`${hid?";opacity:.45":""}">`
   +`<span style="width:26px;flex:none;color:var(--wr)">`
   +`${esc(r.mark)}</span>`
   +`<span style="flex:1;overflow:hidden;text-overflow:ellipsis;`
   +`white-space:nowrap">${esc(okName(r.kind))} `
   +`${esc(dm2(r.w,r.h))}`
   +`${r.sill?` · ج ${m2(r.sill)}`:""}`
   +`${r.pan>1?` · ${r.pan} مصاريع`:""}`
   +`${r.dep?` · ع ${m2(r.dep)}`:""}</span>`
   +`<span>${r.n}</span></div>`;
 }).join("")
 +`<div class="ro" style="display:flex;gap:6px;margin-top:5px;`
 +`color:var(--ok)"><span style="flex:1">المجموع `
 +`(${D2.rows.length} نوعاً)</span><span>${D2.total}</span></div>`
 +(D2.bad?`<p class="hint warn">${D2.bad} فتحة معطوبة داخل `
  +`الجدول — الفاحص يسمّيها</p>`:"")
 +`<p class="hint">الرمز مشتقٌّ من ترتيب الجدول: تقريرٌ لا حقلٌ `
  +`يُخزَّن على الفتحة. انقر صفاً لتحديد فتحاته.</p>`;
}
function renderSummary(box){
 const sc=scene();
 const le=looseEnds(2).length;
 const B=sc.B;
 const W=[];
 if(le)W.push(`${le} طرف غير متّصل`);
 if(sc.bad)W.push(`${sc.bad} فتحة معطوبة`);
 if(sc.stale)W.push(`${sc.stale} منطقة قديمة`);
 if(sc.loose)W.push(`${sc.loose} بُعداً معلَّقاً`);
 if(sc.over)W.push(`${sc.over} بُعداً بنصّ بديل`);
 if(sc.hidden)W.push(`${sc.hidden} كياناً مخفيّاً`);
 box.innerHTML=
  `${S.walls.length} جدار · ${S.opens.length} فتحة · `
  +`${S.cols.length} عمود · ${S.fixt.length} أداة · `
  +`${S.stairs.length} درج<br>`
  +`${S.areas.length} منطقة · ${S.dims.length} بُعد · `
  +`${S.chains.length} سلسلة · ${S.anno.length} تأشير`
  +(B?`<br>المدى ${esc(dm2(B.x1-B.x0,B.y1-B.y0,"م"))}`:"")
  +(W.length
   ?`<br><span class="warn">${W.join(" · ")}</span>`
   :"<br>لا ملاحظات");
}
export function syncToggles(){
 [...document.querySelectorAll("[data-rb]")].forEach(b=>
  b.classList.toggle("on",!!+S.rb[b.dataset.rb]));
 const u=$("#bUndo"), r=$("#bRedo");
 if(u)u.disabled=!canUndo();
 if(r)r.disabled=!canRedo();
 try{syncStatus(); syncOverlay()}catch(e){}
}
function renderAxes(box){
 box.innerHTML=`${S.grid.xs.length} محوراً رأسياً (حروف) · `
  +`${S.grid.ys.length} أفقياً (أرقام)`;
}
export function refresh(reload){
 if(reload)loadForms();
 /* حرسٌ في موضعٍ واحد: أي تعديلٍ للطبقات أو حذفٍ أو تراجعٍ قد
    يُبقي في التحديد ما لا يُحدَّد. التنظيف هنا يُغني عن نداءٍ في
    كل مسار — وهو صامتٌ لأن مساره الخاص يُبلِّغ بعدده. */
 pruneSel();
 syncToggles();
 markAllDirty();
 renderVisible(1);      /* المرئيّ المتّسخ · والمغلق يبقى موسوماً */
 draw();
}
```

<a id="f-js-ui-quickprops-js"></a>

---

## `js/ui/quickprops.js`

```javascript
/* ═══ الخصائص السريعة ═══
   بطاقةٌ صغيرة قرب المحدَّد تحمل أهمّ حقوله. لا مُثبِّتَ فيها ولا
   قراءةَ خاصّة: تبثّ سمتي data-bk= وdata-bf= نفسيهما اللتين
   يبثّهما التعديل الجماعي (عبر fldCtl في props.js)، فيتولّاها
   معالج props.js بمُثبِّتات batch.js ورسائل رفضها وخطوة تراجعها —
   والحقل المختلف يظهر «متعدّداً» ولا يُكتب من تلقائه، كما في
   اللوحة الكاملة تماماً.

   وتُسجَّل في panels.js فتُرسَم مع المرئيّ وتُوسَم متّسخةً كغيرها. */
import {FLD,fldOf,readField,groupSel} from "../core/batch.js";
import {NAME} from "../core/ents.js";
import {mnum} from "../core/units.js";
import {UIS,uiSet,saveUI} from "./store.js";
import {icon} from "./icons.js";
import {reg,renderPanel,markDirty} from "./panels.js";
import {selList,shapeOfSel,W2S,V} from "./canvas.js";
import {HOOK} from "./bus.js";
/* ═══ المتحكِّمُ من المولِّد الواحد ═══
   والغلافُ .qf يبقى هيئتَها: المُوحَّدُ عقدُ الحقل — اسمُ السمة
   وقائمةُ العناصر والحدُّ وحالُ التعدّد — لا صفُّ العرض. */
import {fldCtl} from "./props.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||14):"";
const cl=(v,a,b)=>v<a?a:(v>b?b:v);

/* أهمّ حقولٍ لكل نوع — والباقي في اللوحة الكاملة */
export const QF={
 wall:["t","align","type"],
 open:["kind","w","h","sill"],
 col:["kind","w","h","type"],
 fix:["kind","w","d","mir"],
 stair:["w","n","up"],
 area:["name","fill","showArea"],
 dim:["kind","txt"],
 chain:["total"],
 anno:["hm","al","pre"]
};
export const qpOn=()=>!!UIS.qp;

function ctl(kind,f,r){
 const C={kind, val:r.mixed?null:r.value, mixed:!!r.mixed,
  own:r.own};
 const c=fldCtl(f,C.val,C);
 if(f.t==="chk")
  return `<label class="chk">${c} ${esc(f.n)}</label>`;
 return `<span class="qf"><label>${esc(f.n)}</label>${c}</span>`;
}
export function renderQuick(box){
 const L=selList();
 if(!L.length){box.innerHTML=""; return}
 const G=groupSel(L);
 const KS=Object.keys(G).filter(k=>G[k].length);
 if(KS.length!==1){
  box.innerHTML=`<p class="hint">${L.length} عنصر من `
   +`${KS.length||0} نوعاً — افتح «الخصائص» للتعديل الجماعي</p>`;
  return;
 }
 const kind=KS[0], arr=G[kind];
 const keys=QF[kind]||[];
 const rows=[];
 keys.forEach(k=>{
  const f=(FLD[kind]||[]).find(x=>x.k===k);
  if(!f)return;
  const r=readField(kind,arr,k);
  if(!r.own)return;
  rows.push(ctl(kind,f,r));
 });
 box.innerHTML=rows.length?rows.join("")
  :`<p class="hint">لا حقولَ سريعة لهذا النوع</p>`;
 box.querySelectorAll('[data-mix="1"]')
  .forEach(el=>{el.indeterminate=true});
}
function head(){
 const h=$("#qpHead");
 if(!h)return;
 const L=selList();
 if(!L.length){h.textContent=""; return}
 const G=groupSel(L);
 const KS=Object.keys(G).filter(k=>G[k].length);
 h.textContent=(L.length===1)
  ? `${L[0].id} ${NAME[L[0].k]||L[0].k}`
  : `${L.length} ${KS.length===1?(NAME[KS[0]]||KS[0]):"عنصر"}`;
}
/* الموضع: قرب المحدَّد عند كل تحديدٍ جديد، إلّا إن سحبتَها فتثبت */
function place(){
 const card=$("#qpCard");
 if(!card)return;
 if(UIS.qpFree){
  card.style.insetInlineStart=(+UIS.qpX||16)+"px";
  card.style.insetBlockStart=(+UIS.qpY||16)+"px";
  return;
 }
 const L=selList();
 let p=null;
 if(L.length){
  const sh=shapeOfSel(L[L.length-1]);
  if(sh)p=W2S(sh[0],sh[1]);
 }
 const w=card.offsetWidth||236, h=card.offsetHeight||120;
 const x=p?cl(p[0]+22,4,Math.max(4,V.w-w-4)):8;
 const y=p?cl(p[1]+18,4,Math.max(4,V.h-h-4)):8;
 card.style.insetInlineStart=Math.round(x)+"px";
 card.style.insetBlockStart=Math.round(y)+"px";
}
export function quickSel(){
 const card=$("#qpCard");
 if(!card)return;
 const L=selList();
 const show=qpOn()&&L.length>0&&!UIS.clean;
 card.hidden=!show;
 if(!show)return;
 head();
 markDirty("quick");
 renderPanel("quick");
 place();
}
export function qpToggle(){
 uiSet("qp",UIS.qp?0:1);
 quickSel();
 HOOK.report("in",`الخصائص السريعة: ${UIS.qp?"ظاهرة":"مخفيّة"}`);
 HOOK.toggles();
 return UIS.qp;
}
let DRG=null;
export function wireQuick(){
 const card=$("#qpCard");
 if(!card)return false;
 reg("quick","#qpBody",renderQuick,"خصائص سريعة");
 card.addEventListener("click",e=>{
  if(e.target.closest("#qpClose")){qpToggle(); return}
 });
 card.addEventListener("keydown",e=>{
  if(e.key==="Escape"){e.target.blur(); return}
  e.stopPropagation();
 });
 /* السحب من الترويسة — وأوّل سحبةٍ تفكّ التتبّع فتثبت البطاقة */
 card.addEventListener("mousedown",e=>{
  if(UIS.lockUI)return;
  if(!e.target.closest(".qpH")||e.target.closest("button"))return;
  const r=card.getBoundingClientRect();
  const st=$("#stage").getBoundingClientRect();
  DRG={px:e.clientX,py:e.clientY,
   x:r.left-st.left,y:r.top-st.top,on:0,
   rtl:getComputedStyle(document.documentElement).direction==="rtl"};
  document.documentElement.classList.add("dragging");
 });
 addEventListener("mousemove",e=>{
  if(!DRG)return;
  const dx=e.clientX-DRG.px, dy=e.clientY-DRG.py;
  if(!DRG.on&&Math.hypot(dx,dy)<4)return;
  DRG.on=1;
  const card2=$("#qpCard");
  const w=card2.offsetWidth||236, h=card2.offsetHeight||120;
  const ix=DRG.rtl?(V.w-DRG.x-w):DRG.x;
  UIS.qpX=Math.round(cl(ix+(DRG.rtl?-dx:dx),0,Math.max(0,V.w-w)));
  UIS.qpY=Math.round(cl(DRG.y+dy,0,Math.max(0,V.h-h)));
  UIS.qpFree=1;
  place();
 });
 addEventListener("mouseup",()=>{
  if(!DRG)return;
  const on=DRG.on;
  DRG=null;
  document.documentElement.classList.remove("dragging");
  if(on)saveUI();
 });
 return true;
}
```

<a id="f-js-ui-recoverdialog-js"></a>

---

## `js/ui/recoverdialog.js`

```javascript
/* ═══ حوار استعادة ما بعد التعطّل (لوحدة persist.js الاختيارية) ═══
   يُعرَض عند الإقلاع إن وُجدت لقطةٌ محفوظة. يرث theme.css.
   ⚠ غير مربوطٍ تلقائياً بـ app.js — انظر ملاحظة التكامل في
   core/persist.js قبل استخدامه، فمِسطَر لديه استعادة جلسةٍ فعليّة
   مبنيّة أصلاً في core/state.js. */
export function maybeOfferRecovery(autosave){
 return autosave.peek().then(info=>{
  if(!info) return false;
  return new Promise(res=>{
   const ov=document.createElement("div");
   ov.className="rc-overlay"; ov.dir="rtl";
   const when=new Date(info.ts).toLocaleString("ar");
   ov.innerHTML=`
    <div class="rc-box" role="dialog" aria-modal="true"
      aria-label="استعادة عمل سابق">
      <h3 class="rc-title">وُجد عملٌ غير محفوظ</h3>
      <p class="rc-msg">هناك نسخةٌ محفوظة تلقائياً من ${when}.
        هل تريد استعادتها؟</p>
      <div class="rc-btns">
        <button class="rc-yes">استعادة</button>
        <button class="rc-no">تجاهل</button>
      </div>
    </div>`;
   document.body.appendChild(ov);
   const done=(v)=>{ ov.remove(); res(v); };
   ov.querySelector(".rc-yes").addEventListener("click",async()=>{
    await autosave.recover(); done(true); });
   ov.querySelector(".rc-no").addEventListener("click",()=>{
    autosave.clear(); done(false); });
  });
 });
}
```

<a id="f-js-ui-ribbon-render-js"></a>

---

## `js/ui/ribbon/render.js`

```javascript
/* ═══ مصيّر الشريط ═══
   يُبنى مرّةً واحدة، ولا يُهدَم إلّا عند تبديل القشرة أو ظهور
   تبويبٍ سياقيّ. والمزامنة تقارن ولا تبني — لأن syncPrompt تُنادى
   مع كل حركة مؤشّر، وهي العلّة التي أصلحتها و٠ في شريط الخيارات
   فلا تُعاد هنا.

   الحالة كلّها في السمات (data-*) لا في متغيّراتٍ موازية، فلا
   يفترق المرسوم عن المخزَّن. */
import {RIBBON,CTX,QAT} from "./schema.js";
import {icon} from "../icons.js";
import {UIS} from "../store.js";
import * as R from "../../tools/registry.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s):"";
const AR="٠١٢٣٤٥٦٧٨٩";
export const arNum=s=>String(s).replace(/\d/g,d=>AR[+d]);

/* هويّة العنصر: أمرٌ أو فعلٌ أو مفتاح — واحدةٌ لا تلتبس */
const idOf=it=>it.cmd!=null?`c:${it.cmd}`
 :(it.act?`a:${it.act}`:(it.tog?`t:${it.tog}`:""));
const attrOf=it=>it.cmd!=null?`data-cmd="${esc(it.cmd)}"`
 :(it.act?`data-act="${esc(it.act)}"`
 :(it.tog?`data-tog="${esc(it.tog)}"`:""));

function btn(it,big){
 const cls=big?"rbBig":"rbSm";
 const sz=big?24:16;
 const g=ic(it.ico,sz);
 return `<button type="button" class="${cls}" ${attrOf(it)} `
  +`data-key="${esc(idOf(it))}" title="${esc(it.n)}">`
  +(g||`<span class="noic" aria-hidden="true"></span>`)
  +`<span class="lb">${esc(it.n)}</span></button>`;
}
const itemHtml=it=>it.group
 ? `<div class="rbCol">${it.group.map(x=>btn(x,0)).join("")}</div>`
 : btn(it,!!it.big);

function panelHtml(tid,p){
 return `<section class="rbp" data-panel="${esc(tid)}/${esc(p.id)}" `
  +`role="group" aria-label="${esc(p.n)}">`
  +`<div class="rbpBody">${(p.items||[]).map(itemHtml).join("")}</div>`
  +`<div class="rbpFoot"><span>${esc(p.n)}</span>`
  +(p.dlg?`<button type="button" class="rbDlg" `
    +`data-act="dlg:${esc(p.dlg)}" title="افتح اللوحة" `
    +`aria-label="افتح لوحة ${esc(p.n)}">◥</button>`:"")
  +`</div></section>`;
}
function tabBtn(t,ctx){
 return `<button type="button" class="rbTab${ctx?" ctx":""}" `
  +`role="tab" id="rbT-${esc(t.id)}" aria-controls="rbP-${esc(t.id)}" `
  +`aria-selected="false" tabindex="-1" data-tab="${esc(t.id)}">`
  +`<span class="tn">${esc(t.n)}</span>`
  +(t.kt?`<span class="kt" hidden>${arNum(t.kt)}</span>`:"")
  +`</button>`;
}
const paneHtml=t=>`<div class="rbPane" id="rbP-${esc(t.id)}" `
 +`role="tabpanel" aria-labelledby="rbT-${esc(t.id)}" hidden>`
 +(t.panels||[]).map(p=>panelHtml(t.id,p)).join("")+`</div>`;

/* التبويب السياقي واحدٌ يتبدّل محتواه — لا تسعةٌ تُبنى وتُخفى */
let ctxKind=null;

export function buildRibbon(){
 const rb=$("#ribbon");
 if(!rb)return 0;
 rb.innerHTML=`
<div id="rbTabs" role="tablist" aria-label="شريط الأوامر">
 ${RIBBON.map(t=>tabBtn(t)).join("")}
 <span class="gap"></span>
 <button type="button" id="rbToggle" class="rbTgl"
  title="اطوِ الشريط · Ctrl+F1" aria-label="اطوِ الشريط">▲</button>
</div>
<div id="rbPanes">${RIBBON.map(paneHtml).join("")}</div>`;
 ctxKind=null;
 setTab(RIBBON.some(t=>t.id===UIS.tab)?UIS.tab:"home",1);
 return RIBBON.length;
}
export function buildQAT(){
 const q=$("#qat");
 if(!q)return 0;
 q.innerHTML=QAT.map(it=>it.sep?`<span class="dv"></span>`
  :`<button type="button" data-act="${esc(it.act)}" `
   +`data-key="a:${esc(it.act)}" title="${esc(it.n)}" `
   +`aria-label="${esc(it.n)}">${ic(it.ico,16)}</button>`).join("");
 return QAT.filter(x=>!x.sep).length;
}
/* ═══ التبويبات ═══ */
export function setTab(id,quiet){
 const rb=$("#ribbon");
 if(!rb)return false;
 const tabs=[...rb.querySelectorAll(".rbTab")];
 if(!tabs.some(b=>b.dataset.tab===id))return false;
 tabs.forEach(b=>{
  const on=b.dataset.tab===id;
  b.setAttribute("aria-selected",on?"true":"false");
  b.tabIndex=on?0:-1;
  b.classList.toggle("on",on);
 });
 [...rb.querySelectorAll(".rbPane")].forEach(p=>{
  p.hidden=(p.id!=="rbP-"+id);
 });
 if(!quiet&&!id.startsWith("ctx"))UIS.tab=id;
 return true;
}
export const curTab=()=>{
 const b=document.querySelector("#ribbon .rbTab.on");
 return b?b.dataset.tab:null;
};
/* ═══ التبويب السياقي ═══
   kind=null يزيله. والعودة إلى تبويب المستخدم عند الزوال — لا
   يبقى الشريط على لوحٍ فارغ. */
export function setCtx(kind){
 const rb=$("#ribbon");
 if(!rb)return;
 if(kind===ctxKind)return;
 const old=rb.querySelector('[data-tab="ctx"]');
 const oldPane=document.getElementById("rbP-"+"ctx");
 if(!kind||!CTX[kind]){
  ctxKind=null;
  if(old){
   const wasOn=old.classList.contains("on");
   old.remove();
   if(oldPane)oldPane.remove();
   if(wasOn)setTab(RIBBON.some(t=>t.id===UIS.tab)?UIS.tab:"home",1);
  }
  return;
 }
 const d=CTX[kind];
 const t={id:"ctx",n:d.n,panels:d.panels};
 const wasOn=!!(old&&old.classList.contains("on"));
 if(old)old.remove();
 if(oldPane)oldPane.remove();
 $("#rbTabs").insertAdjacentHTML("afterbegin",tabBtn(t,1));
 $("#rbPanes").insertAdjacentHTML("beforeend",paneHtml(t));
 ctxKind=kind;
 /* الانتقال التلقائي مرّةً عند أول تحديد فقط — لا يُقفز
    بالمستخدم كلّما بدّل نوع المحدَّد */
 if(wasOn||UIS.ctxAuto)setTab("ctx",1);
}
export const ctxOf=()=>ctxKind;

/* ═══ الطيّ ═══ */
export function setMin(v){
 const rb=$("#ribbon");
 if(!rb)return;
 const on=v?1:0;
 rb.classList.toggle("min",!!on);
 UIS.ribbonMin=on;
 const t=$("#rbToggle");
 if(t){
  t.textContent=on?"▼":"▲";
  t.title=(on?"افتح الشريط":"اطوِ الشريط")+" · Ctrl+F1";
 }
}
/* ═══ المزامنة ═══ رخيصةٌ بقصد: تُنادى مع كل حركة مؤشّر ═══ */
let lastCur=null, BTN=null;
export function invalidateSync(){BTN=null; lastCur=null}

export function syncRibbon(){
 if(UIS.shell!=="ribbon")return;
 if(!BTN)BTN=[...document.querySelectorAll("#ribbon [data-cmd]")];
 const cur=R.active()?R.T.def.id:"";
 if(cur===lastCur)return;
 lastCur=cur;
 BTN.forEach(b=>b.classList.toggle("on",b.dataset.cmd===cur));
}
/* حالة المفاتيح تُقرأ من هدفها — مصدرٌ واحد لا نسخةٌ ثانية */
export function syncRibbonTogs(){
 if(UIS.shell!=="ribbon")return;
 document.querySelectorAll("#ribbon [data-tog]").forEach(b=>{
  const t=document.querySelector(b.dataset.tog);
  if(!t){b.disabled=true; return}
  const on=(t.type==="checkbox")?t.checked:t.classList.contains("on");
  b.classList.toggle("on",!!on);
 });
 ["undo","redo"].forEach(k=>{
  const src=document.querySelector("#b"+k[0].toUpperCase()+k.slice(1));
  if(!src)return;
  document.querySelectorAll(`[data-act="${k}"]`)
   .forEach(b=>{b.disabled=src.disabled});
 });
}
/* ═══ KeyTips ═══ */
export function showKT(on){
 document.querySelectorAll("#ribbon .kt, #appBtn .kt")
  .forEach(e=>{e.hidden=!on});
 document.documentElement.classList.toggle("kt",!!on);
}
```

<a id="f-js-ui-ribbon-schema-js"></a>

---

## `js/ui/ribbon/schema.js`

```javascript
/* ═══ مخطّط الشريط ═══
   وصفٌ لا كود: كل تبويبٍ ولوحٍ وزرّ سطرٌ في هذا الملفّ، فتحريك
   أمرٍ لا يمسّ المصيّر ولا الموصِّل.

   قواعد التنظيم:
     • كل لوحٍ يُقاد بزرِّه الأشيع كبيراً، ثم أعمدةٌ صغيرة.
     • العمود ثلاثة أزرارٍ على الأكثر، ولا عمودَ بزرٍّ وحيد —
       الزرّ المفرد يُكتَب مستقلاً لا داخل {group}.
     • ما للوحِ لوحةٌ جانبية فله مُفتتِح (dlg) في ذيله.
     • ترتيب اللوحات ترتيبُ عملٍ: ارسم ثم عدّل ثم حرّر.

   ثلاثة أنواعٍ من العناصر، ولا رابع:
     {cmd}  أداة مسجَّلة — تُنادى بـ R.begin مباشرةً
     {act}  نقرٌ بالوكالة على زرٍّ قائم في اللوحة الجانبية
     {tog}  مفتاح حالةٍ يعكس حالة هدفه ويُبدّلها بنقره
   {group:[…]} عمودٌ صغير · big:1 زرٌّ كبير بأيقونته فوق تسميته.

   قاعدة الموضع الواحد تسري على الأوامر، ويُستثنى تبويب «رئيسي»
   صريحاً: هو سطح تسريعٍ لأشيع الأدوات، وموضعها الأساسي تبويبها. */

const G=(...items)=>({group:items});

/* ═══ التبويبات الثابتة ═══ */
export const RIBBON=[

/* ── رئيسي: سطح التسريع ── */
{id:"home", n:"رئيسي", kt:"1", panels:[

 {id:"draw", n:"رسم", dlg:"proj", items:[
  {cmd:"wall", n:"جدار",   ico:"wall", big:1},
  {cmd:"rect", n:"مستطيل", ico:"rect", big:1},
  G({cmd:"col",     n:"عمود",         ico:"col"},
    {cmd:"gridcols",n:"أعمدة المحاور",ico:"gridcols"},
    {cmd:"area",    n:"منطقة",        ico:"area"})]},

 {id:"mod", n:"تعديل", items:[       /* التحويلات: موضعٌ ودوران */
  {cmd:"move", n:"نقل", ico:"move", big:1},
  {cmd:"copy", n:"نسخ", ico:"copy", big:1},
  G({cmd:"rotate",n:"دوران",ico:"rotate"},
    {cmd:"mirror",n:"مرآة", ico:"mirror"},
    {cmd:"offset",n:"إزاحة",ico:"offset"}),
  G({cmd:"chamfer",   n:"كسر الركن",    ico:"chamfer"},
    {cmd:"array",     n:"مصفوفة",       ico:"array"},
    {cmd:"arraypolar",n:"مصفوفة قطبية", ico:"arraypol"})]},

 {id:"edit", n:"تحرير", items:[      /* الحواف: قصٌّ ووصل */
  {cmd:"trim", n:"قصّ", ico:"trim", big:1},
  G({cmd:"extend", n:"تمديد",ico:"extend"},
    {cmd:"stretch",n:"شدّ",   ico:"stretch"},
    {cmd:"break",  n:"قطع",   ico:"brk"}),
  G({cmd:"divide",n:"قسمة", ico:"divide"},
    {cmd:"weld",  n:"لحم",   ico:"weld"}),
  G({cmd:"match",n:"مطابقة",        ico:"match"},
    {cmd:"sel",  n:"تحديد بالمعرّف", ico:"selid"},
    {cmd:"del",  n:"حذف",            ico:"del"})]},

 {id:"ann", n:"تأشير", dlg:"props", items:[
  {cmd:"dim",  n:"بُعد", ico:"dim",  big:1},
  {cmd:"text", n:"نصّ",  ico:"text", big:1},
  G({cmd:"chain",n:"سلسلة",ico:"chain"},
    {cmd:"lead", n:"قائد",  ico:"lead"},
    {cmd:"level",n:"منسوب", ico:"level"})]},

 {id:"lay", n:"الطبقات", dlg:"lays", items:[
  {act:"laysDlg", n:"مدير الطبقات", ico:"layers", big:1},
  G({act:"lAll",   n:"أظهر الكل",  ico:"grid"},
    {act:"lUnlock",n:"افتح المقفل",ico:"unlock"})]},

 {id:"tool", n:"أدوات", dlg:"insp", items:[
  {cmd:"measure",n:"قياس", ico:"measure",big:1},
  {act:"inspect",n:"افحص", ico:"inspect",big:1},
  G({act:"undo",n:"تراجع",  ico:"undo"},
    {act:"redo",n:"إعادة",  ico:"redo"},
    {act:"fit", n:"ملاءمة", ico:"fit"})]}]},

/* ── معماري ── */
{id:"arch", n:"معماري", kt:"2", panels:[
 {id:"walls", n:"جدران", items:[
  {cmd:"wall",  n:"جدار",   ico:"wall",  big:1},
  {cmd:"rect",  n:"مستطيل", ico:"rect",  big:1},
  G({cmd:"offset", n:"إزاحة",      ico:"offset"},
    {cmd:"chamfer",  n:"كسر الركن",ico:"chamfer"}),
  G({cmd:"divide", n:"قسمة", ico:"divide"},
    {cmd:"weld",     n:"لحم",   ico:"weld"})]},

 {id:"opens", n:"فتحات", items:[
  {cmd:"door", n:"باب",  ico:"door",  big:1},
  {cmd:"win",  n:"شباك", ico:"window",big:1},
  G({cmd:"fixed",  n:"شباك ثابت", ico:"fixed"},
    {cmd:"opening",n:"فتحة صافية", ico:"opening"}),
  G({cmd:"arch", n:"مقنطرة", ico:"arch"},
    {cmd:"niche", n:"كوّة",    ico:"niche"})]},

 {id:"parts", n:"أجزاء", items:[
  {cmd:"stair",   n:"درج",           ico:"stair",   big:1},
  {cmd:"col",     n:"عمود",          ico:"col",     big:1},
  {cmd:"gridcols",n:"أعمدة المحاور", ico:"gridcols",big:1}]},

 {id:"fixt", n:"صحية", items:[
  G({cmd:"wc",   n:"كرسي",  ico:"wc"},
    {cmd:"lav",  n:"مغسلة", ico:"lav"},
    {cmd:"bidet",n:"شطّاف", ico:"bidet"}),
  G({cmd:"shower",n:"دُش",   ico:"shower"},
    {cmd:"tub",   n:"بانيو", ico:"tub"},
    {cmd:"fd",    n:"صفاية", ico:"fd"}),
  G({cmd:"sink",n:"مجلى",   ico:"sink"},
    {cmd:"wm",   n:"غسّالة", ico:"wm"},
    {cmd:"ur",   n:"مبولة",  ico:"ur"})]},

 {id:"areas", n:"مناطق", dlg:"sched", items:[
  {cmd:"area",   n:"منطقة", ico:"area",   big:1},
  {cmd:"arearef",n:"تحديث", ico:"arearef",big:1},
  G({act:"asCsv",   n:"جدول المساحات",ico:"csv"},
    {act:"schedDlg",n:"اللوحة",       ico:"table"})]}]},

/* ── إدراج ── */
{id:"ins", n:"إدراج", kt:"3", panels:[
 {id:"ref", n:"المرجع", dlg:"ref", items:[
  {act:"rImp", n:"استورد DXF",ico:"ref",big:1},
  G({cmd:"refalign",n:"محاذاة", ico:"refalign"},
    {cmd:"refcal",  n:"معايرة", ico:"refcal"},
    {cmd:"refmove", n:"نقل",    ico:"refmove"}),
  G({act:"rRst",n:"صفّر التحويل",ico:"undo"},
    {act:"rClr",n:"أزِل المرجع",  ico:"del"})]},

 {id:"data", n:"بيانات", items:[
  {act:"schedDlg", n:"جدول المساحات",ico:"table",big:1},
  {act:"oschedDlg",n:"جدول الفتحات", ico:"table",big:1},
  G({act:"asCsv",n:"مساحات CSV",ico:"csv"},
    {act:"osCsv",n:"فتحات CSV",  ico:"csv"})]},

 {id:"axg", n:"المحاور", dlg:"axes", items:[
  {cmd:"axis", n:"محور",ico:"axis",big:1},
  G({cmd:"gridcols",n:"أعمدة المحاور",ico:"gridcols"},
    {act:"axClr",    n:"امسح المحاور", ico:"del"})]}]},

/* ── تأشير ── */
{id:"annt", n:"تأشير", kt:"4", panels:[
 {id:"dims", n:"أبعاد", items:[
  {cmd:"dim",  n:"بُعد",  ico:"dim",  big:1},
  {cmd:"chain",n:"سلسلة", ico:"chain",big:1},
  G({cmd:"chaincmp",n:"قارن السلسلة",ico:"chaincmp"},
    {cmd:"measure",  n:"قياس",         ico:"measure"})]},

 {id:"txt", n:"نصوص", items:[
  {cmd:"text", n:"نصّ",  ico:"text", big:1},
  {cmd:"lead", n:"قائد", ico:"lead", big:1},
  G({cmd:"level",n:"منسوب",ico:"level"},
    {cmd:"axis", n:"محور",  ico:"axis"})]},

 {id:"sty", n:"الهيئة", dlg:"proj", items:[
  {act:"projDlg",n:"إعداد المشروع",    ico:"props",big:1},
  {act:"defsDlg",n:"افتراضات الأدوات", ico:"shell",big:1}]}]},

/* ── عرض ── */
{id:"view", n:"عرض", kt:"5", panels:[
 {id:"nav", n:"تنقّل", items:[
  {act:"fit",  n:"ملاءمة",     ico:"fit",  big:1},
  {act:"clean",n:"شاشة نظيفة", ico:"clean",big:1},
  G({act:"rbMin",n:"اطوِ الشريط",ico:"rbmin"},
    {act:"theme",n:"السِّمة",     ico:"theme"},
    {act:"shell",n:"القشرة",     ico:"shell"})]},

 {id:"aids", n:"مساعدات", items:[
  G({tog:'[data-rb="ortho"]',n:"تعامد", ico:"ortho"},
    {tog:'[data-rb="polar"]',n:"قطبي",  ico:"polar"},
    {tog:'[data-rb="snap"]', n:"التقاط",ico:"osnap"}),
  G({tog:'[data-rb="grips"]',n:"مقابض", ico:"grips"},
    {tog:'[data-rb="ends"]', n:"أطراف", ico:"ends"},
    {act:"osPop",            n:"أنماط الالتقاط",ico:"osnap"})]},

 {id:"inp", n:"الإدخال", items:[
  G({tog:'[data-rb="dyn"]',n:"إدخال حركي",ico:"dyn"},
    {act:"qpTog", n:"خصائص سريعة",ico:"qp"},
    {act:"rclick",n:"الزرّ الأيمن", ico:"ctx"}),
  G({act:"cmdMode",  n:"موضع سطر الأوامر",ico:"cmdl"},
    {act:"cmdFloat", n:"سطر أوامر عائم",  ico:"float"},
    {act:"cmdBottom",n:"أعِده إلى الأسفل",ico:"down"})]},

 {id:"disp", n:"عرض الأجسام", dlg:"proj", items:[
  G({tog:"#oJoins",n:"دمج الأركان",    ico:"weld"},
    {tog:"#oSolo", n:"أعمدة مستقلّة",  ico:"col"},
    {act:"projDlg",n:"تعبئة الجدران …", ico:"props"})]},

 {id:"pan", n:"لوحات", items:[
  G({act:"propsDlg",n:"الخصائص",ico:"panel"},
    {act:"laysDlg", n:"الطبقات", ico:"layers"},
    {act:"inspDlg", n:"الفاحص",  ico:"inspect"}),
  G({act:"refDlg",  n:"المرجع", ico:"ref"},
    {act:"sheetDlg", n:"الورقة", ico:"sheet"},
    {act:"stateDlg", n:"الحالة",  ico:"grid"}),
  G({act:"schedDlg", n:"مساحات", ico:"table"},
    {act:"oschedDlg", n:"فتحات",  ico:"table"}),
  G({act:"defsDlg",n:"الافتراضات",ico:"shell"},
    {act:"aiDlg",   n:"المساعد",   ico:"ai"})]},

 {id:"dock", n:"الإرساء", items:[
  {act:"wsMenu",n:"أسطح العمل",ico:"ws",big:1},
  G({act:"wsArch", n:"معماري",ico:"wall"},
    {act:"wsAnnot",n:"تأشير",  ico:"dim"},
    {act:"wsOut",  n:"إخراج",  ico:"xport"}),
  G({act:"dockAutoS",n:"إخفاء العمود الأيمن",ico:"autohide"},
    {act:"dockTabS", n:"تبويبات الأيمن",     ico:"tabs"}),
  G({act:"dockAutoE",n:"إخفاء العمود الأيسر",ico:"autohide"},
    {act:"dockTabE", n:"تبويبات الأيسر",     ico:"tabs"})]}]},

/* ── إدارة ── */
{id:"mng", n:"إدارة", kt:"6", panels:[
 {id:"laym", n:"الطبقات", dlg:"lays", items:[
  {act:"laysDlg",n:"مدير الطبقات",ico:"layers",big:1},
  G({act:"lAll",   n:"أظهر الكل",  ico:"grid"},
    {act:"lUnlock",n:"افتح المقفل",ico:"unlock"})]},

 {id:"chk", n:"التدقيق", dlg:"insp", items:[
  {act:"inspect",n:"افحص",ico:"inspect",big:1}]},

 {id:"prj", n:"المشروع", dlg:"defs", items:[
  {act:"fnew",    n:"مشروع جديد",      ico:"fnew", big:1},
  {act:"defsDlg", n:"افتراضات الأدوات", ico:"shell",big:1},
  G({act:"dRst",n:"أعِد المصنع",ico:"undo"},
    {act:"undo", n:"تراجع",      ico:"undo"},
    {act:"redo", n:"إعادة",      ico:"redo"})]}]},

/* ── إخراج ── */
{id:"out", n:"إخراج", kt:"7", panels:[
 {id:"sht", n:"الورقة", dlg:"sheet", items:[
  {act:"sheetDlg", n:"الورقة والعنوان",ico:"sheet",big:1},
  {act:"shCenter", n:"تمركز على الرسم", ico:"fit"}]},

 {id:"exp", n:"التصدير", dlg:"export", items:[
  {act:"xDxf",n:"DXF",ico:"dxf",big:1},
  {act:"xSvg",n:"SVG",ico:"svg",big:1},
  G({act:"xPng",n:"PNG",ico:"png"},
    {act:"xPdf",n:"PDF",ico:"pdf"}),
  G({act:"asCsv",n:"مساحات CSV",ico:"csv"},
    {act:"osCsv",n:"فتحات CSV",  ico:"csv"})]},

 {id:"fil", n:"الملفّ", items:[
  {act:"xSave",n:"احفظ",ico:"save",big:1},
  {act:"xOpen",n:"افتح", ico:"open",big:1},
  {act:"fnew", n:"جديد",  ico:"fnew",big:1}]}]}
];

/* ═══ التبويبات السياقية ═══
   تظهر عند تحديدٍ متجانس، وتزول بزواله. أوامرها هي أوامر النوع
   لا أكثر — فلا يظهر «إزاحة» عند تحديد بُعد. */
const CTX_COMMON=(kind)=>({id:"ctx-act", n:"المحدَّد", items:[
 {act:"propsDlg",n:"الخصائص",ico:"props",big:1},
 {act:"delSel",  n:"حذف",    ico:"del",  big:1}]});

export const CTX={
 wall:{n:"جدار", panels:[
  {id:"cw1", n:"تعديل الجدار", items:[
   {cmd:"offset",n:"إزاحة",ico:"offset",big:1},
   {cmd:"weld",  n:"لحم",  ico:"weld",  big:1},
   {cmd:"chamfer",n:"كسر الركن",ico:"chamfer",big:1},
   G({cmd:"break", n:"قطع",  ico:"brk"},
     {cmd:"divide",n:"قسمة", ico:"divide"},
     {cmd:"trim",  n:"قصّ",   ico:"trim"}),
   G({cmd:"extend", n:"تمديد",ico:"extend"},
     {cmd:"stretch",n:"شدّ",   ico:"stretch"},
     {cmd:"match",  n:"مطابقة",ico:"match"})]},
  {id:"cw2", n:"فتحات عليه", items:[
   G({cmd:"door",n:"باب", ico:"door"},
     {cmd:"win", n:"شباك",ico:"window"},
     {cmd:"niche",n:"كوّة",ico:"niche"})]},
  CTX_COMMON("wall")]},

 open:{n:"فتحة", panels:[
  {id:"co1", n:"الفتحة", items:[
   {cmd:"match",n:"مطابقة",ico:"match",big:1},
   G({act:"oschedDlg",n:"جدول الفتحات",ico:"table"},
     {act:"osCsv",    n:"CSV",          ico:"csv"})]},
  CTX_COMMON("open")]},

 area:{n:"منطقة", panels:[
  {id:"ca1", n:"المنطقة", items:[
   {cmd:"arearef",n:"تحديث",ico:"arearef",big:1},
   G({act:"nameSeq", n:"سمِّ بالتسلسل",  ico:"renum"},
     {act:"schedDlg",n:"جدول المساحات",ico:"table"},
     {act:"asCsv",   n:"CSV",           ico:"csv"})]},
  CTX_COMMON("area")]},

 dim:{n:"بُعد", panels:[
  {id:"cd1", n:"البُعد", items:[
   {cmd:"match",     n:"مطابقة",           ico:"match",big:1},
   {act:"clrDimTxt", n:"امسح النصّ البديل", ico:"del"}]},
  CTX_COMMON("dim")]},

 chain:{n:"سلسلة", panels:[
  {id:"cc1", n:"السلسلة", items:[
   {cmd:"chaincmp",n:"قارن بالهندسة",ico:"chaincmp",big:1}]},
  CTX_COMMON("chain")]},

 anno:{n:"تأشير", panels:[
  {id:"ct1", n:"التأشير", items:[
   {cmd:"match",n:"مطابقة",ico:"match",big:1}]},
  CTX_COMMON("anno")]},

 col:{n:"عمود", panels:[
  {id:"ck1", n:"العمود", items:[
   {cmd:"match",      n:"مطابقة",     ico:"match",big:1},
   {act:"renumCols",  n:"أعد الترقيم", ico:"renum"},
   G({cmd:"array",     n:"مصفوفة",       ico:"array"},
     {cmd:"arraypolar",n:"مصفوفة قطبية", ico:"arraypol"})]},
  CTX_COMMON("col")]},

 fix:{n:"أداة صحية", panels:[
  {id:"cf1", n:"الأداة", items:[
   {cmd:"match",   n:"مطابقة",             ico:"match",big:1},
   {act:"fixSnap", n:"ألصِق بأقرب جدار",    ico:"weld"}]},
  CTX_COMMON("fix")]},

 stair:{n:"درج", panels:[
  {id:"cs1", n:"الدرج", items:[
   {cmd:"match",n:"مطابقة",ico:"match",big:1}]},
  CTX_COMMON("stair")]}
};

/* ═══ شريط الوصول السريع ═══ */
export const QAT=[
 {act:"fnew", n:"جديد", ico:"fnew"},
 {act:"xOpen",n:"افتح", ico:"open"},
 {act:"xSave",n:"احفظ", ico:"save"},
 {act:"xPdf", n:"طبع PDF",ico:"print"},
 {sep:1},
 {act:"undo",n:"تراجع", ico:"undo"},
 {act:"redo",n:"إعادة", ico:"redo"},
 {sep:1},
 {act:"fit",  n:"ملاءمة",ico:"fit"},
 {act:"inspect",n:"افحص",ico:"inspect"}
];

/* ═══ استخلاصٌ للفحص الساكن ═══ */
const walk=(items,fn)=>(items||[]).forEach(it=>{
 if(it.group)walk(it.group,fn); else fn(it);
});
export function allItems(){
 const out=[];
 const tabs=RIBBON.concat(Object.keys(CTX).map(k=>CTX[k]));
 tabs.forEach(t=>(t.panels||[]).forEach(p=>walk(p.items,i=>out.push(i))));
 walk(QAT,i=>{if(!i.sep)out.push(i)});
 return out;
}
export const ribbonCmds=()=>[...new Set(allItems()
 .filter(i=>i.cmd).map(i=>i.cmd))];
export const ribbonActs=()=>[...new Set(allItems()
 .filter(i=>i.act).map(i=>i.act))];
export const ribbonIcons=()=>[...new Set(allItems()
 .filter(i=>i.ico).map(i=>i.ico))];
export const ribbonDlgs=()=>[...new Set([]
 .concat(...RIBBON.map(t=>(t.panels||[]).map(p=>p.dlg)))
 .filter(Boolean))];
export const tabIds=()=>RIBBON.map(t=>t.id);
export const panelIdList=()=>[]
 .concat(...RIBBON.map(t=>(t.panels||[]).map(p=>t.id+"/"+p.id)));
```

<a id="f-js-ui-ribbon-wire-js"></a>

---

## `js/ui/ribbon/wire.js`

```javascript
/* ═══ موصِّل الشريط ═══
   الأدوات تُنادى مباشرةً. وكل ما عداها ينقر زرَّه القائم في
   اللوحة الجانبية بعد فتح قسمه — فلا منطقَ مكرّراً، ولوحةٌ واحدة
   هي المرجع، ويرى المستخدم أين يسكن الأمر فيتعلّم موضعه.
   ولهذا يبقى #tools في الشجرة مخفيّاً: معالجه هو المرجع. */
import {RIBBON,CTX} from "./schema.js";
import {buildRibbon,buildQAT,setTab,curTab,setCtx,setMin,
        syncRibbon,syncRibbonTogs,showKT,invalidateSync,
        ctxOf} from "./render.js";
import {UIS,uiSet,saveUI} from "../store.js";
import {renderVisible,markDirty} from "../panels.js";
import * as R from "../../tools/registry.js";
import {draw,selList,setTheme as setCanvasTheme} from "../canvas.js";
import {HOOK} from "../bus.js";
import {revealPanel,dockClean,wsMenuOpen,wsApply,dockStats,
        openPanel,closePanel,toFloat,setAuto,setMode,
        layout} from "../dock.js";
import {PANELS,ZONES} from "../layout.js";
import {cmdMenu,setCmdMode,CMODES,applyCmd} from "../cmdline.js";
import {qpToggle,quickSel} from "../quickprops.js";
import {stSheet,stLock,custOpen} from "../statusbar.js";
import {runInspect} from "../inspector.js";
import {syncOverlay} from "../overlay.js";

const $=s=>document.querySelector(s);
const focusCl=()=>{const c=$("#clIn"); if(c)c.focus()};
const rtlDoc=()=>getComputedStyle(document.documentElement)
 .direction==="rtl";

/* ═══ إظهار لوحة ═══
   الشريط لا يعرف أين تسكن اللوحة: يطلب إظهارها، والإرساء يفتح
   عمودها أو يكشف شارتها أو يقدّم نافذتها. */
export function revealSec(sec){
 return revealPanel(sec);
}
/* ═══ الوكالة ═══
   لا يُنقَر زرٌّ معطَّل — الرسالة أوضح من نقرةٍ لا تفعل شيئاً. */
function proxy(sel,sec){
 if(sec)revealSec(sec);
 const el=document.querySelector(sel);
 if(!el){HOOK.report("wr",`لا زرَّ «${sel}» — لوحته غير مبنيّة`);
  return false}
 if(el.disabled){HOOK.report("in","غير متاح الآن"); return false}
 el.click();
 return true;
}
const P=(sel,sec)=>()=>proxy(sel,sec);

/* ═══ جدول الأفعال ═══ كلّها وكالةٌ إلّا ما لا زرَّ له ═══ */
export const ACT={
 /* الملفّ والتصدير */
 fnew:  P("#bNew","state"),
 xOpen: P("#xOpen","export"),
 xSave: P("#xSave","export"),
 xDxf:  P("#xDxf","export"),
 xSvg:  P("#xSvg","export"),
 xPng:  P("#xPng","export"),
 xPdf:  P("#xPdf","export"),
 asCsv: P("#bAsCsv","sched"),
 osCsv: P("#bOsCsv","osched"),

 /* المرجع */
 rImp:  P("#rImp","ref"),
 rClr:  P("#rClr","ref"),
 rRst:  P("#rRst","ref"),

 /* الطبقات والفحص */
 lAll:    P("#lAll","lays"),
 lUnlock: P("#lUnlock","lays"),
 inspect: P("#bInsp","insp"),

 /* العامّة */
 undo: P("#bUndo"),
 redo: P("#bRedo"),
 fit:  P("#bFit"),
 help: P("#bHelp"),

 /* الورقة والمحاور والافتراضات */
 shCenter: P("#shCenter","sheet"),
 axClr:    P("#bAxClr","axes"),
 dRst:     P("#dRst","defs"),
 osPop:    P("#osBtn"),

 /* ═══ المسح الكامل ═══
    لا زرَّ له في اللوحة فهو الأصل هنا. ولا سبيل إليه قبل اليوم
    إلّا من أدوات المطوّر — وهو مطلبٌ عمليّ لمن يستعمل حاسباً
    مشتركاً. والوعد يُذكَر بتفصيله لأنه لا يُتراجَع عنه. */
 purgeAll: ()=>{
  if(!confirm("مسحُ كلِّ ما هو محفوظ في هذا المتصفّح؟\n\n"
   +"• المشروع الجاري وجلسته\n"
   +"• تفضيلات الواجهة وأسطح العمل والمناظر\n"
   +"• خيارات الأدوات وافتراضاتها\n"
   +"• إعداد المزوّد ومفتاحه\n\n"
   +"لا يمسّ الملفّات التي حفظتها على قرصك. "
   +"احفظ مشروعك أوّلاً إن أردت الإبقاء عليه."))return;
  import("../../io/store.js")
   .then(M=>M.purge())
   .then(r=>{
    HOOK.report("ok",`مُسح ${r.keys.length} مفتاحاً`
     +(r.idb?" وقاعدة البيانات":"")
     +" — أعِد تحميل الصفحة للبدء من المصنع");
   })
   .catch(e=>HOOK.report("er","المسح: "+String(e.message||e)));
 },

 /* ═══ القياس ═══
    نقرةٌ أولى تُشغِّله، وثانيةٌ تعرض الحصيلة وتُطفئه. ومُطفأٌ
    افتراضاً فلا كلفةَ لمن لا يسأل. */
 perfShow: ()=>{
  import("../../core/perf.js").then(M=>{
   if(!M.P.on){
    M.perfOn(1);
    HOOK.report("in","القياس مُشتغِل — حرّك شيئاً أو اسحب مقبضاً، "
     +"ثم اختر «القياس» مرّةً أخرى للحصيلة");
    return;
   }
   M.perfReport().split("\n").forEach(l=>{
    HOOK.report(/^⚠/.test(l)?"wr":"in",l);
   });
   M.perfOn(0);
   HOOK.report("ok","أُطفئ القياس");
  }).catch(e=>HOOK.report("er","القياس: "+String(e.message||e)));
 },

 /* السياقية — أزرارٌ تظهر بحسب المحدَّد */
 delSel:    P("#pDel","props"),
 renumCols: P("[data-brenum]","props"),
 clrDimTxt: P("[data-bcleartxt]","props"),
 nameSeq:   P("[data-bname]","props"),
 fixSnap:   P("[data-fsnap]","props"),

 /* فتح اللوحات */
 propsDlg: ()=>revealSec("props"),
 laysDlg:  ()=>revealSec("lays"),
 schedDlg: ()=>revealSec("sched"),
 oschedDlg:()=>revealSec("osched"),
 refDlg:   ()=>revealSec("ref"),
 sheetDlg: ()=>revealSec("sheet"),
 inspDlg:  ()=>revealSec("insp"),
 projDlg:  ()=>revealSec("proj"),
 defsDlg:  ()=>revealSec("defs"),
 stateDlg: ()=>revealSec("state"),
 aiDlg:    ()=>revealSec("ai"),

 /* القشرة — لا زرَّ لها في اللوحة فهي الأصل هنا */
 rbMin: ()=>setMin(!UIS.ribbonMin),
 clean: ()=>setClean(!UIS.clean),
 theme: ()=>setTheme(UIS.theme==="dark"?"light":"dark"),
 shell: ()=>setShell(UIS.shell==="ribbon"?"classic":"ribbon")
 ,
 /* ═══ و٢ — الإرساء وأسطح العمل ═══ */
 wsMenu: ()=>{
  const b=document.querySelector('[data-act="wsMenu"]');
  const mid=innerWidth/2;
  const r=b?b.getBoundingClientRect()
   :{bottom:120,["l"+"eft"]:mid,["r"+"ight"]:mid};
  wsMenuOpen(rtlDoc()?r.left:r.right,r.bottom);
 },
 wsArch:  ()=>wsApply("arch"),
 wsAnnot: ()=>wsApply("annot"),
 wsOut:   ()=>wsApply("out"),
 dockAutoS: ()=>setAuto("s",!layout().auto.s),
 dockAutoE: ()=>setAuto("e",!layout().auto.e),
 dockTabS:  ()=>setMode("s",layout().mode.s==="acc"?"tab":"acc"),
 dockTabE:  ()=>setMode("e",layout().mode.e==="acc"?"tab":"acc")
 ,
 /* ═══ و٤ ═══ */
 cmdMode: ()=>{
  const b=document.querySelector('[data-act="cmdMode"]');
  const r=b?b.getBoundingClientRect()
   :{left:innerWidth/2,right:innerWidth/2,top:innerHeight-40};
  cmdMenu(rtlDoc()?r.left:r.right,r.top);
 },
 cmdFloat:  ()=>setCmdMode("float"),
 cmdBottom: ()=>setCmdMode("bottom"),
 qpTog:     ()=>qpToggle(),
 rclick: ()=>{
  const O=["auto","enter","menu"];
  const N={auto:"تلقائي — Enter مع أداة وقائمةٌ في السكون",
   enter:"Enter دائماً",menu:"قائمة دائماً"};
  const i=(O.indexOf(UIS.rclick)+1)%O.length;
  uiSet("rclick",O[i]);
  HOOK.report("in",`الزرّ الأيمن: ${N[O[i]]}`);
 },

 /* ═══ شريط الحالة ═══ يملك أفعاله، وتُعلَن هنا فيبقى الجدول واحداً */
 stSheet: ()=>stSheet(),
 stLock:  ()=>stLock(),
 stScale: ()=>revealSec("proj"),
 stWarn:  ()=>{runInspect(); revealSec("insp")},
 stCust:  ()=>{
  const b=document.querySelector('[data-act="stCust"]');
  const r=b?b.getBoundingClientRect()
   :{top:innerHeight-30,left:innerWidth/2,right:innerWidth/2};
  custOpen(rtlDoc()?r.left:r.right, r.top);
 }
};
/* ═══ السِّمة والشاشة النظيفة والقشرة ═══ */
export function setTheme(t){
 const v=(t==="light")?"light":"dark";
 uiSet("theme",v);
 setCanvasTheme(v);          /* يضبط data-theme ويُبطل كاش النقوش */
 HOOK.report("in",`السِّمة: ${v==="light"?"فاتحة":"داكنة"}`);
}
export function setClean(on){
 const v=on?1:0;
 uiSet("clean",v);
 document.documentElement.classList.toggle("clean",!!v);
 dockClean();                    /* مزامنةٌ فقط — العلَم مملوكٌ هنا */
 applyCmd();                     /* سطر الأوامر والسجل: مالكهما cmdline */
 const rb=$("#ribbon");
 if(rb&&UIS.shell==="ribbon")rb.hidden=!!v;
 const tb=$("#tools");
 if(tb&&UIS.shell==="classic")tb.hidden=!!v;
 quickSel();                     /* البطاقة السريعة تختفي وتعود */
 syncOverlay();                  /* البوصلة وبطاقة المنظور */
 dispatchEvent(new Event("resize"));
 HOOK.report("in",v?"شاشة نظيفة · Ctrl+0 يعيد الواجهة"
  :"عادت الواجهة");
}
export function setShell(s){
 const v=(s==="ribbon")?"ribbon":"classic";
 uiSet("shell",v);
 const rb=$("#ribbon"), tb=$("#tools");
 if(v==="ribbon"){
  if(rb&&!rb.dataset.built){buildRibbon(); rb.dataset.built="1"}
  setMin(UIS.ribbonMin);
  if(rb)rb.hidden=!!UIS.clean;
  if(tb)tb.hidden=true;
  invalidateSync();
  syncRibbon(); syncRibbonTogs(); ribbonSel();
 }else{
  if(rb)rb.hidden=true;
  if(tb)tb.hidden=!!UIS.clean;
 }
 document.documentElement.dataset.shell=v;
 dispatchEvent(new Event("resize"));
 HOOK.report("in",`القشرة: ${v==="ribbon"?"شريط ربّون":"مسطّحة"}`);
 return v;
}
/* ═══ التبويب السياقي من التحديد ═══ */
let lastSig=null;
export function ribbonSel(){
 if(UIS.shell!=="ribbon"){lastSig=null; return}
 const L=selList();
 let kind=null;
 if(L.length){
  kind=L[0].k;
  for(const s of L)if(s.k!==kind){kind=null; break}
 }
 const sig=kind?(kind+":"+L.length):"";
 if(sig===lastSig)return;
 lastSig=sig;
 setCtx(CTX[kind]?kind:null);
 invalidateSync();
 syncRibbon();
}
/* ═══ التنفيذ ═══
   runSpec يقبل الوصف مباشرةً، فتشترك فيه قائمةُ السياق والشريط
   وقائمةُ التطبيق — مُنفِّذٌ واحد لا ثلاثة. */
export function runSpec(sp){
 if(!sp)return false;
 if(sp.cmd!==undefined&&sp.cmd!==null){
  if(!sp.cmd)R.cancel(true); else R.begin(sp.cmd);
  HOOK.prompt(); draw(); focusCl();
  return true;
 }
 if(sp.tog){
  const t=document.querySelector(sp.tog);
  if(t)t.click();
  syncRibbonTogs();
  return true;
 }
 const act=sp.act;
 if(!act)return false;
 if(act.startsWith("dlg:")){revealSec(act.slice(4)); return true}
 const fn=ACT[act];
 if(!fn){HOOK.report("wr",`فعلٌ غير معروف: ${act}`); return false}
 try{fn()}
 catch(e){HOOK.report("er",String(e.message||e))}
 return true;
}
export function runItem(el){
 if(!el)return false;
 if(el.dataset.act==="wsMenu")el.dataset.wsx="1";
 return runSpec({cmd:el.dataset.cmd,act:el.dataset.act,
  tog:el.dataset.tog});
}
/* ═══ التوصيل ═══ */
export function wireRibbon(){
 const rb=$("#ribbon");
 if(!rb)return false;

 rb.addEventListener("click",e=>{
  const tab=e.target.closest("[data-tab]");
  if(tab){setTab(tab.dataset.tab); saveUI(); return}
  if(e.target.closest("#rbToggle")){setMin(!UIS.ribbonMin);
   saveUI(); dispatchEvent(new Event("resize")); return}
  runItem(e.target.closest("[data-cmd],[data-act],[data-tog]"));
 });
 /* نقرتان على التبويب النشط تطويان — كما في أوتوكاد */
 rb.addEventListener("dblclick",e=>{
  const tab=e.target.closest("[data-tab]");
  if(!tab)return;
  e.preventDefault();
  if(tab.dataset.tab===curTab()){setMin(!UIS.ribbonMin); saveUI();
   dispatchEvent(new Event("resize"))}
 });
 /* الأسهم بين التبويبات · وMenu/Home/End — ARIA كاملة */
 const tabList=()=>[...rb.querySelectorAll(".rbTab")];
 rb.addEventListener("keydown",e=>{
  const T=tabList();
  const i=T.findIndex(b=>b===document.activeElement);
  if(i<0)return;
  /* RTL: السهم الأيمن يتقدّم في القراءة العربية */
  const step=(e.key==="ArrowLeft")?1:((e.key==="ArrowRight")?-1:0);
  let j=-1;
  if(step)j=(i+step+T.length)%T.length;
  else if(e.key==="Home")j=0;
  else if(e.key==="End")j=T.length-1;
  else return;
  e.preventDefault();
  T[j].focus();
  setTab(T[j].dataset.tab); saveUI();
 });
 const q=$("#qat");
 if(q)q.addEventListener("click",e=>{
  runItem(e.target.closest("[data-act]"));
 });
 wireKT();
 return true;
}
/* ═══ KeyTips ═══
   Alt وحده يُظهر الدلائل، ثم رقمٌ مفردٌ ينتقل — لا Alt+رقم لأن
   المتصفّح يحتجزه. والأرقام أثبت من الحروف في لوحةٍ عربية: مفتاحها
   الفيزيائي واحدٌ في كل تخطيط، فنقرأ e.code لا e.key.
   ونُنصت في طور الالتقاط لنسبق قاعدة app.js التي تُرسل كل حرفٍ
   مطبوعٍ إلى سطر الإدخال. */
let KT=false, altOnly=false;
function ktOff(){if(!KT)return; KT=false; showKT(false)}

function wireKT(){
 addEventListener("keydown",e=>{
  if(e.key==="Alt"&&!e.ctrlKey&&!e.metaKey&&!e.shiftKey){
   if(!e.repeat)altOnly=true;
   return;
  }
  altOnly=false;
  if(!KT)return;
  if(e.key==="Escape"){e.preventDefault(); e.stopPropagation();
   ktOff(); return}
  const m=/^Digit([0-9])$/.exec(e.code||"");
  if(m&&!e.ctrlKey&&!e.altKey&&!e.metaKey){
   e.preventDefault(); e.stopPropagation();
   const d=m[1];
   ktOff();
   if(d==="0"){const b=$("#appBtn"); if(b)b.click(); return}
   const t=RIBBON.find(x=>x.kt===d);
   if(t){setTab(t.id); saveUI();
    const b=document.querySelector(`[data-tab="${t.id}"]`);
    if(b)b.focus();
   }
   return;
  }
  ktOff();               /* أي مفتاحٍ آخر يُغلق ويمرّ */
 },true);

 addEventListener("keyup",e=>{
  if(e.key!=="Alt")return;
  if(!altOnly)return;
  altOnly=false;
  e.preventDefault();
  if(UIS.shell!=="ribbon")return;
  KT=!KT;
  showKT(KT);
 },true);

 addEventListener("mousedown",()=>ktOff(),true);
 addEventListener("blur",()=>ktOff());
}
export const ktOn=()=>KT;

/* ═══ الارتفاع ═══
   الشاشة القصيرة تُطوى مرّةً واحدة ولا تُقاوَم بعدها: إن فتحها
   المستخدم بقيت مفتوحة. */
let autoDone=false;
export function autoFit(){
 if(UIS.shell!=="ribbon"||autoDone)return;
 if(innerHeight<760&&!UIS.ribbonMin){
  autoDone=true;
  setMin(1); saveUI();
  HOOK.report("in","طُوي الشريط لضيق الشاشة · Ctrl+F1 يفتحه");
 }
}
```

<a id="f-js-ui-statusbar-js"></a>

---

## `js/ui/statusbar.js`

```javascript
/* ═══ شريط الحالة ═══
   سجلٌّ لا قالب: كل عنصرٍ سطرٌ هنا، وقائمة التخصيص تُبنى منه.
   والعنصر المُخفى يبقى في الشجرة بسمة hidden لا يُنزَع — لأن
   الشريط ينقر [data-rb] بالوكالة، فنزعه يعطّل مفتاحه هناك بلا
   خطأٍ ظاهر. والنقر على المخفيّ برمجياً يعمل.

   وما ليس له مُنفِّذ اليوم ليس فيه: لا وزنَ خطٍّ ولا شفافيةَ ولا
   طبقةً حالية ولا نظامَ إحداثيات — تدخل مع مراحلها. */
import {S,edit} from "../core/state.js";
import {icon} from "./icons.js";
import {UIS,uiSet,saveUI} from "./store.js";
import {scene} from "../core/render.js";
import {HOOK} from "./bus.js";
import {scl} from "../core/units.js";

const $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s)
 .replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const ic=(n,s)=>UIS.icons?icon(n,s||14):"";

/* k: txt · rb · act · dv · gap ═══ opt=1 يقبل الإخفاء */
export const ITEMS=[
 {k:"txt", id:"pos",  el:"stPos", cls:"mono num", n:"الإحداثيات",
  t:"موضع المؤشّر بالمتر", opt:1, v:"0.00 , 0.00 م"},
 {k:"dv"},
 {k:"rb", id:"grid",  rb:"grid",  n:"الشبكة",   ico:"grid",
  t:"إظهار الشبكة · F6", opt:1},
 {k:"rb", id:"gsnap", rb:"gsnap", n:"خطوة",     ico:"gsnap",
  t:"الالتقاط على خطوة الشبكة · F9", opt:1},
 {k:"rb", id:"ortho", rb:"ortho", n:"تعامد",    ico:"ortho",
  t:"F8", opt:1},
 {k:"rb", id:"polar", rb:"polar", n:"قطبي",     ico:"polar",
  t:"F10", opt:1, pop:"polBtn"},
 {k:"rb", id:"snap",  rb:"snap",  n:"التقاط",   ico:"osnap",
  t:"التقاط الكائنات · F3", opt:1, pop:"osBtn"},
 {k:"rb", id:"grips", rb:"grips", n:"مقابض",    ico:"grips",
  t:"F11", opt:1},
 {k:"rb", id:"ends",  rb:"ends",  n:"أطراف",    ico:"ends",
  t:"علامات الأطراف غير المتّصلة · F12", opt:1},
 {k:"rb", id:"paths", rb:"paths", n:"المسارات", ico:"paths",
  t:"المسار المرسوم للجدار المحاذي", opt:1},
 {k:"rb", id:"dyn", rb:"dyn", n:"إدخال حركي", ico:"dyn",
  t:"حقولٌ عند المؤشّر · Ctrl+D", opt:1},
 {k:"dv"},
 {k:"act", id:"sheet", act:"stSheet", n:"الورقة", ico:"sheet",
  t:"إظهار الورقة وبلوك العنوان", opt:1},
 {k:"act", id:"scale", act:"stScale", n:"1:100", ico:"",
  t:"المقياس — انقر لإعداد المشروع", opt:1, cls:"mono num wide"},
 {k:"act", id:"quick", act:"qpTog", n:"", ico:"qp",
  t:"الخصائص السريعة", opt:1},
 {k:"act", id:"cmd", act:"cmdMode", n:"", ico:"cmdl",
  t:"موضع سطر الأوامر والسجل", opt:1},
 {k:"dv"},
 {k:"txt", id:"sel", el:"stSel", n:"المحدَّد", t:"", opt:1, v:""},
 {k:"gap"},
 {k:"txt", id:"hint", el:"stHint", n:"إرشاد الأداة", t:"", opt:1,
  v:""},
 {k:"dv"},
 {k:"act", id:"warn", act:"stWarn", n:"—", ico:"warn",
  t:"ملاحظات المشهد — انقر للفاحص", opt:1},
 {k:"dv"},
 {k:"txt", id:"info", el:"stInfo", n:"آخر رسالة", t:"", opt:1,
  v:""},
 {k:"dv"},
 {k:"act", id:"ws", act:"wsMenu", n:"", ico:"ws",
  t:"أسطح العمل", opt:1},
 {k:"act", id:"lockUI", act:"stLock", n:"", ico:"lock",
  t:"قفل تخطيط اللوحات", opt:1},
 {k:"act", id:"clean", act:"clean", n:"", ico:"clean",
  t:"شاشة نظيفة · Ctrl+0", opt:1},
 {k:"act", id:"cust", act:"stCust", n:"", ico:"menu",
  t:"عناصر شريط الحالة"}
];
export const stIds=()=>ITEMS.filter(x=>x.opt).map(x=>x.id);
export const stItem=id=>ITEMS.find(x=>x.id===id)||null;
const shownOf=id=>{
 const h=UIS.stHide||{};
 return !h[id];
};
export const stShown=shownOf;

function html(x){
 const hid=(x.opt&&!shownOf(x.id))?" hidden":"";
 if(x.k==="dv")return `<span class="dv"${hid}></span>`;
 if(x.k==="gap")return `<span class="gap"></span>`;
 if(x.k==="txt")
  return `<span id="${esc(x.el)}" data-sti="${esc(x.id)}"`
   +` class="${esc(x.cls||"")}"${hid}${x.t?` title="${esc(x.t)}"`:""}`
   +`>${esc(x.v||"")}</span>`;
 if(x.k==="rb")
  return `<button type="button" data-rb="${esc(x.rb)}"`
   +` data-sti="${esc(x.id)}" title="${esc(x.n+" · "+(x.t||""))}"`
   +` aria-pressed="false"${hid}>${ic(x.ico)}`
   +`<span class="lb">${esc(x.n)}</span></button>`
   +(x.pop?`<button type="button" id="${esc(x.pop)}" class="stPop"`
    +` data-sti="${esc(x.id)}" title="خيارات ${esc(x.n)}"`
    +` aria-label="خيارات ${esc(x.n)}"${hid}>▾</button>`:"");
 return `<button type="button" data-act="${esc(x.act)}"`
  +` data-sti="${esc(x.id)}" class="${esc(x.cls||"")}"`
  +` title="${esc(x.t||x.n)}"${hid}>${ic(x.ico)}`
  +(x.n?`<span class="lb">${esc(x.n)}</span>`:"")+`</button>`;
}
export function buildStatus(){
 const box=$("#stItems");
 if(!box)return 0;
 box.innerHTML=ITEMS.map(html).join("");
 return ITEMS.length;
}
/* ═══ المزامنة ═══ رخيصةٌ: تُنادى مع كل تحديثٍ للواجهة ═══
   لا مذاكرةَ في مستوى الوحدة: buildStatus يعيد بناء العناصر إلى
   نصوصها المُعلَنة بلا تصفيرها، فمذاكرةٌ هنا تكذب بعد أي إعادة
   بناء. القارن هو محتوى العنصر نفسه. */
export function syncStatus(){
 const box=$("#stItems");
 if(!box)return;
 box.querySelectorAll("[data-rb]").forEach(b=>{
  const on=!!+S.rb[b.dataset.rb];
  b.classList.toggle("on",on);
  b.setAttribute("aria-pressed",on?"true":"false");
 });
 const sh=box.querySelector('[data-act="stSheet"]');
 if(sh)sh.classList.toggle("on",!!+S.sheet.on);
 const lk=box.querySelector('[data-act="stLock"]');
 if(lk){
  lk.classList.toggle("on",!!UIS.lockUI);
  lk.title=UIS.lockUI?"تخطيط اللوحات مقفل":"قفل تخطيط اللوحات";
 }
 const cl=box.querySelector('[data-act="clean"]');
 if(cl)cl.classList.toggle("on",!!UIS.clean);
 const sc=box.querySelector('[data-act="stScale"] .lb');
 const scv=scl(S.meta.scale);
 if(sc&&sc.textContent!==scv)sc.textContent=scv;
 const qp=box.querySelector('[data-act="qpTog"]');
 if(qp)qp.classList.toggle("on",!!UIS.qp);
 const w=box.querySelector('[data-act="stWarn"]');
 if(!w)return;
 let n=0, tip="لا ملاحظات";
 try{
  const s=scene(), P=[];
  if(s.bad)P.push(`${s.bad} فتحة معطوبة`);
  if(s.stale)P.push(`${s.stale} منطقة قديمة`);
  if(s.loose)P.push(`${s.loose} بُعداً معلَّقاً`);
  if(s.over)P.push(`${s.over} بُعداً بنصّ بديل`);
  if(s.hidden)P.push(`${s.hidden} كياناً مخفيّاً`);
  n=s.bad+s.stale+s.loose+s.over+s.hidden;
  if(P.length)tip=P.join(" · ");
 }catch(e){}
 const lb=w.querySelector(".lb");
 const txt=n?String(n):"—";
 if(lb&&lb.textContent!==txt)lb.textContent=txt;
 w.classList.toggle("bad",n>0);
 w.title=tip+" — انقر للفاحص";
}
/* ═══ قائمة التخصيص ═══ */
export function custOpen(x,y){
 const m=$("#stMenu");
 if(!m)return;
 const rtl=getComputedStyle(document.documentElement)
  .direction==="rtl";
 m.innerHTML=`<div class="pmH">عناصر شريط الحالة</div>`
  +ITEMS.filter(i=>i.opt).map(i=>
   `<button type="button" class="pmI" data-stc="${esc(i.id)}">`
   +`${ic(i.ico||"panel")}<span>${esc(i.n||i.id)}</span>`
   +`<span class="ky">${shownOf(i.id)?"●":""}</span></button>`)
   .join("")
  +`<div class="pmS"></div>`
  +`<button type="button" class="pmI" data-stc="__all">`
  +`${ic("grid")}<span>أظهر الكل</span></button>`;
 m.hidden=false;
 const w=m.offsetWidth||226, h=m.offsetHeight||300;
 const cl=(v,a,b)=>v<a?a:(v>b?b:v);
 m.style.insetInlineStart=Math.round(
  cl(rtl?(innerWidth-x-4):(x-w+4),4,innerWidth-w-4))+"px";
 m.style.insetBlockStart=Math.round(cl(y-h-8,4,innerHeight-h-8))+"px";
}
export const custClose=()=>{
 const m=$("#stMenu");
 if(m)m.hidden=true;
};
function custRun(id){
 UIS.stHide=UIS.stHide||{};
 if(id==="__all"){
  UIS.stHide={};
  HOOK.report("in","أُظهرت كل عناصر شريط الحالة");
 }else{
  const it=stItem(id);
  if(!it||!it.opt)return;
  if(UIS.stHide[id])delete UIS.stHide[id]; else UIS.stHide[id]=1;
  HOOK.report("in",`${it.n||id}: ${shownOf(id)?"ظاهر":"مخفيّ"}`);
 }
 saveUI();
 buildStatus();
 syncStatus();
 HOOK.toggles();
 custOpen(innerWidth/2,innerHeight-30);
}
/* ═══ التوصيل ═══
   المفاتيح [data-rb] يتولّاها app.js بالتفويض على #status، فلا
   تُربَط هنا مرّةً ثانية. */
export function wireStatus(){
 buildStatus();
 const box=$("#stItems");
 if(!box)return false;
 document.addEventListener("click",e=>{
  const c=e.target.closest("[data-stc]");
  if(c){custRun(c.dataset.stc); return}
 });
 addEventListener("mousedown",e=>{
  const m=$("#stMenu");
  if(!m||m.hidden)return;
  if(!m.contains(e.target)&&!e.target.closest('[data-act="stCust"]'))
   custClose();
 },true);
 addEventListener("keydown",e=>{
  const m=$("#stMenu");
  if(e.key==="Escape"&&m&&!m.hidden){
   e.preventDefault(); e.stopPropagation(); custClose();
  }
 },true);
 return true;
}
/* ═══ أفعالٌ يملكها الشريط ═══ */
export function stSheet(){
 edit(()=>{S.sheet.on=+S.sheet.on?0:1});
 HOOK.report("in",`الورقة: ${+S.sheet.on?"ظاهرة":"مخفيّة"}`);
 HOOK.refresh(0);
}
export function stLock(){
 uiSet("lockUI",UIS.lockUI?0:1);
 syncStatus();
 HOOK.report("in",UIS.lockUI
  ?"تخطيط اللوحات مقفل — لا سحبَ ولا تحجيم"
  :"تخطيط اللوحات مفتوح");
}
```

<a id="f-js-ui-store-js"></a>

---

## `js/ui/store.js`

```javascript
/* ═══ مخزن الواجهة ═══
   حالة النافذة لا حالة الرسم: مفتاحٌ منفصل لا يدخل pack() ولا
   التاريخ ولا ملفّ المشروع. فتح لوحةٍ ليس تعديلاً على المخطط،
   وحفظ المشروع لا يحمل تفضيلات نافذتك إلى حاسبٍ آخر.

   وS.lay يبقى في المشروع لأن إخفاء طبقةٍ يغيّر ما يُصدَّر — فهو
   حالة رسم. هذا هو الفرق بين المخزنَين، وهو صريح لا ضمنيّ. */

const KEY="mistar.ui";

export const DEFUI=()=>({
 shell:"ribbon",       /* classic | ribbon — الشريط المجمَّع افتراضياً */
 theme:"dark",        /* dark | light */
 icons:1,
 tab:"home",          /* آخر تبويبٍ نشط */
 ribbonMin:0,         /* الشريط مطويّ */
 clean:0,             /* الشاشة النظيفة */
 ctxAuto:0,           /* الانتقال التلقائي إلى التبويب السياقي */
 layout:null,         /* يُطبَّع في dock.js — DEFLAY مصنعه */
 ws:{},               /* أسطح عملٍ محفوظة بالاسم */
 wsCur:"",            /* السطح الجاري */
 logH:120,
 stHide:{},           /* معرّف عنصرٍ في شريط الحالة ⇒ 1 مخفيّ */
 views:[],            /* مناظر مسمّاة — إحداثيُّ عرضٍ لا بياناتُ رسم */
 compass:1, vpLabel:1, navbar:1, lockUI:0,
 cmdMode:"bottom", cmdOpa:100, cmdX:24, cmdY:0, cmdW:620,
 rclick:"auto",       /* auto | enter | menu */
 qp:1, qpFree:0, qpX:16, qpY:16
});
export const UIS=DEFUI();
let FAIL=0;

export function loadUI(skip){
 if(skip)return false;              /* وضع الإنقاذ: المصنع وحده */
 if(typeof localStorage==="undefined")return false;
 try{
  const raw=localStorage.getItem(KEY);
  if(!raw)return false;
  const d=JSON.parse(raw);
  if(!d||typeof d!=="object")return false;
  const def=DEFUI();
  Object.keys(def).forEach(k=>{
   if(d[k]!==undefined&&typeof d[k]===typeof def[k])UIS[k]=d[k];
  });
  if(!/^(classic|ribbon)$/.test(UIS.shell))UIS.shell="ribbon";
  if(!/^(dark|light)$/.test(UIS.theme))UIS.theme="dark";
  UIS.icons=UIS.icons?1:0;
  if(typeof UIS.tab!=="string"||!/^[\w-]{1,16}$/.test(UIS.tab))
   UIS.tab="home";
  ["ribbonMin","clean","ctxAuto"].forEach(k=>{UIS[k]=UIS[k]?1:0});
  UIS.logH=Math.max(0,Math.min(400,+UIS.logH||120));
  /* بعد — الكائن المستوي وحده: المصفوفة تمرّ من typeof فيدخل
     `ws:[]` فيُعطب Object.keys ويُفرغ أسطح العمل صامتاً */
  const plain=v=>!!v&&typeof v==="object"&&!Array.isArray(v);
  if(!plain(UIS.layout))UIS.layout=null;
  if(!plain(UIS.ws))UIS.ws={};
  UIS.ws=Object.fromEntries(Object.entries(UIS.ws).slice(0,24));
  UIS.wsCur=String(UIS.wsCur||"").slice(0,32);
  if(!plain(UIS.stHide))UIS.stHide={};
  UIS.views=(Array.isArray(UIS.views)?UIS.views:[])
   .filter(v=>v&&typeof v.n==="string"&&isFinite(v.k)
    &&isFinite(v.cx)&&isFinite(v.cy)).slice(0,24);
  ["compass","vpLabel","navbar","lockUI"]
   .forEach(k=>{UIS[k]=UIS[k]?1:0});
  if(!/^(bottom|top|float)$/.test(UIS.cmdMode))UIS.cmdMode="bottom";
  if(!/^(auto|enter|menu)$/.test(UIS.rclick))UIS.rclick="auto";
  UIS.cmdOpa=Math.max(40,Math.min(100,+UIS.cmdOpa||100));
  ["qp","qpFree"].forEach(k=>{UIS[k]=UIS[k]?1:0});
  ["cmdX","cmdY","cmdW","qpX","qpY"].forEach(k=>{
   UIS[k]=isFinite(+UIS[k])?Math.round(+UIS[k]):0;
  });
  return true;
 }catch(e){return false}
}
let T=null, ONFAIL=null, WARNED=false;
/* الفشل يُقال مرّةً: الصمت هو ما كان يُفقِد المستخدم تخطيطه بلا
   كلمة، فيكتشفه في الإقلاع التالي — وuiFailed كان يُقرأ في boot
   وحده، فامتلاءٌ وسط الجلسة لا يُبلَّغ عنه أبداً. */
export const setUiError=f=>{ONFAIL=(typeof f==="function")?f:null};
export function saveUI(){
 if(typeof localStorage==="undefined")return;
 if(T)clearTimeout(T);
 T=setTimeout(()=>{
  T=null;
  try{
   localStorage.setItem(KEY,JSON.stringify(UIS));
   FAIL=0;
  }catch(e){
   FAIL++;
   if(!WARNED&&ONFAIL){
    WARNED=true;
    ONFAIL((e&&e.name)||"خطأ");
   }
  }
 },500);
}
export const uiFailed=()=>FAIL;

export function uiSet(k,v){
 if(!(k in UIS))return false;
 UIS[k]=v; saveUI();
 return true;
}
/* ═══ انفتاح اللوحات ═══
   يسكن في layout.p[id].o — مصدرٌ واحد. وpanels.js يكتب هنا عند
   كل طيٍّ أو فتح، فلا يفترق المرسوم عن المحفوظ. */
const pOf=id=>{
 if(!UIS.layout||!UIS.layout.p)return null;
 return UIS.layout.p[id]||null;
};
export function secSet(id,open){
 const p=pOf(id);
 if(!p)return;
 p.o=open?1:0;
 saveUI();
}
export const secOpen=(id,def)=>{
 const p=pOf(id);
 return p?!!p.o:!!def;
};
export function resetUI(){
 const d=DEFUI();
 Object.keys(d).forEach(k=>{UIS[k]=d[k]});
 saveUI();
}
```

<a id="f-js-ui-theme-js"></a>

---

## `js/ui/theme.js`

```javascript
/* ═══ الألوان ═══
   لا جدولَ ألوانٍ هنا بعد الآن: مصدرٌ واحد في core/layers.js
   (resolve) يقرأه القماش والمصدِّرون الأربعة معاً. كانت ألوان
   الطبع منسوخةً هنا وفي png.js وsvg.js وقد تفرّقت فعلاً —
   م٠(٣) توحّدها في core/laydef.js.

   السِّمة الداكنة تعيد لون الشاشة كما هو، فلا يتغيّر منها بكسل.
   والفاتحة تستعمل ألوان الطبع نفسها: فما تراه على الشاشة الفاتحة
   هو ما يخرج على الورق. */
import {resolve} from "../core/layers.js";
import {PRN} from "../core/laydef.js";
export {PRN as PRINT};   /* من كان يستورد PRINT يبقى عاملاً */
const DARK={
 /* ═══ منسجمة مع css/theme.css — نظام "الأتيليه" ═══ */
 bg:"#101317",
 gMinor:"#171b21", gMajor:"#1e242b", gAxis:"#3a434e",
 hatch:"rgba(167,176,187,.55)", solid:"#727c88",
 tint:"rgba(200,164,92,.07)",
 sel:"#e3b567", sel2:"#c8a45c",
 grip:"#6ea8fe", gripHov:"#e3b567", gripHot:"#e8736f",
 gripLn:"#101317",
 snap:"#6cc08a", snapLn:"rgba(108,192,138,.5)",
 pre:"rgba(110,168,254,.62)", preTx:"#cfe3ff",
 preLn:"rgba(110,168,254,.5)",
 chip:"rgba(16,19,23,.9)", chip2:"rgba(16,19,23,.85)",
 trkO:"rgba(108,192,138,.45)", trkP:"rgba(227,181,103,.42)",
 guide:"rgba(227,181,103,.5)", path:"rgba(110,168,254,.25)",
 loose:"#e8736f", bar:"#a7b0bb",
 win:"#6ea8fe", winF:"rgba(110,168,254,.08)",
 cross:"#6cc08a", crossF:"rgba(108,192,138,.08)",
 ghost:"#e3b567",
 warn:"#e3b567", bad:"#e8736f", er:"#e8736f",
 dflt:"#e9edf2"
};
const LIGHT={
 /* ═══ منسجمة مع css/theme.css — نظام "الأتيليه" ═══
    bg هنا يطابق --bg (#f4f5f7) لا الأبيض الخالص، ليتّسق مع
    خلفية الواجهة المحيطة. الفرق عن الورق المطبوع طفيفٌ جداً
    (٪٣ رمادية)، ولو أردت مطابقة الطباعة ١٠٠٪ أعد bg إلى #ffffff. */
 bg:"#f4f5f7",
 gMinor:"#ebedf1", gMajor:"#e2e5ea", gAxis:"#b8c0ca",
 hatch:"rgba(75,83,93,.55)", solid:"#78818c",
 tint:"rgba(138,106,32,.13)",
 sel:"#8a5a00", sel2:"#8a6a20",
 grip:"#1f68cc", gripHov:"#8a5a00", gripHot:"#b3261e",
 gripLn:"#f4f5f7",
 snap:"#137a43", snapLn:"rgba(19,122,67,.5)",
 pre:"rgba(31,104,204,.7)", preTx:"#0e4794",
 preLn:"rgba(31,104,204,.5)",
 chip:"rgba(244,245,247,.94)", chip2:"rgba(244,245,247,.9)",
 trkO:"rgba(19,122,67,.5)", trkP:"rgba(138,90,0,.5)",
 guide:"rgba(138,90,0,.55)", path:"rgba(31,104,204,.3)",
 loose:"#b3261e", bar:"#4b535d",
 win:"#1f68cc", winF:"rgba(31,104,204,.08)",
 cross:"#137a43", crossF:"rgba(19,122,67,.08)",
 ghost:"#8a5a00",
 warn:"#8a5a00", bad:"#b3261e", er:"#b3261e",
 dflt:"#171a1e"
};
export const SCREEN={dark:DARK,light:LIGHT};
export const KEYS=Object.keys(DARK);

let CUR="dark";
export const pal=()=>SCREEN[CUR];
export const themeName=()=>CUR;
export const isDark=()=>CUR==="dark";
export function setPal(t){
 CUR=(t==="light")?"light":"dark";
 document.documentElement.dataset.theme=CUR;
 return CUR;
}
/* ═══ الطبقة على الشاشة ═══
   لا جدولَ ألوانٍ هنا بعد الآن: مصدرٌ واحد في core/layers.js
   يقرأه القماش والمصدِّرون معاً. */
export const layCss=L=>resolve(L,CUR).css;

/* لون الأوّلية — العطب والتحذير يسبقان الطبقة */
export const primCss=g=>{
 const P=SCREEN[CUR];
 return g.bad?P.bad:(g.warn?P.warn:layCss(g.L||"0"));
};
```

<a id="f-js-ui-tour-js"></a>

---

## `js/ui/tour.js`

```javascript
/* ═══ جولة أول تشغيل ═══ */
import {S} from "../core/state.js";
import {setOpt,begin,cancel} from "../tools/registry.js";
import {SIZES,applySize} from "../tools/presets.js";
import {paletteIsOpen} from "./palette.js";
import {HOOK} from "./bus.js";

const K="mistar.tour", $=s=>document.querySelector(s);
const esc=s=>String(s==null?"":s).replace(/&/g,"&amp;").replace(/</g,"&lt;")
 .replace(/>/g,"&gt;").replace(/"/g,"&quot;");
const size=id=>SIZES.find(p=>p.id===id);
const seen=()=>{try{return localStorage.getItem(K)==="1"}catch(e){return true}};
const mark=()=>{try{localStorage.setItem(K,"1")}catch(e){}};
const STEPS=[
 {t:"أهلاً بك في CivilDraft",s:"الإدخال بالمتر، والتخزين بالمليمتر. سطر الإدخال أسفل اللوحة هو مركز الأوامر. ست خطوات قصيرة ونصير جاهزين."},
 {t:"١ · ارسم غرفة",s:"فعّلتُ أداة «مستطيل» بسماكة ٠٫٢٥ م. انقر ركناً ثم اكتب <code>4x3</code> واضغط Enter.",
  go(){setOpt("rect","t","0.25");setOpt("rect","type","ext");setOpt("rect","align","l");begin("rect")},
  ok(){return S.walls.length>=4}},
 {t:"٢ · ضع باباً",s:"فعّلتُ «باب غرفة» ٠٫٩٠ × ٢٫١٠ م. انقر على أي جدار، وEsc ينهي الأداة.",
  go(){const p=size("s.door.room");if(p)applySize(p)},ok(){return S.opens.some(o=>/^(door|double|sliding)$/.test(o.kind))}},
 {t:"٣ · ضع شباكاً",s:"فعّلتُ «شباك غرفة نوم» ١٫٥٠ × ١٫٤٠ م. انقر على جدار خارجي.",
  go(){const p=size("s.win.room");if(p)applySize(p)},ok(){return S.opens.some(o=>/^(window|fixed)$/.test(o.kind))}},
 {t:"٤ · اخبز المنطقة",s:"فعّلتُ أداة «منطقة». انقر داخل الغرفة فتُحسَب مساحتها وتُحفظ ككائن مستقل.",
  go(){cancel(true);begin("area")},ok(){return S.areas.length>=1}},
 {t:"٥ · افحص عملك",s:"اضغط <b>F7</b>. الفاحص يجمع الملاحظات الهندسية واشتراطات التصميم، ويخبرك ولا يصلح."},
 {t:"٦ · لوحة الأوامر",s:"اضغط <b>Ctrl+K</b>. ابحث بالعربية أو اللاتينية عن الأدوات والمقاسات والقوالب.",
  ok(){return paletteIsOpen()}},
 {t:"جاهز",s:"<b>F1</b> للمساعدة · <b>Ctrl+Z</b> للتراجع · <b>Ctrl+S</b> للحفظ. وعملك يُحفظ تلقائياً."}
];
let I=-1,TM=null;
const stop=()=>{if(TM){clearInterval(TM);TM=null}};
export function tourEnd(q){
 stop();I=-1;mark();const B=$("#tour");if(B)B.hidden=true;
 if(!q)HOOK.report("in","انتهت الجولة — Ctrl+K لكل شيء");
}
function watch(){
 stop();const st=STEPS[I];if(!st?.ok)return;
 TM=setInterval(()=>{let done=false;try{done=!!st.ok()}catch(e){}
  if(done)go(I+1,1)},400);
}
function render(){
 const B=$("#tour"),st=STEPS[I];if(!B||!st)return;
 B.innerHTML=`<div class="tc" role="note"><div class="th"><b>${esc(st.t)}</b>
  <span class="tn">${I+1} من ${STEPS.length}</span></div><div class="tb">${st.s}</div>
  <div class="tf"><button data-t="next">${I+1<STEPS.length?(st.ok?"تخطّ هذه":"التالي"):"تمّ"}</button>
  <button data-t="end" class="gh">أنهِ الجولة</button></div></div>`;
 B.hidden=false;
}
function go(i,auto){
 stop();if(i>=STEPS.length){tourEnd();return}I=i;const st=STEPS[I];
 if(st.go){try{st.go()}catch(e){HOOK.report("wr",e.message)}}
 render();if(auto)HOOK.report("ok","تمّت الخطوة");HOOK.prompt();watch();
}
export function tourStart(){go(0)}
export const tourActive=()=>I>=0;
export function tourMaybe(safe,had){
 if(safe||had||seen()||S.walls.length||S.areas.length)return false;
 setTimeout(tourStart,700);return true;
}
export function wireTour(){
 const B=$("#tour");if(!B)return;B.hidden=true;
 B.addEventListener("click",e=>{
  const b=e.target.closest("[data-t]");if(!b)return;
  if(b.dataset.t==="end")tourEnd();else go(I+1);
 });
}
```

---

# المساعد الذكي (`js/ai/`)

<a id="f-js-ai-ctx-js"></a>

---

