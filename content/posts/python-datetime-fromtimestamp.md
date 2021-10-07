---
title: "Python Datetime의 fromtimestamp() 사용 시 주의할 점"
categories: [Python]
tags: [Python, datetime, fromtimestamp]
draft: false
date: 2018-09-13T00:08:00+09:00
comments: true
---

최근에 Timestamp가 있는 데이터를 다루다가, 신기한 버그를 발견하여 여기에 기록해 본다. 해당 문제가 수정된 것 같기는 한데, 아직 배포는 안 된 것 같다.

### Windows에서 발생하는 버그

{{< figure src="/img/python_fromtimestamp_windows.png" caption="fromtimestamp 함수의 버그 (Windows)">}}

[해당 이슈](https://bugs.python.org/issue29097)에 대해 설명한 부분을 확인하면, __*0에서 86399*__ 사이의 값을 입력하면 OSError가 발생한다.

### Ubuntu에서는?

잘 동작한다. 심지어 Windows 10에 깔린 Ubuntu를 이용했고, 3.5.x 버전인데도 멀쩡히 잘 돌아간다. 

{{< figure src="/img/python_fromtimestamp_ubuntu.png" caption="Ubuntu에서는 정상적으로 동작한다.">}}

### 참고자료

* [https://bugs.python.org/issue29097](https://bugs.python.org/issue29097)
* [https://docs.python.org/3/whatsnew/changelog.html](https://docs.python.org/3/whatsnew/changelog.html)