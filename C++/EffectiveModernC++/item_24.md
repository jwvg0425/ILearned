#universal reference, rvalue reference

어떤 타입 T에 대한 rvalue reference는 T&&로 나타낸다. 따라서 소스 코드에 나타나는 T&&들은 모두 T 타입에 대한 rvalue reference로 봐도 될 것이다. 진짜 그럴까? 안타깝게도 실제로는 전혀 그렇지 않다.

```C++
//rvalue reference
void f(Widget&& param);

//rvalue reference
Widget&& var1 = Widget();

//not rvalue reference
auto&& var2 = var1;

//rvalue reference
template<typename T>
void f (std::vector<T>&& param);

//not rvalue reference
template<typename T>
void f(T&& param);
```

T&& 는 두 가지의 의미를 갖고 있다. 하나는 rvalue reference, 또다른 하나는 해당 타입이 rvalue reference ***거나*** lvalue reference라는 것이다(즉, T& 거나 T&& 둘 모두 가능하다는 것). 심지어는 이건 const냐 c아니냐 volatile이냐 아니냐까지 모두 상관하지 않고 받아들인다. 사실상 거의 어떤 타입이든 다 가능하다고 봐도 되는 것이다. 이런 성격 때문에 이름 짓기 좋아하는 meyers는 이걸 ***universal reference***라고 부른다.

## universal reference가 나타나는 경우

universal reference는 두 가지 상황에서 발생한다. universal reference가 나타나는 가장 일반적인 상황은 함수 템플릿의 인자로 사용되는 상황이다.

```C++
//param은 universal reference
template<typename T>
void f(T&& param);
```

또 다른 상황은 auto 타입 추론이다.

```C++
//var2는 universal reference
auto&& var2 = var1;
```

결국 universal reference는 일반적으로 ***타입 추론***을 하는 상황에서 나타난다는 것이다. 첫번째 함수 f에서 param의 타입이 추론되고, 두 번째 var2 = var1;에서 var2의 타입이 추론된다. 아래와 같이 타입 추론 없이 &&을 쓰는 경우에는 명백하게 rvalue reference이다.

```C++
//이 경우는 명백하게 rvalue reference이다.
void f(Widget&& param);

Widget&& var1 = Widget();
```

universal reference 역시 참조이므로 반드시 초기화가 되어야한다. 이 초기화가 universal reference가 lvalue reference가 될 지 rvalue reference가 될 지를 결정한다. 초기화할 때 넘어오는 인자에 따라 lvalue reference가 될 수도 rvalue reference가 될 수도 있는 것이다.

```C++
template<typename T>
void f(T&& param); //param은 universal reference

Widget w;

f(w); //w = lvalue이므로 param의 타입은 Widget&

f(std::move(w)); //std::move(w)는 rvalue이므로 param의 타입은 Widget&&.
```

universal reference가 나타나기 위해 타입 추론이 필요한 것은 사실이지만 그걸로 충분하진 않다. universal reference가 나타나기 위해서는 반드시 레퍼런스의 형태가 ***T&&***이어야 한다. 이게 무슨 뜻인지 예제를 통해 살펴보자.

```C++
//이 경우 param의 타입은 반드시 rvalue reference이다!
template<typename T>
void f(std::vector<T>&& param);
```

위와 같은 코드에서 f가 실행될 때 T의 타입이 추론되긴 하지만, param의 타입이 ```T&&```가 아니라 ```std::vector<T>&&```이기 때문에 이건 universal reference가 될 수 없다. 이 경우 param의 타입은 무조건 rvalue reference가 된다. 그래서 아래와 같은 현상이 발생한다.

```C++
std::vector<int> v;

//error! lvalue를 rvalue reference로 바인딩할 수 없다.
f(v);
```

심지어는 const같은 미묘한 제약정도만 붙어도 universal reference는 발생하지 않는다.

```C++
//param은 무조건 rvalue reference 타입.
template<typename T>
void f(const T&& param);
```

좋아, 그럼 템플릿 안에 있는 T&&는 모두 universal reference로 보면 되겠군! 이라고 생각할 수 있지만 꼭 그렇지만은 않다는 점도 명심해야한다. 템플릿 안에 있는 T&&지만 타입 추론이 일어나지 않는 경우도 존재한다.

```C++
//std::vector의 구현 일부
template<class T, class Allocator = allocator<T> >
class vector
{
public:
    ...
    void push_back(T&& x);
    ...
};
```

위 코드에서 push_back은 T&&를 인자로 받는다. 그럼 T&&는 universal reference겠네? 그렇지 않다. T&&는 rvalue reference다.

```C++
std::vector<Widget> v;

//위 선언에 의해 구체화되는 클래스
class vector<Widget, allocator<Widget> >
{
public:
    //rvalue reference
    void push_back(Widget&& x);
};
```

위 코드와 같이, vector가 구체화될 때 push_back의 인자도 같이 정해지기 때문에 push_back을 호출할 때 타입 추론이 일어나는 것이 아니다. 반면 vector의 emplace_back 멤버 함수의 경우 타입 추론이 일어난다.

```C++
template<class T, class Allocator = allocator<T> >
class vector
{
public:
    ...
    template<class... Args>
    void emplace_back(Args&&... args);
    ...
};
```

emplace_back이 받는 Args의 경우 타입 T와는 무관하다. 따라서 Args의 경우 타입 추론이 일어나고, 이 형태는 T&& 형태와 완전히 같으므로(parameter pack은 일단 무시) 이건 universal reference가 되는 것이다.

그리고 위에서 언급한 auto&&의 경우도 마찬가지로 타입 추론이 일어나며, T&& 형태와 완전히 같으므로 universal reference가 되는 것이다. C++11에서는 이걸 쓸 일이 그렇게 많진 않지만 C++ 14부터는 달라진다. C++14에서 람다의 매개변수로 auto&&를 쓸 수 있기 때문이다.

```C++
auto timeFuncInvocation =
    [](auto&& func, auto&&... params)
    {
        beginTimer();
        std::forward<decltype(func)>(func)
        (std::forward<decltype(params)>(params)...);
        endTimer();
    }
```

이런식의 구현이 가능하기 때문에 굉장히 유용하게 쓸 수 있다.
