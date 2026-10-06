# 2주차 상세 가이드 — ARP·ICMP와 네트워크 계층 이해

이번 주의 기준 구성은 다음과 같다.

```text
                         pbl-br-staff
                ┌────────────┼────────────┐
                │            │            │
             pbl-c1        pbl-c2       pbl-web
             eth0          eth0          eth0
        192.168.10.10  192.168.10.11  192.168.10.20
             /26            /26            /26
                                            │
                                      TCP 8080 HTTP

기본 게이트웨이: 없음
라우터: 없음
DNS/DHCP: 아직 사용하지 않음
```

Linux의 network namespace는 네트워크 장치·IP 스택·라우팅 테이블 등을 분리하고, veth 쌍을 이용하면 각 namespace를 Linux bridge에 연결할 수 있다. :chatgpt-content-reference{index="0"}

---

# 1. 목표·사전 지식·이번 주 범위

## 1.1 학습 목표

2주차 종료 시 A·B·C·D 모두 다음을 자기 캡처를 근거로 설명할 수 있어야 한다.

- MAC 주소와 IP 주소의 용도를 구분한다.
- Ethernet 프레임 안에 IPv4 패킷이 들어가는 구조를 찾는다.
- ARP Request와 ARP Reply를 찾는다.
- ICMP Echo Request와 Echo Reply를 연결한다.
- Ethernet 브로드캐스트와 유니캐스트를 구분한다.
- ARP 캐시에 대상 MAC 주소가 이미 있으면 매 ping 전에 ARP가 반복되지 않을 수 있음을 설명한다.
- ARP → ICMP와 Ethernet → IPv4 → TCP → HTTP 구조를 비교한다.
- 관찰한 Wireshark 필드를 OSI/TCP/IP 계층 개념과 연결한다.

Wireshark에서 실제로 사용할 대표 필드인 `eth.src`, `eth.dst`, `ip.src`, `ip.dst`, `icmp.ident`, `icmp.seq`가 현재 Wireshark 필드로 제공된다. :chatgpt-content-reference{index="1"}

## 1.2 사전 지식

이번 주에는 다음 정도만 알고 시작하면 된다.

- IP 주소: 논리적인 네트워크 주소
- MAC 주소: 같은 Ethernet 구간에서 프레임을 전달할 때 사용하는 링크 계층 주소
- `/26`: 이번 주에는 세 장치가 모두 같은 네트워크 `192.168.10.0/26`에 속한다는 사실까지만 사용한다.
- `ping`: ICMP Echo를 이용해 통신을 시험한다.
- `curl`: HTTP 요청을 발생시킨다.

`/26` 계산법 자체는 3주차에서 더 자세히 다룬다.

## 1.3 이번 주에 하지 않는 것

아직 다음 기능은 추가하지 않는다.

- 기본 게이트웨이
- 라우터
- 서버망
- DHCP
- DNS
- 정적 라우팅
- NAT

또한 개인 Windows PC나 Ubuntu 서버의 실제 관리 인터페이스 설정을 변경하지 않는다.

특히 다음과 같은 명령은 사용하지 않는다.

```text
호스트 관리 NIC의 주소 변경
호스트 기본 경로 삭제
호스트 전체 ARP/neighbor 캐시 초기화
iptables -F
nft flush ruleset
호스트 네트워크 서비스 전체 재시작
```

---

# 2. 셋업 담당자 A의 활동

A가 환경을 준비하지만, 이후 3절의 공통 실습은 A도 다른 세 명과 동일하게 수행한다.

## 2.1 필요한 명령 확인

**실행 위치: Linux 호스트**
**권한: 조회는 일반 사용자, 설치가 필요할 때만 관리자**

```bash
command -v ip
command -v bridge
command -v ping
command -v curl
command -v python3
command -v tcpdump
```

모두 경로가 표시되면 그대로 진행한다.

예:

```text
/usr/sbin/ip
/usr/bin/ping
/usr/bin/curl
```

없는 프로그램이 있다면 Ubuntu 패키지 설치가 가능한 환경에서만 필요한 항목을 설치한다.

```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping curl python3 tcpdump
```

패키지가 이미 있다면 다시 설치할 필요가 없다.

---

## 2.2 1주차 선행 환경 확인

**실행 위치: Linux 호스트**
**권한: 일부 `sudo` 필요**

### ① bridge 확인

```bash
ip -br link show pbl-br-staff
```

정상이라면 `pbl-br-staff`가 존재하고 대체로 `UP` 상태여야 한다.

추가 확인:

```bash
ip -d link show pbl-br-staff
```

출력에 `bridge`가 보여야 한다.

없는 경우에만 생성한다.

```bash
sudo ip link add pbl-br-staff type bridge
sudo ip link set pbl-br-staff up
```

**주의:** 실제 물리 NIC를 이 bridge에 연결하지 않는다.

---

### ② namespace 확인

```bash
ip netns list
```

2주차 시작 전 최소한 다음이 기대된다.

```text
pbl-c1
pbl-web
```

이미 `pbl-c2`가 있으면 삭제하지 말고 상태부터 확인한다.

---

### ③ pbl-c1 확인

```bash
sudo ip netns exec pbl-c1 ip -br link
sudo ip netns exec pbl-c1 ip -4 -br addr
sudo ip netns exec pbl-c1 ip route
```

기준 상태:

```text
eth0     UP     192.168.10.10/26
```

라우팅 테이블에는 기본적으로 다음과 같은 **직접 연결 경로만** 있어야 한다.

```text
192.168.10.0/26 dev eth0 ...
```

`default via ...`는 이번 주 정상 상태가 아니다.

---

### ④ pbl-web 확인

```bash
sudo ip netns exec pbl-web ip -br link
sudo ip netns exec pbl-web ip -4 -br addr
sudo ip netns exec pbl-web ip route
```

기준:

```text
eth0     UP     192.168.10.20/26
```

역시 기본 경로는 필요 없다.

---

## 2.3 pbl-c1 또는 pbl-web이 아예 없는 경우의 복구

정상으로 존재한다면 이 단계는 건너뛴다.

### pbl-c1이 없는 경우

**실행 위치: Linux 호스트**
**권한: 관리자**

```bash
sudo ip netns add pbl-c1

sudo ip link add pbl-c1-br type veth peer name pbl-c1-ns
sudo ip link set pbl-c1-ns netns pbl-c1

sudo ip link set pbl-c1-br master pbl-br-staff
sudo ip link set pbl-c1-br up

sudo ip netns exec pbl-c1 ip link set lo up
sudo ip netns exec pbl-c1 ip link set pbl-c1-ns name eth0
sudo ip netns exec pbl-c1 ip addr add 192.168.10.10/26 dev eth0
sudo ip netns exec pbl-c1 ip link set eth0 up
```

### pbl-web이 없는 경우

```bash
sudo ip netns add pbl-web

sudo ip link add pbl-web-br type veth peer name pbl-web-ns
sudo ip link set pbl-web-ns netns pbl-web

sudo ip link set pbl-web-br master pbl-br-staff
sudo ip link set pbl-web-br up

sudo ip netns exec pbl-web ip link set lo up
sudo ip netns exec pbl-web ip link set pbl-web-ns name eth0
sudo ip netns exec pbl-web ip addr add 192.168.10.20/26 dev eth0
sudo ip netns exec pbl-web ip link set eth0 up
```

veth는 항상 서로 연결된 쌍으로 생성되며 한쪽을 namespace로 이동시켜 이런 구성을 만들 수 있다. :chatgpt-content-reference{index="2"}

### namespace는 있지만 `eth0`가 없는 경우

잘못 남은 해당 **실습 namespace 하나만** 재생성하는 편이 초보자 실습에서는 안전하다.

단, `pbl-web`에서 HTTP 서버가 실행 중이면 먼저 2.7절의 PID 방식으로 그 프로세스만 종료한다.

호스트 전체 네트워크를 초기화하지 않는다.

---

## 2.4 pbl-c2 추가

### 먼저 존재 여부 확인

```bash
ip netns list | grep -w pbl-c2
```

아무것도 나오지 않으면 새로 생성한다.

**실행 위치: Linux 호스트**
**권한: 관리자**

```bash
sudo ip netns add pbl-c2

sudo ip link add pbl-c2-br type veth peer name pbl-c2-ns
sudo ip link set pbl-c2-ns netns pbl-c2

sudo ip link set pbl-c2-br master pbl-br-staff
sudo ip link set pbl-c2-br up

sudo ip netns exec pbl-c2 ip link set lo up
sudo ip netns exec pbl-c2 ip link set pbl-c2-ns name eth0
sudo ip netns exec pbl-c2 ip addr add 192.168.10.11/26 dev eth0
sudo ip netns exec pbl-c2 ip link set eth0 up
```

확인:

```bash
sudo ip netns exec pbl-c2 ip -br link
sudo ip netns exec pbl-c2 ip -4 -br addr
sudo ip netns exec pbl-c2 ip route
```

기준:

```text
eth0    UP    192.168.10.11/26
```

기본 경로는 없어야 한다.

---

## 2.5 bridge에 연결된 인터페이스 확인

**실행 위치: Linux 호스트**

```bash
bridge link show master pbl-br-staff
```

새로 만든 명칭을 사용했다면 대략 다음 host-side veth들이 보여야 한다.

```text
pbl-c1-br
pbl-c2-br
pbl-web-br
```

단, 1주차에서 다른 이름으로 정상 구성했다면 그 실제 이름을 기록하고 그대로 사용한다.

**중요:** 인터페이스 이름 자체보다 세 namespace가 동일한 `pbl-br-staff`에 연결되었다는 것이 중요하다.

---

## 2.6 각 장치의 실제 IP와 MAC 주소 기록

MAC 주소는 자동 생성될 수 있으므로 예시값을 보고 베끼지 않는다.

### pbl-c1

```bash
sudo ip netns exec pbl-c1 ip -4 -br addr show dev eth0
sudo ip netns exec pbl-c1 ip link show dev eth0
```

### pbl-c2

```bash
sudo ip netns exec pbl-c2 ip -4 -br addr show dev eth0
sudo ip netns exec pbl-c2 ip link show dev eth0
```

### pbl-web

```bash
sudo ip netns exec pbl-web ip -4 -br addr show dev eth0
sudo ip netns exec pbl-web ip link show dev eth0
```

`ip link` 출력의 다음 부분을 찾는다.

```text
link/ether 3a:12:...
```

이 값이 해당 `eth0`의 실제 MAC 주소다.

A는 다음 표를 채운다.

| 장치 | IP | 실제 MAC |
|---|---|---|
| pbl-c1 | 192.168.10.10/26 | 직접 기록 |
| pbl-c2 | 192.168.10.11/26 | 직접 기록 |
| pbl-web | 192.168.10.20/26 | 직접 기록 |

이 표는 이후 패킷 분석의 정답을 대신하는 것이 아니라 **실제 패킷 값과 대조하기 위한 환경 기록**이다.

---

## 2.7 HTTP 서비스 정상 여부 확인

**실행 위치: Linux 호스트 → pbl-web**
**권한: 관리자**

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

`LISTEN`이 나오면 기존 서비스를 그대로 사용한다.

없다면 실습용 페이지를 준비한다.

```bash
mkdir -p "$HOME/pbl-web-root"

cat > "$HOME/pbl-web-root/index.html" <<'EOF'
<!doctype html>
<html>
<head><title>PBL Week 2</title></head>
<body>
<h1>PBL Week 2 OK</h1>
<p>ARP, ICMP and HTTP observation lab.</p>
</body>
</html>
EOF
```

그 다음 HTTP 서버를 실행한다.

```bash
sudo sh -c "ip netns exec pbl-web \
python3 -m http.server 8080 \
--bind 192.168.10.20 \
--directory '$HOME/pbl-web-root' \
> '$HOME/pbl-web-root/server.log' 2>&1 \
& echo \$! > /run/pbl-web-8080.pid"
```

확인:

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

### 이 방식의 이유

namespace는 네트워크를 분리하지만 파일시스템 전체를 자동으로 독립시키지는 않는다. 따라서 실습용 HTML, 로그, PID 파일을 구분하여 관리한다.

서비스를 종료해야 할 때도 `killall python3` 같은 전체 종료 명령을 사용하지 않는다.

이 실습에서 시작한 서비스라면:

```bash
sudo kill "$(sudo cat /run/pbl-web-8080.pid)"
sudo rm -f /run/pbl-web-8080.pid
```

처럼 **기록한 PID 하나만** 종료한다.

---

## 2.8 정상 통신 확인

### c1 → c2

```bash
sudo ip netns exec pbl-c1 ping -c 2 192.168.10.11
```

### c1 → web

```bash
sudo ip netns exec pbl-c1 ping -c 2 192.168.10.20
```

### c2 → web

```bash
sudo ip netns exec pbl-c2 ping -c 2 192.168.10.20
```

### HTTP

```bash
sudo ip netns exec pbl-c1 \
curl --max-time 3 http://192.168.10.20:8080/
```

HTML 또는 `PBL Week 2 OK`가 나타나면 된다.

---

## 2.9 캡처 저장 위치 준비

**실행 위치: Linux 호스트**

```bash
mkdir -p "$HOME/pbl-captures/week2"
```

2주차 기본 캡처 위치는 **bridge가 아니라 요청을 발생시키는 클라이언트 namespace의 `eth0`**이다.

예:

```text
pbl-c1이 요청 발생 → pbl-c1의 eth0에서 캡처
pbl-c2가 요청 발생 → pbl-c2의 eth0에서 캡처
```

이렇게 하면 “어느 장치가 실제로 송·수신한 패킷인가?”가 명확하다.

---

# 3. 실습 참여자 A·B·C·D의 활동

모든 참가자가 직접 수행한다.

공유 서버에서 네트워크 상태를 변경하는 단계는 서로 겹치지 않게 **한 명씩 차례로 수행**한다. 내려받은 pcap 분석은 동시에 해도 된다.

공통 기준 실험은 다음으로 한다.

```text
송신: pbl-c1 192.168.10.10
대상: pbl-c2 192.168.10.11
캡처: pbl-c1의 eth0
```

각 참가자는 시작 전에 다음 세 가지를 작성한다.

1. 첫 ping 전에 어떤 패킷이 나타날 것이라고 예상하는가?
2. 두 번째 ping에서도 같은 ARP가 나타날 것이라고 예상하는가?
3. ICMP 패킷의 Ethernet 목적지 MAC은 무엇일 것이라고 예상하는가?

팀원과 답을 맞추기 전에 개인 기록을 먼저 남긴다.

---

## A. ARP 캐시 확인

**실행 위치: Linux 호스트 → pbl-c1**

```bash
sudo ip netns exec pbl-c1 ip neigh show dev eth0
```

특히 c2 항목을 확인한다.

```bash
sudo ip netns exec pbl-c1 \
ip neigh show 192.168.10.11 dev eth0
```

항목이 있을 수도 있고 없을 수도 있다.

예:

```text
192.168.10.11 lladdr xx:xx:xx:xx:xx:xx REACHABLE
```

또는 아무것도 출력되지 않을 수 있다.

`ip neigh`는 IP 주소와 링크 계층 주소 사이의 이웃 정보를 표시한다. Linux에는 `REACHABLE`, `STALE`, `DELAY`, `PROBE`, `FAILED` 등의 상태가 있다. :chatgpt-content-reference{index="3"}

**기록할 것**

```text
실험 시작 전 c2 neighbor 항목:
있음 / 없음
MAC:
상태:
```

상태 문자열을 곧바로 “정상/장애”로 단순화하지 않는다.

---

## B. 캡처 시작

가능하면 Windows에서 서버로 접속한 PowerShell/SSH 창을 두 개 사용한다.

- SSH 창 1: 캡처
- SSH 창 2: 캐시 확인과 ping

### SSH 창 1

**실행 위치: Linux 호스트 → pbl-c1**
**권한: 관리자**

파일명에서 `A` 부분은 본인 식별용 문자열로 변경한다.

```bash
sudo ip netns exec pbl-c1 \
tcpdump -i eth0 -nn -s 0 -U \
-w "$HOME/pbl-captures/week2/A_c1_arp_icmp.pcap"
```

다음과 비슷한 메시지가 나타나면 캡처 중이다.

```text
listening on eth0 ...
```

아직 `Ctrl+C`를 누르지 않는다.

`-nn`은 이름/서비스명 변환을 줄이고, `-s 0`은 패킷을 충분히 저장하기 위한 설정이다.

---

## C. 캐시를 비운 상태에서 첫 ping 실행

### ① pbl-c1의 **실습용 eth0에 있는 c2 항목만** 제거

**실행 위치: SSH 창 2 → Linux 호스트 → pbl-c1**

```bash
sudo ip netns exec pbl-c1 \
ip neigh flush to 192.168.10.11 dev eth0
```

이 명령은 대상 namespace의 지정 인터페이스에 한정하여 이웃 캐시를 제거한다. `ip neigh flush ... dev ...` 형식은 Linux의 공식 iproute2 인터페이스가 제공하는 기능이다. :chatgpt-content-reference{index="4"}

확인:

```bash
sudo ip netns exec pbl-c1 \
ip neigh show 192.168.10.11 dev eth0
```

정상적으로 제거되었다면 보통 아무 항목도 나타나지 않는다.

### 절대 대신 실행하지 말 것

```bash
sudo ip neigh flush ...
```

를 **호스트 관리 namespace에서 전체 대상으로 실행하지 않는다.**

그리고 다음도 사용하지 않는다.

```text
iptables -F
nft flush ruleset
systemctl restart networking
```

---

### ② 첫 ping

```bash
sudo ip netns exec pbl-c1 \
ping -c 1 -W 1 192.168.10.11
```

성공 시 대략:

```text
64 bytes from 192.168.10.11: ...
```

### ③ 바로 캐시 확인

```bash
sudo ip netns exec pbl-c1 \
ip neigh show 192.168.10.11 dev eth0
```

실제 출력의 MAC과 상태를 기록한다.

### 예상 관찰

캐시가 없던 상태라면 일반적으로 다음 순서가 기대된다.

```text
MAC 주소 확인 과정
↓
ICMP Echo Request
↓
ICMP Echo Reply
```

이 단계에서는 아직 “어떤 MAC이 broadcast였는가?”를 답안으로 확정하지 말고 캡처에서 직접 확인한다.

---

## D. 동일 대상으로 두 번째 ping 실행

캐시를 다시 지우지 않는다.

첫 ping 직후 실행한다.

```bash
sudo ip netns exec pbl-c1 \
ping -c 1 -W 1 192.168.10.11
```

다시 확인:

```bash
sudo ip netns exec pbl-c1 \
ip neigh show 192.168.10.11 dev eth0
```

그 다음 SSH 창 1로 돌아가:

```text
Ctrl+C
```

로 tcpdump만 종료한다.

파일 확인:

```bash
ls -lh "$HOME/pbl-captures/week2/A_c1_arp_icmp.pcap"
```

필요하면 소유권을 현재 사용자에게 돌린다.

```bash
sudo chown "$USER:$USER" \
"$HOME/pbl-captures/week2/A_c1_arp_icmp.pcap"
```

---

## E. 첫 번째와 두 번째 ping의 ARP·ICMP 차이 분석

### Windows PC로 파일 내려받기

**실행 위치: 개인 Windows PowerShell**

```powershell
scp <SSH계정>@<관리서버주소>:~/pbl-captures/week2/A_c1_arp_icmp.pcap .
```

`<SSH계정>`과 `<관리서버주소>`를 그대로 입력하는 것이 아니다.

예를 들어 SSH 접속 명령이

```powershell
ssh student@10.20.30.40
```

였다면 각각:

```text
SSH계정 = student
관리서버주소 = 10.20.30.40
```

이다.

SSH로 접속한 서버 측 주소를 확인할 필요가 있다면 Linux 호스트에서:

```bash
echo "$SSH_CONNECTION"
```

을 확인할 수 있다. 단, 실제 관리 주소가 무엇인지 모르겠다면 기존에 정상 접속할 때 사용하던 주소를 기준으로 한다.

---

## 3.E.1 Wireshark에서 ARP와 ICMP만 보기

pcap 파일을 연 뒤 Display Filter에 입력한다.

```text
arp || icmp
```

### ARP Request만

```text
arp.opcode == 1
```

### ARP Reply만

```text
arp.opcode == 2
```

Wireshark는 ARP의 `arp.opcode`, 송신/목적 MAC 및 IPv4 관련 필드를 별도로 제공한다. :chatgpt-content-reference{index="5"}

### Echo Request

```text
icmp.type == 8
```

### Echo Reply

```text
icmp.type == 0
```

### Broadcast Ethernet 프레임 확인

```text
eth.dst == ff:ff:ff:ff:ff:ff
```

---

## 3.E.2 ARP 패킷에서 펼칠 항목

ARP Request 하나를 선택한다.

Packet Details 창에서:

```text
Frame
Ethernet II
Address Resolution Protocol
```

을 각각 펼친다.

### Ethernet II에서 확인

- Source
- Destination
- Type

특히 `Destination`을 기록한다.

그리고 `Type`이 무엇으로 표시되는지 확인한다.

ARP가 **TCP 또는 UDP 안에 있는지 찾지 않는다.**

ARP는 이 실습의 Ethernet에서 직접 전달되며 IPv4/TCP/UDP 패킷의 일부가 아니다.

---

### Address Resolution Protocol에서 확인

- Hardware type
- Protocol type
- Hardware size
- Protocol size
- Opcode
- Sender MAC address
- Sender IP address
- Target MAC address
- Target IP address

실제 값은 보고서에 그대로 옮긴다.

ARP Reply에서도 같은 항목을 확인한 뒤 Request와 비교한다.

---

## 3.E.3 ICMP에서 확인할 항목

Echo Request 하나를 선택한다.

다음을 차례로 펼친다.

```text
Ethernet II
Internet Protocol Version 4
Internet Control Message Protocol
```

### Ethernet II

- Source MAC
- Destination MAC
- Type

### IPv4

- Source Address
- Destination Address
- Time to Live
- Protocol

Wireshark의 IPv4 필드에는 `ip.src`, `ip.dst`, `ip.ttl`, `ip.proto` 등이 제공된다. :chatgpt-content-reference{index="6"}

### ICMP

- Type
- Code
- Identifier
- Sequence Number

Echo Request는 보통 Type 8, Echo Reply는 Type 0으로 보인다.

---

## 3.E.4 ICMP 요청과 응답 연결

요청과 응답을 단순히 “바로 다음 패킷이라서” 연결하지 않는다.

다음을 비교한다.

```text
Request:
Source IP
Destination IP
Identifier
Sequence Number

Reply:
Source IP
Destination IP
Identifier
Sequence Number
```

같은 Echo 교환에서는 요청과 응답의:

- Identifier
- Sequence Number

를 대조하고, IP 방향이 반대로 되어 있는지 확인한다.

Wireshark의 관련 필터 필드는:

```text
icmp.ident
icmp.seq
```

이다. :chatgpt-content-reference{index="7"}

예를 들어 실제 캡처에서 Identifier가 `0x1234`였다면:

```text
icmp.ident == 0x1234
```

처럼 좁힐 수도 있다.

**주의:** 이번 실습처럼 `ping -c 1` 명령을 두 번 별도로 실행하면 첫 번째 ping 프로세스와 두 번째 ping 프로세스의 Identifier가 같을 필요는 없다.

따라서:

> “첫 번째 ping과 두 번째 ping이므로 Identifier가 같아야 한다.”

라고 판단하지 않는다.

Identifier와 Sequence는 각각의 **Request ↔ Reply 쌍을 연결**할 때 사용한다.

---

## 3.E.5 첫 ping과 두 번째 ping 비교표

직접 채운다.

| 항목 | 첫 ping | 두 번째 ping |
|---|---|---|
| 실행 직전 c2 캐시 | | |
| ARP Request 관찰 | | |
| ARP Reply 관찰 | | |
| Echo Request 관찰 | | |
| Echo Reply 관찰 | | |
| Echo Request 목적지 MAC | | |
| Echo Request 출발지/목적지 IP | | |
| ICMP Identifier | | |
| ICMP Sequence | | |

두 번째 ping에서도 ARP가 관찰되었다면 무조건 오답으로 처리하지 않는다. 실험 간격, neighbor 상태 변화, 인터페이스 이벤트 등이 있었는지 먼저 점검한다.

---

# F. 웹 요청 패킷과 ICMP 패킷의 계층 비교

이번에는 pbl-c1 → pbl-web HTTP 요청을 캡처한다.

## F-1. 캡처 시작

**실행 위치: Linux 호스트 → pbl-c1**

```bash
sudo ip netns exec pbl-c1 \
tcpdump -i eth0 -nn -s 0 -U \
-w "$HOME/pbl-captures/week2/A_c1_http.pcap"
```

---

## F-2. 별도 SSH 창에서 HTTP 요청

```bash
sudo ip netns exec pbl-c1 \
curl --max-time 3 http://192.168.10.20:8080/
```

응답을 확인한 다음 캡처 창에서 `Ctrl+C`한다.

---

## F-3. Wireshark 필터

```text
tcp.port == 8080 || http
```

ARP까지 함께 살펴보려면:

```text
arp || tcp.port == 8080 || http
```

Wireshark는 HTTP를 별도의 `http` 프로토콜로 분석할 수 있다. :chatgpt-content-reference{index="8"}

HTTP로 자동 해석되지 않는다면 먼저:

```text
tcp.port == 8080
```

으로 패킷이 존재하는지 확인한다.

그 후 필요한 경우 Wireshark에서:

```text
Analyze
→ Decode As...
```

를 열어 TCP 8080 트래픽을 HTTP로 해석하도록 지정한다.

---

## F-4. HTTP 패킷에서 펼칠 계층

HTTP 요청 패킷 하나를 선택한다.

Packet Details에서:

```text
Ethernet II
Internet Protocol Version 4
Transmission Control Protocol
Hypertext Transfer Protocol
```

을 차례로 펼친다.

기록할 항목:

### Ethernet

```text
Source MAC
Destination MAC
EtherType
```

### IPv4

```text
Source IP
Destination IP
Protocol
```

### TCP

```text
Source Port
Destination Port
```

목적지 포트가 이번 환경에서는 `8080`인지 확인한다.

### HTTP

가능하면 다음을 확인한다.

```text
Request Method
Request URI
Host
```

---

## F-5. ICMP와 비교

각자 다음 표를 완성한다.

| 비교 | ICMP Echo | HTTP 요청 |
|---|---|---|
| Ethernet 존재 | | |
| IPv4 존재 | | |
| TCP 존재 | | |
| ICMP 존재 | | |
| HTTP 존재 | | |
| 포트 번호 사용 여부 | | |
| 출발지/목적지 MAC 확인 가능 | | |
| 출발지/목적지 IP 확인 가능 | | |

또한 ARP 패킷 하나와 비교하여:

> ARP 패킷에도 IPv4 헤더가 있는가?

를 Packet Details의 실제 계층 목록으로 확인한다.

---

## F-6. OSI·TCP/IP 계층과 연결

이번 주에는 지나치게 엄격하게 계층 번호를 외우는 것보다 **실제 필드가 어디에 있는지**를 우선한다.

| 관찰 대상 | 이번 실습에서 연결할 개념 |
|---|---|
| Ethernet Source/Destination MAC | 링크/네트워크 접근 계층 |
| ARP | 같은 Ethernet 구간의 IP↔MAC 해석 |
| IPv4 Source/Destination IP | 인터넷/네트워크 계층 |
| ICMP Echo | IP 계층의 제어·진단 성격 |
| TCP Source/Destination Port | 전송 계층 |
| HTTP | 응용 계층 |

ARP는 교재에 따라 OSI 계층을 표현하는 방식이 조금 다를 수 있으므로 “무조건 OSI 몇 계층”만 외우기보다는 이번 캡처에서 확인되는 사실을 우선한다.

즉:

```text
Ethernet 프레임 안에 ARP가 직접 있음
```

과

```text
Ethernet
 └ IPv4
    └ ICMP
```

및

```text
Ethernet
 └ IPv4
    └ TCP
       └ HTTP
```

의 차이를 설명하면 된다.

---

# G. 각자 다른 장치를 대상으로 재실행

네 명 모두 한 번씩 직접:

1. 캐시 확인
2. 대상 neighbor 항목 제거
3. 캡처 시작
4. 첫 ping
5. 두 번째 ping
6. 캡처 종료
7. Wireshark 분석

을 수행한다.

현재 endpoint가 세 개뿐이므로 네 명에게 네 개의 서로 다른 **목적지 장치**를 줄 수는 없다. 따라서 네 명이 서로 다른 **송신→수신 조합**을 사용한다.

예시 배정:

| 참가자 | 송신 | 대상 | 캡처 위치 |
|---|---|---|---|
| A | pbl-c2 | pbl-web | pbl-c2 eth0 |
| B | pbl-web | pbl-c1 | pbl-web eth0 |
| C | pbl-c2 | pbl-c1 | pbl-c2 eth0 |
| D | pbl-c1 | pbl-web | pbl-c1 eth0 |

예를 들어 A의 경우:

```bash
sudo ip netns exec pbl-c2 \
ip neigh flush to 192.168.10.20 dev eth0
```

캡처:

```bash
sudo ip netns exec pbl-c2 \
tcpdump -i eth0 -nn -s 0 -U \
-w "$HOME/pbl-captures/week2/A_individual.pcap"
```

다른 창:

```bash
sudo ip netns exec pbl-c2 ping -c 1 -W 1 192.168.10.20
sudo ip netns exec pbl-c2 ping -c 1 -W 1 192.168.10.20
```

나머지 참가자도 자신의 source namespace와 destination IP만 정확히 바꾼다.

**중요:** 다른 사람이 실행한 pcap을 복사해서 제출하면 개인 실습으로 인정하지 않는다.

---

# 4. 팀 비교 토의

팀 토의는 **개인 분석을 먼저 작성한 뒤** 시작한다.

각 참가자는 자기 캡처에서 최소한 다음 패킷 번호를 제시한다.

```text
ARP Request: Frame #
ARP Reply: Frame #
첫 Echo Request: Frame #
첫 Echo Reply: Frame #
두 번째 Echo Request: Frame #
두 번째 Echo Reply: Frame #
HTTP Request 또는 TCP/8080 패킷: Frame #
```

그 후 다음 질문을 비교한다.

### 질문 1

첫 번째 ping과 두 번째 ping에서 관찰한 ARP 수는 같았는가?

다르면 무엇이 달랐는가?

### 질문 2

ARP Request와 ICMP Echo Request의 목적지 MAC은 같았는가?

패킷 번호를 근거로 답한다.

### 질문 3

ARP Request에는:

```text
Ethernet II
IPv4
ICMP
```

가 모두 있는가?

실제 Packet Details를 근거로 설명한다.

### 질문 4

ICMP에서 요청과 응답을 연결할 때 어떤 필드를 사용했는가?

단순히 패킷 순서만 사용했는지 확인한다.

### 질문 5

HTTP 패킷에는 ICMP와 달리 어떤 계층이 추가되어 있는가?

### 질문 6

MAC 주소와 IP 주소 중 어느 것이 Ethernet 헤더에 들어 있으며, 어느 것이 IPv4 헤더에 들어 있는가?

### 질문 7

각 사람의 MAC 주소가 다른데도 통신 흐름이 동일한 구조였는가?

MAC 값을 외우는 것이 아니라 구조를 비교한다.

### 토론 기록 방법

다음 세 범주를 분리한다.

```text
관찰 사실:
Frame 3의 Ethernet Destination은 xx:xx:...

추정:
캐시 항목을 이용했을 가능성이 있다.

추가 확인:
해당 시점의 ip neigh 상태를 확인해야 한다.
```

“패킷이 안 보이므로 반드시 유실됐다”처럼 관찰 범위를 넘어서는 단정은 하지 않는다.

---

# 5. 오류 해결

## 5.1 `Cannot open network namespace`

예:

```text
Cannot open network namespace "pbl-c2"
```

확인:

```bash
ip netns list
```

namespace 자체가 없으면 2절의 생성 과정으로 돌아간다.

---

## 5.2 `Device "eth0" does not exist`

확인:

```bash
sudo ip netns exec pbl-c2 ip link
```

`eth0` 대신 예상하지 않은 이름이 있다면 해당 구성 과정을 다시 확인한다.

namespace가 불완전하게 만들어졌다면 그 **실습 namespace만** 재구성하고 호스트 네트워크를 초기화하지 않는다.

---

## 5.3 ping에서 `Network is unreachable`

다음 순서로 본다.

```bash
sudo ip netns exec pbl-c1 ip -4 -br addr
sudo ip netns exec pbl-c1 ip route
```

확인할 것:

1. `eth0`가 UP인가?
2. `192.168.10.10/26`이 있는가?
3. `192.168.10.0/26 dev eth0` 직접 연결 경로가 있는가?

이번 주에는 기본 게이트웨이를 추가해서 해결하면 안 된다.

---

## 5.4 ping timeout

먼저 상대 장치를 확인한다.

```bash
sudo ip netns exec pbl-c2 ip -br link
sudo ip netns exec pbl-c2 ip -4 -br addr
```

bridge:

```bash
bridge link show master pbl-br-staff
```

neighbor 상태:

```bash
sudo ip netns exec pbl-c1 ip neigh show dev eth0
```

`INCOMPLETE` 또는 `FAILED`가 있다고 해서 즉시 “서버가 다운됨”으로 결론 내리지 않는다.

다음과 같은 가능성을 확인한다.

- 상대 인터페이스 DOWN
- bridge 연결 누락
- 상대 IP 오류
- 잘못된 캡처 위치
- veth 구성 오류

---

## 5.5 첫 ping인데 ARP가 안 보임

순서대로 확인한다.

### ① 캐시를 실제로 지웠는가?

```bash
sudo ip netns exec pbl-c1 \
ip neigh show 192.168.10.11 dev eth0
```

### ② 올바른 namespace에서 지웠는가?

잘못:

```text
Linux 호스트의 neighbor 캐시 제거
```

정상:

```text
pbl-c1 내부 eth0의 192.168.10.11 항목 제거
```

### ③ 캡처를 ping 전에 시작했는가?

ping부터 실행하고 tcpdump를 시작했다면 최초 ARP를 놓칠 수 있다.

### ④ 캡처 인터페이스가 맞는가?

기준:

```text
pbl-c1 → pbl-c2 통신
캡처 → pbl-c1 eth0
```

---

## 5.6 두 번째 ping에도 ARP가 보임

즉시 실습 실패라고 단정하지 않는다.

먼저:

```bash
sudo ip netns exec pbl-c1 \
ip neigh show 192.168.10.11 dev eth0
```

를 확인한다.

다음 상황이 있었는지도 본다.

- 첫 ping과 두 번째 ping 사이가 길었음
- neighbor 상태가 변경됨
- 인터페이스를 내렸다 올림
- 다른 실습자가 캐시를 제거함
- namespace를 재구성함

Linux의 neighbor 항목은 `REACHABLE`, `STALE`, `DELAY`, `PROBE` 등 여러 상태를 가질 수 있다. 따라서 “ARP는 정확히 한 번만 발생한다”가 규칙은 아니다. :chatgpt-content-reference{index="9"}

---

## 5.7 HTTP 접속 실패

먼저 ping:

```bash
sudo ip netns exec pbl-c1 ping -c 1 192.168.10.20
```

ping이 성공해도 HTTP 서비스까지 정상이라는 뜻은 아니다.

pbl-web에서:

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

서비스가 없다면 2.7절 방식으로 다시 실행한다.

로그:

```bash
cat "$HOME/pbl-web-root/server.log"
```

---

## 5.8 Wireshark에서 HTTP로 표시되지 않음

다음 필터부터 확인한다.

```text
tcp.port == 8080
```

TCP 패킷은 있는데 HTTP가 없다면:

```text
Analyze → Decode As...
```

에서 TCP 8080을 HTTP로 분석해 본다.

HTTP 표시가 안 된다는 이유만으로 “HTTP 요청이 전송되지 않았다”고 단정하지 않는다.

---

## 5.9 캡처 파일이 열리지 않음

파일 확인:

```bash
ls -lh "$HOME/pbl-captures/week2"
```

소유권 확인:

```bash
ls -l "$HOME/pbl-captures/week2"
```

필요하면 해당 파일 하나만:

```bash
sudo chown "$USER:$USER" \
"$HOME/pbl-captures/week2/파일명.pcap"
```

확장자를 임의로 `.pcapng`로 바꾸지 않는다.

이번 명령으로 작성한 파일은 `.pcap`으로 유지한다.

---

## 5.10 Wireshark에서 checksum 관련 경고가 보임

가상 인터페이스·checksum offloading 등의 영향으로 캡처 시점에 checksum 관련 표시가 생길 수 있다.

이번 주에는 해당 표시만 보고:

```text
패킷 손상
공격
통신 실패
```

라고 판단하지 않는다.

실제 ping/curl 결과와 요청·응답 흐름을 함께 본다.

---

# 6. 개인 제출물·평가

A·B·C·D 모두 **개인별로** 제출한다.

## 6.1 캡처 파일

최소:

```text
<이름>_arp_icmp.pcap
<이름>_http.pcap
<이름>_individual.pcap
```

파일명 규칙은 팀에서 통일해도 된다.

---

## 6.2 개인 실험 기록

### ① 실험 조건

```text
송신 namespace:
송신 IP:
송신 MAC:

대상 namespace:
대상 IP:
대상 MAC:

캡처 namespace:
캡처 인터페이스:
실행 시각:
```

### ② 사용한 Wireshark 필터

최소 다음 종류를 기록한다.

```text
ARP+ICMP 확인 필터:
ARP Request 필터:
ARP Reply 필터:
Echo Request 필터:
Echo Reply 필터:
HTTP/TCP 8080 필터:
Broadcast 확인 필터:
```

---

## 6.3 근거 패킷 표

| 근거 | Frame 번호 | Src MAC | Dst MAC | Src IP | Dst IP | 추가 필드 |
|---|---:|---|---|---|---|---|
| ARP Request | | | | | | Opcode |
| ARP Reply | | | | | | Opcode |
| 첫 Echo Request | | | | | | ID/Seq |
| 첫 Echo Reply | | | | | | ID/Seq |
| 두 번째 Echo Request | | | | | | ID/Seq |
| 두 번째 Echo Reply | | | | | | ID/Seq |
| HTTP 요청 | | | | | | TCP ports |

ARP 행에서 IPv4 Source/Destination이 없다면 억지로 채우지 않는다.

ARP 안의 `Sender IP address`, `Target IP address`와 IPv4 헤더의 `Source/Destination Address`는 같은 종류의 필드가 아니므로 구분해 적는다.

---

## 6.4 캐시 전후 비교

다음을 기록한다.

```text
실험 시작 전:
캐시 제거 직후:
첫 ping 직후:
두 번째 ping 직후:
```

각 시점마다:

- IP
- MAC
- neighbor 상태

를 기록한다.

---

## 6.5 개인 서술 문제

각자 자기 문장으로 답한다.

### 문제 1

MAC 주소와 IP 주소는 각각 어떤 목적으로 사용되는가?

### 문제 2

ARP Request와 ARP Reply의 Ethernet 목적지 주소가 어떻게 달랐는가?

반드시 패킷 번호를 근거로 쓴다.

### 문제 3

첫 번째 ping과 두 번째 ping에서 ARP의 발생 여부가 왜 달라질 수 있는가?

### 문제 4

ICMP Echo Request와 Reply가 한 쌍임을 어떤 필드로 확인했는가?

### 문제 5

ARP, ICMP, HTTP 패킷에서 관찰되는 계층을 각각 적는다.

### 문제 6 — 필수

> **왜 모든 패킷의 목적지 MAC이 브로드캐스트가 아닌가?**

자신의 ARP 및 ICMP 근거 패킷 번호를 하나 이상 포함한다.

---

## 6.6 평가 기준

| 평가 요소 | 확인 내용 |
|---|---|
| 환경 이해 | IP와 실제 MAC을 정확히 구분했는가 |
| 캡처 절차 | 트래픽 전에 캡처를 시작했는가 |
| ARP 분석 | Request/Reply의 필드와 주소를 확인했는가 |
| ICMP 분석 | Type, Identifier, Sequence로 요청·응답을 연결했는가 |
| 캐시 비교 | 첫·두 번째 ping의 차이를 실제 결과로 설명했는가 |
| 계층 이해 | Ethernet/IP/ICMP/TCP/HTTP 구조를 구분했는가 |
| 근거 제시 | Frame 번호를 제시했는가 |
| 판단 품질 | 관찰과 추정을 구분했는가 |
| 개인 수행 | 본인이 직접 생성한 실험 결과인가 |

MAC 값을 외우거나 Wireshark 필터를 암기하는 것보다 “왜 이 필드를 확인하는가”를 설명하는 것을 우선한다.

---

# 7. 종료와 정상 상태 복구

2주차 실험은 IP 주소나 기본 경로를 일부러 변경하지 않으므로 실습 후 최종 상태는 다음과 같아야 한다.

```text
pbl-c1   192.168.10.10/26
pbl-c2   192.168.10.11/26
pbl-web  192.168.10.20/26

모두 pbl-br-staff 연결
기본 게이트웨이 없음
pbl-web TCP 8080 정상
```

## 7.1 남아 있는 tcpdump 확인

```bash
ps -ef | grep '[t]cpdump'
```

자신이 실수로 백그라운드 실행한 실습 tcpdump가 있다면 해당 PID를 확인하여 **그 프로세스만** 종료한다.

```bash
sudo kill <확인한-PID>
```

`killall tcpdump`처럼 다른 사람의 캡처까지 종료할 수 있는 명령은 사용하지 않는다.

---

## 7.2 최종 IP 상태

```bash
sudo ip netns exec pbl-c1 ip -4 -br addr
sudo ip netns exec pbl-c2 ip -4 -br addr
sudo ip netns exec pbl-web ip -4 -br addr
```

---

## 7.3 최종 라우팅 상태

```bash
sudo ip netns exec pbl-c1 ip route
sudo ip netns exec pbl-c2 ip route
sudo ip netns exec pbl-web ip route
```

각 namespace에 `default` 경로가 없어야 한다.

---

## 7.4 최종 통신 시험

```bash
sudo ip netns exec pbl-c1 ping -c 1 192.168.10.11
sudo ip netns exec pbl-c1 ping -c 1 192.168.10.20
sudo ip netns exec pbl-c2 ping -c 1 192.168.10.20
```

HTTP:

```bash
sudo ip netns exec pbl-c1 \
curl --max-time 3 http://192.168.10.20:8080/
```

모두 정상이면 2주차 환경을 유지한다.

**pbl-c2를 삭제하지 않는다.** 이후 주차에서도 사용하는 장치다.

ARP 캐시 역시 실습을 마쳤다는 이유로 강제로 비울 필요가 없다. 캐시는 정상적인 네트워크 상태의 일부다.

---

# 교사용 해설 — 실습자에게 분석 전 제공하지 않음

## 1. 캐시를 비운 뒤 첫 ping의 전형적인 흐름

pbl-c1이 `192.168.10.11`로 보내려 하지만 해당 MAC을 모르면 일반적으로:

```text
1. ARP Request
2. ARP Reply
3. ICMP Echo Request
4. ICMP Echo Reply
```

가 관찰된다.

단, 실제 Frame 번호는 다른 트래픽 유무에 따라 달라지므로 정답에 특정 번호를 미리 제시하면 안 된다.

---

## 2. ARP Request

전형적인 c1 → c2 ARP Request:

```text
Ethernet
Source MAC      = pbl-c1 MAC
Destination MAC = ff:ff:ff:ff:ff:ff
Type            = ARP

ARP
Opcode           = Request
Sender MAC       = pbl-c1 MAC
Sender IP        = 192.168.10.10
Target IP        = 192.168.10.11
Target MAC       = 미확정 상태
```

즉 c1은 아직 c2의 MAC을 모르므로 같은 Ethernet 구간의 장치들에게 질문한다.

---

## 3. ARP Reply

전형적으로:

```text
Ethernet
Source MAC      = pbl-c2 MAC
Destination MAC = pbl-c1 MAC

ARP
Opcode           = Reply
Sender MAC       = pbl-c2 MAC
Sender IP        = 192.168.10.11
Target MAC       = pbl-c1 MAC
Target IP        = 192.168.10.10
```

이 일반적인 요청/응답에서 ARP Reply는 요청자에게 유니캐스트된다.

ARP가 항상 broadcast라고 설명하면 안 된다.

---

## 4. 첫 ICMP Echo Request

ARP로 c2의 MAC을 알게 된 뒤에는 전형적으로:

```text
Ethernet
Source MAC      = pbl-c1 MAC
Destination MAC = pbl-c2 MAC

IPv4
Source IP       = 192.168.10.10
Destination IP  = 192.168.10.11

ICMP
Type            = 8
Code            = 0
Identifier      = 실제 캡처값
Sequence        = 실제 캡처값
```

가 된다.

Echo Reply에서는 Ethernet/IP 방향이 반대이고 ICMP Type은 0이 된다.

---

## 5. 두 번째 ping에 ARP가 보통 없는 이유

첫 ping 과정에서 c1은:

```text
192.168.10.11 → pbl-c2의 MAC
```

관계를 neighbor 캐시에 학습한다.

두 번째 ping이 곧바로 실행되면 해당 정보를 재사용할 수 있으므로 새로운 ARP Request 없이 ICMP를 전송할 수 있다.

다만 이것을:

> “두 번째 ping에는 절대로 ARP가 발생하지 않는다.”

라고 가르치면 안 된다.

neighbor 상태 만료, 재검증, 인터페이스 변화 등으로 추가 ARP가 발생할 수 있다. Linux의 neighbor 테이블은 여러 상태를 관리하며 `ip neigh`로 조회·제어할 수 있다. :chatgpt-content-reference{index="10"}

---

## 6. “왜 모든 패킷의 목적지 MAC이 브로드캐스트가 아닌가?”의 핵심 답

핵심은 다음과 같다.

ARP Request는 “192.168.10.11의 MAC 주소를 누가 가지고 있는가?”를 같은 Ethernet 구간에 물어봐야 하므로 일반적으로 broadcast 목적지 MAC을 사용한다.

하지만 ARP Reply를 받은 뒤 pbl-c1은 pbl-c2의 MAC 주소를 알게 된다. 그 뒤 ICMP Echo Request처럼 **특정 장치로 보내는 Ethernet 프레임은 그 장치의 MAC 주소를 목적지로 하는 유니캐스트**로 보낼 수 있다.

따라서:

```text
목적지 IP가 존재한다고
모든 Ethernet 목적지 MAC이 broadcast인 것은 아니다.
```

broadcast는 필요한 경우 여러 장치에게 전달하기 위한 것이며, 특정 상대의 MAC을 이미 아는 통신까지 계속 broadcast할 이유가 없다.

학생이 반드시 자기 캡처의 ARP Request와 Echo Request Frame 번호를 제시하도록 한다.

---

## 7. ARP·ICMP·HTTP 계층 정답 기준

### ARP

```text
Ethernet
└─ ARP
```

이번 Ethernet 실습에서 ARP가 TCP나 UDP 위에서 동작한다고 답하면 오답이다.

### ICMP Echo

```text
Ethernet
└─ IPv4
   └─ ICMP
```

### HTTP

```text
Ethernet
└─ IPv4
   └─ TCP
      └─ HTTP
```

Ethernet의 `eth.src`/`eth.dst`, IPv4의 `ip.src`/`ip.dst`, ICMP의 `icmp.ident`/`icmp.seq`는 Wireshark에서 직접 확인 가능한 필드다. :chatgpt-content-reference{index="11"}

---

## 8. ICMP 요청·응답 채점 시 주의점

학생이 다음과 같이 설명하면 적절하다.

> Echo Request와 Echo Reply의 출발지·목적지 IP가 서로 반대이며, Identifier와 Sequence Number를 비교하여 같은 요청에 대한 응답인지 확인했다.

반면:

> 바로 다음 패킷이니까 응답이다.

만으로 판단했다면 근거가 부족하다.

또한 별도의 `ping -c 1` 명령을 두 번 실행했으므로 **첫 번째 ping과 두 번째 ping 사이의 Identifier가 서로 달라도 정상**이다. 비교해야 하는 것은 각 Echo Request와 그에 대응하는 Echo Reply다.