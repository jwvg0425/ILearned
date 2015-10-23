# 추론된 타입 보는 법

요즘 IDE는 워낙 성능이 좋아 auto, template등에서 타입이 어떻게 추론됐는지 궁금하다면 그냥 그 위에 마우스만 갖다대도 알려준다(visual studio).

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/good_ide.png)

하지만 위 스크린샷에서도 볼 수 있듯이 std::vector<int>가 `std::vector<int, std::allocator<int> >`라고 뜬다. 일반적으로 잘 쓰지 않는 형태의 타입으로 보여줄 때가 많기 때문에 어떤 타입의 경우 길이가 너무 길어져서 보기 힘들어진다.

그럴 땐 아래 몇 가지 방법을 활용해보자.

## 컴파일러 진단

의도적으로 컴파일 에러를 일으켜서 추론된 타입을 확인하는 방법이다.

```C++
//타입을 확인하기 위한 템플릿 클래스.
template<typename T>
class TD;

auto k = &v[3];
TD<decltype(k)> kType;
```

위 코드에서 k의 타입이 궁금해 마우스를 올렸다간 아래 사진처럼 심히 곤란한 상황에 빠지게 된다.

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/no-dap.png)

하지만 아래 `TD<decltype(k)> kType;` 과 같은 구문을 쓴 다음 컴파일을 하면 TD가 구현이 되어있지 않기 때문에 컴파일 에러와 맞닥뜨리게 되는데, 이 때 친절하게 컴파일러가 k의 타입을 가르쳐준다.

> error C2079: 'kType'은(는) 정의되지 않은 class `TD<int *>`'을(를) 사용합니다.


## 런타임 출력

typeid 연산자를 이용해서 런타임에 특정 오브젝트의 타입을 확인할 수 있다.

```C++
std::cout << typeid(x).name << std::endl;
std::cout << typeid(y).name << std::endl;
```

위와 같이 typeid(object).name을 통해 어떤 object의 타입 이름을 알 수 있고, 이를 화면에 출력함으로써 타입을 런타임에 확인할 수 있다. 컴파일러 별로 출력되는 내용에는 조금씩 차이가 있으니 이 부분은 자신이 쓰는 컴파일러에서 어떤 식으로 출력하는 지 알아보면서 확인하는 것이 좋다. visual studio의 경우 다음과 같은 방식으로 출력한다.

```C++
int i = 0;
const int a = 3;
int& ri = i;

typeid(i).name; // => int
typeid(a).name; // => int
typeid(ri).name; // => int

template<typename T>
void f(const T& param)
{
	std::cout << typeid(T).name() << std::endl;
	std::cout << typeid(param).name() << std::endl;
}
//f(i)호출하면
//int
//int
//라고 출력
```

뭔가 이상하다. 각자 다 타입이 다른데 싹 다 int로 출력된다. typeid가 갖고 있는 치명적인 문제점이 바로 이것인데, const, volatile, reference등의 지정자를 싸그리 무시해버린다. 따라서 typeid로는 정확한 타입을 얻을 수 없다.

## Boost.TypeIndex

boost 라이브러리를 이용하면 런타임에도 정확한 타입 정보를 얻을 수 있다. [boost](http://www.boost.org/) 라이브러리를 설치한 후 boost/type_index.hpp를 include하면 사용 가능하다.

```C++
#include <boost/type_index.hpp>
using boost::typeindex::type_id_with_cvr;

int i = 0;
const int a = 3;
int& ri = i;

//int 출력
std::cout << type_id_with_cvr<decltype(i)>().pretty_name() << std::endl;

//int const 출력
std::cout << type_id_with_cvr<decltype(a)>().pretty_name() << std::endl;

//int & 출력
std::cout << type_id_with_cvr<decltype(ri)>().pretty_name() << std::endl;

template<typename T>
void f(const T& param)
{
    std::cout << type_id_with_cvr<T>().pretty_name() << std::endl;
    std::cout << type_id_with_cvr<decltype(param)>().pretty_name() << std::endl;
}

//f(i) 호출하면
//int
//int const &
//출력
```

이런 방법들이 추론된 타입을 확인할 때 굉장히 유용하긴 하지만 가장 중요한 건 [item 1,2](item_01_02.md), [item 3](item_03.md)에서 다룬 내용을 정확히 이해하는 것이다. 타입 추론 방식이 동작하는 방식을 정확히 이해하자.