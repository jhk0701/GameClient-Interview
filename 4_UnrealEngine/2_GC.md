# Unreal Engine GC
## 언리얼 GC의 동작 방식
* C++에선 지원하지 않는 GC를 언리얼 엔진에서 구현해서 사용
    * 언리얼 엔진의 GC는 UObject를 상속한 객체에 대해서 작동
    * GC 방식 Mark And Sweep
        * Mark 단계 : Root에서 출발해 참조 가능한 객체를 마킹
        * Sweep 단계 : 마킹되지 않은 객체(== 참조되지 않은 객체)를 제거
        * 즉, 살아있는 객체를 Mark하고, 마킹되지 않은 객체를 Sweep하는 방식
### 참조 여부 기준
1. UPROPERTY() - UObject*에 UPROPERTY() 매크로가 붙은 경우 참조된 것으로 판단
    ```cpp
    UPROPERTY()
    UMyObject* MyObj;
    ```
2. AddToRoot - 루트 셋에 직접 등록
    ```cpp
    MyObj->AddToRoot();
    // GC에 의해 절대 수거되지 않음
    // 더 이상 필요 없을 땐, RemoveFromRoot() 필수 호출
    ```
3. FGCObject 상속 - UObject 아닌 클래스에서 참조 유지
    ```cpp
    class FMyManager : public FGCObject
    {
        virtual void AddReferencedObjects(FReferenceCollector& Collector) override
        {
            Collector.AddReferencedObject(MyObj); // GC에 참조 알림
        }
    };
    ```
## GC 실행 주기
* 언리얼 엔진의 GC는 매 프레임마다 실행되진 않음
* 일정 시간 간격으로 실행 : 기본값 60초, 설정 변경 가능
* 수동 호출 가능 : ForceGarbageCollection()
## UPROPERTY 없이 UObject* 사용 시 위험성
* UPROPERTY 없이 UObject*를 사용하면, GC가 다음 사이클에서 해당 객체의 참조가 없다고 판단하고 수거해감
* 해당 객체는 댕글링 포인터가 됨
* 그러므로 UPROPERTY()를 붙여 GC가 참조를 추적하게하여 수거되지 않도록 하는 것이 중요
## TWeakObjectPtr
* GC와 함께 안전하게 약참조
```cpp
// GC 수거를 막지 않으면서 참조를 유지하고 싶을 때
TWeakObjectPtr<UMyObject> WeakRef = MyObj;

// 사용 전 유효성 반드시 체크
if (WeakRef.IsValid())
{
    WeakRef->DoSomething();
}
// GC가 수거하면 자동으로 nullptr 처리됨
// 일반 포인터와 달리 댕글링 포인터 위험 없음
```