# decltype 이해하기

## decltype의 동작 및 활용

decltype은 declared type의 의미로 해당 식이 나타내는 타입을 그대로 돌려준다. 어떤 이름(name)이나 표현식(expression)을 decltype에 집어넣으면 decltype은 해당 이름이나 표현식의 타입을 알려준다.

[item 1,2](item_01_02.md)에서 다뤘던 auto나 template에서의 타입 추론과 decltype이 다른 점은, decltype은 해당 표현식이나 이름의 타입을 있는 그대로 돌려준다는 점이다.

```C++
const int i = 0; // decltype(i) => const int
                 // decltype(&i) => const int*
const int& ri = i; // decltype(ri) => const int&
int f(int a, int b); //decltype(f) => int(int,int)
                     //decltype(f(1,1)) => int
class c
{
    int a; //decltype(c::a) => int
    float b; //decltype(c::b) => float
}
```

decltype은 주로 함수 템플릿에서 매개 변수의 타입에 따라 리턴값의 타입이 바뀌는 함수를 만들고 싶을 때 사용한다.

```C++
// C++ 11에서는 불가능!
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}

```
위 함수에서 c[i]의 타입은 Container의 종류에 따라 바뀐다. 따라서 리턴 타입은 해당 함수의 매개 변수의 타입에 의해 결정되는데, 이 때 리턴값을 auto로 쓴다고 해서 자동으로 추론해주지 않는다. C++11의 경우 single-statement lambda에 대해서만 리턴 타입을 추론해주고 나머지 경우에는 자동으로 리턴 타입을 추론해주지는 않는다. 따라서 C++11에서 리턴 값의 타입이 해당 함수의 매개 변수의 타입에 따라 바뀌게 만들고 싶다면 다른 방법을 이용해야한다. C++11에서는 이 때 trailing return type 구문을 이용한다.

```C++
//이렇게 쓰면 가능
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
   -> decltype(c[i]) // -> 뒤에 decltype을 이용해 이 함수의 리턴 타입 설정.
{
    authenticateUser();
    return c[i];
}
```

위와 같이 작성할 경우 decltype(c[i])에 의해 authAndAccess의 리턴 타입이 결정된다. 하지만 C++ 14에서는 모든 경우에 대해 리턴 타입을 auto로 쓰기만 해도 자동으로 리턴 타입을 추론해준다.

```C++
//C++ 14에서는 이렇게 써도 동작한다.
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i]; //c[i]로부터 리턴 타입을 추론함.
}
```

다만 이 때 생기는 큰 문제점은, 이 경우 [item 1,2](item_01_02.md)에서 설명한 auto의 타입 추론 방식이 그대로 적용된다는 것이다. 따라서 c[i]가 어떤 T타입의 레퍼런스(T&)를 리턴할 경우 auto의 타입 추론에 의해 이 함수의 리턴 타입은 T&가 아니라 T가 돼버린다. 이 때문에 코드 작성자의 의도와 엇나가버릴 수 있다.

```C++
std::vector<int> v;

authAndAccess(v,5) = 10; //가능해야 할 것 같지만 v[5]의 타입이
                         //int&가 아니라 int로 추론돼서 컴파일 불가.
```

이건 그냥 auto를 써가지고는 해결이 안 된다. trailing return type 구문을 쓰는 건 좀 깔끔하지 못 해 보인다(C++11에서 나아진 게 없다). 그래서 이럴 때는 다음의 문법을 쓰면 된다. 

## decltype(auto)

C++ 14에서 지원하는 decltype(auto)는 타입 추론을 하되 그 규칙을 decltype의 규칙에 따른다는 의미이다. 그냥 auto가 [item 1,2](item_01_02.md)에서 설명한 방식대로 동작하는 반면, decltype(auto)는 제일 처음 설명한 decltype의 동작대로 타입을 추론한다. 예를 통해 확인해보자.

```C++
int i;
const int& ci = i;
auto           ai  = ci; //auto           => const int로 추론
decltype(auto) ai2 = ci; //decltype(auto) => const int&로 추론
```

따라서 템플릿 함수에서 auto 대신 decltype(auto)를 반환형으로 쓰면 리턴 타입을 해당 표현식이 나타내는 타입 그대로 나타낼 수 있다.

```C++
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}

std::vector<int> v;
authAndAccess(v,5) = 10; // 이제 c[i]의 타입 그대로 int&를 리턴한다!
```

하지만 아직도 문제가 있다. 위 템플릿 함수에서, c는 Container& 타입이다. 만약 이 함수에 좌측값인 컨테이너가 아니라 우측 값인 컨테이너를 넘기고 싶을 땐 어떻게 해야될까? 일단 T&로는 우측값을 받을 수 없다. 우측값을 이용하기 위해 Container&가 아니라 Container&&라고 universal reference를 이용해도 아래와 같은 문제가 생긴다.

```C++
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container“& c, Index i)
{
    authenticateUser();
    return c[i];
}
class Enemy;
std::queue<Enemy> makeEnemyQueue();

//firstEnemy가 가리키는 값은??
auto firstEnemy = authAndAccess(makeEnemyQueue(), 0);
```
c에 넘어온 makeEnemyQueue()는 우측값이며, 따라서 이 문장이 끝나면 사라지는 임시 객체이다. 문제는 이 경우 c[i]가 임시 객체 내부의 한 요소를 가리키고 있으므로, 해당 문장이 끝난 후 firstEnemy는 dangling pointer가 되어버린다. 이 문제를 해결하기 위해서 인자로 좌측값 레퍼런스와 우측값 레퍼런스를 받는 두 개의 함수를 각각 만드는 방법도 있지만, 이건 유지 보수 측면에서 비용이 높다.

여기에 std::forward까지 이용해서 인자로 넘어온 컨테이너가 lvalue인 경우, rvalue인 경우 모두에 대해 정상 동작하게 만들 수 있다.

```C++
//C++ 14
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i) //universal reference 사용
{
    authenticateUser();
    return std::forward<Container>(c)[i]; //std::forward로 c를 좌측값, 우측값에 맞게 처리.
}

//C++ 11. decltype(auto)를 쓸 수 없으니 trailing return type 구문을 쓰자.
template<typename Container, typename Index>
auto authAndAccess(Container&& c, Index i)
    ->decltype(std::forward(Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

##decltype의 예외

decltype은 거의 대부분의 경우 생각한 그대로 동작하지만, 조금 헷갈리게 동작하는 케이스가 있다. decltype은 표현식이 lvalue인 경우 이걸 lvalue reference로 추론하는데, 표현식이 아니라 그냥 lvalue인 이름이 올 경우에는 이를 reference가 아닌 그냥 타입 그대로 추론한다. 이 때문에 약간 헷갈리는 상황이 발생한다.

```C++
decltype(auto) f1()
{
    int x = 0;
    return x; //decltype(x)는 int이므로, f는 int형을 리턴.
}

decltype(auto) f2()
{
    int x = 0;
    return (x); // decltype((x))는 int&이다! x는 이름이지만 (x)는 표현식이다. 
}
```

단순히 생각하면 x와 (x)는 같은 의미이지만 decltype은 이 둘을 다르게 생각한다. x는 변수의 이름이지만 (x)는 엄연한 표현식이기 때문이다. 이와 마찬가지로 decltype(++x)같은 것도 int가 아니라 int&로 추론한다. decltype 안에 들어가는게 표현식인지, 아니면 단순한 이름인지만 고려하면 크게 헷갈리지 않을 듯.