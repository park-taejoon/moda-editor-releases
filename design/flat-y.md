# Flat-Y 재설계 스케치

> 상태: 설계 초안 (구현 착수 전)
> 대상 문제: `bindYjs`의 deep-mirror 모델에서 컨테이너 경계 구조 연산이
> 동시 원격 편집을 고아화시키는 근본 한계 (WORK.md 보류 항목)

## 1. 현재 모델과 손실 지점

`collab-yjs`는 doc과 **동형의 중첩 Y 트리**를 미러로 둔다:

```
root = ydoc.getMap("moda")  →  { type:"doc", children: Y.Array<YNode> }
YNode = Y.Map { type, attrs:Y.Map, marks?:Y.Map,
                children?:Y.Array<YNode>, text?:Y.Text }
```

`Point{path,offset}`의 path가 그대로 Y 트리 좌표가 되는 게 이 모델의
장점인데, **Y 아이템은 한 번 부착되면 다른 부모로 재부착할 수 없다**
(CRDT identity는 탄생 시 위치에 묶인다). 그래서 자식을 다른 노드로
"옮기는" 연산은 실제로 `삭제 + JSON 재구축`이다 — 삭제된 서브트리
아이템에 동시 원격 편집이 anchor되면 그 편집은 tombstone에 붙어
고아가 되고, GC 이후 최종 문서는 수렴하지만 **편집 내용 자체는
유실**된다.

손실이 일어나는 지점 (`steps.ts`):

| 연산                     | 재구축 대상                   | 현재 완화                    | 잔여 손실                           |
| ------------------------ | ----------------------------- | ---------------------------- | ----------------------------------- |
| `split` (컨테이너)       | head/tail 중 작은 쪽 children | 큰 쪽 identity 보존          | 작은 쪽 서브트리 안 동시 편집       |
| `join` (컨테이너/텍스트) | 작은 쪽 통째 삭제             | 큰 쪽 살림 + `adoptPrevMeta` | 지워진 쪽 안 동시 편집              |
| `insert` 마크 분할       | 새 형제 세그먼트              | 원본(head) identity 보존     | 꼬리 세그먼트 안 동시 편집          |
| `replaceDoc`             | 전체 children                 | —                            | 통째 tombstone (정의상 파괴라 수용) |

대표 시나리오: 피어 A가 리스트 아이템 끝에서 Enter(split)하는 동안
피어 B가 그 아이템의 다음 블록을 편집 → B의 편집이 재구축된 서브트리에
anchor돼 소실.

## 2. 목표와 비목표

**목표**

- 컨테이너 경계 구조 연산(split/join/indent/unwrap/DnD)에서 콘텐츠
  아이템의 재구축을 없앤다 — 동시 원격 편집 손실 0
- `Point{path,offset}` 로컬 모델·Step 계약은 그대로 유지
- 피어 간 최종 doc JSON 수렴 보장 (결정적 복구 규칙)

**비목표**

- 같은 아이템의 동시 삭제 병합 — 이미 Yjs가 제공
- 로컬 undo/히스토리 정책, awareness API 변경
- 텍스트 내부 문자 수준 병합 — 현행 Y.Text 그대로 유지
- `replaceDoc`의 파괴적 의미 — 문서 교체는 정의상 전체 tombstone

## 3. 후보 모델 비교

### A. 블록 수준 flat + 마커 (권장)

컨테이너 중첩만 평면화하고, **인라인 서브트리는 블록 아이템 안에 현행
구조 그대로** 둔다:

```
root = ydoc.getMap("moda")  →  { type:"doc", items: Y.Array<FlatItem> }

FlatItem = Y.Map:
  { k:"open", type, attrs:Y.Map, marks?:Y.Map }     — 컨테이너 열기
  { k:"close" }                                     — 컨테이너 닫기
  { k:"leaf", type, attrs:Y.Map, marks?:Y.Map,
    children?:Y.Array<InlineNode> }                 — 인라인 포함 블록
                                                    (현행 YNode 재사용)
  { k:"leaf", type, attrs, marks? }                 — 블록 atom (hr 등)
```

예: `doc( blockquote( p("a"), p("b") ), ul( li( p("c") ) ) )`

```
[open blockquote][leaf p("a")][leaf p("b")][close]
 [open ul][open li][leaf p("c")][close][close]
```

**핵심 성질**: 콘텐츠를 담는 아이템(leaf)은 항상 단일 루트 배열에
anchor된다. 컨테이너 구조는 앞뒤 마커 쌍으로만 인코딩되므로 **구조
변경은 마커만 건드리고 콘텐츠 아이템은 절대 이동·재구축하지 않는다**.

연산별 비용:

| 연산                             | flat 번역                                           | 재구축                                                                |
| -------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------- |
| 문단 중간 Enter                  | leaf의 inline 배열 분할 (현행과 동일 — 꼬리 재구축) | 현행과 동일                                                           |
| 컨테이너 경계 Enter (li 시작 등) | `close`+`open` 마커 쌍 삽입                         | **없음**                                                              |
| blockquote 해제/감싸기           | 마커 쌍 삭제/삽입                                   | **없음**                                                              |
| 리스트 indent/outdent            | 해당 li의 마커 쌍 이동(삭제+삽입)                   | **없음**                                                              |
| 블록 join (Backspace 병합)       | 두 leaf의 inline 병합, 하나 삭제                    | 지워진 leaf만 (현행과 동일, 작음)                                     |
| 컨테이너 join                    | 마커 쌍 삭제 (+ 내부 마커 조정)                     | **없음**                                                              |
| 블록 DnD                         | leaf(와 둘러싼 마커 구간) 삭제+재삽입               | 이동 블록 자체만 (원격의 "그 블록 안" 동시 편집은 여전히 위험 — 후술) |
| 마크 분할                        | inline 형제 분할 (현행과 동일)                      | 현행과 동일                                                           |

### B. 완전 flat (인라인까지 평면)

모든 텍스트 세그먼트·atom도 루트 배열의 아이템:

```
[open p][leaf t1][leaf img][close p][open li][open p][leaf t2][close p][close li]
```

장점: 마크 경계 분할도 leaf 분할이 되어 이론상 더 깨끗. 단점: 모든
번역 코드를 새로 써야 하고, Y.Text 이벤트→스텝 역직역(가장 복잡한
경로)이 전면 재작성된다. 텍스트 아이템 identity(글자 추가 시 같은
Y.Text 재사용)는 A에서도 유지되므로, 실익 대비 비용이 크다.

### C. deep 유지 + 사후 복구

tombstoned 서브트리의 원격 편집을 다시 읽어 재적용. Yjs는 GC 전에도
삭제 아이템을 정상 순회 못 하고, "어느 편집이 고아화됐는지" 판별이
본질적으로 불안정 — 수렴 보장이 어렵다. 기각.

### D. y-prosemirror 도입

y-prosemirror도 XmlElement 중첩 트리라 같은 재부착 불가 제약이 있고,
우리 doc 모델은 PM이 아니라 변환 계층을 새로 둬야 한다. 기각.

> **판정 변경 (단계 1 구현 시)**: 초기 권장은 A였으나 구현 분석에서
> B(완전 flat)를 채택했다. A의 잔여 약점 — "문단 중간 Enter가 leaf의
> inline 배열을 분할하며 꼬리 인라인을 재구축" — 이 컨테이너 경계
> 외에 남는 유일한 콘텐츠 재구축 경로인데, 완전 flat이면 꼬리도
> `[close][open]` 마커 쌍만으로 해결된다. B의 반대 근거였던 "텍스트
> identity"는 leaf 안에 `Y.Text` 슬롯을 두는 것으로 동일하게 유지되고,
> "인라인 번역 재작성" 비용은 어차피 단계 2–3에서 인라인 스텝의 아이템
> 좌표 해석이 필요해 실제 추가분이 작다고 판단했다.

## 4. 상세 설계 (A안 — 실제 구현은 B, §단계 1 결과 참고)

### 4.1 flat index ↔ Point 해석

로컬 스텝의 `Point{path,offset}`를 flat 배열 좌표로 바꿀 해석기가
필요하다:

- `pathToItem(path)` — doc 트리 순회 순서(전위, 마커 포함)와 items
  인덱스가 1:1 대응한다. 매 스텝마다 O(문서) 선형 탐색; 스텝 배치당 한
  번의 walk로 인덱스 맵을 캐시하면 충분하다 (문서 크기 ~수백 블록).
- `itemIndexToPath(i)` — 역방향도 같은 walk로 산출.
- 구현체: `items`를 한 번 스캔해 `{ markerStack, path→itemIndex }`
  맵을 만드는 `flatIndex(root)` 헬퍼. 스텝 번역 배치 내에서 갱신된
  items 기준으로 재계산.

### 4.2 로컬 Step → flat 번역

`applyStepToY`의 컨테이너 경로만 바뀐다 — 인라인(텍스트/마크/atom)
번역은 현행 그대로:

- `insert` 블록: 목표 컨테이너의 items 구간을 찾아 `offset` 위치에
  leaf/marker 아이템 삽입
- `split`:
  - leaf 내부 분할 → inline children 분할 (현행 로직)
  - 컨테이너 분할 → 분할 지점에 `[close][open(같은type/attrs/marks)]`
    마커 쌍 삽입. **children 이동 없음**
- `join`:
  - leaf+leaf → 인라인 병합 후 한쪽 삭제 (현행 로직)
  - 컨테이너 join → 두 컨테이너 사이의 `close`+`open` 마커 쌍 삭제
- `setAttr`/`addMark`/`removeMark` → 대상 아이템의 맵 갱신 (동일)
- `replaceDoc` → items 전체 삭제 후 재구축 (동일)

### 4.3 Y 이벤트 → Step 역직역

`observeDeep` 이벤트 대상이 4종으로 단순해진다:

1. **루트 items 배열 델타** — insert된 아이템을 JSON화해 insert 스텝,
   delete는 현재 doc에서 대응 노드를 찾아 remove 스텝. 마커 삽입은
   "컨테이너 open/close"로 해석 — 재구성된 doc diff로 스텝을 합성하는
   방식이 더 안전할 수 있다 (후술).
2. **leaf의 children 배열 델타** — 현행 childrenDeltaToSteps 재사용
3. **leaf의 text 슬롯 델타** — 현행 textDeltaToSteps 재사용
4. **attrs/marks 맵 keysChanged** — 현행 그대로

마커 이벤트 처리가 새 부분이다. 두 접근:

- **(a) 스텝 합성**: 마커 삽입/삭제를 split/join 스텝으로 직역 —
  의미 보존이 까다롭다 (마커 쌍이 다른 트랜잭션에서 따로 올 수 있음)
- **(b) 구조 diff (권장)**: 이벤트 배치 처리 후 `flatToDoc(root)`를
  재구성해 현재 로컬 doc과 `diffDoc(oldDoc, newDoc)` → 스텝열 합성.
  텍스트 델타는 이미 정밀하게 역직역되므로 diff는 **구조 변경만**
  담당 — 범위를 마커 이벤트가 있을 때로 한정한다. diffDoc은
  노드 identity 비교 대신 위치·내용 비교의 최소 편집열을 낸다.

### 4.4 재구성과 복구 규칙 (수렴의 핵심)

`flatToDoc` = items의 **순수 결정적 스택 파서** — 동시 편집으로 마커가
어긋나도 모든 피어가 같은 함수로 같은 doc을 얻는다:

```
stack = [doc]
for item in items:
  open  → stack.push(새 노드)
  leaf  → stack.top.content.push(노드)
  close → stack.pop()  (doc까지 닫으면 나머지는 무시)
끝까지 안 닫힌 open → 자동 close
```

**복구 규칙 (스키마 무관, 콘텐츠 보존 우선)**:

1. 고아 `close` (매칭 open 없음) → 무시
2. 안 닫힌 `open` → 문서 끝에서 자동 close
3. leaf가 허용 안 되는 위치 (예: 컨테이너를 요구하는 부모 안에
   텍스트) → **버리지 않고** 가장 가까운 유효 부모에 래핑
   (`defaultBlockTypeOf` 등 스키마 기본값으로)
4. open의 type이 스키마에 없으면 paragraph로 강등

결정적이면 어떤 규칙이든 수렴한다 — 규칙 선택은 "손실 최소화"의
품질 문제지 정합성 문제가 아니다. **이것이 flat의 근본적 강점**:
구조가 틀어져도 콘텐츠 아이템은 모두 살아 있어 복구가 가능하다.

### 4.5 캐럿 (carets.ts)

- `pointToRelPos`: 텍스트 오프셋 → leaf의 Y.Text 상대위치 (동일).
  컨테이너 경계 오프셋 → **루트 items 배열의 인덱스** 상대위치로
  변경 (현재는 각 노드의 children 배열).
- `relPosToPoint`: 절대 인덱스 → flatIndex walk로 Point 복원.
  삭제된 leaf의 위치는 "남은 인접 아이템" 기준으로 점근 — Y
  relative position이 자체 처리.

### 4.6 undo (undo.ts)

`UndoManager`가 추적하는 scope는 Y 아이템 — 마커 아이템도 로컬 생성이면
추적된다. 구조 undo가 "leaf 복원 + 마커 복원"을 같이 되돌리므로 현행
메커니즘 재사용 가능. 검증 필요: 마커만 지워지는 undo 경계가
비정형 시퀀스를 만들지 않는지 (복구 규칙이 흡수).

### 4.7 마이그레이션

deep 트리와 flat 배열은 형태가 달라 **동일 rootName 공존 불가**:

- `rootName` 버저닝 (`"moda"` → `"moda2"`) — 신버전 피어끼리만 통신,
  구버전 문서는 첫 접속 피어가 flat으로 일회성 업로드
- 또는 `bindYjsFlat` 별도 진입점으로 공존 제공 → 점진 전환 후 deep 제거

## 5. 구현 단계

| 단계 | 내용                                                                | 검증                                                                    |
| ---- | ------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 0 ✅ | **차등 fuzz 하네스** (`src/fuzz.test.ts`) — 시드 고정 2피어 랜덤 op | 완료: 수렴 24시드 통과, 보존 24시드+고정재현 `it.fails`로 유실 문서화   |
| 1 ✅ | `flat.ts` — docToFlat/flatToDoc + 복구 파서 + flatIndex             | 완료: 왕복·비정형 시퀀스·인덱스 유닛 11건 (`src/flat.test.ts`)          |
| 2 ✅ | `flat-steps.ts` — 스텝 번역 (로컬→Y)                                | 완료: core `applyStep` 차등 오라클 유닛 31건 (`src/flat-steps.test.ts`) |
| 3 ✅ | `flat-diff.ts` 구조 diff + `flat-bind.ts` 바인딩                    | 완료: 역직역 유닛 27건 + 2피어 수렴·보존 통합 7건                       |
| 4 ✅ | carets/undo 적응 — 루트 형태 자동 감지                              | 완료: 왕복·추적·awareness·undo 스코프 6건 (`src/flat-carets.test.ts`)   |
| 5 ✅ | rootName 버저닝 + 마이그레이션 경로 — `flat-migrate.ts`             | 완료: 변환/멱등/중첩/migrateFrom 7건 (`src/flat-migrate.test.ts`)       |
| 6 ✅ | e2e collab.spec deep+flat 파라미터화 + collab.md/migration.md       | 완료: `?me-collab=flat` 데모 스위치, sync/undo/캐럿 e2e ×2 모드         |

추정 규모: 신규 ~800–1,200줄 (convert+steps+events+repair), 기존
`convert.ts`의 inline 슬롯 헬퍼(yText/yMarksMap 등)는 재사용.
기존 코드 ~2,000줄 중 절반은 inline 레이어라 그대로 쓴다.

## 6. 리스크와 완화

- **비정형 시퀀스**: 동시 마커 편집이 만드는 케이스 조합이 무한 —
  fuzz(단계 0)가 필수 선행물. 복구가 순수 함수라 수렴 자체는 보장
- **블록 DnD의 잔여 손실**: 이동 블록 자체는 여전히 delete+insert —
  "그 블록 안 동시 편집"은 고아화 가능. 완화: DnD를 `cut+paste`가
  아니라 재배치 시 임시 "hold" 구역에 두는 방식이나, 드래그 동안
  로컬 편집 잠금 정책으로 실손실 최소화
- **와이어 크기**: 마커가 노드 수를 ~1.5–2배로. 블록당 2바이트급
  오버헤드 — 무시 가능한 수준
- **성능**: `flatIndex` walk O(문서) per 이벤트 배치 — 스텝당이
  아니라 배치당이면 현행 yNodeAt 순회와 동급. 대용량은 향후
  인덱스 캐시로 최적화 여지
- **마커 attrs의 동시 편집**: 마커 이동은 delete+insert라 그 마커
  자체 attrs의 동시 변경은 고아화 — 마커는 보통 구조 전용이고
  taskItem.checked 같은 콘텐츠성 attrs는 leaf에 두면 해소

## 7. 결론

손실의 근본 원인은 "Y 아이템 재부착 불가"와 "중첩 트리"의 결합이다.
컨테이너를 마커 쌍으로 평면화하면 구조 변경이 콘텐츠를 건드리지
않아 재구축 자체가 사라지고, 어긋난 시퀀스도 결정적 복구 파서로
모든 피어가 같은 doc에 수렴한다 — **잃을 게 없는 구조 변경**.

다음 행동: 단계 0의 fuzz 하네스부터 만들어 현행 모델의 실손실
시나리오를 재현·고정하고, 그 스위트가 flat에서 그대로 통과하는지로
설계를 검증한다.

### 단계 0 결과 (완료)

`src/fuzz.test.ts` — mulberry32 시드 RNG + 2피어 시뮬레이터.
op 풀: insertText/split(depth 1~3)/splitItem/liftItem/sinkItem/
insertNode/mark/setAttr/replaceNode/delete·경계삭제.

- **수렴 스위트(24시드 × 25라운드) 전부 통과** — 고아화된 편집은
  양쪽 doc에서 결국 사라져 최종 JSON은 항상 수렴한다 (이론 확인)
- **보존 스위트(24시드, delete·replaceNode 제외) 전부 유실** —
  순수 구조 연산만으로 매 시드 다수 삽입 토큰 소실 → `it.fails`로
  고정 (flat 전환 시 "예상된 실패가 통과" 신호로 자동 뒤집힘)
- **고정 재현 1건**: A의 blockquote split + B의 꼬리 문단 동시 삽입
  → 재구축된 서브트리 안 삽입이 양쪽에서 유실 (`it.fails` 고정)

### 단계 1 결과 (완료)

`src/flat.ts` — **완전 flat** 인코딩(설계의 하이브리드에서 변경 —
문단 중간 split의 꼬리 인라인 재구축도 막기 위해 인라인까지 마커/leaf
아이템으로 평면화). 구성:

- `docToFlat` — doc → `[open …][leaf …][close]` 아이템열
- `flatToDoc` — 순수 결정적 스택 파서 + 복구 규칙 5개 (고아 close
  무시 / 미종료 open 자동 close / 고아 인라인 leaf 기본 블록 래핑 /
  인라인 전용 부모 속 블록 투명 / 미등록 타입 마커 투명)
- `flatIndex` — 아이템 → doc path·부모·형제 매핑 (단계 3의 이벤트
  역직역이 사용)

`src/flat.test.ts` 11건: 왕복(문단·중첩·혼합+마크+attrs·빈 컨테이너·
인라인 atom), 비정형 복구 5종, flatIndex path 해석 + 고아 close.
구현 중 잡은 세부: text는 스키마 미등록 특례 타입 — 인라인 취급을
명시해야 "block" 그룹에 잘못 수용되지 않는다.

### 단계 2 결과 (완료)

`src/flat-steps.ts` — `applyStepToFlat(root, step)`. deep 판의
"부모 children 배열 조회"가 전부 사라지고 루트 배열의 인덱스
연산으로 환원된다:

- **node split** — 자식 경계에 `[close][open(메타 복제)]` 마커 쌍
  삽입. 꼬리 서브트리 재구축이 완전히 사라졌다
- **컨테이너 join** — 인접 `[close prev][open next]` 2아이템 삭제.
  어느 쪽을 재구축할지의 딜레마 자체가 없다 (불변식 테스트로
  leaf 아이템 보존을 단언)
- **insert/remove/marks/split-text** — deep 판과 같은 의미론,
  "형제 삽입"이 루트 배열 인접 삽입으로 단순화

검증: 차등 오라클 — 같은 doc+Step을 core `applyStep`과 flat 미러
경유에 적용해 `docJsonEqual` 비교, 31건. 구현 중 잡은 함정 2개:

1. **미통합 Y.Map의 get/entries는 빈 값을 돌려준다** — 복제된
   marks를 빈 맵으로 덮어쓰는 버그의 원인. 새 아이템의 상태는
   읽지 말고 소스 기준으로 판정한다
2. 마크 부분 커버의 tail은 **갱신 전** 마크로 만들어야 한다 —
   갱신된 원본을 클론하면 마크가 꼬리로 새어나간다

### 단계 3 결과 (완료)

Y→로컬 방향은 **구조 diff**를 채택했다 (per-delta 분류 대신):
동시 편집이 만드는 items 델타 형태는 무한 조합이라 분류기가
깨지기 쉬워, 로컬 doc(before)과 flatToDoc(after)의 결정론적 트리
diff로 before→after를 만드는 Step열을 산출한다.

- `src/flat-diff.ts` — `flatDiffSteps(before, root)`. 방출 스텝은
  {insert, remove, setAttr, addMark, removeMark, replaceDoc}의
  부분집합 — split/join은 remove+insert로 환원되므로 임의 변환을
  커버한다. 텍스트는 접두/접미 문자 diff로 최소화, 정렬은 탐욕
  최근접 매칭(동형 노드는 제자리 diff)
- `src/flat-bind.ts` — `bindYjsFlat(editor, ydoc, {rootName})`.
  observeDeep은 감지용으로만 쓰고 diff가 역직역을 담당.
  기본 rootName "moda-flat"으로 deep과 공존 (단계 5 버저닝 전제)

검증: 역직역 유닛 27건(모든 op 분류 + 배열 직접 수술로 만든
비정형 변경) + 2피어 통합 7건. **핵심 증명: "split vs 꼬리 문단
동시 편집"에서 삽입 문자가 양쪽에 살아남는다** — fuzz의 `it.fails`
로 고정된 deep 유실 시나리오가 flat에서 보존됨을 확인.

### 단계 4 결과 (완료)

carets/undo는 **전용 판을 만들지 않고** `carets.ts`의 좌표 함수에
루트 형태 자동 감지를 넣어 해결했다 — `flatItems(root)` 유무로
flat/deep을 판별하므로 `createCollabCarets`/`createCollabUndo`가
`rootName: "moda-flat"`만 넘기면 그대로 동작한다:

- `pointToRelPos` — flat이면 텍스트는 leaf의 Y.Text 위치로,
  컨테이너/doc 경계는 루트 items 배열 인덱스로 인코딩
- `relPosToPoint` — flat이면 Y.Text→leaf 맵의 flatIndex path,
  Y.Array(items)→`flatIndexToPoint`로 경계 해석.
  close 마커 위치는 "부모 다음 경계"가 아니라 **닫힌 노드의
  내용 끝**(`{path: 노드, offset: 자식 수}`)으로 해석해야 문단 끝
  커서가 다음 문단 앞으로 튀지 않는다
- undo는 무수정 — `Y.UndoManager`의 스코프는 YType이라 flat 루트
  맵을 그대로 받고, 셀렉션 저장/복원은 위 자동 감지 경로를 탄다

검증(`src/flat-carets.test.ts` 6건): 좌표 왕복, 삽입 추적,
**다른 문단 split 후 커서 leaf가 아이템 재배치 없이 새 path를
따라감**(구조 변경 생존의 실증), leaf 문자 tombstone 수렴,
awareness 발행/렌더, undo 로컬 스코프(원격 편집 유지).

알려진 한계: 커서가 붙은 leaf 자체를 split하면 Y.Text가 잘려
커서는 원본 끝으로 수렴한다 — 문자를 새 아이템으로 옮기는 건
어떤 모델도 불가능한 Yjs의 근본 한계(deep과 동일).

### 단계 5 결과 (완료)

`src/flat-migrate.ts` — deep → flat 업그레이드 경로:

- `FLAT_VERSION = 1` — flat 루트의 `"v"` 슬롯에 와이어 포맷 버전을
  기록 (업로드·마이그레이션 양쪽). 구형 피어는 슬롯을 무시해
  하위호환 유지.
- `migrateDeepToFlat(ydoc, {from="moda", to="moda-flat"})` —
  `yToDoc(deepRoot)` → `docToFlat` 스냅샷 일괄 변환. 대상이 차
  있거나 deep이 비어 있으면 false(멱등, 덮어쓰기 없음). deep 루트는
  구형 피어 공존을 위해 보존한다 — 삭제는 배포 정책.
- `bindYjsFlat({migrateFrom: "moda"})` — 바인드 시 flat이 비고 지정
  deep 루트에 문서가 있으면 자동 변환 후 평소 pull 경로로 진입.
- 루트명: deep `"moda"` / flat `"moda-flat"` — 같은 Doc에 공존,
  점진 cutover 가능.

검증(`src/flat-migrate.test.ts` 7건): 스냅샷 동치성, 중첩 구조
보존, 멱등, 빈 deep 스킵, migrateFrom 자동 pull·폴백 업로드·
마이그레이션 후 정상 편집.

### 단계 6 결과 (완료)

- 데모 배선: `apps/dev-shared`의 `bindCollabDemo`가
  `?me-collab=flat`을 읽어 `bindYjsFlat(rootName: "moda-flat")`으로
  바인딩하고 캐럿/undo에 같은 rootName을 넘긴다 — 기본은 여전히
  deep이다.
- e2e: `collab.spec.ts`를 deep/flat 두 모드로 파라미터화 — 동기화+
  로컬 undo, 원격 캐럿/선택 2개 시나리오를 두 인코딩에 동일하게
  검증 (coreOnly = react+vanilla).
- 문서: `docs/guide/features/collab.md`에 flat-Y 섹션(동기·공존·
  마이그레이션·한계), `docs/guide/migration.md` 표에 API 추가.

## 8. 알려진 한계 — 해결됨 (감사 후 정정)

감사에서 확인된 세 구조적 리스크는 전부 수정됐다:

1. **번역 불가 스텝의 묵묵한 발산 → 해결** — `applyStepToFlat`이
   boolean을 반환하고, `bindYjsFlat`은 스텝 실패 시 미러↔사후 doc의
   동치를 검사해 실제 발산 때만 replaceDoc으로 미러를 재구축한다
   (양쪽 no-op이면 재구축하지 않는다 — `flat-resync.test.ts`가
   두 경로를 검증). 주의: `transaction` 이벤트는 state 커밋 전에
   발화하므로 사후 doc은 `tr.steps`를 직접 적용해 구한다.
2. **flatIndex ↔ flatToDoc의 비정형 시퀀스 path 발산 → 해결** —
   `flatIndex`가 스키마 인지형으로 파서와 동일한 스택 의미론을
   공유한다(규칙 3 가상 래퍼·규칙 5 투명). 스키마는 `setFlatSchema`
   WeakMap 레지스트리로 루트에 등록되고, `bindYjsFlat`이 바인드
   시점에 등록하므로 carets 등 스키마를 모르는 호출자도 파서와
   같은 좌표계를 얻는다 (`flat.test.ts`의 패리티 스위트).
3. **flatChildBoundaryIdx의 배열 끝 폴백 → 해결** — 부모 부재·
   leaf 부모·음수 자식 인덱스는 -1로 실패를 표면화하고 모든
   호출자가 false로 전파한다 (1번의 재동기화로 수렴). doc 루트의
   끝 경계(index.length)는 core splice의 append와 같은 의미라
   유효 경계로 유지한다.

남은 인코딩 제약: doc 루트 자체의 attrs는 flat에 표현되지 않는다
(루트 맵은 type·v·items만 둔다) — core에서도 실사용이 없고 deep
모델도 같은 제약이므로 보류.
