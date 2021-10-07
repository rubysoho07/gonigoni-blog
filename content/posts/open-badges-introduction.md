---
title: "Open Badges 소개"
date: 2021-05-01T12:55:57+09:00
tags: [Open Badges, Standard]
draft: false
categories: [Everything]
comments: true
---

저는 초등학생 때 아람단에 가입해서 이런저런 활동들을 해 봤는데요. 열심히 활동하는 편은 아니었지만, 뭔가 활동을 하면 배지를 얻을 수 있었습니다. 적극적으로 활동하는 친구들은 아람단 단복에 정말 많은 배지들을 달고 있었던 기억이 납니다.

이러한 배지 시스템은 스마트 폰의 앱이나 웹 사이트에서도 볼 수 있습니다. 예를 들어 제가 운동할 때 사용하는 앱 중 하나인 Nike Training Club은 특정한 조건을 달성하면 새로운 기록을 만들 수 있는데요.

아래는 Nike Training Club 앱에서 제가 달성한 기록들 중 일부입니다.

{{< figure src="/img/open-badges-introduction-1.jpeg" width="30%" >}}

그런데, 제가 달성한 기록을 전 세계의 어떤 플랫폼에서든 인증받을 수 있다면 어떨까요? 문서나 카드로 만든 자격증이 없어도, 제가 다른 플랫폼으로 저의 데이터를 옮기더라도 저의 기록을 보존할 수 있다면 좋지 않을까요?

이런 문제를 해결하기 위해 모질라(Mozilla) 재단은 2010년에 Open Badges라는 규격을 소개했습니다. 지금은 여러 에듀테크 관련 기업/기관들이 모인 IMS Global Learning Consortium에서 규격을 관리하고 있는데요. 

이번 글에서는 Open Badges의 기본 구조를 소개하고, 어떻게 활용할 수 있을지에 대해 이야기 해 보겠습니다. 

# 기본 구조

Open Badges는 스킬이나 성취를 표현하기 위한 메타데이터를 포함한 디지털 배지입니다. Open Badges에는 다음과 같은 특성이 있습니다. 

* Open Badges의 규격은 표준화되어 있고, 웹에서 쉽게 공유할 수 있습니다. 
* 각각의 Open Badge는 이미지 및 관련 정보들을 포함하고 있고, 배지를 받은 사람, 발행자, 그리고 이를 증명하는 자료를 포함하고 있습니다. 
* Open Badges는 하나의 플랫폼에서 다른 플랫폼으로 쉽게 옮길 수 있습니다. 

Open Badges의 기본 구조는 다음 그림으로 요약할 수 있습니다.

{{< figure src="https://openbadges.org/sites/default/files/assets/Open%20Badges/Build%20Page/OB_Best_Practices.svg" caption="출처: [openbadges.org](https://openbadges.org/build) / This image is licenced under a [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/)." >}}

하나의 Open Badge는 다음 요소를 갖고 있습니다. 

* Issuer Profile: 배지를 수여하는 개인이나 단체에 대한 정보입니다. 
* BadgeClass: 발행자가 알 수 있는 성취에 대한 정보입니다. 예를 들어 코스에 대한 정보, 이미지 등이 들어갈 수 있습니다. 
* Assertion: 배지에 대한 개인의 성취 정보입니다. Assertion은 하나의 BadgeClass와 연결하고, 배지를 받는 사람의 정보를 포함합니다. 상세한 정보에는 배지를 언제 받았는지, 수령자에 대한 정보, (필수는 아니지만) 증거 자료 및 만료 일자 등이 있겠지요.

전체적인 시스템 안에는 다음과 같은 역할이 있습니다.

* Issuer: BadgeClass의 생성과 Assertion의 발급을 담당하는 애플리케이션
* Displayer: Open Badge들과 연관된 정보를 표시하고, 배지에 대한 인증을 제공하는 애플리케이션
* Host: 배지를 import하고, 취합하고, 공개적으로 제공하는 애플리케이션. 배지를 획득한 사람들이 요청에 따라 배지를 다른 서비스로 내보내는 작업도 할 수 있습니다.

그렇다면, 각각의 사용자들은 배지를 어떻게 만들고 이용할 수 있을까요?

## Badge 만들기 (개발자)

제가 개발자다 보니, 개발자 관점에서 보는 내용이 좀 더 길어질 것 같네요. 이 부분이 필요 없다면 다음으로 넘어가셔도 무방합니다. [Open Badges 스펙을 구현한 프로그램들](https://site.imsglobal.org/certifications?refinementList%5Bstandards_lvlx%5D%5B0%5D=Open%20Badges)이 이미 존재하기 때문에, 굳이 처음부터 개발할 필요가 없기도 합니다. 

현재 최신 표준은 [Open Badges v2.0](https://www.imsglobal.org/sites/default/files/Badges/OBv2p0Final/index.html)이며, 2.1 버전이 개발 중입니다. 아래 설명들은 2.0 버전을 기준으로 합니다. 

Open Badges의 데이터는 JSON-LD 포맷으로 구성되어 있습니다. 이러한 데이터는 성취에 대한 메타데이터를 포함하고 있는데요. 예제를 보여드리겠습니다. 

```json
{
  "@context": "https://w3id.org/openbadges/v2",
  "id": "https://example.org/assertions/123",
  "type": "Assertion",
  "recipient": {
    "type": "email",
    "identity": "alice@example.org"
  },
  "issuedOn": "2016-12-31T23:59:59+00:00",
  "verification": {
    "type": "hosted"
  },
  "badge": {
    "type": "BadgeClass",
    "id": "https://example.org/badges/5",
    "name": "3-D Printmaster",
    "description": "This badge is awarded for passing the 3-D printing knowledge and safety test.",
    "image": "https://example.org/badges/5/image",
    "criteria": {
      "narrative": "Students are tested on knowledge and safety, ....."
    },
    "issuer": {
      "id": "https://example.org/issuer",
      "type": "Profile",
      "name": "Example Maker Society",
      "url": "https://example.org",
      "email": "contact@example.org",
      "verification": {
         "allowedOrigins": "example.org"
      }
    }
  }
}
```

기본적인 데이터 클래스는 다음과 같습니다. 

* [Assertion](https://www.imsglobal.org/sites/default/files/Badges/OBv2p0Final/index.html#Assertion): 위의 설명과 링크를 참조하세요.
* [BadgeClass](https://www.imsglobal.org/sites/default/files/Badges/OBv2p0Final/index.html#BadgeClass): 위의 설명과 링크를 참조하세요. 
* [Profile](https://www.imsglobal.org/sites/default/files/Badges/OBv2p0Final/index.html#Profile): Open Badge를 사용하는 entity나 단체를 표현하는 정보의 모음입니다. Issuer를 Profile로 표현하여야 하고, 배지를 받는 사람이나 배지를 주는 사람들을 표현하는 데 사용할 수 있습니다.

그러면 위의 예제는 다음과 같이 설명할 수 있습니다. 

* `type`이 `Assertion`이므로, 개인이 받은 배지에 대한 정보입니다. 
* 이 배지를 받은 사람은 `recipient` 속성에 있는 값으로 확인할 수 있습니다. 이메일 주소가 `alice@example.org`인 사람에게 발급되었네요. 
* 이 배지는 UTC 기준으로 2016년 12월 31일 23:59:59에 발급되었습니다. (`issuedOn` 속성을 확인하세요)
* 이 Assertion을 검증하기 위한 정보는 `verification`에서 확인할 수 있습니다. `type`이 `hosted`인 경우, Assertion의 ID를 기준으로 검증하면 된다고 합니다. 
* 배지의 내용은 `badge`에서 확인할 수 있습니다.
    * `BadgeClass` 타입으로 생성합니다. 
    * `name`과 `description`으로 이 배지의 이름과 설명을 확인할 수 있습니다. 
    * `image`는 이 배지의 이미지입니다. PNG나 SVG 이미지여야 한다고 하네요. 
    * `criteria`는 이 배지를 얻기 위한 조건을 설명합니다. 
    * `issuer`는 이 배지를 발행한 기관에 대한 정보입니다. `Profile` 타입으로 생성합니다.
        * `name`, `url`, `email`등의 정보로 `Example Maker Society`라는 단체라는 것을 알 수 있습니다.

### 제공하여야 하는 기능들

Open Badges를 구현하려면 어떤 기능이 필요한지 요약해 보겠습니다. 

* Publishing Badge Objects: Open Badges 데이터는 JSON-LD 문서로 생성하여야 한다.

* Data Validation: 배지를 구성하는 Badge Object들이 적절하게 생성되었는지 확인하는 과정
  * 모든 필수적인 배지 객체는 적절히 연결되어 있고, Validator에서 `200 OK` 메시지를 반환하여 사용 가능하다. (200 OK 외에도 3xx redirect가 허용되지만, 적절한 자원이 200 OK를 반환하여 종료하여야 함)
  * Optional한 연결된 배지 객체는 사용 가능하다.
  * 각각의 배지 객체는 유효한 JSON-LD 문서이다.
  * 각각의 배지 객체는 클래스에 대한 모든 필수 속성을 포함한다.
  * 각각의 배지 객체는 어휘에 정의된 속성과, 이 속성에 정의된 데이터 타입에 맞는 값을 포함한다.

* Verification: 배지를 구성하는 데이터들이 목적에 맞는지 확인하는 과정
  * 모든 배지 객체는 data validation을 통과한다. 
  * 모든 배지 객체는 VerificationObjects에 선언된 규칙에 따라 적절한 Issuer Profile에 의해 생성되었다. 
  * Assertion은 예상되는 수령인의 유효한 속성 값에 부여되었다. (예: recipient의 이메일 주소 값이 유효한 경우)
  * 배지를 발행한 사람(기관)은 선언된 BadgeClass의 Assertion을 수여할 권한이 있다. 
  * Assertion이 만료되지 않았다. 
  * 배지를 발행한 사람(기관)이 Assertion을 취소하지 않았다.

배지 데이터의 발급과 검증 과정들은 [Implementation](https://www.imsglobal.org/sites/default/files/Badges/OBv2p0Final/index.html#implementation) 문서에 좀 더 자세하게 나와 있으니, 관심이 있다면 이 문서를 한 번 참고해 보시는 것이 좋겠네요.

## Badge 발행하기 (기관)

* 이 문단의 내용은 [링크](https://openbadges.org/issue)한 글을 기반으로 작성하였습니다. 

Open Badges를 발행할 수 있는 기관에는 어떤 것들이 있을까요? 위 링크에서는 다음과 같이 이야기 하고 있습니다. 

* 학교 및 대학교
* 고융주
* 커뮤니티 및 비영리 단체
* 정부 기관
* 도서관 및 박물관
* 이벤트 주최자
* 특정한 스킬을 필요로 하는 회사나 단체

기관에서는 다음과 같은 과정으로 배지를 디자인합니다. 

* 학습 과정 및 평가를 제공
* Open Badge의 기술 표준에 맞게, 학습 과정과 평가를 표현할 수 있는 배지를 생성
* 배지 기준을 달성한 경우 배지를 제공

## Badge를 얻고 공유하기 (사용자)

* 이 문단의 내용은 [링크](https://openbadges.org/earn)한 글을 기반으로 작성하였습니다. 

학생이나 수강생이 배지를 받았다면, 이를 관리할 수 있는 툴로 배지를 관리할 수 있습니다. 여러 기관에서 배지를 받았어도, 이를 한 곳에 모아 관리할 수 있다고 하네요. (물론 프로그램마다 기능이 다를 수는 있습니다)

이러한 배지들은 개인이 특정한 스킬을 얻었음을 증명하는 데 사용할 수 있습니다. 만약 사람을 채용하는 입장이라면, 사람들이 이력서에 올린 배지들을 쉽게 검증할 수도 있을 것입니다. 

# 어떻게 활용할 수 있을까?

Open Badges를 이용하는 일반적인 경우는 다음과 같습니다. 

* 학교나 기관에서 강의를 만들고, 조건에 따라 배지를 획득할 수 있도록 합니다. 
* 학생은 강의를 듣고, 배지를 획득합니다. 
* 획득한 배지는 이력서에 넣거나, 링크드인 프로필에 올릴 수 있습니다.

이 글에서는 Open Badges와는 조금 다르지만, 실제로 디지털 배지를 사용하는 예를 한 번 들어보겠습니다. 

만약 AWS 자격증을 획득하신 적이 있다면, [AWS Training and Certification](https://aws.training) 페이지가 익숙할 것입니다.

자격증 사이트에서 읽었던 공지 중에, 다음과 같은 글이 있었습니다. 

> Share AWS Certifications with New Digital Badges
>
> AWS Certification digital badges now provided via Credly’s Acclaim platform
>
> AWS Certification provides digital badges as a benefit of getting AWS Certified to showcase certification status. With digital badges on Credly’s Acclaim platform, you now have more flexible options for recognition and verification. Take advantage of one-click badge sharing on social media newsfeeds, tools for embedding verifiable badges on websites, and an optional public profile with all earned AWS Certification badges.

> 새 디지털 배지로 AWS 인증 공유
>
> 이제 Credly Acclaim 플랫폼을 통해 제공되는 AWS 인증 디지털 배지.
>
> AWS 인증은 인증 상태를 보여주는 디지털 배지를 인증 취득 혜택 중 하나로 제공합니다. 이제 Credly Acclaim 플랫폼에서 디지털 배지가 제공되므로 손쉽게 인정을 받고 확인할 수 있게 되었습니다. 소셜 미디어 뉴스피드에서 한 번의 클릭으로 배지를 공유하고, 도구를 통해 확인 가능한 배지를 웹 사이트에 포함시키고, 모든 취득 AWS 인증 배지가 포함된 공개 프로필(선택 사항)을 이용할 수 있습니다. 

이 공지에 나와있는 것처럼, 이전에 받았던 AWS 자격증을 Credly의 Acclaim 플랫폼으로 이전하는 작업이 있었는데요. 이 Acclaim 시스템이 [Open Badges 규격에 대한 인증](https://site.imsglobal.org/certifications/credly/acclaim#cert_pane_nid_403553)을 받은 상태이긴 합니다. 하지만 [Credly의 문서](https://www.credly.com/docs/issued_badges#get-issued-badges)를 참고해 보면, Open Badges 규격과는 다르게 운영되고 있는 것을 볼 수 있습니다.

이제 배지를 확인해 보겠습니다. 

Credly에서 [제가 받은 AWS Certified Solutions Architect - Associate 배지](https://www.credly.com/badges/457a156b-a8db-4cab-b94b-ac6bc1f55913)를 확인해 보면, 다음과 같은 정보를 보실 수 있습니다. 

* 획득한 날짜
* 배지 이미지
* 배지에 대한 설명
* 만료 일자

그리고 상단의 Verify 버튼을 누르면, 이 배지를 검증할 수도 있습니다. 그 결과 아래 스크린샷과 같은 내용을 확인할 수 있습니다.

{{< figure src="/img/open-badges-introduction-2.png" width="30%" >}}

이 내용의 의미는 다음과 같습니다. 

* AWS의 Training and Certification이 2020년 4월 12일에 Credly를 사용하여 발급하였습니다.
* 저에게 발급한 배지임을 확인할 수 있습니다.
* 마지막으로 업데이트된 날짜는 5월 7일입니다.

이 자격증을 갖고 있다고 이력서에 썼는데, 채용 담당자가 이력서의 내용을 검증할 때 유용하겠죠. 

# 마무리하며

최근에 나타나는 이슈 중 하나를 언급해 보면, 구직자나 고용주들 간 원하는 역량이 일치하지 않아 고생하는 사례들을 종종 볼 수 있습니다. 요즘 같은 시기에는 개발자 채용에서 이러한 문제를 흔히 볼 수 있는데요. 

예전에 출장을 가서 Open Badges에 대한 내용들을 보면, 구직자와 고용주 간 불일치를 이러한 Digital Badge로 해결해 보자는 이야기들이 종종 나왔던 것 같습니다. 학교나 기관은 구직자가 배운 것을 증명할 수 있는 디지털 배지를 발급하고, 고용주는 구직자의 역량을 검증하는 데 디지털 배지를 이용할 수 있을 것입니다. 그리고 이러한 배지의 신뢰성을 더하기 위해 블록체인을 도입해 보자는 이야기도 나왔던 것 같네요. 

저는 굳이 이런 거창한 이야기들이 아니어도, 사람들이 성취한 것을 데이터로 만들고 쉽게 검증할 수 있는 시스템을 만드려는 노력이 흥미롭게 느껴졌습니다. 그래서 이번달 블로그 글의 주제는 Open Badges로 정해 보았습니다.

마지막으로 이 글은 **지금 제가 재직중인 회사의 입장과는 무관함을 알려드립니다.**

# 참고 자료

* [IMS Open Badges](https://openbadges.org/)
* [Mozilla의 Open Badges 도움말](https://support.mozilla.org/ko/products/open-badges)
