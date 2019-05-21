# 하이퍼레저 기반 검수 시스템 Transaction Flow

하이퍼레저 패브릭 런타임 트랜잭션 처리 흐름은 다음과 같습니다



## 1. 어플리케이션 SDK의 트랜잭션 제안

**어플리케이션 SDK**, 즉 클라이언트(여기서 말하는 클라이언트는 검수 시스템에서 책을 등록하는 클라이언트가 아닌 트랜잭션을 제안하는 주체 노드를 말합니다 i.e. CLIENT, QA)가 **블록체인 네트워크에 트랜잭션을 요청**합니다. 검수 시스템에 경우 CLIENT가 책을 등록하는 createBook이나 QA들이 피드백을 작성하는 addFeedback 과 같은 **트랜잭션을 실행하기 위해 요청(제안)하는 것**입니다

그 **트랜잭션에 대해 승인하는 피어를 endorsing peer(승인 노드)**라고 하는데, 클라이언트는 이러한 endorsing peer들에게 해당 트랜잭션 요청을 전송합니다. 이 요청은 **transaction proposal이라는 형태로 전송**되는데, 어플리케이션 SDK는 이를 gRPC 프로토콜로 전송할 수 있는 프로토콜 버퍼 형태로 변형합니다. 또한 사용자의 신원 정보를 서명 형태로 proposal에 기입합니다



![](./image/step1.png)





## 2. Endorsing peer(승인 노드)가 서명을 확인하고 트랜잭션 실행

Endorsing peer가 어플리케이션 SDK로부터 트랜잭션 요청(제안)을 전달 받으면, 다음과 같은 절차로 해당 요청을 처리합니다. 이해를 돕기 위해 QA가 책에 피드백(검수)을 진행하는 트랜잭션을 요청한다고 가정합시다



1. **트랜잭션 형식에 맞게 내용이 잘 채워져 있는지 확인합니다 **

   - QA는 피드백을 진행하기 위해 comment를 작성해야 한다. 즉, 이 부분이 잘 채워져 있는지 endorsing peer들이 확인합니다

   - QA는 어떤 Book에 대해서 피드백을 작성할 것인지도 명시해야 한다. 즉, 이 부분이 잘 채워져 있는지 endorsing peer들이 확인합니다

     

2. **이전에 제출된 적이 있는 트랜잭션이 아닌지 확인합니다**

   - Endorsing peer들은 피드백을 진행하는 트랜잭션이 이전에 제출된 것은 아닌지 확인합니다

     

3. **서명이 유효한지(MSP를 통해서) 확인합니다**

   - 해당 트랜잭션을 요청한 당사자가 MSP를 통해 신원이 확인되는지(승인되지 않은 사람이 검수를 진행하면 안되니까) 확인합니다. MSP는 Membership Service Provider의 약자로, 블록체인 내 신원 인증 기관(PKI)이라고 생각하면 됩니다

     

4. **트랜잭션을 제출한 클라이언트가 그럴 권한이 있는지 확인합니다 **

   - 해당 트랜잭션을 요청한 당사자가 QA가 맞는지(CLIENT가 검수를 하는건 말이 안되니까) 확인합니다



이렇게 해서 확인이 되면, **endorsing peer는 트랜잭션 요청을 인자로 받아서 체인코드(우리 프로젝트의 경우 자바스크립트로 작성된 트랜잭션)를 실행**합니다. 이렇게 실행된 체인코드는 현재 상태 데이터베이스에 대해 실행되어 **응답값, readSet/writeSet 을 포함하는 결과값을 반환**합니다. 이때, 실행된 트랜잭션이 바로 블록체인에 적용되지 않고(원장을 업데이트 하지 않음), **반환된 결과값을 proposal response의 형태로 어플리케이션 SDK로 전송**합니다



![](./image/step2.png)



## 3. Proposal response 검토

어플리케이션 SDK이 **endorsing peer로 부터 돌려받은 proposal response를 비교하는 작업**을 거칩니다. 이때 어플리케이션 SDK는 **Endorsement policy**(경모 선배가 그렇게 강조하던 체인코드 폴리시가 이걸 말하는 겁니다)**를 만족하는 proposal response가 왔는지(특정 endorsing peer로 부터 결과를 받았는지) 검토**합니다. 이 단계에서 SDK가 endorsement policy를 검토하지 않는다고 해도 committing 단계에서 각 피어가 별도로 검토합니다

![](./image/step3.png)



## 4. 어플리케이션 SDK가 트랜잭션 전달

검토가 완료되면 어플리케이션 SDK가 **Transaction message에** 1단계에서 요청한 **Proposal과** 3단계에서 받은 **Proposal resopnse를 담아서** **Ordering service(정렬자)에게 보냅니다**. 트랜잭션에는 **read/write set이 들어가 있고, endorsing peer의 서명과 채널ID가 포함**되어 있습니다. **Orderer는** 트랜잭션 내용에 관계 없이, **모든 채널에서 발생하는 트랜잭션을 받아서 시간 순서대로 정렬**하여 블럭을 생성합니다

Orderer가 트랜잭션을 정렬하는데 **합의하는 방식은 기본적으로 pluggable(변경 가능한)**하나, 현재는 개발/테스트 환경에서 합의하는 **SOLO**와 프로덕션 환경에서 합의하는 **Kafka** 두 가지 알고리즘을 제공합니다



![](./image/step4.png)



## 5. Committing Peer에서 트랜잭션을 검증하고 커밋

**트랜잭션 블럭이 해당 채널의 모든 Committing Peer에게 전달**됩니다. 각각의 committing peer는 **블럭 내의 모든 트랜잭션이 각각의 endorsement policy를 준수하는지, world state 값의 read-set 버전이 맞는지**(read-set 버전은 트랜잭션 이후에만 변경됨) 확인합니다. **검사 과정이 끝나면 블럭 내의 트랜잭션에는 valid/invalid 값이 태그**됩니다



![](./image/step5.png)



## 6. 블록체인에 블록 추가

각 피어는 최종 검증을 마친 블럭을 채널 내 체인에 이어붙입니다. 그리고 유효한 트랜잭션의 경우 write-set이 stateDB에 입력됩니다. 이러한 과정이 끝나면 이벤트를 발생히켜 어플리케이션 SDK에게 작업 결과에 대해 알립니다



![](./image/step6.png)





# 트랜잭션 흐름도

위 과정을 하나의 그림으로 나타내면 다음과 같습니다



![](./image/transaction_flow.png)





최종 수정일 -  2019년 5월 21일