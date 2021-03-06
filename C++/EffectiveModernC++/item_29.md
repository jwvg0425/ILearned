# move가 별로 좋지 않다고 가정해라

C++11에서 추가된 move semantics는 그 설명만 들으면 굉장히 좋아보인다. move semantics를 이용하면 컨테이너 이동을 포인터 복사하는데 걸리는 시간만에 처리하고, 임시 객체의 복사를 굉장히 빠르게 수행할 수 있다. 하지만, 실제로는 이런 표현들은 굉장히 과장된 것이다. ***move 연산은 존재하지도, 비용이 싸지도, 사용되지도 않는다고 가정하라.***

 일단 move semantics를 지원하지 않는 타입에 대해서 먼저 살펴보자. 우선 C++ 98 시절에 작성한 타입들의 경우, C++11 기반의 컴파일러로 바꾸고 move를 쓰고 한다 해도 별로 성능 상의 이득을 못 볼 가능성이 있다. [[item 17]]에서 말했듯이 default move 생성자, 이동 대입 연산자는 복사 연산, 이동 연산, 파괴자중 어느 하나도 만들지 않았을 경우에만 자동으로 생성된다. 그렇지 않을 경우 move를 썼다하더라도 copy를 이용하기 때문에 어떤 성능 상의 이득도 얻을 수 없다.

 심지어 move 연산이 명확히 명시되었다하더라도 생각한 만큼 이득을 못 보는 경우도 있다. C++ 11 라이브러리의 컨테이너들은 모두 move 연산을 지원하지만, 몇몇 컨테이너들은 move 효율이 떨어지고 또 move 효율이 좋은 컨테이너라 해도 그 컨테이너 안에 들어가 있는 데이터들이 move 효율이 떨어질 수 있기 때문이다.

예를 들어, std::vector와 std::array를 비교해보자.

```C++
std::vector<Widget> vw1;

//vw1을 vw2로 move하는 건 상수 시간에 가능.
auto vw2 = std::move(vw1);
```

위와 같은 std::vector의 move 연산은 아래 그림과 같은 형태로 수행된다.

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/vector_move.PNG)  

vector는 내부적으로 데이터를 힙 공간에 할당한 후 그 영역에 대한 포인터를 관리하기 때문에, move는 단순히 vw2가 vw1이 가리키던 포인터를 가리키도록 만들면 되는 것이다. 즉, 포인터를 할당하는 상수시간 만에 move가 가능해서 효율이 굉장히 좋다.

반면에, std::array의 경우를 보자.

```C++
std::array<Widget, 10000> aw1;

//aw1을 aw2로 move하는 건 선형 시간이 걸린다.
//aw1의 모든 원소를 aw2로 move시켜야 함.
auto aw2 = std::move(aw1);
```

위 코드는 아래 그림과 같은 형태로 수행된다.

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/array_move.PNG)  

std::array는 데이터를 힙에 올려서 관리하는게 아니라 컨테이너 내부에 직접 관리하기 때문에 std::vector때처럼 포인터를 단순 교환하는 방식으로는 move를 수행할 수 없다. 결국 내부 컨테이너 원소 각각에 대해 move 연산을 수행하는 식으로 동작한다. move 효율이 copy보다 더 뛰어난 객체에 대한 컨테이너라면 어느 정도 더 빠를 순 있겠으나 결국 둘 다 선형 시간이므로 생각한 만큼의 고효율이 나오지는 않는다는 것이다.

반면에, std::string같이 상수 시간의 move와 선형 시간의 copy를 가짐에도 불구하고 move가 별로 고효율이라고 생각할 수 없는 특이한 녀석도 있다. 이는 SSO(small string optimization)이라고 불리는 최적화 때문인데, 간단히 말하자면 std::string은 보통 15글자 안쪽 정도의 짧은 문자열의 경우 힙에 할당하여 저장하지 않고 내부적으로 갖고 있는 버퍼에 저장한다. 따라서 이런 작은 크기의 string을 move하는 건 copy보다 더 빠르다고 할 수가 없게 되는 것이다.

하지만 그럼 길이가 긴 문자열에 대해서는 훨씬 효율이 좋은 것 아닌가? 라는 생각이 들 수도 있다. 그러나 괜히 SSO라는 기법이 있는게 아니다. 프로그램에서 쓰이는 대다수의 문자열들은 길이가 그다지 길지 않다. 만약 대다수 문자열들이 길이가 굉장히 길고 일부만 짧았다면 굳이 SSO 같은 구현을 하지 않았을 것이다. 그러니 대부분의 경우 std::string의 move 효율이 별로 뛰어날 수 없는 것이다(물론 자신이 짜는 프로그램이 굉장히 긴 길이의 문자열을 빈번하게 사용한다면 move의 효율이 뛰어날거라 생각해도 좋다).

심지어 빠른 move 연산을 지원하며, 확실히 move 연산이 일어날 상황이라 하더라도 copy가 일어나는 경우도 있다. [item 14](item_14.md)는 표준의 몇몇 컨테이너들이 강한 예외 안정성을 보장하는 연산을 제공함을 설명하고 있다. 그리고이 보장에 의거한 C++ 98 코드는 C++11로 업그레이드 했을 때도 깨지지 않아야하고, 그래서 move 연산이 어떤 예외도 던지지 않을 때에만 copy 연산이 move 연산으로 바뀔 수 있는 것이다. 따라서 move가 copy보다 훨씬 빠르고 문맥상 move가 일어날 수 있는 상황이라 하더라도 move가 noexcept로 선언되지 않았다면 컴파일러는 copy 연산을 수행하도록 만들 수도 있다는 것이다.

 따라서 C++11 의 move semantics가 별로 뛰어나다고 생각할 수 없는 상황은 요약하면 아래와 같을 것이다.

- ***이동 연산을 하지 않는 경우*** move 연산을 제공하지 않아 move하는데 실패한 경우다. 이 경우 copy가 일어난다.
- ***move가 더 빠르지 않은 경우*** 몇몇 객체들은 move 연산이 copy연산보다 딱히 빠르지 않다.
- ***move를 쓸 수 없는 경우*** 위에서 언급한 강한 예외 안정성에 대한 보장 때문에 move를 쓸 수 없는 경우 등이 있다.

move semantics가 별로 효율적이지 않은 또다른 경우가 있다.

- source object가 lvalue일 때. 몇가지 예외를 제외하고([item 25](item_25.md) 참고) move 연산의 source object는 반드시 rvalue여야한다.

이 장에서 ***move 연산은 존재하지도, 비용이 싸지도, 사용되지도 않는다고 가정하라.***라고 말했지만, 그건 일반적인 코드(template 등)를 짤 때 해당하는 이야기다. template 코드 등에서 move 연산은 실제 적용될 타입이 뭘지 알 수 없기 때문이다. 반면 명확하게 자기가 쓰는 타입과 move 지원 여부를 다 알고 있고 그 효율까지 알고 있는 경우에는 당연히 적절한 상황에 move를 쓰면 성능 향상을 얻을 수 있을 것이다.