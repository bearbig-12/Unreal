# Unreal Engine Blueprint 학습 정리

> 언리얼 엔진 Blueprint 학습 내용을 날짜별로 정리합니다.

---

## 2026-03-10 - 언리얼 엔진 주요 클래스 계층 구조

### 클래스 계층도

## Unreal Object 계층 구조

```mermaid
graph TD
    A[Object]
    A1[Actor<br/>+ 레벨에 배치]
    A11[Controller<br/>+ 폰 제어]
    A111[PlayerController<br/>+ 플레이어 입력]
    A12[Pawn<br/>+ 컨트롤러 입력]
    A121[Character<br/>+ 물리적 움직임]
    A2[ActorComponent<br/>+ 액터에서 역할 수행]
    A21[SceneComponent<br/>+ 트랜스폼 보유]

    A --> A1
    A1 --> A11
    A11 --> A111
    A1 --> A12
    A12 --> A121
    A --> A2
    A2 --> A21
```

### 클래스 역할 정리표

| 클래스 | 상위 클래스 | 주요 특징 | 용도 |
|--------|------------|-----------|------|
| **Object** | - | 모든 UE 객체의 루트 | 가비지 컬렉션, 리플렉션 시스템 기반 |
| **Actor** | Object | 레벨에 배치 가능 | 게임 세계에 존재하는 모든 오브젝트의 기반 |
| **ActorComponent** | Object | 액터에 붙어서 역할 수행 | 기능 단위를 모듈화하여 액터에 부착 |
| **Controller** | Actor | 폰(Pawn)을 제어 | AI 또는 플레이어가 폰을 조종하는 로직 담당 |
| **PlayerController** | Controller | 플레이어 입력 처리 | 키보드/마우스/게임패드 입력을 폰에 전달 |
| **Pawn** | Actor | 컨트롤러로부터 입력 수신 | 플레이어 또는 AI가 조종할 수 있는 엔티티 |
| **Character** | Pawn | 물리적 움직임 (CharacterMovement) | 걷기/달리기/점프 등 캐릭터 이동 내장 |
| **SceneComponent** | ActorComponent | 트랜스폼(위치/회전/스케일) 보유 | 3D 공간에서 위치를 가지는 컴포넌트의 기반 |

---

### 핵심 개념 정리

#### Object
- 언리얼 엔진의 모든 클래스의 최상위 부모
- **가비지 컬렉션**, **직렬화(Serialization)**, **리플렉션(Reflection)** 시스템 제공
- `UCLASS()`, `UPROPERTY()`, `UFUNCTION()` 매크로가 동작하는 근간

#### Actor vs ActorComponent
- **Actor**: 레벨에 직접 배치되는 독립적인 개체 (예: 캐릭터, 총, 트리거 박스)
- **ActorComponent**: Actor에 붙어서 기능을 추가하는 모듈 (예: 체력 컴포넌트, 인벤토리 컴포넌트)
- 컴포넌트 기반 설계로 코드 재사용성 향상

#### Controller / PlayerController
- **Controller**는 Pawn과 분리된 설계 → 같은 Pawn을 AI/플레이어 모두 조종 가능
- **PlayerController**는 플레이어 1명당 1개 존재, HUD·카메라 관리도 담당
- Pawn이 파괴되어도 Controller는 유지됨 → 리스폰 로직 구현에 활용

#### Pawn / Character
- **Pawn**: 조종 가능한 최소 단위, 이동 방식은 직접 구현해야 함
- **Character**: Pawn + `CharacterMovementComponent` 내장
  - 걷기, 달리기, 점프, 수영, 비행 등 기본 이동 제공
  - `CapsuleComponent`(충돌), `SkeletalMeshComponent`(메시) 기본 포함

#### SceneComponent
- 3D 공간 내 **트랜스폼(Transform)** 을 가지는 컴포넌트의 기반 클래스
- 계층 구조로 부모-자식 관계 구성 가능 (예: 총구 위치를 손 본에 부착)
- `UStaticMeshComponent`, `UCameraComponent` 등이 SceneComponent를 상속

---

### Blueprint 실습 포인트

- [ ] `Character` 블루프린트 생성 후 `CharacterMovementComponent` 설정 탐색
- [ ] `PlayerController`에서 입력 바인딩 (Enhanced Input System)
- [ ] `ActorComponent` 블루프린트로 재사용 가능한 체력 시스템 만들기
- [ ] `SceneComponent`를 이용한 소켓 부착 실습

---

## 2026-03-12 - 캐릭터 기본 조작 및 카메라 시야 조작 구현

### 구현 내용

Enhanced Input System을 활용하여 캐릭터 이동과 마우스 시야 조작을 구현했다.

---

### 개념 정리

#### Enhanced Input System 이란?

기존 레거시 Input System(Project Settings → Input)을 대체하는 UE5 표준 입력 시스템이다.

| 구성 요소 | 역할 |
|-----------|------|
| **Input Action (IA)** | 논리적 입력 단위 정의 (이동, 시야, 점프 등) |
| **Input Mapping Context (IMC)** | 물리 키/버튼을 IA에 매핑. 런타임에 교체 가능 |
| **Modifier** | 입력값을 가공 (반전, 축 교환, 감도 등) |
| **Trigger** | 언제 액션이 발동되는지 조건 설정 |

**장점:** IMC를 런타임에 교체할 수 있어서 UI 모드, 전투 모드, 탈것 탑승 등 상황별 입력 전환이 쉽다.

---

#### Input Action Value Type

| 타입 | 핀 | 사용 예 |
|------|----|---------|
| `Bool` | 단일 bool | 점프, 공격 (눌림/뗌) |
| `Axis1D (float)` | 단일 float | 트리거 버튼, 마우스 휠 |
| `Axis2D (Vector2D)` | X, Y | 마우스 이동, 스틱 |
| `Axis3D (Vector)` | X, Y, Z | 6DOF 컨트롤러 |

---

#### Modifier 종류

| Modifier | 동작 | 주요 사용처 |
|----------|------|------------|
| **Negate** | 값을 반전 (-1 곱셈) | S키(후진), 마우스 Y축 보정 |
| **Swizzle Input Axis Values** | 축 순서 교환 (YXZ 등) | A/D키 → Y축으로 이동시킬 때 |
| **Scale** | 값에 스칼라 곱 | 감도 조절 |
| **Dead Zone** | 일정 범위 내 입력 무시 | 스틱 드리프트 방지 |

> **왜 D키에 Swizzle이 필요한가?**
> 키보드 키는 기본적으로 `Axis1D`(X축)로 값이 들어온다.
> 좌우 이동은 Y축이 필요하므로 Swizzle(YXZ)로 X→Y로 바꿔줘야 한다.

---

#### Trigger 종류

| Trigger | 발동 조건 |
|---------|-----------|
| **Triggered** (기본) | 입력값이 임계값 이상인 동안 매 프레임 |
| **Down** | 입력이 눌린 상태인 동안 |
| **Pressed** | 눌리는 순간 1회 |
| **Released** | 떼는 순간 1회 |
| **Hold** | 일정 시간 이상 누르고 있을 때 |

---

#### Controller Rotation vs Pawn Rotation

언리얼에서 회전은 **Controller**가 들고 있는 `ControlRotation`이 기준이다.

```
마우스 이동
    ↓
Add Controller Yaw/Pitch Input (Pawn 함수 → 내부적으로 Controller에 전달)
    ↓
PlayerController.ControlRotation 변경
    ↓
SpringArm (Use Pawn Control Rotation = true) → ControlRotation 따라감
    ↓
Camera 회전
```

- `Add Controller Yaw Input` / `Add Controller Pitch Input` : Pawn의 함수이지만 실제로는 Controller의 회전값을 바꿈
- Character의 **Use Controller Rotation Yaw**를 끄면 몸통은 고정되고 카메라(SpringArm)만 회전

---

#### SpringArm (카메라 붐)

캐릭터와 카메라 사이의 거리를 유지하고, 벽에 가까워지면 자동으로 카메라를 당겨주는 컴포넌트.

| 주요 설정 | 설명 |
|-----------|------|
| `Target Arm Length` | 카메라와 캐릭터 사이 거리 |
| `Use Pawn Control Rotation` | Controller 회전을 따라 SpringArm 회전 (**카메라 회전의 핵심**) |
| `Do Collision Test` | 벽 충돌 시 카메라 자동 당김 |
| `Enable Camera Lag` | 카메라가 부드럽게 따라오는 딜레이 효과 |

---

### 파일 구성

| 파일 | 역할 |
|------|------|
| `BP_GameMode` | 기본 게임 모드 설정 |
| `BP_PlayerController` | 입력 처리 및 캐릭터 조작 로직 |
| `BP_Character` | 캐릭터 (SpringArm + Camera 포함) |
| `IMC_Default` | Input Mapping Context (키-액션 매핑) |
| `IA_Move` | 이동 Input Action (Axis2D) |
| `IA_Look` | 시야 Input Action (Axis2D) |

---

### Enhanced Input 구조

```
마우스 이동 / 키 입력
    ↓
IMC_Default (키 → 액션 매핑)
    ↓
IA_Move / IA_Look (액션 정의)
    ↓
BP_PlayerController (BeginPlay에서 IMC 등록 + 액션 바인딩)
    ↓
Add Movement Input / Add Controller Yaw·Pitch Input
```

---

### BP_PlayerController - BeginPlay 설정

- `Get Enhanced Input Local Player Subsystem` → `Add Mapping Context (IMC_Default, Priority 0)`
- Enhanced Input을 사용하려면 BeginPlay에서 반드시 IMC를 등록해야 함

---

### IA_Move 바인딩

| 키 | Modifier | 역할 |
|----|----------|------|
| W | 없음 | 전진 (X+) |
| S | Negate | 후진 (X-) |
| D | Swizzle (YXZ) | 우이동 (Y+) |
| A | Swizzle (YXZ) + Negate | 좌이동 (Y-) |

- `EnhancedInputAction IA_Move` → Triggered → `Add Movement Input` (Target: Get Controlled Pawn)

---

### IA_Look 바인딩

- `EnhancedInputAction IA_Look` → Triggered
  - `Action Value X` → `Add Controller Yaw Input`
  - `Action Value Y` → `Add Controller Pitch Input`
  - Target: `Get Controlled Pawn`

**IMC_Default 매핑:** `Mouse XY 2D-Axis` → `IA_Look`

**IA_Look Modifier 설정:**
- Negate: Y만 체크 ✅ (X 체크 해제 - 좌우 반전 방지)
- Y축 Negate는 마우스 위아래 방향 보정을 위해 필요

---

### BP_Character - 카메라 설정

```
CapsuleComponent
└── Mesh (SkeletalMesh)
└── SpringArm
    └── Camera
```

**SpringArm 핵심 설정:**
- `Use Pawn Control Rotation` = **True** ← 컨트롤러 회전을 카메라가 따라감

**Camera 설정:**
- `Use Pawn Control Rotation` = **False** (SpringArm이 이미 처리)

---

### 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| 마우스 이동해도 카메라 회전 안 됨 | IMC_Default에서 IA_Look이 `Left Mouse Button`에 매핑되어 있었음 | `Mouse XY 2D-Axis`로 변경 |
| 카메라가 여전히 안 돌아감 | SpringArm `Use Pawn Control Rotation` 미설정 | True로 변경 |
| 좌우 시야가 반대로 조작됨 | IA_Look Negate 모디파이어가 X축까지 반전 | Negate에서 X 체크 해제 |

---

### Blueprint 실습 포인트 업데이트

- [x] `Character` 블루프린트 생성 후 `CharacterMovementComponent` 설정 탐색
- [x] `PlayerController`에서 입력 바인딩 (Enhanced Input System)
- [ ] `ActorComponent` 블루프린트로 재사용 가능한 체력 시스템 만들기
- [ ] `SceneComponent`를 이용한 소켓 부착 실습

---

*학습 환경: Unreal Engine | Blueprint*
