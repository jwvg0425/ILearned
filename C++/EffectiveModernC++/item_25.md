# std::move, std::forward의 바른 사용

## 적합한 위치에 쓰기

```C++
class Widget
{
public:
    Widget(Widget&& rhs)
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
    { ... }
    ...

private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```

rvalue reference는 ***이동(move)될 수 있는*** 객체를 말한다. 반드시 이동되는 것은 아니지만 어쨌든 이동될 가능성은 있다는 뜻이다. 그리고 이렇게 rvalue reference로 인자를 넘겨서 move를 통해 멤버들을 초기화하는 경우는 그 목적에 따라 std::move를 쓰는 게 적합할 것이다.

반면에 universal reference를 쓰는 경우엔, rvalue일 수도 있고 아닐 수도 있다. 이럴 때에는 std::forward가 더 적합하다.

```C++
class Widget
{
public:
    template<typename T>
    void setName(T&& newName)
    { name = std::forward<T>(newName); }
    ...
};
```

만약 위 코드에서 std::forward 대신 std::move를 쓴다면 버그가 발생할 수 있다. std::move는 아무런 조건 없이 무조건 rvalue reference로 바꿔버리기 때문에([item 23](item_23.md) 참조) 이동될 거라고 생각지 않은 객체가 이동되어버리는 불상사가 발생할 수 있기 때문이다.

```C++
class Widget
{
public:
    //std::move 이용. 컴파일은 되지만 안 좋은 코드다!
    template<typename T>
    void setName(T&& newName)
    { name = std::move(newName); }
    ...

private:
    std::string name;
    ...
};

std::string getWidgetName(); //팩토리 함수

Widget w;

auto n = getWidgetName();

//문제 발생! 이 코드에서 n의 값이 lvalue임에도 move되어버림
w.setName(n);

//이 뒷부분부터는 n에 어떤 값이 들어가 있을지 알 수 없다...
```

위와 같이 단순히 std::move를 써버리면 끔찍한 사태가 발생할 수 있다. std::move는 인자로 넘어온 객체가 lvalue인지 rvalue인지 신경쓰지 않고 rvalue reference로 바꿔버리므로 move할 생각이 없었던 객체를 마음대로 move시켜버려 오동작을 일으킬 수 있는 것이다. 그렇다면 템플릿을 쓰지 말고 오버로딩을 이용해서 std::move를 쓸 상황과 안 쓸 상황을 구분해주면 어떨까?

```C++
class Widget
{
public:
    //lvalue인 경우 copy
    void setName(const std::string& newName)
    { name = newName; }

    //rvalue reference인 경우 move
    void setName(std::string&& newName)
    { name = std::move(newName); }
};
```

이렇게하면 동작 자체는 매끔하게 잘 된다. 하지만 성능적인 부분에서 이슈가 있다.

```C++
w.setName("Adela Novak");
```

처음의 템플릿을 사용한 setName의 경우 위 함수 호출은 T 타입이 const char*이 되어 임시 std::string 객체를 만들지 않고 다이렉트로 name에 저장이 된다. 하지만 아래의 오버로딩의 경우, "Adela Novak"를 저장할 std::string 임시 객체를 생성한 다음 그 내용을 std::move를 이용해 이동시키고 마지막에 다시 임시 객체를 파괴하는 과정을 거쳐야만 한다. 인자 타입이 std::string으로 되어 있기 때문에 상수 문자열 "Adela Novak"을 먼저 std::string 타입으로 변환해야만 하기 때문이다. 이런 추가적인 비용은 구현 따라 시스템 따라 다르긴 하지만, 어쨌든 템플릿 대신에 오버로딩을 쓸 경우 런타임에 추가적인 비용을 지불해야할 가능성이 생긴다.

 하지만 이것보다 큰 오버로딩의 문제점은, 함수가 받는 인자의 개수가 많아질 수록 오버로딩해야하는 함수의 개수가 기하급수적으로 늘어난다는 것이다. 인자를 4개 받는 함수의 경우 4개 각각에 대해 lvalue, rvalue인 경우를 고려해야하므로 16개나 되는 함수를 오버로딩해야할 것이다. 무려 2^n에 비례하는 오버로딩이 필요하니 유지보수 비용이 거의 말이 안 된다고 볼 수 있다. 반면에 템플릿을 이용하면 무한대 개수의 인자를 단 하나의 함수로 편리하게 선언하여 쓸 수 있으므로 유지보수에도 유리하고 보기에도 깔끔하다.

```C++
//깔끔하게 하나의 함수로 모든 경우를 커버한다
template<class T, class... Args>
shared_ptr<T> make_shared(Args&&... args);

template<class T, class... Args>
unique_ptr<T> make_unique(Args&&... args);
```

다른 함수에 인자를 넘길 때 std::forward를 써주는게 정석이긴 하지만(그렇게 해야 의도한 대로 함수의 호출이 이루어질 수 있다), 항상 그런 것은 아니다. 하나의 함수 내에서 2개 이상의 함수에 인자를 넘겨주는 경우 제일 처음부터 std::forward로 넘겨줬다간 두 번째 함수 호출에서 문제가 발생할 수 있기 때문이다.

```C++
template<typename T>
void setSignText(T&& text)
{
    //std::forward로 넘기면 rvalue인 경우 이후 과정에서 문제될 수 있다.
    sign.setText(text); 

    auto now = std::chrono::system_clock::now();

    //위에서 std::forward로 넘겼을 경우
    //여기서 text 값을 보장 못함. 이런 경우는 맨 마지막만 std::forward
    signHistroy.add(now, std::forward<T>(text));
}
```

std::move 역시 이와 유사하다. 이런 케이스에서는 std::move의 경우도 마찬가지로 맨 마지막에만 써주자.

## 함수의 리턴값 최적화

값을 리턴하는 함수이며, rvalue reference 혹은 universal reference를 리턴하고자 할 때 std::move나 std::forward를 리턴하는 값에 적용시키고 싶을 수 있다. 예를 들어, 아래와 같은 상황을 생각해보자.

```C++

//ver 1. move 사용
Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs); //lhs를 리턴 값으로 move시킴.
}

//ver 2. move 사용 안함
Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return lhs; //lhs 그냥 리턴
}
```

위 두 가지를 비교해보자. 첫 번째 move를 쓰는 경우 lhs의 값이 함수의 리턴값을 저장하는 장소로 move될 것이다. 반면 두 번째 move를 쓰지 않는 경우는 함수의 리턴값을 저장하는 장소로 lhs의 값을 copy해야만 한다. 만약 Matrix 함수가 move 연산을 지원하고 그 효율이 copy보다 훨씬 뛰어나다면 위 두 가지의 성능 차이는 상당할 것이다. 혹 Matrix 함수가 move 연산을 지원하지 않는다 하더라도, move를 일단 적용시켜놓으면 move를 지원하지 않는 경우는 copy를 쓰고 나중에 move가 추가되거나 하면 그 때 즉시 move를 쓰도록 다시 컴파일 할 수 있기 때문에 어떻게보나 move를 쓰는게 낫다. 이 논리는 universal reference에도 그대로 적용된다.

```C++
template<typename T>
Fraction reduceAndCopy(T&& frac)
{
    frac.reduce();
    
    //rvalue면 move, lvalue면 copy
    return std::forward<T>(frac);
}
```

std::forward가 없다면 frac이 rvalue라 할 지라도 copy를 해서 조금 성능이 떨어지게 될 것이다. 따라서 이런 경우 리턴값에 적절히 std::move나 std::forward를 적용시켜준다면 성능상의 최적화 효과를 얻을 수 있다.

### 제대로 알고 쓰자

위의 최적화 방법을 확장해서 써서는 안되는 경우에까지 적용시키려들 수가 있다. 그러나, 저런식으로 std::move나 std::forward를 리턴값에 쓰는 건 아무 때나 적용해도 되는게 아니다. 아래 코드를 보자.

```C++
Widget makeWidget()
{
    Widget w;

    ...

    return w;
}
```

이 경우 처음 w를 생성하고, 뭔가 작업을 한다음 w를 리턴한다. 이 때, 리턴하는 과정에서 w가 lvalue이므로 리턴값으로의 복사가 일어나고, 이 복사에 드는 비용을 없애기 위해 앞에서 다룬 최적화 방법을 적용시키고 싶을 수 있다.

```C++
Widget makeWidget()
{
    Widget w;

    ...

    return std::move(w);
}
```

그러나, 이렇게 해서는 안 된다. 이런 식으로 지역 변수를 리턴하는 코드에서는 리턴값에 std::move같은 걸 쓰면 안 된다. 위의 copy 버젼 함수는 실제로 copy를 수행하지 ***않는다***. RVO(return value optimization)라고 불리는 기법으로, 컴파일러가 위와 같이 지역 변수 w를 애초부터 메모리의 리턴 값을 저장하는 위치에 메모리를 할당함으로써 부가적인 복사가 일어나지 않도록 만드는 최적화 때문이다. 이런 RVO는 값을 리턴하는 함수에서 1. 지역 변수의 타입이 함수의 리턴 타입과 동일할 때, 2. 그 지역 변수가 리턴되는 값일 때 두 가지 조건을 만족할 경우 일어난다. 다시 아까 전의 코드를 살펴보자.

```C++
Widget makeWidget()
{
    Widget w;

    ...

    return w;
}
```

여기서 w는 위에서 말한 두 가지 조건을 모두 만족한다. w의 타입은 makeWidget의 리턴 타입과 동일하며, w가 makeWidget이 리턴하는 바로 그 객체다. 따라서 이 코드에서는 RVO가 일어나 return 값에 대한 copy가 일어나지 않는다(copy elision). 반면 리턴 값에 std::move를 쓰는 경우, std::move(w)는 w의 rvalue reference를 돌려준다. 이 건 w를 돌려주는 게 아니라 w의 참조를 돌려주는 것이므로 RVO의 두 번째 조건을 만족하지 못하며, 따라서 최적화는 일어나지 않고 컴파일러는 w의 값을 리턴값을 저장하는 장소로 move하는 연산을 수행해야만 한다(RVO가 적용될 경우 어떤 부가적인 연산도 필요 없으므로 명백한 손해다).

하지만 그래도 혹시 미심쩍다고 생각할 수 있다. RVO의 조건을 만족하지만 copy를 피하기 힘든 경우, 그러니까 함수의 제어 경로가 다양해서 리턴하는 값이 제각각인 경우는 컴파일러가 어떤 지역 변수를 리턴 값으로 정해야할지 알기 힘들어 RVO가 적용될 수 없을 텐데 이럴 땐 std::move를 써야하지 않나? 라고 생각할 수도 있는 것이다.

```C++
Widget foo()
{
    Widget a, b, c, d;

    //이런 경우 a,b,c,d 넷 다 RVO의 조건을 만족하지만
    //어떤 객체를 return value 위치에 저장할지 판단할 수 없다!
    if(val == 1) return a;
    else if(val == 2) return b;
    else if(val == 3) return c;
    else return d;
}
```

그러니 이런 경우에는 std::move를 적용시키는게 명백히 더 효율적이지 않을까? 안타깝지만 이 것도 틀린 이야기다.

컴파일러는 RVO의 조건을 만족하지만 return값에 대한 copy 연산을 회피할 수 없는 경우에, ***리턴 될 수 있는 모든 객체를 rvalue로 취급하도록*** 컴파일을 한다. 그러니까, 위 예제 코드가 컴파일 될 때 컴파일러는 실제로

```C++
Widget foo()
{
    Widget a, b, c, d;

    //a,b,c,d 모두 rvalue로 판단함!
    if(val == 1) return std::move(a);
    else if(val == 2) return std::move(b);
    else if(val == 3) return std::move(c);
    else return std::move(d);
}
```

라고 쓰여있는 것으로 생각하고 컴파일을 한다는 것이다. 이건 값 전달에 의해 함수로 넘어온 매개 변수에 대해서도 마찬가지로 적용된다.

```C++
//사람 시점
Widget makeWidget(Widget w)
{
    ...
    return w; //값 전달에 의한 매개변수를 그대로 리턴
}

//컴파일러 시점
Widget makeWidget(Widget w)
{
    ...
    return std::move(w); //컴파일러는 rvalue로 취급
}
```

결국 지역 변수를 값으로 리턴하는 함수에서 std::move를 쓰는 건 괜히 컴파일러가 RVO를 적용시키지 못하도록 헷갈리게만 만들 뿐이지 하등 도움이 되지 않는다는 이야기다. std::move를 리턴 값에 적용하는게 적합한 경우도 있지만(처음에 다룬 참조 관련 케이스) 그렇지 않은 경우도 있으니 적절한 상황을 잘 구분해서 쓰도록 하자.