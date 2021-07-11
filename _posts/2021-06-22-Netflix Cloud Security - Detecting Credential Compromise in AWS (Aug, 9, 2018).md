# 넷플릭스 정보 보안 : AWS에서 자격증명 손상 탐지하기
## Will Bengtson, Netflix Security Tools and Operations
---
\* 본 포스팅은 학습목적으로 Netflix Information Security : Detecting Credential Compromise in AWS 글을 번역한 것으로 정확한 내용은 다음 링크에서 원문을 참고하시기 바랍니다.

https://netflixtechblog.com/netflix-cloud-security-detecting-credential-compromise-in-aws-9493d6fd373a

---
클라우드 환경을 운영하는 모든 사람에게 자격증명 손상은 중요한 문제입니다. 조직에서 이러한 자격증명 손상에 대해 지속적인 탐지와 대응을 하지않고 클라우드 리소스를 인프라의 일부로 사용할 경우 시간이 경과될수록 문제는 더 명백해집니다. 자격증명 손상이 일으키는 주변 효과는 매우 다양합니다. 공격자는 이러한 일을 발생시켜 당신의 인스턴스를 Bitcoin을 채굴하는데 사용할지도 모르고 심지어 데이터를 탈취하고 인프라를 삭제할 수는 등의 더 심각한 상황이 발생할 수 있습니다.

우리 넷플릭스 보안 운영 팀은 여러분의 AWS 환경 외부에서 사용되는 임시적인 보안 자격 증명을 탐지하는 새로운 방법을 공유하고 싶습니다. 당신의 AWS 환경이 "당신의 AWS 계정과 연결된 모든 AWS 리소스"라고 생각하세요.

## 장점
- AWS에 IP 할당에 관한 사전 지식 없이 여러분의 환경 외부에서 EC2 임시 보안 자격증명을 사용한 API Call을 탐지할 수 있게됩니다.
- 당신은 0에서 전체 복구까지 6시간 안에 할 수 있을 겁니다.
- 이 방법은 잠재적 손상을 확인하기 위해 과거의 AWS CloudTrail 데이터 뿐만 아니라 실시간으로 적용될 수 있습니다.

## 범위
이 포스트에서는 환경 외부에서 손상된 AWS 인스턴스 자격증명(STS 자격증명)을 탐지하는 방법을 보여줍니다. ECS, EKS 등의 다른 임시 자격증명을 사용하여 이 작업을 수행할 수도 있음을 참고하세요.

## 유용한 점
공격자는 당신의 일반적인 자격 증명 손상의 탐지 방법 뿐만아니라 애플리케이션이 실행되는 위치 또한 이해합니다. AWS를 공격할때 공격자는 주로 그들의 AWS 계정에서 탈취한 여러분의 AWS 자격증명 사용하려합니다.
아마도 여러분은 이미 여러분의 환경에서 "위험한" AWS API 호출에 주의를 기울이고 있을 수도 있습니다-이는 훌륭한 첫 걸음입니다.--하지만 공격자는 여러분이 주의를 기울일 것이라는 것을 알고 위험하지 않은(무해한) API 호출을 먼저 시도할 가능성이 높습니다. 다음 단계는 API 호출이 당신의 환경 외부에서 발생하는지를 확인해야합니다. 지금은 일반적으로 사용하는 AWS IP 주소공간이 많이 알려져 있기 때문에 API 호출이 AWS 외부에서 발생하는 지는 쉽게 확인이 가능합니다. 만약에 여러분의 IP가 아니고 AWS IP에서 발생한 경우 우리는 다른 방법(마법)이 필요합니다. 바로 그것이 우리가 여기에 포스팅한 방법입니다.

## 동작 방식
우리가 소개할 방법을 이해하기 위해서 첫번째로 여러분은 AWS가 자격증명을 EC2 인스턴스로 전달하는 방법과 CloudTrail 항목을 레코드별로 분석하는 방법을 알아야합니다.  

우리는 먼저 CloudTrail에서 추출한 모든 EC2 Assume Role 데이터 테이블을 구성하겠습니다. 각 테이블 항목은 인스턴스ID, Assumed Role, API 호출 IP Address, TTL 값(TTL이 이 테이블을 작게 만드는데 도움됨)을 나타냅니다.
우리는 인스턴스에서 API 호출의 Source IP를 검사하여 호출자가 AWS 환경안에 있는지 빠르게 확인할 수 있습니다.

![Picture1]({{site.url}}/assets/img/2021-06-22/(Aug,%209,%202018)%20Netflix%20Cloud%20Security%20-%20Detecting%20Credential%20Compromise%20in%20AWS-1.png)

## Assume Role
여러분이 IAM 역할로 EC2 인스턴스를 시작하면 AWS EC2 서비스가 인스턴스에 지정된 역할을 맡고 해당 임시 자격 증명을 EC2 metadata 서비스에 전달합니다. 이 AssumeRole 작업은 아래 키 필드로 CloudTrail에 표시됩니다.  

>eventSource : sts.amazonaws.com  
>eventName : AssumeRole  
>requestParameters.roleArn : arn:aws:iam::12345678901:role/rolename  
>requestParameters.roleSessionName : i-12345678901234  
>sourceIPAddress : ec2.amazonaws.com  

우리는 CloudTrail Log에서 위와 같은 임시 인스턴스 자격증명에 대한 Amazon Resource Name(ARN)을 확인할 수 있습니다.
AWS는 EC2 메타데이터 서비스의 자격증명을 1-6시간마다 새로고침(refresh)한다는 것을 염두하세요.  

위에서 표시된 EC2 서비스의 AssumeRole 작업 기록을 아래 항목에 따라 정리해보겠습니다.  

>Instance-Id  
>AssumedRole-Arn  
>IPs  
>TTL  

우리는 requestParameter.roleSessionName 필드에서 Instance-Id를 확인할 수 있습니다. 각 AssumRole 작업에 대해 위의 표에 존재하는 값(항목)이 있는지 확인합니다. 항목이 없으면 하나 새로 만듭니다.
항목이 존재하면 TTL 값을 업데이트하고 이를 유지합니다. 인스턴스가 계속 실행 중이므로 이 항목이 만료되지 않도록 하고싶을때 우리는 TTL을 업데이트 합니다. AWS에서 주기적으로 refresh하기 때문에 안전한 TTL은 6시간이지만 더 길게 TTL을 만들 수도 있습니다. 여러분은 AssumeRole CloudTrail 기록에서 requestParameters.roleArn와 requestParameters.roleSessionName을 가져와서 AssumedRole-Arn을 구성할 수 있습니다.  

예를 들어, 위 항목에 대한 AssumedRole-Arn은 아래 결과로 나타납니다.

>arn:aws:iam::12345678901:assumed-role/rolename/i-12345678901234

이 AssumeRole-Arn은 CloudTrail에서 이러한 임시 자격 증명을 사용하는 모든 호출에 대한 userIdentity.arn 항목으로 표시됩니다.
This AssumeRole-Arn becomes your userIdentity.arn entry in CloudTrail for all calls that use these temporary credentials.

## AssumedRole 호출
이제 우리는 Instance-ID와 AssumeRole-ARN 테이블이 있으므로 우리는 임시자격 증명을 사용하여 각 CloudTrail 기록 분석을 시작할 수 있습니다. 각 인스턴스ID/세션 행은 고정된 IP 주소 없이 시작됩니다.(우리는 이 방법을 사용하면 모든 IP 주소를 사전에 확인할 필요가 없다고 했습니다.)

CloudTrail 이벤트에서 그것이 assumed role에의해 생성된 것인지 확인하기 위해 해당 레코드의 유형을 분석합니다. userIdentity.type 값과 AssumedRole이 같은지를 확인하면됩니다. 만약에 AssumedRole이면 테이블에서 AssumeRole-Arn 열에 해당하는 userIdentity.arn 열을 잡습니다. 
userIdentity.arn은 requestParameters.roleSessionName 값을 가지고 있으므로 우리는 instance-id 추출할 수 있고 테이블에서 행이 존재하는 지 확인할 수 있습니다. 만약에 행이 존재하면 우리는 어떤 IP가 이 AssumeRole-Arn으로 잠겨있는지 확인해야합니다. IP가 없는 경우 레코드의 sourceIPAddress로 테이블을 업데이트 하고 이는 모든 호출이 수신되어야 하는 우리 IP 주소가 됩니다. 그리고 이 부분이 전체 방법에서의 핵심으로 만약 sourceIPAddress에서 호출을 확인했을 때 이전에 관찰된 IP 주소와 일치하지 않는 경우

```
우리는 해당 자격 증명이 할당된 인스턴스 이외에 다른 곳에서 사용되고 있음을 탐지했고 이는 자격 증명이 손상되었다고 가정할 수 있습니다.
```

테이블에 해당하는 행이 없으면 이러한 판단이 불가능하므로 그런 CloudTrail 이벤트는 삭제합니다. 하지만 AWS가 EC2내에 임시 자격 증명을 처리하는 방식으로 인해 최대 6시간 동안만 이런 현상이 지속됩니다. 그 이후에는 우리 환경에서는 모든 AssumeRole 항목을 가지게 되므로 더 이상 이벤트를 삭제할 필요가 없습니다.

## 예외 사례
오탐(가짜긍정-false positive)의 방지를 위해 몇 가지 예외 사례를 생각해야합니다.
- 특정 API 호출에서 AWS는 사용자의 자격증명을 사용하고 사용자를 대신하여 호출하는 경우
- 만약 당신의 서비스에서 AWS VPC Endpoint가 있는 경우 이러한 호출은 연결된 프라이빗 IP 주소로 로그에 표시됩니다.
- 인스턴스에 새 ENI를 연결하거나 새 주소를 연결하는 경우 CloudTrail에서 AssumedRole-Arn에 대한 추가적인 IP를 확인할 수 있습니다.

## 사용자를 대신하여 AWS에서 호출
CloudTrail 레코드를 살펴보면 앞에서 언급한 ***AssumeRole*** 작업 외부에 ***\<servicename\>.amazonaws.com***으로 표시되는 ***sourceIPAddress***를 볼 수 있습니다.
여러분은 이러한 계정에 호출이 나타나는 것을 설명하고 이러한 호출에서 AWS를 신뢰하려 할 것 입니다. 이런 정보를 지속 추적하고 정보 알림을 제공하는 것이 좋습니다.  

## AWS VPC Endpoints
서비스에 대한 VPC 엔드포인트를 사용하는 VPC에서 API 호출을 수행한 경우 ***sourceIPAddress***는 인스턴스에 배정된 퍼블릭 IP 또는 VPC의 NAT 게이트웨이 주소 대신 프라이빗 IP 주소를 보여줍니다. 이 경우 테이블에 지정된 ***instance-id/AssumeRole-Arn*** 행에 대한 [퍼블릭IP, 프라이빗 IP] 목록이 필요합니다.

## 인스턴스에 새 ENI 또는 새 주소를 연결한 경우
EC2 인스턴스에 새 ENI를 연결하거나 새 IP 주소를 연결하는 경우가 있습니다. 이럴 경우 CloudTrail 레코드에서 AssumedRole-Arn에 대한 추가 IP를 확인할 수 있습니다. 오탐을 방지하기 위해 다음 작업을 고려해야합니다. 예외 사례를 해결하는 한가지 방법은 인스턴스에 새 IP를 연결하는 CloudTrail 레코드를 검사하고 새 IP가 인스턴스에 연결될 때 마다 행이 있는 테이블을 생성하는 것입니다. 이는 CloudTrail에서 보여지는 잠재적 IP 변경 수를 이야기합니다. 
만약 잠금 IP와 일치하지 않는 sourceIPAddress가 표시되면 인스턴스에 대해 새 IP에서 호출이 있는지 확인하세요. 새 IP가 있다면 이 IP를 AssumeRole-Arn 테이블 항목의 IP 열에 추가하고 연결을 추적하는 추가 테이블에서 항목을 제거하고 알람을 제거합니다.


## 공격자가 먼저 접근
이러한 질문을 할 수 있습니다 : "우리가 잠금 IP를 자격 증명과 함께 보여지는 첫 API 호출로 설정했을 때 공격자의 IP가 잠금 IP인 경우가 없을까요?" 네, 이 접근 방법으로 인해 자격 증명이 손상되어 공격자의 IP를 잠금테이블에 추가할 약간의 가능성이 있습니다.
드문 경우로 공격자의 IP 잠금 후 EC2 인스턴스가 첫 번째 API 호출을 수행할 때 "손상"을 감지합니다. 이를 최소화하기 위해 EC2 인스턴스를 처음 부팅할 때 CloudTrail에 로깅되었음을 알리는 AWS API 호출을 수행하는 스크립트를 추가할 수 있습니다.

# Summary
우리가 공유한 방법론은 높은 수준의 CloudTrail 친숙도와 AssumRole 호출 로깅 방법이 필요합니다. 하지만 당신의 AWS 환경이 성장하고 계정 수가 증가함에 따른 확장성과 계정에 할당된 IP 주소의 전체 목록을 유지할 필요가없는 단순성 등 여러가지 이점이 있습니다. 
"깊이 있는 방어"가 진리임을 명심하세요.
이는 AWS에서 보안 전술의 계층을 하나로 구성해야함을 의미합니다.

여러분의 환경에서 이러한 방법보다 더 나은 방법이 있으면 꼭 알려주세요.

---

# 번역 리뷰
거창한 방법을 설명하는 것 같지만 EC2 인스턴스의 Role이 사용하는 첫 IP와 이후 변경된 IP를 추적하여 내가 가지고 있지 않은 목록의 IP가 등장하면 탈취된 것이라고 간단하게 요약할 수 있습니다. 번역하기 전에는 뭔가 어려워 보였는데 번역을 해보니 쉽게 이해한 것 같습니다. (혹시 필자가 잘못이해하고 있다면 꼭 피드백을 부탁드립니다.) 
지금은 AWS에서 GuardDuty나 Detective, Security Hub 같은 다양한 보안 서비스가 나온 만큼 이 포스팅과 같은 방법이 자동화 되어있을 것으로 보이네요.(GuardDuty에서 Newly observed API Calls같은 behavior을 확인하면 될 것 같네요.)
앞으로도 이런 리뷰를 자주해야겠습니다. 공부가 많이 되는 거 같아요 ㅎㅎ
다음 번역은 이 포스팅의 후속인 Netflix Information Security: Preventing Credential Compromise in AWS(https://netflixtechblog.com/netflix-information-security-preventing-credential-compromise-in-aws-41b112c15179)를 할 예정입니다.
끝까지 읽어 주셔서 감사합니다.
