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
### UCLASS
* c++ 클래스를 언리얼 엔진의 리플렉션 시스템에 등록시키는 매크로
    * 언리얼 엔진이 해당 클래스를 인식하고 관리할 수 있게 됨