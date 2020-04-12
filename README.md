# 블로그 원본

## 저장소 클론 후 해야 할 일

```shell script
git submodule init
git submodule update
```

## 새 글 만들기

```shell script
hugo new posts/(제목).md
```

## 테스트 하기

```shell script
hugo server -D
```

* `-D` 옵션을 추가하면 Draft를 포함하여 빌드함
* 확인은 http://localhost:1313 으로 할 수 있음

## 빌드하기

```shell script
hugo -D
```

* 결과는 `./public` 에서 확인할 수 있음
* `draft: false`인 글은 빌드하지 않음

## 배포하기

```shell script
cd public
git commit -am "Commit Message"
git push origin master
```

아니면 다음과 같이 한 번에 push 할 수도 있다.

```shell script
git push --recurse-submodules=on-demand
```

## 참고한 글들

* [Hugo Quick Start](https://gohugo.io/getting-started/quick-start/)
* [Git 도구 - 서브모듈](https://git-scm.com/book/ko/v2/Git-%EB%8F%84%EA%B5%AC-%EC%84%9C%EB%B8%8C%EB%AA%A8%EB%93%88)