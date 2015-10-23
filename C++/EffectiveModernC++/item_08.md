# 0과 NULL보다는 nullptr를 써라

0, NULL은 포인터 타입이 아니다. 그래서 C++98에서는 함수 오버로딩에서 포인터를 인자로 받는 함수에 null pointer를 넘길 때 애로사항이 꽃피는 경우가 많았다.

```C++
void f(int);
void f(bool);
void f(void*);

f(0); //f(void*)가 아니라 f(int)가 호출.

f(NULL); //보통 f(int)가 호출됨. f(void*)가 호출될 일은 절대 없다.
```

NULL은 컴파일러마다 조금씩 다르긴 하지만 보통 0L(long 타입의 0) 또는 0으로 선언되어 있다. 그래서 "NULL(null pointer)로 함수를 호출하겠다"라는 프로그래머의 의도와는 다르게 컴파일러는 실제 의미인 0L 또는 0을 인자로 넘긴 것으로 받아들여 f(int)를 호출하게 되는 것이다. 그래서 C++98에서는 정수 타입과 포인터를 같이 오버로딩하는 걸 피하는게 권장된다. C++11에서 도입된 nullptr을 이용하면 이런 문제를 해결할 수 있다(하지만 여전히 C++98과 마찬가지로 정수 타입과 포인터를 같이 오버로딩은 하지 않는 게 좋다).

 nullptr의 장점은 이게 정수 타입이 아니라는 것이다. 사실 이건 포인터 타입도 아니다. 그럼에도 불구하고 nullptr는 모든 타입의 포인터처럼 생각할 수 있다. nullptr의 실제 타입은 std::nullptr_t인데, std::nullptr_t는 모든 포인터 타입과 암시적으로 변환이 가능하다. 이런 특징 때문에 nullptr을 모든 타입의 포인터처럼 쓸 수 있는 것이다. 그래서 위의 C++98에서의 오버로딩 문제도 nullptr을 쓰면 쉽게 해결된다.

```C++
f(nullptr): // f(void*)가 호출된다.
```

nullptr의 장점은 이것만이 아니다. 아래 코드를 한 번 보자.

```C++
auto result = findRecord( x, y, z );

if(result == 0)
{
}
```

이 코드에서 result 타입이 뭔지 쉽게 판단할 수 있을까? findRecord가 리턴하는 값이 포인터 타입인지 아니면 정수 타입인지 구분하기 쉽지 않다. 0이 null pointer의 의미로도 사용되기 때문이다. 반면에 nullptr을 사용한 아래 코드를 보자.

```C++
auto result = findRecord( x, y, z );

if(result == nullptr)
{
}
```

누가 봐도 result가 포인터 타입이라는 것은 자명하다.

템플릿에서도 nullptr가 많은 장점을 갖고 있다. 적절한 뮤텍스가 락이 걸렸을 때만 몇몇 함수를 호출할 수 있게끔 하고 싶다고 하자. 또 각각의 함수들은 서로 다른 타입의 포인터를 인자로 받는다.

```C++
// 이 함수들은 적절한 뮤텍스가 락이 걸렸을 때만 호출될 수 있다.
int    f1(std::shared_ptr<Widget> spw);
double f2(std::unique_ptr<Widget> upw);
bool   f3(Widget* pw);
```

이 함수들을 null pointer를 이용해서 호출하는 코드는 아래와 같은 형태가 될 것이다.

```C++
std::mutex f1m, f2m, f3m; //f1,f2,f3을 위한 mutex

// C++ 11 방식의 typedef. item 9 참고
using MuxGuard = std::lock_guard<std::mutex>;

...

{
    MuxGuard g(f1m); //f1을 위한 mutex 락
    auto result = f1(0); //0(null pointer)f1로 전달
}   //mutex 락 해제

{
    MuxGuard g(f2m); //f2를 위한 mutex 락
    auto result = f2(NULL); // NULL(null pointer) f2로 전달
}   //mutex 락 해제

{
    MuxGuard g(f3m); //f3을 위한 mutex 락
    auto result = f3(nullptr); // nullptr f3으로 전달
}   //mutex 락 해제
```

자세히 보면 f1을 쓰든 f2를 쓰든 f3을 쓰든 전체적인 코드의 패턴이 똑같다는 것을 알 수 있다. 뮤텍스에 락을 걸고, 함수를 호출하고, 락을 풀고. 이런 코드의 반복은 굉장히 지저분하고 유지보수하기 힘들어지니, 이 패턴을 하나의 템플릿으로 바꿔보자.

```C++

//C++ 11 스타일
template<typename FuncType,
         typename MuxType,
         typename PtrType>
auto lockAndCall(FuncType func,
                 MuxType& mutex,
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);
    return func(ptr);
}

//C++ 14 스타일
template<typename FuncType,
         typename MuxType,
         typename PtrType>
decltype(auto) lockAndCall(FuncType func,
                           MuxType& mutex,
                           PtrType ptr)
{
    MuxGuard g(mutex);
    return func(ptr);
}
```

이제 이 템플릿을 이용해서 훨씬 깔끔하게 함수를 호출할 수 있을 것이다.

```C++
auto result1 = lockAndCall(f1, f1m, 0);
auto result2 = lockAndCall(f2, f2m, NULL);
auto result3 = lockAndCall(f3, f3m, nullptr);
```

하지만 이 때, 첫 번째와 두 번째 호출은 컴파일 에러가 발생한다. lockAndCall의 마지막 인자로 0이 전달되면 PrtType은 int가 되는데, 문제는 lockAndCall 함수 내부의 func(ptr)을 호출하는 부분에서 발생한다. 이 때 func함수는 f1함수고, f1함수는 인자로 std::shared_ptr<Widget>타입을 받는다. 그러나 0(int)을 std::shared_ptr<Widget>타입으로 변환할 수는 없다. 그래서 여기서 컴파일 에러가 발생하게 되는 것이다. 두 번째 호출 역시 거의 동일한 이유때문에 컴파일 에러가 발생한다(정수 계열의 타입을 std::unique_ptr<Widget>타입으로 변환할 수 없다)

 반면에 nullptr를 집어넣는 건 아무런 문제도 발생하지 않는다. lockAndCall 템플릿 함수에 nullptr이 인자로 넘어가면, ptr의 타입(PtrType)은 std::nullptr_t로 추론된다. 그런데 위에서도 말했듯이 std::nullptr_t는 모든 종류의 포인터와 암시적으로 변환이 가능하다. 따라서 함수 인자로 넘어갈 때 std::shared_ptr<Widget>이나 std::unique_ptr<Widget>등등의 타입과 암시적으로 변환이 일어나서 함수 호출할 때 아무런 문제를 일으키지 않는 것이다.

nullptr을 쓰면 이렇듯 코드 가독성도 좋아지고, 오버로딩, 템플릿 등에서 골치 아픈 일을 획기적으로 줄여줄 수 있는 장점을 갖고 있으니 null pointer를 써야 하는 부분에서는 0이나 NULL을 쓰지말고 nullptr을 쓰자.