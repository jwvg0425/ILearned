# auto를 쓰자!

## auto의 장점들

 auto를 이용하는 게 명시적으로 타입을 서명해주는 것보다 나은 점이 굉장히 많다.

### 간결함

 auto를 쓰면 길고 복잡한 타입을 간단히 표기할 수 있으니 손가락도 덜 아프고 보기도 좋다. 특히 STL을 쓰다보면 iterator를 써야하는 경우가 굉장히 많은데 iterator 관련 타입들은 하나같이 타입 이름이 길어서 일일히 쓰다보면 지치기도 지치고 보기도 안 좋다. 이럴 때 auto를 이용하면 코드가 훨씬 깔끔해진다.

```C++
template<typename It>
void f(It a)
{
    typename std::iterator_traits<It>::value_type
        currValue = *b; //극혐
}
```

해당 반복자가 가리키는 값의 타입을 얻어 오기 위해선 위 코드처럼 굉장히 장황한 코드를 써야한다. 하지만 auto를 쓰면 저 긴 타입을 단순히 auto 라는 4글자만으로 대체할 수 있으니 훨씬 간결하고 보기가 좋다.

### 초기화

auto를 쓰려면 해당 변수의 값을 무조건 초기화시켜줘야한다. 따라서 초기화 되지 않은 변수로 인한 에러들이 발생할 위험을 없앨 수 있다.

```C++
int i; //초기화가 안 됨. 지역 변수일 경우 문제를 일으킬 수 있다.
auto x; //초기화가 없어서 타입 추론 불가! 컴파일 에러 발생.
auto x = 0; // x 초기화도 됐고, 타입도 자동으로 추론됨. 굿굿.
```

### 명시하기 까다로운 타입 대체

Closure와 같이 타입 자체가 명시적으로 표현하는 게 까다로운 경우에도 auto를 쓰면 편하게 타입을 지정해줄 수 있다.

```C++

//C++11. auto로 람다의 타입을 지정할 필요없이 람다 저장하는 변수 선언 가능.
auto derefUPLess =
  [](const std::unique_ptr<Widget>& p1,
     const std::unique_ptr<Widget>& p2)
   { return *p1 < *p2; };

//C+++14에서는 람다의 매개변수 타입도 auto로 선언 가능.
auto defrefLess =
  [](const auto& p1, const auto& p2)
   { return *p1 < *p2; };
```

물론 굳이 auto를 쓰지 않더라도 std::function을 이용해도 Closure를 저장하는 타입의 변수를 선언할 수 있다. std::function은 C++11 표준 라이브러리에 있는 '호출 가능한 오브젝트'를 일반화한 템플릿이다. 그래서 함수 포인터, 람다 등등 함수처럼 동작하는 애들은 모두 std::function 타입의 변수에 저장할 수 있다. 하지만 std::function을 쓸 때는 그 개체가 어떤 함수를 저장할지 명시를 해줘야 한다. 예를 들어 위 코드의 derefUPLess의 타입을 명시해주려면 아래와 같이 써야한다.

```C++
std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)>
  derefUPLess = [](const std::unique_ptr<Widget>& p1,
                   const std::unique_ptr<Widget>& p2)
                   { return *p1 < *p2; };
```

auto가 아니라 std::function를 쓰는게 auto에 비해 타입 길이도 너무 길 뿐 아니라 속도나 메모리 면에서도 손해다. auto로 썼을 경우에는 정확히 Closure를 쓰는 만큼의 메모리를 차지하지만, std::function을 쓰는 경우에는 그렇지 않다. std::function은 모든 종류의 호출 가능한 오브젝트에 대해 제대로 동작해야하기 때문에 아무래도 더 느리고 덩치도 더 클 수 밖에 없다. 그래서 어떤 면으로 보나 auto를 쓰는 게 나으니 auto를 쓰자.

### 이식성 향상

auto를 쓰면 이식성도 좋아진다. 우리가 일반적으로 아무 생각없이 쓰는 다음 코드를 한 번 예로 들어 보자.

```C++
std::vector<int> v;

unsigned size = v.size();
```

STL 벡터 컨테이너의 size를 보통 unsigned라고 생각해서 아무 생각없이 unsigned형 변수에 저장을 받는다. 하지만 실제로 벡터 컨테이너의 size가 리턴하는 타입은 32비트 운영체제냐 64비트 운영체제냐에 따라 다르게 정의되어 있다. 이걸 unsigned로 받아버리면 32비트에서는 제대로 동작할지 몰라도 64비트에서는 타입이 달라져서 문제를 일으킬 소지가 있다. 이걸 unsigned로 받는게 아니라 auto로 받는다면 대입되는 값의 타입에 따라 자동으로 타입이 바뀌므로 따로 코드를 수정할 필요가 없다. 그래서 32비트, 64비트 혹은 운영체제 등 환경에 따라 타입이 달라질 가능성이 있는 곳에 auto를 쓰면 따로 코드를 수정하지 않아도 되므로 높은 이식성을 확보할 수 있다.

### 프로그래머보다 똑똑한 auto

std::unordered_map을 쓰는 경우를 생각해보자. 이 STL 컨테이너는 Hash Table을 구현한 컨테이너인데, key의 타입과 value의 타입 두 가지를 지정해줘야 한다. 보통 std::unordered_map을 쓸 때는 다음과 같이 선언해서 사용한다.

```C++
std::unordered_map<std::string, int> hash;
```

key의 타입을 `std::string, value`의 타입을 int로 지정한 것이다. 여기서 주의해야할 점이 있다. hash의 각 요소들은 key와 value의 쌍이므로 `std::pair`의 형태로 저장되는데, 이 때 이 `std::pair`의 타입은 `std::pair<std::string, int>`가 아니다. 선언할 때 key의 타입을 std::string으로 했기 때문에 저런 형태가 될 거라고 생각하지만 실제로는 `std::pair<const std::string, int>`이다. key는 수정될 수 없는 값이기 때문이다. 따라서 hash의 각 원소를 순회할 때 직접 타입을 명시하면 자신도 모르게 실수할 수 있다. 하지만 auto는 코딩하는 프로그래머보다 똑똑하다. 쉽게 실수할 수 있는 부분에서 알아서 정확한 타입을 추론해서 적용시켜주므로 auto를 쓰면 저런 실수를 할 일은 없다.

```C++
//직접 타입을 명시하다 보면 이렇게 실수하기 쉽다.
for(const std::pair<std::string, int>& p : m)
{
   //doSomething
}

//하지만 auto를 쓰면 타입을 알아서
// std::pair<const std::string, int>로 정확하게 추론해준다.
for(const auto& p : m)
{
    //doSomething
}
```
> 하지만 auto가 장점만 있고 쓰기 편한 만능의 도구인 것은 아니다. auto도 쓸 때 조심해야할 부분들이 몇 가지 있다. [item 1,2](item_01_02.md)에서 item2와 [item 6](item_06.md)을 참조.