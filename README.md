# spanish-learning-app
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
<title>西班牙語學習</title>
<!-- Tailwind CDN removed for better local-file compatibility on iPhone -->
<style>
  body{font-family:-apple-system,"PingFang HK",sans-serif;}
  .dark body{background:#0B141A;color:#F5F7F8;}
  .card{background:#fff;border-radius:1.1rem;box-shadow:0 2px 10px rgba(0,0,0,.08);padding:1rem;}
  .dark .card{background:#142028;box-shadow:0 2px 10px rgba(0,0,0,.4);}
  .btn{border-radius:1.1rem;padding:.75rem 1rem;font-weight:700;text-align:center;}
  .btn-p{background:#FF6B57;color:#fff;}
  .btn-s{background:#EAF7F6;color:#0B6E6E;}
  .dark .btn-s{background:#1C2B33;color:#5FC0B5;}
  .chip{display:inline-block;border-radius:999px;padding:.25rem .7rem;font-size:.7rem;font-weight:700;}
  .navbtn.active{color:#FF6B57;}
  .navbtn{color:#9aa3a8;}
  input,textarea{width:100%;border-radius:1rem;border:1px solid #e5e9eb;padding:.7rem 1rem;background:#fff;}
  .dark input,.dark textarea{background:#142028;border-color:#28333A;color:#fff;}
  #toast{position:fixed;bottom:90px;left:50%;transform:translateX(-50%);z-index:60;}
</style>
</head>
<body class="bg-orange-50 dark:bg-[#0B141A] text-gray-800">
<noscript>
  <div style="padding:40px 20px;text-align:center;font-family:sans-serif;color:#A82C1C;">
    <p style="font-size:40px;">⚠️</p>
    <p style="font-weight:bold;">此檢視器唔支援 JavaScript</p>
    <p style="font-size:14px;color:#666;">請用 Chrome 或 Safari 瀏覽器開啟呢個檔案。</p>
  </div>
</noscript>
<div id="app" class="max-w-md mx-auto min-h-screen pb-24"></div>
<nav class="fixed bottom-0 left-0 right-0 bg-white/95 dark:bg-[#142028]/95 border-t border-gray-100 dark:border-gray-700 max-w-md mx-auto">
  <div class="grid grid-cols-5 text-center text-[11px] font-bold">
    <button class="navbtn py-2.5 flex flex-col items-center" data-v="dashboard"><span class="text-xl">🏠</span>首頁</button>
    <button class="navbtn py-2.5 flex flex-col items-center" data-v="vocab"><span class="text-xl">📚</span>單字</button>
    <button class="navbtn py-2.5 flex flex-col items-center" data-v="review"><span class="text-xl">🔄</span>複習</button>
    <button class="navbtn py-2.5 flex flex-col items-center" data-v="mistakes"><span class="text-xl">📕</span>錯題</button>
    <button class="navbtn py-2.5 flex flex-col items-center" data-v="more"><span class="text-xl">⋯</span>更多</button>
  </div>
</nav>
<div id="toast" class="hidden bg-black text-white text-sm px-4 py-2 rounded-full"></div>

<script>
/* ===================== DATA LAYER (localStorage) ===================== */
const DB_KEY = "es_app_db_v1";
const SCHEDULE = [1,3,7,14,30,60]; // days

// Some in-app browsers (e.g. printer apps, embedded webviews) block
// localStorage on file:// pages. Fall back to an in-memory store so the
// app still works during this session even if data can't be saved.
let MEMORY_FALLBACK = null;
let STORAGE_OK = true;
try {
  localStorage.setItem("__test__","1");
  localStorage.removeItem("__test__");
} catch(e){ STORAGE_OK = false; }

function loadDB(){
  try{
    if(!STORAGE_OK) throw new Error("no storage");
    let raw = localStorage.getItem(DB_KEY);
    if(!raw){
      const db = {words:[], mistakes:[], logs:[], settings:{dark:false, streak:0, lastDate:null}};
      db.words = SEED.map((w,i)=>freshWord(w,"seed-"+i));
      localStorage.setItem(DB_KEY, JSON.stringify(db));
      return db;
    }
    return JSON.parse(raw);
  }catch(e){
    if(MEMORY_FALLBACK) return MEMORY_FALLBACK;
    const db = {words:[], mistakes:[], logs:[], settings:{dark:false, streak:0, lastDate:null}};
    db.words = SEED.map((w,i)=>freshWord(w,"seed-"+i));
    MEMORY_FALLBACK = db;
    return db;
  }
}
function saveDB(db){
  try{
    if(!STORAGE_OK) throw new Error("no storage");
    localStorage.setItem(DB_KEY, JSON.stringify(db));
  }catch(e){
    MEMORY_FALLBACK = db; // keep data alive for this session at least
  }
}
let DB = loadDB();
function persist(){ saveDB(DB); }

function freshWord(w, id){
  return Object.assign({
    id: id || ("w-"+Date.now()+"-"+Math.random().toString(36).slice(2,7)),
    pos:"", exampleEs:"", exampleZh:"", source:"", syllables:"", tip:"",
    interval:0, rep:0, ef:2.5, due:Date.now(), status:"new",
    correct:0, incorrect:0, createdAt:Date.now()
  }, w);
}

/* SM-2 lite */
function applySM2(w, quality){
  let ef = w.ef + (0.1 - (5-quality)*(0.08+(5-quality)*0.02));
  if(ef < 1.3) ef = 1.3;
  let rep = w.rep, interval;
  if(quality < 3){
    rep = 0; interval = SCHEDULE[0]; w.incorrect++;
  } else {
    w.correct++;
    interval = rep < SCHEDULE.length ? SCHEDULE[rep] : Math.round(w.interval*ef);
    rep++;
  }
  w.rep = rep; w.ef = ef; w.interval = interval;
  w.due = Date.now() + interval*86400000;
  w.status = rep===0 ? "learning" : (rep>=SCHEDULE.length && ef>=2.3 ? "mastered" : (rep>=2 ? "review":"learning"));
  return w;
}
function todayKey(){ return new Date().toISOString().slice(0,10); }

/* ===================== SEED DATA (user's Day 1-3 Duolingo words) ===================== */
const SEED = [
{spanish:"hola",meaningEn:"hello",meaningZh:"你好",pos:"感嘆詞",exampleEs:"¡Hola! ¿Cómo estás?",exampleZh:"你好！你怎麼樣？",syllables:"O-la",tip:"'h' 不發音！讀成「歐啦」，別讀成英文 hola 帶 h 音。"},
{spanish:"chao",meaningEn:"bye",meaningZh:"再見/拜拜",pos:"感嘆詞",exampleEs:"¡Chao, gracias!",exampleZh:"拜拜，謝謝！",syllables:"CHAO",tip:"發音似廣東話「炒」，輕快乾脆。"},
{spanish:"gracias",meaningEn:"thank you",meaningZh:"謝謝/唔該",pos:"感嘆詞",exampleEs:"No, gracias.",exampleZh:"不用了，謝謝。",syllables:"GRA-cias",tip:"重音在 'gra'，r 是單彈舌，不是英文捲舌音。"},
{spanish:"por favor",meaningEn:"please",meaningZh:"請/唔該",pos:"片語",exampleEs:"Un té, por favor.",exampleZh:"一杯茶，唔該。",syllables:"por fa-VOR",tip:"v 發成接近 b 的雙唇音，不要用英文咬唇 v。"},
{spanish:"y",meaningEn:"and",meaningZh:"和/而且",pos:"連接詞",exampleEs:"Café y té, por favor.",exampleZh:"咖啡和茶，謝謝。",syllables:"Y",tip:"短促讀「衣」，不要像英文 why 讀成雙元音。"},
{spanish:"o",meaningEn:"or",meaningZh:"或者/還是",pos:"連接詞",exampleEs:"¿Café o té?",exampleZh:"咖啡還是茶？",syllables:"O",tip:"乾脆「歐」音，不要像英文 or 那樣帶捲舌尾音。"},
{spanish:"un",meaningEn:"a/an/one",meaningZh:"一個（陽性）",pos:"冠詞",exampleEs:"Un taco, por favor.",exampleZh:"請給我一個塔可餅。",syllables:"UN",tip:"讀「溫」，尾音舌尖頂上牙齦。"},
{spanish:"agua",meaningEn:"water",meaningZh:"水",pos:"名詞",exampleEs:"Quiero agua, por favor.",exampleZh:"我想要水，謝謝。",syllables:"A-gua",tip:"讀「阿瓜」，重音在 a，要飽滿乾脆。"},
{spanish:"pan",meaningEn:"bread",meaningZh:"麵包",pos:"名詞",exampleEs:"Quiero un sándwich de pan.",exampleZh:"我想要這種麵包做的三明治。",syllables:"PAN",tip:"似廣東話「班」，不要讀成英文 pan（鍋）的扁音。"},
{spanish:"té",meaningEn:"tea",meaningZh:"茶",pos:"名詞",exampleEs:"Quiero té con azúcar.",exampleZh:"我想要加糖的茶。",syllables:"TÉ",tip:"t 不送氣，似廣東話「爹」，不要有英文噴氣感。"},
{spanish:"café",meaningEn:"coffee",meaningZh:"咖啡",pos:"名詞",exampleEs:"Un café por favor.",exampleZh:"一杯咖啡，謝謝。",syllables:"ca-FÉ",tip:"重音在最後 fé，c 不送氣似廣東話「加」。"},
{spanish:"helado",meaningEn:"ice cream",meaningZh:"冰淇淋/雪糕",pos:"名詞",exampleEs:"Un helado de chocolate.",exampleZh:"一個巧克力雪糕。",syllables:"e-LA-do",tip:"h 不發音！重音在 la，字尾 d 極輕。"},
{spanish:"sándwich",meaningEn:"sandwich",meaningZh:"三明治",pos:"名詞",exampleEs:"Un sándwich de pollo.",exampleZh:"一個雞肉三文治。",syllables:"SÁND-wich",tip:"重音在第一音節，ch 讀「奇」音。"},
{spanish:"taco",meaningEn:"taco",meaningZh:"塔可餅",pos:"名詞",exampleEs:"Quiero un taco por favor.",exampleZh:"請給我一個塔可餅。",syllables:"TA-co",tip:"t、c 都不送氣，似廣東話「打糕」。"},
{spanish:"vaso de",meaningEn:"glass of",meaningZh:"一杯（量詞）",pos:"片語",exampleEs:"Un vaso de agua, por favor.",exampleZh:"請給我一杯水。",syllables:"VA-so de",tip:"v 似 b 音，de 讀「爹」不是英文 dee。"},
{spanish:"con azúcar",meaningEn:"with sugar",meaningZh:"加糖",pos:"片語",exampleEs:"Un café con azúcar.",exampleZh:"一杯加糖的咖啡。",syllables:"con a-ZÚ-car",tip:"重音在 zú，字尾 r 輕彈舌，不是英文捲舌。"},
{spanish:"con leche",meaningEn:"with milk",meaningZh:"加牛奶",pos:"片語",exampleEs:"Café con leche, por favor.",exampleZh:"咖啡加奶，謝謝。",syllables:"con LE-che",tip:"ch 似廣東話「車」，不帶英文 sh 沙沙聲。"},
{spanish:"con hielo",meaningEn:"with ice",meaningZh:"加冰",pos:"片語",exampleEs:"Un café con hielo.",exampleZh:"一杯加冰的咖啡。",syllables:"con HIE-lo",tip:"h 不發音，ie 讀似中文「耶」。"},
{spanish:"quiero",meaningEn:"I want/love",meaningZh:"我想要/我愛",pos:"動詞",exampleEs:"Yo quiero un helado.",exampleZh:"我想要一個雪糕。",syllables:"QUIE-ro",tip:"qu 發 k 音，r 單彈舌不是英文捲舌。"},
{spanish:"quieres",meaningEn:"you want",meaningZh:"你想要",pos:"動詞",exampleEs:"¿Quieres un sándwich?",exampleZh:"你想要一個三明治嗎？",syllables:"QUIE-res",tip:"中間 r 單彈舌，別習慣性捲舌。"},
{spanish:"perfecto",meaningEn:"perfect",meaningZh:"完美/太好了",pos:"形容詞",exampleEs:"El café está perfecto.",exampleZh:"這杯咖啡太完美了。",syllables:"per-FEC-to",tip:"重音在 fec，t 不送氣似廣東話「打」。"}
];

/* ===================== PRONUNCIATION WARNING CONTENT ===================== */
const SOUNDS = [
{t:"單字母 R（單彈舌）",en:"英文母語者習慣把 r 讀成捲舌音（如 red）。",es:"西班牙語單 r 只是舌尖輕彈上顎一次，短而快。",ex:"pero（但是，單彈）vs perro（狗，雙彈）意思完全不同！"},
{t:"RR／字首 R（顫音）",en:"英文無此音，常被讀成捲舌或喉音代替。",es:"要連續彈舌2-3次，像引擎「嘟嘟嘟」，需要練習。",ex:"perro（狗）、Roma（羅馬）字首也要彈舌。"},
{t:"Ñ",en:"英文沒有此字母，常被讀成普通 n。",es:"舌面貼上顎的鼻音，似英文 canyon 中的 ny。",ex:"niño（小男孩）讀 NI-nyo，不是 NI-no。"},
{t:"J（及 ge/gi）",en:"英文 j 讀 /dʒ/（如 jump），常被誤用。",es:"是喉嚥摩擦音，似清喉嚨的「呵」氣聲。",ex:"jamón（火腿）讀近似「哈蒙」。"},
{t:"LL",en:"英文母語者讀成普通 l（如 llama 讀成 lama）。",es:"多數地區讀似英文 y（yes），不是 l。",ex:"llamar 讀近似「呀馬」。"},
{t:"母音 A E I O U",en:"英文母音常是雙元音滑音（go 其實是 goʊ）。",es:"五個母音永遠純粹短促不滑動：阿/誒/衣/噢/嗚。",ex:"no 讀短促「噢」，不要拖成英文 noʊ。"}
];

/* ===================== UTILS ===================== */
function speak(text, rate=0.85){
  if(!window.speechSynthesis) return;
  window.speechSynthesis.cancel();
  const u = new SpeechSynthesisUtterance(text);
  u.lang = "es-ES"; u.rate = rate;
  const voices = window.speechSynthesis.getVoices();
  const v = voices.find(v=>v.lang==="es-ES") || voices.find(v=>v.lang&&v.lang.startsWith("es"));
  if(v) u.voice = v;
  window.speechSynthesis.speak(u);
}
function norm(s){ return (s||"").toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g,"").trim(); }
function toast(msg){
  const t = document.getElementById("toast");
  t.textContent = msg; t.classList.remove("hidden");
  clearTimeout(window._toastTimer);
  window._toastTimer = setTimeout(()=>t.classList.add("hidden"), 1800);
}
function dueWords(){ return DB.words.filter(w=>w.due<=Date.now()).sort((a,b)=>a.due-b.due); }
function el(html){ const d=document.createElement("div"); d.innerHTML=html.trim(); return d.firstChild; }

/* ===================== ROUTER ===================== */
let VIEW = "dashboard";
let STATE = {}; // ephemeral per-view state
const app = document.getElementById("app");

function nav(v){ VIEW = v; STATE = {}; render(); }
document.querySelectorAll(".navbtn").forEach(b=>b.addEventListener("click", ()=>{
  if(b.dataset.v==="more"){ openMore(); return; }
  nav(b.dataset.v);
}));

function openMore(){
  app.appendChild(el(`
  <div id="moreSheet" class="fixed inset-0 bg-black/40 z-40 flex items-end">
    <div class="bg-white dark:bg-[#142028] rounded-t-3xl w-full max-w-md mx-auto p-5 pb-8">
      <div class="w-10 h-1.5 bg-gray-200 dark:bg-gray-600 rounded-full mx-auto mb-4"></div>
      <h3 class="font-bold text-lg mb-3">更多功能</h3>
      <div class="grid grid-cols-2 gap-3">
        <button data-v="pronunciation" class="card text-left morelink"><div class="text-2xl mb-1">🎙️</div><div class="font-bold text-sm">發音警告區</div></button>
        <button data-v="grammar" class="card text-left morelink"><div class="text-2xl mb-1">📐</div><div class="font-bold text-sm">文法</div></button>
        <button data-v="settings" class="card text-left morelink"><div class="text-2xl mb-1">⚙️</div><div class="font-bold text-sm">設定</div></button>
      </div>
    </div>
  </div>`));
  document.getElementById("moreSheet").addEventListener("click", (e)=>{
    if(e.target.id==="moreSheet") document.getElementById("moreSheet").remove();
  });
  document.querySelectorAll(".morelink").forEach(b=>b.addEventListener("click",()=>{
    nav(b.dataset.v); document.getElementById("moreSheet")?.remove();
  }));
}

function render(){
  document.querySelectorAll(".navbtn").forEach(b=>b.classList.toggle("active", b.dataset.v===VIEW));
  if(VIEW==="dashboard") return renderDashboard();
  if(VIEW==="vocab") return renderVocab();
  if(VIEW==="review") return renderReview();
  if(VIEW==="mistakes") return renderMistakes();
  if(VIEW==="pronunciation") return renderPronunciation();
  if(VIEW==="grammar") return renderGrammar();
  if(VIEW==="settings") return renderSettings();
}

function header(title, sub, backTo){
  return `<div class="sticky top-0 bg-orange-50/90 dark:bg-[#0B141A]/90 backdrop-blur px-4 pt-5 pb-3 flex items-center gap-3 z-10">
    ${backTo?`<button onclick="nav('${backTo}')" class="w-9 h-9 rounded-full bg-white dark:bg-[#142028] shadow flex items-center justify-center">←</button>`:""}
    <div><h1 class="font-extrabold text-xl">${title}</h1>${sub?`<p class="text-xs text-gray-400">${sub}</p>`:""}</div>
  </div>`;
}

/* ===================== DASHBOARD ===================== */
function renderDashboard(){
  const words = DB.words;
  const newCount = words.filter(w=>w.status==="new").length;
  const mastered = words.filter(w=>w.status==="mastered").length;
  const difficult = words.filter(w=>w.incorrect>=2 && w.status!=="mastered").length;
  const logs = DB.logs;
  const acc = logs.length ? Math.round(logs.filter(l=>l.correct).length/logs.length*100) : 0;
  const todayLogs = logs.filter(l=>l.date===todayKey());
  const streak = DB.settings.streak||0;
  app.innerHTML = `
  ${header("西班牙語學習")}
  <div class="px-4">
    <div class="flex items-center justify-between mb-3">
      <p class="text-sm text-gray-400">你好 👋</p>
      <div class="flex items-center gap-1 bg-yellow-100 dark:bg-yellow-900/30 rounded-full px-3 py-1">🔥 <b>${streak}</b></div>
    </div>
    <button onclick="nav('review')" class="btn btn-p w-full text-lg mb-4">開始今日複習 (${dueWords().length}) →</button>
    <div class="grid grid-cols-2 gap-3">
      ${stat("📝","今日已複習",todayLogs.length)}
      ${stat("✨","新單字",newCount)}
      ${stat("🏆","已掌握",mastered)}
      ${stat("🎯","準確率",acc+"%")}
      ${stat("🔥","連續天數",streak+" 天")}
      ${stat("⚠️","困難單字",difficult)}
    </div>
    <div class="card mt-4">
      <h3 class="font-bold text-sm mb-2">過去 7 天複習量</h3>
      ${weekBars()}
    </div>
    <div class="grid grid-cols-2 gap-3 mt-4">
      <button onclick="nav('pronunciation')" class="card text-left"><div class="text-2xl mb-1">🎙️</div><div class="font-bold text-sm">發音警告區</div></button>
      <button onclick="nav('grammar')" class="card text-left"><div class="text-2xl mb-1">📐</div><div class="font-bold text-sm">文法練習</div></button>
    </div>
  </div>`;
}
function stat(icon,label,val){
  return `<div class="card"><div class="text-lg mb-1">${icon}</div><div class="text-2xl font-extrabold">${val}</div><div class="text-xs text-gray-400">${label}</div></div>`;
}
function weekBars(){
  let html = '<div class="flex items-end gap-2 h-24">';
  for(let i=6;i>=0;i--){
    const d = new Date(); d.setDate(d.getDate()-i);
    const key = d.toISOString().slice(0,10);
    const n = DB.logs.filter(l=>l.date===key).length;
    const h = Math.min(100, n*12+4);
    html += `<div class="flex-1 flex flex-col items-center gap-1">
      <div class="w-full bg-teal-400 dark:bg-teal-600 rounded" style="height:${h}px"></div>
      <span class="text-[10px] text-gray-400">${"日一二三四五六"[d.getDay()]}</span>
    </div>`;
  }
  return html+'</div>';
}

/* ===================== VOCAB ===================== */
function renderVocab(){
  STATE.filter = STATE.filter || "all";
  STATE.q = STATE.q || "";
  const STATUS_LABEL = {new:"新",learning:"學習中",review:"複習中",mastered:"已掌握"};
  let words = DB.words.filter(w=>{
    if(STATE.filter!=="all" && w.status!==STATE.filter) return false;
    if(STATE.q && !(w.spanish.includes(STATE.q)||w.meaningZh.includes(STATE.q)||w.meaningEn.toLowerCase().includes(STATE.q.toLowerCase()))) return false;
    return true;
  }).sort((a,b)=>a.spanish.localeCompare(b.spanish));

  app.innerHTML = `
  ${header("單字庫","共 "+DB.words.length+" 個單字")}
  <div class="px-4">
    <div class="flex gap-2 mb-3">
      <input id="vq" placeholder="搜尋單字、意思…" value="${STATE.q}" class="flex-1">
      <button onclick="openUpload()" class="btn btn-s">📤</button>
      <button onclick="openAddWord()" class="btn btn-p">＋</button>
    </div>
    <div class="flex gap-2 overflow-x-auto mb-3 pb-1">
      ${["all","new","learning","review","mastered"].map(f=>`<button onclick="setVocabFilter('${f}')" class="chip shrink-0 ${STATE.filter===f?"bg-gray-800 text-white dark:bg-gray-200 dark:text-gray-900":"bg-gray-100 dark:bg-gray-700"}">${f==="all"?"全部":STATUS_LABEL[f]}</button>`).join("")}
    </div>
    <div class="flex flex-col gap-2">
      ${words.length===0?'<p class="text-center text-sm text-gray-400 py-12">沒有找到單字</p>':
      words.map(w=>`<button onclick="openWordDetail('${w.id}')" class="card text-left flex items-center justify-between">
        <div><div class="font-bold">${w.spanish} <span class="chip bg-teal-50 dark:bg-teal-900/30 text-teal-600 dark:text-teal-300">${STATUS_LABEL[w.status]}</span></div>
        <div class="text-xs text-gray-400">${w.meaningZh||w.meaningEn}</div></div>
        ${w.incorrect>=2?'<span class="text-orange-500">⚠️</span>':""}
      </button>`).join("")}
    </div>
  </div>`;
  document.getElementById("vq").addEventListener("input", e=>{ STATE.q=e.target.value; renderVocab(); document.getElementById("vq").focus();});
}
function setVocabFilter(f){ STATE.filter=f; renderVocab(); }

function openWordDetail(id){
  const w = DB.words.find(x=>x.id===id);
  showSheet("單字詳情", `
    <div class="card">
      <div class="flex justify-between items-start">
        <div>
          <span class="chip bg-teal-50 dark:bg-teal-900/30 text-teal-600 dark:text-teal-300">${w.pos||"其他"}</span>
          <h2 class="font-extrabold text-3xl mt-1">${w.spanish}</h2>
          <p class="text-sm text-gray-400">${w.meaningEn}</p>
          <p class="font-medium">${w.meaningZh}</p>
        </div>
        <button onclick="speak('${esc(w.spanish)}')" class="w-12 h-12 rounded-full bg-orange-50 dark:bg-orange-900/30 text-xl">🔊</button>
      </div>
      ${w.syllables?`<p class="text-sm mt-2"><b>音節：</b><span class="font-mono text-teal-600 dark:text-teal-300">${w.syllables}</span></p>`:""}
      ${w.exampleEs?`<div class="bg-orange-50 dark:bg-gray-700/40 rounded-xl p-3 mt-2">
        <p class="text-sm font-medium">${w.exampleEs} <button onclick="speak('${esc(w.exampleEs)}')" class="text-xs">🔊</button></p>
        <p class="text-xs text-gray-400">${w.exampleZh||""}</p></div>`:""}
      <div class="bg-orange-50 dark:bg-orange-900/20 rounded-xl p-3 mt-2">
        <p class="text-xs font-bold text-orange-600 dark:text-orange-300 mb-1">⚠️ 常見發音錯誤</p>
        <p class="text-xs">${w.tip || "留意母音要短促飽滿，避免用英文習慣加滑音尾巴。"}</p>
      </div>
    </div>
    <button onclick="deleteWord('${id}')" class="btn btn-s w-full mt-3 text-orange-500">刪除這個單字</button>
  `);
}
function deleteWord(id){
  DB.words = DB.words.filter(w=>w.id!==id);
  persist(); closeSheet(); renderVocab();
}
function esc(s){ return (s||"").replace(/'/g,"\\'"); }

function openAddWord(){
  showSheet("新增單字", `
    <div class="flex flex-col gap-3">
      <input id="aw_es" placeholder="西班牙文單字 *">
      <input id="aw_en" placeholder="英文意思">
      <input id="aw_zh" placeholder="中文意思">
      <input id="aw_ex" placeholder="例句（西班牙文）">
      <input id="aw_exzh" placeholder="例句翻譯（中文）">
      <button onclick="submitAddWord()" class="btn btn-p">新增到單字庫</button>
    </div>`);
}
function submitAddWord(){
  const es = document.getElementById("aw_es").value.trim();
  if(!es) return;
  const w = freshWord({
    spanish: es,
    meaningEn: document.getElementById("aw_en").value,
    meaningZh: document.getElementById("aw_zh").value,
    exampleEs: document.getElementById("aw_ex").value,
    exampleZh: document.getElementById("aw_exzh").value,
    source: "手動新增"
  });
  DB.words.push(w); persist(); closeSheet(); renderVocab(); toast("已新增單字！");
}

function openUpload(){
  showSheet("上傳單字檔案", `
    <p class="text-sm text-gray-500 mb-3">支援 <b>JSON</b>／<b>CSV</b>／<b>TXT</b>。JSON 範例欄位：word/spanish, meaning_en, meaning_zh, example_es, example_zh。</p>
    <input type="file" id="upfile" accept=".json,.csv,.txt" class="mb-2">
    <p id="upstatus" class="text-sm text-teal-600 dark:text-teal-300"></p>
  `);
  document.getElementById("upfile").addEventListener("change", handleUpload);
}
function handleUpload(e){
  const f = e.target.files[0];
  if(!f) return;
  const reader = new FileReader();
  reader.onload = ()=>{
    try{
      const rows = parseUploadText(f.name, reader.result);
      if(rows.length===0){ document.getElementById("upstatus").textContent = "沒有找到單字，請檢查格式。"; return; }
      rows.forEach(r=>{
        DB.words.push(freshWord({spanish:r.spanish, meaningEn:r.meaningEn||"", meaningZh:r.meaningZh||"",
          exampleEs:r.exampleEs||"", exampleZh:r.exampleZh||"", source:r.source||"上傳", tip:r.tip||""}));
      });
      persist();
      document.getElementById("upstatus").textContent = `成功匯入 ${rows.length} 個單字！`;
      toast(`匯入 ${rows.length} 個單字`);
      setTimeout(()=>{closeSheet(); renderVocab();}, 900);
    }catch(err){ document.getElementById("upstatus").textContent = "解析失敗：" + err.message; }
  };
  reader.readAsText(f);
}
function parseUploadText(filename, text){
  const ext = filename.split(".").pop().toLowerCase();
  const ALIAS = {word:"spanish",spanish:"spanish",es:"spanish",
    meaning_en:"meaningEn",english:"meaningEn",en:"meaningEn",meaning:"meaningEn",
    meaning_zh:"meaningZh",chinese:"meaningZh",zh:"meaningZh",cantonese:"meaningZh",
    example_es:"exampleEs",example:"exampleEs",sentence:"exampleEs",
    example_zh:"exampleZh",duolingo_source:"source",source:"source",unit:"source",
    pronunciation_tip_zh:"tip",pronunciation_tip:"tip"};
  function mapKey(k){ return ALIAS[k.trim().toLowerCase().replace(/\s+/g,"_")] || null; }
  if(ext==="json"){
    const data = JSON.parse(text);
    const arr = Array.isArray(data) ? data : (data.words||data.vocabulary||[]);
    return arr.map(raw=>{
      const row={};
      Object.entries(raw).forEach(([k,v])=>{ const m=mapKey(k); if(m && typeof v==="string") row[m]=v; });
      return row;
    }).filter(r=>r.spanish);
  }
  if(ext==="csv"){
    const lines = text.split(/\r?\n/).filter(l=>l.trim());
    const headers = lines[0].split(",").map(h=>mapKey(h.trim()));
    return lines.slice(1).map(line=>{
      const vals = line.split(",").map(v=>v.trim());
      const row={}; headers.forEach((h,i)=>{ if(h) row[h]=vals[i]; });
      return row;
    }).filter(r=>r.spanish);
  }
  // txt
  return text.split(/\r?\n/).filter(l=>l.trim()).map(line=>{
    let parts = line.includes("=")?line.split("="):line.includes(",")?line.split(","):[line];
    return {spanish:(parts[0]||"").trim(), meaningEn:(parts[1]||"").trim()};
  }).filter(r=>r.spanish);
}

/* ===================== REVIEW ===================== */
const MODES = ["flashcard","typing","multiple-choice","es-to-meaning"];
function renderReview(){
  if(!STATE.queue){
    const due = dueWords().slice(0,20);
    STATE.queue = due.map(w=>({word:w, mode: MODES[Math.floor(Math.random()*MODES.length)]}));
    STATE.idx = 0; STATE.correctCount=0; STATE.revealed=false; STATE.flipped=false; STATE.mc=null;
  }
  if(STATE.queue.length===0){
    app.innerHTML = header("每日複習","",null) + `<div class="px-4 pt-10 text-center"><p class="text-5xl mb-3">✅</p><p class="text-gray-500">目前沒有到期的單字，去學習頁新增更多單字吧！</p></div>`;
    return;
  }
  if(STATE.idx >= STATE.queue.length){
    app.innerHTML = header("每日複習","",null) + `<div class="px-4 pt-10 text-center"><p class="text-5xl mb-3">🎉</p><p class="font-bold text-xl">複習完成！</p>
    <p class="text-gray-500">答對 ${STATE.correctCount} / ${STATE.queue.length}</p>
    <button onclick="nav('dashboard')" class="btn btn-p w-full mt-4">返回首頁</button></div>`;
    return;
  }
  const item = STATE.queue[STATE.idx];
  const w = item.word;
  let body = "";
  if(item.mode==="flashcard"){
    body = `<button onclick="flipCard()" class="bg-orange-50 dark:bg-gray-700/50 rounded-2xl py-10 text-center w-full">
      ${!STATE.flipped?`<p class="font-extrabold text-3xl">${w.spanish}</p>`:`<p class="font-extrabold text-2xl text-teal-600 dark:text-teal-300">${w.meaningZh}</p><p class="text-sm text-gray-400">${w.meaningEn}</p>`}
      <p class="text-xs text-gray-400 mt-3">點擊翻面 ↻</p></button>`;
    if(STATE.flipped && !STATE.revealed){
      body += `<div class="grid grid-cols-2 gap-3 mt-3">
        <button onclick="submitReview(false)" class="btn btn-s">不記得 ✗</button>
        <button onclick="submitReview(true)" class="btn btn-p">記得 ✓</button></div>`;
    }
  } else if(item.mode==="typing"){
    body = `<p class="text-xs font-bold text-gray-400 uppercase mb-2">打字測驗：看意思拼出單字</p>
    <p class="text-center text-gray-600 dark:text-gray-300 py-2">${w.meaningZh} ${w.syllables?("("+w.syllables+")"):""}</p>
    <input id="revInput" placeholder="輸入西班牙文…" ${STATE.revealed?"disabled":""}>`;
  } else if(item.mode==="es-to-meaning"){
    body = `<p class="text-xs font-bold text-gray-400 uppercase mb-2">看西班牙文，輸入意思</p>
    <p class="text-center font-extrabold text-3xl py-3">${w.spanish} <button onclick="speak('${esc(w.spanish)}')" class="text-sm">🔊</button></p>
    <input id="revInput" placeholder="輸入意思…" ${STATE.revealed?"disabled":""}>`;
  } else { // multiple-choice
    if(!STATE.mc){
      const others = DB.words.filter(x=>x.id!==w.id).sort(()=>Math.random()-0.5).slice(0,3);
      STATE.mc = [w.meaningZh, ...others.map(o=>o.meaningZh)].sort(()=>Math.random()-0.5);
    }
    body = `<p class="text-xs font-bold text-gray-400 uppercase mb-2">選擇題：這是什麼意思？</p>
    <p class="text-center font-extrabold text-3xl py-3">${w.spanish}</p>
    <div class="flex flex-col gap-2">
      ${STATE.mc.map(opt=>`<button onclick="submitMC('${esc(opt)}')" ${STATE.revealed?"disabled":""} class="card text-left ${STATE.revealed&&opt===w.meaningZh?"border-2 border-teal-400":""}">${opt}</button>`).join("")}
    </div>`;
  }

  app.innerHTML = `
  ${header("每日複習", (STATE.idx+1)+" / "+STATE.queue.length, "dashboard")}
  <div class="px-4">
    <div class="w-full h-1.5 bg-gray-100 dark:bg-gray-700 rounded-full mb-4 overflow-hidden">
      <div class="h-full bg-orange-400" style="width:${((STATE.idx+1)/STATE.queue.length)*100}%"></div>
    </div>
    <div class="card flex flex-col gap-4">
      ${body}
      ${(!STATE.revealed && (item.mode==="typing"||item.mode==="es-to-meaning")) ? `<button onclick="submitTextReview()" class="btn btn-p w-full">提交答案</button>`:""}
      ${STATE.revealed ? `<div class="rounded-xl p-3 text-sm font-medium ${STATE.lastCorrect?"bg-teal-50 dark:bg-teal-900/30 text-teal-700 dark:text-teal-300":"bg-orange-50 dark:bg-orange-900/30 text-orange-600 dark:text-orange-300"}">
        ${STATE.lastCorrect?"✓ 答對了！":("✗ 答錯了。正確答案："+w.spanish+"（"+w.meaningZh+"）")}</div>
        <button onclick="nextReview()" class="btn btn-s w-full">下一題 →</button>` : ""}
    </div>
  </div>`;
}
function flipCard(){ STATE.flipped = true; renderReview(); }
function submitMC(opt){
  const w = STATE.queue[STATE.idx].word;
  submitReview(opt===w.meaningZh, opt);
}
function submitTextReview(){
  const val = document.getElementById("revInput").value;
  const item = STATE.queue[STATE.idx]; const w = item.word;
  let ok;
  if(item.mode==="typing") ok = norm(val)===norm(w.spanish);
  else ok = w.meaningZh.includes(val.trim()) || norm(w.meaningEn).includes(norm(val)) || norm(val).includes(norm(w.meaningEn));
  submitReview(ok, val);
}
function submitReview(correct, answer){
  const item = STATE.queue[STATE.idx]; const w = item.word;
  STATE.revealed = true; STATE.lastCorrect = correct;
  const quality = correct ? 4 : 2;
  applySM2(w, quality);
  DB.logs.push({wordId:w.id, mode:item.mode, correct, date:todayKey(), ts:Date.now()});
  if(!correct){
    DB.mistakes.push({id:"m-"+Date.now()+Math.random(), type:"vocabulary", spanish:w.spanish,
      userAnswer: answer||"(無作答)", correctAnswer:w.meaningZh, ts:Date.now(), resolved:false});
  } else { STATE.correctCount++; }
  // update streak
  const today = todayKey();
  if(DB.settings.lastDate !== today){
    const y = new Date(); y.setDate(y.getDate()-1);
    DB.settings.streak = (DB.settings.lastDate === y.toISOString().slice(0,10)) ? (DB.settings.streak||0)+1 : 1;
    DB.settings.lastDate = today;
  }
  persist();
  renderReview();
}
function nextReview(){
  STATE.idx++; STATE.revealed=false; STATE.flipped=false; STATE.mc=null;
  renderReview();
}

/* ===================== MISTAKES ===================== */
function renderMistakes(){
  STATE.tab = STATE.tab || "vocabulary";
  const TYPE_LABEL={vocabulary:"單字",grammar:"文法",pronunciation:"發音"};
  const list = DB.mistakes.filter(m=>m.type===STATE.tab).sort((a,b)=>b.ts-a.ts);
  app.innerHTML = `
  ${header("錯題本","共 "+DB.mistakes.length+" 筆")}
  <div class="px-4">
    <button onclick="weeklyReview()" class="btn btn-p w-full mb-4">🔁 開始錯題複習</button>
    <div class="grid grid-cols-3 gap-2 mb-4">
      ${["vocabulary","grammar","pronunciation"].map(t=>`<button onclick="setMistakeTab('${t}')" class="card ${STATE.tab===t?"border-2 border-teal-400":""}">
        <div class="text-xl">${t==="vocabulary"?"📚":t==="grammar"?"📐":"🎙️"}</div>
        <div class="text-xs font-bold">${TYPE_LABEL[t]}</div>
        <div class="text-[10px] text-gray-400">${DB.mistakes.filter(m=>m.type===t&&!m.resolved).length} 未解決</div></button>`).join("")}
    </div>
    <div class="flex flex-col gap-2">
      ${list.length===0?'<p class="text-center text-sm text-gray-400 py-12">這個分類目前沒有錯題</p>':
      list.map(m=>`<div class="card">
        <div class="flex justify-between items-start gap-2">
          <div class="flex-1">
            <p class="text-sm font-semibold">${m.spanish||m.prompt||""}</p>
            <p class="text-xs text-orange-500">你的答案：${m.userAnswer}</p>
            <p class="text-xs text-teal-600 dark:text-teal-300">正確答案：${m.correctAnswer}</p>
          </div>
          ${m.resolved?'<span class="chip bg-teal-50 dark:bg-teal-900/30 text-teal-600">已解決</span>':`<button onclick="resolveMistake('${m.id}')" class="chip bg-gray-100 dark:bg-gray-700">標記解決</button>`}
        </div></div>`).join("")}
    </div>
  </div>`;
}
function setMistakeTab(t){ STATE.tab=t; renderMistakes(); }
function resolveMistake(id){
  const m = DB.mistakes.find(x=>x.id===id); m.resolved=true; persist(); renderMistakes();
}
function weeklyReview(){
  const ids = new Set(DB.mistakes.filter(m=>!m.resolved && m.spanish).map(m=>m.spanish));
  DB.words.forEach(w=>{ if(ids.has(w.spanish)) w.due = Date.now(); });
  persist();
  nav("review");
}

/* ===================== PRONUNCIATION ===================== */
function renderPronunciation(){
  app.innerHTML = `
  ${header("發音警告區","廣東話／英文使用者常見錯誤","dashboard")}
  <div class="px-4 flex flex-col gap-3">
    ${SOUNDS.map(s=>`<div class="card">
      <h3 class="font-bold mb-1">${s.t}</h3>
      <p class="text-xs text-orange-500 mb-1"><b>英文習慣：</b>${s.en}</p>
      <p class="text-xs text-teal-600 dark:text-teal-300 mb-1"><b>西班牙語規則：</b>${s.es}</p>
      <p class="text-xs text-gray-500"><b>例子：</b>${s.ex}</p>
    </div>`).join("")}
  </div>`;
}

/* ===================== GRAMMAR (lite) ===================== */
const GRAMMAR = [
{t:"名詞陰陽性",s:"西班牙語每個名詞都有陽性/陰性，影響冠詞和形容詞形式。多數 -o 結尾是陽性，-a 結尾是陰性。",ex:"el café（陽性）／la leche（陰性）"},
{t:"Ser vs Estar",s:"ser 用於本質/永久特質（身份、來源地）；estar 用於狀態/位置（暫時情況）。",ex:"Soy de Hong Kong（永久）／Estoy cansado（暫時）"},
{t:"現在時態",s:"-ar/-er/-ir 動詞有固定變化規律。querer：quiero, quieres, quiere, queremos, queréis, quieren。",ex:"Yo quiero un café."},
{t:"反身動詞",s:"反身代詞 me/te/se/nos/os/se 放在動詞之前，表示動作返回主語本身。",ex:"Me llamo Ana.（我叫 Ana）"}
];
function renderGrammar(){
  app.innerHTML = `${header("文法","核心概念精選","dashboard")}
  <div class="px-4 flex flex-col gap-3">
    ${GRAMMAR.map(g=>`<div class="card">
      <h3 class="font-bold mb-1">${g.t}</h3>
      <p class="text-sm text-gray-600 dark:text-gray-300 mb-2">${g.s}</p>
      <div class="bg-orange-50 dark:bg-gray-700/40 rounded-xl px-3 py-2 text-sm">${g.ex}</div>
    </div>`).join("")}
  </div>`;
}

/* ===================== SETTINGS ===================== */
function renderSettings(){
  app.innerHTML = `${header("設定","","dashboard")}
  <div class="px-4 flex flex-col gap-3">
    <div class="card flex items-center justify-between">
      <span class="font-medium">深色模式</span>
      <input type="checkbox" id="darkToggle" ${DB.settings.dark?"checked":""} class="w-5 h-5">
    </div>
    <button onclick="resetData()" class="btn btn-s text-orange-500">清除所有資料並重設</button>
    <p class="text-xs text-gray-400 text-center mt-2">所有資料只儲存在你的瀏覽器本機（localStorage），不會上傳到任何伺服器。</p>
  </div>`;
  document.getElementById("darkToggle").addEventListener("change", e=>{
    DB.settings.dark = e.target.checked; persist(); applyDark();
  });
}
function resetData(){
  if(confirm("確定要清除所有資料嗎？此動作無法復原。")){
    localStorage.removeItem(DB_KEY); DB = loadDB(); nav("dashboard");
  }
}
function applyDark(){
  document.documentElement.classList.toggle("dark", !!DB.settings.dark);
}

/* ===================== SHEET (modal) ===================== */
function showSheet(title, html){
  closeSheet();
  app.appendChild(el(`
  <div id="sheetOverlay" class="fixed inset-0 bg-black/40 z-40 flex items-end">
    <div class="bg-white dark:bg-[#142028] rounded-t-3xl w-full max-w-md mx-auto p-5 pb-8 max-h-[85vh] overflow-y-auto">
      <div class="w-10 h-1.5 bg-gray-200 dark:bg-gray-600 rounded-full mx-auto mb-4"></div>
      <div class="flex justify-between items-center mb-3">
        <h3 class="font-bold text-lg">${title}</h3>
        <button onclick="closeSheet()" class="w-8 h-8 rounded-full bg-gray-100 dark:bg-gray-700">✕</button>
      </div>
      <div>${html}</div>
    </div>
  </div>`));
  document.getElementById("sheetOverlay").addEventListener("click",(e)=>{
    if(e.target.id==="sheetOverlay") closeSheet();
  });
}
function closeSheet(){ document.getElementById("sheetOverlay")?.remove(); }

/* ===================== INIT ===================== */
window.addEventListener("error", function(e){
  const app = document.getElementById("app");
  if(app && !app.innerHTML.trim()){
    app.innerHTML = '<div style="padding:40px 20px;text-align:center;color:#A82C1C;font-family:sans-serif;">'+
      '<p style="font-size:40px;margin-bottom:10px;">⚠️</p>'+
      '<p style="font-weight:bold;margin-bottom:8px;">呢個瀏覽器無法正常顯示此 App</p>'+
      '<p style="font-size:14px;color:#666;">請用 Chrome 或 Safari 開啟此檔案（唔要用 HP / 印表機 App 或其他預覽工具）。</p></div>';
  }
});
try{
  applyDark();
  render();
}catch(e){
  document.getElementById("app").innerHTML = '<div style="padding:40px 20px;text-align:center;color:#A82C1C;font-family:sans-serif;">'+
    '<p style="font-size:40px;margin-bottom:10px;">⚠️</p>'+
    '<p style="font-weight:bold;margin-bottom:8px;">載入失敗</p>'+
    '<p style="font-size:14px;color:#666;">請用 Chrome 或 Safari 開啟此檔案。錯誤：'+(e&&e.message)+'</p></div>';
}
if(window.speechSynthesis) window.speechSynthesis.onvoiceschanged = ()=>{};
</script>
</body>
</html>