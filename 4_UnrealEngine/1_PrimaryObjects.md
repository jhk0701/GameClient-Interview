# UE 주요 객체
## UObject
* 언리얼 엔진에서의 가장 기반(상위) 클래스
* 트랜스폼, 렌더링 기능이 없는 논리적 객체
    * Level에 배치할 순 없는 단순 객체
* 언리얼 엔진의 GC 시스템을 지원하는 클래스
* 에디터, 리플렉션, 직렬화, BP 연동 등 언리얼 엔진의 기능을 제공하는 출발점
### 인스턴스 생성
* `new` 연산자를 사용해선 안됨
    * 모든 UObject의 메모리는 언리얼 엔진의 GC를 통해 관리됨
    * `new`, `delete` 를 사용해서 수동으로 메모리를 관리하면 메모리가 손상될 수 있음
* 동적 생성 시, `NewObject<>`, `CreateDefaultSubobject<>` 사용
    * `CreateDefaultSubobject` : 생성자에서만 사용 가능 (CDO 생성 관련)
    * `NewObject` : 게임 런타임에서 UObject를 생성하는 표준 방법
        * 생성한 인스턴스는 UPROPERTY() 매크로가 붙은 포인터에 할당돨 때 GC에서 참조 중이라 판단
        * 그렇지 않으면 Root에서부터 참조가 닿지 않는 것으로 간주하고 GC가 정리해버림
### 사용처
* 게임 내 논리적 데이터 관리
* 에디터 확장 기능 작성
* BP 함수 라이브러리 작성
* 액터없이 GC 대상 객체 생성
* UI나 리소스 설정 데이터를 저장할 객체 생성
## AActor
* 레벨에 배치할 수 있는 객체
    * 레벨에 배치하기 위한 모든 객체들의 기반 클래스
* UObject 파생 클래스 중 하나
* 위치, 회전, 스케일 등 트랜스폼(Transform) 정보를 가짐
    * 실질적으로 움직이기 위해선 RootComponent가 필요
    * Transform 관련 함수는 액터에 있음
* 컴포넌트들을 포함한 복합적인 객체를 구성할 수 있음
    * 이동, 렌더링 방식 등 여러 기능을 가진 컴포넌트를 액터에 붙여 기능을 구성
* 충돌, 렌더링, 네트워크 동기화, 수명 관리 등을 지원
### 인스턴스 생성
* AActor는 `SpawnActor`를 통해서 동적 생성 가능
### 네트워크 동기화
* 멀티플레이 환경에서 Replication (복제) 가능
### 라이프사이클 함수가 존재
* Beginplay, Tick, Endplay, Destroyed 등 라이프사이클 함수가 존재
* 주요 라이프사이클 함수의 호출 시점
    * BeginPlay
        * 게임 시작 시 레벨에 배치된 액터
        또는 런타임에 SpawnActor()로 생성된 액터가 
        초기화를 완료한 후 최초 1회 호출
        * 게임 로직 초기화에 사용
    * Tick
        * 매 프레임마다 호출
        * bCanEverTick = false;로 비활성화 가능 (성능 최적화)
        * TickInterval로 호출 간격 조정 가능
    * EndPlay : 액터가 정리될 때 호출
        * 액터가 Destory(), 레벨 전환, 게임 종료 등의 이유로 월드에서 제거될 때 호출
        * EEndPlayReason으로 호출 원인 구분 가능
        * 타이머, 델리게이트 등 리소스 정리에 적합한 시점
* Constructor와 BeginPlay의 차이
    * Constructor : 
        * 에디터에서도 실행.
        * 컴포넌트 생성 및 기본값 설정 등의 목적을 위해 사용
        * 단, 게임 로직이 포함되어선 안됨 -> 아직 월드가 실행된 시점이 아니기 때문
    * BeginPlay :
        * 게임 실행 시점에서만 호출
        * 월드, 다른 액터가 참조 가능
        * 게임 로직 최적화에 적합

## APawn
* 플레이어나 AI가 조종할 수 있는 액터
* 컨트롤러를 통해 입력을 받아 움직임을 수행
### 주요기능
* Controller 연결
    * Possess/Unpossess : 컨트롤러가 해당 폰을 조정하거나 해제
* 입력 처리 기능 : PlayerController를 통한 입력 처리
* MovementComponent : 움직임에 관련된 컴포넌트를 통해 이동 가능
### 폰을 쓰는 이유
* 단순한 움직임 로직이 필요한 경우
* 중력, 점프, 복잡한 충돌이 필요없는 객체
* 캐릭터 이외의 조종 가능한 게임 오브젝트 구현 용도
    * 드론, 탱크, 비행기 등
* 기타 AI 제어 대상으로 활용

## ACharacter
* 언리얼 엔진에서 지상형 캐릭터의 움직임과 애니메이션을 쉽게 구현할 수 있도록 만든 APawn 확장 클래스
* 내장된 UCharacterMovementComponent로 복잡한 움직임 지원
* 애니메이션 시스템과 쉽게 연동가능
    * 내장된 USkeletaMeshComponent를 통해 쉽게 애니메이션을 실행할 수 있도록 지원
* 기본적으로 캡슐 충돌, 스켈레탈 메시, 무브먼트 컴포넌트를 포함
### 주요 기능
* 점프 : Jump, StopJumping
* 걷기, 달리기 : AddMovementInput
* 중력, 공중 제어 : UCharacterMovementComponent
* 애니메이션 연동 : USkeletalMeshComponent과 UAnimInstance와 쉽게 연결하여 애니메이션 처리
* 네트워크 복제(Replicate) 지원

## 객체 간 차이, AActor와 APawn, ACharacter의 차이점
* AActor
    * UObject를 상속해서 레벨에 배치할 수 있는 객체의 기반 클래스
    * Transform, Tick, 라이프 사이클, 네트워크 리플리케이션 지원
    * 컴포넌트 부착으로 기능 확장
    * SpawnActor로 인스턴스화 가능
* APawn
    * AActor를 상속한 클래스
    * PlayerController 또는 AIController에 의해 Possess(빙의)되어 조종할 수 있다.
    * SetupPlayerInputComponent()로 입력을 바인딩할 수 있다
    * 기본 이동 컴포넌트가 없어, 별도로 부착해줘야한다.
* ACharacter
    * APawn을 상속한 클래스
    * 기본적으로 몇가지 컴포넌트들이 부착되어 있다.
        * UCapsuleComponent : 충돌체 용도이며 루트 컴포넌트로 설정되어 있다.
        * UCharacterMovementComponent : 이동 로직이 구현되어 있으며, 네트워크 동기화를 지원한다.
        * USkeletalMeshComponent : 스켈레탈 메쉬 표현과 애니메이션을 기능을 지원한다.
    * 이를 통해 지상형 캐릭터를 만드는 기본 틀을 제공한다
* 선택 기준
    * 조종이 불필요하고 단순 레벨에 배치 용도 -> AActor
    * 조종해야하고, 지상 이동 외의 방식으로 이동 -> APawn
    * 지상 이동하는 캐릭터 -> ACharacter

# Component
* 컴포넌트 기반 설계
    * 객체를 확장하는 방법 중 하나로 객체에 부품(컴포넌트)을 조합하여 기능을 확장하는 방법입니다.
    * 상속의 경우, 컴파일 타임에 클래스 구조가 고정되고, 기능 추가 시 새로운 파생 클래스를 만들어야 하기에 런타임 중에 기능을 확장하기 어렵습니다.
    * 런타임에 동적으로 컴포넌트를 추가, 제거 가능하고, 기능 단위로 재사용할 수 있습니다.
## 주요 컴포넌트
- CapsuleComponent (그외 ShapeComponent) : 콜리전 처리를 위해 사용
- CharacterMovementComponent : 캐릭터의 이동 로직 처리를 위해
- WidgetComponent : 3D 공간에 UI를 표시하기 위해
- NiagaraComponent : 나이아가라를 이용한 VFX 재생을 위해
- AudioComponent : 사운드 재생을 위해
- 그 외에도 콤보 액션 연계나 캐릭터의 스탯 기능을 위해 자체 액터 컴포넌트를 작성하기도 함
## ActorComponent vs SceneComponent
* ActorComponent
    * 액터에 장착시킬 수 있는 컴포넌트들의 기반 클래스
    * Transform 속성이 없어 3D 공간 상의 위치, 회전, 크기 개념이 없음
        * Attach 불가 -> 다른 컴포넌트에 붙일 수 없음
    * 순수 로직, 기능 처리에 적합    
* SceneComponent
    * 액터 컴포넌트를 상속하여 Transform 속성을 가진 클래스.
    * 다른 SceneComponent에 Attach 가능
        * 계층 구조 구성 가능
    * 액터의 RootComponent 역할을 할 수 있다.
* Transform, Attach 기능의 차이 여부가 핵심

# CDO : Class Default Object
* 클래스가 로드될 때 언리얼 엔진이 자동으로 생성하는 해당 클래스의 기본 인스턴스
* CDO 생성 시점
    * 엔진 초기화 시점 (게임 실행 전)
    * 각 UCLASS에 대해 1개씩 자동생성
    * 엔진 시작 시, 생성자 호출로 만들어지는 객체가 이것
## CDO 목적
* 해당 클래스의 기본 값 저장소 역할
* 새 인스턴스 생성 시 CDO의 값을 복사하여 초기화
* 에디터의 Details 패널 기본값이 곧 CDO 값
## 핵심 특징
* 클래스 당 1개만 존재 (싱글톤과 유사하지만 다름)
* 게임 로직 실행 대상이 아님 (BeginPlay 호출 안됨)
    * CDO를 통해서 특정 게임 로직 함수를 호출하는 것은 부적절
    * 기본값 읽기 용도로 사용
* 직접 수정 가능하지만 일반적으로 권장하지 않음
    * UPROPERTY() 매크로에서 CDO의 값에 대한 수정 범위를 열어둘 수 있음
        * EditAnywhere : CDO, Instance 어디서든 수정 가능
        * EditDefaultOnly : CDO 값(BP 기본값) 수정 가능
        * EditInstanceOnly : 배치된 인스턴스만 수정 가능
* GetClass()->GetDefaultObject()로 접근 가능
## 요약
* CDO는 클래스당 1개만 존재하는 기본 인스턴스로,<br>새 객체 생성 시 기본값의 원본 역할을 하며
에디터의 Details 패널 기본값이 CDO의 값입니다.