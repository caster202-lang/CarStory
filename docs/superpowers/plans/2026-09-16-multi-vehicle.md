# 다중 차량 전환 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 카스토리가 차량 여러 대를 등록·전환하고, 차량마다 기록이 완전히 분리되게 만든다.

**Architecture:** 저장은 `vehicles[]` + 전 차량 기록(각각 `vehicleId` 보유), 메모리의 `vehicle`/`items`/`fuelLogs`/`maintLogs`는 활성 차량만 걸러낸 **뷰**로 둔다. 전역 이름을 그대로 유지하므로 이 값들을 **읽는** 코드는 손대지 않고, **저장하는** 11곳만 병합 저장 헬퍼로 바꾼다.

**Tech Stack:** 단일 HTML 파일(`index.html`), 순수 JS, localStorage. 테스트는 파일에 내장된 `selfTest()`.

## Global Constraints

- 모든 코드는 `index.html` 한 파일 안에 둔다. 파일을 쪼개지 않는다.
- 저장 키는 전부 `carstory_` 로 시작한다. **예전 `mile_*` 키를 읽어오는 코드는 넣지 않는다.**
- 백업 형식은 **3판(`version:3`)** 을 유지한다. 판을 올리지 않는다.
- 주석과 화면 문구는 한국어로 쓰고, 기존 코드의 말투(존댓말 안내 문구, 평서체 주석)를 따른다.
- 새 파일을 만들지 않는다. `docs/` 아래 문서는 예외다.

## 테스트 실행 방법

`file://`로 열면 콘솔을 읽을 수 없다. 반드시 로컬 서버로 띄운다.

```bash
cd "/c/Users/USER/Desktop/차계부" && python -m http.server 8765 --bind 127.0.0.1
```

브라우저에서 `http://localhost:8765/index.html?test&v=<매번 바꾸는 숫자>` 를 연다.
`&v=` 를 매번 바꾸지 않으면 캐시된 예전 파일이 돌아간다.

- 통과: 콘솔에 `selfTest 통과`
- 실패: 콘솔에 `selfTest 실패: <메시지>` 로 throw

작업이 끝나면 PowerShell로 서버를 내린다. Git Bash의 `pkill`은 Windows python 프로세스를 못 잡는다.

```
Get-CimInstance Win32_Process -Filter "Name='python.exe'" | Where-Object { $_.CommandLine -like '*http.server*8765*' } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

---

## File Structure

| 파일 | 책임 | 변경 |
|---|---|---|
| `index.html` | 앱 전체 (마크업 · 스타일 · 로직 · 테스트) | Task 0~6 모두 여기를 고친다 |
| `README.md` | 사용 설명 | Task 2(로고), Task 7 |
| `차계부-앱-기획서.md` | 기획·구현 상태 (gitignore 대상) | Task 7 |

---

### Task 0: selfTest가 진짜 저장소를 건드리지 않게 막는다

**먼저 해야 한다.** 지금 `selfTest()`는 `saveMaintItem()`을 부르고, 그 안에서 실제로
`localStorage`에 쓴다. 검사는 JS 변수만 복원하고 저장소는 되돌리지 않아서,
**`?test`를 한 번 열면 저장된 소모품·교체 이력이 검사용 데이터로 덮어써진다.**

이번 계획의 Task 4·5 검사는 `saveVehicle()`·`deleteActiveVehicle()`까지 부르므로
이걸 먼저 막지 않으면 피해가 커진다.

**Files:**
- Modify: `index.html` — `selfTest()` 의 try/finally

**Interfaces:**
- Consumes: 없음
- Produces: 없음. `selfTest()` 가 도는 동안 `save()` 가 아무것도 쓰지 않는다.

- [ ] **Step 1: 실패하는 검사를 쓴다**

`selfTest()` 안 **맨 첫 검사로** 넣는다 (기존 `a(maintLine(...)` 줄 앞).

```js
    // 검사가 진짜 저장소를 건드리면 ?test 한 번에 사용자 기록이 날아간다
    localStorage.setItem(KEYS.items, '"검사전"');
```

그리고 **`try` 블록의 맨 마지막 검사로** — `console.log('%c selfTest 통과 ', ...)` 줄 **바로 앞**에 넣는다.
`finally` 에 두면 통과 로그가 찍힌 뒤에 예외가 나서 결과가 헷갈린다.

```js
    a(localStorage.getItem(KEYS.items) === '"검사전"', '검사는 저장소를 건드리지 않아야 한다');
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=1` 을 연다.
예상: `selfTest 실패: 검사는 저장소를 건드리지 않아야 한다`
(`saveMaintItem()` 이 `KEYS.items` 를 덮어썼기 때문이다)

- [ ] **Step 3: 검사 동안 저장을 끈다**

`selfTest()` 의 첫 줄과 `finally` 를 바꾼다. `save` 는 함수 선언이라 다시 대입할 수 있다.
검사용 코드를 `save()` 본체에 넣지 않는 게 핵심이다 — 제품 코드에 검사 전용 분기를 만들지 않는다.

`const okVehicle = vehicle, okFuel = fuelLogs;` 줄을 이렇게 바꾼다.

```js
  const okVehicle = vehicle, okFuel = fuelLogs;
  // 검사는 저장소를 건드리지 않는다. 안 막으면 ?test 한 번에 실제 기록이 날아간다.
  const 진짜save = save;
  save = function(){};
```

`finally` 블록을 이렇게 바꾼다.

```js
  } finally {
    save = 진짜save;
    vehicle = okVehicle; fuelLogs = okFuel;
  }
```

`save = 진짜save;` 가 **맨 앞**이어야 한다. 그래야 복원 도중에도 저장이 일어나지 않는다.

- [ ] **Step 4: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=2`
예상: `selfTest 통과`

- [ ] **Step 5: 확인용으로 넣은 줄을 정리한다**

Step 1에서 넣은 `localStorage.setItem(KEYS.items, '"검사전"');` 줄을 지운다.
`finally` 안의 확인 검사는 **남긴다** — 앞으로 저장을 부르는 검사를 또 넣었을 때
같은 사고를 잡아준다. 대신 비교 대상을 "검사 전 값"으로 바꾼다.

`selfTest()` 첫 줄 근처, `const 진짜save = save;` **앞**에 넣는다.

```js
  const 검사전items = localStorage.getItem(KEYS.items);
```

통과 로그 앞의 검사를 바꾼다.

```js
    a(localStorage.getItem(KEYS.items) === 검사전items, '검사는 저장소를 건드리지 않아야 한다');
```

- [ ] **Step 6: 다시 통과를 확인한다**

`http://localhost:8765/index.html?test&v=3`
예상: `selfTest 통과`

- [ ] **Step 7: 커밋한다**

```bash
git add index.html && git commit -m "selfTest가 진짜 저장소를 건드리던 문제 수정

selfTest가 saveMaintItem을 부르고 그 안에서 localStorage에 실제로 썼다.
검사는 JS 변수만 복원하고 저장소는 그대로 둬서, ?test를 한 번 열면
저장된 소모품·교체 이력이 검사용 데이터로 덮어써졌다.

검사 동안 save를 빈 함수로 바꿔 막는다. save 본체에 검사용 분기를
넣지 않으려고 바깥에서 갈아끼우는 방식을 썼다.
저장소가 그대로인지 확인하는 검사도 함께 남겼다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 1: 저장 키 이름을 carstory_로 바꾼다

가장 먼저 한다. 이후 모든 태스크가 새 이름 위에서 진행되도록.

**Files:**
- Modify: `index.html:7` (테마 선반영 인라인 스크립트)
- Modify: `index.html:720-722` (KEYS와 그 위 주석)
- Modify: `index.html:995` (THEME_KEY)
- Modify: `index.html:1259` (BACKUP_KEY)

**Interfaces:**
- Consumes: 없음
- Produces: `KEYS`, `THEME_KEY`, `BACKUP_KEY` — 값이 모두 `carstory_` 접두사로 바뀐다. 이후 태스크는 이 이름을 쓴다.

- [ ] **Step 1: 실패하는 검사를 쓴다**

`selfTest()` 안, 기존 `a(['dark','light'].indexOf(currentTheme()) !== -1, ...)` 줄 **바로 앞**에 넣는다.

```js
    // 저장 키는 앱 이름(carstory)에 맞춘다
    a(Object.values(KEYS).every(k => k.indexOf('carstory_') === 0), '데이터 키는 모두 carstory_로 시작한다');
    a(THEME_KEY.indexOf('carstory_') === 0, '화면 모드 키도 carstory_');
    a(BACKUP_KEY.indexOf('carstory_') === 0, '백업 기록 키도 carstory_');
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=101` 을 연다.
예상: `selfTest 실패: 데이터 키는 모두 carstory_로 시작한다`

- [ ] **Step 3: 키 이름을 바꾼다**

`index.html:7` 의 인라인 스크립트에서 `'mile_theme'` 을 `'carstory_theme'` 으로 바꾼다.

`KEYS` 위의 주석과 정의를 통째로 바꾼다.

```js
// 저장 키. 오픈 전이라 예전 mile_* 키를 읽어오는 코드는 두지 않는다.
// 오픈한 뒤에는 이 이름을 바꾸면 안 된다. 바꾸는 순간 사용자가 쌓은 기록을 못 읽는다.
const KEYS = { vehicle:'carstory_vehicle', items:'carstory_items', fuel:'carstory_fuel', logs:'carstory_logs' };
```

`THEME_KEY`, `BACKUP_KEY` 를 바꾼다.

```js
const THEME_KEY = 'carstory_theme';
```

```js
const BACKUP_KEY = 'carstory_backup';
```

- [ ] **Step 4: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=102` 를 연다.
예상: `selfTest 통과`

기존 검사 `'전체 초기화를 해도 화면 모드 설정은 남아야 함'` 도 계속 통과해야 한다.

- [ ] **Step 5: 커밋한다**

```bash
git add index.html && git commit -m "저장 키를 carstory_로 정리

오픈 전이라 예전 mile_* 키를 읽어오는 코드는 두지 않는다.
한 번 넣으면 영구히 들고 다녀야 하는 코드라서다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: 브랜드를 색 칩 + 이름으로 표시한다

데이터 구조와 무관해서 먼저 끝낸다.

**Files:**
- Modify: `index.html` — `.brand-badge` CSS 아래에 `.brand-chip` 추가
- Modify: `index.html` — `LOGO_BASE`·`BRAND_SLUG` 삭제, `brandLogo()` 삭제, `brandChip()` 추가
- Modify: `index.html` — `vehicleTitle()`, `renderHome()` 의 배지 줄
- Modify: `index.html` — 로고 관련 selfTest 검사 3개 교체
- Modify: `README.md` — "브랜드 로고에 대해" 절 삭제

**Interfaces:**
- Consumes: `BRAND_COLOR`, `esc()`
- Produces:
  - `brandChip(brandName)` → `string`. 브랜드 색 배경에 브랜드 이름을 넣은 `<span class="brand-chip">`. 이름이 없거나 `'기타'` 면 빈 문자열.
  - `vehicleTitle()` → `string`. **모델명만** 돌려준다. 모델명이 없으면 브랜드로 대신하고, 둘 다 없으면 `'차량 미등록'`.

- [ ] **Step 1: 실패하는 검사를 쓴다**

`selfTest()` 안, 기존 `a(brandLogo('BMW')...)` 로 시작하는 **3줄을 지우고** 그 자리에 넣는다.

```js
    a(brandChip('제네시스').indexOf('제네시스') !== -1, '칩에 브랜드 이름이 통째로 들어간다');
    a(brandChip('제네시스').indexOf(BRAND_COLOR['제네시스']) !== -1, '칩에 브랜드 색이 쓰인다');
    a(brandChip('기타') === '', '브랜드가 기타면 칩을 만들지 않는다');
    a(brandChip('') === '', '브랜드가 없으면 칩을 만들지 않는다');
    const 원차칩 = vehicle;
    vehicle = { brand:'제네시스', name:'G80', odo:100 };
    a(vehicleTitle() === 'G80', '제목은 모델명만 보여준다');
    vehicle = { brand:'BMW', name:'', odo:100 };
    a(vehicleTitle() === 'BMW', '모델명이 없으면 브랜드로 대신한다');
    vehicle = { brand:'기타', name:'내 차', odo:100 };
    a(vehicleTitle() === '내 차', '기타 브랜드면 모델명만');
    vehicle = { brand:'제네시스', name:'G80', odo:100 };
    renderHome();
    const 홈표시 = document.getElementById('homeBadgeWrap').innerHTML +
                   document.getElementById('homeVehicleName').textContent;
    a(홈표시.split('제네시스').length - 1 === 1, '홈 카드에 브랜드 이름이 한 번만 나온다');
    vehicle = 원차칩;
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=201`
예상: `ReferenceError: brandChip is not defined`

- [ ] **Step 3: 칩 스타일을 추가한다**

`.brand-badge { ... }` 블록 **바로 뒤**에 넣는다.

```css
  .brand-chip {
    display:inline-flex; align-items:center;
    padding:5px 10px; border-radius:9px;
    color:#fff; font-size:11.5px; font-weight:700;
    letter-spacing:-0.02em; white-space:nowrap; flex:none;
  }
  .cluster-head .brand-chip { font-size:13px; padding:7px 12px; border-radius:11px; }
```

- [ ] **Step 4: brandChip을 만들고 brandLogo를 걷어낸다**

`LOGO_BASE` 상수 줄과 `BRAND_SLUG` 객체 전체를 지운다. `brandLogo()` 함수 전체를 지운다.
`brandBadge()` 는 **남긴다** — 브랜드 선택 드롭다운이 쓰고 있다.

`brandBadge()` 바로 뒤에 넣는다.

```js
// 브랜드 색 바탕에 브랜드 이름을 통째로 넣은 칩.
// 제조사 로고 이미지는 쓰지 않는다. 상표라서 공개·수익화 시 브랜드마다 조건을 따로 확인해야 한다.
function brandChip(name){
  if (!name || name === '기타') return '';
  return '<span class="brand-chip" style="background:' + (BRAND_COLOR[name] || '#6b6f76') + '">' +
    esc(name) + '</span>';
}
```

- [ ] **Step 5: 제목에서 브랜드를 뺀다**

`vehicleTitle()` 을 통째로 바꾼다.

```js
// 브랜드는 칩이 보여주므로 제목은 모델명만 쓴다.
// 모델명을 안 적은 차는 보여줄 게 없으니 브랜드로 대신한다.
function vehicleTitle(){
  if (!vehicle) return '차량 미등록';
  const brand = vehicle.brand === '기타' ? '' : vehicle.brand;
  return (vehicle.name || '').trim() || brand || '차량 미등록';
}
```

- [ ] **Step 6: 홈 카드가 칩을 쓰게 한다**

`renderHome()` 의 아래 두 줄을

```js
  const brand = vehicle && vehicle.brand && vehicle.brand !== '기타' ? vehicle.brand : null;
  document.getElementById('homeBadgeWrap').innerHTML = brandLogo(brand);
```

이렇게 바꾼다.

```js
  // 제목이 모델명을 보여줄 때만 칩을 붙인다.
  // 모델명이 없으면 제목이 브랜드를 대신 쓰므로 칩까지 붙이면 같은 말이 두 번 나온다.
  const 칩쓸까 = !!(vehicle && (vehicle.name || '').trim() && vehicle.brand);
  document.getElementById('homeBadgeWrap').innerHTML = 칩쓸까 ? brandChip(vehicle.brand) : '';
```

- [ ] **Step 7: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=202`
예상: `selfTest 통과`

- [ ] **Step 8: README에서 로고 절을 지운다**

`## 브랜드 로고에 대해` 제목부터 그 절 끝(`제조사별 사용 조건을 따로 확인해야 합니다.` 줄)까지 지운다.
`**차량**` 항목의 `홈 차량 카드에 실제 브랜드 로고.` 문장을
`홈 차량 카드에 브랜드 색 이름표.` 로 바꾼다.

- [ ] **Step 9: 커밋한다**

```bash
git add index.html README.md && git commit -m "제조사 로고를 브랜드 색 이름표로 교체

수익화를 염두에 둔 공개라 제조사 로고는 브랜드마다 상표 사용 조건을
확인해야 한다. 브랜드 선택 드롭다운이 이미 색 배지 + 이름으로
그려지고 있어서, 홈 카드만 혼자 로고를 쓰던 것을 맞췄다.

제목은 모델명만 남겼다. 칩과 제목에 브랜드가 두 번 나오기 때문이다.
모델명을 안 적은 차는 제목이 브랜드를 대신 쓰고 칩을 붙이지 않는다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: 데이터 계층을 다중 차량으로 바꾼다

화면은 그대로 두고 속만 바꾼다. 이 태스크가 끝나도 사용자 눈에는 변화가 없어야 한다.

**Files:**
- Modify: `index.html` — `KEYS`와 전역 선언부
- Modify: `index.html` — `save(KEYS.*)` 호출 11곳
- Modify: `index.html` — `resetAll()`, `loadSampleData()`
- Modify: `index.html` — 부팅부(파일 맨 아래 `renderAll();` 앞)

**Interfaces:**
- Consumes: Task 1의 `KEYS`
- Produces:
  - `vehicles: Array` · `activeId: string|null` · `allItems/allFuel/allLogs: Array` — 저장되는 원본
  - `vehicle/items/fuelLogs/maintLogs` — 활성 차량만 걸러낸 뷰 (이름과 의미는 지금과 같다)
  - `newVehicleId()` → `string`
  - `selectVehicle(id)` → `void`. 뷰 네 개를 다시 만들고 `activeId`를 저장한다. `id`가 목록에 없으면 첫 번째 차로, 목록이 비었으면 `null`로 맞춘다.
  - `saveItems()` · `saveFuel()` · `saveLogs()` · `saveVehicles()` → `void`

- [ ] **Step 1: 실패하는 검사를 쓴다**

`selfTest()` 안, 맨 마지막 검사(`'모든 브랜드에 공식/국내권장/설명이 있어야 함'`) **바로 뒤**에 넣는다.

```js
    // ---- 다중 차량: 차마다 기록이 완전히 갈린다 ----
    const 원다중 = { vehicles, activeId, allItems, allFuel, allLogs, vehicle, items, fuelLogs, maintLogs };
    vehicles = [{ id:'vA', brand:'BMW', name:'320d', fuel:'diesel', odo:1000 },
                { id:'vB', brand:'기아', name:'K5', fuel:'gas', odo:2000 }];
    allItems = [{ id:'i1', name:'엔진오일', vehicleId:'vA' },
                { id:'i2', name:'와이퍼',   vehicleId:'vB' }];
    allFuel  = [{ id:'f1', date:'2026-09-01', liters:40, cost:70000, odo:1000, full:true, vehicleId:'vA' }];
    allLogs  = [{ id:'L1', name:'엔진오일', date:'2026-09-01', odo:1000, cost:85000, vehicleId:'vA' }];

    selectVehicle('vA');
    a(vehicle.name === '320d', 'A를 고르면 A가 활성 차량이 된다');
    a(items.length === 1 && items[0].name === '엔진오일', 'A의 소모품만 보인다');
    a(fuelLogs.length === 1 && maintLogs.length === 1, 'A의 주유·이력만 보인다');

    selectVehicle('vB');
    a(vehicle.name === 'K5', 'B를 고르면 B가 활성 차량이 된다');
    a(items.length === 1 && items[0].name === '와이퍼', 'B의 소모품만 보인다');
    a(fuelLogs.length === 0 && maintLogs.length === 0, 'B는 주유·이력이 비어 있다');

    // 저장은 활성 차량 몫만 갈아끼워야 한다
    items.push({ id:'i3', name:'배터리' });
    saveItems();
    a(allItems.length === 3, '저장하면 전체 목록에 더해진다');
    a(allItems.filter(x => x.vehicleId === 'vA').length === 1, '저장해도 A 기록은 그대로다');
    a(items.every(x => x.vehicleId === 'vB'), '저장하면서 활성 차량 id가 찍힌다');

    selectVehicle('vA');
    a(items.length === 1 && items[0].name === '엔진오일', 'A로 돌아오면 A 기록만 보인다');

    // 없는 id를 고르면 첫 번째 차로 떨어진다
    selectVehicle('없는id');
    a(activeId === 'vA', '없는 차를 고르면 첫 번째 차로 맞춘다');

    // 차가 없으면 빈 상태
    vehicles = [];
    selectVehicle(null);
    a(vehicle === null && items.length === 0, '차가 없으면 빈 상태가 된다');

    vehicles = 원다중.vehicles; activeId = 원다중.activeId;
    allItems = 원다중.allItems; allFuel = 원다중.allFuel; allLogs = 원다중.allLogs;
    vehicle = 원다중.vehicle; items = 원다중.items;
    fuelLogs = 원다중.fuelLogs; maintLogs = 원다중.maintLogs;
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=301`
예상: `ReferenceError: vehicles is not defined`

- [ ] **Step 3: 저장 키와 전역을 바꾼다**

`KEYS` 정의를 바꾼다.

```js
const KEYS = { vehicles:'carstory_vehicles', active:'carstory_active',
               items:'carstory_items', fuel:'carstory_fuel', logs:'carstory_logs' };
```

전역 선언부(`let vehicle = load(...)` 부터 `let maintLogs = load(...)` 까지)를 통째로 바꾼다.

```js
// 저장되는 원본 — 전 차량 것이 한 덩어리로 들어 있고, 기록마다 vehicleId가 붙어 있다
let vehicles = load(KEYS.vehicles, []);
let activeId = load(KEYS.active, null);
let allItems = load(KEYS.items, []);
let allFuel  = load(KEYS.fuel, []);
let allLogs  = load(KEYS.logs, []);

// 화면이 쓰는 뷰 — 활성 차량 것만 걸러낸 복사본.
// 이름을 예전 그대로 두는 게 핵심이다. 이 값들을 읽기만 하는 코드는 한 줄도 고칠 필요가 없다.
let vehicle = null, items = [], fuelLogs = [], maintLogs = [];
```

- [ ] **Step 4: 전환과 병합 저장을 만든다**

전역 선언 바로 뒤에 넣는다.

```js
function newVehicleId(){
  return 'v' + Date.now().toString(36) + Math.random().toString(36).slice(2, 6);
}
// 활성 차량을 바꾸고 뷰 네 개를 다시 만든다.
// 없는 id를 주면 첫 번째 차로, 차가 하나도 없으면 빈 상태로 맞춘다.
function selectVehicle(id){
  const found = vehicles.find(v => v.id === id) || vehicles[0] || null;
  activeId  = found ? found.id : null;
  vehicle   = found;
  items     = allItems.filter(x => x.vehicleId === activeId);
  fuelLogs  = allFuel.filter(x  => x.vehicleId === activeId);
  maintLogs = allLogs.filter(x  => x.vehicleId === activeId);
  save(KEYS.active, activeId);
}
// 뷰는 복사본이라 그대로 저장하면 다른 차 기록이 통째로 날아간다.
// 전체 목록에서 활성 차량 몫만 걷어내고 지금 뷰로 갈아끼운다.
function saveItems(){
  items = items.map(x => x.vehicleId === activeId ? x : {...x, vehicleId: activeId});
  allItems = allItems.filter(x => x.vehicleId !== activeId).concat(items);
  save(KEYS.items, allItems);
}
function saveFuel(){
  fuelLogs = fuelLogs.map(x => x.vehicleId === activeId ? x : {...x, vehicleId: activeId});
  allFuel = allFuel.filter(x => x.vehicleId !== activeId).concat(fuelLogs);
  save(KEYS.fuel, allFuel);
}
function saveLogs(){
  maintLogs = maintLogs.map(x => x.vehicleId === activeId ? x : {...x, vehicleId: activeId});
  allLogs = allLogs.filter(x => x.vehicleId !== activeId).concat(maintLogs);
  save(KEYS.logs, allLogs);
}
function saveVehicles(){
  save(KEYS.vehicles, vehicles);
  save(KEYS.active, activeId);
}
```

- [ ] **Step 5: 저장 호출 11곳을 바꾼다**

각 줄을 아래 표대로 바꾼다. 같은 줄에 여러 개가 있으면 전부 바꾼다.

| 기존 | 바꿀 것 |
|---|---|
| `save(KEYS.vehicle, vehicle)` | `saveVehicles()` |
| `save(KEYS.items, items)` | `saveItems()` |
| `save(KEYS.fuel, fuelLogs)` | `saveFuel()` |
| `save(KEYS.logs, maintLogs)` | `saveLogs()` |

`importData()` 안의 네 개가 한 줄에 몰려 있는 곳은 Task 6에서 다시 손대므로, 지금은 아래처럼만 바꿔둔다.

```js
    saveVehicles(); saveItems(); saveFuel(); saveLogs();
```

주행거리 갱신 버튼의 `vehicle.odo = n; save(KEYS.vehicle, vehicle);` 은
뷰와 원본이 같은 객체를 가리키므로 `vehicle.odo = n; saveVehicles();` 로 바꾸면 된다.

- [ ] **Step 6: resetAll과 예시 데이터를 고친다**

`resetAll()` 의 `localStorage.removeItem` 줄들과 전역 초기화를 바꾼다.

```js
  Object.values(KEYS).forEach(k => localStorage.removeItem(k));
  localStorage.removeItem(BACKUP_KEY);
  vehicles = []; allItems = []; allFuel = []; allLogs = [];
  selectVehicle(null);
```

`loadSampleData()` 에서 예시를 넣는 부분을 바꾼다. 예시는 **차 한 대를 새로 만들어** 담는다.

```js
  const v = { id: newVehicleId(), ...s.vehicle };
  vehicles.push(v);
  saveVehicles();
  selectVehicle(v.id);
  items = s.items.map(x => ({...x}));
  fuelLogs = s.fuelLogs.map(x => ({...x}));
  maintLogs = [];
  saveItems(); saveFuel(); saveLogs();
```

- [ ] **Step 7: 부팅할 때 활성 차량을 고르게 한다**

파일 맨 아래 `renderAll();` **바로 앞**에 넣는다.

```js
selectVehicle(activeId);
```

- [ ] **Step 8: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=302`
예상: `selfTest 통과`

- [ ] **Step 9: 눈으로 확인한다**

`http://localhost:8765/index.html?v=303` 을 열고 차량 탭 → 예시 데이터를 넣는다.
홈·소모품·주유 탭이 지금까지와 똑같이 보여야 한다. 사용자 눈에 보이는 변화가 없어야 맞다.

- [ ] **Step 10: 커밋한다**

```bash
git add index.html && git commit -m "데이터 계층을 다중 차량 구조로

vehicles[] + activeId로 저장하고, 화면이 쓰는 vehicle/items/fuelLogs/
maintLogs는 활성 차량만 걸러낸 뷰로 둔다. 전역 이름을 그대로 유지해서
읽기만 하는 코드는 손대지 않고 저장하는 11곳만 병합 저장으로 바꿨다.

읽는 곳을 전부 filter로 바꾸는 쪽은 하나만 놓쳐도 다른 차 기록이
섞여 보이는데 눈에 잘 안 띈다. 저장은 틀리면 바로 드러난다.

화면은 아직 그대로다. 전환 UI는 다음 커밋.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: 차량 전환·추가 화면

**Files:**
- Modify: `index.html` — 홈 차량 카드 마크업, 차량 목록 모달 추가, `vEditId` 히든 필드 추가
- Modify: `index.html` — `saveVehicle()`, `renderVehicleForm()`
- Modify: `index.html` — 목록 모달 CSS

**Interfaces:**
- Consumes: Task 2의 `brandChip()`, Task 3의 `vehicles`/`activeId`/`selectVehicle()`/`saveVehicles()`/`newVehicleId()`
- Produces:
  - `openVehicleList()` · `closeVehicleList()` → `void`
  - `pickVehicle(id)` → `void`. 그 차로 전환하고 모달을 닫는다.
  - `startNewVehicle()` → `void`. 모달을 닫고 차량 탭의 **빈 폼**을 띄운다.
  - `renderVehicleList()` → `void`. 모달 안 목록을 그린다.

- [ ] **Step 1: 실패하는 검사를 쓴다**

Task 3에서 넣은 다중 차량 검사 블록의 **복원 줄 바로 앞**에 넣는다.

```js
    // 차를 고치면 id가 유지돼야 한다. id가 바뀌면 그 차 기록이 전부 미아가 된다.
    vehicles = [{ id:'vA', brand:'BMW', name:'320d', fuel:'diesel', year:2021, odo:1000 }];
    allItems = [{ id:'i1', name:'엔진오일', vehicleId:'vA' }];
    selectVehicle('vA');
    document.getElementById('vEditId').value = 'vA';
    document.getElementById('vBrand').value = 'BMW';
    document.getElementById('vMakeModel').value = '330i';
    document.getElementById('vFuel').value = 'gas';
    document.getElementById('vYear').value = '2022';
    document.getElementById('vOdo').value = '1500';
    saveVehicle();
    a(vehicles.length === 1, '고치는 것이지 새로 만드는 게 아니다');
    a(vehicles[0].id === 'vA', '차를 고쳐도 id가 유지된다');
    a(vehicles[0].name === '330i' && vehicles[0].fuel === 'gas', '고친 내용이 반영된다');
    a(items.length === 1, 'id가 유지되므로 기록도 그대로 붙어 있다');

    // 새 차를 저장하면 목록에 더해지고 그 차가 활성이 된다
    document.getElementById('vEditId').value = '';
    document.getElementById('vBrand').value = '기아';
    document.getElementById('vMakeModel').value = 'K5';
    document.getElementById('vFuel').value = 'gas';
    document.getElementById('vYear').value = '';
    document.getElementById('vOdo').value = '10';
    saveVehicle();
    a(vehicles.length === 2, '새 차가 목록에 더해진다');
    a(vehicle.name === 'K5', '새로 만든 차가 활성 차량이 된다');
    a(items.length === 0, '새 차는 기록이 비어 있다');
    a(vehicles[0].id !== vehicles[1].id, '차마다 id가 다르다');

    // 목록에 등록된 차가 모두 나온다
    renderVehicleList();
    const 차목록 = document.getElementById('vehicleList').innerHTML;
    a(차목록.indexOf('330i') !== -1 && 차목록.indexOf('K5') !== -1, '목록에 등록된 차가 모두 나온다');
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=401`
예상: `TypeError: Cannot read properties of null (reading 'value')` — `vEditId` 가 아직 없다.

- [ ] **Step 3: 마크업을 추가한다**

홈 차량 카드의 `<div class="cluster-head">` 줄을 이렇게 바꾼다.

```html
        <div class="cluster-head" id="homeCarHead" onclick="openVehicleList()">
```

차량 탭 폼의 `<label>모델명</label>` 이 든 `<div class="field">` **바로 앞**에 히든 필드를 넣는다.

```html
          <input type="hidden" id="vEditId">
```

`fuelModalBackdrop` 모달 **바로 뒤**에 차량 목록 모달을 넣는다.

```html
<div class="modal-backdrop" id="vehicleListBackdrop">
  <div class="modal">
    <h2>내 차</h2>
    <div id="vehicleList"></div>
    <div class="modal-actions">
      <button class="btn" onclick="closeVehicleList()">닫기</button>
      <button class="btn primary" onclick="startNewVehicle()">차 추가</button>
    </div>
  </div>
</div>
```

- [ ] **Step 4: 목록 스타일을 추가한다**

`.brand-chip` 블록 뒤에 넣는다.

```css
  #homeCarHead { cursor:pointer; }
  .veh-row {
    display:flex; align-items:center; gap:10px; width:100%;
    padding:11px 12px; margin-bottom:8px;
    border:1px solid var(--line); border-radius:12px;
    background:var(--surface); color:var(--ink);
    font-family:var(--font); font-size:13.5px; text-align:left; cursor:pointer;
  }
  .veh-row.on { border-color:var(--accent); background:var(--accent-soft); }
  .veh-row .veh-sub { margin-left:auto; font-size:11.5px; color:var(--ink-dim); }
```

- [ ] **Step 5: 목록 동작을 만든다**

`renderVehicleForm()` **바로 앞**에 넣는다.

```js
// ---------- 차량 고르기 ----------
function openVehicleList(){
  if (!vehicles.length) return;   // 등록된 차가 없으면 열 것이 없다
  renderVehicleList();
  document.getElementById('vehicleListBackdrop').classList.add('open');
}
function closeVehicleList(){
  document.getElementById('vehicleListBackdrop').classList.remove('open');
}
function renderVehicleList(){
  const el = document.getElementById('vehicleList');
  if (!el) return;
  el.innerHTML = vehicles.map(v => {
    const 이름 = (v.name || '').trim() || (v.brand === '기타' ? '' : v.brand) || '이름 없는 차';
    const 칩 = (v.name || '').trim() && v.brand ? brandChip(v.brand) : '';
    const 거리 = v.odo != null ? v.odo.toLocaleString() + 'km' : '';
    return '<button class="veh-row' + (v.id === activeId ? ' on' : '') + '" ' +
      'onclick="pickVehicle(\'' + v.id + '\')">' + 칩 +
      '<span>' + esc(이름) + '</span>' +
      '<span class="veh-sub">' + 거리 + '</span></button>';
  }).join('');
}
function pickVehicle(id){
  selectVehicle(id);
  closeVehicleList();
  renderAll();
  switchScreen('home');
}
// 새 차를 만드는 입구는 여기 하나뿐이다.
// 차량 탭을 그냥 열었을 때는 선택된 차를 고치는 화면이라 새 차가 생기지 않는다.
function startNewVehicle(){
  closeVehicleList();
  document.getElementById('vEditId').value = '';
  setBrand('');
  document.getElementById('vMakeModel').value = '';
  document.getElementById('vFuel').value = '';
  document.getElementById('vYear').value = '';
  document.getElementById('vOdo').value = '';
  switchScreen('vehicle');
}
```

- [ ] **Step 6: 저장이 id를 지키게 한다**

`saveVehicle()` 을 통째로 바꾼다.

```js
function saveVehicle(){
  const brand = document.getElementById('vBrand').value;
  const name = document.getElementById('vMakeModel').value.trim();
  const year = numOf('vYear');
  const odo = numOf('vOdo');
  if (!brand && !name) { alert('브랜드나 모델명 중 하나는 입력해주세요'); return; }
  const fuel = document.getElementById('vFuel').value;
  const data = { brand, name, fuel, year, odo: odo != null ? odo : 0 };
  const editId = document.getElementById('vEditId').value;
  // id를 새로 만들면 그 차에 붙어 있던 기록이 전부 미아가 된다. 고칠 때는 반드시 유지한다.
  if (editId){
    const i = vehicles.findIndex(v => v.id === editId);
    vehicles[i] = {...vehicles[i], ...data};
    saveVehicles();
    selectVehicle(editId);
  } else {
    const v = { id: newVehicleId(), ...data };
    vehicles.push(v);
    saveVehicles();
    selectVehicle(v.id);
  }
  renderAll();
  switchScreen('home');
}
```

`renderVehicleForm()` 이 폼을 채울 때 `vEditId` 도 같이 채우게, 함수 안 맨 앞에 넣는다.

```js
  document.getElementById('vEditId').value = vehicle ? vehicle.id : '';
```

- [ ] **Step 7: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=402`
예상: `selfTest 통과`

- [ ] **Step 8: 눈으로 확인한다**

`http://localhost:8765/index.html?v=403` 에서
차량 등록 → 홈 카드 클릭 → "차 추가" → 두 번째 차 등록 → 카드 클릭해서 전환.
각 차의 소모품·주유 탭이 서로 섞이지 않는지 본다.

- [ ] **Step 9: 커밋한다**

```bash
git add index.html && git commit -m "차량 전환·추가 화면

홈 차량 카드를 누르면 등록된 차 목록이 뜨고 골라서 바꾼다.
목록의 '차 추가'가 새 차를 만드는 유일한 입구이고, 차량 탭을 그냥
열면 지금처럼 선택된 차를 고치는 화면이다.

saveVehicle이 차량 객체를 통째로 새로 만들고 있어서 그대로 두면
고칠 때마다 id가 날아가고 그 차 기록이 전부 미아가 됐다. vEditId를
두고 id를 유지하게 고쳤다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: 차량 삭제

**Files:**
- Modify: `index.html` — 차량 탭에 삭제 버튼 추가
- Modify: `index.html` — `deleteVehicleCounts()`, `deleteActiveVehicle()` 추가

**Interfaces:**
- Consumes: Task 3의 `vehicles`/`allItems`/`allFuel`/`allLogs`/`selectVehicle()`/`saveVehicles()`, Task 2의 `vehicleTitle()`
- Produces:
  - `deleteVehicleCounts(id)` → `{items:number, fuel:number, logs:number}`
  - `deleteActiveVehicle()` → `void`

- [ ] **Step 1: 실패하는 검사를 쓴다**

Task 4에서 넣은 검사들 **뒤**, 복원 줄 앞에 넣는다.

```js
    // 차를 지우면 그 차 기록만 사라지고 다른 차 것은 남는다
    vehicles = [{ id:'vA', brand:'BMW', name:'320d', odo:1000 },
                { id:'vB', brand:'기아', name:'K5',  odo:2000 }];
    allItems = [{ id:'i1', name:'엔진오일', vehicleId:'vA' },
                { id:'i2', name:'와이퍼',   vehicleId:'vA' },
                { id:'i3', name:'배터리',   vehicleId:'vB' }];
    allFuel  = [{ id:'f1', date:'2026-09-01', liters:40, cost:70000, odo:1000, full:true, vehicleId:'vA' }];
    allLogs  = [{ id:'L1', name:'엔진오일', date:'2026-09-01', odo:1000, cost:85000, vehicleId:'vA' }];
    selectVehicle('vA');
    const 셈 = deleteVehicleCounts('vA');
    a(셈.items === 2 && 셈.fuel === 1 && 셈.logs === 1, '지워질 기록 건수를 센다');

    const 원confirm2 = window.confirm;
    window.confirm = () => true;
    deleteActiveVehicle();
    window.confirm = 원confirm2;
    a(vehicles.length === 1 && vehicles[0].id === 'vB', '지운 차가 목록에서 빠진다');
    a(allItems.length === 1 && allItems[0].vehicleId === 'vB', '지운 차의 소모품만 사라진다');
    a(allFuel.length === 0 && allLogs.length === 0, '지운 차의 주유·이력도 사라진다');
    a(activeId === 'vB', '남은 차로 전환된다');

    window.confirm = () => true;
    deleteActiveVehicle();
    window.confirm = 원confirm2;
    a(vehicles.length === 0 && vehicle === null, '마지막 차를 지우면 빈 상태가 된다');
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=501`
예상: `ReferenceError: deleteVehicleCounts is not defined`

- [ ] **Step 3: 삭제 버튼을 추가한다**

`index.html:578` 의 저장 버튼

```html
        <button class="btn primary block" onclick="saveVehicle()">저장</button>
```

**바로 뒤**에 넣는다. 새 CSS 클래스를 만들지 않고, 전체 초기화 버튼(`index.html:601`)이
이미 쓰고 있는 인라인 방식을 그대로 따른다.

```html
        <button class="btn block" onclick="deleteActiveVehicle()" style="color:var(--danger); margin-top:8px;">이 차 삭제</button>
```

- [ ] **Step 4: 삭제를 만든다**

`saveVehicle()` 바로 뒤에 넣는다.

```js
function deleteVehicleCounts(id){
  return {
    items: allItems.filter(x => x.vehicleId === id).length,
    fuel:  allFuel.filter(x  => x.vehicleId === id).length,
    logs:  allLogs.filter(x  => x.vehicleId === id).length
  };
}
// 주인 없는 기록을 남기지 않는다. 안 지우면 쓰지도 않는 데이터가 쌓이고 백업만 커진다.
function deleteActiveVehicle(){
  if (!vehicle) return;
  const id = vehicle.id;
  const c = deleteVehicleCounts(id);
  const 이름 = vehicleTitle();
  if (!confirm(이름 + '을(를) 지울까요?\n소모품 ' + c.items + '건 · 주유 ' + c.fuel +
               '건 · 교체 이력 ' + c.logs + '건이 함께 지워집니다. 되돌릴 수 없어요.')) return;
  vehicles = vehicles.filter(v => v.id !== id);
  allItems = allItems.filter(x => x.vehicleId !== id);
  allFuel  = allFuel.filter(x  => x.vehicleId !== id);
  allLogs  = allLogs.filter(x  => x.vehicleId !== id);
  save(KEYS.items, allItems); save(KEYS.fuel, allFuel); save(KEYS.logs, allLogs);
  saveVehicles();
  selectVehicle(vehicles[0] ? vehicles[0].id : null);
  renderAll();
  switchScreen('home');
}
```

- [ ] **Step 5: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=502`
예상: `selfTest 통과`

- [ ] **Step 6: 커밋한다**

```bash
git add index.html && git commit -m "차량 삭제 — 그 차 기록까지 함께

지워질 건수를 보여주고 확인받은 뒤 그 차의 소모품·주유·교체 이력을
모두 지운다. 주인 없는 기록을 남기면 쓰지도 않는 데이터가 쌓이고
백업 파일만 커진다.

지운 뒤엔 남은 차로 전환하고, 남은 차가 없으면 빈 상태로 간다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 6: 백업이 여러 대를 담는다

**Files:**
- Modify: `index.html` — `vehicleIdOf()` 삭제, `backupPayload()`·`normalizeBackup()`·`importData()` 교체
- Modify: `index.html` — `renderBackupNotice()` 의 기록 수 계산

**Interfaces:**
- Consumes: Task 3의 전역들과 `newVehicleId()`
- Produces:
  - `backupPayload()` → `{app, version:3, exportedAt, vehicles, activeId, items, fuelLogs, maintLogs}`
  - `normalizeBackup(data)` → `{vehicles, activeId, items, fuelLogs, maintLogs}`. 2판·어제 3판·새 3판을 모두 같은 모양으로 편다.

- [ ] **Step 1: 실패하는 검사를 쓴다**

기존 백업 검사 블록(`const 원차백업 = vehicle, ...` 으로 시작하는 곳)을 **통째로 지우고** 그 자리에 넣는다.

```js
    // ---- 백업: 여러 대를 담고, 예전 파일도 읽힌다 ----
    const 원백업 = { vehicles, activeId, allItems, allFuel, allLogs, vehicle, items, fuelLogs, maintLogs };
    vehicles = [{ id:'vA', brand:'BMW', name:'320d', fuel:'diesel', odo:1000 },
                { id:'vB', brand:'기아', name:'K5', fuel:'gas', odo:2000 }];
    allItems = [{ id:'i1', name:'엔진오일', vehicleId:'vA' },
                { id:'i2', name:'와이퍼',   vehicleId:'vB' }];
    allFuel = []; allLogs = [];
    selectVehicle('vA');

    const 짐 = backupPayload();
    a(짐.version === 3, '백업은 3판 그대로');
    a(짐.vehicles.length === 2, '두 대를 모두 담는다');
    a(짐.activeId === 'vA', '어느 차를 보고 있었는지도 담는다');
    a(짐.items.length === 2, '전 차량 기록을 담는다');
    a(짐.items.every(x => x.vehicleId), '기록마다 차량 id가 붙어 있다');
    a(짐.vehicle === undefined, '호환용 vehicle 키는 더 쓰지 않는다');

    const 되읽기 = normalizeBackup(짐);
    a(되읽기.vehicles.length === 2 && 되읽기.activeId === 'vA', '담은 그대로 되읽힌다');

    // 2판 백업 — 차량 한 대, vehicleId 없음
    const 이판 = { version:2, vehicle:{ brand:'기아', name:'K5', odo:100 },
                   items:[{ id:'m9', name:'와이퍼' }], fuelLogs:[], maintLogs:[] };
    const 편2 = normalizeBackup(이판);
    a(편2.vehicles.length === 1 && !!편2.vehicles[0].id, '2판을 읽으면 차량에 id를 만들어 붙인다');
    a(편2.items[0].vehicleId === 편2.vehicles[0].id, '2판 기록에도 차량 id를 채운다');
    a(편2.activeId === 편2.vehicles[0].id, '2판은 그 한 대가 활성 차량이 된다');

    // 어제 형식(3판이지만 vehicles가 한 대 + vehicle 키가 있던 파일)
    const 어제 = { version:3, vehicles:[{ id:'vZ', brand:'BMW', name:'i4', odo:5 }],
                   vehicle:{ id:'vZ', brand:'BMW', name:'i4', odo:5 },
                   items:[{ id:'m1', name:'와이퍼', vehicleId:'vZ' }], fuelLogs:[], maintLogs:[] };
    const 편3 = normalizeBackup(어제);
    a(편3.vehicles.length === 1 && 편3.activeId === 'vZ', '어제 형식도 그대로 읽힌다');

    vehicles = 원백업.vehicles; activeId = 원백업.activeId;
    allItems = 원백업.allItems; allFuel = 원백업.allFuel; allLogs = 원백업.allLogs;
    vehicle = 원백업.vehicle; items = 원백업.items;
    fuelLogs = 원백업.fuelLogs; maintLogs = 원백업.maintLogs;
```

- [ ] **Step 2: 실패를 확인한다**

`http://localhost:8765/index.html?test&v=601`
예상: `selfTest 실패: 두 대를 모두 담는다`

- [ ] **Step 3: 내보내기를 고친다**

`vehicleIdOf()` 는 더 쓰이지 않으므로 **함수째 지운다**.
`backupPayload()`, `normalizeBackup()` 을 통째로 바꾼다.

```js
function backupPayload(){
  return {
    app:'카스토리', version:3, exportedAt:new Date().toISOString(),
    vehicles, activeId,
    items: allItems, fuelLogs: allFuel, maintLogs: allLogs
  };
}
// 2판(차량 한 대, id 없음)과 3판을 모두 같은 모양으로 펴서 돌려준다
function normalizeBackup(data){
  const 목록 = Array.isArray(data.vehicles) && data.vehicles.length
    ? data.vehicles.map(v => ({...v, id: v.id || newVehicleId()}))
    : (data.vehicle ? [{...data.vehicle, id: data.vehicle.id || newVehicleId()}] : []);
  const 기본 = 목록[0] ? 목록[0].id : null;
  const 활성 = 목록.some(v => v.id === data.activeId) ? data.activeId : 기본;
  const 붙이기 = list => (Array.isArray(list) ? list : []).map(x => ({...x, vehicleId: x.vehicleId || 기본}));
  return {
    vehicles: 목록, activeId: 활성,
    items: 붙이기(data.items), fuelLogs: 붙이기(data.fuelLogs), maintLogs: 붙이기(data.maintLogs)
  };
}
```

- [ ] **Step 4: 불러오기를 고친다**

`importData()` 안에서 Task 3에 임시로 둔 `saveVehicles(); saveItems(); saveFuel(); saveLogs();` 가 있는
블록을 이렇게 바꾼다.

```js
    const 편 = normalizeBackup(data);
    vehicles = 편.vehicles;
    allItems = 편.items;
    allFuel  = 편.fuelLogs;
    allLogs  = 편.maintLogs;
    save(KEYS.vehicles, vehicles);
    save(KEYS.items, allItems); save(KEYS.fuel, allFuel); save(KEYS.logs, allLogs);
    selectVehicle(편.activeId);
    markBackedUp();   // 방금 불러온 파일이 곧 최신 백업이다
    renderAll();
```

- [ ] **Step 5: 백업 안내가 전 차량을 세게 한다**

`renderBackupNotice()` 의 기록 수 계산을 바꾼다.

```js
  const 기록수 = allItems.length + allFuel.length + allLogs.length;
```

- [ ] **Step 6: 통과를 확인한다**

`http://localhost:8765/index.html?test&v=602`
예상: `selfTest 통과`

- [ ] **Step 7: 눈으로 확인한다**

`http://localhost:8765/index.html?v=603` 에서 차 두 대를 등록하고 각각 기록을 넣은 뒤,
내보내기 → 전체 초기화 → 불러오기. 두 대와 각자의 기록이 그대로 돌아와야 한다.

- [ ] **Step 8: 커밋한다**

```bash
git add index.html && git commit -m "백업이 여러 대를 담는다

형식은 3판 그대로 두고 vehicles에 전부 담는다. 미리 만들어둔 자리가
이제 실제로 쓰인다. 어느 차를 보고 있었는지(activeId)도 함께 적는다.

호환용 vehicle 키는 더 쓰지 않는다. 다만 읽기는 계속 지원해서
2판 백업과 어제 형식으로 내보낸 파일도 그대로 열린다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 7: 문서를 맞춘다

**Files:**
- Modify: `README.md`
- Modify: `차계부-앱-기획서.md` (gitignore 대상이라 커밋에는 안 들어간다)

**Interfaces:**
- Consumes: Task 1~6의 결과
- Produces: 없음

- [ ] **Step 1: README를 고친다**

- `**차량**` 항목에 한 줄 추가: `- 차 여러 대를 등록하고, 홈 차량 카드를 눌러 전환합니다. 차마다 소모품·주유·교체 이력이 따로 쌓여요`
- "데이터 저장 위치" 절의 백업 설명에 한 줄 추가: `백업 파일에는 등록한 차가 전부 담깁니다.`
- `## 다음 후보 기능 (v2)` 에서 `- 여러 대 차량 등록 (백업 형식에 자리는 준비됨)` 줄을 지운다

- [ ] **Step 2: 기획서를 고친다**

- `### 구현 상태` 의 **차량** 표에 다중 차량 행을 넣는다
- `**남은 알려진 한계**` 에서 `- 차량 1대만 등록 가능 ...` 줄을 지우고
  `- 차량 간 기록 이동은 없다. 잘못된 차에 넣었으면 지우고 다시 넣어야 한다` 로 바꾼다
- `## 8. 다음 단계 (To-Do)` 의 `- [ ] 여러 대 차량 등록 ...` 을 `- [x] 여러 대 차량 등록 (2026-09-16)` 으로 바꾼다

- [ ] **Step 3: 마지막으로 전체 검사를 돌린다**

새 탭에서 `http://localhost:8765/index.html?test` 를 연다.
콘솔에 `selfTest 통과` 한 줄만 있고 오류·경고가 없어야 한다.

- [ ] **Step 4: 커밋하고 서버를 내린다**

```bash
git add README.md && git commit -m "README를 다중 차량까지 반영

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

PowerShell에서:

```
Get-CimInstance Win32_Process -Filter "Name='python.exe'" | Where-Object { $_.CommandLine -like '*http.server*8765*' } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```
