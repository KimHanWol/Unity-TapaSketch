# 타파아트 구현 가이드

> 기획은 [GDD.md](GDD.md). 이 문서는 구현 방향, 구조, 데이터 형식, 알고리즘, 작업 순서를 정의한다.

---

## 1. 기술 스택

| 영역 | 선택 | 비고 |
|---|---|---|
| 엔진 | Unity 6 LTS | 2D 템플릿(URP 2D 불필요, Built-in 또는 URP 중 택1) |
| UI | uGUI + TextMeshPro | 모바일 안정성 우선 |
| 애니메이션 | DOTween (무료판) | 클리어 연출, UI 트윈 |
| JSON | `com.unity.nuget.newtonsoft-json` | 레벨/세이브 |
| 다국어 | Unity Localization | MVP는 한국어만, 구조만 미리 |
| 광고 | Google Mobile Ads Unity 플러그인 | MVP 이후 |
| 결제 | Unity IAP | MVP 이후 |
| 테스트 | Unity Test Framework (EditMode) | Core 로직 필수 |

화면: 세로 고정, 기준 해상도 1080x1920, Canvas Scaler `Scale With Screen Size` (Match 0.5), Safe Area 대응.

---

## 2. 폴더 구조

```
Assets/
  _Project/
    Scripts/
      Core/            ← 순수 C#. UnityEngine 참조 금지. (asmdef: TapaArt.Core, noEngineReferences: true)
        Model/         Cell, Board, Clue, LevelData
        Rules/         ClueCalculator, RuleValidator
        Solver/        UniqueSolver, HumanSolver(힌트/난이도)
        Generator/     ImageToSolution, ClueReducer, DifficultyGrader
      Game/            MonoBehaviour: BoardView, CellView, InputController, UndoStack, ClearSequence
      UI/              TitleScreen, MapScreen, GameHud, SettingsPanel, Popups
      Meta/            SaveSystem, ProgressService, SettingsService, AudioService, HintService
      Editor/          LevelEditorWindow, LevelBatchValidator  (asmdef: Editor 전용)
    Tests/EditMode/    Core 단위 테스트 (asmdef: TapaArt.Tests)
    Levels/            chapter01.json, chapter02.json ...  (Resources 또는 Addressables)
    Art/  Audio/  Fonts/  Prefabs/  Scenes/
```

**핵심 원칙: 퍼즐 규칙·솔버·생성기는 전부 `Core`에 두고 Unity 없이 테스트 가능해야 한다.** 에디터 도구와 게임은 Core를 호출만 한다.

---

## 3. 씬 구성

| 씬 | 내용 |
|---|---|
| `Boot` | 서비스 초기화(세이브 로드, 설정 적용, 레벨 데이터 로드) 후 Main으로 |
| `Main` | 타이틀 + 월드맵 (패널 전환). 설정 패널 공용 |
| `Game` | 퍼즐 플레이 + 클리어 연출 |

씬 간 데이터 전달은 정적 `GameSession` (선택한 레벨 ID 등) 또는 서비스 싱글톤으로.

---

## 4. 데이터 형식

### 4.1 좌표계
- `(x, y)`, 원점은 **왼쪽 위**, x는 오른쪽, y는 아래로 증가.
- 1차원 인덱스: `i = y * width + x`.

### 4.2 레벨 JSON (챕터 파일 하나에 레벨 배열)
```json
{
  "chapter": 1,
  "title": { "ko": "첫 스케치북", "en": "First Sketchbook" },
  "theme": "pencil_white",
  "levels": [
    {
      "id": 12,
      "name": { "ko": "고양이", "en": "Cat" },
      "width": 7,
      "height": 7,
      "solution": [
        ".#...#.",
        ".##.##.",
        ".#...#.",
        ".#.#.#.",
        ".#...#.",
        ".#####.",
        "......."
      ],
      "clues": [
        { "x": 0, "y": 0, "v": [2] },
        { "x": 3, "y": 2, "v": [1, 1, 1] }
      ],
      "difficulty": 2.4,
      "techniques": ["clue_intersection", "no_2x2", "connectivity"],
      "tutorial": null
    }
  ]
}
```
- `solution`: `#` = 칠함, `.` = 빈칸. 행 문자열 배열 (사람이 읽고 diff 보기 쉽게).
- `clues.v`: 오름차순 정렬. `0`은 `[0]`.
- 숫자 칸은 반드시 solution에서 `.`여야 한다 (검증 도구가 확인).
- 위 예시 값은 형식 설명용이며 실제 유효한 퍼즐이 아니다.

### 4.3 세이브 (`Application.persistentDataPath/save.json`)
```json
{
  "version": 1,
  "progress": { "12": { "stars": 3, "hintsUsed": 0, "bestTimeSec": 95 } },
  "currentLevelId": 13,
  "inProgress": { "levelId": 13, "cells": "..#..x..", "elapsedSec": 40, "hintsUsed": 1 },
  "hints": 3,
  "adsRemoved": false,
  "firstLaunchDone": true
}
```
- `inProgress.cells`: 셀 상태 문자열 (`.` 빈칸, `#` 칠함, `x` 점).
- 쓰기: 셀 변경 후 0.5초 디바운스 + `OnApplicationPause(true)` 시 즉시 저장. 임시 파일에 쓴 뒤 교체(원자적 저장).
- 설정은 별도 `settings.json` 또는 PlayerPrefs.
- `version` 필드로 마이그레이션 대비.

---

## 5. 핵심 알고리즘 (Core)

### 5.1 숫자(clue) 계산 — `ClueCalculator`
- 주변 8칸을 **시계 방향 고리**로 순회: `(-1,-1) (0,-1) (1,-1) (1,0) (1,1) (0,1) (-1,1) (-1,0)`.
- 격자 밖 칸은 **빈칸 취급**.
- 고리를 따라 연속된 칠한 칸 묶음의 길이를 모은다. 고리는 원형이므로 **처음과 끝이 이어진 묶음은 합친다.**
- 결과를 오름차순 정렬. 칠한 칸이 없으면 `[0]`, 8칸 모두면 `[8]`.
- 성능용 사전 계산: 8비트 패턴(256가지) → clue 테이블. 고리 비트마스크로 바로 조회.

### 5.2 규칙 검사 — `RuleValidator`
정답과 비교하지 않고 규칙으로만 판정한다.
- `CheckClue(cell)`: 현재 칠한 칸 기준 clue == 목표? / 아직 가능? / 이미 불가능? (점 표시 칸은 확정 빈칸, 미정 칸은 둘 다 가능으로 보고 가능성 판단)
- `Find2x2()`: 칠한 2x2 목록.
- `IsConnected()`: 칠한 칸 BFS, 컴포넌트 1개인지.
- `FindTrappedComponents()`: 미정/칠함 칸으로 다른 칠한 칸에 닿을 수 없는 덩어리 (오류 표시용).
- **클리어 판정** = 모든 clue 일치 + 2x2 없음 + 연결 1개 + 숫자 칸 미칠함. 유일해답이 보장되므로 이게 곧 정답이다. (점 표시는 판정에 무관)

### 5.3 유일해답 솔버 — `UniqueSolver` (오프라인/에디터 전용)
목표: 해의 개수를 **최대 2개까지** 센다 (0 = 불가능, 1 = 유일, 2 = 중복).
1. 각 숫자 칸마다 고리 256패턴 중 clue와 맞고 격자 밖 제약에 맞는 **후보 패턴 목록**을 만든다.
2. 전파 루프
   - 후보들이 모두 동의하는 칸은 확정.
   - 확정된 칸과 모순되는 후보 제거.
   - 2x2: 세 칸이 칠함이면 나머지 하나는 빈칸.
   - 연결성: 미정+칠함 칸으로 연결했을 때 떨어지는 칠한 칸이 있으면 모순. 칠한 칸 전체가 이어지려면 반드시 지나야 하는 칸(단절점)은 칠함 확정.
3. 막히면 가장 후보가 적은 칸을 골라 분기(백트래킹), 해 2개를 찾으면 중단.
- 15x15까지 에디터에서 수 초 이내면 충분하다. 부족하면 SAT 솔버(외부 도구) 대안.

### 5.4 사람식 솔버 — `HumanSolver` (힌트 + 난이도)
사람이 쓰는 추론 기법을 쉬운 순서대로 적용하고, **처음으로 진전이 생긴 기법과 칸**을 반환한다.

| 등급 | 기법 ID | 내용 |
|---|---|---|
| 1 | `clue_trivial` | `0` 주변 전부 빈칸, `8` 주변 전부 칠함 |
| 1 | `clue_intersection` | 한 숫자의 모든 후보 패턴이 동의하는 칸 |
| 2 | `no_2x2` | 칠한 3칸이 있는 2x2의 나머지는 빈칸 |
| 2 | `connectivity_escape` | 칠한 덩어리의 유일한 탈출구는 칠함 |
| 3 | `connectivity_cut` | 연결을 끊는 칸은 칠할 수 없음 / 단절점은 칠함 |
| 3 | `clue_pair` | 인접한 두 숫자의 후보를 함께 걸러냄 |
| 4 | `contradiction_1step` | 한 칸을 가정해서 전파하면 바로 모순 → 반대로 확정 |

- **힌트**: 현재 판에서 HumanSolver를 한 단계 실행 → 칸 + 기법 ID → 기법별 설명 문구 템플릿. 틀린 칸이 있으면(정답과 비교) 그걸 먼저 알려준다.
- **MVP 대체안**: 정답과 비교해 아직 안 맞춘 칸 하나를 공개. 설명형은 이후.
- **난이도 점수**: 풀이 전체에서 쓴 최고 등급 + 등급별 사용 횟수 가중합. 사람식 솔버로 끝까지 못 풀면(추측 필요) **레벨로 쓰지 않는다.**

### 5.5 그림 → 퍼즐 생성 파이프라인 — `Generator`
```
입력 그림(에디터에서 그리기 또는 PNG)
 → ① 격자화: 목표 크기로 축소, 이진화(#/.)
 → ② 2x2 제거: 꽉 찬 2x2가 있으면 윤곽선만 남기거나(내부 비우기) 한 칸씩 제거, 그림 모양 보존 우선
 → ③ 연결: 떨어진 덩어리를 최단 경로로 잇기 (BFS, 그림 윤곽에 가까운 칸은 비용 낮게 → 자연스러운 선)
 → ④ 숫자 전체 배치: 빈칸 전부에 clue 계산
 → ⑤ 숫자 줄이기: 무작위 순서로 하나씩 빼보고 UniqueSolver가 여전히 유일 + HumanSolver로 풀리면 제거, 아니면 되돌림
 → ⑥ 난이도 측정, 목표 범위 밖이면 ⑤를 다른 순서/제거량으로 재시도
 → ⑦ JSON 저장
```
- ②③ 이후 그림이 어색해지면 에디터에서 사람이 수정하고 다시 ④부터.
- 숫자를 많이 남기면 쉬워지고, 많이 빼면 어려워진다. 챕터 목표 난이도에 맞춰 ⑤에서 제거 비율을 조절.
- ④ 단계에서도 유일하지 않을 수 있다. 그 경우 그림을 한두 칸 수정해야 한다 (에디터가 중복 해 차이를 보여주면 편함).

---

## 6. 레벨 에디터 (Editor 도구)
`TapaArt/Level Editor` EditorWindow. **레벨 150개를 만들려면 이 도구의 속도가 제일 중요하다.**
- 격자 크기 설정, 클릭/드래그로 정답 그림 그리기
- 실시간 경고: 2x2(빨강), 떨어진 덩어리(색 구분)
- 버튼: `PNG 불러오기`, `2x2 자동 정리`, `자동 연결`, `숫자 생성(전체)`, `숫자 줄이기`, `유일해 검증`, `난이도 측정`, `플레이 테스트`, `저장`
- 중복 해가 있으면 두 해의 차이 칸 표시
- 챕터 파일 불러오기/순서 변경
- `LevelBatchValidator`: 모든 챕터 JSON을 검사 (clue 재계산 일치, 유일해, 사람식 솔버로 풀림, ID 중복 없음). CI나 빌드 전에 실행.

---

## 7. 게임 화면 구현 메모
- 격자: 셀 프리팹(Image) 풀링. 15x15 = 225개로 충분히 가벼움. 셀 크기는 화면 폭 기준 계산, 10x10 초과 시 줌/팬 가능한 ScrollRect 또는 커스텀 줌.
- 입력: `IPointerDownHandler/IDragHandler` 로 셀 좌표 계산. 드래그 중에는 첫 칸이 바뀐 상태를 지나간 칸에 적용. 드래그 전체를 Undo 1단계로.
- 칠한 칸 연결 모양: 상하좌우 이웃 칠함 여부 4비트(16가지)로 스프라이트/라운딩 선택 → 선처럼 보이게.
- 오류 표시는 셀 변경 시 변경 칸 주변만 재검사 (연결성만 전체 BFS).
- 클리어 연출: 칠한 칸들을 한 칸에서 시작한 BFS 순서대로 딜레이를 줘 다시 그림.
- 맵 썸네일: 레벨 solution으로 작은 Texture2D를 런타임 생성 후 캐시 (레벨별 이미지 에셋 불필요).
- 맵: 레벨 노드 150+개 → 화면에 보이는 것만 활성화(풀링). 길은 스플라인 또는 미리 정한 좌표 패턴 반복.

---

## 8. 작업 순서 (마일스톤)
각 단계는 별도 코딩 세션 하나 분량을 목표로 한다.

| # | 내용 | 완료 기준 |
|---|---|---|
| M1 | Core: 모델, ClueCalculator, RuleValidator + 테스트 | EditMode 테스트 통과 (고리 wrap, 격자 경계, 0/8, 다중 숫자) |
| M2 | Core: UniqueSolver, HumanSolver + 테스트 | 알려진 퍼즐 몇 개로 유일/중복/난이도 판정 검증 |
| M3 | 게임 화면: 하드코딩 레벨 1개 플레이 | 탭/드래그/점/되돌리기, 오류 표시, 클리어 판정 |
| M4 | 레벨 에디터 + 생성 파이프라인 + 배치 검증 | 에디터에서 그림 → 검증된 JSON 저장 |
| M5 | 레벨 로딩, 세이브, 월드맵, 진행/해금 | 맵에서 레벨 선택 → 클리어 → 다음 해금, 재시작 후 복원 |
| M6 | 타이틀, 설정, 사운드, 튜토리얼 | 첫 실행 → 튜토리얼 → 맵 흐름 |
| M7 | 클리어 연출, 아트 적용, 힌트 UI | GDD의 클리어 연출 구현 |
| M8 | 레벨 제작 (챕터 1~2, 45개) | 배치 검증 통과 |
| M9 | 광고/IAP, 다국어, 출시 준비 | 내부 테스트 트랙 업로드 |

---

## 9. 코딩 규칙
- `Core`는 `UnityEngine` 참조 금지. `System`만 사용.
- 퍼즐 로직 변경 시 테스트 먼저/함께 추가.
- 매직 넘버(광고 주기, 힌트 개수 등)는 `GameConfig` ScriptableObject 한 곳에.
- 레벨 JSON은 사람이 손으로 고치지 않고 에디터 도구로만 수정 (검증 우회 방지).
