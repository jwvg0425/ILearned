# std::move, std::forward 이해하기

std::move와 std::forward에 대해 가장 먼저 알아야 할 점은, 이들 함수는 실제로 어떤 동작을 수행하지 않는다는 것이다. std::move가 실제로 객체를 move시키지 않고, std::forward도 실제로 객체를 forward하지 않는다. 이들 함수는 단순히 캐스팅을 수행하는 함수(함수 템플릿)일 뿐이다. std::move는 무조건 인자로 넘어온 값을 rvalue reference로 캐스팅해주고, std::forward는 특정한 조건이 만족될 경우에만 인자로 넘어오는 값을 rvalue reference로 캐스팅해준다.

## std::move

먼저 std::move의 동작부터 이해해보자. std::move의 동작을 간략하게 구현하면 아래와 같이 구현할 수 있다.

```C++
//C++ 11 스타일
template<typename T>
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType = 
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}

//C++ 14 스타일
template<typename T>
decltype(auto) move(T&& param)
{
    using ReturnType = remove_refernce_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

std::move는 객체에 대한 universal reference([item 24](item_24.md) 참조)를 취해서 해당 오브젝트에 대한 rvalue reference를 돌려준다. type deduction 부분을 다루면서 짧게 언급하고 넘어갔었지만 T&&가 반드시 rvalue reference를 의미하는 것은 아니다. 이 경우 함수 호출시 넘어온 타입 T가 lvalue reference라면 T&&는 T 타입에 대한 lvalue reference를 의미하게 된다. 이런 경우 때문에 캐스팅을 하기 전 주어진 T타입으로부터 레퍼런스를 제거한 후(type_traits의 remove_reference 이용 - [item 9](item_09.md) 참조) 그 타입에 대한 rvalue reference로 캐스팅하여 리턴해줌으로써 항상 인자로 넘어온 객체의 rvalue reference를 리턴하도록 만들고 있다.

```C++
Widget& w;

//이 경우 w의 타입이 Widget&(lvalue reference)이므로
//타입 추론에 의해 T&&는 Widget&로 취급받는다.
std::move(w);

//위에서 언급한 C++ 14 스타일 간략한 std::move 구현 코드
//위 호출에서 이 코드의 T와 T&&(param의 타입)는 모두 Widget&로 취급됨.
template<typename T>
decltype(auto) move(T&& param)
{
    //remove_reference_t<Widget&> = Widget
    //remove_reference_t<Widget&>&& = Widget&&
    //따라서 ReturnType = Widget&&
    using ReturnType = remove_refernce_t<T>&&;

    //Widget&&로 캐스팅해서 돌려주므로 객체에 대한 rvalue_reference가 됨
    return static_cast<ReturnType>(param);
}
```

std::move는 실제로 객체를 move시켜주는 것이 아니라는 점이 정말 중요하다. 계속 언급된 이야기지만 std::move는 단순히 rvalue reference로 캐스팅을 해줄 뿐이다. 즉, std::move는 실제로 move를 시켜주는 것이 아니라 단순히 move될 수 있는 조건만 갖추게 해주는 함수라는 것이다. 그리고 실제로도 rvalue reference로 인자를 넘겼다고 해서 무조건 move가 되는게 아니다. 아래 예제를 보자.

```C++
class Annotation
{
public:
    explicit Annotation(const std::string text);
};
```

이 클래스는 text의 값을 읽어서 복사해 저장한다. 즉, text의 값을 읽기만 하면 되므로 const 지정자를 붙이는 것이 적절해보인다.
```C++
class Annotation
{
public:
    explict Annotation(const std::string text)
    : value(std::move(text))
    { }

private:
    std::string value;
}
```

그리고 인자로 넘어온 text 값은 value 멤버에 저장하는 용도이므로 std::move를 이용해서 깔끔하게 이동을 시키면 속도도 빠르고 좋을 것 같다. 하지만, 이 경우 std::move를 쓴다해도 실제로 move 연산이 호출되지 않는다. 왜 그럴까?

```C++
class string
{
public:
    ...
    //string의 생성자 중 2가지 - 복사/이동
    string(const string& rhs); //copy
    string(string&& rhs); //move
    ...
};
```

Annotation에서 멤버를 초기화할 때 std::move(text)의 결과로 text는 const std::string&& 타입으로 캐스팅되게 된다. 하지만 위의 std::string 생성자에서도 알 수 있듯이 move 생성자는 std::string&&을 타입으로 받고(일반적으로 move 생성자는 이동시킨 후 값을 수정해야할 필요가 있기 때문에 인자에 const 지정을 하지 않는다), 이 const의 차이 때문에 move 생성자는 호출될 수가 없다. 반면 copy 생성자가 인자로 받는 const string&과 const string&&은 서로 호환이 되기 때문에 의도와는 다르게 copy 생성자가 호출되어버리는 것이다. 프로그램의 동작 자체는 copy든 move든 똑같기 때문에 실제로 move가 호출되는지 copy가 호출되는 지 찾기도 힘들다.

 따라서, 만약 move 연산을 허용하고 싶다면 객체의 타입으로 const를 지정해선 안되며, std::move는 단지 그게 move될 가능성이 있도록 만들어줄 뿐이지 실제로 move를 시키는 것도 move가 됨을 보장하는 것도 아니라는 점을 반드시 명심해야한다.

## std::forward

std::forward의 경우도 std::move와 그리 큰 차이는 없다. 다만 std::forward의 경우는 std::move와는 다르게 조건부로 rvalue reference로 캐스팅할 뿐이다. std::forward는 다음과 같이 universal reference 인자를 다른 함수로 전달하는 함수 템플릿에서 많이 쓰인다.

```C++
//lvalue, rvalue에 따라 다른 동작을 하는 함수
void process(const Widget& lvalArg);
void process(Widget&& rvalArg);

template<typename T>
void logAndProcess(T&& param)
{
    auto now = 
        std::chrono::system_clock::now();

    makeLogEntry("calling 'process'", now);
    process(std::forward<T>(param));
}

//lvalue, rvalue 넘긴 인자에 따라 다른 process가 호출되어야 한다.
logAndProcess(w);
logAndProcess(std::move(w));
```

위와 같은 상황에서 std::forward는 넘어온 인자가 lvalue냐 rvalue냐에 따라 process에 적절한 인자를 넘겨주는 역할을 한다. 만약 std::forward<T> 없이 param을 그냥 넘긴다면, 기본적으로 param은 함수의 인자고 따라서 lvalue이므로(이름이 있다) 무조건 lvalue를 인자로 받는 process가 호출이 될 것이다. 따라서 인자로 넘어온 param이 rvalue로 초기화 됐을 때만 rvalue reference로 캐스팅하고 그렇지 않으면 캐스팅하지 않아야한다. std::forward가 하는 일이 바로 이 것이다. std::forward는 인자가 rvalue로 초기화되었을 때에만 rvalue로 캐스팅해준다.

std::forward는 인자로 넘어온 값이 rvalue로 초기화되었는지 아닌지를 판단하기 위해 template의 타입 인자 T를 활용한다. 이 T 타입 안에 인자로 넘어온 값이 rvalue인지 아닌지 판단할 수 있는 정보가 들어있고, std::forward는 그걸 이용해서 판단을 하는 것이다(그래서 std::forward의 템플릿 타입 인자로 T를 넘겨준다). 정확한 내용은 [item 28](item_28.md)에서 다룬다.

그런데 잘 생각해보면 어차피 둘 다 rvalue reference로 캐스팅하는게 목적이고, std::forward의 경우 타입 인자로 값 타입이 넘어오면 rvalue reference로 캐스팅을 해준다. 그러면 std::move를 쓰는 부분에 std::forward를 써도 잘 동작하지 않을까? 기술적으로만 따지면 아무런 차이가 없고 실제로도 잘 동작한다고 한다. 하지만 이 두 가지를 적절히 잘 분리해서 쓰는게 훨씬 깔끔하고 가독성도 좋다. std::move는 이름 그대로 move를 위한 준비를 하는것이고, std::forward는 원래 lvalue냐 rvalue냐에 따라 다른 함수로 적절히 객체를 넘기는(pass, forward) 역할을 하는 것이므로 둘을 잘 구분해서 쓰는게 훨씬 좋을 것이다.