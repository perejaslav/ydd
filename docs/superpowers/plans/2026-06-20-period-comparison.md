# Сравнение периодов — План реализации

> **Для агентов:** ОБЯЗАТЕЛЬНЫЙ суб-скилл: superpowers:subagent-driven-development (рекомендуется) или superpowers:executing-plans для пошаговой реализации. Шаги используют синтаксис чекбоксов (`- [ ]`).

**Goal:** Добавить в single-file SPA `dashboard.html` режим сравнения двух XLSX-файлов Яндекс.Директа с расчётом дельты по метрикам, плюс два баг-фикса надёжности (CTR по role, toast вместо alert).

**Architecture:** Подход A (параллельные датасеты). Каждый файл парсится в независимый `datasetA`/`datasetB` чистой функцией `parseData`. Существующий single-режим не меняется. При включённом тумблере и двух загруженных файлах включается режим `compare`, в котором агрегаты двух датасетов сопоставляются (`matchByLabel`) и считается дельта. Баг-фиксы встраиваются в общий код, поскольку single-режим их тоже использует.

**Tech Stack:** Vanilla JS (без сборщика), Chart.js 4.4.7 (CDN), SheetJS/xlsx 0.18.5 (CDN). Single file `dashboard.html`.

## Важное замечание по тестированию

Проект — single-file SPA без бэкенда, без сборщика, без тест-раннера. Классический TDD-цикл (red-green) неприменим. Адаптация: каждая задача завершается **ручной верификацией в браузере** с явными шагами «открыть → действие → ожидаемый результат». Для проверки served-файла используется `npx serve .` (или `python -m http.server`), открытый в браузере. Файл-образцы: `2026-06-20_12-40-28_immortal223.xlsx` (июньский, новый формат) — уже в репозитории.

## Структура файла (зоны ответственности в одном `dashboard.html`)

Изменения вносятся в `dashboard.html`. Блоки переупорядочиваются по зонам (разделители-комментарии уже есть):

```
STATE           — mode, datasetA, datasetB, compareMetric + существующие глобалы + METRIC_DIRECTION
HELPERS         — hasValue, toNum, fmt, getMeta, formatValue + НОВЫЕ: roleBy(), toast()
FILE HANDLING   — handleFile(target), parseData(rows, fileName) → чистая функция
DETECT          — detectColumns(dataset) — параметризуется
AGGREGATION     — aggregateBy(dimCol, dataset) + НОВЫЕ: matchByLabel(), deltaRow()
RENDER SINGLE   — renderKPIs/Charts/Table — без изменений логики, только role-фикс
RENDER COMPARE  — renderDeltaKPIs/Charts/Table — НОВЫЙ блок
FILTRATION      — applyFilters — в compare работает по обоим датасетам
UI CONTROLS     — тумблер, слоты файлов — НОВЫЙ блок
```

---

## Task 1: Toast-нотификации (замена alert)

**Файлы:**
- Modify: `dashboard.html` — добавить контейнер `#toasts` в `<body>`, CSS, функцию `toast()`; заменить 3 вызова `alert()`.

Сначала toast, потому что последующие задачи (parseData, handleFile) генерируют ошибки, которые должны идти через toast, а не alert.

- [ ] **Step 1: Добавить CSS и контейнер**

В `<style>`, после `.empty-state` (примерно строка 75), добавить:

```css
#toasts { position: fixed; bottom: 20px; right: 20px; display: flex; flex-direction: column; gap: 10px; z-index: 1000; max-width: 360px; }
.toast { background: var(--surface2); border: 1px solid var(--border); border-left: 4px solid var(--accent); border-radius: 8px; padding: 12px 16px; color: var(--text); font-size: 13px; box-shadow: 0 4px 12px rgba(0,0,0,.4); display: flex; align-items: flex-start; gap: 10px; animation: toastIn .2s ease; }
.toast.error { border-left-color: var(--red); }
.toast.success { border-left-color: var(--green); }
.toast.info { border-left-color: var(--accent); }
.toast .toast-close { margin-left: auto; cursor: pointer; color: var(--text2); font-size: 16px; line-height: 1; background: none; border: none; }
@keyframes toastIn { from { opacity: 0; transform: translateX(20px); } to { opacity: 1; transform: none; } }
```

В `<body>` сразу после `<header>` добавить:

```html
<div id="toasts"></div>
```

- [ ] **Step 2: Добавить функцию toast()**

В блок HELPERS (после `safeChart`, ~строка 259), добавить:

```js
function toast(message, type = 'info', timeout = 5000) {
  const container = document.getElementById('toasts');
  const el = document.createElement('div');
  el.className = `toast ${type}`;
  el.innerHTML = `<span>${message}</span><button class="toast-close" aria-label="Закрыть">×</button>`;
  const close = () => { el.remove(); clearTimeout(timer); };
  el.querySelector('.toast-close').addEventListener('click', close);
  container.appendChild(el);
  const timer = setTimeout(close, timeout);
}
```

- [ ] **Step 3: Заменить alert() в handleFile**

В `handleFile` (строки ~282–294) заменить `alert(...)` на `toast(..., 'error')`:

```js
      if (typeof XLSX === 'undefined' || typeof XLSX.read !== 'function') {
        toast('Библиотека XLSX не загружена. Откройте страницу через локальный сервер или проверьте подключение к CDN.', 'error');
        return;
      }
```

и в `catch`:

```js
    } catch (err) {
      toast('Ошибка при чтении файла: ' + (err.message || err), 'error');
    }
```

- [ ] **Step 4: Заменить alert() в parseData**

В `parseData` две ошибки (строки ~314, ~353–357):

```js
  if (headerIdx === -1) { toast('Не удалось найти заголовки в файле. Убедитесь, что файл содержит отчёт Яндекс.Директ.', 'error'); return; }
```

и:

```js
  if (metrics.length === 0) {
    toast('Файл загружен, но не содержит распознаваемых столбцов с метриками.', 'error');
    document.getElementById('emptyState').querySelector('p').textContent = 'Файл загружен, но не содержит распознаваемых столбцов с метриками. Убедитесь, что это выгрузка Яндекс.Директ.';
    return;
  }
```

- [ ] **Step 5: Заменить alert() в checkDeps()**

В `checkDeps` (строки ~265–274) заменить первое условие — сейчас оно меняет текст кнопки, добавим toast при первой загрузке:

оставить существующую логику смены текста кнопки, но заменить console.warn для Chart на toast:

```js
  if (typeof Chart === 'undefined') {
    toast('Chart.js не загрузился — графики не будут работать.', 'error', 8000);
  }
```

- [ ] **Step 6: Вербальная проверка — grep на отсутствие alert**

Поискать `alert(` в `dashboard.html`. Должно остаться 0 совпадений (кроме возможных в комментариях — проверить вручную).

- [ ] **Step 7: Ручная верификация в браузере**

Запустить `npx serve .` в корне проекта, открыть в браузере.
1. Загрузить любой файл → появляется дашборд, **никакого alert**.
2. Выбрать битый файл (создать копию .xlsx и испортить — переименовать в .xlsx текстовый файл) → появляется toast внизу справа, красная полоса, авто-исчезновение через 5с.
3. Проверить закрытие toast по крестику.

- [ ] **Step 8: Commit**

```bash
git add dashboard.html
git commit -m "feat: toast-нотификации вместо alert()"
```

---

## Task 2: Role-метки для показов/кликов + баг-фикс CTR

**Файлы:**
- Modify: `dashboard.html` — `COL_META` (добавить role), `aggregateBy` (искать по role), `renderKPIs` (искать по role).

- [ ] **Step 1: Добавить role в COL_META для Показы/Клики**

В `COL_META` найти строки `'Показы'` и `'Клики'` (строки ~161–162) и добавить role:

```js
  'Показы':                      { type: 'metric', agg: 'sum', fmt: 'int', role: 'impressions' },
  'Клики':                       { type: 'metric', agg: 'sum', fmt: 'int', role: 'clicks' },
```

- [ ] **Step 2: Добавить хелпер roleBy()**

В блок HELPERS (после `getMeta`), добавить:

```js
function roleBy(roleName, dataset) {
  const ms = (dataset && dataset.metrics) || metrics;
  return ms.find(m => m.meta && m.meta.role === roleName);
}
```

`dataset` опционален — по умолчанию берёт текущий глобальный `metrics` (для single-режима).

- [ ] **Step 3: Починить CTR в aggregateBy**

В `aggregateBy` (строки ~571–574) заменить хардкод имён на role-поиск. Но `aggregateBy` сейчас читает глобальный `metrics` — оставляем как есть в этой задаче (параметризация будет в Task 3), только чиним CTR:

```js
      } else if (met.meta.calc === 'ctr') {
        const impMetric = metrics.find(mm => mm.meta.role === 'impressions');
        const clkMetric = metrics.find(mm => mm.meta.role === 'clicks');
        const imp = impMetric ? m[impMetric.name + '_sum'] : (m['Показы_sum'] || 0);
        const clk = clkMetric ? m[clkMetric.name + '_sum'] : (m['Клики_sum'] || 0);
        result[met.name] = imp ? (clk / imp * 100) : 0;
      } else {
```

fallback на `'Показы_sum'`/`'Клики_sum'` сохраняем — страховка, если role не определён (старые данные).

- [ ] **Step 4: Починить CTR в renderKPIs**

В `renderKPIs` (строки ~620–623) заменить хардкод:

```js
    } else if (m.meta.calc === 'ctr') {
      const impMetric = roleBy('impressions');
      const clkMetric = roleBy('clicks');
      const impName = impMetric ? impMetric.name : 'Показы';
      const clkName = clkMetric ? clkMetric.name : 'Клики';
      const imp = filteredData.reduce((s, r) => s + toNum(r[impName]), 0);
      const clk = filteredData.reduce((s, r) => s + toNum(r[clkName]), 0);
      value = formatValue(imp ? clk / imp * 100 : 0, 'pct');
    } else {
```

- [ ] **Step 5: Ручная верификация**

`npx serve .`, загрузить `2026-06-20_12-40-28_immortal223.xlsx`.
1. KPI-карточка CTR показывает разумное значение (не 0, не NaN).
2. График «CTR по ...» — значения совпадают с KPI.
3. Проверить через консоль браузера: `aggregateBy(metrics[0].isPrimaryDim, filteredData)` → в результате есть ключ CTR с числовым значением.

- [ ] **Step 6: Commit**

```bash
git add dashboard.html
git commit -m "fix: CTR-агрегация по role вместо хардкода имён колонок"
```

---

## Task 3: parseData → чистая функция + параметризация detectColumns/aggregateBy

**Файлы:**
- Modify: `dashboard.html` — `parseData`, `detectColumns`, `getUniqueValues`, `aggregateBy`, `handleFile`.

Это рефакторинг-фундамент: после него parseData возвращает dataset, а compare-режим (Task 5+) сможет парсить два файла независимо.

- [ ] **Step 1: Переписать parseData как чистую функцию**

Заменить текущую `parseData(rows)` (строки ~299–364). Новая сигнатура: `parseData(rows, fileName) → dataset | null`. Возвращает объект или null при ошибке. Не трогает глобалы:

```js
function parseData(rows, fileName) {
  // Find header row
  let headerIdx = -1;
  for (let i = 0; i < Math.min(15, rows.length); i++) {
    if (!rows[i]) continue;
    const nonEmpty = rows[i].filter(c => c !== null && c !== undefined && c !== '');
    if (nonEmpty.length >= 3) {
      const rowStr = rows[i].join('|').toLowerCase();
      if (rowStr.includes('кампания') || rowStr.includes('показы') || rowStr.includes('клики')) {
        headerIdx = i;
        break;
      }
    }
  }
  if (headerIdx === -1) { toast('Не удалось найти заголовки в файле. Убедитесь, что файл содержит отчёт Яндекс.Директ.', 'error'); return null; }

  const headerRow = rows[headerIdx].map(h => (h || '').toString().trim());

  // Parse data rows
  const rawData = [];
  for (let i = headerIdx + 1; i < rows.length; i++) {
    const row = rows[i];
    if (!row) continue;
    const first = (row[0] || '').toString().trim();
    if (!first || first.startsWith('с ') || first.startsWith('Итого') || first.startsWith('Всего')) continue;
    const obj = {};
    headerRow.forEach((h, idx) => { if (h) obj[h] = row[idx] !== undefined ? row[idx] : null; });
    headerRow.forEach(h => {
      if (h && obj[h] !== undefined && obj[h] !== null) {
        const v = parseFloat(obj[h]);
        if (!isNaN(v)) obj[h] = v;
      }
    });
    rawData.push(obj);
  }

  const dimsAndMetrics = detectColumns(headerRow, rawData);
  if (dimsAndMetrics.metrics.length === 0) {
    toast('Файл загружен, но не содержит распознаваемых столбцов с метриками.', 'error');
    return null;
  }

  // Extract period
  let period = '';
  for (let i = 0; i < Math.min(headerIdx, 5); i++) {
    if (rows[i] && rows[i][0]) {
      const s = rows[i][0].toString();
      if (s.includes('период') || s.match(/\d{2}\.\d{2}\.\d{4}/)) { period = s; break; }
    }
  }

  return {
    rawData, headerRow, fileName,
    dimensions: dimsAndMetrics.dimensions,
    metrics: dimsAndMetrics.metrics,
    period: period || `${rawData.length} строк загружено`
  };
}
```

- [ ] **Step 2: Параметризовать detectColumns**

Заменить `detectColumns()` (строки ~369–411). Новая сигнатура: `detectColumns(headerRow, rawData) → { dimensions, metrics }`. Не трогает глобалы:

```js
function detectColumns(headerRow, rawData) {
  const dimensions = [];
  const metrics = [];

  headerRow.forEach(col => {
    if (!col) return;
    const meta = getMeta(col);

    if (meta && meta.type === 'dim') {
      dimensions.push({ name: col, values: getUniqueValues(col, rawData), meta });
    } else if (meta && meta.type === 'metric') {
      metrics.push({ name: col, meta });
    } else {
      const sampleVals = rawData.slice(0, 100).map(r => r[col]).filter(v => v !== undefined && v !== null && v !== '');
      if (sampleVals.length === 0) return;
      const numCount = sampleVals.filter(v => !isNaN(parseFloat(v))).length;
      const ratio = numCount / sampleVals.length;
      if (ratio < 0.3) {
        dimensions.push({ name: col, values: getUniqueValues(col, rawData), meta: {} });
      } else {
        const uniqueNums = [...new Set(sampleVals.filter(v => !isNaN(parseFloat(v))).map(v => parseFloat(v)))];
        const looksPct = uniqueNums.length > 1 && uniqueNums.every(v => v >= 0 && v <= 100) && uniqueNums.some(v => v > 0 && v < 1);
        metrics.push({ name: col, meta: { agg: 'avg', fmt: looksPct ? 'pct' : 'int' } });
      }
    }
  });

  const primaryDim = dimensions.find(d => d.meta && d.meta.role === 'campaign')
    || dimensions.find(d => d.values.length > 1 && d.values.length <= 50)
    || dimensions[0];
  if (primaryDim) metrics.forEach(m => m.isPrimaryDim = primaryDim.name);

  return { dimensions, metrics };
}
```

- [ ] **Step 3: Параметризовать getUniqueValues**

Заменить `getUniqueValues(col)` (строки ~413–419):

```js
function getUniqueValues(col, data) {
  const vals = data.map(r => r[col]).filter(v => v !== undefined && v !== null && v !== '');
  return [...new Set(vals)].sort((a, b) => {
    if (typeof a === 'number' && typeof b === 'number') return a - b;
    return String(a).localeCompare(String(b));
  });
}
```

- [ ] **Step 4: Параметризовать aggregateBy**

Заменить `aggregateBy(dimCol, data)` (строки ~539–582). Новая сигнатура: `aggregateBy(dimCol, data, metricSet)` где `metricSet` опционален (по умолчанию глобальный `metrics`). Также ищет CTR-metrics по role внутри `metricSet`:

```js
function aggregateBy(dimCol, data, metricSet) {
  const ms = metricSet || metrics;
  const map = {};
  data.forEach(r => {
    const k = r[dimCol];
    if (k === undefined || k === null || k === '') return;
    if (!map[k]) {
      map[k] = { _count: 0 };
      ms.forEach(m => {
        if (m.meta.agg === 'sum') map[k][m.name + '_sum'] = 0;
        else { map[k][m.name + '_sum'] = 0; map[k][m.name + '_n'] = 0; }
      });
    }
    const m = map[k];
    m._count++;
    ms.forEach(met => {
      const raw = r[met.name];
      if (met.meta.agg === 'sum') {
        m[met.name + '_sum'] += toNum(raw);
      } else {
        if (hasValue(raw)) { m[met.name + '_sum'] += parseFloat(raw); m[met.name + '_n']++; }
      }
    });
  });

  const impMetric = ms.find(mm => mm.meta.role === 'impressions');
  const clkMetric = ms.find(mm => mm.meta.role === 'clicks');

  return Object.entries(map).map(([k, m]) => {
    const result = { label: k, _count: m._count };
    ms.forEach(met => {
      if (met.meta.agg === 'sum') {
        result[met.name] = m[met.name + '_sum'];
      } else if (met.meta.calc === 'ctr') {
        const imp = impMetric ? m[impMetric.name + '_sum'] : (m['Показы_sum'] || 0);
        const clk = clkMetric ? m[clkMetric.name + '_sum'] : (m['Клики_sum'] || 0);
        result[met.name] = imp ? (clk / imp * 100) : 0;
      } else {
        const n = m[met.name + '_n'];
        result[met.name] = n ? m[met.name + '_sum'] / n : null;
      }
    });
    return result;
  });
}
```

- [ ] **Step 5: Переписать handleFile под новую parseData**

Пока single-режим — `handleFile` парсит в `datasetA` и синхронизирует глобалы для совместимости существующего рендера. Task 5 добавит параметр target. Сейчас:

```js
function handleFile(e) {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = function(ev) {
    try {
      if (typeof XLSX === 'undefined' || typeof XLSX.read !== 'function') {
        toast('Библиотека XLSX не загружена. Откройте страницу через локальный сервер или проверьте подключение к CDN.', 'error');
        return;
      }
      const wb = XLSX.read(ev.target.result, { type: 'array' });
      const ws = wb.Sheets[wb.SheetNames[0]];
      const json = XLSX.utils.sheet_to_json(ws, { header: 1 });
      const ds = parseData(json, file.name);
      if (!ds) return;
      datasetA = ds;
      // Синхронизировать глобалы для существующего single-режима
      rawData = ds.rawData;
      dimensions = ds.dimensions;
      metrics = ds.metrics;
      headerRow = ds.headerRow;
      document.getElementById('periodLabel').textContent = ds.period;
      buildUI();
      applyFilters();
      document.getElementById('emptyState').classList.add('hidden');
      document.getElementById('dashboardContent').classList.remove('hidden');
      document.getElementById('filtersBar').classList.remove('hidden');
    } catch (err) {
      toast('Ошибка при чтении файла: ' + (err.message || err), 'error');
    }
  };
  reader.readAsArrayBuffer(file);
}
```

- [ ] **Step 6: Объявить datasetA, datasetB, mode, compareMetric в STATE**

В блоке STATE (строки ~211–219), добавить после существующих глобалов:

```js
let mode = 'single';        // 'single' | 'compare'
let datasetA = null;        // { rawData, dimensions, metrics, headerRow, period, fileName }
let datasetB = null;
let compareMetric = null;   // активная метрика для дельты (по умолчанию — расход)
```

- [ ] **Step 7: Ручная верификация — single-режим работает как раньше**

`npx serve .`, загрузить `2026-06-20_12-40-28_immortal223.xlsx`.
1. KPI, графики, таблица — все отображаются как до рефакторинга.
2. Применить фильтр — таблица/графики обновляются.
3. В консоли: `datasetA` существует, `datasetA.metrics` — массив, `datasetA.rawData` — массив строк. `typeof parseData` — `'function'`.
4. Проверить: `detectColumns` больше не мутирует глобалы — вызвать `detectColumns(datasetA.headerRow, datasetA.rawData)` в консоли, должно вернуть объект `{dimensions, metrics}`, не затронув глобальные `metrics`.

- [ ] **Step 8: Commit**

```bash
git add dashboard.html
git commit -m "refactor: parseData как чистая функция, параметризация detectColumns/aggregateBy"
```

---

## Task 4: METRIC_DIRECTION + deltaRow + matchByLabel (хелперы сравнения)

**Файлы:**
- Modify: `dashboard.html` — блок HELPERS, добавить словарь направлений и функции дельты/матчинга.

- [ ] **Step 1: Добавить METRIC_DIRECTION в STATE**

В блоке STATE, после `compareMetric`:

```js
// Направление метрики: 'up' = рост хорошо, 'down' = падение хорошо
const METRIC_DIRECTION = {
  impressions: 'up', clicks: 'up', conversions: 'up', ctr: 'up', cr: 'up',
  spend: 'down', cpc: 'down', cpa: 'down', bounce: 'down'
};
```

- [ ] **Step 2: Добавить функцию metricDirection()**

В блок HELPERS:

```js
// Определяет направление метрики по role или по имени
function metricDirection(met) {
  if (met.meta && met.meta.role && METRIC_DIRECTION[met.meta.role]) return METRIC_DIRECTION[met.meta.role];
  const n = met.name.toLowerCase();
  if (n.includes('показ') || n.includes('клик') || n.includes('конверс') && !n.includes('цена')) return 'up';
  if (n.includes('ctr') || n.includes('cr') && n.includes('%')) return 'up';
  if (n.includes('расход') || n.includes('cpc') || n.includes('cpa') || n.includes('цена')) return 'down';
  if (n.includes('отказ')) return 'down';
  return 'up'; // по умолчанию рост = хорошо
}
```

- [ ] **Step 3: Добавить role в COL_META для ключевых метрик**

Дополнить `COL_META` role-метками для метрик, которые будут сравниваться. Найти строки и добавить role:

```js
  'Расход, ₽':                   { type: 'metric', agg: 'sum', fmt: 'money', role: 'spend' },
  'Расход (руб.)':               { type: 'metric', agg: 'sum', fmt: 'money', role: 'spend' },
  'Расход, руб.':                { type: 'metric', agg: 'sum', fmt: 'money', role: 'spend' },
  'Конверсии':                   { type: 'metric', agg: 'sum', fmt: 'int', role: 'conversions' },
  'CPC, ₽':                      { type: 'metric', agg: 'avg', fmt: 'money', role: 'cpc' },
  'CPA, ₽':                      { type: 'metric', agg: 'avg', fmt: 'money', role: 'cpa' },
  'CR, %':                       { type: 'metric', agg: 'avg', fmt: 'pct', role: 'cr' },
  'Отказы, %':                   { type: 'metric', agg: 'avg', fmt: 'pct', role: 'bounce' },
  'Отказы (%)':                  { type: 'metric', agg: 'avg', fmt: 'pct', role: 'bounce' },
  'Процент отказов':             { type: 'metric', agg: 'avg', fmt: 'pct', role: 'bounce' },
  'CTR, %':                      { type: 'metric', agg: 'calc', calc: 'ctr', fmt: 'pct', role: 'ctr' },
  'CTR (%)':                     { type: 'metric', agg: 'calc', calc: 'ctr', fmt: 'pct', role: 'ctr' },
```

(Эти строки уже есть в COL_META — нужно только добавить `, role: '...'` в каждое.)

- [ ] **Step 4: Добавить deltaRow()**

В блок AGGREGATION:

```js
// Считает дельту для одной метрики между значением aVal и bVal
// Возвращает { abs, pct, direction, isGood }
function deltaRow(aVal, bVal, met) {
  const a = (aVal === null || aVal === undefined || isNaN(aVal)) ? null : +aVal;
  const b = (bVal === null || bVal === undefined || isNaN(bVal)) ? null : +bVal;
  if (a === null && b === null) return { abs: null, pct: null, direction: null, isGood: null };
  const aSafe = a === null ? 0 : a;
  const bSafe = b === null ? 0 : b;
  const abs = bSafe - aSafe;
  const pct = aSafe !== 0 ? (abs / Math.abs(aSafe) * 100) : null;
  const dir = metricDirection(met);
  let isGood;
  if (pct === null || pct === 0) isGood = null;
  else isGood = dir === 'up' ? (pct > 0) : (pct < 0);
  return { abs, pct, direction: dir, isGood };
}
```

- [ ] **Step 5: Добавить matchByLabel()**

В блок AGGREGATION:

```js
// Сопоставляет строки двух агрегатов по label (primaryDim), регистронезависимо
function matchByLabel(rowsA, rowsB) {
  const norm = s => String(s).trim().toLowerCase();
  const mapB = new Map(rowsB.map(r => [norm(r.label), r]));
  const setA = new Set(rowsA.map(r => norm(r.label)));
  const matched = [], onlyA = [], onlyB = [];
  rowsA.forEach(a => {
    const b = mapB.get(norm(a.label));
    if (b) matched.push({ a, b }); else onlyA.push(a);
  });
  rowsB.forEach(b => { if (!setA.has(norm(b.label))) onlyB.push(b); });
  return { matched, onlyA, onlyB };
}
```

- [ ] **Step 6: Ручная верификация через консоль**

`npx serve .`, загрузить файл. В консоли:
```js
const a = aggregateBy(datasetA.metrics[0].isPrimaryDim, datasetA.rawData, datasetA.metrics);
const { matched, onlyA, onlyB } = matchByLabel(a, a); // сравнение с собой
console.log(matched.length, onlyA.length, onlyB.length); // ожидаемо: N, 0, 0
const met = datasetA.metrics.find(m => m.meta.role === 'spend') || datasetA.metrics[0];
console.log(deltaRow(matched[0].a[met.name], matched[0].b[met.name], met)); // ожидаемо: pct=0, isGood=null
```

- [ ] **Step 7: Commit**

```bash
git add dashboard.html
git commit -m "feat: хелперы сравнения — METRIC_DIRECTION, deltaRow, matchByLabel"
```

---

## Task 5: UI тумблера и слотов загрузки

**Файлы:**
- Modify: `dashboard.html` — `<header>`, CSS, блок UI CONTROLS.

- [ ] **Step 1: Добавить CSS для тумблера и слотов**

В `<style>`, после `.upload-btn:hover`, добавить:

```css
.compare-toggle { display: flex; align-items: center; gap: 8px; font-size: 13px; color: var(--text2); cursor: pointer; user-select: none; }
.switch { position: relative; width: 36px; height: 20px; background: var(--surface2); border: 1px solid var(--border); border-radius: 10px; transition: .2s; flex-shrink: 0; }
.switch::after { content: ''; position: absolute; top: 2px; left: 2px; width: 14px; height: 14px; background: var(--text2); border-radius: 50%; transition: .2s; }
.compare-toggle input { display: none; }
.compare-toggle input:checked + .switch { background: var(--accent); border-color: var(--accent); }
.compare-toggle input:checked + .switch::after { left: 18px; background: #fff; }
.file-slot { display: flex; align-items: center; gap: 6px; background: var(--surface2); border: 1px solid var(--border); border-radius: 6px; padding: 4px 8px; font-size: 12px; }
.file-slot.loaded { border-color: var(--green); }
.file-slot.error { border-color: var(--red); }
.file-slot .slot-name { color: var(--text2); }
.file-slot .slot-file { color: var(--text); max-width: 140px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.file-slot button { background: none; border: none; color: var(--accent); cursor: pointer; font-size: 12px; padding: 0; }
.file-slot button:hover { text-decoration: underline; }
.compare-hint { background: var(--surface); border: 1px dashed var(--border); border-radius: 8px; padding: 10px 16px; color: var(--text2); font-size: 13px; text-align: center; margin-bottom: 16px; }
```

- [ ] **Step 2: Переписать разметку header**

Заменить существующий `<header>` (строки ~85–94):

```html
<header>
  <div>
    <h1><span>Яндекс.Директ</span> — Дэшборд</h1>
    <div class="period" id="periodLabel">Загрузите файл статистики</div>
  </div>
  <div class="upload-area">
    <label class="compare-toggle" title="Сравнить два файла">
      <input type="checkbox" id="compareToggle" />
      <span class="switch"></span>
      <span>Сравнение периодов</span>
    </label>
    <div id="slotA" class="file-slot hidden">
      <span class="slot-name">A:</span>
      <span class="slot-file" id="slotAName">не загружен</span>
      <button id="slotABtn">Выбрать</button>
      <input type="file" id="fileInputA" accept=".xlsx,.xls" hidden />
    </div>
    <button class="upload-btn" id="uploadBtn" onclick="document.getElementById('fileInput').click()">Загрузить Excel</button>
    <input type="file" id="fileInput" accept=".xlsx,.xls" />
    <input type="file" id="fileInputB" accept=".xlsx,.xls" hidden />
  </div>
</header>
```

- [ ] **Step 3: Добавить блок UI CONTROLS с обработчиками**

Перед `// FILE HANDLING` (или после хелперов), добавить:

```js
// ============================================================
// UI CONTROLS: тумблер сравнения и слоты файлов
// ============================================================
const compareToggle = document.getElementById('compareToggle');
const slotA = document.getElementById('slotA');
const slotABtn = document.getElementById('slotABtn');
const uploadBtn = document.getElementById('uploadBtn');
const fileInputA = document.getElementById('fileInputA');
const fileInputB = document.getElementById('fileInputB');

// Универсальный обработчик файла: target = 'A' | 'B'
function handleFileTarget(file, target) {
  if (!file) return;
  const reader = new FileReader();
  reader.onload = function(ev) {
    try {
      if (typeof XLSX === 'undefined' || typeof XLSX.read !== 'function') {
        toast('Библиотека XLSX не загружена. Проверьте подключение к CDN.', 'error');
        return;
      }
      const wb = XLSX.read(ev.target.result, { type: 'array' });
      const ws = wb.Sheets[wb.SheetNames[0]];
      const json = XLSX.utils.sheet_to_json(ws, { header: 1 });
      const ds = parseData(json, file.name);
      if (!ds) { updateSlotStatus(target, 'error'); return; }
      if (target === 'A') {
        datasetA = ds;
        rawData = ds.rawData; dimensions = ds.dimensions; metrics = ds.metrics; headerRow = ds.headerRow;
      } else {
        datasetB = ds;
      }
      updateSlotStatus(target, 'loaded', file.name);
      updateModeAndRender();
    } catch (err) {
      toast('Ошибка при чтении файла ' + target + ': ' + (err.message || err), 'error');
      updateSlotStatus(target, 'error');
    }
  };
  reader.readAsArrayBuffer(file);
}

function updateSlotStatus(target, status, fileName) {
  const slot = target === 'A' ? slotA : null; // slotB рендерится в режиме compare
  const nameEl = document.getElementById('slot' + target + 'Name');
  if (!nameEl) return;
  nameEl.textContent = status === 'loaded' ? fileName : (status === 'error' ? '⚠ не распознан' : 'не загружен');
}

function updateModeAndRender() {
  if (compareToggle.checked && datasetA && datasetB) {
    mode = 'compare';
  } else {
    mode = 'single';
  }
  renderForMode();
}

function renderForMode() {
  // Без файлов — пустое состояние
  if (!datasetA && !datasetB) {
    document.getElementById('emptyState').classList.remove('hidden');
    document.getElementById('dashboardContent').classList.add('hidden');
    document.getElementById('filtersBar').classList.add('hidden');
    return;
  }
  // Только A (или A без B при выключенном тумблере) — single-режим
  if (mode === 'single') {
    if (datasetA) {
      document.getElementById('periodLabel').textContent = datasetA.period;
      buildUI();
      applyFilters();
      document.getElementById('emptyState').classList.add('hidden');
      document.getElementById('dashboardContent').classList.remove('hidden');
      document.getElementById('filtersBar').classList.remove('hidden');
    }
    return;
  }
  // compare-режим — рендер в Task 6
  document.getElementById('emptyState').classList.add('hidden');
  document.getElementById('filtersBar').classList.add('hidden');
  document.getElementById('dashboardContent').classList.remove('hidden');
  toast('Режим сравнения включён. Рендер будет добавлен в следующей задаче.', 'info');
}

// Тумблер: показать/скрыть слот A, переключить кнопку
compareToggle.addEventListener('change', () => {
  if (compareToggle.checked) {
    slotA.classList.remove('hidden');
    uploadBtn.classList.add('hidden');
  } else {
    slotA.classList.add('hidden');
    uploadBtn.classList.remove('hidden');
  }
  updateModeAndRender();
});

slotABtn.addEventListener('click', () => fileInputA.click());
fileInputA.addEventListener('change', e => { handleFileTarget(e.target.files[0], 'A'); e.target.value = ''; });
fileInputB.addEventListener('change', e => { handleFileTarget(e.target.files[0], 'B'); e.target.value = ''; });
```

- [ ] **Step 4: Удалить старый прямой обработчик fileInput**

Удалить строку (~276):
```js
document.getElementById('fileInput').addEventListener('change', handleFile);
```
Старый `handleFile(e)` из Task 3 нужно удалить полностью (он заменён на `handleFileTarget`). Перед удалением убедиться, что `#fileInput` (кнопка «Загрузить Excel» в single-режиме) тоже использует `handleFileTarget`. Добавить привязку в Step 3-блоке:

```js
document.getElementById('fileInput').addEventListener('change', e => { handleFileTarget(e.target.files[0], 'A'); e.target.value = ''; });
```

- [ ] **Step 5: Ручная верификация — переключение UI**

`npx serve .`:
1. По умолчанию: тумблер выключен, видна кнопка «Загрузить Excel», слот A скрыт.
2. Нажать «Загрузить Excel», выбрать файл → single-режим, дашборд отображается.
3. Включить тумблер → слот A появляется с именем файла, кнопка «Загрузить Excel» скрывается.
4. Слот B в этом шаге ещё не добавлен — compare-режим не активируется. Проверить: при включённом тумблере и только A показывается single-режим (updateModeAndRender ставит mode='single' т.к. нет datasetB).
5. Выключить тумблер → слот A скрыт, кнопка «Загрузить Excel» возвращается, single-режим.
6. В консоли: `datasetA` — объект, `datasetB` — null, `mode` — 'single'.

- [ ] **Step 6: Commit**

```bash
git add dashboard.html
git commit -m "feat: UI тумблера сравнения и слот файла A"
```

---

## Task 6: Слот файла B + активация compare-режима

**Файлы:**
- Modify: `dashboard.html` — разметка slot B, updateSlotStatus для B, рендер подсказки.

- [ ] **Step 1: Добавить slot B в разметку header**

В header, сразу после `<div id="slotA" ...>...</div>` (перед кнопкой upload), добавить:

```html
    <div id="slotB" class="file-slot hidden">
      <span class="slot-name">B:</span>
      <span class="slot-file" id="slotBName">не загружен</span>
      <button id="slotBBtn">Выбрать</button>
    </div>
```

- [ ] **Step 2: Привязать slotB в UI CONTROLS**

В блок UI CONTROLS добавить (рядом с slotA-привязками):

```js
const slotB = document.getElementById('slotB');
const slotBBtn = document.getElementById('slotBBtn');
```

И в обработчиках — показывать slotB вместе с slotA при включении тумблера. Обновить обработчик `compareToggle change`:

```js
compareToggle.addEventListener('change', () => {
  if (compareToggle.checked) {
    slotA.classList.remove('hidden');
    slotB.classList.remove('hidden');
    uploadBtn.classList.add('hidden');
  } else {
    slotA.classList.add('hidden');
    slotB.classList.add('hidden');
    uploadBtn.classList.remove('hidden');
  }
  updateModeAndRender();
});

slotBBtn.addEventListener('click', () => fileInputB.click());
```

- [ ] **Step 3: Поправить updateSlotStatus для B**

В `updateSlotStatus` заменить заглушку:

```js
function updateSlotStatus(target, status, fileName) {
  const nameEl = document.getElementById('slot' + target + 'Name');
  if (!nameEl) return;
  const slot = document.getElementById('slot' + target);
  if (!slot) return;
  slot.classList.remove('loaded', 'error');
  if (status === 'loaded') { slot.classList.add('loaded'); nameEl.textContent = fileName; }
  else if (status === 'error') { slot.classList.add('error'); nameEl.textContent = '⚠ не распознан'; }
  else { nameEl.textContent = 'не загружен'; }
}
```

- [ ] **Step 4: Добавить баннер-подсказку в renderForMode**

Обновить `renderForMode` — добавить баннер если только один файл загружен в compare-режиме:

```js
function renderForMode() {
  if (!datasetA && !datasetB) {
    document.getElementById('emptyState').classList.remove('hidden');
    document.getElementById('dashboardContent').classList.add('hidden');
    document.getElementById('filtersBar').classList.add('hidden');
    return;
  }
  if (mode === 'single') {
    if (datasetA) {
      document.getElementById('periodLabel').textContent = datasetA.period;
      buildUI();
      applyFilters();
      document.getElementById('emptyState').classList.add('hidden');
      document.getElementById('dashboardContent').classList.remove('hidden');
      document.getElementById('filtersBar').classList.remove('hidden');
    }
    return;
  }
  // compare-режим: проверяем наличие обоих файлов
  const content = document.getElementById('dashboardContent');
  document.getElementById('emptyState').classList.add('hidden');
  document.getElementById('filtersBar').classList.add('hidden');
  content.classList.remove('hidden');
  if (!datasetA || !datasetB) {
    const missing = !datasetA ? 'A (основной)' : 'B (сравнения)';
    content.innerHTML = `<div class="compare-hint">Загрузите файл ${missing} для сравнения периодов.</div>`;
    return;
  }
  // Оба файла есть — рендер сравнения (Task 7)
  renderCompare();
}
```

- [ ] **Step 5: Ручная верификация**

`npx serve .`:
1. Включить тумблер → оба слота A и B видны.
2. Загрузить файл в A → slotA зелёный с именем, content показывает «Загрузите файл B для сравнения».
3. Загрузить тот же файл в B → slotB зелёный. Пока renderCompare не реализован — увидим ошибку в консоли (`renderCompare is not defined`). Это ожидаемо для этого шага; исправится в Task 7. Чтобы не падало, временно в renderForMode заменить вызов `renderCompare()` на `toast('Оба файла загружены. Рендер сравнения — следующая задача.', 'info')` и вернуть. **После проверки вернуть `renderCompare()` обратно.**
4. Загрузить битый файл в B → slotB красный «⚠ не распознан», toast с ошибкой, A остаётся.
5. Выключить тумблер → слоты скрыты, single-режим по datasetA.

- [ ] **Step 6: Commit**

```bash
git add dashboard.html
git commit -m "feat: слот файла B и баннер-подсказка compare-режима"
```

---

## Task 7: renderCompare — KPI с дельтой

**Файлы:**
- Modify: `dashboard.html` — новый блок RENDER COMPARE, функция renderCompare.

- [ ] **Step 1: Добавить заглушку renderCompare + структуру контента**

В новом блоке RENDER COMPARE (после RENDER SINGLE, перед TABLE или в конце скрипта):

```js
// ============================================================
// RENDER COMPARE
// ============================================================
function renderCompare() {
  if (!datasetA || !datasetB) return;

  // primaryDim должен совпадать по имени в обоих датасетах
  const primaryA = datasetA.metrics[0] && datasetA.metrics[0].isPrimaryDim;
  const primaryB = datasetB.metrics[0] && datasetB.metrics[0].isPrimaryDim;
  if (!primaryA || !primaryB) {
    toast('Не удалось определить основное измерение в одном из файлов.', 'error');
    return;
  }

  const content = document.getElementById('dashboardContent');
  content.innerHTML = '';

  // Подпись периода
  document.getElementById('periodLabel').textContent = datasetA.period + ' → ' + datasetB.period;

  // Метрики для сравнения — пересечение по role, fallback по имени
  const compareMetrics = buildCompareMetrics(datasetA.metrics, datasetB.metrics);
  if (compareMetrics.length === 0) {
    content.innerHTML = '<div class="compare-hint">Нет общих метрик между файлами A и B для сравнения.</div>';
    return;
  }

  // Сетка KPI
  const kpiGrid = document.createElement('div');
  kpiGrid.className = 'kpi-grid';
  kpiGrid.id = 'kpiGrid';
  content.appendChild(kpiGrid);

  // Графики
  const chartsGrid = document.createElement('div');
  chartsGrid.className = 'charts-grid';
  chartsGrid.id = 'chartsGrid';
  content.appendChild(chartsGrid);

  // Таблица
  const tableSection = document.createElement('div');
  tableSection.innerHTML = `
    <div class="section-title">Сравнение по «${primaryA}»</div>
    <div class="table-card"><div class="table-scroll">
      <table id="dataTable"><thead><tr id="tableHead"></tr></thead><tbody id="tableBody"></tbody></table>
    </div></div>`;
  content.appendChild(tableSection);

  renderDeltaKPIs(compareMetrics);
  renderDeltaCharts(primaryA, compareMetrics);
  renderDeltaTable(primaryA, compareMetrics);
}
```

- [ ] **Step 2: Добавить buildCompareMetrics()**

В блок RENDER COMPARE:

```js
// Сопоставляет метрики двух датасетов по role, затем по имени
function buildCompareMetrics(metricsA, metricsB) {
  const result = [];
  const priority = ['spend', 'impressions', 'clicks', 'ctr', 'conversions', 'cpc', 'cpa', 'cr', 'bounce'];
  // По role
  priority.forEach(role => {
    const a = metricsA.find(m => m.meta.role === role);
    const b = metricsB.find(m => m.meta.role === role);
    if (a && b) result.push({ a, b });
  });
  // По имени (если не вошло по role)
  metricsA.forEach(a => {
    if (result.some(p => p.a === a)) return;
    const b = metricsB.find(m => m.name === a.name);
    if (b && !result.some(p => p.b === b)) result.push({ a, b });
  });
  return result;
}
```

- [ ] **Step 3: Добавить renderDeltaKPIs()**

```js
function renderDeltaKPIs(compareMetrics) {
  const grid = document.getElementById('kpiGrid');
  grid.innerHTML = '';

  // Агрегаты-итоги по каждому датасету (одна группа — весь файл)
  const aggA = aggregateAll(datasetA);
  const aggB = aggregateAll(datasetB);

  const clsColors = ['accent', 'cyan', 'green', 'pink', 'orange', 'red', 'green', 'orange', 'red'];

  compareMetrics.slice(0, 12).forEach((pair, i) => {
    const { a, b } = pair;
    const valA = aggA[a.name];
    const valB = aggB[b.name];
    const d = deltaRow(valA, valB, a);

    const card = document.createElement('div');
    card.className = `kpi-card ${clsColors[i % clsColors.length]}`;
    const label = a.name.replace(/\s*[\(\[,].*$/, '').replace(/\s+₽?$/, '');
    const arrow = d.pct === null ? '' : (d.isGood === null ? '→' : (d.pct > 0 ? '▲' : '▼'));
    const pctStr = d.pct === null ? '—' : (d.pct > 0 ? '+' : '') + d.pct.toFixed(1) + '%';
    const colorClass = d.isGood === null ? '' : (d.isGood ? 'status-good' : 'status-bad');
    const aStr = formatValue(valA, a.meta.fmt);
    const bStr = formatValue(valB, b.meta.fmt);
    card.innerHTML = `
      <div class="kpi-label">${label}</div>
      <div class="kpi-value">${bStr}</div>
      <div class="kpi-sub ${colorClass}">${arrow} ${pctStr} &nbsp;<span style="color:var(--text2)">${aStr} → ${bStr}</span></div>`;
    grid.appendChild(card);
  });
}
```

- [ ] **Step 4: Добавить aggregateAll()**

В блок AGGREGATION:

```js
// Итог по всем строкам датасета для каждой метрики (одна виртуальная группа)
function aggregateAll(dataset) {
  const result = {};
  dataset.metrics.forEach(met => {
    if (met.meta.agg === 'sum') {
      result[met.name] = dataset.rawData.reduce((s, r) => s + toNum(r[met.name]), 0);
    } else if (met.meta.calc === 'ctr') {
      const impM = dataset.metrics.find(mm => mm.meta.role === 'impressions');
      const clkM = dataset.metrics.find(mm => mm.meta.role === 'clicks');
      const impName = impM ? impM.name : 'Показы';
      const clkName = clkM ? clkM.name : 'Клики';
      const imp = dataset.rawData.reduce((s, r) => s + toNum(r[impName]), 0);
      const clk = dataset.rawData.reduce((s, r) => s + toNum(r[clkName]), 0);
      result[met.name] = imp ? clk / imp * 100 : 0;
    } else {
      let sum = 0, n = 0;
      dataset.rawData.forEach(r => { if (hasValue(r[met.name])) { sum += parseFloat(r[met.name]); n++; } });
      result[met.name] = n ? sum / n : null;
    }
  });
  return result;
}
```

- [ ] **Step 5: Временные заглушки renderDeltaCharts/renderDeltaTable**

Пока только KPI, добавим заглушки (Task 8 и 9 их заменят):

```js
function renderDeltaCharts(primaryA, compareMetrics) {
  Object.values(charts).forEach(c => c.destroy());
  charts = {};
  document.getElementById('chartsGrid').innerHTML = '';
}
function renderDeltaTable(primaryA, compareMetrics) {
  document.getElementById('tableHead').innerHTML = '';
  document.getElementById('tableBody').innerHTML = '';
}
```

- [ ] **Step 6: Ручная верификация KPI**

`npx serve .`, включить тумблер, загрузить файл в A и **тот же файл** в B.
1. Отображается KPI-сетка сравнения.
2. Все дельты = 0% (стрелка →, цвет нейтральный), т.к. A === B.
3. Значение в карточке = значение из B.
4. Подпись периода в header: «периодA → периодB».
5. Проверить направления: если загрузить разные файлы (модифицировать копию — поменять пару значений расхода в B), карточка «Расход» при росте должна быть красной (goodWhenDown), «Показы» при росте — зелёной.

- [ ] **Step 7: Commit**

```bash
git add dashboard.html
git commit -m "feat: renderCompare — KPI с дельтой между файлами A и B"
```

---

## Task 8: renderDeltaCharts — 3 графика сравнения

**Файлы:**
- Modify: `dashboard.html` — заменить заглушку renderDeltaCharts.

- [ ] **Step 1: Реализовать renderDeltaCharts с 3 графиками**

Заменить заглушку `renderDeltaCharts`:

```js
function renderDeltaCharts(primaryA, compareMetrics) {
  Object.values(charts).forEach(c => { try { c.destroy(); } catch (e) {} });
  charts = {};
  const grid = document.getElementById('chartsGrid');
  grid.innerHTML = '';

  // compareMetric по умолчанию — расход, иначе первая общая метрика
  const cmp = compareMetrics.find(p => p.a.meta.role === 'spend') || compareMetrics[0];
  if (!cmp) return;

  const aggA = aggregateBy(primaryA, datasetA.rawData, datasetA.metrics);
  // primaryDim в B может называться так же (роль campaign) — используем имя primaryA
  const primaryB = datasetB.metrics[0] && datasetB.metrics[0].isPrimaryDim;
  const aggB = aggregateBy(primaryB, datasetB.rawData, datasetB.metrics);

  const { matched, onlyA, onlyB } = matchByLabel(aggA, aggB);

  // Топ-15 по сумме cmp в A+B
  const withSum = matched.map(({ a, b }) => ({
    a, b,
    sum: (a[cmp.a.name] || 0) + (b[cmp.b.name] || 0)
  })).sort((x, y) => y.sum - x.sum);
  const top15 = withSum.slice(0, 15);

  // 1. Grouped bar: A vs B по топ-15
  const card1 = makeChartCard(`${cmp.a.name.replace(/\s*[\(\[,].*$/, '')} — A vs B (топ-15)`, 'сравнение', true);
  grid.appendChild(card1);
  charts.grouped = safeChart(card1.querySelector('canvas'), {
    type: 'bar',
    data: {
      labels: top15.map(r => r.a.label),
      datasets: [
        { label: 'A', data: top15.map(r => +(r.a[cmp.a.name] || 0).toFixed(2)), backgroundColor: '#6366f199' },
        { label: 'B', data: top15.map(r => +(r.b[cmp.b.name] || 0).toFixed(2)), backgroundColor: '#22c55e99' }
      ]
    },
    options: chartOpts('A vs B', null, false)
  });

  // 2. Diverging bar: дельта % по топ-15
  const card2 = makeChartCard(`Дельта % (${cmp.a.name})`, 'ухудшение / улучшение');
  grid.appendChild(card2);
  const deltaData = top15.map(r => {
    const d = deltaRow(r.a[cmp.a.name], r.b[cmp.b.name], cmp.a);
    return { label: r.a.label, pct: d.pct, isGood: d.isGood };
  });
  charts.diverging = safeChart(card2.querySelector('canvas'), {
    type: 'bar',
    data: {
      labels: deltaData.map(r => r.label),
      datasets: [{
        label: 'Δ %',
        data: deltaData.map(r => r.pct === null ? 0 : +r.pct.toFixed(1)),
        backgroundColor: deltaData.map(r => r.isGood === null ? '#8b8fa8' : (r.isGood ? '#22c55e' : '#ef4444'))
      }]
    },
    options: {
      responsive: true, maintainAspectRatio: false,
      plugins: { legend: { display: false } },
      scales: {
        x: { ticks: { color: '#8b8fa8', maxRotation: 45 }, grid: { color: '#2e3348' } },
        y: { title: { display: true, text: 'Δ %', color: '#8b8fa8' }, ticks: { color: '#8b8fa8' }, grid: { color: '#2e3348' } }
      }
    }
  });

  // 3. Scatter: A по X, B по Y, размер = расход B
  const card3 = makeChartCard(`${cmp.a.name}: A vs B (диагональ = без изменений)`, 'пузырьковая', true);
  grid.appendChild(card3);
  const scatterData = matched.map(({ a, b }) => ({
    x: +(a[cmp.a.name] || 0).toFixed(2),
    y: +(b[cmp.b.name] || 0).toFixed(2),
    r: Math.max(5, Math.sqrt(Math.abs(b[cmp.b.name] || 1)) * 0.8),
    label: a.label
  }));
  charts.scatter = safeChart(card3.querySelector('canvas'), {
    type: 'bubble',
    data: { datasets: [{ label: 'кампании', data: scatterData, backgroundColor: '#6366f166', borderColor: '#6366f1' }] },
    options: {
      responsive: true, maintainAspectRatio: false,
      plugins: {
        legend: { labels: { color: '#e4e6f0', font: { size: 11 } } },
        tooltip: { callbacks: { label: ctx => ctx.raw.label + ': A=' + ctx.raw.x + ', B=' + ctx.raw.y } }
      },
      scales: {
        x: { title: { display: true, text: 'A', color: '#8b8fa8' }, ticks: { color: '#8b8fa8' }, grid: { color: '#2e3348' } },
        y: { title: { display: true, text: 'B', color: '#8b8fa8' }, ticks: { color: '#8b8fa8' }, grid: { color: '#2e3348' } }
      }
    }
  });
}
```

- [ ] **Step 2: Ручная верификация графиков**

`npx serve .`, тумблер вкл, тот же файл в A и B.
1. Grouped bar: пары столбцов A (фиолетовый) и B (зелёный) равной высоты — т.к. A===B.
2. Diverging bar: все столбцы ≈ 0, серые.
3. Scatter: все точки на диагонали y=x.

Затем модифицировать копию файла (изменить расход пары кампаний в B), перезагрузить B:
4. Grouped bar: столбцы A и B разной высоты для изменённых кампаний.
5. Diverging bar: красные/зелёные столбцы по изменённым кампаниям. Для расхода (goodWhenDown) рост = красный, падение = зелёный.
6. Scatter: точки сместились с диагонали.

- [ ] **Step 3: Commit**

```bash
git add dashboard.html
git commit -m "feat: renderDeltaCharts — grouped/diverging/scatter сравнения"
```

---

## Task 9: renderDeltaTable — таблица с дельтой

**Файлы:**
- Modify: `dashboard.html` — заменить заглушку renderDeltaTable.

- [ ] **Step 1: Реализовать renderDeltaTable**

Заменить заглушку:

```js
function renderDeltaTable(primaryA, compareMetrics) {
  const head = document.getElementById('tableHead');
  const body = document.getElementById('tableBody');

  const aggA = aggregateBy(primaryA, datasetA.rawData, datasetA.metrics);
  const primaryB = datasetB.metrics[0] && datasetB.metrics[0].isPrimaryDim;
  const aggB = aggregateBy(primaryB, datasetB.rawData, datasetB.metrics);
  const { matched, onlyA, onlyB } = matchByLabel(aggA, aggB);

  // Колонки: label + каждая compare-метрика (A, B, Δ%, сгруппировано)
  // Для компактности показываем до 3 ключевых метрик
  const cols = compareMetrics.slice(0, 3);

  // Заголовок
  const thCells = [`<th onclick="sortTable('__label__')" style="text-align:left">${primaryA}</th>`];
  cols.forEach(pair => {
    const name = pair.a.name.replace(/\s*[\(\[,].*$/, '').replace(/\s+₽?$/, '');
    thCells.push(`<th class="num" style="text-align:right">A: ${name}</th>`);
    thCells.push(`<th class="num" style="text-align:right">B: ${name}</th>`);
    thCells.push(`<th class="num" style="text-align:right">Δ%</th>`);
  });
  head.innerHTML = thCells.join('');

  // Строки — совпавшие, затем только A, только B
  const rows = [];
  matched.forEach(({ a, b }) => {
    const cells = [`<td>${a.label}</td>`];
    cols.forEach(pair => {
      const d = deltaRow(a[pair.a.name], b[pair.b.name], pair.a);
      const colorClass = d.isGood === null ? '' : (d.isGood ? 'status-good' : 'status-bad');
      const pctStr = d.pct === null ? '—' : (d.pct > 0 ? '+' : '') + d.pct.toFixed(1) + '%';
      cells.push(`<td class="num">${formatValue(a[pair.a.name], pair.a.meta.fmt)}</td>`);
      cells.push(`<td class="num">${formatValue(b[pair.b.name], pair.b.meta.fmt)}</td>`);
      cells.push(`<td class="num ${colorClass}">${pctStr}</td>`);
    });
    rows.push(`<tr>${cells.join('')}</tr>`);
  });

  if (onlyA.length) {
    const span = 1 + cols.length * 3;
    rows.push(`<tr><td colspan="${span}" style="background:var(--surface2);color:var(--text2);font-style:italic">Только в A (${onlyA.length})</td></tr>`);
    onlyA.forEach(a => {
      const cells = [`<td>${a.label}</td>`];
      cols.forEach(pair => {
        cells.push(`<td class="num">${formatValue(a[pair.a.name], pair.a.meta.fmt)}</td>`);
        cells.push(`<td class="num">—</td>`);
        cells.push(`<td class="num">—</td>`);
      });
      rows.push(`<tr>${cells.join('')}</tr>`);
    });
  }
  if (onlyB.length) {
    const span = 1 + cols.length * 3;
    rows.push(`<tr><td colspan="${span}" style="background:var(--surface2);color:var(--text2);font-style:italic">Только в B (${onlyB.length})</td></tr>`);
    onlyB.forEach(b => {
      const cells = [`<td>${b.label}</td>`];
      cols.forEach(pair => {
        cells.push(`<td class="num">—</td>`);
        cells.push(`<td class="num">${formatValue(b[pair.b.name], pair.b.meta.fmt)}</td>`);
        cells.push(`<td class="num">—</td>`);
      });
      rows.push(`<tr>${cells.join('')}</tr>`);
    });
  }

  body.innerHTML = rows.join('');
}
```

- [ ] **Step 2: Защитить sortTable в compare-режиме**

Текущий `sortTable(col)` ссылается на глобальные `metrics`/`filteredData` — в compare-режиме это неактуально. Минимальная защита: в `sortTable` добавить проверку:

```js
function sortTable(col) {
  if (mode === 'compare') return; // сортировка в compare-режиме не поддерживается в этой итерации
  if (sortCol === col) sortDir *= -1;
  else { sortCol = col; sortDir = -1; }
  renderTable();
}
```

(Сортировка compare-таблицы вынесена за скоуп — YAGNI; заголовки кликабельны визуально, но в compare-режиме клик игнорируется. Уточнить в коде комментарием.)

- [ ] **Step 3: Ручная верификация таблицы**

`npx serve .`, тумблер вкл:
1. Загрузить одинаковые файлы → таблица: совпавшие кампании с A=B, Δ%=0 (нейтральный цвет), блоков «только в A/B» нет.
2. Загрузить разные файлы (B с изменённым расходом) → Δ% ненулевые, цветные по направлению.
3. Создать B с кампанией, отсутствующей в A → блок «Только в B».
4. Клик по заголовку колонки в compare → ничего не происходит (защита).

- [ ] **Step 4: Commit**

```bash
git add dashboard.html
git commit -m "feat: renderDeltaTable — таблица с дельтой A vs B"
```

---

## Task 10: Финальная интеграционная проверка + cleanup

**Файлы:**
- Modify: `dashboard.html` — финальный прогон всех сценариев спеки.

- [ ] **Step 1: Прогнать все 7 сценариев из спеки**

`npx serve .`, для каждого сценария проверить ожидаемый результат:

1. **Sanity (A=B):** A=июньский, B=копия → все дельты 0%, scatter на диагонали, diverging серый. ✅
2. **Ненулевая дельта:** B=модифицированная копия (изменён расход пары кампаний) → ненулевые Δ%, diverging показывает направление, расход рост=красный. ✅
3. **Смешанные форматы:** проверить, что и майский, и июньский форматы определяют primaryDim с role campaign (в консоли: `datasetA.metrics[0].isPrimaryDim` — название колонки кампании). Если файла в майском формате нет — пропустить с пометкой. ⚠️
4. **Битый B:** битый файл в B → toast с ошибкой, slotB красный, A остаётся активным (single-режим). ✅
5. **Переключение тумблера:** выкл→вкл при загруженных A,B → мгновенно compare без перезагрузки. ✅
6. **Single без изменений:** тумблер выкл, загрузить один файл → прежнее поведение, все 8 графиков, таблица с сортировкой. ✅
7. **«Только в A/B»:** B с другим набором кампаний → блоки в таблице. ✅

- [ ] **Step 2: Grep-проверка отсутствия alert и заглушек**

Найти в `dashboard.html`:
- `alert(` — должно быть 0 (кроме комментариев).
- `TODO`, `TBD`, `следующая задача`, `будет добавлен` — 0 (временные заглушки из Task 6/7 должны быть заменены).

- [ ] **Step 3: Проверить cleanup графиков**

В консоли при переключении single↔compare несколько раз:
- `Object.keys(charts)` — разумное число (не растёт бесконечно).
- Не должно быть утечки canvas-элементров: `document.querySelectorAll('canvas').length` — стабильно.

- [ ] **Step 4: Обновить context.md краткой записью о новой фиче**

В `context.md`, в секцию «Не реализованные метрики/фичи», убрать строку «Сравнение периодов (MoM/YoY) — полностью отсутствует» и добавить в конец блока «Архитектура» краткое описание режима compare со ссылкой на спеку.

- [ ] **Step 5: Commit финальный**

```bash
git add dashboard.html context.md
git commit -m "feat: сравнение периодов — финальная интеграция и cleanup"
```

---

## Self-Review плана

**1. Spec coverage (покрытие спеки):**
- Режим сравнения двух файлов → Tasks 5, 6, 7, 8, 9 ✅
- Тумблер в header → Task 5 ✅
- KPI с дельтой + цвет по направлению → Task 7 (renderDeltaKPIs) + Task 4 (METRIC_DIRECTION) ✅
- 3 графика (grouped/diverging/scatter) → Task 8 ✅
- Таблица с Δ + блоки только в A/B → Task 9 ✅
- Матчинг точный, регистронезависимый → Task 4 (matchByLabel) ✅
- Баг CTR по role → Task 2 ✅
- toast вместо alert → Task 1 ✅
- parseData как чистая функция → Task 3 ✅
- Single-режим не изменён → проверка в Task 10 (сценарий 6) ✅
- Критерий «фильтры в compare по обоим датасетам» → НЕ покрыт отдельной задачей. **Добавлено ниже как Task 11.** ⚠️

**2. Placeholder scan:** заглушки renderDeltaCharts/renderDeltaTable в Task 7 явно помечены и заменяются в Tasks 8/9. Временный toast в Task 6 Step 5 явно оговорён с инструкцией вернуть обратно. Других TODO/TBD нет. ✅

**3. Type consistency:** `handleFileTarget`, `updateModeAndRender`, `renderForMode`, `renderCompare`, `aggregateAll`, `buildCompareMetrics`, `matchByLabel`, `deltaRow`, `metricDirection`, `roleBy`, `toast`, `updateSlotStatus` — имена согласованы между задачами. `datasetA/B` структура `{rawData, dimensions, metrics, headerRow, period, fileName}` единая везде. ✅

Добавляю недостающую задачу на фильтры в compare-режиме.

---

## Task 11: Фильтры в compare-режиме (по обоим датасетам)

**Файлы:**
- Modify: `dashboard.html` — показывать фильтры в compare-режиме, применять к обоим датасетам.

- [ ] **Step 1: Добавить buildFilters из объединённых значений в compare-режиме**

В `renderForMode`, блок compare (где `renderCompare()`), перед вызовом добавить показ filtersBar и построение объединённых фильтров. Обновить ветку:

```js
  // Оба файла есть — рендер сравнения
  document.getElementById('filtersBar').classList.remove('hidden');
  buildCompareFilters();
  renderCompare();
```

- [ ] **Step 2: Добавить buildCompareFilters()**

В блок FILTRATION:

```js
function buildCompareFilters() {
  const bar = document.getElementById('filtersBar');
  bar.innerHTML = '';
  filterSelects = {};

  // primaryDim берем из A
  const primaryA = datasetA.metrics[0] && datasetA.metrics[0].isPrimaryDim;
  if (!primaryA) return;

  // Объединить значения primaryDim из A и B
  const valsA = datasetA.dimensions.find(d => d.name === primaryA);
  const valsB = datasetB.dimensions.find(d => d.name === datasetB.metrics[0].isPrimaryDim);
  if (!valsA && !valsB) return;

  const allVals = [...new Set([
    ...(valsA ? valsA.values : []),
    ...(valsB ? valsB.values : [])
  ])].sort((a, b) => String(a).localeCompare(String(b)));

  if (allVals.length <= 1) return;

  const group = document.createElement('div');
  group.className = 'filter-group';
  const label = document.createElement('label');
  label.textContent = primaryA;
  const sel = document.createElement('select');
  sel.innerHTML = '<option value="">Все</option>' + allVals.map(v => `<option value="${v}">${v}</option>`).join('');
  sel.addEventListener('change', renderCompare);
  group.appendChild(label);
  group.appendChild(sel);
  bar.appendChild(group);
  filterSelects['__label__'] = sel;

  // Кнопка сброса
  const resetBtn = document.createElement('button');
  resetBtn.className = 'reset-btn';
  resetBtn.textContent = 'Сбросить';
  resetBtn.addEventListener('click', () => {
    Object.values(filterSelects).forEach(s => s.value = '');
    renderCompare();
  });
  bar.appendChild(resetBtn);
}
```

- [ ] **Step 3: Применять фильтр в renderCompare и renderDelta**

В начале `renderCompare`, после получения primaryA/B, отфильтровать rawData обоих датасетов по выбранному primaryDim, если фильтр установлен. Добавить в renderCompare перед агрегатами:

```js
  // Применить фильтр primaryDim, если выбран
  const filterSel = filterSelects['__label__'];
  const filterVal = filterSel ? filterSel.value : '';
  let dataA = datasetA.rawData;
  let dataB = datasetB.rawData;
  if (filterVal) {
    dataA = datasetA.rawData.filter(r => String(r[primaryA]) === filterVal);
    const primaryBName = datasetB.metrics[0].isPrimaryDim;
    dataB = datasetB.rawData.filter(r => String(r[primaryBName]) === filterVal);
  }
```

И в `renderDeltaCharts`/`renderDeltaTable`/`aggregateAll`-вызовах внутри renderCompare использовать `dataA`/`dataB` вместо `datasetA.rawData`/`datasetB.rawData`. Для KPI — передавать отфильтрованные данные в aggregateAll: создать вариант `aggregateAllFromData(data, metrics)` или параметризовать aggregateAll. Минимально — добавить опциональный параметр:

Обновить `aggregateAll(dataset, overrideData)`:

```js
function aggregateAll(dataset, overrideData) {
  const rows = overrideData || dataset.rawData;
  // ... далее использовать rows вместо dataset.rawData
  dataset.metrics.forEach(met => {
    if (met.meta.agg === 'sum') {
      result[met.name] = rows.reduce((s, r) => s + toNum(r[met.name]), 0);
    } else if (met.meta.calc === 'ctr') {
      // ... rows.reduce ...
    } else {
      // ... rows.forEach ...
    }
  });
}
```

В `renderDeltaKPIs` вызвать `aggregateAll(datasetA, dataA)` и `aggregateAll(datasetB, dataB)` — для этого `dataA`/`dataB` нужно пробросить из `renderCompare`. Минимально: сделать `dataA`/`dataB` глобальными для compare-сессии:

```js
let compareDataA = null, compareDataB = null;
```

В renderCompare после фильтрации:
```js
  compareDataA = dataA;
  compareDataB = dataB;
```

И в renderDeltaKPIs/Charts/Table использовать `compareDataA`/`compareDataB` вместо `datasetX.rawData`.

- [ ] **Step 4: Объявить compareDataA/B в STATE**

В блок STATE:
```js
let compareDataA = null, compareDataB = null;
```

- [ ] **Step 5: Ручная верификация фильтров**

`npx serve .`, тумблер вкл, оба файла загружены:
1. В filtersBar — один фильтр по primaryDim (кампания) со всеми кампаниями из A и B.
2. Выбрать кампанию → KPI, графики, таблица показывают только эту кампанию (одна строка в таблице, одна точка в scatter).
3. Сбросить → обратно полный набор.
4. Выбрать кампанию, которой нет в одном из файлов → в таблице появится в блоке «Только в A» или «Только в B».

- [ ] **Step 6: Commit**

```bash
git add dashboard.html
git commit -m "feat: фильтр по primaryDim в compare-режиме (по обоим датасетам)"
```

---

## Финальный шаг: обновить спеку/контекст и summary

- [ ] **Финальный commit (если были правки context.md в Task 11)** — объединить с Task 10 или отдельным.

После всех задач:
- single-режим работает как раньше (сценарий 6).
- compare-режим: KPI с дельтой, 3 графика, таблица с Δ, фильтр по кампании.
- CTR считается по role, alert заменён на toast.
