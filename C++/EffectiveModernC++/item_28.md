#reference collapsing 이해

universal reference에서 어떻게 타입 T로부터 넘어온 인자가 lvalue인지 rvalue인지 알 수 있을까? 이제 드디어 그에 관한 이야기를 할 차례다.

```C++
template<typename T>
void func(T&& param);
```

원리는 간단하다. 만약 인자로 lvalue가 넘어온다면 T는 lvalue reference로 추론되고, rvalue가 넘어온다면 T는 레퍼런스가 아닌 타입(값 타입)으로 추론된다. 따라서,

```C++
Widget widgetFactory();

Widget w;

func(w); //T는 Widget&

func(widgetFactory()); // T는 Widget
```

그런데 뭔가 이상하다. 여기서 T가 각각 Widget&, Widget으로 추론된다고 했는데, 그럼 그걸 코드로 나타내면 아래와 같은 형태가 될 것이다.

```C++
void func(Widget& && param);
void func(Widget && param);
```

두번째건 말이 되지만 첫번째건 말이 안된다. 왜냐하면, C++은 기본적으로 ***레퍼런스에 대한 레퍼런스***를 허용하지 않기 때문이다. 이런 레퍼런스에 대한 레퍼런스를 우리가 직접 작성하는 건 컴파일 에러가 되지만 몇몇 특수한 상황에서는 그게 용납이 된다. 위에서 본 universal reference같은 경우가 그 예인데, 위 경우 ```void func(Widget& && param);```이 ```void func(Widget& param);```이 된다는 걸 이미 앞선 아이템들에서 확인했다.

이런 몇몇 특수한 상황에서 컴파일러는 레퍼런스에 대한 레퍼런스가 발생했을 때 reference collapsing이라는 특정한 규칙에 의해 그냥 레퍼런스로 만들어버린다. 레퍼런스는 lvalue, rvalue의 2가지가 있으니 종합하면 총 4가지 케이스가 있을 수 있다. 거기에 대해 컴파일러는 아래 규칙에 따라 레퍼런스를 붕괴시킨다(collapse).

 - 둘 중 하나라도 lvalue reference면 결과는 lvalue reference이다.
 - 그 외의 경우는 rvalue reference이다.

간단하다. 즉 아래처럼 되는 것이다.

```C++
T& & = T&
T& && = T&
T&& & = T&
T&& && = T&&
```

##std::forward 원리


std::forward는 바로 이 규칙을 이용하여 넘어온 인자가 lvalue인지 rvalue인지를 알아낸다.

```C++
template<typename T>
T&& forward(remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```

std::forward의 간략한 구현이다. 이제 아래 코드를 바탕으로 forward의 동작 원리를 알아보자.

```C++
template<typename T>
void f(T&& fParam)
{
    ...
    someFunc(std::forward<T>(fParam));
}
```
여기서, 함수 f의 인자로 Widget타입의 lvalue가 넘어왔다고 하자. 그러면 타입 T는 Widget&으로 추론될 것이다. 그리고, std::forward는 ```std::forward<Widget&>```의 형태로 인스턴스화되어 호출된다.

```C++
Widget& && forward(remove_reference_t<Widget&>& param)
{
    return static_cast<Widget& &&>(param);
}
```

```remove_reference_t<Widget&>```은 Widget이므로,

```C++
Widget& && forward(Widget& param)
{
    return static_cast<Widget& &&>(param);
}
```

여기서 다시 앞에서 언급한 reference collapsing 규칙에 의해,

```C++
Widget& forward(Widget& param)
{
    return static_cast<Widget&>(param);
}
```

결국 lvalue가 넘어왔을 경우 lvalue refernce로 캐스팅해서 돌려주게되므로 의도한 동작을 수행하게 된다. 이제 rvalue가 인자로 넘어간 상황을 생각해보자. rvalue가 인자로 넘어가면 타입 T는 Widget으로 추론될 것이다.

```C++
Widget&& forward(remove_reference_t<Widget>& param)
{
    return static_cast<Widget&&>(param);
}
```

여기서 ```remove_reference_t<Widget>```은 Widget 그대로이므로,

```C++
Widget&& forward(Widget& param)
{
    return static_cast<Widget&&>(param);
}
```

이때 함수의 리턴값으로 나온 rvalue reference는 rvalue로 취급하므로 std::forward는 자신의 역할을 완벽하게 수행하게 된다.

## reference collapsing 발생 상황

reference collapsing, 그러니까 레퍼런스에 대한 레퍼런스를 그냥 레퍼런스로 만들어주는 상황은 총 4가지가 존재한다.

### template instantiation

가장 흔한 케이스다. 템플릿을 인스턴스화 하는 과정에서 레퍼런스에 대한 레퍼런스가 나타나면 reference collapsing 규칙을 적용시킨다. forward 설명하면서 다뤘으니 생략.

### auto 타입 추론

auto 키워드로부터 타입을 만들어낼 때도 reference collapsing 규칙이 적용된다. 기본적인 규칙은 template instantiation때와 똑같다.

```C++
Widget widgetFactory();
Widget w;

auto&& w1 = w;
auto&& w2 = widgetFactory();
```

위와 같은 구문이 있을 때, auto&&는 template때와 마찬가지 과정을 거친다(각각 ```Widget& && w1 -> Widget& w1, Widget&& w2```).

여기까지 살펴봤으면 알겠지만, universal reference는 사실 그냥 rvalue reference다. 단, 아래 조건이 만족되는 경우 universal reference라고 부를 수 있는 것이다.

- ***타입 추론이 lvalue와 rvalue를 구분할 수 있을 때*** 타입 T의 lvalue가 T&로, rvalue가 T로 추론되는 경우를 말하는 것이다.

- ***reference collapsing이 일어날 때***

### typedef 또는 alias declaration을 쓸 때

```C++
template<typename T>
class Widget
{
public:
    typedef T&& RvalueRefToT;
};
```

위와 같은 코드가 있을 때 Widget을 lvalue reference 타입을 이용해 인스턴스화한 경우,

```C++
Widget<int&> w;

typedef int& && RvalueRefToT; //Widget<int&>::RvalueRefToT
```

이런 식으로 reference에 대한 reference가 발생할 수 있다. 이 경우에도 reference collapsing이 적용된다.

### decltype

decltype을 포함한 타입의 분석중에 레퍼런스에 대한 레퍼런스가 나타나면 reference collapsing이 발생한다.

```C++
int& func(int k);

decltype(func(3))& t; // decltype(func(3)) -> int&
                      // int& & -> int&
                      // int& t;
```