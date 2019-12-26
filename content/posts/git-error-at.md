---
title: "Git 원격 저장소에서 사용자 이름에 '@'이 들어가면 동작하지 않는 경우"
categories: Git
tags: [Git]
draft: true
date: 2018-10-02T23:46:00+09:00
---

부서에서 사용하는 CodeCommit에 Push를 하려고 하는데, 아래와 같은 오류 때문에 Push를 할 수 없었다.

```
fatal: UriFormatException encountered.
queryUrl
```

보통 이런 문제가 발생하면 오류 메시지를 먼저 찾아 보는 편인데, 이상하게 검색이 잘 안 됐었다. (컨디션이 안 좋아서 그랬던가...-_-) 그러다가 다음과 같은 내용을 발견했다. 

[https://github.com/Microsoft/Git-Credential-Manager-for-Windows/issues/587](https://github.com/Microsoft/Git-Credential-Manager-for-Windows/issues/587)

내용을 요약하면 다음과 같다.

* Git 원격 저장소의 사용자 이름이 이메일 주소인 경우 이러한 오류가 발생한다.
* RFC3986에 따르면, URI에 이미 할당되어 있는 문자는 `:/?#[]@!$&'()*+,;=`이다.
* `@`의 경우, `%40`으로 바꾸면(escaping) 문제를 해결할 수 있다. 

위에 올라온 프로그램의 경우는 사용자 이름을 체크해서 변경이 필요하면 변경하고, 아니면 그대로 사용하는 식으로 처리한 것 같다.

그래서, 이 문제를 어떻게 해결하는 방법은 두 가지가 있다. 

1. 계정 정보를 넣을 때 `@` 대신에 `%40`을 넣어 escaping 한다. (두번째 참고 자료를 확인!)
2. 사용자 이름에 escaping이 필요한 문자는 제외시킨다. 

일단 우리 부서만 AWS를 사용하고 있어서, AWS IAM의 사용자 이름에 escaping이 필요한 문자는 제외시키기로 했다. 만약에 Git 원격 저장소의 사용자 이름이 이메일 주소여야 한다면, escaping을 하여야 할 것이다.

### 참고자료

[https://github.com/Microsoft/Git-Credential-Manager-for-Windows/issues/587](https://github.com/Microsoft/Git-Credential-Manager-for-Windows/issues/587)
[https://stackoverflow.com/questions/37342809/git-remote-url-with-username-in-form-of-email-address](https://stackoverflow.com/questions/37342809/git-remote-url-with-username-in-form-of-email-address)