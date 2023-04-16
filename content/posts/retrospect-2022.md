---
title: "2022년 회고 - 팀 운영 후기"
date: 2022-12-24T00:00:00+09:00
tags: [회고, 리더십]
draft: false
categories: [Everything]
comments: true
---

2022년은 우여곡절이 있었던 해였습니다. 주변의 여러 상황 때문에 팀 리더라는 직책을 맡게 되었습니다. 사실 재직 중인 회사에 입사하면서 고려했던 부분은 아니었습니다. 지난 회사에서도 팀 빌딩 경험이 있다 보니, 이번에는 피하고 싶다는 생각이 있었기 때문입니다. 그렇지만, 회사 생활이라는 것이 제 마음대로 되는 게 아니더라구요. 다행히 제가 요청한 대로 TO가 나왔고, 계획에 맞게 인원을 뽑을 수 있었습니다. 올해 회고는, 1인 팀에서 세 명으로 팀을 키운 이야기, 그리고 이를 바탕으로 1년 간 팀을 운영했던 이야기를 해 보고자 합니다. 

# 채용 

## 어떤 기준으로 팀원을 채용할 것인가?

데브옵스나 인프라 관련 업무를 수행해 본 분이 없었다 보니, 제가 기준을 정할 수 있었습니다. 저는 데브옵스 엔지니어로 근무한 경력이 오래 되지는 않았기 때문에, 전에 재직한 회사에서의 경험을 바탕으로 기준을 세울 수밖에 없었습니다. 

기본적인 기준은 세 가지로 잡았습니다. 

1. 운영체제에 대한 이해: 서버 OS로 리눅스를 쓰는 경우가 많다 보니 이와 관련된 질문을 준비했습니다. 임베디드 환경 기반으로 개발을 했던 경험이 있어, 여러 레벨의 질문을 준비했습니다. 
2. 네트워크에 대한 이해: 네트워크에 대한 기본적인 이해가 필요하다고 생각해서, AWS 환경에서 네트워크를 구성할 때 겪을 수 있는 문제를 위주로 준비했습니다. 
3. AWS에 대한 이해: 팀 인프라가 모두 AWS에 있습니다. 그러다 보니 AWS의 여러 서비스에 대해 이해하고 있는지가 중요했습니다. 레벨은 일단 Solution Architect - Associate 자격증 시험에서 언급될 만한 것으로 준비했습니다. 

추가로 요구하는 경험은 다음과 같았습니다. 제가 이전 회사에서 경험하지 못했던 부분이지만, 서비스 인프라의 방향성을 잡는 측면에서 아래 내용이 중요하다고 생각했습니다. 

1. IaC(Infrastructure as Code)에 대한 이해: 사실 예전에 CloudFormation을 사용해 본 경험이 있긴 합니다. 다만 클라우드 벤더에 덜 의존하는 개발 환경을 만들기 위해 이번에는 Terraform을 사용해 보기로 했습니다. 실제로도 여러 Provider를 이용해야 할 일이 생겨서 좋은 선택이었다고 생각합니다. 채용 시에는 Terraform이나 CloudFormation의 사용 경험에 대해 질문 드렸습니다.
2. 컨테이너 기반 시스템의 이해: Docker나 Kubernetes에 대한 경험에 대해 질문 드렸습니다. 만약 이러한 경험이 있다면, 어떤 배경으로 컨테이너를 사용했는지, 운영 시 겪은 이슈는 무엇이었는지에 대한 추가 질문을 드렸습니다.

## 팀원 채용 과정

팀원을 채용하면서 느꼈던 것은, 회사의 인지도가 높은 편은 아니었다는 점, 그리고 회사에서 처음 채용하는 포지션이라 시행 착오가 많았다는 점이었습니다. 이전 회사에서는 팀장님이 따로 계셨고, 저는 이력서를 검토하여 진행 여부만 검토하거나, 기술 면접만 들어가서 의견을 드리는 역할이었습니다. 그래서 이번 회사에서 처음으로 전체적인 채용 프로세스를 주도하는 경험을 했습니다. 게다가 제가 채용을 진행할 때는, 회사에 리크루터가 없었기 때문에 채용 업무에 익숙하지 않았습니다. 그러다 보니 면접 프로세스에 대한 안내 중 실수한 적도 있었습니다. (예를 들어 면접 일정 안내가 제대로 안 되어서 놓친 분도 있었습니다. 이 부분에 대해서는 죄송하다는 말씀을 드리고 싶습니다.)

코딩 테스트는 따로 진행하지 않았습니다. 이직을 하면서 코딩 테스트를 보는 경우도 있긴 했습니다. 다만 사람을 가려서 뽑아야 할 상황이 아니라고 판단했고, 최대한 많은 후보를 만나뵙고 싶었기 때문입니다. 정말 지원자가 몰리는 경우라면 좀 더 까다롭게 보기 위해 테스트를 진행했겠지만, 데브옵스 엔지니어나 클라우드 인프라 엔지니어의 풀이 넓지 않은 상황이라 생각해서 코딩 테스트는 배제했습니다. 

실무 면접에서는 주로 기술적인 질문을 많이 드렸습니다. 쉬운 질문부터 어려운 질문으로 단계적으로 넘어가려고 노력을 했습니다. 어려운 질문부터 주면 당황하는 경우가 있고, 쉬운 문제에도 제대로 답변을 하지 못할 가능성이 있기 때문입니다. 그리고 "답변을 잘 못 하는 것이 합/불을 결정하는 것이 아니다. 지원자 분의 장점과 단점을 파악하기 위함이니 아는 대로 답변해 달라"는 말씀을 많이 드렸던 것 같네요. 

2차 면접으로는 문화 면접이 있었는데, 이 부분은 대표님이 주로 진행하다 보니 제가 말씀드릴 부분은 없어 보입니다. 

그 결과 우여곡절 끝에 두 분을 모실 수 있었고, 다행히 저보다 인프라 관련 업무를 오래 하신 분들이 합류하셨습니다. 

# 팀 운영

## 기본적인 규칙

팀 리더로서 어떤 일을 해야하는지 익숙하지 않아서, 다음 책들을 읽고 상위 리더들과 면담하면서 방향을 잡았습니다. 

* 팀장의 탄생, 줄리 주오 저 (지금 회사에 입사하기 한참 전에 읽었습니다)
* 개발 7년차, 매니저 1일차, 카미유 푸르니에 저
* 구글이 목표를 달성하는 방식 OKR, 크리스티나 워드케 저
* 실리콘밸리의 팀장들, 킴 스콧 저 (최근에 읽었는데, 바쁘다고 대충 읽어서 다시 한 번 읽어야 할 것 같습니다)

우선 회사의 기본 규칙과 충돌하지 않도록 다음과 같은 규칙을 설정했습니다. 

1. 새로운 인원이 합류하면 첫 1주는 팀원 모두 출근 (단, 저는 1~2 개월 간의 회사 내부 교육 기간 동안 계속 출근했습니다)
2. 이후에는 회사에서 지정한 필수 출근일을 제외하고 자유롭게 재택근무 선택 가능
3. 금요일 오후에는 다음주의 출/퇴근 시간, 휴가 사용 여부를 미리 공유할 것
4. 연차 사용은 회사 정책을 따름. 사유를 굳이 공유할 필요는 없으나 휴가 일정은 공유할 것
5. 스크럼은 매일 하지 않고 주 3회 시행 (예전에 같이 일하던 분들과 해 보니 이정도가 적당했던 경험이 있었습니다)

> 기본적인 규칙을 정할 때는, '구성원을 서로 신뢰할 수 있다'는 점이 중요하다는 생각이 들었습니다. 이를 위해서, 되도록 각자 하는 업무를 공개된 자리에서 공유하도록 했습니다. 한편, 구두로 이야기 하는 것도 최대한 정리해서 공개하려고 노력했습니다. 다른 사람에게 맡긴 경우도 있지만, 회의록이든 구두로 이야기 한 내용이든 문제가 없다면 제가 되도록 정리했습니다. 

> 팀원의 자율적인 재택 근무는 제가 재택 근무를 선호하는 편이어서 + 팀원들의 집이 사무실과 멀어서 결정한 사항입니다. '재택 근무 정책은 팀 단위로 자율적으로 설정할 수 있다'는 회사 정책 덕분에 쉽게 설정할 수 있었습니다. 다만, 재택 근무는 소통이 비동기적임에 유의해야 합니다. 간단히 말해서, 내가 말한 것에 대해 답변을 바로 받을 가능성이 낮아질 수밖에 없습니다. 제가 성격이 급한 것 같은데, 답변을 바로 받지 않으면 초조해 지는 경우가 있었습니다. 의사 소통의 특성을 고려하여, 상대방이 늦게 회신할 수도 있다는 점을 명심하려고 노력하고 있습니다. 

## 온보딩

회사 차원에서 제공하는 온보딩 코스가 있지만, 팀 내부에서도 별도의 온보딩 프로세스를 진행했습니다. 팀 온보딩은 다음과 같은 코스로 구성했습니다. 

1. 현재 인프라 구성과 개선이 필요한 사항
2. 네트워크와 리눅스에 대해 다시 한 번 짚어보기 
3. 각자가 가진 테크 스택을 고려하여, 익숙하지 않은 것을 연습하고 피드백 받기

첫번째, 두번째 항목은 개발 부서 분들이 같이 들을 수 있도록 했습니다. 서로 인사하는 기회가 될 수 있도록 하는 것이 주된 목표였구요. 개발 부서들의 경우 상대적으로 인프라 관련 내용에 익숙하지 않은 경우가 있기 때문에 이를 보충하는 것도 의도하였습니다. 다만 2번 교육은 타겟을 잘못 잡은 느낌이 들었습니다. 교육을 완료한 후에 생각해 보니, 저보다 오랫동안 인프라 관련 업무를 하신 분들이라 2번은 필요없겠다는 생각이 들었습니다. 저연차라면 이런 교육들이 도움이 될 수도 있었을 것 같습니다.

## 팀 내 프로세스

재직중인 회사는, 회사 차원에서 2주 단위의 스프린트를 수행합니다. 스프린트 마지막 날(금요일)은 회고와 다음 스프린트 플래닝을 진행했습니다. 애자일이나 소프트웨어 엔지니어링과 관련된 여러 책들을 읽어 보면 스프린트에 할 일을 정하는 여러 기법들을 소개하고 있습니다. 플래닝 포커라던가, 포스트잇을 붙인다던가, ... 사실 1년이 지난 자금까지도, 팀을 어떻게 운영해야 할 지, 이런 기법들이 우리 팀에 맞는 것인지 잘 모르겠습니다. 그러다 보니 간단하게, 해야 할 일과 할 수 있는 일을 모아두고 우선 순위를 정해서 진행하는 경우가 많았습니다. 

그리고 스프린트 내 업무 계획은 다른 부서에서도 볼 수 있도록 팀 슬랙 채널에 공유 드렸습니다. 왜 이런 선택을 했냐면, 제가 입사하기 전에는 데브옵스 팀이나 인프라 업무에 대해 어떻게 설명을 해야 할 지 몰랐기 때문입니다. 차라리 저나 저희 팀이 하는 업무 목록을 공유하는 게 더 빠르겠다는 생각이 들어서 팀이 하는 업무는 공개적으로 공유하려고 노력했습니다. 

> 생각해 보면 제가 일 욕심이 많아서 그랬던 것 같은데요. 스프린트를 돌아보면 계획했던 것보다 수행률이 떨어지는 경우가 많았습니다. 좀 여유있게 일정을 생각하는 습관을 들여야 할 것 같더라구요. 

그리고 중간 관리자 역할을 하다 보니 중간 관리자로서 상위 관리자와의 소통도 고려할 수 밖에 없었습니다. 구두로 의사결정이 이루어지는 것을 싫어하는 편이라, 문서화된 소통을 중요하게 생각했습니다. 상위 관리자와 소통할 때는 요약을 하거나 좀 더 보기 편하게 작성하려는 노력을 했습니다.

## 팀원과의 소통

지금 회사에 입사하기 전까지는 1:1 개념에 익숙하지 않았습니다. 그냥 각 개인과 면담하는 거 아닌가? 라는 생각이 들긴 했는데, 생각했던 것보다 팀을 운영하는 데 중요한 요소라는 생각이 들었습니다. 어떤 이야기를 할 지 고민하다가, 어떤 업무를 하고 있는지, 업무의 방해 요인은 무엇인지, 회사에 궁금한 것들은 무엇인지에 대해 주로 이야기를 했습니다. 시간이 좀 더 남으면 경력과 관련된 이야기도 해 보려고 노력했습니다. 다만 제가 남의 사생활에 대해 꼬치꼬치 캐묻는 것을 싫어해서, 사생활과 관련된 이야기는 본인이 이야기 하기 전까지는 이야기를 하지 않았습니다. (저도 저의 사생활에 대해 이야기하는 것을 별로 안 좋아하는 편입니다)

이전에 1:1 대화를 했던 기록을 찾아보며 느낀 방해 요인은, 세 가지였던 것 같습니다. 정리하고 보니 피할 수 없는 부분인 것 같더라구요. 최대한 이런 리스크를 줄이려고 노력해야겠습니다. 

* 비동기적인 소통에 따른 의사 결정 지체
* 여러 프로젝트를 진행하며 발생하는 context switching
* 인프라 관련 업무에서 빠질 수 없는(...) 긴급한 장애 대응

> 개인적으로는 1:1을 정기적으로 진행하지 못해서 아쉬웠습니다. 일이 바쁘다는 핑계로, 거의 1개월에 1번 정도로 진행했던 것 같네요. 어떻게 보면 일정을 최대한 강제하는 방법이 필요해 보였습니다. 또한 여러 주변 상황으로 인해 소통의 기회를 많이 가지지 못했다는 점, 저의 내향적인 성격 때문에 구성원 간 소통을 잘 챙기지 못한 부분도 아쉬웠습니다. 생각해 보면 small talk가 부족했던 것 같네요.

# 잘한 것과 잘못한 것들

## 잘한 것

제가 제 칭찬을 하려니 어색하지만, 잘 했다고 생각하는 점은 세가지로 압축할 수 있습니다. 

**첫째, 저 혼자 일하는 1인 팀에서 팀을 세 명으로 늘리고, 팀의 프로세스를 정착시킨 부분입니다.** 이 부분은 저희 팀 뿐만 아니라 같이 일하는 여러 분들의 도움이 컸습니다. 

**둘째, 회사 내부에 데브옵스 엔지니어가 없었기 때문에, 여러 프로세스들을 최대한 정착시킨 점입니다.** 예를 들어서 장애 대응 시 대응 프로세스가 있는데요. 전사에 장애 발생을 알리고, 고객 상담 시에도 상황 공유가 될 수 있도록 조치하고, 장애 상황이 끝나면 장애 리포트를 작성하여 공유하는 프로세스가 정착되었습니다. 

**마지막으로, 인프라 비용 증가를 어느 정도 통제하고, 새로운 기술이나 시장에서 통용되는 기술들을 많이 시도해 본 것입니다.** 이직 준비를 하면서 '다음 회사에서는 이걸 해 봐야지'라는 생각을 많이 했는데요. 그런 것들을 팀원 분들의 도움으로 이루어 낼 수 있었습니다. 

## 잘못한 것

물론 잘못한 것들도 있습니다. 

**첫번째로는 내향적인 성격이라 팀 내/외부 소통에 있어서 미숙했던 부분이 많았습니다.**

특히 비대면으로 일하는 경우가 많은데, 큰 리액션과 빠른 반응이 필요했다는 생각이 듭니다. 예를 들어 슬랙 메시지에 이모지를 달아 놓는다거나, 스레드로 답변을 드리는 것들 말이죠. 한 편으로는 이전에도 이야기 했듯 기본적인 마인드셋이 중요해 보입니다. '바로 답장이 올 가능성이 낮다'는 점을 인지하고 행동해야 겠다는 생각이 들었습니다. 

**두번째로는 협업 시 팀 내부 프로세스를 제대로 정립하지 못했다는 점이 아쉬웠습니다.**

예를 들어서 코드를 작성할 때 코드 컨벤션은 어떻게 할 거냐, 코드 리뷰는 어떤 것을 중점적으로 봐야 하냐와 같은 것들이 있겠네요. 다만 최소한 코드 변경 사항을 확인하고 Pull Request에 답글을 다는 부분, 스크럼, 스프린트 단위의 회고는 착실히 진행 중입니다. 

**세번째로는 진짜 해야 할 일을 제대로 못한 점입니다.** 

투입할 수 있는 리소스가 한정적이기 때문에, 좀 더 가용성이 높고 사람 손을 덜 타도록 인프라를 개선하고 싶었습니다. 하지만 회사 차원의 우선 순위가 있다 보니, 연초에 설정했던 목표 중 이루지 못한 것들이 있어 아쉬웠습니다.

**마지막은, 제가 '흔적이 남는 소통'을 중요시 하기 때문에 발생한 이슈입니다.** 

저희 회사는 주로 노션, 지라, 슬랙을 협업 툴로 이용하고 있는데요. 그러다 보니 노션 페이지라던가 지라 이슈, 슬랙 등으로 소통하는 경우가 많았습니다. 다만 업무의 히스토리가 여러 곳에 분산되어 있어서 정리가 안 되었던 문제가 있습니다. 하나의 인프라 구성이 어떤 과정을 통해 진화했는지 확인하려면 여기저기에 흩어진 자료를 찾아야 하는데요. 내년에는 이 부분을 개선해 보려고 합니다. 

# 마무리 

외부 업체에 계신 분들과 이야기할 때, '팀장님'이라는 말씀을 해 주시면 지금도 어색하다는 생각이 들더라구요. 전 회사에서는 '대리' 직급이자 팀원의 위치였고, 지금 회사에서는 그냥 '윤곤 님'으로 불리고 있어서 그런 것 같습니다. 

갑자기 팀 리더라는 자리를 맡으면서, 제가 지금까지 거쳐온 회사의 팀장님들을 떠올려 보았습니다. 생각해 보면, 모든 팀장님들과 동료 분들로부터 배울 점이 있었다는 생각이 들었습니다. 어떤 분은 팀 분위기를 자율적으로 이끌어 주셨고, 어떤 분은 다른 팀과 소통하는 스킬이 좋았고... 저는 그 과정에서 팀원에게는 자율성을 주고(물론 하는 일을 확인 해야겠죠), 말과 행동을 조심하려고 노력했습니다. 

특히, 리더의 말과 행동은 기본적으로 팀원에게 긍정적이든, 부정적이든 영향을 준다고 생각합니다. 그래서 팀 내부 또는 외부의 사람들에게 말이나 행동을 함부로 하지 않도록 노력했습니다. 다만, 이런 과정에서 제가 팀 리더로서 고민했던 점을 다른 사람들과 이야기 하는 걸 어렵게 생각했습니다. 아마 주변에 비슷한 역할을 하는 분이 없다 보니 조언을 구하기 힘들다는 생각이 들었기 때문인 것 같습니다.

2023년은 저에게 어떤 해가 될 지, 또는 어떤 역할로 업무를 수행할지 모르겠습니다. 만약 리더로서 계속 일하게 된다면 부족한 부분들을 보완해 보려고 합니다. 

읽어주셔서 감사합니다. 