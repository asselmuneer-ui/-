# -
مواعيد الأذان على حسب بلدك
<!doctype html>
<html lang="ar">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>مواقيت الصلاة — عالمي</title>
<style>
  :root{ --accent:#00695c; --bg:#f7f9f9; --card:#ffffff; --muted:#666;}
  body{font-family:system-ui,-apple-system,"Segoe UI",Roboto,"Noto Sans Arabic",Arial; background:var(--bg); 
       margin:0; padding:20px; direction:rtl; color:#111}
  .container{max-width:820px;margin:0 auto}
  header{display:flex;gap:12px;align-items:center;justify-content:space-between;margin-bottom:18px}
  h1{font-size:1.2rem;margin:0}
  .controls{display:flex;gap:8px;flex-wrap:wrap}
  input,button,select{padding:8px 10px;border:1px solid #ddd;border-radius:8px;font-size:0.95rem}
  button{background:var(--accent);color:#fff;border:none;cursor:pointer}
  .card{background:var(--card);border-radius:12px;padding:16px;margin-bottom:14px;box-shadow:0 6px 18px rgba(3,3,3,0.06)}
  .times{display:grid;grid-template-columns:repeat(3,1fr);gap:10px}
  .time-item{padding:10px;border-radius:8px;background:#fbfcfc;display:flex;flex-direction:column;align-items:center;justify-content:center}
  .time-item.current{border:2px solid var(--accent);box-shadow:0 6px 18px rgba(0,0,0,0.06)}
  .label{font-size:0.9rem;color:var(--muted);margin-bottom:6px}
  .tt{font-weight:600;font-size:1.1rem}
  .meta{display:flex;gap:12px;align-items:center;flex-wrap:wrap;margin-top:8px;color:var(--muted)}
  #countdown{font-weight:700;color:var(--accent)}
  footer{font-size:0.85rem;color:var(--muted);text-align:center;margin-top:8px}
  @media(max-width:640px){.times{grid-template-columns:repeat(2,1fr)}}
</style>
</head>
<body>
  <div class="container">
    <header>
      <h1>مواقيت الصلاة — على حسب مدينتك</h1>
      <div class="controls">
        <input id="city" placeholder="المدينة (مثال: Cairo)" aria-label="المدينة">
        <input id="country" placeholder="البلد (مثال: Egypt)" aria-label="البلد">
        <button id="fetchBtn">جلب المواقيت</button>
        <button id="locBtn" title="استخدام موقعك الجغرافي">استخدام موقعي</button>
      </div>
    </header>

    <div id="result" class="card" aria-live="polite" hidden>
      <div class="meta" id="placeLine"><strong id="placeName"></strong> • <span id="dateG"></span> • <span id="dateH"></span> • <span id="tz"></span></div>

      <div class="card" style="margin-top:12px">
        <div class="meta">الآن: <span id="nowTime"></span> • التالي: <span id="nextPrayer"></span> • يبقى <span id="countdown"></span></div>
      </div>

      <div style="margin-top:12px" class="times" id="timesGrid">
        <!-- عناصر المواقيت ستضاف هنا -->
      </div>
    </div>

    <div id="error" style="color:#b71c1c; margin-top:8px"></div>

    <footer>
      مبني باستخدام AlAdhan API — مجاني ولا يتطلب مفتاح. لديك خيار حساب المحسوب محلياً باستخدام مكتبة adhan.js إن رغبت. (مراجع في الأسفل)
    </footer>
  </div>

<script>
/*
  صفحة بسيطة لعرض مواقيت الصلاة بالاعتماد على AlAdhan API.
  ملاحظات تقنية:
  - عندما نستخدم موقع الجهاز نطلب الإحداثيات ثم نستخدم /timings endpoint.
  - عندما يدخل المستخدم مدينة/بلد نستخدم /timingsByCity.
  - لتحديد الصلاة الحالية ونحسب العد التنازلي نستخرج الوقت الحالي في توقيت المدينة باستخدام Intl.DateTimeFormat مع timeZone (وقت API ضمن الحقل meta.timezone).
*/

const fetchBtn = document.getElementById('fetchBtn');
const locBtn = document.getElementById('locBtn');
const cityIn = document.getElementById('city');
const countryIn = document.getElementById('country');
const result = document.getElementById('result');
const errorEl = document.getElementById('error');
const placeName = document.getElementById('placeName');
const dateG = document.getElementById('dateG');
const dateH = document.getElementById('dateH');
const tzEl = document.getElementById('tz');
const timesGrid = document.getElementById('timesGrid');
const nowTimeEl = document.getElementById('nowTime');
const nextPrayerEl = document.getElementById('nextPrayer');
const countdownEl = document.getElementById('countdown');

let currentTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
let countdownInterval = null;

fetchBtn.addEventListener('click', ()=> {
  const city = cityIn.value.trim();
  const country = countryIn.value.trim();
  if(!city || !country){
    errorEl.textContent = 'رجاءً أدخل اسم المدينة واسم البلد أو استخدم زر "استخدام موقعي".';
    return;
  }
  errorEl.textContent = '';
  getByCity(city, country);
});

locBtn.addEventListener('click', ()=> {
  errorEl.textContent = '';
  if(!navigator.geolocation){
    errorEl.textContent = 'تصفحك لا يدعم تحديد الموقع.';
    return;
  }
  navigator.geolocation.getCurrentPosition(pos => {
    const lat = pos.coords.latitude;
    const lon = pos.coords.longitude;
    getByCoordinates(lat, lon);
  }, err => {
    errorEl.textContent = 'لم نتمكن من الوصول للموقع: ' + (err.message || err.code);
  }, {enableHighAccuracy:false, timeout:10000});
});

async function getByCity(city, country){
  showLoading();
  try{
    const url = `https://api.aladhan.com/v1/timingsByCity?city=${encodeURIComponent(city)}&country=${encodeURIComponent(country)}&method=2`;
    const res = await fetch(url);
    const data = await res.json();
    if(data.code !== 200) throw new Error(data.status || 'خطأ من الخادم');
    renderResult(data.data);
  }catch(e){
    errorEl.textContent = 'خطأ أثناء جلب المواقيت: ' + e.message;
    hideLoading();
  }
}

async function getByCoordinates(lat, lon){
  showLoading();
  try{
    const url = `https://api.aladhan.com/v1/timings?latitude=${lat}&longitude=${lon}&method=2`;
    const res = await fetch(url);
    const data = await res.json();
    if(data.code !== 200) throw new Error(data.status || 'خطأ من الخادم');
    renderResult(data.data);
  }catch(e){
    errorEl.textContent = 'خطأ أثناء جلب المواقيت: ' + e.message;
    hideLoading();
  }
}

function showLoading(){
  result.hidden = true;
  timesGrid.innerHTML = '';
  errorEl.textContent = 'جاري التحميل...';
}

function hideLoading(){
  errorEl.textContent = '';
}

function renderResult(data){
  hideLoading();
  result.hidden = false;
  // مكان و تواريخ
  const meta = data.meta || {};
  const timezone = meta.timezone || Intl.DateTimeFormat().resolvedOptions().timeZone;
  currentTimezone = timezone;
  tzEl.textContent = timezone;
  // place name best effort
  let place = '';
  if(data.meta && data.meta.timezone){ place = `${data.meta.latitude?.toFixed?.(2) || ''}, ${data.meta.longitude?.toFixed?.(2) || ''}` }
  if(data.address) place = data.address || place;
  // AlAdhan returns date object
  const dateReadable = (data.date && (data.date.readable || data.date.gregorian?.date)) || new Date().toLocaleDateString('ar-SA');
  const hijri = (data.date && (data.date.hijri?.date || data.date.hijri?.format)) || '--';
  placeName.textContent = (data.meta && data.meta.timezone) ? (data.data?.location?.city || place || 'موقع محدد') : (data.data?.location?.city || 'الموقع');
  dateG.textContent = dateReadable;
  dateH.textContent = 'هجري: ' + hijri;

  // مواقيت
  const timings = data.timings || data;
  const order = ['Fajr','Sunrise','Dhuhr','Asr','Maghrib','Isha'];
  const labels = {Fajr:'الفجر',Sunrise:'الشروق',Dhuhr:'الظهر',Asr:'العصر',Maghrib:'المغرب',Isha:'العشاء'};

  timesGrid.innerHTML = '';
  order.forEach(k=>{
    const t = timings[k];
    const div = document.createElement('div');
    div.className = 'time-item';
    div.id = 'time-'+k;
    div.innerHTML = `<div class="label">${labels[k]||k}</div><div class="tt">${t}</div>`;
    timesGrid.appendChild(div);
  });

  // الآن والعد التنازلي — نستخدم توقيت المنطقة مع Intl
  startClockAndCountdown(timings, timezone);
}

function startClockAndCountdown(timings, timezone){
  if(countdownInterval) clearInterval(countdownInterval);
  function tick(){
    // نأخذ الوقت الحالي في توقيت المكان كتوقيت hh:mm (24h)
    const nowParts = new Intl.DateTimeFormat('en-GB',{hour:'2-digit',minute:'2-digit',hour12:false,timeZone:timezone}).formatToParts(new Date());
    const nowStr = nowParts.filter(p=>p.type!=='literal').map(p=>p.value).join(':'); // "HH:MM"
    nowTimeEl.textContent = nowStr + ' (' + timezone + ')';

    // نحسب الصلاة التالية بإيجاد أول وقت أكبر من nowStr
    const prayerOrder = ['Fajr','Sunrise','Dhuhr','Asr','Maghrib','Isha'];
    let next = null;
    for(const p of prayerOrder){
      const t = timings[p]; // شكل "05:12" أو "05:12 (EET)" — نأخذ أول 5-6 أحرف hh:mm
      const hhmm = extractHHMM(t);
      if(compareTime(hhmm, nowStr) > 0){
        next = {key:p, time:hhmm};
        break;
      }
    }
    // إذا لم نجد (بعد العشاء) فالتالي هو فجر اليوم التالي
    if(!next){
      next = {key:'Fajr', time: extractHHMM(timings['Fajr']), nextDay:true};
    }

    nextPrayerEl.textContent = (next ? (labels[next.key]||next.key) + ' — ' + next.time : '--');
    // حساب الفرق بالدقائق/ثواني
    const diffMs = diffToTimeInTZ(next.time, timezone, !!next.nextDay);
    if(diffMs <= 0){
      countdownEl.textContent = 'قريباً';
    } else {
      countdownEl.textContent = msToHMS(diffMs);
    }

    // تمييز الصلاة الحالية (أقرب واحدة التي تحتوي now)
    highlightCurrentPrayer(timings, timezone);
  }

  tick();
  countdownInterval = setInterval(tick, 1000);
}

/* --- مساعدة زمنية --- */
function extractHHMM(t){
  if(!t) return '00:00';
  const m = t.match(/(\d{1,2}:\d{2})/);
  return m ? padHHMM(m[1]) : '00:00';
}
function padHHMM(hhmm){
  const [h,m]=hhmm.split(':').map(s=>s.padStart(2,'0'));
  return `${h}:${m}`;
}
// مقارنة كسلاسل "HH:MM"
function compareTime(a,b){
  if(a===b) return 0;
  return a > b ? 1 : -1;
}
// نحسب الفرق من الآن (في نفس توقيت المنطقة) إلى timeStr (HH:MM). إذا nextDay true نضيف يوم
function diffToTimeInTZ(timeStr, timeZone, nextDay=false){
  const [hh,mm] = timeStr.split(':').map(Number);
  // نكوّن تاريخ الآن في tz كـ components ثم نبني تاريخ UTC مقارب باستخدام toLocaleParts trick
  const now = new Date();
  // نحصل على أجزاء التاريخ في المنطقة المطلوبة
  const parts = new Intl.DateTimeFormat('en-GB',{
    timeZone: timeZone,
    year:'numeric',month:'numeric',day:'numeric',hour:'2-digit',minute:'2-digit',second:'2-digit',hour12:false
  }).formatToParts(now).reduce((acc,p)=>{acc[p.type]=p.value;return acc},{});
  const year = Number(parts.year), month = Number(parts.month), day = Number(parts.day);
  // نُنشئ نص ISO لتلك المنطقة ثم نستخدم Date.parse مع timezone offset غير متاح. بدلاً من ذلك نستخدم Date.UTC مع افتراض أن القيم هي لتلك المنطقة،
  // ثم نحستخدم Date.toLocaleString مع timezone للتقريب. أسلوب بسيط: نحسب فرق بالثواني بين تمثيلي الآن في tz و الوقت الهدف في tz بواسطة إنشاء Date objects محلية
  // لإنشاء وقت الهدف في نفس tz، نبني تاريخ محلي يمثل نفس سلسلة timezone باستخدام string و نخمن فرق بـ toLocaleString.
  // أبسط وموثوق هنا: نحسب لحظتي الآن وهدف كـ timestamps عبر تحويل كلٍ منهما إلى صيغة زمنية باستخدام Date.parse على صيغة "YYYY-MM-DDTHH:MM:SS" ثم إنشاء Date باستخدام toLocaleString trick ليس متاح. 
  // لذا سنستخدم طريقة عملية: نحصل على تمثيل UTC لوقت الآن ثم نححسب الفرق بين الآن في tz (باستخدام Intl) و target في tz.
  // لبساطة التنفيذ الآن: نأخذ الآن في tz كـ "HH:MM:SS" ونحول إلى دقائق من بداية اليوم ثم نفعل الحساب. هذا يعطينا فرقاً صحيحاً داخل نفس يوم/توقيت.
  const nowHH = Number(parts.hour), nowMM = Number(parts.minute), nowSS = Number(parts.second);
  let nowMinutes = nowHH*60 + nowMM + nowSS/60;
  let targetMinutes = hh*60 + mm;
  if(nextDay) targetMinutes += 24*60;
  let diffMinutes = targetMinutes - nowMinutes;
  if(diffMinutes < 0) diffMinutes += 24*60; // fallback
  return Math.round(diffMinutes*60*1000);
}

function msToHMS(ms){
  let s = Math.floor(ms/1000);
  const h = Math.floor(s/3600); s -= h*3600;
  const m = Math.floor(s/60); s -= m*60;
  return `${h}س ${m}د ${s}ث`;
}

function highlightCurrentPrayer(timings, timezone){
  const order = ['Fajr','Dhuhr','Asr','Maghrib','Isha'];
  // نجد الفترة الحالية بمقارنة الآن مع حدود الفترات: بين وقت الصلاة ووقت الصلاة التالية
  const nowStr = new Intl.DateTimeFormat('en-GB',{hour:'2-digit',minute:'2-digit',hour12:false,timeZone:timezone}).format(new Date());
  // ازالة جميع الهايلات
  document.querySelectorAll('.time-item').forEach(el=>el.classList.remove('current'));

  for(let i=0;i<order.length;i++){
    const p = order[i];
    const start = extractHHMM(timings[p]);
    // بداية = start، نهاية = وقت الصلاة التالية (أو نهاية اليوم بعد العشاء)
    const nextP = (i+1 < order.length) ? order[i+1] : 'Fajr';
    const end = extractHHMM(timings[nextP]);
    // مقارنة بسيطة: if now >= start && now < end then current
    if(compareTime(nowStr, start) >= 0 && compareTime(nowStr, end) < 0){
      const el = document.getElementById('time-'+p);
      if(el) el.classList.add('current');
      break;
    }
  }
}

/* labels used in multiple places */
const labels = {Fajr:'الفجر',Sunrise:'الشروق',Dhuhr:'الظهر',Asr:'العصر',Maghrib:'المغرب',Isha:'العشاء'};
</script>
</body>
</html>
