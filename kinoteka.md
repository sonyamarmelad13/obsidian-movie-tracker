```dataviewjs
// ═══ КИНОТЕКА РЕДАН v7 ═══

// ▼▼▼ УКАЖИ СВОЙ ПУТЬ К КАРТИНКЕ ▼▼▼
const HERO_IMAGE_PATH = "redan.png";
// ▲▲▲ относительно корня vault ▲▲▲

const STATUSES = ["ПРОСМОТРЕН","В БЭКЛОГЕ","СМОТРЮ","ОТЛОЖЕН","БРОШЕН"];
const STATUSES_YAML = {
  "ПРОСМОТРЕН":"Просмотрен","В БЭКЛОГЕ":"В бэклоге",
  "СМОТРЮ":"Смотрю","ОТЛОЖЕН":"Отложен","БРОШЕН":"Брошен"
};
const ST_CLASS = {
  "ПРОСМОТРЕН":"st-watched","В БЭКЛОГЕ":"st-backlog",
  "СМОТРЮ":"st-watching","ОТЛОЖЕН":"st-paused","БРОШЕН":"st-dropped"
};
const ST_SYM = {
  "ПРОСМОТРЕН":"[✓]","В БЭКЛОГЕ":"[—]",
  "СМОТРЮ":"[▶]","ОТЛОЖЕН":"[‖]","БРОШЕН":"[✕]"
};
const FAV_FIELD     = "Избранное";
const HISTORY_FIELD = "История просмотров";
const STALE_MONTHS  = 6;

function toStr(v) {
  if (v === null || v === undefined) return "";
  if (typeof v === "string") return v.trim();
  if (typeof v === "number") return String(v);
  if (typeof v === "object" && v.path) return v.path.trim();
  if (typeof v === "object" && v.display) return v.display.trim();
  return String(v).trim();
}

function todayISO() {
  const d = new Date();
  return d.getFullYear() + "-" + String(d.getMonth()+1).padStart(2,"0") + "-" + String(d.getDate()).padStart(2,"0");
}

// Получить URL картинки из vault для использования в CSS background
function getHeroImageUrl() {
  const file = app.vault.getAbstractFileByPath(HERO_IMAGE_PATH);
  if (file) return app.vault.getResourcePath(file);
  return null;
}

async function setStatus(filePath, newStatusNorm, opts={}) {
  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return;
  const yamlValue = STATUSES_YAML[newStatusNorm];
  if (!yamlValue) return;
  await app.fileManager.processFrontMatter(file, fm => {
    const wasWatched = (fm["Статус"] === "Просмотрен");
    const becomingWatched = (newStatusNorm === "ПРОСМОТРЕН");
    fm["Статус"] = yamlValue;
    if (becomingWatched) {
      fm["Дата просмотра"] = todayISO();
      if (!Array.isArray(fm[HISTORY_FIELD])) fm[HISTORY_FIELD] = [];
      const today = todayISO();
      const last = fm[HISTORY_FIELD][fm[HISTORY_FIELD].length - 1];
      if (last !== today || opts.forceDup) fm[HISTORY_FIELD].push(today);
    } else if (wasWatched) {
      fm["Дата просмотра"] = null;
    }
  });
}

async function markRewatch(filePath) {
  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return;
  await app.fileManager.processFrontMatter(file, fm => {
    fm["Статус"] = "Просмотрен";
    fm["Дата просмотра"] = todayISO();
    if (!Array.isArray(fm[HISTORY_FIELD])) fm[HISTORY_FIELD] = [];
    fm[HISTORY_FIELD].push(todayISO());
  });
}

async function toggleFavorite(filePath, currentVal) {
  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return !currentVal;
  const next = !currentVal;
  await app.fileManager.processFrontMatter(file, fm => { fm[FAV_FIELD] = next; });
  return next;
}

async function deleteFile(filePath) {
  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return;
  await app.vault.trash(file, true);
}

function openKinopoisk() {
  const candidates = [
    "unofficial-kinopoisk:open-kinopoisk-modal",
    "unofficial-kinopoisk:search",
    "unofficial-kinopoisk:open",
    "kinopoisk-search:open-search-modal",
    "kinopoisk-search:search",
  ];
  for (const id of candidates) {
    if (app.commands.commands[id]) { app.commands.executeCommandById(id); return; }
  }
  const pluginCmds = Object.keys(app.commands.commands).filter(k => k.includes("kinopoisk"));
  if (pluginCmds.length > 0) { app.commands.executeCommandById(pluginCmds[0]); return; }
  app.commands.executeCommandById("command-palette:open");
  new Notice("Введите 'kinopoisk' в палитре команд", 4000);
}

// ── Стили ────────────────────────────────────────
const styleId = "redan-kinoteka-v7";
if (document.getElementById(styleId)) document.getElementById(styleId).remove();
const style = document.createElement("style");
style.id = styleId;
style.textContent = `
  @import url('https://fonts.googleapis.com/css2?family=Unbounded:wght@700;900&family=Share+Tech+Mono&display=swap');

  .rk-root{background:#080808;border:1px solid #2a0000;padding:28px;color:#d0d0d0;font-family:'Share Tech Mono',monospace;position:relative;overflow:hidden;}
  .rk-root::before{content:'';position:absolute;inset:0;background-image:repeating-linear-gradient(0deg,transparent,transparent 49px,rgba(200,0,0,0.05) 50px),repeating-linear-gradient(90deg,transparent,transparent 49px,rgba(200,0,0,0.05) 50px);pointer-events:none;}
  .rk-scanline{position:absolute;inset:0;background:repeating-linear-gradient(0deg,transparent,transparent 2px,rgba(0,0,0,0.06) 2px,rgba(0,0,0,0.06) 4px);pointer-events:none;z-index:10;}

  /* HERO — фиксированная картинка, четко видна */
  .rk-hero{position:relative;height:280px;margin:-28px -28px 22px;overflow:hidden;border-bottom:1px solid #2a0000;background:#0a0000;}
  .rk-hero-bg{
    position:absolute;inset:0;
    background-size:cover;
    background-position:center 35%;
    background-repeat:no-repeat;
    /* Лёгкие эффекты для атмосферы, но картинка читается */
    filter:contrast(1.05) saturate(1.1) brightness(.85);
  }
  /* Тонкий сканлайн-overlay только на картинке */
  .rk-hero-bg::after{
    content:'';position:absolute;inset:0;
    background:repeating-linear-gradient(0deg,transparent,transparent 2px,rgba(0,0,0,0.18) 2px,rgba(0,0,0,0.18) 3px);
    pointer-events:none;
  }
  /* Красноватый VHS-оттенок */
  .rk-hero-bg::before{
    content:'';position:absolute;inset:0;
    background:radial-gradient(circle at 30% 50%,rgba(180,0,0,.08),transparent 60%);
    pointer-events:none;mix-blend-mode:screen;
  }
  /* Затемнение по краям, но не накрывает центр */
  .rk-hero::after{
    content:'';position:absolute;inset:0;z-index:1;pointer-events:none;
    background:
      linear-gradient(180deg,rgba(8,8,8,0) 50%,rgba(8,8,8,.7) 90%,#080808 100%),
      linear-gradient(90deg,rgba(8,8,8,.65) 0%,rgba(8,8,8,0) 30%,rgba(8,8,8,0) 70%,rgba(8,8,8,.45) 100%);
  }
  .rk-hero-content{position:absolute;inset:0;display:flex;align-items:flex-end;padding:24px 28px;z-index:2;flex-wrap:wrap;gap:14px;}
  .rk-hero-spider{font-size:42px;line-height:1;filter:grayscale(1) brightness(0.5) sepia(1) hue-rotate(320deg) saturate(10) drop-shadow(0 0 12px rgba(200,0,0,.6));}
  .rk-hero-title{font-family:'Unbounded',sans-serif;font-size:2.2em;font-weight:900;color:#ff1a1a;letter-spacing:.1em;text-transform:uppercase;text-shadow:0 0 24px rgba(255,30,30,0.6),0 0 2px #ff1a1a,0 4px 20px rgba(0,0,0,.95),0 0 40px rgba(0,0,0,.9);flex:1;line-height:1;}
  .rk-hero-current{font-size:.7em;color:#ff5555;letter-spacing:.12em;text-shadow:0 0 8px rgba(255,80,80,.5),0 2px 8px rgba(0,0,0,.95);max-width:100%;background:rgba(8,0,0,.65);padding:4px 10px;border-left:2px solid #cc0000;backdrop-filter:blur(2px);}
  .rk-hero-current .lbl{color:#883333;margin-right:8px;}

  .rk-hero-actions{display:flex;gap:10px;align-items:center;flex-wrap:wrap;}
  .rk-count{font-size:.82em;color:#ff4444;border:1px solid #550000;padding:5px 14px;background:rgba(17,0,0,.85);letter-spacing:.12em;text-shadow:0 0 8px rgba(255,60,60,0.4);backdrop-filter:blur(4px);}
  .rk-add-btn{display:flex;align-items:center;gap:7px;cursor:pointer;padding:10px 20px;background:rgba(21,0,0,.9);backdrop-filter:blur(6px);border:1px solid #cc0000;color:#ff3333;font-family:'Unbounded',sans-serif;font-size:.78em;font-weight:700;letter-spacing:.12em;text-transform:uppercase;transition:all .2s;text-shadow:0 0 8px rgba(255,50,50,0.5);}
  .rk-add-btn:hover{background:#200000;border-color:#ff2222;color:#ff6666;box-shadow:0 0 18px rgba(220,0,0,0.45);}
  .rk-rand-btn{display:flex;align-items:center;gap:7px;cursor:pointer;padding:10px 20px;background:rgba(21,12,0,.9);backdrop-filter:blur(6px);border:1px solid #886600;color:#ffcc00;font-family:'Unbounded',sans-serif;font-size:.78em;font-weight:700;letter-spacing:.12em;text-transform:uppercase;transition:all .2s;text-shadow:0 0 8px rgba(255,200,0,.5);}
  .rk-rand-btn:hover{border-color:#ffcc00;color:#fff;box-shadow:0 0 18px rgba(255,200,0,.4);transform:rotate(-3deg) scale(1.05);}

  /* СТАТИСТИКА */
  .rk-stats{display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:10px;margin-bottom:18px;}
  .rk-stat{padding:10px 14px;background:#0d0000;border:1px solid #2a0000;border-left:3px solid #cc0000;}
  .rk-stat-lbl{font-size:.62em;color:#883333;letter-spacing:.14em;text-transform:uppercase;margin-bottom:4px;}
  .rk-stat-val{font-size:1.3em;color:#fff;font-weight:700;text-shadow:0 0 8px rgba(255,255,255,.15);font-family:'Unbounded',sans-serif;}
  .rk-stat-sub{font-size:.65em;color:#664444;letter-spacing:.06em;margin-top:2px;}
  .rk-stat.fav{border-left-color:#cc9900;}
  .rk-stat.fav .rk-stat-val{color:#ffcc00;text-shadow:0 0 8px rgba(255,200,0,.3);}
  .rk-stat.watch{border-left-color:#ffaa00;}
  .rk-stat.backlog{border-left-color:#666;}
  .rk-stat.warm{border-left-color:#cc7700;}
  .rk-stat.warm .rk-stat-val{color:#ffaa00;}

  .rk-divider{border:none;border-top:1px solid #2a0000;margin:0 0 18px;}
  .rk-tabs{display:flex;gap:0;margin-bottom:18px;border-bottom:1px solid #2a0000;flex-wrap:wrap;}
  .rk-tab{cursor:pointer;padding:7px 20px;font-family:'Share Tech Mono',monospace;font-size:.8em;letter-spacing:.1em;color:#555;border-bottom:2px solid transparent;transition:all .15s;margin-bottom:-1px;text-transform:uppercase;}
  .rk-tab:hover{color:#cc2222;}
  .rk-tab.active{color:#ff3333;border-bottom-color:#cc0000;text-shadow:0 0 8px rgba(255,50,50,0.4);}
  .rk-tab-count{font-size:.75em;margin-left:5px;color:#440000;}
  .rk-tab.active .rk-tab-count{color:#882222;}

  .rk-filters{display:flex;flex-wrap:wrap;gap:10px;margin-bottom:18px;align-items:center;}
  .rk-label{font-size:.76em;text-transform:uppercase;letter-spacing:.14em;color:#993333;text-shadow:0 0 6px rgba(180,50,50,0.3);}
  .rk-search{background:#0d0000;border:1px solid #2a0000;color:#e0e0e0;padding:7px 14px;font-family:'Share Tech Mono',monospace;font-size:.82em;outline:none;min-width:180px;letter-spacing:.06em;}
  .rk-search::placeholder{color:#550000;}
  .rk-search:focus{border-color:#880000;color:#fff;}
  .rk-btn{cursor:pointer;padding:6px 16px;border:1px solid #220000;background:#0d0000;color:#888;font-size:.8em;font-family:'Share Tech Mono',monospace;letter-spacing:.08em;transition:all .15s;}
  .rk-btn:hover{border-color:#880000;color:#ff3333;}
  .rk-btn.active{background:#1e0000;border-color:#cc0000;color:#ff3333;box-shadow:inset 0 0 10px rgba(200,0,0,0.2);text-shadow:0 0 8px rgba(255,50,50,0.5);}
  .rk-select{background:#0d0000;border:1px solid #2a0000;color:#aaa;padding:6px 10px;font-family:'Share Tech Mono',monospace;font-size:.8em;cursor:pointer;letter-spacing:.06em;}
  .rk-select:focus{outline:none;border-color:#880000;color:#e0e0e0;}

  .rk-modes{display:flex;gap:0;margin-left:auto;}
  .rk-mode-btn{cursor:pointer;width:32px;height:32px;display:flex;align-items:center;justify-content:center;border:1px solid #220000;background:#0d0000;color:#555;font-size:13px;transition:all .15s;border-right:none;}
  .rk-mode-btn:last-child{border-right:1px solid #220000;}
  .rk-mode-btn:hover{color:#ff3333;border-color:#660000;}
  .rk-mode-btn.active{color:#ff3333;border-color:#cc0000;background:#1e0000;text-shadow:0 0 8px rgba(255,50,50,0.5);}

  .rk-sortbar{display:flex;gap:8px;margin-bottom:18px;align-items:center;flex-wrap:wrap;}
  .rk-sortlbl{font-size:.76em;color:#993333;letter-spacing:.12em;text-transform:uppercase;text-shadow:0 0 6px rgba(180,50,50,0.3);}
  .rk-sbtn{cursor:pointer;padding:5px 14px;border:1px solid #1a0000;background:#0a0000;color:#777;font-family:'Share Tech Mono',monospace;font-size:.8em;transition:all .15s;letter-spacing:.06em;}
  .rk-sbtn:hover{color:#cc2222;border-color:#440000;}
  .rk-sbtn.active{color:#ff3333;border-color:#770000;text-shadow:0 0 8px rgba(255,50,50,0.5);}

  .rk-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(180px,1fr));gap:14px;position:relative;}
  .rk-grid.compact{grid-template-columns:repeat(auto-fill,minmax(120px,1fr));gap:10px;}

  .rk-list{display:flex;flex-direction:column;gap:6px;}
  .rk-row{display:grid;grid-template-columns:50px 1fr 100px 80px 70px 80px 140px;gap:10px;align-items:center;padding:6px 12px;background:#0d0000;border:1px solid #1a0000;cursor:pointer;transition:all .15s;}
  .rk-row:hover{border-color:#880000;background:#150000;}
  .rk-row-poster{width:40px;height:60px;object-fit:cover;background:#1a0000;}
  .rk-row-name{color:#fff;font-size:.85em;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;}
  .rk-row-meta{font-size:.7em;color:#884444;}
  .rk-row-rating{color:#ffaa00;font-size:.78em;text-shadow:0 0 6px rgba(255,170,0,0.4);}
  .rk-row-fav{cursor:pointer;font-size:1.1em;color:#333;text-align:center;}
  .rk-row-fav.on{color:#ffcc00;text-shadow:0 0 8px rgba(255,200,0,0.6);}
  .rk-row-fav:hover{color:#cc9900;}
  .rk-row-status{font-size:.7em;letter-spacing:.06em;}

  .rk-card{position:relative;background:#0e0000;border:1px solid #220000;cursor:pointer;transition:border-color .2s,box-shadow .2s,transform .2s;overflow:visible;}
  .rk-card-inner{position:relative;overflow:hidden;background:#0e0000;}
  .rk-card-inner::after{content:'';position:absolute;bottom:0;right:0;width:14px;height:14px;background:linear-gradient(to top-left,#660000 0%,#660000 5px,transparent 5px);z-index:2;}
  .rk-card:hover{border-color:#aa0000;box-shadow:0 0 20px rgba(180,0,0,0.3),inset 0 0 20px rgba(180,0,0,0.07);transform:translateY(-4px);}
  .rk-card.is-series{border-color:#2a1500;}
  .rk-card.is-series:hover{border-color:#884400;box-shadow:0 0 20px rgba(180,100,0,0.25);}

  /* Когда дропдаун открыт — поднимаем карточку выше всех бейджей */
  .rk-card.dropdown-open{z-index:100;}

  .rk-poster-wrap{position:relative;}
  .rk-poster{width:100%;aspect-ratio:2/3;object-fit:cover;display:block;background:#0d0000;filter:sepia(.3) contrast(.85) brightness(.9);}
  .rk-ph{width:100%;aspect-ratio:2/3;display:flex;align-items:center;justify-content:center;font-size:3em;background:repeating-linear-gradient(45deg,#080000 0px,#080000 5px,#0e0000 5px,#0e0000 10px);color:#440000;}
  .rk-pgrad{position:absolute;bottom:0;left:0;right:0;height:50px;background:linear-gradient(to top,#0e0000,transparent);pointer-events:none;}

  .rk-ratingbadge{position:absolute;top:8px;left:8px;font-size:.7em;padding:3px 8px;background:rgba(0,0,0,0.9);border:1px solid #664400;color:#ffaa00;z-index:3;font-family:'Share Tech Mono',monospace;text-shadow:0 0 6px rgba(255,170,0,0.5);}
  .rk-stale{position:absolute;top:38px;left:8px;font-size:.7em;padding:3px 7px;background:rgba(0,0,0,0.9);border:1px solid #664400;color:#cc9900;z-index:3;font-family:'Share Tech Mono',monospace;text-shadow:0 0 6px rgba(200,140,0,.4);}
  .rk-rewatch{position:absolute;top:38px;left:8px;font-size:.62em;padding:2px 7px;background:rgba(0,0,0,0.9);border:1px solid #663300;color:#cc7700;z-index:3;font-family:'Share Tech Mono',monospace;letter-spacing:.06em;}
  /* Тип-бейдж — z-index низкий чтобы не перекрывал дропдаун */
  .rk-type-badge{position:absolute;bottom:8px;left:8px;font-size:.62em;padding:2px 7px;background:rgba(0,0,0,0.85);border:1px solid #2a1500;color:#885500;z-index:3;font-family:'Share Tech Mono',monospace;letter-spacing:.06em;transition:opacity .15s;}
  /* Когда дропдаун открыт — скрываем тип-бейдж */
  .rk-card.dropdown-open .rk-type-badge{opacity:0;pointer-events:none;}

  .rk-fav-btn{position:absolute;top:8px;right:8px;width:28px;height:28px;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,0.85);border:1px solid #330000;font-size:14px;cursor:pointer;z-index:10;transition:all .15s;color:#333;}
  .rk-fav-btn:hover{border-color:#886600;}
  .rk-fav-btn.on{color:#ffcc00;border-color:#886600;text-shadow:0 0 8px rgba(255,200,0,0.6);background:rgba(20,10,0,0.95);}

  .rk-info{padding:10px 12px 10px;position:relative;z-index:2;background:#0e0000;}
  .rk-card.compact .rk-info{padding:6px 8px;}
  .rk-card.compact .rk-name{font-size:.75em;}
  .rk-card.compact .rk-meta,.rk-card.compact .rk-genres,.rk-card.compact .rk-stars,.rk-card.compact .rk-rewatch-btn{display:none;}
  .rk-name{font-weight:700;font-size:.86em;margin-bottom:4px;color:#fff;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;letter-spacing:.04em;text-shadow:0 0 6px rgba(255,255,255,0.15);}
  .rk-meta{font-size:.72em;color:#884444;margin-bottom:6px;letter-spacing:.08em;}
  .rk-genres{display:flex;flex-wrap:wrap;gap:4px;margin-bottom:8px;}
  .rk-genre{font-size:.66em;padding:2px 8px;border:1px solid #440000;background:#110000;color:#cc4444;letter-spacing:.06em;text-transform:uppercase;}
  .rk-stars{display:flex;gap:2px;margin-bottom:10px;}
  .rk-star{color:#330000;font-size:.85em;}
  .rk-star.on{color:#ffaa00;text-shadow:0 0 6px rgba(255,170,0,0.5);}

  .rk-rewatch-btn{display:flex;align-items:center;justify-content:center;gap:6px;cursor:pointer;padding:5px 8px;margin-bottom:6px;border:1px solid #553300;background:#150a00;color:#cc7700;font-family:'Share Tech Mono',monospace;font-size:.66em;letter-spacing:.08em;text-transform:uppercase;transition:all .15s;}
  .rk-rewatch-btn:hover{border-color:#aa6600;color:#ffaa00;background:#1f1000;text-shadow:0 0 6px rgba(255,170,0,.4);}

  /* Статус-дропдаун — высокий z-index чтобы перекрыть бейджи на постере */
  .rk-status-row{position:relative;z-index:200;}
  .rk-status-cur{display:flex;align-items:center;justify-content:space-between;cursor:pointer;padding:6px 8px;border:1px solid #330000;background:#0d0000;font-family:'Share Tech Mono',monospace;font-size:.7em;letter-spacing:.06em;transition:all .15s;user-select:none;}
  .rk-status-cur:hover{border-color:#880000;background:#150000;}
  .rk-status-arrow{font-size:.75em;color:#660000;transition:transform .15s;margin-left:6px;}
  .rk-status-cur.open .rk-status-arrow{transform:rotate(180deg);color:#cc0000;}
  .rk-status-drop{display:none;position:absolute;left:0;right:0;bottom:calc(100% + 2px);background:#0d0000;border:1px solid #770000;z-index:99999;box-shadow:0 -4px 20px rgba(180,0,0,0.5),0 0 30px rgba(0,0,0,.8);}
  .rk-status-drop.visible{display:block;}
  .rk-status-opt{padding:7px 10px;font-size:.7em;letter-spacing:.06em;cursor:pointer;transition:background .1s;font-family:'Share Tech Mono',monospace;border-bottom:1px solid #1a0000;background:#0d0000;}
  .rk-status-opt:last-child{border-bottom:none;}
  .rk-status-opt:hover{background:#1e0000;}
  .rk-status-opt.cur{background:#160000;}

  @keyframes rk-glitch {
    0%   { transform: translate(0); text-shadow: 0 0 8px currentColor; }
    20%  { transform: translate(-2px, 1px); text-shadow: 2px 0 #00ffff, -2px 0 #ff0066, 0 0 8px currentColor; }
    40%  { transform: translate(2px, -1px); text-shadow: -2px 0 #00ffff, 2px 0 #ff0066, 0 0 8px currentColor; }
    60%  { transform: translate(-1px, 0); text-shadow: 2px 0 #00ffff, -1px 0 #ff0066, 0 0 8px currentColor; }
    80%  { transform: translate(1px, 1px); text-shadow: -2px 0 #00ffff, 1px 0 #ff0066, 0 0 8px currentColor; }
    100% { transform: translate(0); text-shadow: 0 0 8px currentColor; }
  }
  .rk-glitch { animation: rk-glitch .5s steps(5) 1; }

  /* ПАУК-КУРСОР */
  .rk-spider-cursor{position:fixed;width:32px;height:32px;pointer-events:none !important;z-index:9998;opacity:0;transition:opacity .25s;will-change:transform,left,top;left:-100px;top:-100px;}
  .rk-spider-cursor.visible{opacity:.85;}
  .rk-spider-cursor svg{width:100%;height:100%;filter:drop-shadow(0 0 6px rgba(220,0,0,.8));pointer-events:none !important;}

  /* CTX */
  .rk-ctx{position:fixed;background:#0d0000;border:1px solid #660000;box-shadow:0 8px 30px rgba(0,0,0,.7),0 0 16px rgba(180,0,0,.3);z-index:99999;min-width:210px;display:none;font-family:'Share Tech Mono',monospace;}
  .rk-ctx.visible{display:block;}
  .rk-ctx-item{padding:9px 14px;font-size:.78em;color:#ccc;cursor:pointer;border-bottom:1px solid #1a0000;letter-spacing:.06em;transition:background .1s,color .1s;display:flex;align-items:center;gap:10px;}
  .rk-ctx-item:last-child{border-bottom:none;}
  .rk-ctx-item:hover{background:#1e0000;color:#ff3333;}
  .rk-ctx-item.danger{color:#aa3333;}
  .rk-ctx-item.danger:hover{background:#2a0000;color:#ff5555;}
  .rk-ctx-item.warm{color:#cc7700;}
  .rk-ctx-item.warm:hover{background:#1f1000;color:#ffaa00;}
  .rk-ctx-icon{width:14px;text-align:center;color:#883333;}
  .rk-ctx-item:hover .rk-ctx-icon{color:#cc3333;}
  .rk-ctx-divider{height:1px;background:#1a0000;margin:0;}
  .rk-ctx-sub{position:relative;}
  .rk-ctx-sub::after{content:'▸';position:absolute;right:14px;top:50%;transform:translateY(-50%);color:#660000;font-size:.85em;}
  .rk-ctx-submenu{position:absolute;left:100%;top:0;background:#0d0000;border:1px solid #660000;display:none;min-width:180px;}
  .rk-ctx-sub:hover .rk-ctx-submenu{display:block;}

  .rk-saving{position:absolute;inset:0;background:rgba(0,0,0,0.8);display:flex;align-items:center;justify-content:center;font-family:'Share Tech Mono',monospace;font-size:.72em;color:#cc0000;letter-spacing:.1em;z-index:20;pointer-events:none;opacity:0;transition:opacity .2s;}
  .rk-saving.show{opacity:1;}

  .st-watched{color:#ff3333!important;border-color:#880000!important;text-shadow:0 0 8px rgba(255,50,50,0.5)!important;}
  .st-backlog{color:#888!important;border-color:#333!important;}
  .st-watching{color:#ffaa00!important;border-color:#775500!important;text-shadow:0 0 6px rgba(255,170,0,0.4)!important;}
  .st-paused{color:#ccaa00!important;border-color:#664400!important;}
  .st-dropped{color:#444!important;border-color:#222!important;}
  .rk-empty{grid-column:1/-1;text-align:center;padding:70px 0;color:#440000;font-size:.9em;letter-spacing:.12em;}
  .rk-empty .big{font-size:3em;margin-bottom:14px;}
`;
document.head.appendChild(style);

// Чистим осиротевших пауков и контекст-меню
document.querySelectorAll(".rk-spider-cursor, .rk-ctx").forEach(n => n.remove());

// ── Данные ───────────────────────────────────────
function hasTag(p, tag) {
  if (!p.tags) return false;
  if (Array.isArray(p.tags)) return p.tags.some(t => toStr(t) === tag);
  return toStr(p.tags) === tag;
}

const filmPages   = dv.pages('"Фильмы"').where(p => hasTag(p, "фильм")).array()
  .map(p => Object.assign({}, p, { _type: "film" }));
const seriesPages = dv.pages('"Фильмы"').where(p => hasTag(p, "сериал")).array()
  .map(p => Object.assign({}, p, { _type: "series" }));
const allPages = [...filmPages, ...seriesPages];

const allGenres = new Set();
allPages.forEach(p => {
  const g = p.Жанр;
  if (Array.isArray(g)) g.forEach(x => { const s = toStr(x); if (s) allGenres.add(s); });
  else if (g) { const s = toStr(g); if (s) allGenres.add(s); }
});

function calcStats() {
  const norm = s => toStr(s).toUpperCase();
  let watched = 0, backlog = 0, totalScore = 0, scored = 0, totalMinutes = 0, rewatches = 0;
  const genreCount = {};
  allPages.forEach(p => {
    const st = norm(p.Статус);
    if (st === "ПРОСМОТРЕН") watched++;
    if (st === "В БЭКЛОГЕ")  backlog++;
    if (p.Оценка) { totalScore += Number(p.Оценка); scored++; }
    if (st === "ПРОСМОТРЕН" && p.Продолжительность) totalMinutes += Number(p.Продолжительность) || 0;
    const gs = Array.isArray(p.Жанр) ? p.Жанр : (p.Жанр ? [p.Жанр] : []);
    gs.forEach(g => { const s = toStr(g); if (s) genreCount[s] = (genreCount[s]||0) + 1; });
    const hist = p[HISTORY_FIELD];
    if (Array.isArray(hist) && hist.length > 1) rewatches += hist.length - 1;
  });
  const avg = scored > 0 ? (totalScore / scored).toFixed(1) : "—";
  const topGenre = Object.entries(genreCount).sort((a,b)=>b[1]-a[1])[0];
  const hours = Math.round(totalMinutes / 60);
  return { watched, backlog, avg, scored, topGenre, hours, total: allPages.length, rewatches };
}

function isStale(p) {
  if (toStr(p.Статус).toUpperCase() !== "В БЭКЛОГЕ") return false;
  const created = p.file?.ctime;
  if (!created) return false;
  const d = new Date(created.ts || created);
  const months = (Date.now() - d.getTime()) / (1000*60*60*24*30);
  return months >= STALE_MONTHS;
}
function rewatchCount(p) {
  const hist = p[HISTORY_FIELD];
  if (Array.isArray(hist)) return Math.max(0, hist.length - 1);
  return 0;
}

let activeTab = "all", activeStatus = "ВСЕ", activeGenre = "ВСЕ";
let searchQuery = "", sortField = "Название", sortAsc = true;
let viewMode = "grid";

const root = this.container;
root.innerHTML = "";
const wrap = root.createEl("div", { cls: "rk-root" });
wrap.createEl("div", { cls: "rk-scanline" });

// HERO
const hero = wrap.createEl("div", { cls: "rk-hero" });
const heroBg = hero.createEl("div", { cls: "rk-hero-bg" });

// Устанавливаем фиксированную картинку из vault
const heroUrl = getHeroImageUrl();
if (heroUrl) {
  heroBg.style.backgroundImage = `url('${heroUrl}')`;
} else {
  heroBg.style.background = "linear-gradient(135deg,#0a0000,#1a0a1a,#0a0000)";
  console.warn("[Кинотека] Не найдена картинка по пути:", HERO_IMAGE_PATH);
}

const heroContent = hero.createEl("div", { cls: "rk-hero-content" });
heroContent.createEl("div", { cls: "rk-hero-spider", text: "🕷" });

const heroLeft = heroContent.createEl("div", { attr:{style:"flex:1;min-width:200px;"} });
heroLeft.createEl("div", { cls: "rk-hero-title", text: "КИНОТЕКА" });

const watchingNow = allPages.filter(p => toStr(p.Статус).toUpperCase() === "СМОТРЮ");
if (watchingNow.length > 0) {
  const cur = heroLeft.createEl("div", { cls: "rk-hero-current" });
  cur.createEl("span", { cls: "lbl", text: "// СМОТРЮ:" });
  cur.createEl("span", { text: watchingNow.map(p => toStr(p.Название)).slice(0,3).join(" • ") });
}

const heroActions = heroContent.createEl("div", { cls: "rk-hero-actions" });
const countBadge = heroActions.createEl("span", { cls: "rk-count" });

const randBtn = heroActions.createEl("button", { cls: "rk-rand-btn" });
randBtn.createEl("span", { text: "🎲" });
randBtn.createEl("span", { text: "ЧТО ПОСМОТРЕТЬ" });
randBtn.onclick = () => {
  const backlog = allPages.filter(p => toStr(p.Статус).toUpperCase() === "В БЭКЛОГЕ");
  if (!backlog.length) { new Notice("Бэклог пуст 🕸", 3000); return; }
  const pick = backlog[Math.floor(Math.random() * backlog.length)];
  new Notice("🎲 " + toStr(pick.Название), 4000);
  app.workspace.openLinkText(pick.file.path, "", false);
};

const addBtn = heroActions.createEl("button", { cls: "rk-add-btn" });
addBtn.createEl("span", { text: "＋" });
addBtn.createEl("span", { text: "ДОБАВИТЬ" });
addBtn.onclick = () => openKinopoisk();

const statsRow = wrap.createEl("div", { cls: "rk-stats" });
function renderStats() {
  statsRow.innerHTML = "";
  const s = calcStats();
  const items = [
    { cls: "watch",   lbl: "ПРОСМОТРЕНО",  val: s.watched, sub: s.total ? Math.round(s.watched/s.total*100) + "% от всего" : "" },
    { cls: "backlog", lbl: "В БЭКЛОГЕ",    val: s.backlog, sub: s.total ? Math.round(s.backlog/s.total*100) + "% от всего" : "" },
    { cls: "",        lbl: "СРЕДНЯЯ ОЦЕНКА", val: s.avg,   sub: s.scored + " оценено" },
    { cls: "",        lbl: "ЧАСОВ ПРОСМОТРА", val: s.hours, sub: "среди просмотренного" },
    { cls: "warm",    lbl: "ПЕРЕСМОТРОВ",  val: s.rewatches, sub: "повторных раз" },
    { cls: "fav",     lbl: "ТОП-ЖАНР",     val: s.topGenre ? s.topGenre[0] : "—", sub: s.topGenre ? s.topGenre[1] + " шт" : "" },
  ];
  items.forEach(it => {
    const st = statsRow.createEl("div", { cls: "rk-stat " + it.cls });
    st.createEl("div", { cls: "rk-stat-lbl", text: it.lbl });
    st.createEl("div", { cls: "rk-stat-val", text: String(it.val) });
    st.createEl("div", { cls: "rk-stat-sub", text: it.sub });
  });
}
renderStats();

wrap.createEl("hr", { cls: "rk-divider" });

const tabsEl = wrap.createEl("div", { cls: "rk-tabs" });
const tabDefs = [
  { id:"all",    label:"ВСЕ" },
  { id:"film",   label:"ФИЛЬМЫ" },
  { id:"series", label:"СЕРИАЛЫ" },
  { id:"fav",    label:"★ ИЗБРАННОЕ" },
];
const tabEls = {};
tabDefs.forEach(t => {
  const el = tabsEl.createEl("div", { cls: "rk-tab" + (t.id === "all" ? " active" : "") });
  el.createEl("span", { text: t.label });
  const cs = el.createEl("span", { cls: "rk-tab-count", text: "" });
  el.onclick = () => {
    activeTab = t.id;
    Object.values(tabEls).forEach(e => e.el.classList.remove("active"));
    el.classList.add("active");
    render();
  };
  tabEls[t.id] = { el, countSpan: cs };
});

const filterRow = wrap.createEl("div", { cls: "rk-filters" });
filterRow.createEl("span", { cls: "rk-label", text: "//" });
const searchEl = filterRow.createEl("input", { cls: "rk-search" });
searchEl.placeholder = "ПОИСК_"; searchEl.type = "text";
searchEl.oninput = () => { searchQuery = searchEl.value.toLowerCase(); render(); };

filterRow.createEl("span", { cls: "rk-label", text: "ST:" });
const statusBtns = {};
["ВСЕ", ...STATUSES].forEach(s => {
  const btn = filterRow.createEl("button", { cls: "rk-btn", text: s });
  if (s === "ВСЕ") btn.classList.add("active");
  statusBtns[s] = btn;
  btn.onclick = () => {
    activeStatus = s;
    Object.values(statusBtns).forEach(b => b.classList.remove("active"));
    btn.classList.add("active");
    render();
  };
});

filterRow.createEl("span", { cls: "rk-label", text: "GN:" });
const genreSelect = filterRow.createEl("select", { cls: "rk-select" });
["ВСЕ ЖАНРЫ", ...Array.from(allGenres).sort()].forEach(g => {
  const opt = genreSelect.createEl("option", { text: g });
  opt.value = g === "ВСЕ ЖАНРЫ" ? "ВСЕ" : g;
});
genreSelect.onchange = () => { activeGenre = genreSelect.value; render(); };

const modesEl = filterRow.createEl("div", { cls: "rk-modes" });
const modeDefs = [["grid","▦"],["compact","▤"],["list","☰"]];
const modeBtns = {};
modeDefs.forEach(([id,sym]) => {
  const b = modesEl.createEl("button", { cls: "rk-mode-btn" + (id===viewMode?" active":""), text: sym });
  b.title = id;
  modeBtns[id] = b;
  b.onclick = () => {
    viewMode = id;
    Object.values(modeBtns).forEach(x => x.classList.remove("active"));
    b.classList.add("active");
    render();
  };
});

const sortBar = wrap.createEl("div", { cls: "rk-sortbar" });
sortBar.createEl("span", { cls: "rk-sortlbl", text: "SORT:" });
["Название","Год","Рейтинг","Оценка","Дата просмотра"].forEach(f => {
  const btn = sortBar.createEl("button", { cls: "rk-sbtn", text: f });
  btn.dataset.field = f;
  if (f === sortField) btn.classList.add("active");
  btn.onclick = () => {
    if (sortField === f) sortAsc = !sortAsc;
    else { sortField = f; sortAsc = true; }
    sortBar.querySelectorAll(".rk-sbtn").forEach(b => {
      b.classList.remove("active"); b.textContent = b.dataset.field;
    });
    btn.classList.add("active");
    btn.textContent = f + (sortAsc ? "↑" : "↓");
    render();
  };
});

const grid = wrap.createEl("div", { cls: "rk-grid" });

// ── ПАУК-КУРСОР: создаём на body, ПОЗИЦИОНИРУЕМ через clientX/Y напрямую ──
// Но чистим при перерендере
const spiderEl = document.body.createDiv({ cls: "rk-spider-cursor rk-spider-current" });
spiderEl.dataset.kinotekaSpider = "1";
// Удаляем при следующем рендере все пауки кроме текущего
document.querySelectorAll(".rk-spider-cursor").forEach(n => { if (n !== spiderEl) n.remove(); });
spiderEl.innerHTML = `<svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#cc0000" stroke-width="1" fill="none" stroke-linecap="round">
    <circle cx="12" cy="12" r="3" fill="#660000"/>
    <circle cx="12" cy="9" r="2" fill="#880000"/>
    <line x1="9" y1="11" x2="3" y2="7"/>
    <line x1="9" y1="13" x2="2" y2="13"/>
    <line x1="9" y1="14" x2="3" y2="18"/>
    <line x1="10" y1="15" x2="7" y2="22"/>
    <line x1="15" y1="11" x2="21" y2="7"/>
    <line x1="15" y1="13" x2="22" y2="13"/>
    <line x1="15" y1="14" x2="21" y2="18"/>
    <line x1="14" y1="15" x2="17" y2="22"/>
  </g>
</svg>`;

let spiderTargetX = -100, spiderTargetY = -100;
let spiderX = -100, spiderY = -100;
let spiderActive = false;
let spiderRaf = null;
let spiderHideTimer = null;

function animateSpider() {
  spiderRaf = null;
  if (!spiderEl.isConnected) return;
  spiderX += (spiderTargetX - spiderX) * 0.2;
  spiderY += (spiderTargetY - spiderY) * 0.2;
  const angle = Math.atan2(spiderTargetY - spiderY, spiderTargetX - spiderX) * 180 / Math.PI + 90;
  spiderEl.style.left = spiderX + "px";
  spiderEl.style.top = spiderY + "px";
  spiderEl.style.transform = `translate(-50%, -50%) rotate(${angle}deg)`;
  const dist = Math.hypot(spiderTargetX-spiderX, spiderTargetY-spiderY);
  if (spiderActive || dist > 0.5) {
    spiderRaf = requestAnimationFrame(animateSpider);
  }
}

// ИСПРАВЛЕНО: паук теперь следует за реальной позицией мыши, а не правее карточки
function showSpider(mouseX, mouseY) {
  if (spiderHideTimer) { clearTimeout(spiderHideTimer); spiderHideTimer = null; }
  spiderTargetX = mouseX + 22;
  spiderTargetY = mouseY + 8;
  if (!spiderActive) {
    spiderX = spiderTargetX; spiderY = spiderTargetY;
    spiderActive = true;
    spiderEl.classList.add("visible");
  }
  if (!spiderRaf) animateSpider();
}

function hideSpider() {
  if (spiderHideTimer) clearTimeout(spiderHideTimer);
  spiderHideTimer = setTimeout(() => {
    spiderActive = false;
    spiderEl.classList.remove("visible");
  }, 80);
}

window.addEventListener("blur", hideSpider);
document.addEventListener("mouseleave", hideSpider);
document.addEventListener("mousemove", (e) => {
  if (!spiderActive) return;
  if (!e.target.closest(".rk-card") && !e.target.closest(".rk-row")) {
    hideSpider();
  }
});

// ── КОНТЕКСТНОЕ МЕНЮ ─────────────────────────────
const ctxMenu = document.body.createDiv({ cls: "rk-ctx" });
ctxMenu.dataset.kinotekaCtx = "1";
document.querySelectorAll(".rk-ctx").forEach(n => { if (n !== ctxMenu) n.remove(); });

function hideCtx() { ctxMenu.classList.remove("visible"); }
document.addEventListener("click", hideCtx);
document.addEventListener("scroll", hideCtx, true);

function showCtx(x, y, p, refs) {
  ctxMenu.innerHTML = "";

  function item(icon, label, onclick, klass) {
    const el = ctxMenu.createDiv({ cls: "rk-ctx-item" + (klass ? " " + klass : "") });
    el.createEl("span", { cls: "rk-ctx-icon", text: icon });
    el.createEl("span", { text: label });
    el.onclick = (e) => { e.stopPropagation(); hideCtx(); onclick(); };
    return el;
  }

  item("→", "Открыть заметку", () => app.workspace.openLinkText(p.file.path, "", false));

  const isWatched = toStr(p.Статус).toUpperCase() === "ПРОСМОТРЕН";
  if (isWatched) {
    item("👁", "Отметить пересмотр", async () => {
      try {
        await markRewatch(p.file.path);
        if (!Array.isArray(p[HISTORY_FIELD])) p[HISTORY_FIELD] = [];
        p[HISTORY_FIELD].push(todayISO());
        new Notice("👁 Пересмотр записан: " + todayISO(), 2500);
        renderStats();
        if (refs?.bumpRewatch) refs.bumpRewatch();
      } catch(err) { new Notice("Ошибка: " + err.message, 3000); }
    }, "warm");
  }

  const isFav = p[FAV_FIELD] === true;
  item("★", isFav ? "Убрать из избранного" : "В избранное", async () => {
    const next = await toggleFavorite(p.file.path, isFav);
    p[FAV_FIELD] = next;
    refs?.favBtn?.classList?.toggle("on", next);
    tabEls.fav.countSpan.textContent = allPages.filter(x => x[FAV_FIELD] === true).length;
    if (activeTab === "fav" && !next) refs?.card?.remove();
  });

  const subWrap = ctxMenu.createDiv({ cls: "rk-ctx-item rk-ctx-sub" });
  subWrap.createEl("span", { cls: "rk-ctx-icon", text: "◐" });
  subWrap.createEl("span", { text: "Изменить статус" });
  const sub = subWrap.createDiv({ cls: "rk-ctx-submenu" });
  STATUSES.forEach(s => {
    const it = sub.createDiv({ cls: "rk-ctx-item" });
    it.createEl("span", { cls: "rk-ctx-icon " + (ST_CLASS[s]||""), text: ST_SYM[s] || "•" });
    it.createEl("span", { text: s });
    it.onclick = async (e) => {
      e.stopPropagation();
      hideCtx();
      try {
        await setStatus(p.file.path, s);
        p.Статус = STATUSES_YAML[s];
        if (s === "ПРОСМОТРЕН") {
          if (!Array.isArray(p[HISTORY_FIELD])) p[HISTORY_FIELD] = [];
          p[HISTORY_FIELD].push(todayISO());
        }
        if (refs?.applyStatus) refs.applyStatus(s);
        renderStats();
      } catch(err) { new Notice("Ошибка: " + err.message, 3000); }
    };
  });

  ctxMenu.createDiv({ cls: "rk-ctx-divider" });

  item("✕", "Удалить", async () => {
    if (!confirm(`Удалить "${toStr(p.Название)}"? Файл уйдёт в корзину.`)) return;
    try {
      await deleteFile(p.file.path);
      refs?.card?.remove();
      const idx = allPages.indexOf(p);
      if (idx >= 0) allPages.splice(idx, 1);
      renderStats();
      new Notice("Удалено", 2000);
    } catch(err) { new Notice("Ошибка удаления: " + err.message, 3000); }
  }, "danger");

  // ИСПРАВЛЕНО: сначала показываем (display:block), потом измеряем
  ctxMenu.style.left = "0px";
  ctxMenu.style.top = "0px";
  ctxMenu.classList.add("visible");
  const rect = ctxMenu.getBoundingClientRect();
  let px = x + 2, py = y + 2;
  if (px + rect.width > window.innerWidth - 10) px = x - rect.width - 2;
  if (py + rect.height > window.innerHeight - 10) py = window.innerHeight - rect.height - 10;
  if (px < 0) px = 4;
  if (py < 0) py = 4;
  ctxMenu.style.left = px + "px";
  ctxMenu.style.top  = py + "px";
}

function closeAllDropdowns() {
  grid.querySelectorAll(".rk-status-drop.visible").forEach(d => {
    d.classList.remove("visible");
    const card = d.closest(".rk-card");
    if (card) card.classList.remove("dropdown-open");
    d.closest(".rk-status-row")?.querySelector(".rk-status-cur")?.classList.remove("open");
  });
}
document.addEventListener("click", closeAllDropdowns);

function render() {
  grid.innerHTML = "";
  grid.className = "rk-grid";
  if (viewMode === "compact") grid.classList.add("compact");
  if (viewMode === "list") grid.className = "rk-list";

  const norm = s => toStr(s).toUpperCase();

  const favCount = allPages.filter(p => p[FAV_FIELD] === true).length;
  tabEls.all.countSpan.textContent    = allPages.length;
  tabEls.film.countSpan.textContent   = filmPages.length;
  tabEls.series.countSpan.textContent = seriesPages.length;
  tabEls.fav.countSpan.textContent    = favCount;

  let pages = [...allPages];
  if (activeTab === "film")   pages = pages.filter(p => p._type === "film");
  if (activeTab === "series") pages = pages.filter(p => p._type === "series");
  if (activeTab === "fav")    pages = pages.filter(p => p[FAV_FIELD] === true);

  if (activeStatus !== "ВСЕ") pages = pages.filter(p => norm(p.Статус) === activeStatus);
  if (activeGenre  !== "ВСЕ") pages = pages.filter(p => {
    const g = p.Жанр; if (!g) return false;
    if (Array.isArray(g)) return g.some(x => toStr(x) === activeGenre);
    return toStr(g) === activeGenre;
  });
  if (searchQuery) pages = pages.filter(p =>
    norm(p.Название).includes(searchQuery.toUpperCase()) ||
    toStr(p.Режиссер?.path || p.Режиссер).toLowerCase().includes(searchQuery)
  );

  pages.sort((a, b) => {
    let va = a[sortField] ?? "", vb = b[sortField] ?? "";
    if (va === "") return 1; if (vb === "") return -1;
    va = toStr(va).toLowerCase(); vb = toStr(vb).toLowerCase();
    return sortAsc ? va.localeCompare(vb) : vb.localeCompare(va);
  });

  countBadge.textContent = pages.length + " ФАЙЛ" + (pages.length === 1 ? "" : "ОВ");

  if (!pages.length) {
    const e = grid.createEl("div", { cls: "rk-empty" });
    e.createEl("div", { cls: "big", text: activeTab === "fav" ? "★" : "🕸" });
    e.createEl("div", { text: activeTab === "fav" ? "// ИЗБРАННОЕ ПУСТО" : "// ДАННЫЕ НЕ НАЙДЕНЫ" });
    return;
  }

  if (viewMode === "list") renderList(pages);
  else renderGrid(pages);
}

function renderList(pages) {
  pages.forEach(p => {
    const isSeries = p._type === "series";
    const row = grid.createEl("div", { cls: "rk-row" });

    if (p.coverUrl) row.createEl("img", { cls: "rk-row-poster", attr: { src: p.coverUrl } });
    else row.createEl("div", { cls: "rk-row-poster", attr:{style:"display:flex;align-items:center;justify-content:center;color:#440000;"}, text: isSeries ? "📺" : "🕷" });

    const main = row.createEl("div");
    main.createEl("div", { cls: "rk-row-name", text: toStr(p.Название) || p.file.name });
    main.createEl("div", { cls: "rk-row-meta", text: [p.Год, toStr(p.Страна), isSeries ? "СЕРИАЛ":"ФИЛЬМ"].filter(Boolean).join(" · ") });

    row.createEl("div", { cls: "rk-row-meta", text: (Array.isArray(p.Жанр) ? p.Жанр : [p.Жанр]).filter(Boolean).slice(0,2).map(toStr).join(", ") });
    row.createEl("div", { cls: "rk-row-rating", text: p.Рейтинг ? "★ " + p.Рейтинг : "" });
    row.createEl("div", { cls: "rk-row-rating", attr:{style:"color:#fff;"}, text: p.Оценка ? "[" + p.Оценка + "]" : "" });

    let isFav = p[FAV_FIELD] === true;
    const favCell = row.createEl("div", { cls: "rk-row-fav" + (isFav ? " on" : ""), text: "★" });
    favCell.onclick = async (e) => {
      e.stopPropagation();
      isFav = await toggleFavorite(p.file.path, isFav);
      p[FAV_FIELD] = isFav;
      favCell.classList.toggle("on", isFav);
      tabEls.fav.countSpan.textContent = allPages.filter(x => x[FAV_FIELD] === true).length;
      if (activeTab === "fav" && !isFav) row.remove();
    };

    const stCell = row.createEl("div", { cls: "rk-row-status " + (ST_CLASS[toStr(p.Статус).toUpperCase()] || "st-backlog") });
    stCell.textContent = (ST_SYM[toStr(p.Статус).toUpperCase()] || "") + " " + toStr(p.Статус).toUpperCase();

    row.onclick = (e) => {
      if (e.target.closest(".rk-row-fav")) return;
      app.workspace.openLinkText(p.file.path, "", false);
    };
    row.oncontextmenu = (e) => {
      e.preventDefault(); e.stopPropagation();
      showCtx(e.clientX, e.clientY, p, { card: row, favBtn: favCell, applyStatus: (s) => {
        stCell.className = "rk-row-status " + (ST_CLASS[s]||"st-backlog");
        stCell.textContent = (ST_SYM[s]||"") + " " + s;
      }});
    };
    // ИСПРАВЛЕНО: паук следует за курсором
    row.addEventListener("mousemove", (e) => showSpider(e.clientX, e.clientY));
    row.addEventListener("mouseleave", hideSpider);
  });
}

function renderGrid(pages) {
  pages.forEach(p => {
    const isSeries = p._type === "series";
    const card = grid.createEl("div", { cls: "rk-card" + (isSeries ? " is-series" : "") + (viewMode === "compact" ? " compact" : "") });
    const inner = card.createEl("div", { cls: "rk-card-inner" });

    inner.onclick = (e) => {
      if (e.target.closest(".rk-status-row") || e.target.closest(".rk-fav-btn") || e.target.closest(".rk-rewatch-btn")) return;
      app.workspace.openLinkText(p.file.path, "", false);
    };

    // ИСПРАВЛЕНО: паук теперь у курсора, не правее карточки
    card.addEventListener("mousemove", (e) => showSpider(e.clientX, e.clientY));
    card.addEventListener("mouseleave", hideSpider);

    const pw = inner.createEl("div", { cls: "rk-poster-wrap" });
    if (p.coverUrl) {
      const img = pw.createEl("img", { cls: "rk-poster", attr: { src: p.coverUrl, alt: toStr(p.Название) } });
      img.onerror = () => img.replaceWith(makePh(isSeries));
    } else {
      pw.appendChild(makePh(isSeries));
    }
    pw.createEl("div", { cls: "rk-pgrad" });

    if (p.Рейтинг) inner.createEl("div", { cls: "rk-ratingbadge", text: "★" + p.Рейтинг });

    let rewatchBadge = null;
    const rwCount = rewatchCount(p);
    if (rwCount > 0) {
      rewatchBadge = inner.createEl("div", { cls: "rk-rewatch", text: "👁 ×" + (rwCount + 1) });
    } else if (isStale(p)) {
      inner.createEl("div", { cls: "rk-stale", text: "⏳" });
    }

    inner.createEl("div", { cls: "rk-type-badge", text: isSeries ? "[ СЕРИАЛ ]" : "[ ФИЛЬМ ]" });

    let isFav = p[FAV_FIELD] === true;
    const favBtn = inner.createEl("div", { cls: "rk-fav-btn" + (isFav ? " on" : ""), text: "★" });
    favBtn.onclick = async (e) => {
      e.stopPropagation();
      try {
        isFav = await toggleFavorite(p.file.path, isFav);
        p[FAV_FIELD] = isFav;
        favBtn.classList.toggle("on", isFav);
        tabEls.fav.countSpan.textContent = allPages.filter(x => x[FAV_FIELD] === true).length;
        if (activeTab === "fav" && !isFav) card.remove();
      } catch(err) { console.error(err); }
    };

    const saving = inner.createEl("div", { cls: "rk-saving", text: "// СОХРАНЕНИЕ..." });

    const info = card.createEl("div", { cls: "rk-info" });
    info.createEl("div", { cls: "rk-name", text: toStr(p.Название) || p.file.name });
    info.createEl("div", { cls: "rk-meta", text: [p.Год, toStr(p.Страна)].filter(Boolean).join(" // ") });

    const gr = info.createEl("div", { cls: "rk-genres" });
    const genres = Array.isArray(p.Жанр) ? p.Жанр : (p.Жанр ? [p.Жанр] : []);
    genres.slice(0, 3).forEach(g => {
      const s = toStr(g); if (!s) return;
      gr.createEl("span", { cls: "rk-genre", text: s });
    });

    if (p.Оценка) {
      const stars = info.createEl("div", { cls: "rk-stars" });
      const s5 = Math.round((Number(p.Оценка) / 10) * 5);
      for (let i = 1; i <= 5; i++)
        stars.createEl("span", { cls: "rk-star" + (i <= s5 ? " on" : ""), text: "★" });
    }

    let currentSt = norm2(p.Статус) || "В БЭКЛОГЕ";
    let rewatchBtn = null;

    function ensureRewatchBtn() {
      if (currentSt === "ПРОСМОТРЕН") {
        if (!rewatchBtn) {
          rewatchBtn = info.createEl("div", { cls: "rk-rewatch-btn" });
          rewatchBtn.createEl("span", { text: "👁" });
          rewatchBtn.createEl("span", { text: "ПЕРЕСМОТРЕЛ СЕГОДНЯ" });
          rewatchBtn.onclick = async (e) => {
            e.stopPropagation();
            saving.classList.add("show");
            try {
              await markRewatch(p.file.path);
              if (!Array.isArray(p[HISTORY_FIELD])) p[HISTORY_FIELD] = [];
              p[HISTORY_FIELD].push(todayISO());
              const newCount = rewatchCount(p);
              if (rewatchBadge) rewatchBadge.textContent = "👁 ×" + (newCount + 1);
              else if (newCount > 0) rewatchBadge = inner.createEl("div", { cls: "rk-rewatch", text: "👁 ×" + (newCount + 1) });
              new Notice("👁 " + toStr(p.Название) + " — пересмотр зафиксирован", 2500);
              renderStats();
              rewatchBtn.classList.remove("rk-glitch");
              void rewatchBtn.offsetWidth;
              rewatchBtn.classList.add("rk-glitch");
            } catch(err) {
              new Notice("Ошибка: " + err.message, 3000);
            } finally { saving.classList.remove("show"); }
          };
          info.insertBefore(rewatchBtn, info.querySelector(".rk-status-row"));
        }
      } else if (rewatchBtn) {
        rewatchBtn.remove();
        rewatchBtn = null;
      }
    }

    const statusRow = info.createEl("div", { cls: "rk-status-row" });
    const cur = statusRow.createEl("div", { cls: "rk-status-cur" });
    const curLabel = cur.createEl("span");
    curLabel.className = ST_CLASS[currentSt] || "st-backlog";
    curLabel.textContent = (ST_SYM[currentSt] || "") + " " + currentSt;
    cur.createEl("span", { cls: "rk-status-arrow", text: "▾" });
    const drop = statusRow.createEl("div", { cls: "rk-status-drop" });

    function applyStatusUI(s) {
      currentSt = s;
      curLabel.className = ST_CLASS[s] || "st-backlog";
      curLabel.textContent = (ST_SYM[s] || "") + " " + s;
      curLabel.classList.remove("rk-glitch");
      void curLabel.offsetWidth;
      curLabel.classList.add("rk-glitch");
      buildOptions();
      ensureRewatchBtn();
      const newCount = rewatchCount(p);
      if (newCount > 0) {
        if (rewatchBadge) rewatchBadge.textContent = "👁 ×" + (newCount + 1);
        else rewatchBadge = inner.createEl("div", { cls: "rk-rewatch", text: "👁 ×" + (newCount + 1) });
      }
      renderStats();
    }

    function buildOptions() {
      drop.innerHTML = "";
      STATUSES.forEach(s => {
        const isCur = (s === currentSt);
        const opt = drop.createEl("div", {
          cls: "rk-status-opt " + (ST_CLASS[s] || "") + (isCur ? " cur" : "")
        });
        const label = (isCur && s === "ПРОСМОТРЕН") ? "[👁] ПЕРЕСМОТР" : (ST_SYM[s] || "") + " " + s;
        opt.textContent = label;
        opt.onclick = async (e) => {
          e.stopPropagation();
          drop.classList.remove("visible"); cur.classList.remove("open");
          card.classList.remove("dropdown-open");
          if (isCur && s !== "ПРОСМОТРЕН") return;
          saving.classList.add("show");
          try {
            if (isCur && s === "ПРОСМОТРЕН") {
              await markRewatch(p.file.path);
              if (!Array.isArray(p[HISTORY_FIELD])) p[HISTORY_FIELD] = [];
              p[HISTORY_FIELD].push(todayISO());
              new Notice("👁 Пересмотр записан", 2500);
            } else {
              await setStatus(p.file.path, s);
              p.Статус = STATUSES_YAML[s];
              if (s === "ПРОСМОТРЕН") {
                if (!Array.isArray(p[HISTORY_FIELD])) p[HISTORY_FIELD] = [];
                p[HISTORY_FIELD].push(todayISO());
              }
            }
            applyStatusUI(s);
          } catch(err) {
            console.error(err);
            new Notice("Ошибка: " + err.message, 4000);
          } finally { saving.classList.remove("show"); }
        };
      });
    }
    buildOptions();
    ensureRewatchBtn();

    cur.onclick = (e) => {
      e.stopPropagation();
      const isOpen = drop.classList.contains("visible");
      closeAllDropdowns();
      if (!isOpen) {
        drop.classList.add("visible");
        cur.classList.add("open");
        card.classList.add("dropdown-open"); // ИСПРАВЛЕНО: скрываем тип-бейдж
      }
    };

    card.oncontextmenu = (e) => {
      e.preventDefault(); e.stopPropagation();
      showCtx(e.clientX, e.clientY, p, {
        card, favBtn,
        applyStatus: (s) => applyStatusUI(s),
        bumpRewatch: () => {
          const nc = rewatchCount(p);
          if (nc > 0) {
            if (rewatchBadge) rewatchBadge.textContent = "👁 ×" + (nc + 1);
            else rewatchBadge = inner.createEl("div", { cls: "rk-rewatch", text: "👁 ×" + (nc + 1) });
          }
        }
      });
    };
  });
}

function norm2(s) { return toStr(s).toUpperCase(); }
function makePh(isSeries) {
  const ph = document.createElement("div");
  ph.className = "rk-ph";
  ph.textContent = isSeries ? "📺" : "🕷";
  return ph;
}

render();
```
