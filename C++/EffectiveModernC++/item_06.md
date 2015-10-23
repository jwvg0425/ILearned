# auto의 의도치 않은 타입추론 대처하기

[item 5](item_05.md)에서 auto가 굉장히 다양한 장점을 갖고 있다고 이야기했지만, 가끔씩 auto가 의도치 않은 방향으로 동작할 때가 있다. 가장 간단한 예로 `std::vector<bool>`이 있다.

```C++
std::vector<bool> vb = {true, true, true};
auto vb2 = vb[2]; //vb2의 타입은 bool이겠지?
```

안타깝지만 vb2의 타입은 bool이 아니다. std::vector<bool>의 경우 내부적으로 효율성을 위해 1비트당 하나의 값을 저장한다(bool은 1비트면 저장 가능하므로). 따라서 vb[2]같은 경우 어떤 지점에서 2비트 뒤의 주소 같은 건 가져올 수 없으므로 bool& 타입을 리턴하는 것이 아니라 std::vector<bool>::reference 타입을 리턴한다. 이 타입은 bool 타입으로의 변환을 지원하기 때문에 명시적으로 bool 타입으로 지정해서 대입받을 경우에는 문제가 생기지 않는다. 하지만 위와 같이 auto를 쓸 경우 의도치않은 타입 추론이 일어나버리는 것이다. 이런 std::vector<bool>::reference 타입 같은 걸 proxy class라고 한다. 이런 클래스는 어떤 다른 타입의 동작을 구현하고 모방하기 위해 만들어지는 클래스다.

 이런 식으로 외부에 잘 드러나지 않는 proxy class들이 auto와 얽히면 의도치 않은 타입 추론을 일으키곤 한다. 하지만 이게 auto 자체가 문제인게 아니라, auto가 단지 우리가 의도한 타입을 추론하지 못하는 게 문제인 것이다. 따라서 이런 경우auto가 어떻게 추론해야할 지 명시적으로 타입을 초기화해주면 된다.

## 명시적 타입 초기화 기법

 C++계의 공인된 초고수 Scott Meyers는 이 기법을 명시적 타입 초기화 기법(the explicitly typed initializer idiom)이라는 세련된 이름으로 부른다. 이름은 거창한데 내용은 사실 별 거 없고 간단하다. static_cast를 이용해 proxy class같이 의도치 않은 타입 추론이 일어나게 만드는 녀석들을 어떻게 추론해야할 지 친절히 가르쳐주는 것이다.

```C++
std::vector<bool> vb;
auto vb2 = static_cast<bool>(vb[2]); // 제대로 bool로 추론!
```

static_cast에 의해 bool로 캐스팅된 vb[2]를 저장하게 되므로 auto도 프로그래머의 의도대로 bool 타입이 되는 것이다. 따라서 이런 심히 곤란한 경우에는 명시적 타입 초기화 기법을 이용해 auto의 타입을 정해주자.

이외에도 이 기법을 사용하면 프로그래머의 의도가 확실히 코드에 드러난다.

```C++
double f();

float a = f();
auto b = static_cast<float>(f());
```

a,b의 두 초기화를 비교해보자. 둘의 동작은 같지만 보는 사람 입장에서 b의 초기화가 의도가 더 분명하다. a의 경우 f의 원래 선언을 모르는 이상 f가 원래 리턴값이 float인지 아니면 double인데 float으로 대입받고 싶어서 float타입의 변수에 받은 건지 알기 힘들다. 하지만 b의 경우 일단 f()의 리턴 타입이 뭐가 됐든 간에 b는 그 값을 float 타입으로 바꿔서 받고 싶다는 의사가 분명하다. 따라서 코드를 보는 사람이 좀 더 이 코드의 의도를 파악하기가 쉬울 것이다.