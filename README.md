#2018 Naver Campus Hackday summer - 초당 1만번 요청에도 끄떡없는 선착순 쿠키 발급 시스템 서버 개발

## 개발환경
- Ubuntu 16.04 (64-bit)
- Node.js v8.11.2
- Redis 4.0
- MySQL 5.7

- Naver Cloud Platform Server 
  - hackday-server01
    - Ubuntu Server 16.04 (64-bit)
    - 8vCPU, 8GB Mem, 50GB Disk + HDD 100GB
  - hackday-server02
    - Ubuntu Server 16.04 (64-bit)
    - 8vCPU, 8GB Mem, 50GB Disk + HDD 100GB
  - hackday-server03 (DBServer : Redis & MySQL)
    - Ubuntu Server 16.04 (64-bit)
    - 4vCPU, 16GB Mem, 50GB Disk + HDD 100GB

## 부하 테스트 하드웨어 & 소프트웨어 환경
- Windows
- Intel Core i3-2120 CPU, 8GB Mem, SSD 250GB
- JMeter 이용

## Database Structure
![database structure](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/Structure.png?raw=true)

## Server Structure
![server structure](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/structure1_1.png?raw=true)

## Simple Flowchart
![simple flowchard](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/structure2.png?raw=true)

### 1) 예약 페이지 & 조회 페이지
![reservation_page](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/%EC%98%88%EC%95%BD%ED%8E%98%EC%9D%B4%EC%A7%80.PNG?raw=true)
이렇게 예약을 하고, 조회는 간단하게 다음과 같이 할 수 있다.

![cookie state](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/%EC%A1%B0%ED%9A%8C%ED%8E%98%EC%9D%B4%EC%A7%80.PNG?raw=true)

### 2) 쿠키 발급
- 일정 기간동안 네이버 웹툰의 쿠키를 발급해 주는 이벤트를 진행한다고 가정한다.
- 초당 1만 건 이상의 GET 요청이 무작위 ID와 함께 들어온다.
- 하지만 시간상 실제로 1만 건에 대해 제대로 테스트 환경을 갖추지 못했고, 일반 데스크탑으로 테스트를 진행하였다.

### 3) Redis Cache 및 MySQL DB동시 이용
- 속도를 위해 메인 메모리 기반의 DB를 이용할 필요가 있었다. 하지만 __쿠키__ 특성상 불안정한 Redis보다는 MySQL DB로 관리해야 했다. 그래서 중요한 데이터를 Redis에 올릴 순 없었다. 그래서 Redis는 단지 누가 먼저 요청을 했는가에 대한 번호표를 발급해 주는 역할만을 하도록 구현했다.
- MySQL에 저장된 쿠키 데이터들은 어느 시점에 조회해도 정확한 결과를 돌려주어야 한다. 그래서 서버에서 쿠키 발급 로직을 진행한다면 동시에 MySQL에도 적용시켜야 한다.

### 4) Transaction과 Lock
- 우선, 순간적인 트래픽을 우선 처리하기 위한 번호표를 엄청나게 나눠주게 되고, 쿠키를 발급하기 위해서는 현재 실제로 발급된 쿠키 수를 정확하게 알아야 한다. 그리고 정확하게 발급해야 한다. 여러 사용자가 동시에 요청하며 DB에 접근하게 되므로 이를 해결하기 위해서는 코드 레벨의 Lock이 필요한데, 이를 위해 mysql에서 __cookie_sync__ 라는 테이블을 만들었다. Lock을 걸 때는 __[어떤 유저가 / 어떤 쿠키를 / 언제 / 락을 걸었는가(has_locked : 0 -> 1)]__ 에 대해 작성하고, 락을 해제할 때는 락을 해제한 정보로 업데이트한다(has_locked : 1 -> 0). 이렇게 해당 쿠키에 대해 동시에 다른 작업이 진행되는 일을 방지한다.

### 내용 업데이트
2개의 서버를 사용하므로 실제로 쿠키를 발급해 줄 때 공유 데이터에 대한 락을 걸어야 한다. Redis에서 락 정보를 기록하고, Polling 방식을 통해 주기적으로 상태를 확인한다. 락을 잡게 되면 쿠키를 발급해 주는 식으로 구현한다.

Redis는 Single Thread 기반으로 Queue에 입력된 쿼리를 순차적으로 수행하는데 그렇기 때문에 Redis에서 락 정보를 다루도록 했다. 여러 클라이언트들의 요청으로 인해 트랜잭션 꼬여 원하는 순서대로 작업이 진행되지 않을 때는 Multi를 이용하여 하나의 트랜잭션으로 묶어 준다.

### 5) 로드밸런싱
ncp 로드밸런싱을 이용했다. 위에서 설명한 서버 구조와 같이, LB를 이용해 2개 서버를 연결하여 진행했다.
![LB](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/LB.PNG?raw=true)

### 6) 테스트
#### 테스트 환경을 일반 데스크탑으로 진행해서 그런지 문제가 있는 것 같다. 추후 알맞는 환경을 구성하여 다시 테스트해서 업데이트할 것이다.
- 테스트는 JMeter를 이용하여 진행하였다.
- 쿠키 1000개 / 1000명의 유저가 규칙적인 ID로 1초 간격으로 GET 요청 10번씩
  - 응답 시간이 4.4 ~ 8.8초 사이인 것을 볼 수 있다.
![CP1000_T1000X10_SEC1](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/CP1000_T1000X10_SEC1.PNG?raw=true)

- 쿠키 10000개 / 500명의 유저가 규칙적인 ID로 1초 간격으로 GET 요청 100번씩
  - 응답 시간이 대체로 0 ~ 8초 사이인 것을 볼 수 있다. 대체로 1초 및 4초 이내로 응답한다.
![CP10000_T500X100_SEC1](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/CP10000_T500X100_SEC1.PNG?raw=true)

- 쿠키 10000개 / 1000명의 유저가 규칙적인 ID로 1초 간격으로 GET 요청 50번씩
  - 응답 시간이 대체로 0 ~ 8.8초 사이인 것을 볼 수 있다. 대체로 4초 이내에 응답한다.
![CP10000_T1000X50_SEC1](https://github.com/hsb0818/NaverHackday2018Summer_CookieServer/blob/master/src/CP10000_T1000X50_SEC1.PNG?raw=true)


### 결과
- 나의 데스크탑 사양으로 테스트해본 결과 쿠키 10000개 대상으로 1000명 유저가 50번, 즉 5만 번 요청을 받았는데 대체로 __[1초 / 4초]__ 이내에 응답을 하는 것을 볼 수 있었다. 비록 현재는 내 __개인용 데스크탑__ 에서 JMeter를 이용해 여러 스레드를 만들어 진행했지만, 만약 테스트 환경을 잘 구축하여 진행한다면 더 혹독한 환경에서도 잘 응답하는 것을 볼 수 있을 것이라 생각한다.

#### 추가 고려사항
- Redis를 단지 대기번호표 주는 용도로만 사용하는게 아쉽다. 좀 더 효율적인 처리를 위한 구조를 생각해 볼 여지가 있다.
- Lock에 대한 부분. 현재는 복합 키로 빨리 검색할 수 있도록 구성했지만, 이런 식으로 Lock을 기록하는 것보다 효율적인 방법이 있을지 고민해 봐야겠다.
- 테스트.. 잘 진행해 보고 싶었다.


## 해커톤 소감(일기ㅎㅎ)
저번 해커톤에서는 초당 500개의 댓글을 처리하는 서버를 개발했었는데, 이번엔 초당 1만번 요청이라니! 네이버 핵데이를 통해 많은 것을 배우게 되는 것 같다. 멘토님께도 많이 배우지만, 팀원 분들에게서도 여러가지를 배우면서 개발할 수 있어 즐거웠다. 언젠가 꼭 훌륭한 서버개발자가 되어야지..
