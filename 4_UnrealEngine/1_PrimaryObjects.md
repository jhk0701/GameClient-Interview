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