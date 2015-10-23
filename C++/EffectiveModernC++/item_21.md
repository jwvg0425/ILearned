# make_shared, make_unique

 std::make_shared와 std::make_unique는 각각 shared_ptr과 unique_ptr을 만들기 위해 사용되는 템플릿 함수들이다. 근데 왜인지는 몰라도 std::make_shared는 C++ 11 표준에 포함이 되어있는데 std::make_unique는 C++ 11표준에 포함이 안 돼있다. std::make_unique는 C++ 14부터 표준에 들어가는데, C++ 11 표준에서 std::make_unique를 쓰고 싶다면 아래의 간단한 코드를 이용하면 된다.

```C++
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

 T타입의 배열이나 custom deleter등을 지원하지 않기 때문에 완벽한 형태의 make_unique라고는 볼 수 없으나 대충 쓰기에는 적절하다. std::make_unique, std::make_shared외에도 std::allocate_shared라는 함수도 있다. 얘는 allocator를 인자로 받아 allocator로부터 메모리를 할당받는다.

## make 함수(std::make_~)의 이점

### 중복 방지

```C++
auto upw1(std::make_unique<Widget>()); //make 사용
std::unique_ptr<Widget> upw2(new Widget); //make 안 씀

auto spw1(std::make_shared<Widget>()); //make 사용
std::shared_ptr<Widget> spw2(new Widget); //make 안 씀
```

new를 쓸 경우에는 같은 타입을 두 번 반복해서 써야한다(unique_ptr의 타입 인자에서 한 번, new 에서 한 번). 반면 make 함수와 auto를 이용하는 경우에는 타입을 한 번만 써도 된다. 중복되는 코드는 최대한 줄이는게 좋다는 소프트웨어 공학적인 관점에서 봤을 때 make 함수가 new를 쓰는 것보다 좀 더 낫다고 볼 수 있다.

### 예외에 대한 안전성

어떤 우선순위에 따라 Widget에 대한 처리를 수행하는 어떤 함수가 있다고 해보자.

```C++
void processWidget(std::shared_ptr<Widget> spw, int priority);

int computePriority(); //우선순위를 계산하는 함수
```

이제 여기서 std::maker_shared 대신 new를 이용해서 이 함수를 호출하는 경우를 살펴보자.

```C++
processWidget(std::shared_ptr<Widget>(new Widget),
               computePriority());
```

놀랍게도 이런 방식의 호출은 자원 누수(resource leak)를 일으킬 위험을 내포하고 있다. std::shared_ptr을 쓰는데 왜 그런 걸까? 그 이유는 바로 ***컴파일러의 코드 번역 순서**때문이다. 당연한 이야기지만 함수가 호출되기 전에 해당 함수의 인자로 넘어가는 것들이 먼저 다 계산이 되어야한다. 위 코드의 경우 아래 세 가지 요소가 먼저 계산이 된 다음 함수의 호출이 일어나야한다.

 - new Widget이 수행되고, 그 결과로 Widget이 반드시 힙 위에 올라가 있어야한다.
 - std::shared_ptr<Widget>의 생성자가 new로 생성된 자원의 포인터를 관리하기 위해 호출되어야한다.
 - computePriority가 수행되어야한다.

그런데 이 때 중요한 건, 이 3가지 작업이 반드시 위에 나열한 순서대로 일어나지는 않는다는 것이다. 물론 new의 경우 std::shared_ptr의 생성자보다 먼저 처리가 되어야만 제대로 std::shared_ptr의 생성자가 호출될 수 있으므로 이 둘의 호출 순서는 명확하지만 computePriority가 언제 호출될 지 순서는 알 수가 없다. 가장 최악의 케이스는 아래 순서대로 수행되는 경우라고 볼 수 있다.

 1. new Widget을 수행한다.
 2. computePriority의 결과값을 계산한다.
 3. std::shared_ptr 생성자를 호출한다.

컴파일러가 이런 순서대로 명령이 수행되도록 컴파일을 했다면, 자원의 누수가 일어날 가능성이 생긴다. 왜냐하면 위 순서에서 2번째 computePriority 함수 수행도중 예외가 발생한다면, 예외로 인해 3번째 std::shared_ptr의 생성자는 호출이 되지 않을 것이고 1번째 과정에서 동적으로 할당된 Widget은 절대 해제되지 않을 것이다. 반면에 std::make_shared를 쓰면 이 문제를 해결할 수 있다.

```C++
processWidget(std::make_shared<Widget>(),
              computePriority());
```

이렇게 할 경우 raw pointer를 std::shared_ptr에 담아서 돌려주는 작업까지 전체가 std::make_shared 함수 내에서 수행되므로 computePriority 함수와의 실행 순서와 상관없이 메모리가 제대로 해제됨을 보장할 수 있다. 물론 std::unique_ptr과 std::make_unique 사이에서도 마찬가지 논리가 성립된다.

### 효율성

 std::make_shared의 또다른 특징은 이게 효율성을 증가시켜준다는 것이다. std::make_shared는 컴파일러가 선형적인 데이터 구조를 통해 더 빠르고 작은 코드를 작성할 수 있게 해준다.

```C++
std::shared_ptr<Widget> spw(new Widget);
```

위와 같이 new를 이용해 std::shared_ptr을 만드는 경우를 생각해보자. 이건 두 번의 메모리 할당 과정을 내포하고 있다. 첫 번째는 new를 통한 Widget 객체의 메모리 할당이며, 두 번째는 std::shared_ptr이 관리할 내부 컨트롤 블록의 메모리 할당이다.

```C++
auto spw = std::make_shared<Widget>();
```

반면 이렇게 std::make_shared를 이용할 경우 메모리 할당은 단 한 번밖에 일어나지 않는다. Widget을 위한 메모리와 컨트롤 블록을 위한 메모리를 연속된 공간에 배치해서 한 번에 할당하면 되기 때문이다. 따라서 메모리 할당 횟수의 감소로 인한 프로그램 속도의 향상을 불러오며, 또한 컨트롤 블록을 다루기 위한 부가적인 정보들(bookkeeping information)의 필요성을 제거함으로써 프로그램의 전체적인 용량도 감소하는 효과를 가져온다. 이는 std::make_shared 뿐만 아니라 std::allocate_shared 역시 해당되는 이야기이다.

## 한계

 make 함수를 이용하는 것은 앞에서 말한 것처럼 굉장히 많은 이점을 갖고 있다. 하지만 항상 그렇듯이 어느 상황에서든 만능으로 동작하는 건 존재하지 않는다. 

### custom deleter

앞의 [item 18](item_18.md), [item 19](item_19.md)에서도 언급했지만 custom deleter를 쓰는 경우 make 함수를 쓸 수 없다.

```C++
std::unique_ptr<Widget, decltype(widgetDeleter)>
    upw(new Widget, widgetDeleter);

std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```

이 경우는 꼭 위 코드처럼 new를 써야 한다. 

### 구현상의 구문론적(syntactic) 문제

make 함수들은 구현 코드상 태생적으로 문제를 갖고 있다. [item 7](item_07.md)에서도 언급한 문제지만, std::initializer_list를 쓰는 생성자와 쓰지 않는 생성자가 같이 오버로딩되어있으면, {}를 써서 생성자를 호출할 경우와 ()를 써서 생성자를 호출할 경우의 결과가 서로 달라질 수 있다. make함수는 생성자를 호출할 때 그 인자들을 그냥 perfect-forward해서 생성자를 호출하는데, 이 때 {}를 써서 호출할 지 ()를 써서 호출할 지를 어떻게 결정해야할까?

```C++
auto upv = std::make_unique<std::vector<int>>(10, 20);

auto spv = std::make_shared(std::vector<int>>(10, 20);
```

예를 들어 위와 같이 코드를 호출했을 때, 과연 20이라는 요소를 10개 가진 벡터가 생성될까 아니면 10,20 이라는 2개의 요소를 가진 벡터가 생성될까? 답은 20이라는 요소 10개로 초기화된다는 것이다. make 함수들은 내부적으로 ()를 써서 생성자를 호출하도록 작성되어있다. 그래서, 만약에 생성자를 {}를 이용해서 호출하고 싶다면 make 함수를 쓸 수 없다. 이 경우 반드시 new를 써서 생성해야만 한다.

```C++
auto initList = { 10, 20 };

auto spv = std::make_shared<std::vector<int>>(initList);
```

std::unique_ptr의 경우 문제가 될 법한 상황이 위의 두 가지 뿐이다. 하지만 std::shared_ptr의 경우 문제가 되는 상황이 두 가지나 더 존재한다.

### new - delete 오버 로딩

 몇몇 클래스들은 new, delete 연산자를 오버로딩하는 경우가 있다. 이 경우 대부분 정확히 해당 클래스의 크기만큼만 메모리를 할당하고 해제하도록 디자인이 되는데, 이 건 std::allocate_shared나 custom deleter를 통한 자원 해제등 std::shared_ptr에서 제공하는 custom allocation 및 deallocation에 별로 어울리지 않는다. std::allocate_shared는 실제로 해당 오브젝트 크기에 더해 control block만큼의 크기를 더 필요로 하기 때문이다. 따라서 자체적인 new, delete를 갖고 있는 클래스를 만들 때 make 함수를 쓰는 건 별로 좋은 생각이 아니다.

### 메모리 낭비

 make 함수를 이용하면 컨트롤 블록과 오브젝트를 연속해서 배치하기 때문에 속도와 크기 면에서 이득을 볼 수 있다고 했었다. 근데 이런 구조가 갖는 문제점이 하나 있다. 만약 레퍼런스 카운트가 0이 되어서 해당 오브젝트가 파괴되는 상황에 도달했다고 치자. 이 경우, 해당 오브젝트가 파괴되어도 그 영역을 포함하는 전체 메모리가 해제되지 않는 상황이 발생할 수 있다. 왜냐하면, 컨트롤 블록이 파괴되는 시점과 오브젝트가 파괴되는 시점이 서로 다르기 때문이다.

 레퍼런스 카운트가 0이 되면 파괴되어야하는 것같지만, 사실 레퍼런스 카운트 말고도 ***weak count***라는게 있기 때문에 weak count까지 0이 되기 전까진 해당 컨트롤 블록이 파괴되어서는 안 된다. 이 weak count는 weak_ptr때문에 있는 카운트인데, 해당 자원이 해제되었더라도 weak_ptr을 통해 그게 파괴되었는지 아닌지를 확인해야하며 이 확인을 위해서는 컨트롤 블록이 살아있어야하기 때문에 오브젝트의 파괴와 컨트롤 블록의 파괴 시점이 달라지는 현상이 발생하는 것이다.

 make 함수를 이용할 경우, 컨트롤 블록과 오브젝트가 한 메모리에 연속해서 존재하도록 메모리를 할당하기 때문에 컨트롤 블록이 사라지기 전까지 오브젝트 메모리 영역까지 해제할 수 없는 문제가 생긴다. 오브젝트의 크기가 작다면야 별 문제가 안 생기겠지만 해당 오브젝트의 크기가 굉장히 크다면 상당한 수준의 메모리 낭비로 이어질 것이다.

```C++

//굉장히 큰 크기의 오브젝트
class ReallyBigType { ... };

auto pBigObj =
    std::make_shared<ReallyBigType>();

//std::shared_ptr과 std::weak_ptr을 여러 개 생성해서 작업을 진행
...

//마지막 std::shared_ptr이 파괴되고 레퍼런스 카운트가 0이 됨.
//오브젝트는 파괴됐지만, 아직 weak_ptr이 남음
...

//이 구간에서, 사용하지 않음에도 불구하고 낭비되는 메모리 공간이 발생!
...

//마지막 weak_ptr이 파괴되고, 이 시점에서야 메모리가 모두 해제됨.
...
```

중간 구간이 길어지면 길어질 수록 메모리 낭비도 심해질 것이다. 하지만 직접 new를 사용하는 경우 레퍼런스 카운트가 0이 되는 시점에서 오브젝트가 할당된 메모리만 해제를 할 수 있으므로 이런 낭비가 사라진다.


## 예외에 안전하게!

 지금까지 std::make_shared를 쓸 수 없는 상황과 쓰는게 적절하지 않은 상황들을 봤다. 근데 이런 경우 그냥 new를 썼다간 앞에서 말한 것과 같은 예외로 인한 자원 누수 문제를 피할 수가 없게 된다. 이럴 때 자원 누수를 피하기 위한 가장 좋은 방법은 ***다른 작업은 일절 하지 않고*** new로 만든 자원을 바로 스마트 포인터의 생성자로 넘기는 것이다. 이렇게 되면 자원의 생성과 스마트 포인터의 생성자 사이에 다른 작업이 끼어들지 않아 예외 발생 때문에 자원 누수가 발생하는 현상을 막을 수 있다.

```C++
//잠재적 위험이 있는 코드
processWidget(
    std::shared_ptr<Widget>(new Widget, cusDel),
    computePriority());

//이렇게 나눠서 하면 예외에 안전!
std::shared_ptr<Widget> spw(new Widget, customDel);
processWidget(spw, computePriority());
```

위 코드처럼 작성하면 예외에는 안전해지지만, 이게 가장 최적화된 코드는 아니다. 왜냐하면 처음의 예외에 대해 안전하지 않은 코드는 인자를 ***rvalue***로 넘기지만 아래의 예외에 안전한 코드는 인자를 ***lvalue***로 넘기기 때문이다. processWidget에서 std::shared_ptr 인자를 값으로 넘겨 받을 경우 lvalue와 rvalue의 차이는 크다. lvalue라면 복사 작업을 수행하는 반면 rvalue라면 이동 작업을 수행하기 때문이다. 이 차이는 꽤 크다. std::shared_ptr에서 복사 작업은 레퍼런스 카운트를 증가시키는 아토믹 연산을 동반하는 반면 이동 작업은 그렇지 않기 때문이다. 따라서 예외에 안전하면서 좀 더 나은 성능을 보이는 코드를 짜고 싶다면, 아래와 같이 std::move를 이용하는 것이 좋다.

```C++
processWidget(std::move(spw), computePriority());
```

