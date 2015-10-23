#SFINAE

템플릿 메타 프로그래밍의 기본이 되는 기법. ***Substitution Failure Is Not An Error***의 줄임말이다. 다짜고짜 SFINAE부터 다루면 너무 어려우니 왜 이 개념이 필요하고 어떤 때에 쓰이는지 차근차근 살펴보자. SFINAE는 함수 템플릿과 관련하여 나타나는 특성이기 때문에 함수 템플릿의 컴파일 과정에 대해 기본적인 개념이 있어야 이해하기가 쉽다.

> 함수 템플릿의 컴파일 과정과 관련된 아래 주제들은 다 하나하나 상세히 파고 들어가면 굉장히 복잡하고 알아야 할 내용이 많지만, SFINAE가 주제지 함수 템플릿이 주제가 아니므로 함수 템플릿과 관련한 내용은 기초적인 개념만 간단히 서술했다. 혹 함수 템플릿의 컴파일 과정에 대한 상세한 내용이 궁금하신 분들은 따로 더 찾아보시는 것을 추천.

## Template Argument Deduction

함수 템플릿은 단순히 정의를 작성한 시점에서는 어떤 실행 가능한 코드도 생성되지 않는다. 함수 템플릿으로부터 실제 코드가 생성되려면 해당 함수 템플릿이 인스턴스화(function template instantiation)되어야한다. 즉, 컴파일러가 실제 함수를 생성할 수 있도록 템플릿의 인자가 모두 결정되어야한다는 것이다. 이는 클래스 템플릿의 경우도 마찬가지인데, 함수 템플릿의 경우에는 클래스 템플릿과는 다르게 인스턴스화 과정이 암시적으로 일어날 수 있다. 이 암시적인 템플릿 인자 결정 과정이 바로 템플릿 인자 추론(Template Argument Deduction)이다.

```C++
//함수 템플릿
template<typename To, typename From>
To convert(From f);

double d;
//호출되는 인자를 바탕으로 From 타입이 뭐가 될 지 추론함
int i = convert<int>(d); //convert<int, double>(double) 호출
char c = convert<char>(d); //convert<char, double>(double) 호출

//ptr의 타입을 바탕으로 To, From 타입을 추론함
int (*ptr)(float) = convert; //convert<int, float>(float) 인스턴스화
```

위 코드 예시에서처럼, 템플릿에서 몇 개의 인자가 생략된다하더라도 컴파일러가 이를 주변 문맥에 따라 적절히 추론하여 템플릿 인자가 무엇이 될 지 결정함으로써 해당 함수 템플릿을 인스턴스화한다.

## Template Argument Substitution

함수 템플릿의 인스턴스화를 통해 모든 템플릿 인자가 결정된 다음에는 Template Argument Substitution 과정을 거친다. Template Argument Substitution은 함수의 매개변수 리스트(parameter list)에 쓰인 모든 템플릿 매개변수(template parameter)를 거기에 해당하는 템플릿 인자(template argument)로 대체하는 과정이다. 말하자면 해당 템플릿을 어떤 함수로 만들어야할지 알아냈으니 이제 실제로 그 함수를 만들어내는 과정인 것이다. 템플릿 매개변수와 인자 사이의 차이가 좀 헷갈릴 수 있는데 일반적인 함수에서 말하는 매개변수와 인자 사이의 관계와 비슷한 의미다. 이를테면 ```template<typename T, typename F>``` 여기서 T, F가 매개변수(parameter)이고, 여기서 T가 int, F가 char로 추론됐다면 이 int와 char이 각각 인자(argument)가 되는 것이다.

```C++
template <typename T>
void f(T t);

int a = 3;

//T 타입이 int여야함을 추론. 
//따라서 매개변수 T를 인자 int로 substitution한 함수 f<int>()를 만들어낸다.
f(a);
```
## Overload resolution

이렇게 deduction 및 substitution 과정이 끝나면 템플릿으로부터 특정한 타입을 가진 하나의 함수가 만들어진다.

```C++
template<typename T>
void f(T a);

int k = 3;
f(k); //여기서 f는 void(int)
```

위 코드에서 f(k); 호출문으로부터 template argument deduction / substitution 과정을 통해 void f(int) 함수가 만들어진다. 이 때 만약 이 함수와 같은 이름의 또 다른 함수가 존재한다면 어떻게 될까?

```C++
template<typename T>
void f(T a);

void f(double b);

int k = 3;
double l = 5;
f(k); //템플릿 함수 void f(int a) 호출
f(l); //void f(double b) 함수 호출
```

이렇게 동일한 이름의 함수가 2개 이상 있을 때(오버로딩된 함수가 있을 때) 컴파일러는 이 함수들 중에서 실제로 어떤 함수를 호출해야할지를 결정해야한다. 이 후보 함수들(호출될 가능성이 있는 함수들) 중에서 실제 호출될 함수를 결정하는 과정을 overload resolution이라고 할 수 있는 것이다. 그리고 우리가 다루고자 하는 SFINAE 개념은 이 overload resolution과 관련있는 개념이다. 이제 필요한 기본 개념들은 간략하게 소개했으니 본론인 SFINAE로 들어가보자.

## SFINAE

드디어 SFINAE다. SFINAE는 앞에서도 말했지만 ***Substitution Failure Is Not An Error***의 줄임말이다. 여기서 Substitution은 앞에서 말한 Template Argument Substitution을 가리키는 것이다. 즉, Template Argument Substitution을 수행하는 과정에서 Substitution이 실패했다 하더라도 그건 컴파일 에러가 아니라는 것이다. 예제를 통해 이게 무슨 의미인지 살펴보자.

```C++
template<typename T>
void f(typename T::type a);

template<typename T>
void f(T a);

f<int>(0); //두번째 템플릿 함수 호출

class Widget
{
public:
    using type = char;
};

char k;
f<Widget>(k); //첫번째 템플릿 함수 호출
```

이 예제에서 ```f<int>(0);```의 호출로부터 Template Argument Substitution이 일어나는 과정을 생각해보자. 함수 호출문에서 T가 int형임을 명시했으므로, 첫번째 함수 템플릿에서 substitution 과정을 거치면 void f(int::type a); 라는 함수가 만들어지게 된다. 마찬가지 과정을 거치면 두번째 함수 템플릿으로부터는 void f(int a); 함수가 만들어진다. 즉 컴파일러는

void f(int::type a);  
void f(int a);

둘 중에 어떤 함수가 호출될 지를 결정해야한다는 것이다. 그런데 뭔가 이상하다. void f(int::type a); 이게 문법적으로 말이 되는가? 이런 원형을 가진 함수를 그냥 작성했다면 얄짤없이 컴파일 에러를 내뱉었을 것이다. 즉, 이 substitution은 정상적인 substitution이 아니다. Substitution에 ***실패(failure)***한 것이다. 여기서 SFINAE의 의미가 드러난다. SFINAE는 Substitution의 실패가 에러가 아님을 뜻하는 말이라고 했다. 즉, 이와 같은 경우에 컴파일러는 이를 에러로 처리하지 않고 단순히 overload set(오버로딩에서 호출될 수 있는 후보 함수 목록)으로부터 substitution에 실패한 함수를 제외해버린다. 이로 인해 실제 호출될 수 있는 후보 함수 목록에는

void f(int a);

하나만이 남는 것이다. 따라서 컴파일러는 두 번째 함수를 호출하게 된다.

### Type SFINAE

SFINAE는 크게 나누면 Type SFINAE와 Expression SFINAE로 나눌 수 있다. Substitution을 할 때 어느 부분에서 실패를 했는지에 따라 분류를 하는 것이다. 굉장히 다양한 사례가 있지만 그 중 개념 이해를 위한 기본적인 사례만 몇 가지 살펴보자.

#### 스코프 지정 연산(::)에서 지정된 스코프 내에 존재하지 않는 타입을 사용한 경우

위 예제에서 다룬 케이스다.

```C++
template <typename T> int f(typename T::B*);
template <typename T> int f(T);
int i = f<int>(0); // 두번째 오버로딩 사용
```
```f<int>(0);```에서 int::B*는 int 스코프 내에 B 타입이 존재하지 않으므로 Substitution Failure를 일으키게 된다. 이 경우 첫번째 함수 템플릿은 무시(overload set에서 제외)된다.

#### T의 멤버에 대한 포인터를 만들려고 시도했는데, T가 클래스 타입이 아닌 경우

```C++
template<typename T>
class is_class {
    typedef char yes[1];
    typedef char no [2];
    template<typename C> static yes& test(int C::*); // C가 클래스인 경우 선택
    template<typename C> static no&  test(...);      // 그 외의 경우 선택
  public:
    static bool const value = sizeof(test<T>(0)) == sizeof(yes);
};

class Widget {};

bool test1 = is_class<int>::value; //false
bool test2 = is_class<Widget>::value; //true
```

SFINAE의 유용한 활용중 하나가 될 수 있는 예제다. is_class<int>::value 에서 value가 어떤식으로 결정되는지 한 번 짚어보자. 우선 타입 T가 int이므로 ```test<T>(0)```은 ```test<int>(0);```이 된다. 여기서 substitution 과정을 거치면, 두 개의 멤버 함수 템플릿은

```C++
static yes& test(int int::*);
static no& test(...);
```

이 된다. 여기서 첫번째, int int::* 에서 int는 클래스가 아니므로 멤버에 대한 포인터를 만들 수 없다. 여기서 substitution failure가 일어나고, 두 번째 함수가 선택되는 것이다(가변 인수 함수는 모든 인자를 받아들일 수 있으므로, 첫번째에서 실패하면 무조건 두번째가 선택된다). 그 결과 ```test<int>(0)```의 리턴 값은 no&가 되는데, no 타입의 size와 yes 타입의 size는 서로 다르므로 value 값은 false가 되는 것이다. 반면 ```test<Widget>(0)```의 경우 int Widget::*은 Widget의 멤버에 대한 타당한 포인터이므로 substitution failure가 일어나지 않는다. 이 때 가변 인수 함수보다는 그렇지 않은 함수가 오버로딩에서 더 우선순위가 높으므로 value 값이 true가 나오게 되는 것이다.

### Expression SFINAE

Expression SFINAE는 expression의 형태가 이상할 때 발생한다. 이 역시 간단한 예제를 살펴보는 것으로 이해하고 넘어가자.

#### 함수 타입에 잘못된 표현식이 사용된 경우

```C++
struct X {};
struct Y { Y(X){} }; // X 는 Y로 변환 가능
 
template <class T>
auto f(T t1, T t2) -> decltype(t1 + t2); // 오버로딩 함수 1
 
X f(Y, Y);  // 오버로딩 함수 2
 
X x1, x2;
X x3 = f(x1, x2);  // 오버로딩 함수 1에서 SFINAE 발생(x1 + x2는 잘못된 형태의 expression)
                   // 따라서 오버로딩 함수 2가 호출됨
```

### 활용

#### std::enable_if

SFINAE 개념이 잘 활용되는 케이스로 std::enable_if가 있다.

```C++
template<bool B, class T = void>
struct enable_if 
{};
 
template<class T>
struct enable_if<true, T> 
{
    using type = T; 
};
```

std::enable_if가 실제로 어떤 형태로 활용되는지는 [item 27](EffectiveModernC++/item_27.md)에 잘 설명되어 있다. 간단히 요약하자면 위 Type SFINAE에서 ***스코프 지정 연산(::)에서 지정된 스코프 내에 존재하지 않는 타입을 사용한 경우***를 활용하여 ```enable_if<true>::type```은 존재하지만 ```enable_if<false>::type```은 존재하지 않으므로 이를 바탕으로 특정한 경우에만 함수 템플릿을 사용할 수 있도록 만드는 것이다.

```C++
template<typename T, 
         typename = typename std::is_enable<is_class<T>::value>::type>
void f(T a);
```

예를 들어 위와 같이 f 함수를 선언할 경우, T 타입이 클래스 타입일 때만 Substitution이 성공하여 f 함수를 쓸 수 있게 된다(만약 T가 클래스가 아니라면 is_class<T>::value는 false가 되고, std::is_enable<false>::type은 존재하지 않으므로 Substitution이 실패하게 된다). 그 외의 경우는 SFINAE에 의해 위 함수 템플릿 자체가 무시되므로 템플릿의 사용에 일종의 제약을 걸 수 있게 되는 것이다.