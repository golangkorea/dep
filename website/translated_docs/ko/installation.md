---
title: 설치하기
---
dep 출시 버전(released version) 으로 설치할 것을 권장합니다. 출시 버전을 이용한다고 해서 안정성이 보장되지는 않지만, 의도치 않은 설치 실패는 피할 수 있습니다.

미리 컴파일된 바이너리는 [releases](https://github.com/golang/dep/releases) 페이지에서 이용할 수 있습니다. MacOS의 경우, Homebrew를 이용해 최신 출시 버전을 설치하거나 최신 버전으로 업그레이드할 수 있습니다:

```sh
$ brew install dep
$ brew upgrade dep
```

dep를 커스터마이징해서 사용하고 싶다면, `go get` 명령어를 통해 dep을 설치할 수 있습니다:

```sh
go get -u github.com/golang/dep/cmd/dep
```

dep은 Go 작업환경(workspace) 및 GOPATH 설정이 필요합니다. Go 작업환경과 GOPATH에 관해 잘 모르신다면, [Go 언어 문서](https://golang.org/doc/code.html#Organization)를 참고해서 로컬 작업환경을 설정하시길 바랍니다. dep의 모델은 GOPATH 설정 없이도 동작 가능한 수준까지 발전할 수도 있지만, 아직까지는 GOPATH 설정이 꼭 필요합니다.