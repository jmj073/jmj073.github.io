# Parametric Search에서 STL을?!

안녕하십니까. 이번 글에서는 parametric search 알고리즘에서 C++의 STL(Standard Template Library)에 있는 `lower_bound` 함수와, `upper_bound` 함수를 사용하는 방법에 대해 얘기해보고자 합니다.

## true 구간과 false구간

C++의 `lower_bound` 함수와 `upper_bound` 함수는 키(key)와 원소(element)를 비교하기위해 기본적으로 less predicate(`<` 연산자)를 사용합니다. 이때, `lower_bound`에서는 `원소 < 키`와 같은 방식으로 사용하고, `upper_bound`에서는 `키 < 원소`와 같은 방식으로 사용합니다.

따라서 less predicate에 대해 false인 구간과 true인 구간이 형성될 것입니다. `lower_bound`는 false 구간의 첫 번째 원소를 반환하고, `upper_bound`는 true 구간의 첫 번째 원소를 반환합니다. 따라서 `lower_bound`는 키 "이상"의 원소를 반환하고, `upper_bound`는 키 "초과"의 원소를 반환한다고 이해할 수 있습니다.

## `lower_bound`의 시그니처

어짜피, `lower_bound`나 `upper_bound`나 함수의 시그니처는 같으므로, `lower_bound` 함수의 시그니처만 살펴보겠습니다. 다음은 시그니처를 간추려서 나타낸 것입니다.

```C++
Iter lower_bound(Iter first, Iter last, const T& value, Compare pred);
```

### C++의 Iterator

iterator는 나열 가능한 데이터 구조(C++의 용어를 사용하자면 container)를 위한 공통 인터페이스입니다. C++은 여기에 포인터를 차용했습니다! 솔직히 말하자면 C++의 iterator는 거지같습니다. 하지만 포인터 연산자를 사용하기 때문에 배열을 위한 iterator를 구현할 필요가 없죠. 왜냐하면 그냥 배열의 포인터를 iterator로 사용하면 되기 때문입니다.

STL container들의 `begin` `end` 함수들은 iterator를 반환합니다. iterator는 `[begin, end)` 형식입니다. 즉, begin에 대해 닫힌 구간이고, end에 대해 열린 구간입니다. 즉, begin은 첫 번째 원소를 가리키지만, end는 마지막 원소 다음 것을 가리킵니다.

`lower_bound`를 vector의 모든 원소에 대해 사용하고 싶다면 `lower_bound(vec.begin(), vec.end())`와 같이 사용하면 됩니다. `pred`를 지정하지 않으면, 기본적으로 less predicate가 사용됩니다.

iterator가 포인터라는 점에서, parametric search를 위한 트릭을 사용할 수 있습니다. 만약 해 공간이 1 ~ 100이면 `lower_bound((char*)1, (char*)100)`과 같이 사용할 수 있습니다. end에 대해 열린 구간이라고 했지만, bound 함수는 end를 반환할 수도 있기 때문에 101이 아니라 100을 사용합니다. segmentation fault가 나지 않을까 의문이 생기실 수 있는데, 해당 iterator(포인터)를 읽거나 쓰지만 않으면 아무 문제가 생기지 않습니다.

### predicate

관심있는 것은 pointer iterator가 가리키는 값 아니라, pointer의 주소 그 자체이므로, predicate는 원소를 참조로써 받아야 합니다. 다음과 같이 말이죠.

```C++
bool predicate(char& element, int key);
```

값으로 받으면 복사를 위해 메모리를 읽게 되므로 segmentation fault가 납니다! 원하는 값은 참조 연산자를 통해 `&element`와 같이 얻을 수 있습니다.

bound 함수들을 parametric search에 사용하는 것의 핵심은 predicate를 사용하는 것입니다! predicate는 원소를 받으므로, 해당 원소를 잘 가공하여 parametric search에 맞는 값으로 바꾸면 됩니다.

## 백준 문제를 풀어보자

백준 알고리즘 풀이 사이트의 1654번 문제인 "랜선 자르기"를 예로 들겠습니다.

입력으로는, 가지고 있는 랜선의 개수 K, 필요한 랜선의 개수 N, 그리고 K 줄에 걸쳐 가지고 있는 각 랜선의 길이가 입력됩니다.

해결해야 하는 것은 길이가 똑같은 랜선을 N개 이상 만들면서, 이를 만족할 때의 랜선의 최대 길이를 구하는 것입니다.

일단 코드는 다음과 같습니다.

```C++
#include <cstdio>
#include <algorithm>

using namespace std;

int lan[10'000];

int main() {
    int K, N;
    scanf("%d %d", &K, &N);

    for (int i = 0; i < K; ++i)
        scanf("%d", lan + i);

    int ans = upper_bound((char*)0, (char*)(1LL << 32), N,
        [&] (int n, char& i) {
            int l = &i - (char*)0, cnt{};
            for (int i = 0; i < K; ++i) {
                cnt += lan[i] / l;
            }
            return cnt < n;
        }) - (char*)1;

    printf("%d", ans);
}
```

`upper_bound`는 false 구간 뒤에 true구간이 오며, true 구간의 첫 번째 원소를 반환합니다. 즉, `cnt`가 `n` 보다 작아지는 첫 지점을 반환합니다. 따라서 거기서 1을 빼주면 그 길이는 `cnt >= n`을 만족할 것입니다.

### lower를 사용할 것인가, upper를 사용할 것인가, predicate는 어떻게 작성할 것인가

위의 예시에서 지정된 iterator의 주소는 오름차순의 등차수열입니다만, 해당 값을 가공한 cnt 값은 내림차순입니다. 이게 참 코드를 작성할 때 햇갈리는 부분입니다. `upper_bound`는 `키 < 원소`라고 했었습니다. 하지만 위의 예시에서는 `원소 < 키`(`cnt < n`)를 사용합니다! 이게 어떻게 된 일이죠? 또한 애초에 왜 `lower_bound`가 아니라 `upper_bound`를 사용한 것일까요?

사실 먼저 결론을 말하자면, `lower_bound`를 사용하든, `upper_bound`를 사용하든 상관 없습니다. 또한, lower, upper, 이상, 초과에 대해서도 생각할 필요가 없습니다. 그저 true와 false구간에 대해서만 생각하면 됩니다.

지금 원하는 것은 `cnt >= n`을 만족하면서도 가장 긴 길이입니다. `cnt >= n`을 만족하는 구간과, 만족하지 않는 구간으로 나눠볼 수 있습니다. 만족하는 구간은 앞쪽에 오고, 만족하지 않는 구간은 뒷쪽에 옵니다. 구하고 싶은 것은 만족하는 구간에서 가장 큰 것입니다. 만족하는 구간의 첫 번째 값은 가장 작은 값입니다. 가장 큰 것은 만족하지 않는 구간에서 1을 뺀 것이죠.

옛날의 제가 왜 lower 대신 upper를 쓴 것인지는 모르겠지만, 현재 문제에서는 upper가 인지적 부담이 덜한것 같습니다.

이번 글은 여기서 마치겠습니다.