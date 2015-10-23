#const 멤버 함수는 thread safe

수학에서 다항식(polynomial)을 나타내는 클래스를 만든다고 해보자. 다항식을 나타내는 클래스니까 다항식의 근(roots)들을 구하는 함수가 있으면 굉장히 편리할 것이다. 근을 구하는 함수는 다항식 자체에는 어떤 영향을 끼치는게 아니니 이건 const 멤버 함수로 선언하자.

```C++
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const;
};
```

 다항식의 근을 계산하는 함수는 비용이 굉장히 비쌀 것이다. 근은 한 번만 계산하면 바뀌는 값이 아니므로 한 번 계산한 다음에는 값을 캐시(cache)해두고 바로바로 불러오는 방식으로 하면 성능이 훨씬 개선된다. 

```C++
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        //roots를 구하지 않은 경우 계산 후 저장
        if(!rootsAreValid)
        {
           ...  //calculate roots and store
           rootsAreValid = true;
        }

        return rootVals;
    }

private:
    mutable bool rootAreValid { false };
    mutable RootsType rootVals{};
};
```

roots가 리턴하는 값 자체는 변하지 않고, 여전히 roots 함수의 호출은 polynomial 클래스 내부의 값을 변화시키지 않는다. 하지만 캐싱을 하기 위한 두 가지 변수(rootAreValid, rootVals)는 상수 함수라도 값을 변경시킬 수 있어야한다. 그래서 이 두가지 값을 mutable로 선언했다.

## 하지만 멀티스레드가 출동한다면 어떨까?

 멀티 스레드 환경에서 두 개의 스레드가 동시에 roots 함수를 호출했다고 하자.

```C++

/* ------ thread 1 ------ */    /* --------- thread 2 --------- */
 auto rootOfP = p.roots();       auto valsGivingZero = p.roots();
```

상수 멤버 함수는 일반적으로 클래스 내부의 값을 바꾸지 않는다. 그래서 함수를 쓰는 입장에서는 이게 동기화 문제를 발생시키지 안을 것이라 생각하고 그냥 사용할 것이다. 하지만 문제는 이 함수가 내부적으로 **캐싱 동작**을 한다는 것이다.캐싱을 하는 과정에서 rootAreValid, rootVals 변수의 값을 바꾸기 때문에 이 변수에 접근해서 값을 바꾸는 중간에 동기화 문제가 발생할 수 있다. 즉 상수 함수 임에도 불구하고 thread safe 하지 않은 함수기 때문에 발생하는 문제인 것이다.

 이 함수를 thread safe하게 바꾸는 간단한 방법 중 하나는 mutex를 사용하는 것이다.

```C++
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m); //lock mutex

        //roots를 구하지 않은 경우 계산 후 저장
        if(!rootsAreValid)
        {
           ...  //calculate roots and store
           rootsAreValid = true;
        }

        return rootVals;
    }

private:
    mutable std::mutex m;
    mutable bool rootAreValid { false };
    mutable RootsType rootVals{};
};
```

 일단 이렇게 하면 roots 함수에서 락이 걸리기 때문에 동기화 문제는 해결된다. 그래도 문제는 있다. std::mutex가 **move-only type**이라는 것이다. 이 때문에 std::mutex를 멤버로 갖고 있는 클래스는 copy하는게 불가능하다. move만 가능해지는 것이다.

## std::atomic

 몇몇 경우에는 std::mutex를 이용하는게 성능이 꽤 떨어진다. 간단한 예로 특정 멤버 함수가 몇 번이나 호출되었는지 세고 싶다고 하자. 이럴 때는 std::mutex를 이용하는 것보다 std::atomic(각 연산은 개별적(원자적)으로 수행됨이 보장됨. item 40(item_40.md) 참고) 카운터를 이용하는 것이 더 성능이 뛰어나다.

```C++
class Point
{
public:
    double distanceFromOrigin() const noexcept
    {
        ++callCount;

        return std::sqrt((x * x) + (y * y));
    };
private:
    mutable std::atomic<unsigned> callCount { 0 };
    double x, y;
};
```

std::atomic을 이용한 간단한 예제다. std::atomic 역시 move-only type이라는 점에 주의.

std::mutex를 쓰는 것보다 std::atomic을 쓰는 것이 비용이 더 싸기 때문에 가능하다면 std::atomic을 쓰는 것이 좋다. 하지만 무턱대고 std::atomic을 써서 아래와 같은 코드를 작성하면 큰 문제점이 발생할 수 있다.

```C++
class Widget
{
public:

    int magicValue() const
    {
        if(caheValid) return cahedValue;
        else
        {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true;

            return cachedValue;
        }
    }
private:
    mutable std::atomic<bool> cacheValid { false };
    mutable std::atomic<int>  cachedValue;
};
```

아까전 다항식의 근을 캐싱하던 예제와 거의 마찬가지로, 어떤 시간이 오래 걸리는 연산의 결과 값을 캐싱해두고 그걸 가져다 쓰는 클래스다. mutex를 써서 전체에 락을 걸지 않고 cacheValid와 cachedValue 각각을 atomic하게 만들었다.

하지만 이 코드는 굉장히 큰 문제점을 갖고 있다. 다음 예를 통해 어떤 문제가 있는지 살펴보자.

 - 한 스레드가 Widget::magicValue를 처음 호출했다. 이 시점에서 cacheValid는 false다. 그러니 연산을 한 뒤 캐싱을 해야한다. val1, val2를 계산해서 그 결과를 cachedValue에 저장했다.

 - 이 시점에서 또 다른 스레드가 Widget::magicValue를 호출했다. cachedValue값을 이미 계산했지만 아직 cacheValid 값을 true로 갱신하지 않은 상황이다. 따라서 아직 캐싱이 되어 있지 않다고 판단하고 이미 계산한 cachedValue 값을 다시 계산하기 시작한다.

 만약 여러 개의 스레드가 동시에 이런 상황에 빠진다면 이건 굉장한 성능 손실을 가져올 것이다. 여기서 문제는 cached가 계산되었음에도 불구하고 cacheValid가 제때 갱신되지 않았다는 것이므로, 이걸 해결하기 위해 둘의 순서를 바꾼다고 해보자.

```C++
class Widget
{
    int magicValue() const
    {
        if(caheValid) return cahedValue;
        else
        {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cacheValid = true;
            cachedValue = val1 + val2;

            return cachedValue;
        }
    }
};
```

이 건 더 큰 문제를 일으킨다.

 - 먼저 한 스레드가 Widget::magicValue를 처음 호출했다.  cacheValid가 false이므로 val1, val2를 계산한 다음 cacheValid에 true를 대입했다.
 - 이 시점에서 또 다른 스레드가 Widget::magicValue를 호출했다. cacheValid 값이 true이므로 cachedValue를 리턴한다(!).

 아직 cachedValue값이 제대로 들어가지 않았음에도 불구하고 cachedValue 값을 리턴하게 된다. 첫 번째 사례가 단순히 성능의 저하를 일으키는 거라면 두 번째 사례는 잘못된 캐시 값으로 인해 프로그램에 완전히 재앙을 일으킬 수도 있다.


### atomic과 mutex의 적절한 사용

 결론은 간단하다. 한 번에 하나의 메모리나 변수만 동기화해도 되는 경우에는 std::atomic을, 한 번에 여러 개의 변수나 메모리를 동기화해야 하는 경우에는 std::mutex를 쓰라는 것이다. 위 Widget 예제의 경우 다항식의 근을 구하는 예제처럼 std::mutex를 쓰는 것이 적절하다.

## 결론

 자신이 짜고 있는 클래스가 정말 절대 멀티 스레드 환경에서 동작할 리가 없음이 보장된다면 꼭 thread safe하게 짜지 않아도 될 것이다. 위와 같은 방법을 써서 thread safe하게 짜면 성능도 좀 떨어지고 move-only 클래스가 되는 등의 부수 효과가 발생하니까. 하지만 요즘은 거의 멀티 스레드 환경이고 절대 멀티 스레드 환경에 동작할 클래스가 아냐! 라고 확신할 수 있을만한 상황이 많지 않다. 그러니 원칙에 따라 const 멤버 함수는 꼭 thread safe하게 작성하도록 하자.