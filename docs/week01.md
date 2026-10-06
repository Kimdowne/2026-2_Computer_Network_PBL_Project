
# 1주차 실습 가이드

## 실습 환경 접속과 첫 패킷 관찰

프로젝트: **「소규모 사내망 설계 및 Wireshark 기반 통신 검증·장애 진단」**

이번 주 실습망은 다음 하나의 LAN만 사용한다.

```text
개인 Windows PC
   │
   │ SSH / SCP
   │ <SERVER_MANAGEMENT_IP>:58888
   │ 실제 관리망
   ▼
공용 Ubuntu 서버
   │
   ├─ pbl-c1
   │   eth0 = 192.168.10.10/26
   │
   ├──── pbl-br-staff ────┤
   │
   └─ pbl-web
       eth0 = 192.168.10.20/26
       TCP 8080
```

**중요:** `192.168.10.10`과 `192.168.10.20`은 공용 서버 내부의 가상 실습망 주소다. 개인 Windows PC의 웹 브라우저에서 `http://192.168.10.20:8080/`을 열어 접속하는 실습이 아니다. HTTP 요청은 반드시 **가상 클라이언트 `pbl-c1` 내부에서** 발생시킨다.

또한 관리용 SSH 접속은 외부에서 **TCP 58888 포트**로 포워딩되어 있다고 가정한다. 따라서 SSH 명령에는 `-p 58888`, SCP 명령에는 `-P 58888`을 사용한다.

---

# 1. 이번 주 목표와 완료 기준

## 1.1 학습 목표

이번 주가 끝났을 때 A·B·C·D 전원이 다음을 할 수 있어야 한다.

1. 클라이언트와 서버를 기초 수준에서 구분한다.
2. IP 주소가 “어느 장치/인터페이스인지”, 포트가 “어느 서비스인지” 구분하는 데 사용된다고 설명한다.
3. 개인 Windows PC에서 공용 Linux 서버에 SSH로 접속한다.
4. `pbl-c1`에서 `pbl-web`의 TCP 8080 서비스로 HTTP 요청을 발생시킨다.
5. **요청보다 먼저 패킷 캡처를 시작한다.**
6. 캡처를 종료하고 `.pcap` 파일을 Windows PC로 내려받는다.
7. Wireshark에서 자신이 만든 TCP/HTTP 통신을 찾는다.
8. 패킷에서 다음을 직접 확인한다.
   - 출발지 IP
   - 목적지 IP
   - 출발지 TCP 포트
   - 목적지 TCP 포트

## 1.2 개인 완료 기준

A·B·C·D 각각이 별도로 다음 과정을 수행해야 한다.

```text
본인 계정 SSH 로그인
→ 실험 시작 시각 기록
→ 환경 상태 확인
→ 본인 캡처 시작
→ pbl-c1에서 HTTP 요청
→ 본인 캡처 종료
→ 본인 Windows PC로 PCAP 다운로드
→ 본인 Wireshark에서 분석
→ 근거 패킷 번호 기록
```

한 사람이 요청을 보내고 다른 세 사람이 그 캡처를 보기만 하는 것은 개인 완료로 인정하지 않는다.

공용 `pbl-c1`을 사용하므로 **실제 요청·캡처는 A → B → C → D처럼 한 명씩 순서대로** 수행한다. PCAP을 내려받은 뒤의 Wireshark 분석은 네 명이 동시에 해도 된다.

---

# 2. 용어 설명

## 클라이언트

서비스를 **요청하는 쪽**이다.

이번 실습에서는:

```text
pbl-c1
192.168.10.10
```

이 클라이언트다.

`pbl-c1`이 웹 페이지를 요청한다.

## 서버

클라이언트의 요청을 기다리다가 서비스를 **제공하는 쪽**이다.

이번 실습에서는:

```text
pbl-web
192.168.10.20
TCP 8080
```

이 서버다.

서버라고 해서 반드시 별도의 물리 컴퓨터여야 하는 것은 아니다. 이번 프로젝트에서는 Linux 네트워크 네임스페이스를 이용해 공용 Linux 서버 한 대 안에 가상의 클라이언트와 서버를 만든다.

## IP 주소

네트워크 통신에서 어느 인터페이스와 통신하는지를 나타내는 주소라고 우선 이해한다.

이번 주에는 다음 두 주소만 기억하면 된다.

```text
pbl-c1  = 192.168.10.10
pbl-web = 192.168.10.20
```

`/26`의 자세한 계산은 이번 주 학습 목표가 아니다.

## 포트

한 장치 안에서 어느 네트워크 서비스를 이용할 것인지 구분하는 번호다.

이번 실습의 웹 서비스는:

```text
TCP 8080
```

을 사용한다.

또한 관리용 SSH 접속에는 외부에서 포워딩된:

```text
TCP 58888
```

을 사용한다.

여기서 두 포트는 용도가 완전히 다르다.

```text
58888 → 개인 PC에서 공용 서버로 관리용 SSH 접속
8080  → pbl-c1에서 pbl-web으로 HTTP 접속
```

클라이언트 측 TCP 포트는 운영체제가 임시로 선택하므로 `52344` 같은 숫자가 보일 수 있으며, 실제 값은 실행할 때마다 달라질 수 있다.

## 인터페이스

패킷이 들어오고 나가는 네트워크 연결 지점이다.

실제 PC에는 Ethernet 또는 Wi-Fi NIC가 있을 수 있다. 이번 실습에서는 Linux가 만든 가상 Ethernet 인터페이스를 사용한다.

예:

```text
pbl-c1 내부: eth0
pbl-web 내부: eth0

Linux 호스트:
pbl-c1-h
pbl-web-h
pbl-br-staff
```

Linux의 veth는 서로 연결된 가상 Ethernet 인터페이스 두 개를 한 쌍으로 만들 수 있으며, 한쪽을 네트워크 네임스페이스에 넣어 서로 다른 가상 네트워크 환경을 연결할 수 있다.

## 패킷

네트워크를 통해 전달되는 데이터의 한 단위라고 우선 이해한다.

웹 페이지를 한 번 요청해도 패킷 하나만 발생하는 것은 아니다. 예를 들어 TCP 연결을 만들기 위한 패킷과 HTTP 요청·응답 패킷 등이 여러 개 나타날 수 있다.

## 캡처

특정 인터페이스를 통과하는 패킷을 관찰하고 파일에 기록하는 것이다.

이번 주에는 Linux 서버의 `tcpdump`가 실습용 인터페이스에서 패킷을 수집한다.

`tcpdump -w 파일명`은 패킷을 저장 파일로 기록할 수 있고 `.pcap` 확장자가 일반적으로 사용된다.

---

# 3. 준비물과 서버 사양 확인 방법

## 3.1 개인 Windows PC 준비

필요한 프로그램:

- Windows PowerShell
- SSH 클라이언트
- SCP 클라이언트
- Wireshark

### 실행 위치 → 개인 Windows PowerShell

```powershell
Get-Command ssh
Get-Command scp
```

### 예상되는 관찰

설치되어 있다면 `ssh.exe`, `scp.exe`의 경로가 표시될 수 있다.

예:

```text
C:\Windows\System32\OpenSSH\ssh.exe
C:\Windows\System32\OpenSSH\scp.exe
```

### 의미

Windows PC에서 SSH 접속과 파일 다운로드가 가능하다는 뜻이다.

---

## 3.2 공용 서버 관리 접속 정보

아래 값은 실제 환경에 맞게 확인한다.

```text
<SERVER_MANAGEMENT_IP>
SSH 외부 포트: 58888
```

`<SERVER_MANAGEMENT_IP>`는 개인 PC에서 접속할 때 사용하는 공용 서버 또는 포트포워딩 장비의 관리 접속 주소다.

`192.168.10.10`, `192.168.10.20`을 여기에 입력하면 안 된다.

관리 접속 구조는 다음과 같이 이해한다.

```text
개인 Windows PC
    │
    │ TCP 58888
    ▼
<SERVER_MANAGEMENT_IP>:58888
    │
    │ 포트포워딩
    ▼
공용 Ubuntu 서버의 SSH 서비스
```

포트포워딩 장비에서 외부 `58888`을 서버의 SSH 포트로 전달하는 구조라면, 서버 자체의 `sshd`가 반드시 58888에서 직접 LISTEN할 필요는 없다.

Linux 서버에서는 현재 관리 주소와 경로를 조회한다.

### 실행 위치 → Linux 호스트

```bash
ip -br addr
ip route
```

### 의미

관리망 상태를 확인하기 위한 조회다.

**이 단계에서는 관리 NIC, IP 주소, 기본 경로를 수정하지 않는다.**

---

## 3.3 Ubuntu 버전과 기본 사양 확인

### 실행 위치 → Linux 호스트

```bash
cat /etc/os-release
uname -m
nproc
free -h
df -h /
```

실제 결과를 기록한다.

---

# 4. 셋업 담당자 A의 활동

> 이 단락은 A가 주도한다.  
> 셋업 완료 후 A도 B·C·D와 같은 개인 실습을 수행한다.

## 4.1 관리 연결 확인

```bash
whoami
echo "$SSH_CONNECTION"
ip -br addr
ip route
```

**관리용 실제 NIC, 실제 IP, 기본 경로, 시스템 전체 방화벽은 변경하지 않는다.**

---

## 4.2 필요한 프로그램 확인

```bash
command -v ip
command -v python3
command -v curl
command -v tcpdump
command -v sshd
```

필요한 경우:

```bash
sudo apt update
sudo apt install -y iproute2 tcpdump python3 curl openssh-server
```

---

## 4.3 SSH 서비스 확인

```bash
sudo systemctl status ssh --no-pager
```

중요한 점은 외부 접속 포트 `58888`과 서버 내부의 `sshd` 수신 포트가 반드시 같지는 않다는 것이다.

예를 들어 다음 구조도 가능하다.

```text
외부 58888
   ↓ 포트포워딩
서버 내부 TCP 22
```

따라서 서버에서:

```bash
sudo ss -lntp | grep ssh
```

를 실행했을 때 `:22`가 보이더라도 외부에서 `58888`로 접속하는 구성이 정상일 수 있다.

---

## 4.4 실습 계정

```text
A → pbl-a
B → pbl-b
C → pbl-c
D → pbl-d
```

계정과 `pbl` 그룹을 준비한다.

```bash
sudo groupadd -f pbl
```

필요한 계정을 생성하고:

```bash
sudo adduser --gecos "" pbl-a
sudo adduser --gecos "" pbl-b
sudo adduser --gecos "" pbl-c
sudo adduser --gecos "" pbl-d
```

그룹에 추가한다.

```bash
sudo usermod -aG pbl pbl-a
sudo usermod -aG pbl pbl-b
sudo usermod -aG pbl pbl-c
sudo usermod -aG pbl pbl-d
```

---

## 4.5 실습 디렉터리

```bash
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week1
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week1/captures
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week1/logs
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week1/www
```

페이지 작성:

```bash
sudo tee /srv/pbl/week1/www/index.html >/dev/null <<'EOF'
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>PBL Week 1</title>
</head>
<body>
  <h1>PBL Week 1 OK</h1>
  <p>This page is served by pbl-web.</p>
</body>
</html>
EOF
```

---

# 4.6 네임스페이스·브리지·veth 구성

사용하는 이름:

```text
pbl-br-staff
pbl-c1-h
pbl-c1-n
pbl-web-h
pbl-web-n
```

기존에 작성한 `/usr/local/sbin/pbl-w1-setup`을 사용한다.

현재 서버에는 Docker의 `FORWARD DROP` 정책이 존재할 수 있으므로, 실습망 통신이 차단되는 경우에는 해당 서버의 방화벽 정책과 실습 bridge의 상호작용을 별도로 점검한다.

다음과 같은 전체 방화벽 초기화는 사용하지 않는다.

```bash
sudo iptables -F
sudo iptables -P FORWARD ACCEPT
```

---

# 4.7 실습자용 제한 스크립트

실습자는 다음 명령만 사용한다.

```bash
sudo /usr/local/sbin/pbl-w1-lab check
sudo /usr/local/sbin/pbl-w1-lab capture-start
sudo /usr/local/sbin/pbl-w1-lab request
sudo /usr/local/sbin/pbl-w1-lab capture-stop
```

이 스크립트는 `pbl-a`, `pbl-b`, `pbl-c`, `pbl-d`만 허용하도록 구성되어 있으므로 셋업용 관리자 계정에서 실행하면:

```text
ERROR: 허용된 실습 계정이 아닙니다
```

가 나오는 것이 정상이다.

관리자 계정은 직접 `ip`, `bridge`, `ss`, `iptables` 등을 이용해 상태를 확인한다.

---

# 5. 실습 참여자 A·B·C·D의 활동

## 5.1 Windows에서 SSH 접속

A의 경우:

```powershell
ssh -p 58888 pbl-a@<SERVER_MANAGEMENT_IP>
```

B:

```powershell
ssh -p 58888 pbl-b@<SERVER_MANAGEMENT_IP>
```

C:

```powershell
ssh -p 58888 pbl-c@<SERVER_MANAGEMENT_IP>
```

D:

```powershell
ssh -p 58888 pbl-d@<SERVER_MANAGEMENT_IP>
```

### 중요

SSH에서 포트 지정은:

```text
-p 58888
```

처럼 **소문자 `-p`**를 사용한다.

다음처럼 작성하지 않는다.

```text
ssh pbl-a@<SERVER_MANAGEMENT_IP>:58888
```

일반적인 OpenSSH 명령행에서는 위 형식이 접속 포트 지정 방법이 아니다.

### 실패 시 확인

`Connection timed out`이면:

- `<SERVER_MANAGEMENT_IP>`가 맞는가?
- 포트포워딩 외부 포트가 58888이 맞는가?
- 라우터 또는 포트포워딩 설정이 정상인가?
- 서버가 켜져 있는가?

`Connection refused`이면:

- 포트포워딩 목적지가 맞는가?
- 서버 SSH 서비스가 실행 중인가?

서버에서:

```bash
sudo systemctl status ssh --no-pager
sudo ss -lntp | grep ssh
```

를 확인한다.

---

## 5.2 실험 시작 시각

```bash
date --iso-8601=seconds
```

기록한다.

---

## 5.3 준비 상태

```bash
sudo /usr/local/sbin/pbl-w1-lab check
```

본인이 `pbl-a`~`pbl-d` 중 하나로 로그인한 상태에서 수행한다.

---

## 5.4 예상 작성

```text
pbl-c1의 192.168.10.10에서
pbl-web의 192.168.10.20 TCP 8080으로 통신할 것으로 예상한다.
```

---

## 5.5 캡처 시작

```bash
sudo /usr/local/sbin/pbl-w1-lab capture-start
```

**반드시 HTTP 요청보다 먼저 실행한다.**

---

## 5.6 웹 요청

```bash
sudo /usr/local/sbin/pbl-w1-lab request
```

실제 요청의 핵심은:

```bash
ip netns exec pbl-c1 \
  curl --noproxy '*' -v --max-time 5 \
  http://192.168.10.20:8080/
```

이다.

여기서 TCP `8080`은 **실습 웹 서비스 포트**이며 SSH 관리용 `58888`과 관계없다.

---

## 5.7 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w1-lab capture-stop
```

출력되는 PCAP 파일명을 기록한다.

---

# 5.8 Windows PC로 PCAP 다운로드

SSH 세션에서:

```bash
exit
```

Windows PowerShell에서:

```powershell
New-Item -ItemType Directory -Force "$HOME\pbl-week1"
```

A의 예:

```powershell
scp -P 58888 pbl-a@<SERVER_MANAGEMENT_IP>:/srv/pbl/week1/captures/<실제-PCAP-파일명>.pcap "$HOME\pbl-week1\"
```

예:

```powershell
scp -P 58888 pbl-a@<SERVER_MANAGEMENT_IP>:/srv/pbl/week1/captures/pbl-a-20260922-110500.pcap "$HOME\pbl-week1\"
```

### 중요

SCP에서는 포트 지정 옵션이 SSH와 다르다.

```text
SSH: ssh -p 58888 ...
         ↑ 소문자

SCP: scp -P 58888 ...
         ↑ 대문자
```

이를 혼동하지 않는다.

---

# 5.9 Wireshark에서 열기

```text
File
→ Open
→ 본인의 .pcap 선택
```

---

# 5.10 통신 필터링

클라이언트:

```text
ip.addr == 192.168.10.10
```

웹 서비스:

```text
tcp.port == 8080
```

둘 다:

```text
ip.addr == 192.168.10.10 && tcp.port == 8080
```

### 주의

PCAP은 `pbl-c1-h`에서 수집하므로 관리용 SSH 트래픽인:

```text
TCP 58888
```

을 분석하는 것이 아니다.

이번 실습에서 분석 대상은:

```text
192.168.10.10 ↔ 192.168.10.20:8080
```

이다.

---

# 5.11 IP 주소 확인

클라이언트 → 서버:

```text
Source      192.168.10.10
Destination 192.168.10.20
```

서버 → 클라이언트는 반대다.

---

# 5.12 TCP 포트 확인

클라이언트 → 서버:

```text
Source Port: 임시 포트
Destination Port: 8080
```

서버 → 클라이언트:

```text
Source Port: 8080
Destination Port: 클라이언트 임시 포트
```

`58888`은 여기 나타나야 하는 값이 아니다. 그것은 Windows PC와 공용 서버 사이의 관리 SSH 접속에 사용되는 포트다.

---

# 5.13 TCP 연결 관찰

```text
tcp.port == 8080
```

으로 SYN, SYN/ACK, ACK 등을 관찰한다.

---

# 5.14 HTTP 요청

```text
http
```

또는:

```text
http.request
```

필터를 사용한다.

HTTP로 자동 해석되지 않으면:

```text
tcp.port == 8080
```

부터 확인한다.

---

# 6. 실패 증상별 점검

| 증상 | 먼저 확인 |
|---|---|
| SSH 접속 시간 초과 | `<SERVER_MANAGEMENT_IP>`, 외부 포트 58888, 포트포워딩 상태 |
| SSH 연결 거부 | 서버 SSH 서비스, 포워딩 목적지 |
| SSH 인증 실패 | 계정·암호·SSH 키 |
| SCP 연결 실패 | `-P 58888` 사용 여부 |
| `pbl-w1-lab` 사용자 오류 | `pbl-a`~`pbl-d` 계정으로 로그인했는지 |
| HTTP timeout | pbl-c1/pbl-web 주소, bridge, 방화벽 |
| HTTP connection refused | pbl-web TCP 8080 LISTEN 여부 |
| Wireshark에 패킷 없음 | 요청보다 먼저 캡처했는지 |
| TCP는 보이는데 HTTP가 안 보임 | `tcp.port == 8080`으로 먼저 확인 |

특히 다음 두 포트를 혼동하지 않는다.

```text
58888 = 개인 PC → 공용 서버 SSH 관리 접속

8080  = pbl-c1 → pbl-web HTTP 실습 통신
```

---

# 7. 개인 제출물과 이해도 질문

## 개인 제출물

```text
참여자:
실험 날짜:
SSH 로그인 계정:
관리 접속 주소:
관리 SSH 포트: 58888
실험 시작 시각:
캡처 시작 시각:
HTTP 요청 시각:
캡처 종료 시각:
PCAP 파일명:
```

Wireshark 근거:

```text
Frame 번호:
Source IP:
Destination IP:
Source TCP Port:
Destination TCP Port:
```

## 추가 이해도 질문

### 질문

`58888`과 `8080`은 각각 무엇에 사용되는가?

정답을 작성할 때 다음처럼 구분할 수 있어야 한다.

```text
58888:
개인 Windows PC에서 공용 서버에 SSH로 관리 접속할 때 사용하는
외부 포워딩 포트

8080:
가상 클라이언트 pbl-c1에서 가상 웹 서버 pbl-web으로
HTTP 요청을 보낼 때 사용하는 웹 서비스 포트
```

---

# 8. 종료·다음 주 보존

각 참여자는:

```bash
sudo /usr/local/sbin/pbl-w1-lab capture-stop
```

후:

```bash
exit
```

로 종료한다.

Windows에 PCAP이 내려받아졌는지 확인한다.

다음 주까지 유지:

```text
pbl-br-staff
pbl-c1
pbl-web

pbl-c1  = 192.168.10.10/26
pbl-web = 192.168.10.20/26

관리 SSH 외부 포트 = 58888
웹 실습 포트       = 8080
```

2주차에는 기존 단일 LAN에 `pbl-c2`를 추가한다.

---

# 1주차 핵심 구분

```text
[관리 통신]

개인 Windows PC
      │
      │ SSH/SCP TCP 58888
      ▼
공용 Linux 서버


[실습 통신]

pbl-c1
192.168.10.10
      │
      │ TCP 8080 / HTTP
      ▼
pbl-web
192.168.10.20
```

따라서 SSH 접속은:

```powershell
ssh -p 58888 pbl-a@<SERVER_MANAGEMENT_IP>
```

PCAP 다운로드는:

```powershell
scp -P 58888 pbl-a@<SERVER_MANAGEMENT_IP>:/srv/pbl/week1/captures/<파일명>.pcap "$HOME\pbl-week1\"
```

웹 요청은 가상 클라이언트 내부에서:

```bash
curl http://192.168.10.20:8080/
```

로 구분한다.

이 세 통신을 서로 섞어서 이해하지 않는 것이 중요하다.