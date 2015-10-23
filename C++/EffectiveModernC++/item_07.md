# 객체를 만들 때 ()과 {}를 구분해라

C++11에서 객체를 초기화하는 방법은 4가지가 존재한다.

```C++
int x(0);       // ()를 이용한 초기화

int y = 0;      // =을 이용한 초기화

int z{ 0 };     // {}를 이용한 초기화

int w = { 0 };  // =과 {}를 이용한 초기화
```

여기서 {}와 = {}는 동일한 의미를 갖기 때문에 사실상 3가지의 방법이 존재한다고 볼 수 있다.

## uniform initialization

C++ 뉴비라면 아래의 두 문장을 헷갈리기 쉽다.

```C++
Widget w1;  // 디폴트 생성자 호출

Widget w2 = w1; // 대입 ㄴㄴ 복사 생성자 호출

w1 = w2; //대입임. 연산자 =이 호출됨.
```

똑같이 =으로 표기되기 때문에 얼핏 대입으로 헷갈릴 수 있지만 Widget w2 = w1; 문장의 경우 복사 생성자가 호출된다. 

또 C++ 98에서는 STL 컨테이너를 특정한 값의 집합을 이용해서 초기화하는게 불가능하다. 위의 헷갈리는 문제나 STL 컨테이너를 초기화할 수 없는 문제등을 해결하기 위해 C++ 11에서는 uniform initialization을 제공한다. 이상한 이름 붙이기 좋아하는 Scott Meyers는 이게 중괄호({})를 이용하기 때문에 braced initialization이라고 부른다고 한다.

```C++
std::vector<int> v{ 1, 3, 5 }; //v는 1,3,5라는 세 개의 원소로 초기화된다. 
```

C++ 11에서는 클래스의 비정적 멤버 변수 초기값을 설정해줄 수 있다. 이 때 =과 {}는 쓸 수 있지만 ()는 못 쓴다.

```C++
class Widget
{
private:
    int m_X{ 0 }; // m_X의 기본값은 0
    int m_Y = 0;  // m_Y의 기본값도 0
    int m_Z(0);   // error! 이렇게는 초기화안됨.
};
```

반면에, 복사할 수 없는 객체(std::atomic같은 것 - item 40에서 다룸.)은 ()나 {}로만 초기화되고, =을 쓰는 걸로는 초기화가 안 된다.

```C++
std::atomic<int> ai1{ 0 }; // 가능

std::atomic<int> ai2(0);   // 가능

std::atomic<int> ai3 = 0;  // 이건 불가능!
```

이런 이유때문에 {}가 uniform initialization이라고 불린다. {}는 어떤 상황에서는 쓸 수 있지만 =을 쓴 초기화나 ()를 쓰는 초기화는 사용 불가능한 케이스가 존재하기 때문이다.

###장점

 {}를 이용한 초기화는 아무 상황에서나 쓸 수 있다는 장점 말고도 다른 몇 가지 장점이 있다. 첫 번째는 바로 내장 타입에 대해 암시적인 축소 변환(narrowing conversion)을 방지해주는 효과가 있다는 것이다. 즉 double->float으로의 암시적 변환이나 double->int로의 암시적 변환같이 데이터의 손실 가능성이 있는 변환이 암시적으로 수행되는 것을 막아준다.

```C++
double x, y, z;

int sum{ x + y + z }; // 에러! double -> int는 안됩니다

//이건 됩니다. 값이 잘리건 말건 경고만 주고 컴파일 됨.
int sum2(x + y + z);
int sum3 = x + y + z;
```

 uniform initialization의 또다른 장점은 모호한 구문(most vexing parse)을 완전히 막아준다는 것이다.

```C++
Widget w1(10); //10을 인자로 하는 Widget의 생성자 호출

Widget w2(); // 인자 없이 Widget의 생성자를 호출하는지,
             // Widget을 리턴하는 함수 w2의 원형을 선언한 건지 모호하다.
             // 이 경우 함수 원형 선언하는 걸로 판단해버림.
```

위와 같이 구문의 해석이 모호한 경우에도 uniform initialization을 이용하면 이런 헷갈림이 없어진다.

```C++
Widget w2{}; //누가 봐도 인자 없이 Widget의 생성자를 호출하는 것.
```

## 이 세상에 무조건 좋은 건 없다

위에서 uniform initialization이 굉장히 좋고 짱짱인 것처럼 묘사했지만 uniform initialization역시 항상 무조건 좋은 것만은 아니다. uniform initialization을 사용할 때에도 주의해야할 점들이 굉장히 많다. 하나씩 살펴보자.

 uniform initialization은 std::initializer_list와 생성자 오버로딩이 얽혔을 때 프로그래머가 생각한 것과는 다른 결과를 불러일으킬 수 있다([item 1,2](item_01_02.md)에서 이미 auto 타입 추론에서 uniform initialization이 일으키는 문제에 대해 살펴보았다).

 일단 생성자에서 인자로 std::initializer_list를 받는 녀석이 없다면 {}를 쓰든 ()를 쓰든 동작의 차이는 없다. 

```C++
class Widget
{
public:
    //std::initializer_list를 인자로 받는 생성자는 없음
    Widget(int i, bool b);
    Widget(int i, double d);
};

Widget w1(10, true); //첫번째 생성자 호출
Widget w2{10, true}; //마찬가지로 첫번째 생성자 호출
Widget w3(10, 5.0);  //두번째 생성자 호출
Widget w4{10, 5.0};  //마찬가지로 두번째 생성자 호출.
```

하지만 std::initializer_list를 인자로 받는 생성자가 하나라도 생긴다면 이야기는 완전히 달라진다. uniform initialization은 std::initializer_list를 굉장히 선호한다. 어떤 방식으로든 인자 목록이 std::initializer_list에 들어갈 수 있다면 std::initializer_list를 사용한 생성자를 호출해버린다. 

```C++
class Widget
{
public:
    Widget(int i, bool b);
    Widget(int i, double d);

    //std::initializer_list를 인자로 받는 생성자 추가
    Widget(std::initializer_list<long double> il);
};

Widget w1(10, true); //첫 번째 생성자 호출

Widget w2{10, true}; //놀랍게도 새로 추가된 세 번째 생성자가 호출됨

Widget w3(10, 5.0);  //두 번째 생성자 호출

Widget w4{10, 5.0};  //이 놈도 세 번째 생성자를 호출함
```

놀랍게도 uniform initialization을 사용한 w2,w4가 모두 세 번째 생성자를 호출하게 바뀌어 버렸다. w2의 경우 10, true가 인자로 들어 있지만 이 두 개의 타입(int, bool)은 모두 long double로 변환 가능하기 때문에 둘 다 long double로 변환되어 3번째 생성자에 인자로 넘어가게 된다. w4 역시 마찬가지. 

 심지어 자주 이용되는 복사, 이동 생성자 마저도 std::initialization_list를 이용한 생성자에게 우선순위를 뺏길 수 있다.

```C++
class Widget
{
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);

    operator float() const; //float으로 변환하는 연산자 추가
};

Widget w5(w4); //복사 생성자가 호출됨
Widget w6{w4}; //복사 생성자가 아니라 세 번째 생성자를 호출해버린다!
Widget w7(std::move(w4)); // 이동 생성자 호출
Widegt w8{std::move(w4)}; // 세 번째 생성자 호출함
```

uniform initialization을 쓰면 w4 -> float()연산을 통해 float으로 변환 -> float은 long double로 변환 가능 -> long double로 변환된 인자를 std::initializer_list에 넣어 세 번째 생성자를 호출 의 과정을 거쳐 복사 생성자가 아니라 세 번째 생성자를 호출해버린다. w8 역시 마찬가지 이유로 세 번째 생성자 호출. 이정도쯤 되면 uniform initialization은 거의 병적으로 std::initializer_list에 집착하는 수준이라고 봐도 될 것 같다. 

또 다른 경우를 살펴보자.

```C++
class Widget
{
public:
    Widget(int i, bool b);
    Widget(int i , double d);

    //bool을 원소로 하는 initializer_list 인자로 받음
    Widget(std::initializer_list<bool> il);
};

Widget w{10, 5.0}; //error 발생!! 뭐지??
```

 두 번째 생성자가 인자의 타입과 정확히 들어맞음에도 불구하고 컴파일러는 이걸 무시해버린다. 위에도 말했지만 uniform initialization은 std::initializer_list에 병적인 집착 증세를 갖고 있어서 인자로 std::initializer_list를 갖고 있는 생성자가 있을 경우 다른 애들은 그냥 무시해버리는 수준이다. 이 경우 std::initializer_list의 원소 타입이 bool이기 때문에 10, 5.0을 bool 타입으로 변환하려고 하니 축소 변환(narrowing conversion)이 일어나서 변환할 수 없기 때문에 컴파일 에러를 일으키는 것이다.

 std::initializer_list를 인자로 받는 생성자가 있음에도 불구하고 uniform initialization이 다른 생성자를 호출하는 경우가 있긴 하다. 바로 인자의 타입이 std::initializer_list의 원소 타입으로 변환할 방법이 단 하나도 존재하지 않는 경우이다. 

```C++
class Widget
{
public:
    Widget(int i, bool b);
    Widget(int i, double d);

    Widget(std::initializer_list<std::string> il);
};

Widget w1(10, true); // 첫번째 생성자 호출
Widget w2{10, true}; // 첫번째 생성자 호출
Widget w3(10, 5.0);  // 두번째 생성자 호출
Widget w4{10, 5.0};  // 두번째 생성자 호출
```

위와 같은 경우 10(int), true(bool), 5.0(double)의 경우 모두 std::string으로 변환할 방법이 아예 존재하지 않는다. 그렇기 때문에 정상적으로 첫번째, 두번째 생성자가 호출되는 것이다.

마지막으로 한 가지 애매하게 해석될 수 있는 경우를 살펴보자.

```C++
class Widget
{
public:
    Widget();
    Widget(std::initializer_list<int> il);
};

Widget w1; //기본 생성자 호출
Widget w2(); //most vexing parse. 함수가 되어버림!
Widget w3{}; // -> 어떤 생성자를 호출할까?
```

위 코드에서 w3의 경우 원소가 하나도 없는 std::initializer_list를 이용해서 두 번째 생성자를 호출할 지, 아니면 인자가 하나도 없으니 첫번째 생성자를 호출할 지 헷갈린다. 여태껏 보여왔던 std::initializer_list에 대한 집착이라면 두 번째 생성자를 호출할 것만 같지만 아니다. 이 경우 첫번째 생성자(기본 생성자)를 이용해서 초기화한다. 참으로 변덕스러운 녀석이다. 원소가 하나도 없는 std::initializer_list를 이용해서 초기화하고 싶은 경우 아래와 같은 방법을 이용해야한다.

```C++
Widget w4({});
Widget w5{{}};

//=과 같이 쓰는 경우에도 마찬가지.
Widget w6 = {}; // 기본 생성자 호출
Widget w7 = {{}}; // 원소가 하나도 없는 std::initializer_list 이용해 호출
```

즉 {}을 인자로 명확히 명시해주어야 한다.

## 코딩시의 주의사항

 uniform initialization을 이용할 때에는 많은 주의를 기울여야한다. 자칫하면 위에서 봤듯이 전혀 의도하지 않은 결과를 일으킬 수 있기 때문이다.

### 클래스 설계자의 관점

가장 기본이 되는 대원칙은, 클래스의 사용자가 ()를 쓰든 {}를 쓰든 코드에 아무 영향을 끼치지 않도록 만들라는 것이다. 이에 반대되는 나쁜 예시가 C++ 표준 라이브러리에도 존재한다.

```C++
std::vector<int> vi(10, 20); //20이라는 원소 10개로 초기화
std::vector<int> vi{10, 20}; //10,20 두 개의 원소로 초기화
```

괄호만 다르고 인자는 완전 동일한데 결과는 천지차이다. 클래스의 인터페이스를 이런 식으로 구성하면 사용자 입장에서든, 코드를 읽는 사람의 입장에서든 혼란스러울 것이라는 건 굉장히 뻔한 일이다. 그러니 클래스를 설계할 때는 웬만하면 ()를 쓰든 {}를 쓰든 사용자에게 영향이 없도록 만들자. 

 또 std::initializer_list를 인자로 받는 생성자가 없던 상황에서 하나를 추가한다고 생각해보자. 이건 정말 큰 파장을 일으킬 것이다. 위에서 길게 말했듯이 uniform initialization은 std::initializer_list에 대해 굉장한 집착을 보이기 때문에 이런 생성자를 추가할 경우 기존에 작성되어 있던 코드들이 의도한 것과는 다른 이상한 동작을 보이게 될 수 있다. 따라서 정말 어쩔 수 없는 경우가 아니라면 이런 식의 클래스 확장은 피하는 것이 좋다.

### 클래스 사용자의 관점

 클래스를 초기화할 때 ()를 많이 쓸 건지 {}를 많이 쓸 건지 보통 둘 중 하나를 선택해야 한다. 둘 중에 하나가 딱히 더 좋은 건 아니고 둘 다 나름의 장단점이 있으니 하나를 택일해서 쓰는 편이 좋다.

 - #### 기본 {}, 특수한 상황에 ()

   객체를 초기화할 때 기본적으로 {}를 쓰는 방식이다. 이 방식은 폭넓게 적용 가능하고, 암시적인 축소 변환도 방지해주고, 또 모호한 구문(most vexing parse)도 사라지게 해준다. 다만 몇가지 케이스(std::vector에서 {}, ()를 쓸 때의 차이점)에 대해 인지하고 있어야하고 이런 경우에는 ()를 써야한다.

 - #### 기본 (), 특수한 상황에 {}

   C++ 98의 전통적인 방식에 가깝게 기본적으로 ()를 쓰고 몇 가지 특수한 케이스에만 {}를 쓰는 방식이다. 이렇게 하면 auto가 std::initializer_list를 타입 추론할 때 생기는 문제점도 피할 수 있고, std::initializer_list를 인자로 받는 생성자 때문에 생기는 골치 아픈 문제로부터도 해방될 수 있다. 다만 컨테이너를 특정한 값의 집합을 이용해 초기화할 때와 같이 {}를 써야만 하는 경우에는 반드시 {}를 써야한다. 


둘중 어느 하나가 좋고 나쁘다 라고 할 수는 없으니, uniform initialization이 어떻게 동작하는지 명확히 이해하고, 자신의 취향에 맞는 스타일을 선택해서 이용하는 것이 좋을 것이다.

### 템플릿 작성시

 템플릿 내부에서 오브젝트를 {}로 생성할지 ()로 생성할지는 굉장히 까다로운 문제다. variadic template을 이용한 예시를 통해 살펴보자.

```C++
template<typename T,
         typename... Ts>
void doSomeWork(Ts&&... params)
{
    T localObject {std::forward<Ts>(params)...); //()을 이용해 지역 객체 생성
    T localObject2(std::forward<Ts>(params)...); //{}를 이용해 지역 객체 생성
}

std::vector<int> v;

//함수 호출
doSomeWork<std::vector<int>>(10, 20);
```

위와 같이 함수 호출을 할 경우 localObject는 20이 10개 들어간 vector가, localObject2는 10,20 두 개의 원소가 들어간 vector가 된다. 이 건 함수 호출을 하는 쪽에서는 판단 가능하지만 함수를 작성하는 쪽에서는 어떤 함수가 호출될 지 전혀 판단할 수 없다(실제 호출 전까지는 타입이 결정되어 있지 않기 때문에, 함수 작성자가 객체마다 서로 다를 생성자의 규칙을 모두 커버할 수는 없다). 동일한 문제가 있는 표준 라이브러리의 함수 std::make_unique와 std::make_shared는 내부적으로 ()를 쓰고 그걸 문서화해두는 걸로 이 문제를 처리했다.