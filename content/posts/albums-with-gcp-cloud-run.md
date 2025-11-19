---
title: "GCP Cloud Run으로 음반 관리하기"
date: 2025-11-19T14:00:00+09:00
tags: [GCP, Cloud Run, 컨테이너, Flask]
draft: false
categories: [DevOps]
comments: true
---

## 들어가며

제가 살면서 처음으로 구매한 음반은 Green Day의 베스트 앨범인 [International Superhits!](https://open.spotify.com/album/6HUIbDhzmqcwxrxUfTuHdW) 입니다. 중학교 2학년 때 제가 살던 동네의 월마트에서 테이프를 샀던 기억이 있습니다. 그로부터 20여 년이 지났는데요. 지금까지 본가와 서울 집에 있는 CD와 테이프를 모두 합쳐보니 150장이 넘더라구요. (참고로 예전에 핫뮤직이라는 락 음악 전문 잡지가 있었는데, 부록으로 주는 샘플러를 빼고 계산했어요. 실제로는 좀 더 많지 않을까 싶습니다) 

{{< figure src="/img/albums-with-cloud-run-1.png" >}}

지금도 가끔씩 "이건 CD로 갖고 있어야겠다"는 앨범이 있어서, 알라딘 중고서점과 같은 곳을 뒤져 볼 때가 있습니다. 검색을 하다 보면 "이거 예전에 샀던 건가?" 라는 생각이 들 때가 있어요. 그래서 제가 지금까지 구매한 앨범을 검색할 시스템이 필요하다고 생각했습니다. 예전에 Django로 만들고 방치했던 것이 있었는데, 이걸 완전히 바꿔서 진행해 보려고 했습니다. 

## 목표

이 프로젝트의 목표는 다음과 같습니다.

* 어디서든 추가, 검색이 가능한 시스템을 만들자. 
* 비용은 최대한 아끼는 방향으로 하자. (트래픽이 거의 없기 떄문)
* 안정성은 크게 고려하지 말자. (나 혼자 사용하기 때문)

## 시스템 구성

{{< figure src="/img/albums-with-cloud-run-2.svg" caption="시스템 구성도" >}}

### 인프라

이 애플리케이션은 고정된 서버에 올릴 규모의 프로그램이 아니라서, 컨테이너로 간단하게 애플리케이션을 돌릴 수 있는 방법을 찾아 봤습니다. 최종으로 이 애플리케이션의 인프라로 선택한 것은 **GCP의 Cloud Run** 입니다. 사실 지금까지 AWS만 써 왔다 보니 GCP는 익숙하지 않았지만, 다양한 후보를 고려하다 보니 여기까지 왔습니다. 

컨테이너를 간단하게 돌릴 수 있는 플랫폼으로 AWS의 App Runner, Lambda를 고려했고, 그 과정에서 GCP의 Cloud Run도 후보에 넣었습니다.

Lambda를 고려하지 않은 이유는, Lambda의 호출 방식에 맞게 애플리케이션을 바꿔야 하기 때문입니다. 기능을 아주 많이 추가하다 보면, 여러 함수로 애플리케이션을 분할해야 할 수도 있다는 점을 고려했습니다. 이러면 관리해야 할 요소가 너무 많아질 것 같다는 생각이 들었어요. 그래서 Lambda는 사용하지 않기로 했습니다.

그래서 컨테이너 기반으로 간단하게 애플리케이션을 돌릴 수 있는 App Runner와 Cloud Run을 비교해 봤는데요. App Runner는 사용하지 않을 때 정지/재시작을 수동으로 관리해야 한다는 점이 걸리더라구요. 서울 리전에 해당 서비스가 제공되지 않는 점도 아쉬웠구요. (참고자료: [App Runner 비용](https://aws.amazon.com/ko/apprunner/pricing/)) 결국 App Runner도 사용하지 않기로 했어요.

Cloud Run의 경우는 서울 리전에서도 사용할 수 있고, 사용하지 않을 때 자동으로 컨테이너 개수를 0으로 축소할 수 있다는 점, 프리 티어를 제공해 준다는 점 때문에 선택했습니다. 컨테이너 이미지 저장도 0.5GB까지는 무료여서 GCP의 Artifact Registry를 사용하기로 했어요. (참고자료: [GCP Cloud Run](https://cloud.google.com/run?hl=ko), [GCP Artifact Registry 가격 책정](https://cloud.google.com/artifact-registry/pricing?hl=ko))

외부 서비스에서 사용하는 키와 같이 민감한 정보 관리는 GCP의 Secret Manager 서비스를 활용했습니다. 무료 티어로 사용 가능한 범위가 있고, Cloud Run에 배포할 때 애플리케이션을 수정하지 않아도 값을 참조할 수 있어서 채택했습니다. (참고자료: [GCP Secret Manager 가격 책정](https://cloud.google.com/secret-manager/pricing?hl=ko))

### 데이터베이스

데이터베이스는 **Supabase**를 이용해 보기로 했습니다. 무료로 사용 가능한 티어가 있고, 사용량이 그에 한참 미치지 못할 것이기 때문에 선택했습니다. 그리고 Supabase 서비스를 이용할 때 다양한 언어를 지원한다는 점도 좋았습니다. 저는 Python이 익숙해서 Python Library를 이용했습니다. (참고자료: [Supabase 요금](https://supabase.com/pricing), [Supabase Python Client Library](https://supabase.com/docs/reference/python/start))

### 애플리케이션

예전에 만들었던 애플리케이션은 Django 기반이었습니다. 꽤나 오래 전에 만들었고, 유지보수를 거의 안 했던 프로젝트여서 과감하게 버리고 새로 만들기로 했어요. 

굳이 웹사이트가 필요한가? 라는 생각이 들어서 백엔드만 제공하는 형태로 구성했습니다. 검색이나 앨범 추가가 필요할 때는 슬랙의 Slash command를 인터페이스로 사용하기로 했어요. 그래서 프론트엔드가 필요 없는 프로젝트가 되었습니다. 요즘은 Django, Flask, FastAPI와 같은 다양한 백엔드 프레임워크가 있다 보니 이 중에 선택했습니다. Django는 이것저것 붙어서 과하다고 생각했고, 비동기 처리가 중요한 프로젝트가 아니여서 FastAPI 대신 **Flask**를 선택했습니다. (일단 저 혼자 쓰기 때문에 화려한 건 필요 없다고 생각했습니다)

프로젝트를 새로 시작하면서 Gemini CLI의 도움을 많이 받았습니다. GCP에 인프라를 올려서 Gemini가 적절하지 않을까? 라는 생각이 들었고, 토큰을 많이 쓰지 않을 것 같아서 무료 티어로도 충분하다고 판단했습니다. 무료로 사용할 수 있는 툴을 찾다 보니 Gemini를 선택했습니다. (회사 일 때문에 사용하는 거고, 회사에서 비용을 지원해 준다고 하면 아마도 다른 툴을 사용하겠죠 😅)

그 외에 CI/CD로는 **GitHub Actions**를 이용했고, 커밋 전 코드 컨벤션 체크를 위해 [Ruff](https://docs.astral.sh/ruff/), [pre-commit](https://pre-commit.com/)을 이용했습니다. 

## 이 프로젝트를 하면서 알게 된 것

이 프로젝트를 진행해 보면서 사소하게 알게 된 것들이 있어서 한 번 공유해 봅니다. 

**Cloud Run은 AMD64 아키텍처만 지원함**

제가 Apple Silicon 기반의 맥북을 사용하고 있다 보니, Cloud Run으로 배포 후 실행되지 않는 현상을 발견했습니다. 알고보니 GCP의 Cloud Run은 AMD64 아키텍처만 지원하고 있었고, 컨테이너 이미지를 빌드할 때 `--platform=linux/amd64` 옵션을 추가로 붙여서 빌드하도록 했습니다.

**Supabase의 무료 버전은 7일간 사용하지 않으면 일시정지됨**

Supabase를 무료 버전으로 사용하다 보니, 거의 일주일을 사용하지 않으면 메일로 해당 프로젝트에 대한 일시 정지 예정 알림을 받습니다. 실제로 해당 프로젝트가 일시 정지 되는 상황을 지켜봤는데, 일시 정지 상태의 프로젝트를 다시 살리려면 4~5분 정도의 시간이 걸리더라구요.

그래서 가끔씩 돌려보기 위해 랜덤으로 음반을 추천하는 slash command 기능을 추가했습니다.

**슬랙의 Slash command는 3초 이내에 응답해야 하고, 슬랙 내 워크플로 기능은 유료 버전에만 제공됨**

슬랙의 Slash command는 3초 이내 응답해야 합니다. (참고자료: [Slack 문서](https://docs.slack.dev/interactivity/implementing-slash-commands/#responding_basic_receipt)) Cloud Run에서 동작하는 컨테이너 인스턴스가 없다면, 콜드 스타트 때문에 처음 요청은 실패한다고 생각해야 합니다. 비용이 안 나가는 것이 우선이라는 점, 그리고 저 혼자만 쓰고 있다는 점 때문에, 이 문제는 그냥 그런가 보다... 라고 생각하고 받아들이고 있습니다. 

한편, 슬랙 내 워크플로 기능을 써 보면 어떨까? 라는 생각을 했는데요. 해당 기능은 유료 버전에만 제공하고 있습니다. ([참고](https://slack.com/intl/ko-kr/help/articles/17542172840595-%EC%9B%8C%ED%81%AC%ED%94%8C%EB%A1%9C-%EA%B5%AC%EC%B6%95--Slack%EC%97%90%EC%84%9C-%EC%9B%8C%ED%81%AC%ED%94%8C%EB%A1%9C-%EB%A7%8C%EB%93%A4%EA%B8%B0)) 테스트용으로 저 혼자 쓰는 슬랙 워크스페이스가 있는데 굳이 이걸 위해 돈을 지불하기는 어렵다고 판단했습니다. 새로 구매하는 앨범에 대해서는 링크를 제공하는 정도로만 기능을 구현했습니다. 

{{< figure src="/img/albums-with-cloud-run-add-album.gif" caption="앨범 추가 1단계: 앨범 추가를 위한 사이트 링크를 가져온다" >}}

{{< figure src="/img/albums-with-cloud-run-add-album-2.gif" caption="앨범 추가 2단계: 웹 브라우저에서 정보를 입력한다" >}}

## 마무리

배포 후 1달 넘게 이 애플리케이션을 사용하고 있는데요. 아래와 같이 비용이 전혀 나가지 않은 것을 볼 수 있었습니다. 목표로 했던 것을 제대로 달성한 느낌입니다.

{{< figure src="/img/albums-with-cloud-run-3.png" caption="비용 관련 통계" >}}

그리고 최근에 알라딘 중고서점에 가서 [구매한 앨범](https://open.spotify.com/album/2qivROlvQ8BcUKTaCA7dL2)이 있는데요. 텐션을 올리기에 좋은 노래가 많아서 한 번 찾아보니, 실물 앨범을 구매한 적이 없었네요. 알라딘에서 찾아 보니, 마침 노원역 근처에 중고 음반이 있어서 저렴하게 들여올 수 있었습니다. 아래와 같이 간단하게 검색이 가능합니다. 아티스트 이름과 앨범 제목을 포함해서 검색해 줍니다.

{{< figure src="/img/albums-with-cloud-run-search.gif" caption="앨범 검색 과정" >}}

최근 몇 달 간 구직 중이어서 시간이 많았습니다. 거창한 건 못 해도 개인적으로 필요한 건 만들어 볼 수 있을 거라 생각해서 이런 작업을 해 봤는데요. 이 과정에서 AI 툴들이 이런 소소한 작업들을 저 대신 해줄 수 있고, 여러 도구를 실무에서는 어떤 방식으로 사용하면 좋을지 고민해 볼 수 있는 시간이었습니다. 한편으로 무언가를 만들어 보고 싶은 사람이 있다면, 적은 비용으로 여러 시도를 해 볼 수 있는 방법이 많겠다는 생각을 했습니다. 

앞으로 이 프로젝트는 개인적으로 계속 운영할 생각이고, 정말 의미가 있다면 각자가 가진 앨범을 공유하는 형태로 확장해 볼 수도 있을 것 같다는 생각도 듭니다. 다만 본업을 일단 잘 해야 할텐데요. 본업을 충실히 하며 틈틈이 소소한 시도를 해봐야 겠습니다. 읽어주셔서 감사합니다.

[GitHub 저장소](https://github.com/rubysoho07/MusicManager)