만약 1:1 이 아닌 1:N의 상황에서 N(목적지IP, 내부 공인 or 사설 IP)인 경우에 출발지와 목적지를 검색할 때 목적지가 DNS 서버가 아닌 곳을 조건으로 줘서
검색을 했는데 DNS 서버가 아닌 곳이 나왔다? 그러면 DDoS 다중 서비스 방해 공격으로도 볼 수 있을 정도의 체급으로 이야기가 달라진다.    

Packet pipelining for high utiliztaion
Pipelining
: sender가 아직 ack pkts이 도착하지 않은("in-flight") 상태에서라도 한 번에 여러 개의 패킷을 전송한다.

시퀀스 넘버의 범위를 늘려야 함
sender나 receiver에서 버퍼링이 발생함

원래 sender의 Utilization 은 (L/R) / (RTT+L/R) 이었는데 만약 3개를 한 번에 보낸다면 U_sender = (3L/R)/(RTT+L/R)이 된다.
패킷을 3개를 한 번에 보냈더니 utilization이 3배 증가했다!

이러한 Pipelined protocols엔 크게 두 가지가 있다.

Go-Back-N
Selective Repeat
간략히 설명하자면 Go-Back-N은 한 번에 N개씩 보내는 프로토콜이다.
Receiver는 여러 개의 ack를 축적해서 보낸다. 만약 순서를 지키지 않고 중간에 다른 패킷이 오면 그 패킷은 받지 않고 ack를 보내지 않는다.
sender는 가장 오래된 unacked packet에 대한 타이머를 가지고 있다. 만약 타이머가 끝나면 그 패킷부터 N개를 다시 다 보낸다.

Selective Repeat은 N개를 보내는 것은 같지만 Receiver은 각 패킷에 대해 개별적인 ack를 보낸다. 그리고 sender 역시 unacked packet에 대해 모든 timer를 유지한다. 만약 타이머가 끝나면 오직 unacked 된 패킷에 대해서만 재전송을 한다. 즉 잘못된 것만 다시 보내는 것!

1. Go-Back-N (GBN)
패킷 헤더에는 k-bit의 sequence number가 있다.
window 는 보낼 패킷들의 묶음같은 것..! 총 N개의 unacked pkts으로 되어있다.

ACK(n) : n번째 패킷까지 다 잘 받았다는 뜻 (cumulative ACK)
Sender는 duplicate ACKs를 받을 수도 있다. (처리필요)
가장 오래된 in-flight pkt에 대한 타이머만 존재한다.
timeout(n) : n번째 패킷에 대한 ACK가 오지 않았으므로 n번째 패킷부터 N개를 다시 보낸다.

Sender
Sender는 이제 rdt_send(data)가 오면 현재 보고있는 seq# 인 base로 부터 N개까지를 한 번에 전송한다. 이러면 nextseq# 가 base로부터 N+1번째 패킷을 가리킬 것이고 이러면 더이상 보내지 않고 스탑한다.

그리고 맨 처음 패킷에 대해서 timer를 켜고 기다린다.

ack가 오면 base를 그 다음 seq #로 한 칸씩 옮겨준다. 즉 윈도우를 오른쪽으로 한 칸씩 옮겨준다. 이러다가 만약 base가 nextseqnum 즉 윈도우 맨 끝까지 오면 전부 다 잘 받은거니까 timer를 멈춰준다. 그 외의 경우에는 윈도우의 가장 왼쪽 패킷에 대한 timer를 켜준다.

만약 잘못된 패킷이 오면 아무것도 안한다.

time out이 되면 다시 타이머를 키고 현재 base로 부터 N개를 재전송한다. 즉 옮겨진 윈도우 안의 패킷을 싹 다 보낸다.


Receiver
Receiver은 패킷이 왔고, 정상 패킷이고, 순서에 맞는 패킷이라면 udt_send(sndpkt)으로 ACK를 보내준다. 그리고 expectedseq#를 하나 증가시켜준다. 만약 이상한 패킷이 오거나 중복된 패킷이 오거나 순서를 건너뛰고 오면 아무것도 안하고 기다린다.

ACK-only : 잘 받은 패킷 중 가장 마지막 패킷에 대한 ACK를 보내준다. (NAK안씀)
ACK가 중복될 수 있다.
오직 expected seqnum 만 기억하면 됨 (패킷 받을 때마다 expected 값 ++ )
만약 순서에 맞지 않은 패킷이 오면 버퍼가 없기 때문에 그냥 지워버리고 가장 마지막으로 받은 정상 패킷에 대한 ACK를 다시 재전송 해준다.
2. Selective Repeat
Receiver는 잘 온 패킷 각각에 대한 ACK를 각각 관리한다.
버퍼가 있어서 순서에 안 맞게 온 애들 임시로 가지고 있다가 순서 맞게 다 도착하면 상위 계층으로 순서대로 보냄
Sender는 ACK가 도착하지 않은 패킷만 다시 전송한다. 즉 각각의 unACKed pkt에 대한 타이머를 설정해야 함
Sender의 window
N개의 연속적인 seq #
전송할 packet의 개수를 제한한다.

Sender
위에서 data가 오면 지금 seq #가 윈도우 안에 있는 유효한 번호인지 확인하고 패킷을 전송한다.
Timeout(n) : 타임 아웃이 되면 n번째 패킷을 다시 보내고 타이머를 재시작 한다.
window 범위 내의 ACK(n)이 오면 pkt n을 받았다고 표시해주고, n이 윈도우의 가장 왼쪽 패킷이었다면 윈도우를 n번째 다음부터로 옮겨준다.
Receiver
window 범위 내의 ACK(n) 이 오면
ACK(n) 을 전송하고
순서에 맞지 않으면 buffer에 일단 저장해둔다
순서가 맞는다면 deliver (이 때 버퍼에 저장해둔 것까지 합쳐서)
그리고 윈도우를 아직 받지 않은 패킷까지로 옮겨준다.
rcvbase-N ~ rcvbase-1 범위의 ACK(n)이 오면
이미 받았던 패킷이니까 ACK(n)만 다시 보내주고 저장은 하지 않는다.
그 외에는 무시한다.


Dilemma
seq #가 window 크기보다 작으면 문제가 생길 수 있음!
(ACK 못 맏아서 재전송 한걸 새로운 패킷으로 인식할 수 있음)
따라서 window보다 seq #가 약 두 배정도 더 많아야 함.
Automatic Repeat Request (ARQ) schemes
Throughput performance analysis
Goal : maximum throughput (=𝜆_max) 구하기

t_I = 하나의 패킷을 transmission 하는데 걸리는 시간
𝑡𝑝 = propagation delay
𝑡𝑝𝑟𝑜𝑐 = processing 하는 시간 at the receiver 
(패킷이 라우터에 도착해서 메모리를 읽고,
어느 라우터로 전송되어야 하는지 목적지를 확인하는 데 걸리는 지연.)
𝑡𝑠 = ACK나 NAK의 transmission time
𝑡𝑜𝑢𝑡 = time out
-> ACK를 잘 받기 위해서는 time out 시간이 delay보다 길어야 함
𝑡𝑜𝑢𝑡 ≥ 2𝑡𝑝 + 𝑡𝑝𝑟𝑜𝑐 + 𝑡𝑠
1. Stop-and-Wait
가정

Saturated network (포화 네트워크) : transmitter는 항상 뭔가 전송할게 있음
완벽한 timeout 설정 : 𝑡𝑜𝑢𝑡 = 2𝑡𝑝 + 𝑡𝑝𝑟𝑜𝑐 + 𝑡𝑠
𝑡𝑇 = 𝑡𝑜𝑢𝑡 + 𝑡𝐼 라고 하자. 즉 𝑡𝑇 는 패킷이 queue 끝에서 전송되기 시작해서 time out 될 때까지의 시간

만약 error가 없다고 하면 𝜆_max = 1 / 𝑡𝑇
(𝑡𝑇가 패킷 하나 보내서 타임아웃까지의 시간이니까 1초에 보낼 수 있는 패킷의 양은 이것의 역수)
error가 있을 경우
𝑝 = 그 프레임이 에러가 있을 확률
𝑡𝑣 = 프레임을 연속적으로 받을 때 그 프레임 사이의 시간 간격
𝑁 = 재전송 횟수
이를 바탕으로 frame을 성공적으로 전송하기 위한 평균 시간(기댓값)을 구하면

재전송이 0번, 1번, ... , n번 일 때 걸리는 시간*확률

이 때


재전송 n번일 때 걸리는 시간은 총 n+1번 패킷을 보낸거니까 (n+1)*tT
재전송 n번 할 확률은 n번을 잘못 보내고 마지막 한 번 제대로 보낸거라 p^n(1-p)

이를 계산하면,


E[N] = 평균 재전송 횟수
E[tv] = E[tv | N = E[N]] : ERROR가 있을 때 평균 시간

따라서 최대 대역폭은 이의 역수!

2. Go-Back-N
가정은 Stop-and-Wait와 동일

충분한 window 크기를 가진 Saturated network
error가 없을 때 프레임 하나를 t_I에 하나씩 보낼 수 있음
패킷에 에러가 있을 때 걸리는 평균 시간 = n번 오류 났을 때 걸리는 시간 * n번 오류날 확률 의 총 합



3. Selective Repeat
가정은 앞에서 좀 더 추가

receiver는 reordering buffer를 무한 개 가지고 있다
𝑝 = P{receiving a frame in error} // 받은 프레임에 에러가 있을 확률
P{successful transmission in first attempt} = 1 − 𝑝 //첫번째 시도에 성공할 확률
P{successful transmission in 𝑛-th attempt} = 𝑝𝑛−1(1 − 𝑝) //n번째 시도에 성공할 확률
k를 전송 성공을 위해 필요한 전송 수라고 하면,

𝑡𝐼는 한 프레임의 transmission time임을 기억해봐!
이 말은 즉, 우리는 단위 시간 당 최대 1/𝑡𝐼 프레임의 전송이 가능하다는 뜻


그래서 이제 (단위시간에 보낼 수 있는 프레임 수의 평균) / (프레임 하나를 보내는데 필요한 재 전송 횟수의 평균) 을 하면 최대 대역폭을 구할 수 있다. 그니까 원래 에러 없을 때의 대역폭 나누기 에러 있으니까 재전송하는데 대역폭을 써야해서 그걸로 나눠주는 것..

Compare ARQ Schemes

알파를 저렇게 가정을 해보자. 


앞에서 구한 람다 max에 t_I를 곱한 값을 구해보면 다음과 같이 나타난다.

 

- α가 1에 가까우면(충분히 작으면) 저 세 가지 방법이 비슷해진다.

  그 말은 즉, t_I 즉 전송 시간이 propagation delay를 결정한다는 뜻. (왜지?

 

- α가 매우 크고 p는 매우 작을 때 즉 αp << 1 일 때 (t_out이 크고 오류 있을 확률이 작을 때)

    - Stop-and-wait 방식은 아주 좋지 않고

    - Go-Back-N 이나 Selective Repeat이 비슷한 성과를 낸다.

 

-  αp ~ 1 이거나  αp > 1 이면 Selective Repeat이 Go-Back-N보다 훨씬 낫다.

 

- 대개로  Selective Repeat이 성능이 좋지만 제일 복잡하다.
