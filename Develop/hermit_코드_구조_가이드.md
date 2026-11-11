# hermit 파트 코드 구조 가이드

> 기준: `main` 브랜치 `6225d6c` (PR #15 머지 후), 2026-11-11
> 대상: 팀원이 hermit 파트 코드(흐름·자본·판정·결과·공통 UI)를 이해하고 연결할 때 참고용
> 경로는 모두 `UGJProject/Assets/Scripts/` 기준입니다.

---

## 1. 한눈에 보기

| 폴더 | 역할 | 주요 파일 |
|---|---|---|
| `Core/` | 시스템끼리 연결하는 이벤트 | `GameEvents.cs` |
| `Core/Flow/` | 씬 이동, 손님 순서, 하루·승패, 결과 화면 | `SceneFlow.cs`, `SceneFlowLink.cs`, `GameFlowManager.cs`, `GameFlow.cs`, `DayCycle.cs`, `PersistentSystems.cs`, `ResultScreen.cs` |
| `Core/Capital/` | 자본(돈) 저장·표시·연출 | `CapitalSystem.cs`, `CapitalDisplay.cs`, `CapitalFeedback.cs` |
| `Core/Judgement/` | 팁 표시 | `TipFeedback.cs` (+ 미사용 `JudgementSystem.cs`) |
| `UI/` | 공통 UI 연출 | `ScreenFader.cs`, `SettlementPopup.cs`, `DayDisplay.cs`, `GameOverDialogue.cs`, `SettingsMenu.cs`, `ButtonClickDelay.cs`, `ClickDelayInstaller.cs`, `UIHoverScale.cs` |

**핵심 규칙 세 가지**
1. **돈은 반드시 `CapitalSystem.Add(금액)`으로만 바뀝니다.** 화면 숫자·연출은 전부 여기서 나가는 신호(`GameEvents.CapitalChanged`)를 받아서 갱신됩니다.
2. **씬 이동은 `SceneFlow`의 함수로만 합니다.** (`SceneManager.LoadScene` 직접 호출 X) 페이드와 손님 순서 기록이 함께 처리됩니다.
3. **시스템 사이 연결은 `GameEvents`로 합니다.** 보내는 쪽은 `GameEvents.RaiseXxx(...)`, 받는 쪽은 `GameEvents.Xxx += 함수`.

---

## 2. 어디에 붙어 있나 (씬·프리팹 배치)

| 오브젝트 | 위치 | 붙어 있는 hermit 컴포넌트 | 비고 |
|---|---|---|---|
| **GameSystems** 프리팹 | Counter, Making 씬 | `PersistentSystems`, `CapitalSystem`, `GameFlowManager`, `JudgementSystem`, `TipFeedback`, `DayDisplay` | 씬이 바뀌어도 **하나만 유지**(DontDestroyOnLoad) |
| **CapitalUI** 프리팹 | Counter, Making 씬 | `CapitalDisplay`, `CapitalFeedback` | 화면 위 돈 표시 |
| **SettingsUI** 프리팹 | Counter, Making 씬 | `SettingsMenu`, `UIHoverScale`, `ButtonClickDelay` | ESC/Setting 버튼 팝업 |
| **SceneFlowLink** 오브젝트 | Counter, Making, Result 씬 | `SceneFlowLink` (+ Making에만 `SettlementPopup`) | Making만 `hookResultSubmit = true` |
| **ClickDelayInstaller** | Counter, Making 씬 | `ClickDelayInstaller` | 씬의 모든 버튼에 클릭 지연 자동 부착 |
| **ResultScreen** 오브젝트 | Result 씬 | `ResultScreen`, `GameOverDialogue` | |
| `Resources/ScreenFader.prefab` | 코드가 자동 생성 | `ScreenFader` | 씬에 둘 필요 없음 |

---

## 3. 게임 한 바퀴: 어떤 코드가 무엇을 일으키나

```
Title ──Start──▶ Counter(손님 대화) ──책상 클릭──▶ Making(키링 제작) ──Submit──▶ 결산 팝업 ──Next──▶ Counter(다음 손님)
                                                                                         └─ 하루 끝/게임오버 ─▶ Result ──Next Day──▶ Counter
                                                                                                                  └─ Title ──▶ Title
```

### 3-1. Title → Counter (새 게임 시작)
| 순서 | 코드 | 하는 일 |
|---|---|---|
| 1 | `Title/TitleScreen.cs` `OnStartButtonClicked()` (Hook) | `SceneFlow.StartNewGame()` 호출 |
| 2 | `SceneFlow.StartNewGame()` | `PersistentSystems.ResetSession()`으로 GameSystems 제거(자본·흐름 초기화) → 손님 번호 0, 결과 비움 → `ScreenFader.LoadScene("Counter")` |
| 3 | `ScreenFader.LoadScene()` | 화면 어두워짐 → 씬 로드 → 밝아짐. 전환 중에는 클릭 막음, 중복 요청 무시 |

### 3-2. Counter (손님 대화)
| 코드 | 하는 일 |
|---|---|
| `SceneFlowLink.Awake()` | `SceneFlow.NextCustomerIndex`를 `DialogueManager.customer`에 넣음 → **몇 번째 손님부터 대화할지 결정** |
| `Dialogue/DialogueManager.Start()` (macro) | 대화 시작 + `Customer_Data.Set_CustomerReceipt(...)`로 **이 손님의 레시피를 저장** (Making에서 판정에 사용) |
| `Dialogue/DeskInteraction.cs` (macro) 책상 클릭 | `SceneFlow.GoToMaking(manager)` 호출 |
| `SceneFlow.GoToMaking()` | 다음 손님 번호(`customer + 1`)와 손님 수를 기록 → Making 로드 |
| `SceneFlowLink.OnDestroy()` | 팀원 코드가 `SceneManager`로 직접 씬을 바꿔도 다음 손님 번호가 이어지도록 `SceneFlow.NoteCounterLeft()` 기록 |
| `DayDisplay` (GameSystems) | Counter·Making 왼쪽 위 "Day N" 표시, 하루 첫 손님 때 가운데 "Day N" 배너 |

### 3-3. Making (키링 제작 → Submit)
| 순서 | 코드 | 하는 일 |
|---|---|---|
| 1 | `Making/ObjectDragManager.EndDrag()` (macro) | 재료를 내려놓을 때 `CapitalSystem.Add(-1f)` → **재료비 $1 차감** |
| 2 | Submit 버튼 → `Making/Result.Result_Kiring()` (macro, 씬에서 연결) | 판정 계산 → `capital.Add(earned)` → **판매 수익 추가** → `GameEvents.RaiseKeyringJudged(...)` (팁 유무 알림) |
| 3 | 같은 Submit 버튼 → `SettlementPopup.Show()` | `SceneFlowLink`가 실행 중에만 Submit 버튼에 덧붙인 동작 (Result_Kiring **다음에** 실행) |
| 4 | `SettlementPopup` | 반투명 패널 + 결산 상자 → Income / Expense / Total을 하나씩 표시, 팁이면 Tip! 도장 → Next Customer(또는 Result) 버튼 |
| 5 | Next 버튼 → `SceneFlow.ReturnFromMaking()` | `GameEvents.RaiseCustomerFinished(null)` → (하루 끝/게임 오버면) Result, 아니면 Counter 로드 |

> ⚠️ Submit 버튼의 `Result_Kiring`에서 오류(예: 레시피 null)가 나면 그 뒤에 붙은 `SettlementPopup.Show()`가 실행되지 않아 **다음으로 넘어가지 않습니다.** 넘어가지 않으면 Console의 오류부터 확인해 주세요.

### 3-4. 손님 처리 완료 → 하루 끝·승패
| 코드 | 하는 일 |
|---|---|
| `GameFlowManager.OnCustomerFinished()` | 처리한 손님 수 +1. **자본 < 0이면 남은 손님과 상관없이 바로 게임 오버**(`GameFlow.ForceGameOver()`) |
| `GameFlow.CustomerFinished()` | 하루 손님 수(`customersPerGame`, 기본 3)를 채우면 종료로 판단 |
| `GameFlowManager.EndGame()` | 결과(`GameResult`) 생성 → `GameEvents.RaiseGameEnded(result)` |
| `SceneFlow` (GameEnded 구독) | `SceneFlow.LastResult`에 결과 저장 → `ReturnFromMaking()`이 Result 씬으로 보냄 |

`GameResult` 판정 기준 (`Core/Flow/GameFlow.cs`, `GameFlowManager.EndGame()`):
- `isWin = 자본 >= 0` (`GameFlow.IsWin`)
- `isDayEnd = isWin && 마지막 날이 아님` → **하루만 끝남** (다음 날로)
- `isWin && 마지막 날(7일차, DayCycle.TotalDays)` → **WIN**
- `!isWin` → **LOSE**

### 3-5. Result 씬
| 코드 | 하는 일 |
|---|---|
| `ResultScreen.Start()` | 제목: `Day N Clear` / `WIN` / `LOSE`, 아래: Money, Day N / 7 |
| `ResultScreen.Awake()` | 하루만 끝났으면 Title 버튼을 **Next Day** 버튼으로 바꿈 (씬 파일은 그대로, 실행 중에만) |
| Next Day → `DayCycle.GoToNextDay()` | 지금 자본을 기억 → 일차 +1 → `SceneFlow.StartNewGame()` (손님 순서 처음부터) |
| `CapitalSystem.Awake()` | `DayCycle.TakeStartingCapital()`로 **전날 자본을 이어받음** (없으면 시작 자본 10) |
| `GameOverDialogue` | LOSE일 때 대사창(팀원 `Dialogue_Canvas 1` 프리팹)이 왼쪽 아래에서 올라오며 "Bankrupt!" |
| Title 버튼 → `SceneFlow.GoToTitle()` | GameSystems 제거 후 Title 로드 |

---

## 4. 돈이 바뀌는 곳 (최종적으로 자본을 바꾸는 코드)

**모든 돈 변화는 `Core/Capital/CapitalSystem.cs`의 `Add(float amount)` 한 곳을 거칩니다.**
`Add`는 `Current`를 바꾸고 `GameEvents.RaiseCapitalChanged(Current)`를 보냅니다.

| 언제 | 호출하는 코드 | 금액 | 실제 게임에서 사용 |
|---|---|---|---|
| 재료를 키링에 놓을 때 | `Making/ObjectDragManager.EndDrag()` (macro) → `capital.Add(-1f)` | **−$1** | ✅ |
| Submit 판정 후 | `Making/Result.Result_Kiring()` (macro) → `capital.Add(earned)` | Perfect: 판매가×1.1 / Pass: 판매가 / Refund: 0 (판매가 = 레시피 요소 수 × $3) | ✅ |
| 다음 날 시작 | `CapitalSystem.Awake()` ← `DayCycle.TakeStartingCapital()` | 전날 자본으로 시작 | ✅ |
| 주문 수락 시 선지급 | `CapitalSystem.OnOrderAccepted()` ← `GameEvents.OrderAccepted` | +레시피 판매가 | ❌ (실제 흐름에서 이 이벤트를 보내는 곳 없음) |
| 판정 이벤트 | `CapitalSystem.OnKeyringJudged()` ← `GameEvents.KeyringJudged` | 팁 +, 환불 − | ⚠️ 받긴 하지만 `Result.cs`가 금액을 0으로 보내서 **변화 없음** (중복 반영 방지) |

**돈을 보여 주는 곳** (모두 `GameEvents.CapitalChanged`를 받음):
- `CapitalDisplay` : 숫자 텍스트 (소수점 2자리)
- `CapitalFeedback` : +/− 숫자 이펙트, 숫자 강조, 임시 효과음
- `SettlementPopup` : Making 씬에서 생긴 + 는 Income, − 는 Expense로 모아 결산 표시

> 돈을 바꾸는 새 기능을 만들 때: `FindAnyObjectByType<CapitalSystem>().Add(금액)`을 호출하면 표시·연출·결산이 자동으로 따라옵니다.

---

## 5. 판정 (구현비율)

실제 게임 판정은 **`Making/Result.cs`의 `Result_Kiring()`**(macro 파일, hermit이 일부 수정)에서 합니다.

```
구현비율 = (일치 개수 − 불일치 개수) / 레시피 요소 총 개수
일치   += Min(놓은 개수, 레시피 개수)      ← 부적(클로버·음표·무지개·숫자) 종류별
불일치 += Max(0, 놓은 개수 − 레시피 개수)
메인 키링·잠금장치(Null/A/B)는 맞으면 일치 +1, 틀리면 불일치 +1
100% 이상 → Perfect(판매가 + 10% 팁) / 80% 이상 → Pass(판매가) / 80% 미만 → Refund(0)
```

- 레시피 데이터: `Making/Receipt_Data.cs` (ScriptableObject, `Assets/Data/Making/Receipt*.asset`)
- 손님별 레시피: 대화 데이터(`DialogueData.receiptData`) → `Customer_Data.receipt` → `Result.Start()`에서 사용
- `Core/KeyringJudge.cs`, `Core/Judgement/JudgementSystem.cs`, `Core/Data/*`(RecipeData 등)는 초기 공통 데이터 규칙용으로 만든 판정 구조입니다. **현재 실제 흐름에서는 쓰지 않고**, 개발용 `Debug/FlowDebugKeys.cs`에서만 사용합니다.

---

## 6. GameEvents (시스템 연결 신호)

`Core/GameEvents.cs` — 보내는 쪽은 `GameEvents.RaiseXxx(...)`, 받는 쪽은 `+=`/`-=`로 구독.

| 이벤트 | 보내는 곳 (실제 흐름) | 받는 곳 |
|---|---|---|
| `CapitalChanged(float)` | `CapitalSystem.Add()` 등 | `CapitalDisplay`, `CapitalFeedback`, `SettlementPopup` |
| `KeyringJudged(customer, result)` | `Result.Result_Kiring()` | `TipFeedback`, `SettlementPopup`(Tip! 도장), `CapitalSystem`(금액 0이라 변화 없음) |
| `CustomerFinished(customer)` | `SceneFlow.ReturnFromMaking()` | `GameFlowManager`(손님 수·종료 판단), `JudgementSystem` |
| `GameEnded(GameResult)` | `GameFlowManager.EndGame()` | `SceneFlow`(LastResult 저장) |
| `PhaseChanged(GamePhase)` | `GameFlowManager` | (구독자 없음 — 화면 전환 담당이 쓸 수 있도록 남겨 둠) |
| `OrderAccepted`, `IngredientUsed`, `KeyringServed` | (실제 흐름에서 보내는 곳 없음) | `CapitalSystem`, `GameFlowManager`, `JudgementSystem` |

> 이벤트를 구독한 MonoBehaviour는 `OnDisable`/`OnDestroy`에서 반드시 `-=`로 해제해 주세요.

---

## 7. 공통 UI 연출 (hermit)

| 파일 | 하는 일 | 쓰는 법 |
|---|---|---|
| `UI/ScreenFader.cs` | 씬 전환 Fade Out(0.5s) → 검은 화면 0.1s → Fade In(0.6s), SmoothStep. 게임 실행 시 첫 Fade In. 직접 `SceneManager`로 바꾼 씬도 Fade In | `ScreenFader.LoadScene("씬이름")` (보통은 `SceneFlow`가 호출) / 시간은 `Resources/ScreenFader.prefab` |
| `UI/ButtonClickDelay.cs` | 버튼을 누르면 0.15초 뒤에 OnClick 실행(눌림 연출·연타 방지), 일시정지 중에도 동작 | 버튼에 붙이기 |
| `UI/ClickDelayInstaller.cs` | 씬의 모든 버튼(꺼진 것 포함)에 `ButtonClickDelay`를 실행 중에 자동 부착 | 씬에 하나 배치 |
| `UI/UIHoverScale.cs` | 커서를 올리면 1.15배 확대, 일시정지 중에도 동작 | 버튼에 붙이기 |
| `UI/SettingsMenu.cs` | 설정 팝업: 볼륨(PlayerPrefs 저장) / Resume / Title / Quit, ESC로 열고 닫기, 열려 있으면 일시정지 | `SettingsUI` 프리팹 배치 |
| `UI/SettlementPopup.cs` | Submit 후 결산 팝업 (3-3 참고). UI는 실행 중에 코드로 생성 | Making `SceneFlowLink` 오브젝트에 부착됨. 문구·색·시간은 인스펙터 |
| `UI/DayDisplay.cs` | "Day N" 라벨과 하루 시작 배너 | GameSystems 프리팹에 부착됨 |
| `Core/Judgement/TipFeedback.cs` | 화면 중앙 Tip!/No Tip (결산 팝업이 있는 씬에서는 표시 안 함) | GameSystems 프리팹에 부착됨 |
| `UI/GameOverDialogue.cs` | LOSE일 때 대사창 슬라이드 업 + "Bankrupt!" | Result 씬 |

---

## 8. 씬을 넘어 유지되는 값 (static)

| 값 | 위치 | 초기화 |
|---|---|---|
| `SceneFlow.NextCustomerIndex` / `LastResult` | `Core/Flow/SceneFlow.cs` | `StartNewGame()` |
| `DayCycle.CurrentDay` / 이어받을 자본 | `Core/Flow/DayCycle.cs` | `DayCycle.BeginNewGame()` |
| GameSystems 오브젝트 | `PersistentSystems.Instance` | `PersistentSystems.ResetSession()` |
| `Customer_Data.receipt`, 재료 재고 (macro) | `Making/Customer_Data.cs` | (팀원 코드) |

> 프로젝트가 **Enter Play Mode에서 Domain Reload를 끈 상태**라 static 값이 Play 사이에 남습니다. hermit 코드는 `[RuntimeInitializeOnLoadMethod(SubsystemRegistration)]`로 Play 시작마다 초기화합니다. 새 static 값을 만들 때 같은 방식으로 초기화해 주세요.

---

## 9. 팀원 작업 시 주의할 점

1. **씬 이동**은 `SceneFlow.GoToMaking(dialogueManager)`, `SceneFlow.ReturnFromMaking()`, `SceneFlow.GoToTitle()`을 써 주세요. 손님 번호·결과·페이드가 함께 처리됩니다.
2. **Making의 Submit 버튼**은 씬에서 `Result.Result_Kiring`을 연결해 두면 됩니다. 결산 팝업과 다음 씬 이동은 `SceneFlowLink`가 자동으로 덧붙입니다. (메서드 이름 `Result_Kiring`으로 버튼을 찾으므로 이름을 바꾸면 연결이 끊깁니다.)
3. **돈**은 `CapitalSystem.Add()`로만 바꿔 주세요. 직접 UI 텍스트를 바꾸면 결산·연출과 어긋납니다.
4. **손님 레시피**는 대화 데이터(`DialogueData.receiptData`)에 꼭 넣어 주세요. 비어 있으면 Making에서 판정·레시피북이 오류로 멈춥니다. Making 씬만 단독 Play해도 레시피가 비어 오류가 납니다(Title 또는 Counter에서 시작).
5. Counter·Making 씬에는 **GameSystems, CapitalUI, SettingsUI** 프리팹이 있어야 돈·흐름·설정이 동작합니다.

---

## 10. 알려진 문제·정리 예정

| 항목 | 내용 |
|---|---|
| Title에서 새 게임 시 일차 초기화 누락 | Hook의 `TitleScreen.OnStartButtonClicked()`가 `DayCycle.BeginNewGame()`을 호출하지 않아, 게임 도중 Title로 돌아가 다시 시작하면 **일차가 1로 돌아가지 않습니다.** `SceneFlow.StartNewGame()` 앞에 `DayCycle.BeginNewGame();` 한 줄이 필요합니다. |
| 결과창 타이밍 (v0.0.3) | 지금은 키링마다 결산 팝업이 뜹니다. v0.0.3에서 **하루에 한 번** 결과창으로 바꿀 예정입니다. |
| 손님 반응 대화 (v0.0.3) | 판정 결과는 `GameEvents.KeyringJudged`로 이미 보내고 있으니, 반응 대화는 이 이벤트를 받아 `result.HasTip`(팁 유무)으로 분기하면 됩니다. |
| 쓰지 않는 코드 | `SceneFlowLink.customerReceipts`(손님별 레시피는 팀원 방식 사용), `Core/Capital/IngredientCostWatcher.cs`(재료비는 ObjectDragManager가 차감), `UI/ViewportAnchor.cs`, `Debug/FlowDebugKeys.cs`, `JudgementSystem`/`KeyringJudge`/`Core/Data/*` |
| 레시피북 화면 크기 | 팀원 `MakingBookCanvas`가 Constant Pixel Size라 Game 창이 작으면 책이 화면 밖에 있습니다 (Scale With Screen Size 권장). |
