---
title: "VSCode와 oh-my-zsh를 사용할 때 터미널 글꼴이 깨진다면"
categories: Linux
tags: [VSCode, Visual Studio Code, zsh, oh-my-zsh, font]
draft: false
date: 2018-10-04T21:34:00+09:00
comments: true
---

최근에 oh-my-zsh를 써 보기 시작했다. (써보면 편하다고는 하는데, 아직까지는 체감을 못하고 있긴 하다.)

그런데 Visual Studio Code에서 터미널을 쓸 때, 아래처럼 글꼴이 깨지는 현상이 있었다. (현재 테마는 "agnoster"이다.)

{{< figure src="/img/vscode-oh-my-zsh-1.png" >}}

이 문제에 대해 찾아보니, 다음과 같은 이슈를 발견할 수 있었다. 

[https://github.com/Microsoft/vscode/issues/7116](https://github.com/Microsoft/vscode/issues/7116)

여기서 제시하는 해결책은 다음과 같다.

1. "SourceCodePro+Powerline+Awesome Regular" 글꼴을 받아 설치한다. [링크](https://github.com/Falkor/dotfiles/raw/master/fonts/SourceCodePro%2BPowerline%2BAwesome%2BRegular.ttf)
2. Visual Studio Code 설정에서 `terminal.integrated.fontFamily` 항목에 `'SourceCodePro+Powerline+Awesome Regular'`를 넣는다. (작은 따옴표를 포함해야 한다!)

그러면 아래와 같이 터미널의 글꼴이 깨지지 않게 된다.

{{< figure src="/img/vscode-oh-my-zsh-2.png" >}}

### 참고자료

1. [https://github.com/Microsoft/vscode/issues/7116](https://github.com/Microsoft/vscode/issues/7116)
2. [https://github.com/powerline/fonts/issues/246](https://github.com/powerline/fonts/issues/246)