# Unreal Engine 5 대표 기능
## 나나이트
## 루멘
## 나이아가라

## 리프렉션 시스템
* 프로그램이 런타임에 자기 자신의 구조(클래스, 변수, 함수 등)를 조회하고 조작할 수 있는 메타 프로그래밍 메커니즘
    * 즉, 코드가 자지 가신을 거울처럼 들여다보는 기능
* C++은 기본적으로 리플렉션을 지원하지 않는다.
    * 그래서 언리얼 엔진은 자체 리플렉션 시스템을 구축
### 리플렉션의 역할
1. 타입 정보 조회 : 런타임에 클래스,변수,함수 정보 접근 가능
2. 에디터 연동 : 디테일 패널 자동 노출
3. 직렬화 : 저장, 로디, 네트워크 복제 자동화
4. 스크립팅 연동 : BP <-> C++ 간 통신

### 리플렉션 동작 원리
1. 매크로로 메타데이터 표시
    * UCLASS(), UPROPERTY(), UFUNCTION() 등 매크로를 붙여 리플렉션 대상으로 등록
    * class 내부 상단에 GENERATED_BODY() 매크로에서 리플렉션을 지원하는 코드가 들어감
2. UHT (Unreal Header Tool) 코드 생성
    * 빌드 과정에서 .h 파일에서 UHT가 매크로를 분석 -> .generated.h 자동 생성
        * .generated.h : 클래스 메타데이터 등록 코드, 변수 오프셋 정보, 함수 포인터 테이블 포함
3. 런타임에 UClass 조회 등등 지원
    * USomething::StaticClass();

### 리플렉션 덕에 가능한 것들
1. 에디터 디테일 패널 자동화
2. 직렬화 (저장/로드) 자동화
3. 네트워크 복제 자동화
4. BP <-> C++ 통신
5. 런타임 타입 조회 (RTTI 대체)
```cpp
// 문자열로 클래스 찾기
UClass* FoundClass = FindObject<UClass>(
    ANY_PACKAGE, TEXT("AMyCharacter"));

// 해당 클래스의 Actor 스폰
AActor* NewActor = GetWorld()->SpawnActor<AActor>(FoundClass);

// 특정 클래스인지 런타임 확인
if (SomeActor->IsA(AMyCharacter::StaticClass()))
{
    // AMyCharacter임이 확인됨
}

// 모든 프로퍼티 순회
for (TFieldIterator<FProperty> It(AMyCharacter::StaticClass()); It; ++It)
{
    FProperty* Prop = *It;
    UE_LOG(LogTemp, Log, TEXT("프로퍼티 이름: %s"), *Prop->GetName());
}
```
### 리플렉션 구조
```
UObject 계층과 리플렉션 메타 객체:

UField
├── UStruct
│   ├── UClass      → 클래스 메타정보 (UCLASS)
│   │   └── AMyCharacter::StaticClass()
│   ├── UScriptStruct → 구조체 메타정보 (USTRUCT)
│   └── UFunction   → 함수 메타정보 (UFUNCTION)
└── FProperty       → 변수 메타정보 (UPROPERTY)
    ├── FFloatProperty
    ├── FIntProperty
    ├── FObjectProperty
    └── FArrayProperty ...
```
```c++
// UClass 내부에 저장된 정보
UClass* MyClass = AMyCharacter::StaticClass();

MyClass->GetName();           // 클래스 이름
MyClass->GetSuperClass();     // 부모 클래스
MyClass->Children;            // 자식 필드 목록
MyClass->ClassFlags;          // 클래스 플래그 (Abstract 등)
MyClass->GetDefaultObject();  // CDO (Class Default Object)
                              //  → 에디터의 기본값이 저장된 인스턴스
```
### 리플렉션의 비용
* 리플렉션을 통한 FindPropertyByName 호출을 매 프레임 단위로 하는 것은 비용이 매우큼
    * Find 계열은 전체를 순회하는 방식으로 동작하기 때문
    * 그러므로 써야한다면 되도록 캐싱해서 한번만 호출하도록 하는 것을 권장
```c++
// ❌ 핫 루프에서 리플렉션 사용 — 느림
void Update(float DeltaTime)
{
    for (AActor* Actor : Actors)
    {
        FProperty* Prop = Actor->GetClass()
                               ->FindPropertyByName("Health"); // 매 프레임 조회
        float* Val = Prop->ContainerPtrToValuePtr<float>(Actor);
        *Val -= DeltaTime;
    }
}

// ✅ 캐싱 후 사용 — 빠름
FProperty* CachedHealthProp = nullptr;

void BeginPlay()
{
    CachedHealthProp = GetClass()->FindPropertyByName("Health"); // 1회만
}

void Update(float DeltaTime)
{
    float* Val = CachedHealthProp->ContainerPtrToValuePtr<float>(this);
    *Val -= DeltaTime;
}
```
### 리플렉션 관련 주의사항
* UHT 의존성 : 매크로가 없으면 리플렉션 대상이 아님
* UObject 필수 : UObject 상속 클래스만 완전한 리플렉션 지원
* 빌드 시간 : UHT 코드 생성으로 빌드 시간 증가
* 런타임 조회 비용 : FindPropertyByName 등은 캐싱 필수
* 포인터 직접 접근 : 리플렉션 우회 시, GC와 충돌 위험

## 매크로
* 매크로들은 모두 언리얼 헤더 툴(UHT)을 통해 리플렉션 시스템에 등록시키키 위한 도구이며, 
GC, BP, 에디터, 네트워크 기능과 연동된다.

### UCLASS
* c++ 클래스를 언리얼 엔진의 리플렉션 시스템에 등록시키는 매크로
    * 언리얼 엔진이 해당 클래스를 인식하고 관리할 수 있게됨
    * GC 대상이 됨
    * BP에서 상속 및 사용이 가능해짐
    * 클래스 내부에 반드시 GENERATED_BODY() 매크로를 선언해야 함
```c++
UCLASS(BlueprintType, Blueprintable)
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()  // 리플렉션 코드 자동 생성 위치

public:
    AMyCharacter();
};
```
#### 주요 지정자 Specifier
* Blueprintable : BP에서 이 클래스를 상속 가능
* BlueprintType : BP에서 변수 타입으로 사용 가능
* Abstract : 직접 인스턴스화 불가 (추상 클래스 표시)
* NotBlueprintable : BP 상속 불가
* MinimalAPI : 다른 모듈에서 타입만 참조 가능
### UPROPERTY
* 멤버 변수를 언리얼 엔진의 리플렉션 시스템에 등록시키는 매크로
    * GC 추적 대상이 됨 (UObject* 보호)
        * UObject 파생 클래스가 UPROPERTY() 매크로가 없다면 GC에 의해 정리됨
    * BP에서 읽기/쓰기 노출 가능
    * 에디터의 Details 패널에 노출 가능
    * 네트워크 리플리케이션 대상으로 지정 가능
    * 직렬화 대상으로 지정 가능 (Save/Load)
```c++
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

    // 에디터 노출 + 블루프린트 읽기/쓰기
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Stats")
    int32 HP;

    // 네트워크 리플리케이션 대상
    UPROPERTY(ReplicatedUsing = OnRep_HP)
    int32 ReplicatedHP;

    // GC 추적 — 이 매크로 없으면 댕글링 포인터 위험
    UPROPERTY()
    UStaticMeshComponent* MeshComponent;
};
```
#### 주요 지정자 Specifier
* EditAnywhere : 에디터 어디서든 수정 가능
* EditDefaultsOnly : 블루프린트 기본값에서만 수정 가능
* EditInstanceOnly : 배치된 인스턴스에서만 수정 가능
* VisibleAnywhere : 에디터에서 보이지만 수정 불가
* BlueprintReadWrite : 블루프린트에서 읽기/쓰기 가능
* BlueprintReadOnly : 블루프린트에서 읽기만 가능
* Replicated : 네트워크 리플리케이션 대상
* ReplicatedUsing=함수명 : 리플리케이션 시 콜백 함수 호출
* Transient : 직렬화 제외 (Save에 포함 안됨)
* SaveGame : SaveGame 직렬화 대상
### UFUNCTION
* 멤버 함수를 언리얼 엔진의 리플렉션 시스템에 등록시키는 매크로
    * BP에서 함수 호출을 가능하게 함
    * 네트워크 RPC 함수로 지정 가능
    * 이벤트로 사용 가능
    * 엔진 내부 시스템(타이머, Dynamic 델리게이트 등)과 연동 가능
 ```c++
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

    // 블루프린트에서 호출 가능
    UFUNCTION(BlueprintCallable, Category = "Action")
    void Attack();

    // 블루프린트에서 오버라이드 가능한 이벤트
    UFUNCTION(BlueprintImplementableEvent)
    void OnDead();

    // C++ 기본 구현 + 블루프린트 오버라이드 가능
    UFUNCTION(BlueprintNativeEvent)
    void OnHit();
    virtual void OnHit_Implementation(); // C++ 구현부

    // 서버 RPC
    UFUNCTION(Server, Reliable)
    void ServerAttack();
};
 ```
#### 주요 지정자 Specifier
* BlueprintCallable : 블루프린트에서 호출 가능
* BlueprintPure : 블루프린트에서 호출 가능 (객체 상태 변경 없음, const)
* BlueprintImplementableEvent : 블루프린트에서 구현, C++에서 호출
* BlueprintNativeEvent : C++ 기본 구현 + 블루프린트 오버라이드 가능
* Server : 서버에서 실행되는 RPC
* Client : 클라이언트에서 실행되는 RPC
* NetMulticast : 서버→모든 클라이언트 실행 RPC
* Reliable : 패킷 손실 시 재전송 보장
* Unreliable : 재전송 보장 없음 (빠름)
* CallInEditor : 에디터에서 버튼으로 직접 호출 가능
### USTRUCT
* C++ 구조체를 언리얼 엔진의 리플렉션 시스템에 등록시키는 매크로
    * UCLASS와 달리 GC 대상이 아님 (UObject 미상속이기 때문)
    * BP에서 구조체 변수로 사용 가능
    * 데이터 묶음 전달에 주로 사용
    * 내부 변수는 UPROPERTY로 등록해야 BP에서 접근 가능
```cpp
USTRUCT(BlueprintType)
struct FPlayerStat
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    int32 HP;

    UPROPERTY(BlueprintReadWrite)
    float Speed;
};
```
### UENUM
* C++ 열거형을 언리얼 엔진의 리플렉션 시스템에 등록시키는 매크로
    * BP에서 Enum타입 변수로 사용 가능
    * `UMETA(DisplayName="...")` 으로 에디터 표시 이름 지정 가능
```cpp
UENUM(BlueprintType)
enum class EPlayerState : uint8
{
    Idle        UMETA(DisplayName = "대기"),
    Moving      UMETA(DisplayName = "이동"),
    Attacking   UMETA(DisplayName = "공격"),
    Dead        UMETA(DisplayName = "사망")
};

// 사용 시
UPROPERTY(BlueprintReadWrite)
EPlayerState CurrentState;
```
### GENERATED_BODY()
* UCLASS, USTRUCT 내부에 선언하는 매크로, 언리얼 헤더 툴(UHT)이 자동 생성하는 코드의 삽입 위치를 지정
    * 리플렉션에 필요한 코드를 자동으로 생성
    * UCLASS, USTRUCT 선언 시 반드시 포함해야함
    * .generated.h 파일과 함께 동작
```cpp
// 빌드 시 UHT가 아래 내용들을 자동 생성
GENERATED_BODY()
// ↓ 내부적으로 생성되는 것들
// - StaticClass()
// - GetClass()
// - 리플렉션 메타데이터
// - 직렬화 코드
```

# Soft Reference & Hard Reference
* Hard Reference
    * 다른 UObject를 직접 참조하고 있는 것을 의미합니다.
    * BP 또는 DataAsset, DataTable 등등에서 다른 객체를 TObjectPtr<>, 다른 클래스를 TSubclassOf<>로 참조하는 경우 등에 해당합니다.
    * 여러 객체를 참조하는 객체를 로드할 경우, 연결된 다른 객체들까지 한번에 로드됩니다.
    * 참조 객체를 바로 사용해야하는 경우 편리합니다.
    * 하지만 하드 레퍼런스의 갯수가 많아지면 로드 시 부하가 있을 수 있습니다.
    * 언제 쓰는가
        * 게임 시작 시, 반드시 필요한 리소스인 경우
        * 항상 메모리에 올려두어야 하는 핵심 에셋
* Soft Reference
    * 하드 레퍼런스와 달리, 다른 객체를 직접 참조하는 것이 아니라 해당 객체의 경로를 갖습니다.
    * 그러므로 소프트 레퍼런스하는 객체를 사용하기 위해서는 런타임 중 수동으로 로드하는 과정이 필요합니다.
    * `LoadSynchronous`를 통한 동기 방식 또는 `UAssetManager`의 `FStreamableManager`를 통해 `RequestAsyncLoad` 비동기 방식으로 로드할 수 있습니다.
        * 단, 동기 방식의 경우 CPU의 메인 스레드에서 로드하기 때문에 블로킹이 발생하므로, 새로운 스레드를 통해 로드하는 비동기 방식을 더 권장됩니다.
    * 하드 레퍼런스에서 발생할 수 있는 로드 과부하를 방지할 수 있습니다.
    * 불필요한 리소스를 사전에 로드하는 것을 방지할 수 있습니다.
    * 언제 쓰는가
        * DLC / 대용량 에셋과 같이 필요 시에만 로드해야하는 에셋
        * 오픈월드에서 특정 구역 진입 시 로드하는 경우
        * DataTable에서 다양한 캐릭터 메시를 관리할 때 등등