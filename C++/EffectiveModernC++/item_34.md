# std::bind보단 람다

C++11에서 거의 대부분의 경우 std::bind보다는 람다 표현식을 쓰는 것이 더 좋은 선택이다. 람다의 기능이 대폭강화된 C++14 역시 마찬가지고. 그 이유에는 여러 가지가 있지만, 우선 가장 첫번째는 가독성이다. std::bind를 이용해 작성된 코드보다 람다 표현식으로 작성된 코드의 가독성이 더 뛰어나다.

```C++
using Time = std::chrono::steady_clock::time_point;

enum class Sound { Beep, Siren, Whistle };

using Duration = std::chrono::steady_clock::duration;

//타임 t가 되면 d 시간동안 s 소리를 냄
void SetAlarm(Time t, Sound s, Duration d);
```

이런 함수가 있을 때, 프로그램 수행중 특정 상황이 되면 해당 시점으로부터 1시간 뒤에 30초동안 소리가 나도록 알람을 설정하고 싶다고 하자. 어떤 소리가 날 지를 인자로 받는 람다 함수를 이용하면 쉽게 해결할 수 있다.

```C++
//C++11
auto setSoundL =
    [](Sound s)
    {
        using namespace std::chrono;

        setAlarm(steady_clock::now() + hours(1),
                 s,
                 seconds(30));
    };

//C++14
auto setSoundL =
    [](Sound s)
    {
        using namespace std::chrono;
        using namespace std::literals;

        setAlarm(steady_clock::now() + 1h,
                 s,
                 30s);
    };
``` 

위 코드를 std::bind를 이용해서 짜려고 하면 일단 아래와 같은 형태로 접근하게 될 것이다(올바른 코드가 아니다. 일단 이렇게 접근해보자).

```C++
using namespace std::chrono;
using namespace std::literals;

using namespace std::placeholders;

auto setSoundB =
    std::bind(setAlarm,
              steady_clock::now() + 1h, // 잘못된 코드!
              _1,
              30s);
```

여기서 _1은 placeholder로, 나중에 setSoundB의 operator()를 호출할 때 들어가는 인자에 순서대로 이름을 붙인 것이라고 생각하면 된다. 이 placeholder도 std::bind의 가독성을 떨어뜨리는 요인중의 하나지만(위 setSoundB의 선언문만 보고 이게 어떤 타입의 인자를 받는지 알기가 힘들다. setAlarm의 정의부까지 봐야만 알 수 있음), 그것보다 먼지 일단 위의 코드는 잘못된 코드다. steady_clock::now() + 1h는 setSoundB 함수 객체가 실행되는 순간에 수행되는게 아니라, 위 bind object가 생성될 때 계산되고 그 값을 내부적으로 저장하고 있게 된다. 즉 우리가 원하는 의도와는 상당히 다른 동작을 하는 함수 객체가 만들어지는 것이다. 이 문제를 해결하려면 steady_clock::now() + 1h 구문의 수행을 setAlarm 함수가 호출되는 시점까지 미뤄야하므로, bind를 중첩해서 써야만 한다.

```C++
auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<steady_clock::time_point>(), 
                              steady_clock::now(), 1h),
              _1,
              30s);
```

척 봐도 훨씬 복잡해진다. 이 정도만 봐도 람다가 훨씬 낫다는 생각이 들 것이다.

 std::bind에는 또다른 문제점이 있다. 바로 오버로딩과 관련된 문제이다. 만약 setAlarm 함수가 오버로딩된 함수라고 하자.

```C++
enum class Volume { Normal, Loud, LoudPlusPlus };

void setAlarm(Time t, Sound s, Duration d, Volume v);
```

위와 같이 알람의 크기까지 받는 새로운 setAlarm 함수가 오버로딩되었다고 하자. 람다는 이전과 다름없이 잘 동작하지만, std::bind 구문은 문제를 일으킨다. 앞선 항목에서도 다뤘지만 오버로딩된 함수의 이름만 갖고는 어떤 setAlarm을 선택해야할지 알 수가 없기 때문이다. 이를 해결하기 위해서는 setAlarm 함수를 적절하게 캐스팅하는 수 밖에 없다.

```C++
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);

auto setSoundB =
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm),
              std::bind(std::plus<>(), //c++ 14에는 타입인자 생략 가능
                        steady_clock::now(),
                        1h),
              _1,
              30s);
```

이런 특징은 std::bind와 람다 표현식 사이에 새로운 차이를 만들어낸다.

```C++
//setAlarm 함수의 body가 이 내부에 inline될 수 있음
setSoundL(Sound::Siren);
```

람다를 사용할 경우 setSoundL의 operator() 내부에는 단순히 setAlarm 함수의 호출 하나 뿐이다. 따라서 이 함수의 호출은 내부에 inline될 수 있다.

```C++
//이 경우 아마 inline되지 않을 것
setSoundB(Sound::Siren);
```

반면, setSoundB의 경우 setAlarm에 대한 포인터를 인자로 넘겼으므로 내부에서는 해당 함수의 주소를 타고 가서 호출해야만 하므로 컴파일러가 inline으로 만들기 힘들다. 이 때문에 람다가 std::bind보다 좀 더 빠른 코드를 생성할 가능성이 높아진다.

또 다른 예제를 살펴보자. 아래 코드는 어떤 변수 val이 지역 변수 lowVal과 highVal 사이에 있는지 확인하는 함수를 람다로 작성한 예제 코드다.

```C++
//C++11
auto betweenL =
    [lowVal highVal]
    (int val)
    { return lowVal <= val && val <= highVal; };

//C++14
auto betweenL =
    [lowVal, highVal]
    (const auto& val)
    { return lowVal <= val && val <= highVal; };
```

아래는 같은 내용의 코드를 std::bind로 짠 코드다.

```C++
using namespace std::placeholders;

//C++11
auto betweenB = 
    std::bind(std::logical_and<bool>(),
              std::bind(std::less_equal<int>(),lowVal, _1),
              std::bind(std::less_equal<int>(), _1, highVal);

//C++14
auto betweenB = 
    std::bind(std::logical_and<>(),
              std::bind(std::less_equal<>(),lowVal, _1),
              std::bind(std::less_equal<>(), _1, highVal);
```

C++11이건 14건 람다 표현식의 경우가 코드의 길이도 짧고 유지보수성도 더 뛰어나리라는 것은 누구나 느낄 수 있을 것이다.

std::bind는 람다에 비해 코드의 동작을 지식이 없으면 추측하기 힘들다. 아래 코드를 보자.

```C++
enum class CompLevel { Low, Normal, High };

Widget compress(const Widget& w,
                CompLevel lev);

Widget w;

using namespace std::placeholders;

auto compressRateB = std::bind(compress, w, _1);
```

여기서 w를 std::bind에 넘길 때 w는 compressRateB 객체 안에 저장되었다가 나중에 compress 함수를 호출할 때 쓰일 것이다. 이 때, w는 레퍼런스로 저장될까, 값으로 저장될까? 이 차이는 크다. 레퍼런스로 저장될 경우 외부에서 일어나는 데이터의 변화에 영향을 받지만 값으로 저장될 경우 영향을 받지 않기 때문이다. 일단 정답은 **값으로 저장된다**이다. 하지만 이게 값으로 저장된다는 걸 알려면 그냥 std::bind가 원래 그렇게 동작한다는 것을 외우고 있어야만 한다. 반면 람다의 경우는, 값으로 캡쳐하는지 레퍼런스로 캡쳐하는지 명확하게 명시되어 있다.

```C++
auto compressRateL =
    [w](CompLevel lev)
    { return compress(w, lev); };
```

이 때 캡쳐에서 w라고 적혀 있으니 이건 값으로 캡쳐될 것이다(&w 라면 레퍼런스). 이런 명시성은 함수의 매개변수에 있어서도 동일하다.

```C++
//인자는 값으로 전달된다
compressRateL(CompLevel::High);
```

반면, std::bind의 경우, 함수 호출시 넘어가는 인자가 값으로 넘어가는지 레퍼런스로 넘어가는지 알기 힘들다. 역시 std::bind가 원래 어떻게 동작하는지 외우고 있어야만 한다(bind object에 넘어가는 모든 인자들은 레퍼런스로 넘어간다. perfect forwarding을 하기 때문).

```C++
//값? 레퍼런스? 코드만 보고 알 수 있는 방법이 없음.
compressRateB(CompLevel::High);
```

결국 람다에 비교하면 std::bind를 사용하는 건 가독성 면이나, 표현 능력 면이나, 효율성 면이나 람다보다 나은 점이 하나도 없는 것이다. 특히 C++ 14부터는 std::bind를 람다 대신에 쓸 만한 합리적인 상황이 단 하나도 존재하지 않는다. C++11의 경우 람다 대신에 std::bind를 써야할 만한 상황이 2가지 정도 있다.

- **Move Capture** C++11의 람다는 move capture를 지원하지 않는다. 이를 흉내내기 위해 람다와 std::bind를 섞어써야할 상황이 있다. 하지만 C++14부터는 새로 추가된 기능인 init capture를 사용하면 되기 때문에 std::bind를 쓸 필요가 없다. 자세한 내용은 [item 32](item_32.md) 참고.

- **Polymorphic function object** std::bind의 경우 함수 호출을 perfect forwarding을 통해 수행하기 때문에, 어떤 종류의 인자든 받아들일 수 있다. 이건 템플릿화된 함수 호출 연산자와 객체를 bind할 때 유용하다.

 ```C++
class PolyWidget
{
public:
    template<typename T>
    void operator()(const T& param);
};

PolyWidget pw;
auto boundPW = std::bind(pw, _1);
```

 이렇게 boundPW를 만들면, 서로 다른 타입의 인자를 넘길 수 있다.

 ```C++
boundPW(1930); //PolyWidget::operator()에 int 넘김

boundPW(nullptr); //PolyWidget::operator()에 nullptr 넘김

boundPW("Rosebud"); //PolyWidget::operator()에 string literal 넘김
```

 C++11의 람다에서는 이런 기능을 구현할 방법이 없다. 반면 C++14에서는 auto 매개변수를 가진 람다를 허용하기 때문에 std::bind를 쓸 필요가 없다.

 ```C++
auto boundPW = [pw](const auto& param)
               { pw(param); };
```
