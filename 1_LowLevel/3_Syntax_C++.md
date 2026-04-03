# C++ 문법
## 1. 연산자 오버라이드는 왜 사용하는가
* int, float 같은 원시 자료형들과 같이 커스텀 타입에 대해서도 사칙연산, 대입과 같은 연산자를 사용하기 위함.
* 즉, 커스텀 타입에 대해서도 코드의 일관성 있도록 사용하기 위해서 연산자 오버라이드를 사용.
* 추가로 비교 연산자같이 특정한 연산자들을 오버라이드함으로써 STL과 알고리즘 라이브러리에 응용해서 사용할 수 있음.

## 2. L-Value vs R-Value
* L-Value : 다시 호출할 수 있는 메모리
    * 일반적으로 선언한 변수들
* R-Value : 한번만 호출할 수 있는 일회성 메모리
    * 일반적으로 리터럴 상수들
    * 단, 리터럴 중에서도 문자열 리터럴은 L-Value임
        * ex) `string str = L"Hello";`
* 문자열 리터럴이 L-Value인 이유
    * 문자열은 배열 형태의 데이터를 식별할 수 있는 메모리 위치를 가지므로 L-Value로 분류됨
        * 문자열 리터럴은 프로그램의 데이터 영역(주로 ROM)에 저장됨
        * 식별자(주소)가 있어서 `&"Hello"`와 같이 주소 연산자를 쓸 수 있음
    * 배열로의 붕괴(Decay)
        * `const char*` 타입으로 표현될 때, 문자열 리터럴은 `const char[N]` 타입의 L-Value에서 `const char*` 로 암시적 형변환된다.
        * 배열이기 때문에 시작 주소를 통해서 `const char*`로 암시적 형변환되는 것

## 3. Move Semantics : 이동 의미론
* std::move() : L-Value를 R-Value로 변환해서 반환시켜준다.
* 반환시칸 R-Value를 변수가 대입받으면 기존 변수의 R-Value의 소유권은 사라진다.

### std::move 역할
* L-Value를 R-Value를 바꿔주는 역할. <br> 실제로 이동을 해주는 함수가 아님.
* 이를 통해서 이동 연산자가 정의되어 있는 경우 변경된 R-Value로 오버로딩 될 수 있음
```cpp
template<typename T>
typename remove_reference<T>::type&& move(T&& param)
{
	using ReturnType = typename remove_reference<T>::type&&;
	
	return static_cast<ReturnType>(param);
}

// C++ 14 이후
template<typename T>
decltype(auto) move(T&& param)
{
	using ReturnType = remove_reference_t<T>&&;

	return static_cast<ReturnType>(param);
}

std::vector<std::string> v;
std::string str = "example";
v.push_back(std::move(str));  // str 은 이제 껍데기만 남음
str.back();                   // 정의되지 않은 작업!
str.clear();                  // 다만 clear 자체는 가능하다.
```
* 때문에 실제로 하는 일은 move가 아니라 R-Value Cast임

### std::forward 역할
* std::forward()의 경우 move와 다르게 R-Value일 때는 R-Value로 전달, L-Value일때는 L-Value로 전달하는 역할
* 원래 c++에서는 참조에 대한 참조는 안되지만, 템플릿 인스턴스화에선 참조에 대한 참조가 허용됨.
    * `template` 메서드를 사용할 때, R-Value 레퍼런스로 매개변수로 넘기더라도 L-Value로 참조됨.
* C++에서는 연역 과정에서 둘 중 하나라도 L-Value 참조라면 결과는 L-Value 참조가 된다는 규칙이 있어 forward를 통해서 L-Value, R-Value를 구분할 수 있음.
```cpp
Some& && forward(typename remove_reference<Some&>::type& param)
{	
    return static_cast<Some& &&>(param);
}
```
* Some&& 의 경우에는 T가 Some로 연역됨

### move vs forward
* move : 이동 준비가 목적 (L-Value -> R-Value 변환)
* forward : 전달일 목적 (L-Value는 L-Value로, R-Value는 R-Value로)

## 4. Cast
* 캐스팅 (형변환)은 명시적 형변환과 암시적(묵시적) 형변환으로 나뉨
### 암시적 형변환
* 다른 표기 없이 호환되는 타입들 간 변환할 수 있게 해줌
    * 데이터 손실이 발생할 수 있음
    * 호환되는 타입을 사전에 구현해야함.
    * 유지보수 측면에서도 알아내기 어렵다는 단점이 있음.

### 명시적 형변환
* 형변환 시, 변환할 타입을 명시해서 변환하는 것
```cpp
// C 스타일
float f = 1.1f;
int a = (int)f;
```
* C++에서는 여러 캐스팅 연산자들을 사용함
    * C 스타일보다 가독성이 높고 위험성을 줄여줌
### `static_cast<T>`
* 기존 C 스타일 형변환 대신 사용
* 형변환이 가능한지에 대한 검사는 컴파일 타임에서 이루어짐
* 단, 다운캐스팅(상속 관계에서 상위->하위 클래스)의 경우 타입 안정성을 보장하지 않음
    * 실제 객체가 변환하려는 타입이 아닌 경우에도 컴파일러는 오류를 발생시키지 않음

```cpp
float f = 1.1f;
int a = static_cast<int>(f);
```
### `dynamic_cast<T>`
* 런타임에 타입 안정성을 검사 (RTTI)
    * 주로 다형성을 가진 클래스 (가상 함수를 가진 클래스)의 포인터나 참조 간의 변환에 사용
    * 가상 테이블을 이용해서 캐스팅이 가능한지 검사
* 다운캐스팅 시, 실제 객체 타입이 목표 타입과 일치하는지 확인하고, 일치하지 않으면 nullptr을 반환
    * 이를 통해 타입 안정성을 제공
* 런타임에 타입 안정성 검사를 진행하고 상속 관계에서 업캐스트나 변환을 지원해주지만 다른 데이터형 변환은 허용하지 않는다.
* 업캐스팅만이 가능하고 만약 가상함수까지 사용한다면 다운캐스팅 또한 지원해줌.

### 강제 캐스팅을 사용하는 것보다 static_cast, dynamic_cast를 사용할 때의 이점
* 컴파일러가 오류 체크를 진행해줌
    * 강제 캐스팅을 사용하면 런타임시 seg fault, runtime error 등 예기치 못한 에러를 발생시킬 수 있는데 컴파일러단에서 오류를 잡아 추후 문제가 될 가능성을 줄여주는 이점을 가지고 있다.
* 상속관계에 있어도 형변환이 가능
    * 다운캐스팅에서 형변환은 안전하지 못하기 때문에 dynamic_cast를 이용한 방법은 안정성을 보장함

### `const_cast<T>`
* const나 volatile 를 제거, 부여하는데 사용
    * 다만 부여의 경우에는 사용하는 경우가 잘 없음
* 주로 상수 객체에 대한 수정이 필요할 때 사용
* 타입 자체 변환보다는 객체의 속성을 변경하는데 초점을 맞춤
```cpp
void NonConstFunc(Some s);

//...
const Some s;
NonConstFunc(s);   // ERROR!
NonConstFunc(const_cast<Some>(s))
```
### `reinterpret_cast<T>`
* 모든 포인터 유형을 다른 포인터 방식으로 변경할 수 있음.
    * 또한 포인터 타입을 정수로 바꾸는데도 가능한 캐스팅
* 비트 단위로 메모리를 분해, 재조립하는 강력한 형변환을 하기 때문에 이해도가 필요


## 전역 변수 vs static
* 데이터 영역에 저장된다는 공통점
* 접근 범위의 차이
    * 전역변수는 특정 클래스나 네임스페이스에 국한되지 않고 어디서든 접근 가능
    * static은 대체로 객체 내부에서 선언하여, 객체를 생성하지 않고도 접근 가능
        * 단, static의 경우 선언된 객체의 범위지정연산자(::)를 통해서 접근해야 함.
        * 이를 통해서, 선언된 객체 내 파일 범위나 블록 범위 내로 제한되어 프로젝트 내에서 이름 충돌이 방지됨

## 함수 포인터와 람다
* 함수 포인터 : 선언된 함수의 포인터(메모리 주소)를 담을 수 있는 변수
* 람다 : 일회성 함수
    * 캡쳐절을 통해서 람다식 생성 당시에 주변 데이터를 담을 수 있다.
    * 캡쳐절이 없다면 매개변수를 이용하는 등의 작업을 수행하는 번거로움이 있음