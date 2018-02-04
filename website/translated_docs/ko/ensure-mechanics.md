---
title: 모델 및 메커니즘
---
dep에는 많은 개별 컴포넌트와 움직이는 부품들이 많지만, 이 모든 부분들이 중앙 모델 주위를 돌고 있다. 이 문서는 그러한 모델을 설명하고, 그 모델의 맥락안에서 dep의 주요 메커니즘들을 탐구하겠다.

## 상태 및 흐름

dep은 패키지 관리자와 상호 작용하는 디스크상의 상태를 분류하고 구성하는 모델로서 "4개의 상태를 갖는 시스템"이라는 아이디어를 중심으로 돌아 간다. 4개의 상태 모델이 갖고 있는 많은 원칙들은 이미 존재하는 패키지 관지자들로 부터 파생되긴 했지만, [이 (긴) 문서](https://medium.com/@sdboyer/so-you-want-to-write-a-package-manager-4ae9c17d9527)를 통해 일관되고 일반적인 모델로 처음으로 뚜렷이 제시되었다.

간단히, 4개의 상태는:

1. [현재 프로젝트의](glossary.md#current-project) 소스코드.
2. [종속 명세서](glossary.md#manifest) - 현재 프로젝트의 종속성 요구사항을 기술하는 파일. dep안에서는 [`Gopkg.toml`](Gopkg.toml.md)파일을 말한다.
3. [잠금](glossary.md#lock) - 종속성 그래프를 전이적으로 완전하고(transitively-complete), 재현할 수 있도록(reproducible) 표현하는 파일. dep안에서는 [`Gopkg.lock`](Gopkg.lock.md) 파일을 지칭한다.
4. 종속성의 소스코드. 현재 dep의 디자인에서는, `vendor` 디렉토리를 말한다.

이 4개의 상태를 다음과 같이 시각적으로 나타낼 수 있다:

![dep의 4가지 상태](assets/four-states.png)

### 기능적 흐름

dep를 이러한 상태들 사이의 관계에 대한 단방향, 기능적 흐름을 부과하는 시스템으로 생각하는 것이 바람직 하다. 이러한 기능들은 위에 언급된 상태들을 왼쪽에서 오른쪽으로 이동하는 입력과 출력으로 취급한다. 특별히, 2개의 기능이 있는데:

* *해결 기능*은 현재 프로젝트내 있는 임포트의 집합과 `Gopkg.toml`내 규칙들을 입력으로 받아서, 전이적으로 완전하고(transitively-complete), 불변하는 종속성 그래프를 출력으로 반환해서 `Gopkg.lock`내에 정보로 저장한다.
* *벤더링 기능*은 `Gopkg.lock`내에 있는 정보를 입력으로 받아서 컴파일러가 잠금(lock)에 지정된 버전을 사용하도록 소스 파일의 디스크상의 배열을 보장한다.

이 두 기능은 다음과 같이 시각적으로 나타낼 수 있다:

![dep의 두 가지 주요 기능](assets/annotated-func-arrows.png)

이것은 `dep ensure`이라고 하는 전형적인 흐름으로, `Gopkg.toml`이 이미 존재할 때 사용된다. 프로젝트가 아직 `Gopkg.toml`를 가지고 있지 않는 경우는, `dep init`을 사용해 생성시킬 수 있다. 이때 본질적인 흐름은 같지만, 다른 입력이 사용된다: `dep init`은 이미 존재하는 `Gopkg.toml`을 읽지 않고, 사용자의 GOPATH와 [다른 툴의 메타데이터]()로 부터 추론한 데이터를 통해 새롭게 만들어 낸다. (다시 말해면, `dep init`은 다른 방식으로 구성된 종속성을 자동적으로 이전시켜 주는 것이다)

이 다이어그램은 코드와 직접적으로 일치하는 면이 있다. 해결 기능은 실제로 생성자와 메서드로 갈라진다 - 우선 [`Solver`](https://godoc.org/github.com/golang/dep/gps#Solver) 타입을 만들고 난 다음 `Solve()` 메서드를 호출한다. 생성자의 입력은 언뜻 봐도 알 만한 [`SolveParameter`](https://godoc.org/github.com/golang/dep/gps#SolveParameters)안에 포장되어 있다.

```go
type SolveParameters struct {
  RootPackageTree pkgtree.PackageTree // Parsed project src; contains lists of imports
  Manifest gps.RootManifest // Gopkg.toml
  ...
}
```

벤더링 기능은 [`gps.WriteDepTree()`](https://godoc.org/github.com/golang/dep/gps#WriteDepTree)이다. 받아 들이는 인자의 수가 꽤 많지만 그 중에 지금 관련있는 것은 [`gps.Lock`](https://godoc.org/github.com/golang/dep/gps#Lock)으로 `Gopkg.lock`내 데이타의 추상화된 형태를 나타낸다.

4개의 상태를 갖는 시스템이라는 개념과 그것을 관통하는 기능적 흐름이 dep의 모든 동작들을 제작하는데 기초인 것이다. dep의 메커니즘을 이해하고자 하면 이 모델을 염두에 두어야 한다.

### 동기화 상태 유지

dep의 디자인 목표중 하나는 이 두 "기능들"이 하는 일과 각 해당 결과안에서 야기하는 변화를 최소화하는 것이다. (참고: 현재 "최소화"는 비용함수를 통해 공식적으로 정의된 것은 아니다.) 결과적으로, 두 기능 모두 기존의 결과를 미리 들여다보고 실제로 수행해야 할 작업을 이해한다.

* The solving function checks the existing `Gopkg.lock` to determine if all of its inputs (project import statements + `Gopkg.toml` rules) are satisfied. If they are, the solving function can be bypassed entirely. If not, the solving function proceeds, but attempts to change as few of the selections in `Gopkg.lock` as possible. 
  * WIP: The current implementation's check relies on a coarse heuristic check that can be wrong in some cases. There is a [plan to fix this](https://github.com/golang/dep/issues/1496).
* The vendoring function hashes each discrete project already in `vendor/` to see if the code present on disk is what `Gopkg.lock` indicates it should be. Only projects that deviate from expectations are written out. 
  * WIP: the hashing check is generally referred to as "vendor verification," and [is not yet complete](https://github.com/golang/dep/issues/121). Without this verification, dep is blind to whether code in `vendor/` is correct or not; as such, dep must defensively re-write all projects to ensure the state of `vendor/` is correct.

물론, 각 기능들이 미리 엿보기를 통해 기존의 결과가 이미 정확하다는 발견을 할 가능성도 있다. 그렇다면 아무런 작업을 할 필요도 없다. 어느 쪽이 든, 각 기능이 완료되면, 변경사항이 있던 없던 출력은 입력에 관하여 정확하다고 확신할 수 있다. 다른 말로 표현하면, 입력과 출력이 "동기화 되었다." 실제로, 동기화 됨은 dep의 "확인된 좋은 상태"이다; `dep ensure`가 (플래그 없이) 종료코드 0으로 끝나는 경우에, 프로젝트내 모든 4개의 상태는 동기화되었다.

## `dep ensure` 플래그와 동작 변화

`dep ensure`의 다양한 플래그는 해결 기능과 벤더링 기능의 동작에 영향을 준다 - 심지어 실행 여부에 영향을 줄 수도 있다. 일부 플래그들은 일시적이나마 프로젝트를 소폭 동기화에서 벗어나게 할 수도 있다. dep의 기본 모델을 염두에 두고 이러한 영향들을 생각하게 되면 가장 빠르게 이해할 수 있다.

### `-no-vendor` 그리고 `-vendor-only`

이 두 플래그는 상호 배타적이며 `dep ensure`의 두 기능중 어느 것이 실제로 실행될 지 결정한다. `-no-vendor`를 전달하게 되면 해결 기능만이 실행되어 새로운 `Gopkg.lock`이 만들어 지고; `-vendor-only`는 해결 기능를 무시하고 벤더링 기능만을 실행하여 `vendor/` 폴더를 기존의 `Gopkg.lock`를 이용해 다시 채운다. 

![단 하나 또는 다른 dep의 기능을 실행 하는 플래그](assets/func-toggles.png)

또한 `-no-vendor`를 전달함은 해결 기능를 무조건적으로 실행하고, 모든 입력이 만족하는지 `Gopkg.lock`를 대상으로 일반적으로 수행하는 사전 검사를 무시하는 영향을 추가로 준다.

### `-add`

`dep ensure -add`의 일반적인 목적은 dep 그래프에 새로운 종속성을 쉽게 도입할 수 있도록 하기 위함이다. `-update`가 [소스 루트](glossary.md#source-root), (예. `github.com/foo/bar`) 제한되는 반면에, `-add`는 어떠한 패키지 임포트 경로도 인자로 받을 수 있다 (예. `github.com/foo/bar` 혹은 `github.com/foo/bar/baz`).

개념적으로, `-add`가 일으킬 수 있는 가능한 변화는 두가지이다. 어떠한 `dep ensure -add`도 이 둘중 적어도 하나는 실행한다.

1. 새로운 종속성(들)을 포함하는 새로운 `Gopkg.lock`를 생성시키기 위해 해결 기능을 실행한다.
2. `Gopkg.toml`에 버전 제약 조건을 첨가한다.

이는 `dep dep -add`에 대한 두 가지 전제 조건을 의미하며, 적어도 하나는 충족되어야 한다.

1. 주어진 임포트 경로는 현재 프로젝트의 임포트 문에 없거나 `Gopkg.toml`의 `required` 리스트에 없다.
2. 주어진 임포트 경로에 상응하는 프로젝트 루트에 대한 `[[constraint]]` 절(stanza)이 `Gopkg.toml`내에 없다.

버전 제약 조건을 명시하는 것 또한 가능하다.

```bash
$ dep ensure -add github.com/foo/bar@v1.0.0
```

버전 제약 조건이 인자에 포함되지 않은 경우는 해결 기능은 가장 최신의 버전을 선택한다. (일반적으로, 최신 semver 릴리즈, 혹은 semver 릴리즈가 없는 경우에는 디폴트 브랜치) 해결 기능이 성공적으로 실행되면, 인자에 의해 명시된 버전이나, 버전이 없는 경우 해결 기능이 선택한 버전이 `Gopkg.toml`에 첨가된다.

입력 및 현재 프로젝트 상태의 다양한 차이로 인해 발생하는 동작의 차이는 아래 표가 가장 잘 보여 준다.

| `dep ensure -add`의 인자       | `Gopkg.toml`내 `[[constraint]]`가 있는 경우 | 임포트나 `required` | 결과                                                                     |
| --------------------------- | ------------------------------------- | --------------- | ---------------------------------------------------------------------- |
| `github.com/foo/bar`        | 아니요                                   | 아니요             | `Gopkg.lock`와 `vendor/`에 일시적으로 첨가되고; 유추된 버전 제약 조건이 `Gopkg.toml`에 추가된다. |
| `github.com/foo/bar@v1.0.0` | 아니요                                   | 아니요             | `Gopkg.lock`와 `vendor/`에 일시적으로 첨가되고; 명시된 버전 제약 조건이 `Gopkg.toml`에 추가된다. |
| `github.com/foo/bar`        | 예                                     | 아니요             | Gopkg.lock와 vendor/에 일시적으로 첨가된다.                                       |
| `github.com/foo/bar@v1.0.0` | 예                                     | -               | **즉각적인 오류**: constraint이 `Gopkg.toml`에 이미 존재한다                         |
| `github.com/foo/bar`        | 아니요                                   | 예               | `Gopkg.lock`로 부터 버전 제약 조건을 유추하고 `Gopkg.toml`에 추가한다.                    |
| `github.com/foo/bar`        | 예                                     | 예               | **즉각적인 오류:** 아무것도 할 것이 없다.                                             |

업데이트된 `Gopkg.lock`를 생성하기 위해서 `dep ensure -add`를 통해 해결 기능이 실행되는 경로들이 있는데, CLI 툴의 인자로 부터 이와 관련된 정보가 메모리내 표현되어 있는 `Gopkg.toml`에 적용된다.

![-add에 의한 모델 수정](assets/required-arrows.png)

추가할 필요가 있는 임포트 경로 인자들은 `required` 리스트를 통해 주입되고 만약 버전 요구가 명시적으로 지정되었다면, `[[constraint]]`에 해당하는 것이 만들어 진다.

만약 해결과정이 성공적이면 이러한 규칙들이 궁극적으로 지속될 수 있겠지만, 적어도 해결과정이 성공할 때까지는 일시적이다. 그리고, 해결하는 입장에서는 일시적인 규칙들과 디스크에서 직접 공급된 규칙들을 구별할 수 없다. 따라서, 해결사에게 `dep ensure -add foo@v1.0.0`는 `"foo"`를 `required` 리스트와, `version = "v1.0.0"`와 함께 `[[constraint]]`에 더하고 난 뒤 `dep ensure`를 실행함으로 `Gopkg.toml`를 수정하는 것과 동일하다.

그러나 이러한 수정은 일시적이어서 성공적인 `dep ensure -add`는 실제로 프로젝트를 동기화에서 벗어나게 할 수도 있다. 제한적 수정은 일반적으로 그렇지 않지만, `required`가 수정되었다면, 프로젝트는 비동기화에 도달할 것이다. 이에 대해 사용자에게 다음과 같은 적절한 경고를 제공한다.

```bash
$ dep ensure -add github.com/foo/bar
"github.com/foo/bar" is not imported by your project, and has been temporarily added to Gopkg.lock and vendor/.
If you run "dep ensure" again before actually importing it, it will disappear from Gopkg.lock and vendor/.
```

### `-update`

`dep ensure -update`의 동작은 해결사 자체의 동작에 밀접하게 연결되어 있다. 자세한 내용 전체는 [해결사 참조 자료](the-solver.md)에서 다룰 주제이나, `-update`를 이해하려는 목적으로 조금 단순화 할 수 있다.

첫째, [기능 최적화](#staying-in-sync)에 대해 논의하는 과정에 나타난 암시를 공고하기 위해, 해결 기능은 실행할 때 실제로 이미 존재하는 `Gopkg.lock`를 염두에 둔다.

![기존의 잠금이 해결 기능속을 다시 공급된다.](assets/lock-back.png)

해결사에 `Gopkg.lock`를 주입하는 것은 필수이다. 만약 해결사가 이전에 선택된 버전들을 디폴트로 보존하도록 하려면, 어디엔가 존재하는 `Gopkg.lock`에 관해 알아야만 한다. 그렇지 않으면, 무엇을 보존할 지 알 수 없다.

따라서, 잠금(lock)은 [이전에 논의된]() `SolveParameters` 구조체에 속성으로 있다. 그것과 2개의 다른 속성들이 `-update`에 대해 특히 중요한 것들이다.

```go
type SolveParameters struct {
  ...
  Lock gps.Lock // Gopkg.lock
  ToChange []gps.ProjectRoot // args to -update
  ChangeAll bool // true if no -update args passed
  ...
}
```

보통의 경우, 해결사가 `Gopkg.lock` 파일 안에 있는 항목에 대한 프로젝트 이름을 접하게 되면, 버전을 읽어서 그 프로젝트의 가능한 버전들로 구성된 큐의 머리에 둔다. 하지만 특정한 종속성이 `dep ensure -update`에 전달되는 경우는, `ToChange` 리스트에 추가된다; 해결사가 `ToChange`에 나열된 프로젝트를 발견하게 되면, 잠금으로 부터 버전을 읽지 않고 지나간다.

"잠금으로 부터 버전을 읽지 않고 지나간다"는 것은 `dep ensure -update github.com/foo/bar`가 `Gopkg.lock`로 부터 `github.com/foo/bar`에 대한 `[[project]]`절을 제거한 다음 `dep ensure`를 실행하는 것과 동일함을 암시한다. 그리고 실제로 그러하다 - 하지만, 그러한 접근방식은 권장하지 않으며, 미래에 등가성(equivalency)를 복잡하게 할 미묘한 변화가 도입될 가능성이 있다.

`-update`이 인자없이 전달되면, `ChangeAll`은 `true`로 설정되고, 해결사가 모든 새로 발견된 프로젝트의 이름에 대해 `Gopkg.lock`를 무시하는 결과를 초래한다. 이것은 `dep ensure -update` 모든 종속성을 인자로 전달함과 동시에 `rm Gopkg.lock && dep ensure`를 실행하는 것과 동일한 것이다. 다시 말하지만, 이러한 접근방식은 권장하지 않으며, 미래에 일어날 변화가 미묘한 차이를 가져올 수도 있다.

`Gopkg.lock`로 부터 버전 힌트가 버전 큐의 머리에 배치되지 않았을 경우에 dep은 이 특정 종속성에 대한 가능한 버전들의 집합을 탐색하게 된다. 이러한 탐색은 [고정된 정렬 순서](https://godoc.org/github.com/golang/dep/gps#SortForUpgrade)에 따라 수행되며, 더 새로운 버전들이 먼저 시도되고, 결과적으로 업데이트가 된다.

예를 들어, `github.com/foo/bar`라는 프로젝트가 다음과 같은 버전들을 가지고 있다고 하자:

```bash
v1.2.0, v1.1.1, v1.1.0, v1.0.0, master
```

만약 `^1.1.0`버전의 그 프로젝트에 의존하고 있고, `Gopkg.lock`안에 `v1.1.0`가 있다고 하면, 이 것은 제약 조건에 일치하는 버전이 3개가 있고, 그 중은 2개는 현재 선택된 것 보다 더 새로운 것임을 의미한다. (`v1.0.0`와 `master` 브랜치 또한 있지만, 이 것들은 `^1.1.0`로 인해 허용되지 않는다.) 일상적인 `dep ensure` 실행은 `v1.1.0`를 복사하고 큐 안에 있는 다른 모든 버전들 보다 앞으로 밀어 넣을 것이다.

```bash
[v1.1.0, v1.2.0, v1.1.1, v1.1.0, v1.0.0, master]
```

그리고 어떤 다른 조건이 제시되어 해결사가 버리지 않는 한 `v1.1.0`는 다시 선택될 것 이다. 그러나 `dep ensure -update github.com/foo/bar`을 실행하면 잠근 버전은 앞으로 추가되지 않는다. 

```bash
[v1.2.0, v1.1.1, v1.1.0, v1.0.0, master]
```

그래서, 어떤 다른 충돌을 배제하면, `v1.2.0`가 선택되어 원하는 업데이트를 결과로 얻는다.

#### `-update` 와 제약 조건 형식들

주어진 예제를 가지고 계속하면, `-update`를 통한 업데이트가 부수적으로 얻어지는 점을 주목하는 것이 중요하다 - 해결사는 결코 새로운 버전을 명시적으로 지정하지 않는다. 잠금으로 부터 얻는 힌트를 더하지 않으므로 큐의 첫번째 버전이 제약 조건을 충족시키는 것이다. 결과적으로, `-update`는 특정 형식의 제약 조건들에만 유효한 것이다.

브랜치에 개정번호를 포함시킴으로 해서 이 기능이 브랜치와도 잘 동작한다는 것을 목격할 수 있다. 만약 유저가 `branch = "master"`를 사용하기 원하고, `Gopkg.lock`가 정식 소스코드의 `master` 브랜치 정점(예를 들어, `bbccdde`) 보다 이전의 개정번호 (예를 들면 `aabbccd`)를 가리키고 있다면, `dep ensure`는 결국 다음과 같이 큐를 만들게 된다.

```bash
[master@aabbccd, v1.1.0, v1.2.0, v1.1.1, v1.1.0, v1.0.0, master@bbccdde]
```

`-update`를 사용하게 되면, 머리에 있는 힌트는 무시되고; `branch = "master"`에 의해 해결사는 모든 유의적 버전들을 거부하고, 결국 `master@bbccdde`에 정착하게 된다.

버전 큐에 있는 모든 버전들은 밑으로 있는 개정번호들을 계속 추적하는데 그 의미는 만약 예를 들어 어떤 상류 프로젝트가 억지로 git 택그들을 푸쉬한 경우도 마찬가지이다.

```bash
[v1.1.0@aabbccd, v1.1.0, v1.2.0, v1.1.1, v1.1.0@bbccdde, v1.0.0, master]
```

따라서, 만약에 상류 프로젝트의 태그가 프로젝트의 종속성에 하나로 강제로 푸쉬되었다면, dep는 `dep ensure -update`에 의해 명시적으로 변화가 허락되기 전까지는 원래의 개정번호를 유지한다.

꼭 알고 가야 할 점은 `-update`의 동작이 제약 조건의 형식에 따라 지배된다는 것이다.

| `Gopkg.toml` 버전 제약조건 형식   | 제약조건 예       | `dep ensure -update` 동작                                                                         |
| ------------------------- | ------------ | ----------------------------------------------------------------------------------------------- |
| `version` (semver 범위)     | `"^1.0.0"`   | 범위가 허락하는 가장 최신의 버전을 갖도록 노력한다.                                                                   |
| `branch`                  | `"master"`   | 지명된 브랜치의 현재 정점으로 움직이도록 노력한다.                                                                    |
| `version` (범위가 아닌 semver) | `"=1.0.0"`   | 상류의 릴리즈가 움직인 경우에만 변화가 일어날 수 있다. (예를 들어, `git push --force <tag>`)                         |
| `version` (semver가 아닌 경우) | `"foo"`      | 상류 릴리즈가 움직인 경우에만 변화가 일어날 수 있다.                                                                  |
| `개정번호`                    | `aabbccd...` | 변화가 가능하지 않다.                                                                                    |
| (없음)                      | (없음)         | [정렬 순서](https://godoc.org/github.com/golang/dep/gps#SortForUpgrade)에 따른, 작동하는 첫번째 버전 (권장 하지 않음) |