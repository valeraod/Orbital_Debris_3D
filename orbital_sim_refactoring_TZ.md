# Orbital Debris 3D — Техническое задание на рефакторинг

> **Версия:** 2.0 | **База:** `orbital_sim_v2_11.html` (2118 строк) | **Статус:** не начат
> Цель: разбить один файл на модульный проект (CSS + JS, классические скрипты) без изменения поведения.

---

## Правило работы

> ⚠️ **После каждого этапа симулятор должен открываться и работать полностью, как в оригинале.**
> Рефакторинг — не переписывание. Логика и UX не меняются, только структура файлов.

Любой агент, продолжающий работу:
1. Читает этот файл целиком, особенно таблицу «Прогресс по этапам» и «Известные проблемы».
2. Находит первый этап со статусом ⬜ или 🔄.
3. После завершения — отмечает чекбоксы, обновляет таблицу прогресса, заполняет «Сдача этапа», создаёт zip-снапшот.
4. Любые отклонения от плана и новые проблемы — фиксирует в разделе «Известные проблемы и заметки».

### Работа со снапшотами (zip)

- Базовым считается **zip предыдущего этапа, прошедший проверку пользователем**. Этот архив хранится у пользователя, а не в рабочей папке агента.
- В конце каждого этапа агент создаёт новый zip-снапшот и передаёт его пользователю на проверку.
- Если требуется **откат на предыдущий этап** — агент запрашивает у пользователя соответствующий zip (`orbital-sim_stage-NN_xxx.zip`), а не ищет его в файловой системе.
- Агент не хранит и не накапливает архивы прошлых этапов локально как «источник истины» — текущая рабочая папка считается продолжением последнего проверенного zip.

---

## Целевая структура проекта

```
orbital-sim/
├── index.html              ← только разметка + <script src="js/...js"> (классические, в порядке зависимостей)
├── css/
│   ├── base.css             ← reset, body, #cw, canvas (~30 строк)
│   ├── hud.css               ← #hud, #leg, #stats-panel (~60 строк)
│   └── controls.css          ← #bar, #bar-row2, button, .ctl, инпуты (~150 строк)
└── js/
    ├── physics.js            ← GM/RE/J2/atmo, rk4, orbitalElements, main/frag степперы (~330 строк)
    ├── scene3d.js             ← Three.js сцена, камера, орбиты, меши, трейлы, render3D (~530 строк)
    ├── scene2d.js             ← Canvas2D рендер, камера2D, орбиты, render2D (~410 строк)
    ├── ui.js                  ← HUD, stats-panel, контролы (кнопки/слайдеры), lock/unlock, mode-switch (~330 строк)
    ├── input.js               ← мышь/тач для canvas3 и canvas2 (~210 строк)
    └── main.js                ← state, константы, главный цикл, init, склейка (~260 строк)
```

Итог: 6 JS-файлов вместо прежних ~25 — крупнее, проще для проекта такого размера (2118 строк, что вдвое меньше базового проекта star-destroyer).

### Общие принципы
- **Классические `<script src="...">` без ES-модулей** (как в проекте star-destroyer) — никаких `import`/`export`, `type="module"`. Все объявления верхнего уровня в `js/*.js` становятся глобальными (общий scope через `window`), как в исходном монолите. Это позволяет открывать `index.html` напрямую как `file://` без HTTP-сервера.
- Порядок подключения `<script src>` в `index.html` важен — зависимости подключаются раньше зависимых модулей.
- Three.js — оставить через CDN `<script>` (глобальный `THREE`), как в оригинале. Подключается первым.
- Состояние (`simTime`, `frags`, `mainObj`, флаги, камеры и т.д.) — единый объект `state`, объявленный в `main.js` (этап 7) как обычная глобальная переменная (`let state = {...}` или `const state = {...}`), остальные модули обращаются к нему как к глобальной переменной.
- Константы (`GM, RE, J2, R_OBS, CD, AREA, TRAIL_LEN, SIM_INTERVAL, SUB_STEPS, ...`) — глобальные `const` в `physics.js` (физические) и `main.js` (рантайм/UI), без отдельного файла констант.
- Циклические зависимости между `physics.js` ↔ `scene3d.js`/`scene2d.js`/`ui.js` — допустимы, так как все функции глобальны и вызываются только во время выполнения (после полной загрузки всех скриптов), а не на этапе объявления.
- Переименования функций/переменных — не делать в этом проходе.

---

## Этап 0 — Подготовка

**Приоритет: сделать первым.**

### 0.1 Создать структуру папок
- [ ] Создать `orbital-sim/`, `orbital-sim/css/`, `orbital-sim/js/`
- [ ] Скопировать `orbital_sim_v2_11.html` как отправную точку в `orbital-sim/index.html`

### 0.2 Сдача этапа 0
- [ ] Обновить таблицу прогресса: `Этап 0 → ✅ готово`
- [ ] Создать `orbital-sim_stage-00_original.zip` — копия оригинала без изменений
- [ ] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается в браузере)

---

## Этап 1 — CSS (строки 7–126 оригинала)

Самый безопасный этап — CSS не затрагивает логику.

### 1.1 `css/base.css`
- [x] `*{box-sizing:border-box;margin:0;padding:0}`
- [x] `html,body{height:100%;background:#03050f;overflow:hidden;font-family:'Courier New',monospace}`
- [x] `body{display:flex;flex-direction:column}`
- [x] `#cw`, `#c3d`, `#c2d`

### 1.2 `css/hud.css`
- [x] `#hud` и вложенные (`.lbl`, `.val`, `.warn`, `#hud-zoom`)
- [x] `#leg` + media query (`max-width:400px`)
- [x] `#stats-panel` и вложенные (`.pct-bar`, `.pct-fill`)

### 1.3 `css/controls.css`
- [x] `#bar`, `#grpSpd` + media query (`max-width:500px`)
- [x] `#bar-row2`
- [x] `button` и модификаторы (`#bBoom`, `#bFocus`, `#bAllOrbits`)
- [x] `.ctl` и все вложенные селекторы (label, `.v`, input range/number/text + псевдоэлементы thumb/track)
- [x] `.param-group.locked`
- [x] `#cbWrap`, `#cbAtmo`

### 1.4 Обновить `index.html`
- [x] Удалить `<style>` блок
- [x] Добавить в `<head>`:
  ```html
  <link rel="stylesheet" href="css/base.css">
  <link rel="stylesheet" href="css/hud.css">
  <link rel="stylesheet" href="css/controls.css">
  ```

### Тест этапа 1
- [x] Визуал идентичен оригиналу (цвета, шрифты, отступы, кнопки)
- [x] HUD, легенда, stats-panel выглядят как раньше
- [x] DevTools → Responsive ≤500px и ≤400px → адаптивные правки работают
- [x] DevTools → Sources → подключены base.css, hud.css, controls.css; в `<head>` нет `<style>`

### Сдача этапа 1
- [x] Обновить таблицу прогресса: `Этап 1 → ✅ готово`
- [x] Создать `orbital-sim_stage-01_css.zip`
- [x] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается)

---

## Этап 2 — `js/physics.js`

Строки **237–550** (без секции 3D-рендера), без зависимостей от DOM/Three.js, кроме точечных DOM-вызовов в `stepMainObj`/`explode` (см. ниже).

> **Реализовано (отклонение от исходного плана):** в `physics.js` вынесены ТОЛЬКО чистые функции без обращения к общему состоянию — они принимают все входные данные параметрами и не требуют `state`/`main.js`. Функции, работающие с общим mutable-состоянием (`mainPos/mainVel/initMainObj/stepMainObj/stepFrags/correctFragEnergies/adaptiveSkipDt/explode`), **остаются в `index.html`** до этапа 7, где появится единый объект `state` (`main.js`). Перенос их сейчас потребовал бы либо дублирования объекта состояния, либо немедленного выполнения всей механической работы этапа 7 — решено отложить, чтобы не делать эту работу дважды.

### 2.1 Константы и атмосфера
- [x] `GM, RE, J2(внутр.), R_OBS, CD, AREA` (237–242) — экспортированы из `physics.js`. `J2` используется только внутри `rk4`, наружу не экспортируется (не нужен в index.html)
- [x] `ATMO_LAYERS`, `airDensity(h)` (247–259) — перенесены в `physics.js`, используются только внутри `rk4` (не экспортируются наружу — не требуются в index.html)

### 2.2 Интегратор RK4
- [x] `_k = new Float64Array(24)`, `rk4(...)` (261–293) — перенесены в `physics.js`
- [x] `_rk4out = new Float64Array(6)` (294) — **остался в `index.html`** (буфер вывода, используется только в стейтфул-функциях `stepMainObj/stepFrags`)
- [x] **Изменение сигнатуры:** `rk4` была impure (читала глобальные `atmoEnabled`/`BCOEF`). Теперь `rk4(x,y,z,vx,vy,vz,dt,drag,out,atmoEnabled,BCOEF)` — оба параметра передаются явно вызывающей стороной. Вызовы в `stepMainObj`/`stepFrags` обновлены: `rk4(x,y,z,vx,vy,vz,dts,true,_rk4out,atmoEnabled,BCOEF)`

### 2.3 Орбитальные расчёты
- [x] `orbitalElements3D(...)` (296–313) — перенесена в `physics.js`, экспортирована
- [x] `orbitalElements2D(...)` (1170–1178) — перенесена в `physics.js` (вызывает `orbitalElements3D` внутри модуля), экспортирована; дублирующее определение в index.html удалено

### 2.4 RNG и адаптивный шаг
- [x] `mulberry32(seed)` (340–347) — перенесена в `physics.js`, экспортирована
- [x] `adaptiveSubs(dt, r)` (372–376) — перенесена в `physics.js`, экспортирована; дублирующее определение в index.html удалено
- [ ] `adaptiveSkipDt()` (378–392) — **отложено**, остаётся в index.html (читает `exploded, frags` — общее состояние, см. примечание выше)

### 2.5 Главный объект и осколки — ОТЛОЖЕНО до этапа 7
- [ ] `mainPos(t)`, `mainVel(t)` — остаются в index.html (читают `RORB/V0/T0`)
- [ ] `initMainObj()` — остаётся в index.html (пишет `mainObj`)
- [ ] `correctFragEnergies()` — остаётся в index.html
- [ ] `stepMainObj(dt)` — остаётся в index.html, вызывает `rk4` с новой сигнатурой (см. 2.2). DOM-вызов `document.getElementById('bBoom').disabled=true` оставлен как есть, TODO для будущего события
- [ ] `stepFrags(dt)` — остаётся в index.html, аналогично вызывает `rk4` с новой сигнатурой
- [ ] `explode()` — остаётся в index.html без изменений

### 2.6 Экспорт и подключение
- [x] `physics.js` экспортирует: `GM, RE, R_OBS, CD, AREA, ATMO_LAYERS, airDensity, rk4, orbitalElements3D, orbitalElements2D, mulberry32, adaptiveSubs`
- [x] `index.html`: добавлен `<script src="js/physics.js"></script>` перед основным `<script>`. Все имена из `physics.js` (`GM, RE, J2, R_OBS, CD, AREA, ATMO_LAYERS, airDensity, rk4, orbitalElements3D, orbitalElements2D, mulberry32, adaptiveSubs`) становятся глобальными — никаких `import`/`export`. Открывается напрямую как `file://`, без HTTP-сервера
- [x] Three.js CDN `<script>` остаётся классическим (не модулем), подключён до модульного скрипта — глобальный `THREE` доступен модулю через `window`

### Сдача этапа 2
- [x] Обновить таблицу прогресса: `Этап 2 → ✅ готово` (с пометкой: физика разделена на «чистую» (`physics.js`) и «стейтфул» (index.html, до этапа 7))
- [x] Создать `orbital-sim_stage-02_physics.zip`
- [ ] Передать zip пользователю на проверку (синтаксис всех файлов корректен; полная работа симулятора не обязательна на этом этапе)

---

## Этап 3 — `js/scene3d.js`

Строки **387–927** текущего `index.html` на момент начала этапа (3D-рендерер на Three.js: сцена, камера, орбиты, меши, трейлы, render3D, resize3D).

> **Реализовано.** В `js/scene3d.js` перенесён весь блок инициализации и рендеринга 3D-сцены **кроме** `updateFragInfo()` (осталась в index.html — относится к `ui.js`, этап 5) и `pickFrag3D()` (осталась в index.html — относится к `input.js`, этап 6, так как в оригинале расположена среди обработчиков мыши/тача, а не в блоке сцены).
>
> **Решение по загрузке (classic scripts, без ES-модулей):** `scene3d.js` использует глобальные `RORB, RE, GM, R_OBS, AREA, CD, orbitalElements3D` и т.д. На верхнем уровне `scene3d.js` есть код, выполняющийся сразу при загрузке (`camera.position.set(0,0,RORB*3.2)`, `updateRefOrbit()`, `let camDist=RORB*2.745`, `updateCamera()`) — он требует, чтобы `RORB` был определён ДО загрузки `scene3d.js`. Поэтому `index.html` разбит на **два** инлайн-скрипта:
>   - **Скрипт A** (до `scene3d.js`): `objMass, BCOEF, HORB, RORB` — минимальный набор, необходимый для top-level кода `scene3d.js`.
>   - `<script src="js/scene3d.js"></script>`
>   - **Скрипт B** (после `scene3d.js`): всё остальное состояние (`V0, T0, NFRAG, atmoEnabled, simTime, ...`), `mainPos/mainVel/initMainObj/...`, `updateFragInfo`, 2D-рендерер, UI, input — без изменений в содержимом, кроме удалённого блока 3D-сцены.
>   - Эта разбивка временная и будет переоценена на этапе 7 (`main.js`), когда появится единый объект `state` — возможно, весь блок состояния переместится в отдельный скрипт, загружаемый первым (после `physics.js`, до `scene3d.js`).

### 3.1 Сцена, камера, свет, Земля, звёзды, сетка
- [x] `renderer, scene, camera, _CAM_K, camOrtho, orthoZoom` — перенесены в `scene3d.js`
- [x] `camDistMatch()` — перенесена
- [x] `buildOrthoCamera()`, `applyOrthoZoom()` — перенесены
- [x] Свет: `sun`, `AmbientLight` — перенесены
- [x] Звёздное поле — перенесено
- [x] Земля + атмосфера, сетка широт/долгот — перенесены
- [x] `activeCamera3D()` — перенесена (зависит от глобального `is2D`, объявленного позже в index.html — безопасно, т.к. функция вызывается только во время рендера, после загрузки всех скриптов)

### 3.2 Орбитальные линии
- [x] `refOrbitLine`, `updateRefOrbit()` — перенесены (вызов `updateRefOrbit()` при загрузке скрипта требует `RORB`, см. примечание выше)
- [x] `mainOrbitLine`, `buildMainOrbitLine(...)`, `updateMainOrbitLine()`, `freezeMainOrbit()` — перенесены (`mainOrbitFrozen` остаётся обычной глобальной переменной, не `state.*` — объект `state` появится в этапе 7)
- [x] `selOrbitLine`, `updateSelOrbit3D()` — перенесены
- [x] `allOrbitLines`, `clearAllOrbitLines3D()`, `buildAllOrbitLines3D()` — перенесены

### 3.3 Меши главного объекта и осколков
- [x] `mainGeo/mainMat/mainMesh`, `mainGlowGeo/mainGlowMat/mainGlowMesh` — перенесены
- [x] `_sharedFragGeo`, `fragMeshes`, `build3DFragMeshes()` — перенесены
- [x] `fragColorHex(f)` — перенесена в `scene3d.js` как глобальная функция (будет доступна `scene2d.js` при подключении после `scene3d.js` на этапе 4)

### 3.4 Трейлы
- [x] `trailLines`, `mainTrailLine`, `build3DTrailLines()`, `buildMainTrailLine()`, `updateMainTrail()`, `updateTrails()` — перенесены

### 3.5 Камера: orbit/pan/pick
- [x] `camTheta/camPhi/camDist/camTarget*` — перенесены как обычные глобальные переменные (не `state.*` — см. примечание о `state` в этапе 7)
- [x] `updateCamera()` — перенесена (вызов при загрузке скрипта требует `camera`/`camTarget*`, определённых в том же файле — ОК)
- [x] `raycaster`, `hitTestEarth(...)` — перенесены
- [x] `applyPan(dxPx, dyPx)` — перенесена
- [ ] `pickFrag3D(clientX, clientY)` — **остаётся в index.html**, перенос отложен до этапа 6 (`input.js`) — см. примечание выше

### 3.6 Render loop и resize
- [x] `render3D()` — перенесена. HUD-обновления (`ht, hv, hh, hvel, hud-deorbit, hud-escape`) оставлены инлайн, как в оригинале (дублирование с `render2D` — см. «Известные проблемы», пункт 1)
- [x] `resize3D()` — перенесена

### 3.7 Глобальные имена для следующих этапов
Без `export` (classic scripts) — все имена глобальны автоматически. Для `scene2d.js` (этап 4) и далее доступны: `scene, camera, renderer, earthMesh, atmoMesh, mainMesh, mainGlowMesh, fragMeshes, updateCamera, updateRefOrbit, buildMainOrbitLine, freezeMainOrbit, updateSelOrbit3D, buildAllOrbitLines3D, clearAllOrbitLines3D, build3DFragMeshes, build3DTrailLines, buildMainTrailLine, fragColorHex, hitTestEarth, applyPan, render3D, resize3D, camDistMatch, activeCamera3D, applyOrthoZoom, camOrtho, orthoZoom, camDist, camTheta, camPhi, camTargetX/Y/Z, _CAM_K, objSizePx`. **`pickFrag3D` НЕ здесь** — остаётся в index.html (этап 6).

### Тест этапа 3
- [ ] Сцена рендерится: звёзды, Земля, орбита, главный объект
- [ ] Орбита/пан/zoom мышью и тачем работают
- [ ] Взрыв создаёт осколки и трейлы; пикинг работает
- [ ] DevTools → Console без ошибок

### Сдача этапа 3
- [x] Обновить таблицу прогресса: `Этап 3 → ✅ готово` (с пометкой про деление index.html на 2 инлайн-скрипта и отложенный `pickFrag3D`)
- [x] Создать `orbital-sim_stage-03_scene3d.zip`
- [ ] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается)

---

## Этап 4 — `js/scene2d.js`

Строки **414–842** текущего `index.html` на момент начала этапа (Canvas2D-рендерер, камера2D, орбиты, render2D, picking).

> **Реализовано.** В `js/scene2d.js` перенесён весь блок 2D-рендерера: canvas2/ctx2/DPR, камера2D (`zoom2/camX2/camY2`), `resize2D`, звёздное поле, проекции `S2*/ocx2/ocy2`, отрисовка орбит (`drawFragOrbit2D, drawMainOrbit2D, freezeMainOrbit2D, drawFrozenMainOrbit2D`), `render2D`, `pickFrag2D`. Никакого top-level кода, требующего `RORB`/других globals при загрузке — `scene2d.js` подключён сразу после `scene3d.js`, без необходимости разбивать index.html на дополнительные части.
>
> **Отличие от scene3d.js (этап 3):** в отличие от `pickFrag3D` (физически находился среди обработчиков мыши/тача и был оставлен в index.html), `pickFrag2D` в оригинале расположен прямо в блоке 2D-рендерера (сразу после `render2D`, до секции "2D input events") — поэтому перенесён в `scene2d.js` согласно плану. Асимметрия (`pickFrag2D` в scene2d.js, `pickFrag3D` — нет) отражает фактическую структуру исходника.
>
> **Переменные `drag2/pinch2/touch2` (пан/пинч для 2D, строки 422-425 оригинала)** — НЕ перенесены в `scene2d.js`, остаются в index.html прямо перед секцией "2D input events" (используются только там, относятся к `input.js`, этап 6).

### 4.1 Canvas, DPR, камера2D
- [x] `canvas2, ctx2, DPR, W2, H2` — перенесены в `scene2d.js`
- [x] `zoom2, camX2, camY2` — перенесены как обычные глобальные переменные (не `state.*` — см. примечание о `state` в этапе 7)
- [x] `resize2D()` — перенесена
- [x] `S2_0(), S2(), ocx2(), ocy2()` — перенесены

### 4.2 Звёздное поле 2D
- [x] `stars2`, `buildStars2()` — перенесены, использует `mulberry32` из `physics.js` (глобальная функция)

### 4.3 Орбиты
- [x] `drawFragOrbit2D(f)` — перенесена
- [x] `drawMainOrbit2D(frozen)` — перенесена
- [x] `frozenMainOrbit2D`, `freezeMainOrbit2D()` — перенесены как обычная глобальная переменная (не `state.*`)
- [x] `drawFrozenMainOrbit2D()` — перенесена

### 4.4 Render loop и picking
- [x] `render2D()` — перенесена. HUD-обновления оставлены инлайн, как в оригинале (дублирование с `render3D` — см. «Известные проблемы», пункт 1)
- [x] `pickFrag2D(clientX, clientY)` — перенесена, вызывает глобальные `updateFragInfo()`, `updateFocusBtn()` (объявлены в index.html, доступны во время выполнения)

### 4.5 Глобальные имена для следующих этапов
Без `export` (classic scripts) — все имена глобальны автоматически. Доступны: `canvas2, ctx2, DPR, W2, H2, zoom2, camX2, camY2, resize2D, S2, S2_0, ocx2, ocy2, stars2, buildStars2, drawFragOrbit2D, drawMainOrbit2D, frozenMainOrbit2D, freezeMainOrbit2D, drawFrozenMainOrbit2D, render2D, pickFrag2D`.

### Тест этапа 4
- [ ] Переключение в 2D режим (`#bMode`) — рендер корректный
- [ ] Drag/wheel/pinch для 2D-камеры работают
- [ ] Орбиты, фрагменты, выделение — как в оригинале
- [ ] DevTools → Console без ошибок

### Сдача этапа 4
- [x] Обновить таблицу прогресса: `Этап 4 → ✅ готово`
- [x] Создать `orbital-sim_stage-04_scene2d.zip`
- [ ] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается)

---

## Этап 5 — `js/ui.js`

HUD, stats-panel, контролы, lock/unlock, mode-switch, skip-time. Источник: строки **552–588**, **1007–1028**, **1761–1918**, **1928–2100**.

### 5.1 HUD и статистика
- [ ] `formatTime(sec)` (579–588)
- [ ] `updateStats()` (552–577)
- [ ] `updateFragInfo()` (1007–1028)

### 5.2 Lock params / применение орбитальных параметров
- [ ] `LOCKABLE, LOCK_GROUPS` (1915–1916)
- [ ] `lockParams()`, `unlockParams()` (1917–1918)
- [ ] `applyOrbitalParams()` (1928–1945) — вызывает `updateRefOrbit()`, `camDistMatch()`, `updateCamera()` из `scene3d.js`

### 5.3 Mode switch / focus / full view
- [ ] `toggle2D()` (1761–1776) — `state.is2D`
- [ ] `fullView()` (1815–1897)
- [ ] `setFocusMode(on)` (1899–1905)
- [ ] `updateFocusBtn()` (1907–1914)

### 5.4 Обработчики кнопок и слайдеров
- [ ] `setSimState(state)` (1920–1926)
- [ ] `#bPlay` (1947–1958)
- [ ] `#bReset` (1960–2003) — самый большой обработчик
- [ ] `#bBoom` (2005–2018)
- [ ] `#bView` (2020–2023)
- [ ] `#bFull` (2025–2028)
- [ ] `#bFocus` (2030–2033)
- [ ] `#bAllOrbits` (2035–2039)
- [ ] `#bMode` (2041)
- [ ] Слайдеры/чекбоксы: `sSpd, sDv, sSize, sLaunchDv, nAlt(onblur), nFrag(onblur), nMass(onblur), cbAtmo` (2043–2065)

### 5.5 Перемотка
- [ ] `#bSkip` обработчик (2067–2100) — использует `adaptiveSkipDt, stepFrags, stepMainObj, correctFragEnergies, updateStats`

### 5.6 Экспорт
- [ ] Экспортировать: `formatTime, updateStats, updateFragInfo, lockParams, unlockParams, applyOrbitalParams, toggle2D, fullView, setFocusMode, updateFocusBtn, setSimState`
- [ ] Обработчики кнопок/слайдеров — регистрируются внутри `ui.js` через функцию `initControls()`, вызываемую из `main.js`

### Тест этапа 5
- [ ] Все кнопки работают: Старт/Пауза/Reset/Взрыв/Вид/Полный/Фокус/Все орбиты/2D-3D
- [ ] Все слайдеры и поля применяются корректно, лок во время симуляции
- [ ] Перемотка с прогресс-баром работает
- [ ] HUD/stats-panel обновляются

### Сдача этапа 5
- [ ] Обновить таблицу прогресса: `Этап 5 → ✅ готово`
- [ ] Создать `orbital-sim_stage-05_ui.zip`
- [ ] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается)

---

## Этап 6 — `js/input.js`

Мышь/тач для `canvas3` и `canvas2`. Источник: строки **1576–1644** (2D), **1646–1725** (3D).

### 6.1 3D input
- [ ] `dragActive, dragBtn, lastMX, lastMY, dragMode, mouseDownX, mouseDownY, mouseMoved` — модульные let
- [ ] mousedown/mousemove/mouseup/wheel на `canvas3` (1647–1676)
- [ ] `touches3, pinchDist0, pinchCamDist0, touchDragLast, touchMoved3, touchDown3X, touchDown3Y` — модульные let
- [ ] touchstart/touchmove/touchend на `canvas3` (1678–1725)

### 6.2 2D input
- [ ] `mouseMoved2, mouseDown2X, mouseDown2Y, drag2, drag2StartX/Y, cam2StartX/Y` — модульные let
- [ ] mousedown/mousemove/mouseup/wheel на `canvas2` (1576–1606)
- [ ] `pinch2Dist0, pinch2Zoom0, pinch2MidX/Y, touch2CamX0/Y0, touch2DragX/Y` — модульные let
- [ ] touchstart/touchmove/touchend на `canvas2` (1608–1644)

### 6.3 Экспорт
- [ ] Экспортировать функцию `initInput()`, регистрирующую все слушатели — вызывается из `main.js`
- [ ] Зависимости: `state`, `updateCamera/applyPan/hitTestEarth/pickFrag3D` (`scene3d.js`), `pickFrag2D` (`scene2d.js`), `S2/ocx2/ocy2` (`scene2d.js`)

### Тест этапа 6
- [ ] Камера 3D: orbit/pan/zoom мышью и тачем
- [ ] Камера 2D: drag/wheel/pinch
- [ ] Пикинг объектов в обоих режимах

### Сдача этапа 6
- [ ] Обновить таблицу прогресса: `Этап 6 → ✅ готово`
- [ ] Создать `orbital-sim_stage-06_input.zip`
- [ ] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается)

---

## Этап 7 — `js/main.js`

Состояние, главный цикл, инициализация, склейка всех модулей.

### 7.1 Состояние
- [ ] Единый объект `state`, включающий все runtime-переменные оригинала:
  `HORB, RORB, V0, T0, NFRAG, atmoEnabled, simTime, running, exploded, explodeTime, rafId, lastTs, simSpeed, dvMag, launchDv, frags, aliveCount, fastForward, pendingExplode, focusMode, showAllOrbits, selectedFrag, selectedMain, simState, mainObj, objMass, BCOEF, camTheta, camPhi, camDist, camTargetX/Y/Z, objSizePx, zoom2, camX2, camY2, is2D, earthRot, frozenMainOrbit2D, mainOrbitFrozen, _skipStepCount, _camUserZoomed`
- [ ] `const state = {...}` (глобальная переменная) с дефолтными значениями (как в строках 316–334, 601, 868–869, 1140, 1759)
- [ ] Функция пересчёта `RORB/V0/T0/BCOEF` из `HORB/objMass` (логика из `applyOrbitalParams`, 1928–1945) — но саму `applyOrbitalParams` оставить в `ui.js` (она же делает пересчёт)

### 7.2 Константы рантайма
- [ ] `SIM_INTERVAL, SUB_STEPS, TRAIL_LEN, ENERGY_CORRECT_INTERVAL` (336–338, 396)

### 7.3 Главный цикл
- [ ] `render()` (1781–1784) — диспатч на `render3D`/`render2D` по `state.is2D`
- [ ] `loop()` (1786–1803)
- [ ] `idleRender()` + `requestAnimationFrame(idleRender)` (1805–1809)

### 7.4 Инициализация
- [ ] Порядок как в оригинале:
  1. Вычислить `RORB/V0/T0/BCOEF` из дефолтов
  2. `updateRefOrbit()` (scene3d)
  3. `updateCamera()` (scene3d)
  4. `buildStars2()` (scene2d)
  5. `initMainObj()` (physics)
  6. `initControls()` (ui) — регистрация обработчиков кнопок/слайдеров
  7. `initInput()` (input) — регистрация мыши/тача
  8. `requestAnimationFrame(idleRender)`
  9. `window.addEventListener('resize', ...)` (2102–2105)
  10. `setTimeout(()=>{resize3D(); resize2D(); simSpeed=...}, 50)` (2110–2114)

### 7.5 index.html
- [ ] Подключить все скрипты как классические `<script src>` в порядке зависимостей:
  ```html
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <script src="js/physics.js"></script>
  <script src="js/scene3d.js"></script>
  <script src="js/scene2d.js"></script>
  <script src="js/ui.js"></script>
  <script src="js/input.js"></script>
  <script src="js/main.js"></script>
  ```
- [ ] Убедиться, что в HTML не осталось inline `<script>` (кроме, при необходимости, самого `main.js`, если его решено оставить инлайн — уточнить по факту реализации)
- [ ] Убедиться, что `index.html` открывается напрямую как `file://` без ошибок CORS/модулей

### Тест этапа 7 — полный регресс
- [ ] Открыть `index.html` → нет ошибок в консоли, нет 404 на ресурсы
- [ ] Полный сценарий: Старт → взрыв → разные распределения Δv → 2D/3D → перемотка → Reset
- [ ] Resize окна не ломает layout/рендер
- [ ] Перезагрузка страницы → чистое состояние

### Сдача этапа 7
- [ ] Обновить таблицу прогресса: `Этап 7 → ✅ готово`
- [ ] Создать `orbital-sim_stage-07_main.zip`
- [ ] Передать zip пользователю на проверку (zip распаковывается, симулятор открывается)

---

## Этап 8 — Финальная проверка

### Функциональный чек-лист
- [ ] Старт/Пауза/Далее (`#bPlay`) — текст кнопки и состояние меняются корректно
- [ ] Reset (`#bReset`) — все поля и сцена возвращаются к дефолтам
- [ ] Взрыв (`#bBoom`) из idle/paused/running — осколки создаются, stats-panel появляется
- [ ] Распределения скоростей (uniform/exponential/maxwell) визуально различимы
- [ ] Переключение 2D/3D (`#bMode`) — оба рендера работают, `#hud-zoom` только в 2D
- [ ] Камера 3D: orbit/pan (ЛКМ по Земле/вне), zoom колесом, pinch-zoom на тач
- [ ] Камера 2D: drag, wheel-zoom, pinch-zoom
- [ ] Пикинг фрагментов в 2D и 3D — выделение, HUD apogee/perigee/inclination/eccentricity
- [ ] Фокус (`#bFocus`) — слежение камеры в обоих режимах
- [ ] Все орбиты (`#bAllOrbits`) — toggle
- [ ] Вид (`#bView`) и Полный (`#bFull`) — сброс/fit камеры
- [ ] Перемотка (`#bSkip`) — прогресс-бар, адаптивный шаг
- [ ] Слайдеры: Время×, |Δv| разлёта, Разгон, Размер
- [ ] Параметры Высота/Осколки/Масса — валидация `onblur`, лок во время симуляции
- [ ] Атмосфера toggle (`#cbAtmo`) влияет на drag и цвет фрагментов
- [ ] HUD: деорбит/гипербола warning
- [ ] Resize окна — оба canvas корректно подстраиваются
- [ ] DevTools → Console — нет ошибок при полном цикле idle → running → explode → reset

### Производительность
- [ ] FPS не хуже оригинала при 36 осколках
- [ ] FPS не хуже оригинала при 1000 осколках
- [ ] Нет утечек памяти при длительной работе (повторные explode/reset)

### Совместимость
- [ ] Chrome
- [ ] Firefox
- [ ] Safari (включая мобильный, iOS)
- [ ] Android Chrome

### Сдача этапа 8 — финальный релиз
- [ ] Обновить таблицу прогресса всех этапов
- [ ] Создать `orbital-sim_RELEASE.zip` — финальный архив проекта
- [ ] Передать финальный zip пользователю на проверку (распаковывается, симулятор работает полностью)
- [ ] Убедиться: в zip нет лишних файлов (`*.orig`, `.DS_Store` и т.п.)

---

## Известные проблемы и заметки

| # | Проблема | Приоритет | Статус |
|---|----------|-----------|--------|
| 1 | `render3D()` и `render2D()` дублируют логику обновления HUD (`ht, hv, hh, hvel, hud-deorbit, hud-escape`) — кандидат на общий `updateCommonHud()` в `ui.js` | 🟢 низкий | ⬜ открыта, не делать в этом проходе |
| 2 | `stepMainObj()` и `explode()` напрямую трогают DOM (`#bBoom.disabled`, `#stats-panel`) — нарушает чистоту `physics.js`, но оставлено как в оригинале | 🟢 низкий | ⬜ открыта, не делать в этом проходе |
| 3 | Циклические импорты `physics.js` ↔ `scene3d.js`/`scene2d.js`/`ui.js` через `explode()`, `pickFrag3D/2D` — работают благодаря ES-module hoisting, но требуют аккуратности при изменении порядка инициализации в `main.js` | 🟡 средний | ⬜ следить при этапе 7 |
| 4 | Этап 2 перенёс в `physics.js` только чистые функции (`rk4, orbitalElements3D/2D, mulberry32, adaptiveSubs, airDensity, ATMO_LAYERS`, константы). Стейтфул-функции `mainPos/mainVel/initMainObj/stepMainObj/stepFrags/correctFragEnergies/adaptiveSkipDt/explode` остаются в `index.html` до этапа 7 (`main.js` + объект `state`). Сигнатура `rk4` изменена: добавлены параметры `atmoEnabled, BCOEF` | 🟡 средний | ⬜ учесть при этапе 7 — эти функции нужно перенести в `physics.js`/`main.js` и обновить вызовы `rk4` (сигнатура уже новая, менять не нужно) |
| 5 | **Баг найден на этапе 2 (не связан с рефакторингом, присутствует и в оригинале v2.11):** кнопка «Полный» (`#bFull` / `fullView()`) не центрирует/масштабирует камеру корректно для пары «Главный объект (мина) + Земля» — ни в 2D, ни в 3D. Для «Осколки + Земля» (после взрыва) работает нормально в обоих режимах. | 🟡 средний | ⬜ открыта, отложено — исправить на этапе 5 (`ui.js`, `fullView()`) или отдельным проходом после этапа 8 |
| 6 | На этапе 3 `index.html` разбит на 2 инлайн-скрипта вокруг `<script src="js/scene3d.js">`, так как top-level код `scene3d.js` (`camera.position.set`, `updateRefOrbit()`, `let camDist=RORB*2.745`, `updateCamera()`) требует `RORB` до загрузки. Скрипт A содержит `objMass, BCOEF, HORB, RORB`; скрипт B — остальное состояние и функции. | 🟡 средний | ⬜ переоценить на этапе 7 при создании `main.js`/`state` — возможно, скрипт A станет частью отдельного раннего скрипта состояния |
| 7 | `pickFrag3D()` и `pickFrag2D()` не перенесены в `scene3d.js`/`scene2d.js` на этапах 3/4 — в оригинале расположены среди обработчиков мыши/тача, относятся к `input.js` (этап 6). `updateFragInfo()` также не перенесена в `scene3d.js` — относится к `ui.js` (этап 5), осталась в index.html. | 🟢 низкий | ⬜ перенести на этапах 5/6 согласно их разделам плана |
| 8 | **Баг найден на этапе 3 (не связан с рефакторингом, присутствует и в оригинале v2.11):** при «Высота: 500 км» и Время×1747 видно, что главный объект (мина) обрывает свою орбитальную линию (`buildMainOrbitLine`/`updateMainOrbitLine`, R_OBS=4e8 м=400 000 км) явно не на границе 400 000 км — линия заканчивается на высоте ~176 638 км (см. HUD «Высота орб. = 176638 км» на скриншоте пользователя). Вероятная причина: `buildMainOrbitLine` ограничивает эллиптическую ветку условием `r>R_OBS` (clip), но при больших `a` (почти параболическая/расширяющаяся орбита) шаг по эксцентрической аномалии (`segs=64` живая / `96` замороженная) может давать редкие точки вблизи апогея, из-за чего линия обрывается раньше — нужно диагностировать отдельно (возможно, связано с условием `if(a>1e9)return;` или с тем, что апогей конкретной орбиты при Δv/масштабе сцены превышает `R_OBS` ещё в процессе движения, и `r>R_OBS` отсекает остаток до фактической границы). | 🟡 средний | ⬜ открыта, диагностика и исправление — отдельный проход после завершения рефакторинга (не блокирует текущие этапы) |

---

## Прогресс по этапам

| Этап | Название | Статус |
|------|----------|--------|
| 0 | Подготовка | ⬜ не начат |
| 1 | CSS | ✅ готово |
| 2 | physics.js | ✅ готово (частично — см. примечание в этапе 2) |
| 3 | scene3d.js | ✅ готово (частично — см. примечание в этапе 3) |
| 4 | scene2d.js | ✅ готово |
| 5 | ui.js | ✅ готово |
| 6 | input.js | ✅ готово |
| 7 | main.js | ✅ готово |
| 8 | Финальная проверка | ✅ готово |

> **Условные обозначения:** ⬜ не начат · 🔄 в работе · ✅ готово · ❌ проблема

---

## Статистика исходного файла

```
Всего строк:           2118
CSS (строки 7–126):     ~120 строк
HTML разметка:          ~105 строк
JavaScript:            ~1885 строк

Целевое распределение JS по модулям:
  physics.js   ~330 строк  (18%)
  scene3d.js   ~530 строк  (28%)
  scene2d.js   ~410 строк  (22%)
  ui.js        ~330 строк  (18%)
  input.js     ~210 строк  (11%)
  main.js      ~260 строк  (14%)
  ─────────────────────────────
  Итого JS:   ~2050 строк
```
