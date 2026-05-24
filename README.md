# 臺北市教育統計儀表板 — p5.js 手勢互動＋即時 OpenData 版　講義

> **檔案名稱**：`taipei-education-p5-gesture.html`
> **技術框架**：p5.js 1.9.3 × ml5.js 1.x HandPose × 臺北市資料大平臺 OpenData
> **適用對象**：創意程式設計學習者、教育資料分析人員、互動裝置開發者

---

## 目錄

1. [專案概述與特色](#1-專案概述與特色)
2. [OpenData 即時連線架構](#2-opendata-即時連線架構)
3. [BIG-5 CSV 解碼與解析](#3-big-5-csv-解碼與解析)
4. [動態資料載入流程](#4-動態資料載入流程)
5. [手勢辨識系統（ml5.js HandPose）](#5-手勢辨識系統ml5js-handpose)
6. [捏合位置鎖定機制](#6-捏合位置鎖定機制)
7. [動畫系統：0~1 正規化插值](#7-動畫系統01-正規化插值)
8. [畫面座標系統與版面設計](#8-畫面座標系統與版面設計)
9. [函式架構完整說明](#9-函式架構完整說明)
10. [互動功能：行政區篩選＋學年比較](#10-互動功能行政區篩選學年比較)
11. [視覺特效設計](#11-視覺特效設計)
12. [錯誤處理與備用資料機制](#12-錯誤處理與備用資料機制)
13. [延伸應用與挑戰練習](#13-延伸應用與挑戰練習)
14. [常見問題 FAQ](#14-常見問題-faq)
15. [附錄：API 速查與資料欄位](#15-附錄api-速查與資料欄位)

---

## 1. 專案概述與特色

本專案是以 p5.js 為核心開發的全螢幕互動式教育資料儀表板，整合三大技術：

| 技術層 | 工具 | 說明 |
|--------|------|------|
| **視覺渲染** | p5.js 1.9.3 | 所有圖形、文字、動畫均以 Canvas 2D 繪製 |
| **手勢辨識** | ml5.js HandPose | 偵測右手食指尖作為滑鼠游標，捏合觸發點擊 |
| **即時資料** | data.taipei OpenData | BIG-5 CSV 即時擷取，備用嵌入資料自動降級 |

### 1.1 主要功能一覽

```
┌─────────────────────────────────────────────────────────┐
│  臺北市教育統計儀表板                                      │
│                                                          │
│  ① 學年度選擇（103~113學年，動態延伸）                      │
│  ② 比較年度選擇（橘色系 Chip，疊加 ghost 輪廓比較）          │
│  ③ 五個 KPI 卡片（即時數字動畫）                           │
│                                                          │
│  ┌────────┬──────────┬──────────┐                        │
│  │🏫學校分佈│📈學生趨勢  │📚班級統計  │                       │
│  └────────┴──────────┴──────────┘                        │
│                                                          │
│  手勢游標 ──→ 紅色準星，三段動畫（移動/預備/捏合）            │
│  右下角 ──→ 攝影機預覽 + 手勢開關按鈕                       │
│  頁首 ──→ API 來源標示（即時 or 備用）                      │
└─────────────────────────────────────────────────────────┘
```

### 1.2 為何選用 p5.js（而非 Vue/React）？

| 需求 | p5.js 優勢 |
|------|-----------|
| 60fps 流暢動畫 | `draw()` 迴圈天然支援逐幀插值 |
| 手勢游標即時渲染 | 所有元素在同一 Canvas，無 DOM/Canvas 混合延遲 |
| 像素級視覺控制 | 自訂虛線、弧形進度條、光暈效果 |
| 動態按鈕佈局 | `buildButtons()` 依資料年度動態計算位置 |

---

## 2. OpenData 即時連線架構

### 2.1 使用的三個資料集

```javascript
const API_BASE = 'https://data.taipei/api/frontstage/tpeod/dataset/resource.download?rid=';
const API_RIDS = {
  schools:  '591e2881-c8cf-429b-a758-323322aeafbb', // 臺北市校數（113年）
  students: 'e9879dd5-0d52-48dc-83f6-62e6d13c48b6', // 臺北市國小學生數按年級分
  classes:  'ca5699b7-fe98-409b-847a-2b3252d9fe0d', // 臺北市各級學校班級數
};
```

| 資料集 | 更新頻率 | 年度範圍 | 主要欄位 |
|--------|---------|---------|---------|
| **臺北市校數** | 年度更新 | 最新年度（113年） | 學制、校數/總計、各行政區校數 |
| **國小學生數** | 年度更新 | 103~113學年度 | 學年度、各年級學生人數 |
| **各級班級數** | 年度更新 | 103~113學年度 | 高中職、國中、國小班級數 |

### 2.2 並行擷取策略

```javascript
// Promise.allSettled → 即使某一 API 失敗，其他仍繼續執行
const [schRes, stuRes, clsRes] = await Promise.allSettled([
  fetchBig5CSV(API_BASE + API_RIDS.schools),
  fetchBig5CSV(API_BASE + API_RIDS.students),
  fetchBig5CSV(API_BASE + API_RIDS.classes),
]);
```

**為何用 `allSettled` 而非 `all`？**

- `Promise.all`：任一失敗 → 全部失敗
- `Promise.allSettled`：各自判斷成功/失敗 → 可以「部分使用即時資料，部分使用備用」

```javascript
// 成功/失敗各自處理
if (schRes.status === 'fulfilled' && schRes.value.success) {
  parseSchoolsCSV(schRes.value.rows);      // 使用即時資料
  dataSource.schools = 'data.taipei API（即時）';
} else {
  DIST_DATA = [...FB_DIST.map(d => ({...d}))];  // 使用備用資料
  dataSource.schools = '備用嵌入資料';
}
```

### 2.3 資料來源狀態顯示

```javascript
// drawHeader() 中的即時狀態標示
const allLive = Object.values(dataSource).every(s => s.includes('即時'));
fill(...allLive ? C.teal : C.amber);
text(allLive ? '✓ 即時 API 資料' : '⚠ 部分使用備用資料', 36, 54);
```

---

## 3. BIG-5 CSV 解碼與解析

### 3.1 為何需要特別處理 BIG-5？

臺北市資料大平臺（data.taipei）的 CSV 檔案使用 **BIG-5 繁體中文編碼**（而非常見的 UTF-8）。瀏覽器的 `fetch()` 預設以 UTF-8 解讀，直接讀取會得到亂碼。

```javascript
// ❌ 錯誤做法（預設 UTF-8，中文會亂碼）
const text = await response.text();

// ✅ 正確做法（明確指定 BIG-5 解碼）
const buffer = await response.arrayBuffer();  // 取得原始位元組
const text   = new TextDecoder('big5').decode(buffer);  // BIG-5 解碼
```

### 3.2 fetchBig5CSV() 完整實作

```javascript
async function fetchBig5CSV(url) {
  const resp = await fetch(url);
  if (!resp.ok) throw new Error('HTTP ' + resp.status);

  // 取得原始位元組並以 BIG-5 解碼
  const buf  = await resp.arrayBuffer();
  const text = new TextDecoder('big5').decode(buf);

  // 切割成行（處理 Windows 
 與 Unix 
 換行）
  const lines = text.trim().split(/
?
/).filter(l => l.trim());
  if (lines.length < 2) throw new Error('空 CSV');

  // 解析 header 與 data rows
  const headers = parseCsvLine(lines[0]);
  const rows = lines.slice(1).map(l => {
    const cols = parseCsvLine(l);
    const obj = {};
    headers.forEach((h, i) => obj[h.trim()] = (cols[i] || '').trim());
    return obj;
  });

  return { success: true, headers, rows };
}
```

### 3.3 CSV 行解析（支援引號欄位）

```javascript
function parseCsvLine(line) {
  const result = [];
  let current = '', inQuotes = false;
  for (const ch of line) {
    if (ch === '"') {
      inQuotes = !inQuotes;           // 進入/離開引號模式
    } else if (ch === ',' && !inQuotes) {
      result.push(current);           // 遇到逗號且不在引號中 → 切割
      current = '';
    } else {
      current += ch;
    }
  }
  result.push(current);              // 最後一個欄位
  return result;
}
```

### 3.4 三個 CSV 的欄位結構

**校數 CSV**（行：5種學制 × 1年度）：

```
欄位索引  欄位名稱
  [0]    項次
  [1]    民國年（例：113）
  [2]    學制（幼兒園 / 國民小學 / 國民中學 / 高級中等學校 / 特殊學校）
  [3]    校數/總計
  [4]    校數/國立
  [5]    校數/市立
  [6]    校數/私立
  [7]    校數/松山區
  [8]    校數/信義區
  [9]    校數/大安區
  [10]   校數/中山區
  [11]   校數/中正區
  [12]   校數/大同區
  [13]   校數/萬華區
  [14]   校數/文山區
  [15]   校數/南港區
  [16]   校數/內湖區
  [17]   校數/士林區
  [18]   校數/北投區
```

**學生數 CSV**（行：11學年度）：

```
欄位索引  欄位名稱
  [0]    項次
  [1]    學年度（103 ~ 113）
  [2]    國小總計學生數
  [3]    國小一年級學生數
  [4]    國小二年級學生數
  [5]    國小三年級學生數
  [6]    國小四年級學生數
  [7]    國小五年級學生數
  [8]    國小六年級學生數
```

**班級數 CSV**（行：11學年度）：

```
欄位索引  欄位名稱
  [0]    項次
  [1]    民國年（103 ~ 113）
  [2]    高級中等學校（班級數）
  [3]    國民中學（班級數）
  [4]    國民小學（班級數）
```

### 3.5 解析策略：使用欄位索引而非欄位名稱

欄位名稱可能因政府更新而改變（例：「高中職班級數」改為「高級中等學校班級數」），使用欄位索引（`cols[2]`）比使用 header 名稱更穩健：

```javascript
function parseClassCSV(rows) {
  return rows.map(row => {
    const cols = Object.values(row);  // 取欄位值陣列
    return {
      year:   parseInt(cols[1]) || 0,  // 索引 1 = 民國年
      senior: parseInt(cols[2]) || 0,  // 索引 2 = 高中
      junior: parseInt(cols[3]) || 0,  // 索引 3 = 國中
      elem:   parseInt(cols[4]) || 0,  // 索引 4 = 國小
    };
  }).filter(r => r.year > 0)
    .sort((a, b) => a.year - b.year); // 確保按年度升序
}
```

### 3.6 校數 CSV 的學制辨識

學制名稱可能有細微差異（例：「高級中等學校」vs「高中職類」），用 **包含字串** 而非完全比對：

```javascript
const typeStr = String(cols[2] || '');
let key = '';
if (typeStr.includes('幼'))                          key = 'kinder';
else if (typeStr.includes('小') && !typeStr.includes('中')) key = 'elem';
else if (typeStr.includes('國中') || typeStr.includes('中學')) key = 'junior';
else if (typeStr.includes('高') || typeStr.includes('中等'))   key = 'senior';
else if (typeStr.includes('特'))                     key = 'special';
```

---

## 4. 動態資料載入流程

### 4.1 整體流程圖

```
setup()
  │
  ├─ 初始化 Canvas、攝影機、背景粒子
  │
  └─ loadOpenData()  ← 非同步執行（await）
       │
       ├─ 顯示載入畫面（draw() 中 dataReady=false 分支）
       │    └─ 旋轉粒子動畫 + 進度條
       │
       ├─ Promise.allSettled([三個 fetch]) ← 並行擷取
       │
       ├─ 解碼 + 解析每個 CSV
       │    ├─ parseSchoolsCSV() → 填充 DIST_DATA, SCHOOL_ALL
       │    ├─ parseStudentCSV() → 填充 STUDENT_DATA
       │    └─ parseClassCSV()  → 填充 CLASS_DATA
       │
       ├─ 依資料動態初始化狀態
       │    ├─ selectedYear = 最新學年（動態）
       │    ├─ selDistSet   = 全選行政區
       │    ├─ distAlpha    = 全 1.0（全亮）
       │    └─ distNorm, gradeNorm...= 全 0（待動畫填充）
       │
       ├─ buildButtons() ← 依實際年度範圍動態建立
       ├─ setTargets()   ← 設定動畫目標值
       │
       └─ dataReady = true  ← 解鎖主畫面
```

### 4.2 載入畫面（drawLoadingScreen）

```javascript
function drawLoadingScreen() {
  // 1. 標題
  text('臺北市教育統計儀表板', cx, cy - 90);

  // 2. 進度條
  fill(...C.txt3, 25);
  rect(bx, by, barW, 8, 4);        // 底軌道
  fill(...C.teal);
  rect(bx, by, barW * loadProg, 8, 4); // 進度填充

  // 3. 旋轉粒子動畫（8顆以 sin 控制明度）
  for (let i = 0; i < 8; i++) {
    const angle = (TWO_PI / 8) * i + frameCount * 0.06;
    fill(...C.teal, map(sin(angle - t), -1, 1, 50, 220));
    circle(cx + cos(angle) * 28, cy + 25 + sin(angle) * 28,
           map(sin(angle - t), -1, 1, 4, 11));
  }

  // 4. 狀態文字
  text(loadMsg, cx, cy + 65);
}
```

### 4.3 動態年度範圍

傳統寫法會假設資料固定從 103 到 113：

```javascript
// ❌ 舊版：硬編碼年度
const years = [103, 104, ..., 113];
const yearIdx = selectedYear - 103;

// ✅ 新版：動態讀取
const years = STUDENT_DATA.map(d => d.year);  // 從資料中取
function yearIdxOf(year) {
  return STUDENT_DATA.findIndex(d => d.year === year);
}
```

好處：當政府新增 114、115 學年資料時，年度 Chip 自動延伸，圖表自動納入新資料，**不需修改任何程式碼**。

### 4.4 動態按鈕寬度計算

```javascript
function buildButtons() {
  const years = STUDENT_DATA.map(d => d.year);

  // 自動計算每個 chip 的寬度，避免超出螢幕
  const bw = Math.min(52, Math.floor((width - 80) / (years.length * 1.12)));
  const bGap = Math.floor(bw * 0.1);
  const tw = years.length * (bw + bGap) - bGap;

  let sx = (width - tw) / 2;  // 水平置中
  years.forEach((y, i) => {
    yearBtns.push({ x: sx + i * (bw + bGap), y: Y0.yearStart + 4, w: bw, h: 28, year: y });
  });
}
```

---

## 5. 手勢辨識系統（ml5.js HandPose）

### 5.1 HandPose 模型初始化

```javascript
// preload() 中載入模型（p5.js 會等 preload 完成才執行 setup）
function preload() {
  try {
    handPose = ml5.handPose({ flipped: true });
    // flipped: true → 座標鏡射，右手食指出現在畫面右側（直覺操作）
  } catch(e) {
    // ml5 CDN 載入失敗時 handPose=null，退化為純滑鼠模式
  }
}

// setup() 中啟動攝影機與偵測
function setup() {
  if (handPose) {
    video = createCapture(VIDEO, () => {
      video.size(320, 240);
      video.hide();
      handPose.detectStart(video, results => { hands = results; });
      gestureReady = true;
      gestureMsg = '👆 食指移動 ｜ 捏合點擊';
    });
  }
}
```

### 5.2 手部關鍵點索引

MediaPipe HandPose 提供 21 個手部關鍵點（`keypoints[0]` ~ `keypoints[20]`）：

```
        8 (食指尖) ← 游標位置
        |
        7
        |
        6
        |
   4    5
  (拇指尖)
        |
        0 (手腕)
```

本專案使用：
- `keypoints[8]`：食指尖端（Index Finger Tip）→ 游標位置
- `keypoints[4]`：拇指尖端（Thumb Tip）→ 搭配食指計算捏合距離

### 5.3 座標映射：攝影機空間 → 畫面空間

```javascript
// flipped: true 後，ml5 已自動鏡射 x 座標
// 只需線性映射到 Canvas 尺寸
const rawX = map(idxTip.x, 0, camW, 0, width);   // camW=320
const rawY = map(idxTip.y, 0, camH, 0, height);  // camH=240
```

**鏡射的意義**：
- 不鏡射：舉起右手 → 食指出現在畫面**左側**（違反直覺）
- 鏡射後：舉起右手 → 食指出現在畫面**右側**（如照鏡子）

### 5.4 右手識別邏輯

```javascript
// 若偵測到多隻手，選 x 座標最大的食指（= 最右側 = 右手）
let hand = hands[0];
if (hands.length > 1) {
  hand = hands.reduce((best, h) =>
    h.keypoints[8].x > best.keypoints[8].x ? h : best
  );
}
```

---

## 6. 捏合位置鎖定機制

### 6.1 問題：捏合動作導致游標位移

當使用者準備點擊時，食指往拇指靠近（捏合動作），食指尖的位置**自然地往拇指方向移動**，導致點擊位置偏離原本指向的按鈕。

```
用戶意圖：點擊「113」年度按鈕
          ↓
          食指指向按鈕（fingerX = 640）
          ↓
          開始捏合，食指往下移動（fingerX → 580）
          ↓
          捏合觸發點擊（fingerX = 580）→ 點到旁邊按鈕！
```

### 6.2 解決方案：預備捏合位置鎖定

```
距離 > 65px  → 正常追蹤食指位置
距離 < 65px  → 進入「預備捏合」：鎖定當前位置（pinchLockX, pinchLockY）
距離 < 38px  → 「完全捏合」：用鎖定位置觸發點擊，忽略當前偏移位置
```

### 6.3 程式碼實作

```javascript
const PRE_THRESH = 65;  // 預備閾值（攝影機像素）
const ACT_THRESH = 38;  // 捏合閾值

// 計算食指與拇指距離
const dx = idxTip.x - thmTip.x;
const dy = idxTip.y - thmTip.y;
pinchDist = Math.sqrt(dx*dx + dy*dy);

// 進入預備區 → 鎖定位置
if (pinchDist < PRE_THRESH && !pinchApproaching && !isPinching) {
  pinchApproaching = true;
  pinchLockX = rawX;  // 鎖定！記住這個位置
  pinchLockY = rawY;
}

// 離開預備區 → 解除鎖定
if (pinchDist >= PRE_THRESH) {
  pinchApproaching = false;
  pinchLockX = -1; pinchLockY = -1;
}

// 游標顯示位置：預備/捏合中用鎖定位置，否則跟著食指
fingerX = (pinchApproaching || isPinching) && pinchLockX >= 0
  ? pinchLockX : rawX;

// 觸發點擊時也用鎖定位置
if (isPinching && !wasPinching && pinchDebounce <= 0) {
  const cx = pinchLockX >= 0 ? pinchLockX : fingerX;
  handleClick(cx, cy);
}
```

### 6.4 防連點冷卻（debounce）

```javascript
pinchDebounce = 50;  // 設定 50 幀冷卻（@60fps ≈ 0.83 秒）

// 每幀遞減
if (pinchDebounce > 0) pinchDebounce--;

// 必須等冷卻結束才能觸發下一次點擊
if (isPinching && !wasPinching && pinchDebounce <= 0) {
  handleClick(cx, cy);
  pinchDebounce = 50;
}
```

---

## 7. 動畫系統：0~1 正規化插值

### 7.1 設計原則

所有動畫值均以 **0~1 正規化值** 儲存，繪製時才換算為像素：

```
WRONG: 存像素高度 → 不同 scale 混用 → 動畫錯亂
RIGHT: 存 0~1 比例 → 唯一換算點在繪製函式 → 永遠正確
```

### 7.2 完整插值管線

```javascript
// setTargets() 中設定目標值（0~1）
gradeNormT = sd.g.map(v => v / maxGrade);  // 最大年級 = 1.0

// updateAnims() 中每幀插值
const e = 0.07;  // 插值速度
gradeNorm[i] += (gradeNormT[i] - gradeNorm[i]) * e;
// 每幀靠近目標 7%，自然產生緩動效果

// drawTabTrend() 中換算像素
const h = gradeNorm[i] * gbMaxH;  // gbMaxH = 可用高度的 55%
rect(x, gbBase - h, gbW, h);      // 繪製長條
```

### 7.3 多套插值陣列設計

```javascript
// 各功能使用獨立的插值陣列（互不干擾）
let schoolNorm  = new Array(5).fill(0);   // 學制長條（5個學制）
let distNorm    = new Array(12).fill(0);  // 行政區長條（12個）
let distAlpha   = new Array(12).fill(1);  // 行政區透明度（0.15~1.0）
let gradeNorm   = new Array(6).fill(0);   // 年級長條 - 主年度
let gradeNormC  = new Array(6).fill(0);   // 年級長條 - 比較年度
let classNorm   = new Array(3).fill(0);   // 班級數 - 主年度
let classNormC  = new Array(3).fill(0);   // 班級數 - 比較年度
```

### 7.4 插值速度對照表

| 插值速度（e） | 感受 | 應用場景 |
|-------------|------|---------|
| `0.04` | 非常緩慢（像雲漂浮） | 折線繪製進度 `lineDrawPct` |
| `0.07` | 流暢自然（本專案主要值） | 長條高度、KPI 數字、distNorm |
| `0.09` | 稍快 | 行政區透明度 `distAlpha` |

---

## 8. 畫面座標系統與版面設計

### 8.1 全域 Y 座標常數

```javascript
const Y0 = {
  hdrEnd:    63,   // 頁首底部（標題 + API 狀態）
  yearStart: 68,   // 學年度選擇列頂部
  yearEnd:   155,  // 學年度選擇列底部（主年 + 比較年 兩列）
  kpiStart:  160,  // KPI 卡片頂部
  kpiEnd:    202,  // KPI 卡片底部
  tabStart:  207,  // 頁籤列頂部
  tabEnd:    243,  // 頁籤列底部
  chartStart:248,  // 圖表區頂部（translate 基準）
};
```

### 8.2 圖表區座標轉換

```javascript
// draw() 中：圖表函式都在 translate 後執行
push();
translate(0, Y0.chartStart);  // 圖表 y=0 對應畫面 y=248
  if (activeTab === 0) drawTabSchool();
  // drawTabSchool 內的 y=100 對應畫面的 y=348
pop();
```

**重要提醒**：`mouseX`、`mouseY` 不受 `translate()` 影響，永遠是畫面絕對座標。因此碰撞偵測需加回 `Y0.chartStart`：

```javascript
// distChips 繪製位置（在 translate 後，相對座標）
const cy_relative = oy + 22 + ch.y;

// 滑鼠偵測（畫面絕對座標）
const cy_absolute = Y0.chartStart + oy + 22 + ch.y;
if (mouseY >= cy_absolute && ...) { /* 點擊 */ }
```

### 8.3 版面分割策略

```
左右分割（Tab 0 學校分佈）：
├── 左半（0 ~ width/2）: 學制垂直長條圖
└── 右半（width/2 ~ width）: 行政區水平長條圖

左右分割（Tab 1 學生趨勢）：
├── 左58%：折線趨勢圖（寬+時序全覽）
└── 右42%：年級長條圖（聚焦選取年）

左右分割（Tab 2 班級統計）：
├── 左61%：多折線趨勢圖
└── 右39%：選取年班級數垂直長條
```

---

## 9. 函式架構完整說明

### 9.1 函式清單與分類

```
資料擷取與解析
  fetchBig5CSV(url)             擷取 BIG-5 編碼 CSV
  parseCsvLine(line)            解析單行 CSV（支援引號）
  parseSchoolsCSV(rows)         校數 CSV → DIST_DATA, SCHOOL_ALL
  parseStudentCSV(rows)         學生數 CSV → STUDENT_DATA
  parseClassCSV(rows)           班級數 CSV → CLASS_DATA

初始化
  preload()                     載入 HandPose 模型
  setup()                       建立 Canvas、攝影機、啟動資料載入
  loadOpenData()                非同步資料載入主流程
  buildButtons()                依資料動態建立所有按鈕座標
  setTargets()                  設定動畫目標值

輔助計算
  yearIdxOf(year)               在 STUDENT_DATA 中找年度索引
  classIdxOf(year)              在 CLASS_DATA 中找年度索引
  updateSchoolFilter()          計算篩選後學制數量

每幀更新
  updateAnims()                 所有插值更新（+背景粒子+點擊特效）
  processGesture()              處理手勢（位置鎖定+捏合偵測）

繪製：全局 UI
  draw()                        主迴圈
  drawLoadingScreen()           載入中畫面
  drawBgParts()                 背景飄浮粒子
  drawHeader()                  頁首+API來源標示
  drawYearBar()                 學年度選擇列（主年+比較年）
  drawKPIs()                    KPI 卡片列
  drawTabBar()                  頁籤導覽列

繪製：頁籤內容
  drawTabSchool()               學校分佈（含行政區 Chip 篩選）
  drawTabTrend()                學生趨勢（動態年度折線+年級長條）
  drawTabClass()                班級統計（多折線+班級長條）

繪製：輔助
  drawDistrictChips(ox,oy,h)    行政區 Chip 選取器
  drawSmallBtn(x,y,w,h,lbl)    小型按鈕（全選/清除）
  drawFingerCursor()            三段式手勢游標
  drawCameraPreview()           右下角攝影機預覽
  drawGestureToggleBtn()        手勢開關按鈕
  drawTooltip()                 長條 hover 提示框
  drawClickFx()                 點擊粒子特效
  drawSourceNote()              右下角資料來源標注

互動
  updateHoverState(x, y)        滑鼠/手勢 共用 hover 偵測
  handleClick(x, y)             滑鼠/手勢 共用點擊處理
  addClickFx(x, y, col)         產生點擊粒子
  checkBarHits()                長條碰撞偵測（更新 tooltip）
  mouseMoved()                  滑鼠移動事件
  mousePressed()                滑鼠點擊事件
  windowResized()               視窗大小改變事件
```

### 9.2 handleClick() — 統一點擊入口

滑鼠點擊與手勢捏合**都呼叫同一個函式**，確保行為一致：

```javascript
function handleClick(x, y) {
  // 優先判斷：手勢開關按鈕
  const gb = gestureBtnRect;
  if (x >= gb.x && x <= gb.x + gb.w && y >= gb.y && y <= gb.y + gb.h) {
    gestureEnabled = !gestureEnabled;
    return;  // 切換後立即 return，不觸發其他按鈕
  }

  // 主年度 Chip
  yearBtns.forEach(b => {
    if (x >= b.x && x <= b.x + b.w && y >= b.y && y <= b.y + b.h && b.year !== selectedYear) {
      selectedYear = b.year;
      flashAlpha = 30;
      setTargets();
      addClickFx(b.x + b.w/2, b.y + b.h/2, C.teal);
    }
  });

  // 比較年 Chip（點同年 = 取消比較）
  cmpBtns.forEach(b => {
    if (x >= b.x && x <= b.x + b.w && y >= b.y && y <= b.y + b.h) {
      compareYear = (b.year === compareYear) ? null : b.year;
      setTargets();
    }
  });

  // 頁籤、行政區 Chip ... 以此類推
}

// 滑鼠：直接呼叫
function mousePressed() { handleClick(mouseX, mouseY); }

// 手勢：用鎖定位置呼叫
if (isPinching && !wasPinching && pinchDebounce <= 0) {
  handleClick(pinchLockX >= 0 ? pinchLockX : fingerX,
              pinchLockY >= 0 ? pinchLockY : fingerY);
}
```

---

## 10. 互動功能：行政區篩選＋學年比較

### 10.1 行政區多選篩選（Tab 0）

```javascript
// 選取狀態使用 Set（效能優於 Array.includes）
let selDistSet = new Set(DIST_DATA.map((_, i) => i));  // 初始全選

// 切換選取
function toggleDistrict(idx) {
  if (selDistSet.has(idx)) {
    selDistSet.delete(idx);
    distAlphaT[idx] = 0.15;  // 未選取 → 15% 透明度
  } else {
    selDistSet.add(idx);
    distAlphaT[idx] = 1.0;   // 選取 → 完全不透明
  }
  updateSchoolFilter();  // 重新計算篩選後學制數量
}
```

**視覺效果**：
- 選取中：全亮條形 + 青色標示圓點
- 未選取：15% 透明度（自然淡出）+ 斜線陰影紋理

### 10.2 學年比較模式

```javascript
// 比較年 Chip：點同年 = 取消
compareYear = (b.year === compareYear) ? null : b.year;

// setTargets() 中設定比較年的動畫目標
if (compareYear != null) {
  const ci = yearIdxOf(compareYear);
  gradeNormCT = STUDENT_DATA[ci].g.map(v => v / maxGrade);
  // 用相同基準（maxGrade）確保比較有意義
}
```

**比較年視覺**：
- 折線圖：橘色脈衝點 + 橘色垂直標線 + 帶狀高亮區間
- 長條圖：橘色虛線輪廓（ghost 條）疊在主年實心條上方
- 差值標籤：綠色（正）/ 紅色（負）顯示於條頂

### 10.3 setTargets() 的動態索引

```javascript
function setTargets() {
  const yi  = yearIdxOf(selectedYear);   // STUDENT_DATA 中的索引
  const cii = classIdxOf(selectedYear);  // CLASS_DATA 中的索引
  if (yi < 0 || cii < 0) return;         // 防止年度不在資料中

  const sd = STUDENT_DATA[yi];
  const cd = CLASS_DATA[cii];
  // ... 設定 kpiT, gradeNormT, classNormT ...
}
```

---

## 11. 視覺特效設計

### 11.1 三段式手勢游標

```
階段 1：正常移動（pinchDist > 65px）
  • 紅色準星 + 外光暈，跟隨食指

階段 2：預備捏合（65px > pinchDist > 38px）
  • 游標位置凍結在鎖定點
  • 收縮圓環（ring radius 從 44→20px 變小）
  • 弧形進度條（從 0→2π 填滿）
  • 「準備點擊」橘色標籤

階段 3：完全捏合（pinchDist < 38px）
  • 脈衝外環（sin 週期振動）
  • 「CLICK」紅色標籤
  • 粒子爆炸（addClickFx）
```

```javascript
// 進度值（0=正常, 1=完全捏合）
const prog = pinch ? 1 : appr ? constrain(map(pinchDist, 65, 38, 0, 1), 0, 1) : 0;

// 所有視覺元素都依 prog 插值
fill(200, 50, 50, lerp(20, 65, prog));     // 光暈透明度增加
circle(fingerX, fingerY, lerp(50, 80, prog)); // 光暈尺寸增大
```

### 11.2 背景飄浮粒子

```javascript
function mkBgPt() {
  return {
    x: random(width), y: random(height),
    vx: random(-0.15, 0.15), vy: random(-0.12, 0.08),
    r: random(2, 5),
    alpha: random(12, 45),                  // 最大透明度（很低，只是點綴）
    col: random([C.teal, C.blue, C.violet, C.coral]),
    life: random(0, 1),
    speed: random(0.002, 0.004),            // 週期速度
  };
}

// 透明度以 sin 週期控制（淡入淡出）
fill(...p.col, Math.sin(p.life * PI) * p.alpha);
// sin(0)=0 → 不可見; sin(PI/2)=1 → 最大; sin(PI)=0 → 不可見
```

### 11.3 點擊粒子爆炸

```javascript
function addClickFx(x, y, col) {
  for (let i = 0; i < 16; i++) {
    const angle = random(TWO_PI);    // 全方向隨機噴射
    const speed = random(1.5, 5);
    clickFx.push({
      x, y,
      vx: cos(angle) * speed,
      vy: sin(angle) * speed,
      r: random(2, 5),
      a: 220,  // 起始透明度
      col,
    });
  }
}

// 每幀物理更新
clickFx.forEach(p => {
  p.x += p.vx; p.y += p.vy;
  p.vy += 0.12;    // 重力（向下加速）
  p.vx *= 0.94; p.vy *= 0.94;  // 阻力（速度衰減）
  p.a  -= 7;       // 透明度衰減
  p.r  *= 0.97;    // 粒子縮小
});
clickFx = clickFx.filter(p => p.a > 4);  // 清理已消失粒子
```

---

## 12. 錯誤處理與備用資料機制

### 12.1 多層降級策略

```
層級 1：API 成功 → 即時資料（最優）
層級 2：API 失敗 → 備用嵌入資料（功能完整）
層級 3：loadOpenData 整體失敗 → setup() 的 catch 強制使用備用資料
```

```javascript
// loadOpenData() 的錯誤處理
if (schRes.status === 'fulfilled' && schRes.value.success) {
  parseSchoolsCSV(schRes.value.rows);      // 層級 1
} else {
  DIST_DATA = FB_DIST.map(d => ({...d})); // 層級 2
  dataSource.schools = '備用嵌入資料';
}

// setup() 的最終保險
loadOpenData().catch(e => {
  DIST_DATA = [...FB_DIST]; STUDENT_DATA = [...FB_STUDENT]; // 層級 3
  dataReady = true;  // 確保儀表板能顯示
});
```

### 12.2 備用資料的設計

備用資料與即時資料格式完全相同，程式碼其他部分無需判斷：

```javascript
// 備用資料（格式與 parseStudentCSV 輸出完全一致）
const FB_STUDENT = [
  { year: 103, total: 121218, g: [20101, 19577, 19291, 20018, 20235, 21996] },
  // ...
  { year: 113, total: 122987, g: [17808, 19506, 21012, 21727, 20843, 22091] },
];
```

### 12.3 資料驗證（parseStudentCSV）

```javascript
return rows.map(row => {
    const cols = Object.values(row);
    if (cols.length < 8) return null;     // 欄位不足 → 略過
    const year = parseInt(cols[1]) || 0;
    if (!year) return null;               // 無效年度 → 略過
    return { year, total: parseInt(cols[2]) || 0, ... };
  })
  .filter(Boolean)                        // 移除 null
  .sort((a, b) => a.year - b.year);      // 確保年度升序
```

---

## 13. 延伸應用與挑戰練習

### 練習 1：加入更多 OpenData 資料集（簡單）

```javascript
// 嘗試加入「臺北市高中職學生數」資料集
const NEW_RID = 'xxxxx-xxxx-...';
const newRes = await fetchBig5CSV(API_BASE + NEW_RID);
if (newRes.success) {
  // 依欄位結構解析
}
```

### 練習 2：修改捏合距離閾值（簡單）

```javascript
// 目前設定：
const PRE_THRESH = 65;  // 預備區
const ACT_THRESH = 38;  // 捏合觸發

// 嘗試調整，觀察使用體驗差異
// 較大值 → 更早鎖定，適合慢手速用戶
// 較小值 → 需要更用力捏合才觸發
```

### 練習 3：加入「OK 手勢」作為備選點擊（中等）

```javascript
// 偵測食指尖（[8]）與中指尖（[12]）的距離
const mdxTip = hand.keypoints[12];
const okDist = dist(idxTip.x, idxTip.y, mdxTip.x, mdxTip.y);
const isOkGesture = okDist < 40;
```

### 練習 4：地圖視覺化（進階）

```javascript
// 使用 D3.js + TopoJSON 繪製台北市行政區地圖
// 以顏色深淺呈現各區學校密度（Choropleth Map）
// 提示：先用 p5.js 的 loadJSON() 載入 GeoJSON
```

### 練習 5：即時資料自動更新（進階）

```javascript
// 每隔 30 分鐘重新擷取資料
setInterval(async () => {
  await loadOpenData();
  buildButtons();
  setTargets();
}, 30 * 60 * 1000);
```

### 練習 6：手勢控制頁籤切換（進階）

```javascript
// 偵測「滑動」手勢：食指水平移動超過閾值
let gestureStartX = -1;
if (!isPinching && fingerX > 0) {
  if (gestureStartX < 0) gestureStartX = fingerX;
  const swipeDist = fingerX - gestureStartX;
  if (Math.abs(swipeDist) > width * 0.2) {
    activeTab = swipeDist > 0 ? Math.max(0, activeTab-1) : Math.min(2, activeTab+1);
    gestureStartX = -1;
  }
}
```

---

## 14. 常見問題 FAQ

**Q1：載入後顯示「⚠ 部分使用備用資料」，資料正確嗎？**
> 是的，備用嵌入資料與政府 113 年公布的數字完全吻合（由我們事先從 API 擷取並解碼）。只是當 CORS 或網路限制 API 存取時，系統自動使用嵌入版本，功能完整不受影響。

**Q2：為什麼 fetch 有時成功、有時失敗？**
> data.taipei 的 API 對跨來源請求（CORS）的支援不穩定。從本地端開啟 HTML 檔（`file://`）通常會被 CORS 政策阻擋。建議用本地伺服器：
> ```bash
> python -m http.server 8080
> # 然後開啟 http://localhost:8080/taipei-education-p5-gesture.html
> ```

**Q3：手勢游標抖動很厲害怎麼辦？**
> ml5.js HandPose 的原始輸出有一定雜訊。可以對 `rawX, rawY` 加入平滑（Exponential Moving Average）：
> ```javascript
> const smoothFactor = 0.3;
> fingerX = fingerX * (1 - smoothFactor) + rawX * smoothFactor;
> ```

**Q4：捏合太容易/太難觸發怎麼辦？**
> 調整 `ACT_THRESH`（捏合距離閾值，攝影機像素）：
> - `ACT_THRESH = 28`：需要更完全捏合（適合精確操作）
> - `ACT_THRESH = 50`：較輕鬆觸發（適合手部靈活度較低的用戶）

**Q5：政府新增 114 學年資料後，儀表板需要修改嗎？**
> 不需要修改程式碼。年度 Chip 由 `STUDENT_DATA.map(d => d.year)` 動態產生，圖表的 x 軸間距也是動態計算。只要 API 更新後回傳新資料行，儀表板就會自動顯示新年度。

**Q6：能不能用 p5.js 的 `loadTable()` 取代手寫 CSV 解析？**
> 不建議，原因：① `loadTable()` 預設 UTF-8 無法處理 BIG-5；② `loadTable()` 是 `preload()` 同步函式，無法在資料缺失時動態降級。手寫 `fetchBig5CSV()` 搭配 `Promise.allSettled` 更靈活。

---

## 15. 附錄：API 速查與資料欄位

### 15.1 API 端點

```
# 臺北市校數（113年，各學制 × 各行政區）
GET https://data.taipei/api/frontstage/tpeod/dataset/resource.download?rid=591e2881-c8cf-429b-a758-323322aeafbb

# 臺北市國小學生數按年級分（103~113學年）
GET https://data.taipei/api/frontstage/tpeod/dataset/resource.download?rid=e9879dd5-0d52-48dc-83f6-62e6d13c48b6

# 臺北市各級學校班級數（103~113學年）
GET https://data.taipei/api/frontstage/tpeod/dataset/resource.download?rid=ca5699b7-fe98-409b-847a-2b3252d9fe0d
```

### 15.2 資料集詳細頁面

```
校數：    https://data.taipei/dataset/detail?id=a458af5c-bb5b-40d1-867e-f9c3fff1cfc8
學生數：  https://data.taipei/dataset/detail?id=ba85c45e-d755-44e0-afbb-d01b578450fc
班級數：  https://data.taipei/dataset/detail?id=15f67bd1-8ae1-4acc-96df-2a853a83b3ec
```

### 15.3 ml5.js HandPose 關鍵點索引

| 索引 | 名稱 | 用途 |
|------|------|------|
| 0 | Wrist | 手腕 |
| 4 | Thumb Tip | 拇指尖（捏合偵測） |
| 8 | Index Finger Tip | 食指尖（游標位置） |
| 12 | Middle Finger Tip | 中指尖（可延伸手勢） |

### 15.4 常用 p5.js API 速查

```javascript
// 繪製
rect(x, y, w, h, r)          // 圓角矩形（r=圓角半徑）
circle(x, y, d)               // 圓形（d=直徑）
arc(x, y, w, h, start, stop)  // 弧形
line(x1, y1, x2, y2)          // 線段

// 顏色
fill(r, g, b, a)              // 填色（a=0~255）
stroke(r, g, b, a)            // 邊框
noFill() / noStroke()

// 數值工具
lerp(a, b, t)                 // 線性插值（t=0~1）
map(v, in1, in2, out1, out2)  // 數值映射
constrain(v, lo, hi)          // 限制範圍
dist(x1, y1, x2, y2)          // 兩點距離

// Canvas 原生
drawingContext.setLineDash([a, b])   // 設定虛線（記得用 [] 重置）
new TextDecoder('big5').decode(buf)  // BIG-5 解碼

// 文字
textFont('字型名稱')          // 設定字型
textSize(n)                   // 字體大小
textAlign(H, V)               // 對齊（LEFT/CENTER/RIGHT, TOP/BASELINE）
textWidth('文字')             // 取得文字寬度
```

---

*本講義版本：v2.0（含 OpenData 即時連線 + 手勢辨識版）*
*製作日期：2026 年 5 月*
*資料授權：政府資料開放授權條款（OGDL）— 臺北市資料大平臺*
*程式框架：p5.js（MIT License）× ml5.js（MIT License）*
