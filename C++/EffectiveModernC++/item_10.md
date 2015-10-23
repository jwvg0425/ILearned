#scoped enum을 쓰자

C++ 98의 enum은 범위 제한이 없다. 한 마디로 말해 아래와 같은 상황이 벌어진다는 것이다.

```C++
enum Color { BLACK, WHITE, RED };

auto WHITE = false; // WHITE가 이미 선언되었다!
```

분명 WHITE는 Color라는 enum 내부의 값 중 하나임에도 불구하고 C++98의 enum은 범위에 제한이 없어 모든 이름 공간에서 선언이 되어버린다. 그래서 C++ 11에서는 scoped enum을 지원한다.

```C++
enum class Color { BLACK, WHITE, RED};

auto WHITE = false; //가능함

Color c = WHITE; // 불가능!
Color c = Color::WHITE; // 가능. 범위를 명시해줘야함.
auto d = Color::WITHE; // 이것도 가능.
```

## 장점들

 일단 전역 이름 공간에 선언되어 버린다는 기존 enum과는 달리 scope가 제한되어 이름 공간을 덜 더럽히게 된다는 장점은 위에서 설명했으니 생략.

### 타입 체킹이 엄격하다

기존의 enum(unscoped enum)은 정수 계통 타입과 암시적으로 변환이 된다.

```C++
enum Color { BLACK, WHITE, RED };

std::vector<std::size_t>
  primeFactors(std::size_t x);

Color c = RED; 

if ( c < 14.5 ) // 색깔과 실수를 비교??
{
    auto factors = primeFactors(c); //색깔의 prime factor를 계산??
}
```

 unscoped enum은 이런 암시적 변환 때문에 논리적으로 말이 안되는 구문들이 허용된다는 문제점이 있다. 반면에 scoped enum은 타입 체킹이 엄격해서 이런식의 암시적 변환을 허용하지 않는다.

```C++
enum class Color { BLACK, WHITE, RED };

Color c = Color::RED;

if ( c < 14.5 )  // error 발생!
{
    auto factors = primeFactors(c); // error 발생!
}
```

그래서 scoped enum을 쓸 때 이런 류의 변환을 하고 싶다면 static_cast를 써서 명시적으로 캐스팅을 해 줘야 한다.

```C++

if( static_cast<double>(c) < 14.5 )
{
    auto factors = primeFactors(static_cast<std::size_t>(c));
}
```

 얼핏 생각하면 괜히 까다로워지는 것 같지만 타입 체킹이 엄격할 수록 프로그램의 잠재적인 논리적 오류들을 컴파일 타임에 좀 더 많이 잡아낼 수 있게 되기 때문에 분명한 장점이라고 할 수 있다.

###전방 선언 가능

 scoped enum은 전방 선언이 가능하다! C++ 특유의 굉장히 느린 컴파일 속도에 골머리를 썩여 본 적 있는 사람들이라면 전방 선언이 갖는 메리트에 대해 아주 잘 알고 있을 것이다.

```C++
enum Color; //전방선언 불가!

enum class Color; // 전방선언 가능!
```

사실 C++11에서는 unscoped enum도 전방 선언이 가능하다. 다만 unscoped enum은 약간의 추가적인 작업이 필요하다. 왜냐하면, C++의 모든 enum은 내부적으로 컴파일러가 결정하는 숨겨진 타입(underlying type)이 있기 때문이다.  예를 들어서 ```enum Color { BLACK, WHITE, RED }; ```라는 enum이 있다고 하자. 컴파일러는 내부적으로 이 enum의 숨겨진 타입을 char로 설정할 것이다. 1바이트면 저걸 표현하는데 충분하니까.

```C++
enum Status 
{
    GOOD = 0,
    FAILED = 1,
    INCOMPLETE = 100,
    CORRUPT = 200,
    INDETERMINATE = 0xFFFFFFFF
};
```

반면 enum에 위 코드처럼 char 형으로는 충분하지 않을 만큼 넓은 범위의 값이 들어간다면 컴파일러는 이 타입을 표현할 수 있을만큼 충분히 큰 정수형의 타입으로 설정할 것이다.

 이 숨겨진 타입이 C++ 98에서 scoped enum을 전방선언할 수 없는 이유다. 컴파일러는 최적화를 위해 enum의 숨겨진 타입을 결정해야만 하는데, 전방선언된 것만 가지고는 해당 enum의 숨겨진 타입이 뭔지를 알 수가 없으니 에러가 나는 것이다. 이런 전방 선언이 불가능하다는 문제점 때문에, 해당 enum의 값이 조금만 바뀌어도 그 enum을 쓰는 모든 파일을 다 다시 컴파일해야 하는 사태가 벌어져 컴파일하는데 걸리는 시간이 굉장히 늘어나버린다.

 하지만 해당 enum을 쓰기 전에 컴파일러가 그 enum의 숨겨진 실제 타입이 뭔지를 파악할 수만 있다면 이런 일은 벌어지지 않을 것이다. C++11의 scoped enum은 숨겨진 타입이 뭔지 항상 알려져 있다.

```C++
enum class Status; //기본적으로 int
```

scoped enum의 경우 기본적으로 int 타입으로 다뤄진다. 이게 마음에 안든다면 직접 타입을 명시해줄 수도 있다.

```C++
enum class Status : std::uint32_t; // underlying type은 std::uint32_t가 됨
```

 C++11에서는 unscoped enum에 대해서도 직접 타입을 지정해줄 수 있다. 그리고 이렇게 할 경우 scoped enum도 전방 선언이 가능해진다(컴파일러가 enum을 쓰기 전에 해당 enum의 숨겨진 실제 타입이 뭔지 알 수 있게 되므로).

```C++
enum Color : std::uint8_t; //unscoped enum을 전방선언!
```

타입 설정은 전방 선언할 때 뿐만이 아니라 enum 정의할 때에도 사용 가능하다(당연한 거지만).

```C++
enum class Status : std::uint32_t
{
    GOOD = 0,
    FAILED = 1,
    ...
}
```

###unscoped enum이 더 유용한 경우

타입 체킹이 엄격한게 scoped enum의 장점이라고 했지만, 가끔씩 이게 코딩할 때 굉장한 짜증을 유발하는 경우가 있다. 바로 아래와 같은 경우.

```C++
//사용자 정보를 저장하는 타입
using UserInfo =
   std::tuple<std::string,  //이름
              std::string,  //email
              std::size_t>; //평판(reputation)

UserInfo uInfo;

enum Fields { uiName, uiEmail, uiReputation };
enum class CFields { name, email, reputation };

// 암시적 변환에 의해 값을 쉽게 가져옴
auto value1 = std::get<uiEmail>(uInfo); 

// 명시적으로 변환해줘야 함. 끔찍하다.
auto value2 = 
    std::get<static_cast<std::size_t>(cFields::email)>(uInfo);
```

enum의 경우 이런식으로 사용하는 일이 많은데 scoped enum을 이용하면 일일히 static_cast를 해줘야하니 굉장히 끔찍해진다.

이 때, std::get이 템플릿이라는 것에 주의해보자. 템플릿의 인자(<>안에 들어가는 녀석들) **컴파일 타임동안에** 결정되어야만 한다. 템플릿의 특성상 컴파일 타임에 코드가 구체화되어야하는데, 이렇게 구체화를 하기 위해선 당연히 코드를 구체화하기 위한 인자들을 모두 알고 있어야할 것이다.

 즉, ```static_cast<std::size_t>(cFields::email)```은 컴파일 타임동안에 계산되는 값이어야하기 때문에 constexpr 함수([item 15](item_15.md) 참조 - 간단히 말해 컴파일 타임에 결과값을 계산할 수 있음을 보장하는 함수)로 표현할 수 있을 것이다. 또 이 함수는 어떤 예외도 발생시키지 않으므로 noexcept([item 14](item_14.md) 참조) 설정도 해주어야할 것이다.

```C++
//C++11 스타일.
//std::underlying_type<E>::type은 underlying type을 알려주는 type traits
template<typename E>
constexpr typename std::underlying_type<E>::type
    toUType(E enumerator) noexcept
{
    return static_cast<typename std::underlying_type<E>::type>(enumerator);
}

//C++14 스타일
template <typename E>
constexpr auto toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```

이 함수 toUType을 이용하면 컴파일 타임에 해당 열거형의 값을 그 열거형의 숨겨진 실제 타입으로 변환한 값을 반환해준다. 이걸 템플릿의 인자로 받아 넣으면 조금은 더 편하게 쓸 수 있다. 이건 모든 열거형에 대해 일반화된 함수기 때문에(리턴 타입까지 일반화돼서 각 열거형에 맞춰 다른 리턴 타입을 가진다) 이대로 복사해서 가져다써도 된다.

```C++
auto val = std::get<toUType(cFields::email)>(uInfo);
```

여전히 몇글자 더 써야하긴 하지만 어쨌든 static_cast를 쓰는 것보다는 편해졌다. 물론 이렇게 해도 함수를 써야만하니 unscoped enum에 비해 조금 불편하긴 하다. 그래도 위에서 언급한 scoped enum의 많은 장점들을 생각해보면 약간의 불편함을 감수하고서라도 scoped enum을 쓰는 편이 더 나을 것이다.