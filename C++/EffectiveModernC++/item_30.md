# perfect forwarding 실패 상황에 익숙해져라

prefect forwarding을 쓰면 하나의 함수에서 다른 함수로 인자들을 ***완벽하게***, 즉 인자 그 자체 뿐만 아니라 인자들의 미묘한 특성들(lvalue니 rvalue니, cv지정자니 하는 것들)까지 전달해준다. 

```C++
//원소 하나 전달
template<typename T>
void fwd(T&& param)
{
    f(std::forward<T>(param));
}

//임의 개수 원소 전달
template<typename... Ts>
void fwd(Ts&&... params)
{
    f(std::forward<Ts>(params)...);
}
```

그러나, 문제는 perfect forwarding이 불가능한 상황이 있다는 것이다. 그냥 f함수를 호출할 때는 잘 되는데 fwd 함수를 거쳐서 호출하는 건 불가능한 경우가 존재하기 때문이다.

```C++
f ( expression ); // 이건 됨
fwd ( expression ); // 이건 안 됨 fwd가 f로 perfect하게 forwarding하는 걸 실패했기 때문
```

##Braced initializers

f가 아래와 같이 정의되어 있다고 하자.

```C++
void f(const std::vector<int>& v);
```

이 경우, braced initializer를 이용해서 f를 호출하면,

```C++
f({ 1, 2, 3 }); // { 1, 2, 3 }이 암시적으로 std::vector<int>로 변환됨
```

반면 fwd는 컴파일이 안된다.

```C++
fwd({ 1, 2, 3 }); // 컴파일 실패!
```

왜냐하면, braced initializer는 perfect forwarding에 실패하는 케이스기 때문이다. 

perfect forwarding에 실패하는 모든 케이스는 동일한 이유를 가진다. f를 직접 호출할 경우(f({1,2,3})같이) 컴파일러는 함수 호출이 일어난 장소로부터 넘어온 인자와 f에 선언된 매개변수의 타입을 살펴본다. 그리고 함수 호출시 넘어온 인자를 f의 매개변수로 변환할 수 있는지 살펴보고, 필요하다면 암시적으로 변환하여 호출을 완료한다. 위 예제의 경우 std::vector<int> 임시 객체를 {1 ,2 ,3}으로부터 생성한 뒤 그걸 f의 매개변수 v에 바인딩하는 것이다.
 만약 f를 fwd같은 함수 템플릿을 이용해 간접적으로 호출할 경우, 컴파일러는 fwd의 호출 장소에서 넘어온 인자와 f의 매개변수를 서로 비교하지 않는다. 대신에, fwd의 인자의 타입이 뭔지 **추론**한다. 그리고 추론된 타입과 f의 매개변수 타입을 비교하는 것이다. perfect forwarding은 다음 두 가지 중 하나가 발생할 때 실패한다.

 - **컴파일러가 타입을 추론할 수 없는 경우** fwd의 매개변수 중 일부의 타입을 추론할 수 없는 경우다. 이 경우 컴파일에 실패하게 된다.
 - **컴파일러가 잘못된 타입을 추론하는 경우** fwd의 매개변수 중 일부의 타입을 잘못 추론하는 경우다. 무슨 뜻이냐면 잘못 추론된 타입으로 인스턴스화된 함수 템플릿이 컴파일 될 수 없거나, fwd에서 추론한 타입을 이용한 함수 f의 호출이 직접 f를 호출하는 것과 다른 결과를 내는 경우를 말하는 것이다. 만약 f가 오버로딩된 함수라면 fwd에서 추론을 잘못했을 때 이상한 오버로딩 함수가 호출될 수 있다.

fwd({1,2,3}) 코드에서, 문제는 braced initializer가 std::initializer_list로 선언되지 않았다는 것이다. 이 경우 컴파일러는 이 타입을 추론할 수 없다. 하지만 [item 1,2](item_01_02.md)에서 설명했듯이 auto의 경우 braced initializer를 std::initializer_list로 추론해낸다. 이렇게 auto를 이용해서 원하는 호출이 가능하긴 하다.

```C++
auto il = { 1, 2, 3 };

fwd(il);
```

## 0이나 NULL을 null pointer로 쓸 때

0이나 NULL을 null pointer로 쓸 경우 이걸 정수 타입의 변수로 잘못 추론하기 때문이다. 자세한 내용은 [item 8](item_08.md)에서 다뤘음.

## 선언만 된 static const 데이터 멤버

일반적으로 static const 정수 데이터는 선언만 해도 된다. 이 경우 컴파일러는 이 상수들에 대한 메모리 공간을 따로 할당하지 않는다(마치 매크로처럼)

```C++
class Widget
{
public:
    static const std::size_t MinVals = 28;
    ...
};
...

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);
```

여기서, Widget::MinVals를 사용해서 widgetData의 초기 capacity 값을 설정했다고 하자. 컴파일러는 정의가 없는 static const 값의 경우 매크로처럼 그냥 해당 값을 그 자리에 채워넣는다. 따라서 MinVal 값을 저장하기 위한 메모리를 할당하지 않아도 되는 것이다. 만약 이 상황에서 MinVal의 주소값을 취하는 연산을 한다면 이건 링크 타임에 에러를 일으킨다.

```C++
void f(std::size_t val);
```

이 때 f에 MinVals를 넘기는 건 되지만 fwd에 MinVals를 넘기는 건 안 된다.

```C++
f(Widget::MinVals); // 잘 된다. f(28)

fwd(Widget::MinVals); // 링크 에러 발생! 
```

원인은 fwd가 인자로 받는게 레퍼런스라는데에 있다. 레퍼런스는 대부분 내부적으로 포인터와 동일하게 취급되기 때문이다. 바이너리 코드에서는 동일하게 포인터처럼 취급되다가 사용할 때만 자동으로 역참조 연산을 수행해주는 형태로 구현하는 것이 레퍼런스기 때문에 실질적으로 포인터와 별 차이가 없는 것이다. 따라서, 앞에서 말했듯이 MinVals의 주소를 참조하다가 링킹 에러를 발생시키게 된다. 해결하는건 쉽다. 그냥 C++ 파일에 정의만 두면 된다.

```C++
const std::size_t Widget::MinVals;
```

##overload function name, template name

함수 f가 아래와 같이 생겼다고 하자

```C++
void f(int (*pf)(int));

//참고 : 이것도 같은 의미.(non-pointer syntax)
void f(int pf(int));
```

함수를 인자로 받아서 뭔가 처리를 하는 함수인 것이다. 그리고 아래와 같은 오버로딩된 함수가 있다.

```C++
int processVal(int value);
int processVal(int value, int priority);
```

이 때 processVal을 f의 인자로 넘겼다고 하자.

```C++
f(processVal);
```

이 경우 f가 받는 인자로부터 두개의 processVal중 첫번째가 저기 들어갈 함수로 적합하다는 것을 알아내고 그 주소를 넘기게 된다. 반면, fwd의 경우 그걸 알 수가 없다.

```C++
fwd(processVal); //에러! 뭘 선택해야할 지 알 수 없음.
```

processVal은 그냥 이름만 가지고는 타입이 없다. 타입이 없으니 타입 추론이 일어날 수도 없고, 따라서 perfect forwarding도 자연히 실패하게 되는 것이다. 같은 문제가 템플릿에서도 발생한다.

```C++
template<typename T>
T workOnVal(T param)
{ ... }

fwd(workOnVal); //에러!
```

workOnVal 이라는 이름만 갖고는 어떻게 인스턴스화해야할 지 알아낼 방법이 없기 때문이다. 이런 함수 포인터를 perfect forwarding 하려면 앞에서 auto를 이용했을 때와 유사하게 지역 변수를 이용하는 수 밖에 없다.

```C++
using ProcessFuncType = int(*)(int);

ProcessFuncType processValPtr = processVal;

fwd(processValPtr);

fwd(static_cast<ProcessFuncType>(workOnVal));
```

## Bitfields

마지막 실패 케이스는 비트필드를 함수의 인자로 받는 경우이다. 무슨 뜻인지 예제를 통해 살펴보자.

```C++
struct IPv4Header
{
    std::uint32_t version:4,
                  IHL:4,
                  DSCP:6,
                  ECN:2,
                  totalLength:16;
};

void f(std::size_t sz);

IPv4Header h;

f(h.totalLength);
```

이건 잘 동작한다. 하지만 fwd를 쓸 경우 이야기가 다르다.

```C++
fwd(h.totalLength); //error!
```

fwd는 인자로 레퍼런스를 받는데 h.totalLength는 const가 아닌 비트필드이기 때문에 발생하는 문제다. C++ 표준에서 **const가 아닌 레퍼런스는 bit field와 바운드 될 수 없다**라고 명시되어 있기 때문이다. 이렇게 만든 이유가 있다. bitfield는 machine의 워드 크기의 일정 부분(32비트 int의 3-5비트와 같이)으로 구성될 수 있고, 이걸 직접 엑세스할 수 있는 방법은 없다. 포인터나 레퍼런스나 내부적으로는 같다고 했는데, 워드를 구성하는 작은 일부 비트 위치를 가리키는 주소 같은 건 존재할 수가 없기 때문이다. 단 비트필드를 값으로 전달하거나 const reference로 전달하는 건 가능하다. 값으로 전달할 경우야 뭐 당연한 것이고, const reference가 가능하다는 것은 신기하다. const reference를 쓸 경우 컴파일러는 해당 비트필드의 복사본을 만든 후 그 복사본이 저장된 타입의 레퍼런스를 쓰는 것이다.

 perfect forwarding 함수에 비트필드를 넘기고 싶은 경우에도 해결책은 간단하다. 위 원리에 따라 다른 곳에 저장한 다음(복사) 그걸 인자로 넘겨주면 된다.

```C++
auto length = static_cast<std::uint16_t>(h.totalLength);

fwd(length);
```
