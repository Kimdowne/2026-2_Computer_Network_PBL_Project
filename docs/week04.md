# 4주차 실습 가이드
## 서브넷 마스크 오류와 로컬·원격 판단

프로젝트: **「소규모 사내망 설계 및 Wireshark 기반 통신 검증·장애 진단」**

이번 주에는 새로운 라우터나 서버망을 추가하지 않는다.

지난 주까지 사용한 하나의 LAN을 그대로 사용하면서 **`pbl-c1`의 서브넷 마스크만 의도적으로 잘못 변경**한다.

```text
개인 Windows PC
       │
       │ SSH / SCP
       │ 실제 관리망
       ▼
공용 Ubuntu 서버
       │
       │      pbl-br-staff
       │   ┌──────┼─────────┐
       │   │      │         │
       ▼   ▼      ▼         ▼
     pbl-c1     pbl-c2    pbl-web
     .10/26     .11/26    .20/26
                           TCP 8080
```

정상 상태:

```text
pbl-c1   = 192.168.10.10/26
pbl-c2   = 192.168.10.11/26
pbl-web  = 192.168.10.20/26

bridge    = pbl-br-staff
gateway   = 없음
수동 경로 = 없음
```

오류 실험에서는 **pbl-c1만** 다음처럼 변경한다.

```text
정상: 192.168.10.10/26
             ↓
오류: 192.168.10.10/28
```

`pbl-c2`와 `pbl-web`은 계속 `/26`이다.

이번 주의 핵심 질문은 다음과 같다.

> 세 장치가 똑같은 Linux bridge에 연결되어 있는데도, pbl-c1의 서브넷 마스크만 잘못되면 왜 어떤 목적지는 통신되고 어떤 목적지는 통신되지 않을 수 있는가?

---

# 1. 이번 주 학습 목표

실습이 끝났을 때 A·B·C·D 전원이 다음을 설명하고 직접 확인할 수 있어야 한다.

1. 같은 Ethernet LAN에 연결되어 있다는 사실과 IP 계층에서 같은 서브넷으로 판단한다는 것은 같은 말이 아님을 설명한다.
2. `/26`과 `/28`에서 네트워크 범위를 계산한다.
3. `192.168.10.10/28`에서 `.11`은 로컬 목적지이고 `.20`은 로컬 목적지가 아님을 계산한다.
4. `ip addr`, `ip route`, `ip route get`의 역할을 구분한다.
5. 잘못된 마스크가 모든 통신을 똑같이 끊는 것은 아님을 관찰한다.
6. ARP·ICMP·TCP 패킷과 명령 실행 오류를 함께 근거로 사용한다.
7. **패킷이 전혀 발생하지 않은 상황도 조건을 충족한다면 중요한 증거가 될 수 있음**을 설명한다.
8. 설정을 `/26`으로 복구한 뒤 같은 시험을 반복해 정상화를 확인한다.

---

# 2. 먼저 계산한다

실습 명령을 실행하기 전에 네 명 모두 개인 기록에 계산 결과와 예상을 작성한다.

## 2.1 정상 상태: 192.168.10.10/26

`/26`의 서브넷 마스크는 다음과 같다.

```text
255.255.255.192
```

마지막 옥텟의 블록 크기:

```text
256 - 192 = 64
```

따라서 `192.168.10.10/26`이 포함되는 범위는:

```text
네트워크 주소     192.168.10.0
호스트 범위       192.168.10.1 ~ 192.168.10.62
브로드캐스트 주소 192.168.10.63
```

따라서:

```text
192.168.10.11 → 같은 /26
192.168.10.20 → 같은 /26
```

이다.

pbl-c1 입장에서는 둘 다 직접 연결된 네트워크에 속한다.

---

## 2.2 오류 상태: 192.168.10.10/28

`/28`의 서브넷 마스크:

```text
255.255.255.240
```

블록 크기:

```text
256 - 240 = 16
```

`.10`이 들어가는 `/28` 블록은:

```text
192.168.10.0 ~ 192.168.10.15
```

이다.

세부적으로:

```text
네트워크 주소     192.168.10.0
호스트 범위       192.168.10.1 ~ 192.168.10.14
브로드캐스트 주소 192.168.10.15
```

따라서:

```text
192.168.10.11 → 같은 /28
192.168.10.20 → 같은 /28이 아님
```

이다.

`.20`이 속하는 `/28`은 다음 블록이다.

```text
192.168.10.16/28

네트워크 주소     192.168.10.16
호스트 범위       192.168.10.17 ~ 192.168.10.30
브로드캐스트 주소 192.168.10.31
```

즉:

```text
pbl-c1 = 192.168.10.10/28

.11 → 로컬 목적지로 판단
.20 → 다른 네트워크의 목적지로 판단
```

하게 된다.

---

# 3. 중요한 개념: 물리적으로 가까운 것과 IP상 로컬인 것은 다르다

이번 환경에서는 세 장치가 모두 `pbl-br-staff`라는 같은 Linux bridge에 연결되어 있다.

Ethernet 수준에서 보면 같은 LAN이다.

하지만 pbl-c1의 IP 계층은 단순히 다음처럼 생각하지 않는다.

```text
"같은 bridge에 연결되어 있네.
그러면 바로 보내자."
```

먼저 자신의 IP 주소와 prefix, 라우팅 테이블을 보고 목적지까지 어떤 경로를 사용할지를 결정한다.

정상 `/26`에서는:

```text
192.168.10.0/26 dev eth0
```

이라는 직접 연결 경로 때문에 `.11`과 `.20` 모두 `eth0`으로 직접 보낼 수 있다.

오류 `/28`에서는 직접 연결 경로가:

```text
192.168.10.0/28 dev eth0
```

으로 줄어든다.

그래서:

```text
.11 → 192.168.10.0/28 안에 있음
.20 → 192.168.10.0/28 밖에 있음
```

이 된다.

그런데 이번 실습에는 기본 게이트웨이도 없고 `.20`으로 가는 수동 경로도 없다.

따라서 pbl-c1은 `.20`으로 보낼 적절한 경로를 찾지 못할 가능성이 높다.

이때 중요한 점은 다음이다.

> 경로가 없으면 반드시 ARP부터 실패하는 것이 아니다.

운영체제가 **패킷을 내보낼 경로 자체를 찾지 못하면 Ethernet 프레임을 만들기 전 단계에서 실패할 수 있다.**

따라서 오류 상태에서 `.20`을 대상으로 ARP·ICMP·TCP 패킷이 하나도 관찰되지 않는 결과도 가능하다.

`ip route get`은 커널이 특정 목적지에 대해 선택할 경로를 조회하며 실제 패킷을 보내지는 않는다. 따라서 이번 실습에서는 실제 시험 전에 커널의 경로 판단을 확인하는 도구로 사용한다.

---

# 4. 명령 실행 전 개인 예상 질문

아래 질문에는 **팀 토의 전에 각자 먼저 답한다.**

## 질문 1

`192.168.10.10/26`에서 `.11`과 `.20`은 각각 로컬인가 원격인가?

```text
.11:
판단 이유:

.20:
판단 이유:
```

## 질문 2

`192.168.10.10/28`에서 `.11`과 `.20`은 각각 로컬인가 원격인가?

```text
.11:
판단 이유:

.20:
판단 이유:
```

## 질문 3

오류 `/28` 상태에서도 `.11`과 ping 통신이 가능할 것으로 예상하는가?

```text
예상:
이유:
```

## 질문 4

오류 `/28` 상태에서 `.20`과 통신하려면 pbl-c1은 어떤 경로가 필요할까?

```text
예상:
```

이번 환경에는 그 경로가 존재하는가?

```text
예 / 아니오
```

## 질문 5

경로가 전혀 없다면 `.20`에 대한 ARP Request가 반드시 발생할까?

```text
예상:
이유:
```

## 질문 6

Wireshark에서 `.20` 관련 패킷이 하나도 보이지 않는다면 무엇을 추가로 확인해야 하는가?

최소 다음을 생각한다.

```text
캡처가 먼저 시작되었는가?
올바른 인터페이스를 캡처했는가?
올바른 PCAP을 열었는가?
필터 때문에 숨겨진 것은 아닌가?
실제로 요청 명령을 실행했는가?
ip route get 결과는 무엇인가?
명령 자체에서 어떤 오류가 나왔는가?
```

---

# 5. 셋업 담당자 A의 활동

이 절은 A가 환경 준비를 주도한다.

A도 준비를 끝낸 뒤에는 B·C·D와 동일한 개인 실험을 다시 수행해야 한다.

---

## 5.1 관리망과 실습망을 먼저 구분한다

### 실행 위치 → Linux 호스트
### 변경 없음

```bash
whoami
echo "$SSH_CONNECTION"
ip -br addr
ip route
```

이 명령은 **호스트 자체의 실제 관리망**을 확인하기 위한 것이다.

이번 주에 변경하는 것은:

```text
pbl-c1 네임스페이스 내부 eth0
```

뿐이다.

다음은 변경 대상이 아니다.

```text
Linux 호스트 실제 NIC
Linux 호스트 관리 IP
Linux 호스트 기본 게이트웨이
개인 Windows PC의 IP 설정
```

---

# 5.2 필요한 명령 확인

### 실행 위치 → Linux 호스트

```bash
command -v ip
command -v ping
command -v curl
command -v tcpdump
command -v python3
```

없는 도구가 있다면 A만 설치한다.

### 관리자 권한 필요

```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping curl tcpdump python3
```

---

# 5.3 네임스페이스 존재 확인

### 실행 위치 → Linux 호스트

```bash
sudo ip netns list
```

정상 상태에서는 최소 다음이 보여야 한다.

```text
pbl-c1
pbl-c2
pbl-web
```

순서는 다를 수 있다.

하나라도 없다면 바로 마스크 실험을 시작하지 않는다.

---

# 5.4 bridge 확인

### 실행 위치 → Linux 호스트

```bash
sudo ip link show pbl-br-staff
sudo ip -br link show master pbl-br-staff
```

확인할 항목:

```text
pbl-br-staff 존재
pbl-c1-h 연결
pbl-c2-h 연결
pbl-web-h 연결
```

인터페이스 이름을 이전 환경에서 다르게 사용했다면 실제 값을 확인한다.

이번 가이드에서는 다음 이름을 기준으로 한다.

```text
pbl-c1-h
pbl-c2-h
pbl-web-h
```

---

# 5.5 세 장치의 주소 확인

### 실행 위치 → Linux 호스트

```bash
sudo ip -n pbl-c1 -br addr
sudo ip -n pbl-c2 -br addr
sudo ip -n pbl-web -br addr
```

실험 시작 전 반드시 다음 상태여야 한다.

```text
pbl-c1
eth0  192.168.10.10/26

pbl-c2
eth0  192.168.10.11/26

pbl-web
eth0  192.168.10.20/26
```

`pbl-c1`에 `/28`이 남아 있다면 이전 실험이 정상적으로 복구되지 않은 것이다.

---

# 5.6 pbl-c2만 존재하지 않을 때

`pbl-c1`, `pbl-web`, `pbl-br-staff`는 정상인데 **pbl-c2만 아직 생성되지 않은 경우** 다음을 사용할 수 있다.

다음 명령은 `pbl-c2`와 `pbl-c2-h`가 모두 존재하지 않는 것을 확인한 뒤에만 실행한다.

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo ip netns add pbl-c2

sudo ip link add pbl-c2-h type veth peer name pbl-c2-n

sudo ip link set pbl-c2-h master pbl-br-staff
sudo ip link set pbl-c2-h up

sudo ip link set pbl-c2-n netns pbl-c2

sudo ip -n pbl-c2 link set pbl-c2-n name eth0
sudo ip -n pbl-c2 link set lo up
sudo ip -n pbl-c2 link set eth0 up

sudo ip -n pbl-c2 addr add 192.168.10.11/26 dev eth0
```

확인:

```bash
sudo ip -n pbl-c2 -br addr
sudo ip -br link show master pbl-br-staff
```

`pbl-c2`가 이미 부분적으로 만들어져 있다면 위 명령을 반복해 중복 자원을 만들지 않는다.

먼저 어떤 항목이 남아 있는지 확인한다.

```bash
sudo ip netns list
sudo ip link show pbl-c2-h
sudo ip -n pbl-c2 link
```

---

# 5.7 라우팅 테이블 사전 확인

이번 실험에서 매우 중요하다.

### 실행 위치 → Linux 호스트

```bash
sudo ip -n pbl-c1 route
```

정상 `/26` 상태에서는 보통 다음과 비슷하다.

```text
192.168.10.0/26 dev eth0 proto kernel scope link src 192.168.10.10
```

특히 다음을 확인한다.

### 기본 경로

```bash
sudo ip -n pbl-c1 route show default
```

### 예상

아무 출력도 없어야 한다.

---

### 전체 main routing table 재확인

```bash
sudo ip -n pbl-c1 route show
```

이번 실습에서는 의도적으로 추가한 다음과 같은 경로가 있으면 안 된다.

예:

```text
default via ...
192.168.10.16/28 via ...
192.168.10.20/32 via ...
```

이런 경로가 있으면 `/28`로 바꿔도 `.20`으로 가는 별도 경로가 남아 실험 결과가 달라질 수 있다.

**라우팅 테이블 전체를 `flush`하지 않는다.**

어떤 경로를 이전 실습에서 직접 추가했다는 사실이 확인된 경우에만 그 **정확한 경로 하나만** 제거한다.

---

# 5.8 정책 라우팅도 확인한다

일반적인 실습 환경에서는 기본 규칙만 있을 가능성이 높지만, 예상과 다른 결과를 방지하기 위해 기록한다.

### 실행 위치 → Linux 호스트

```bash
sudo ip -n pbl-c1 rule
```

특별한 정책 라우팅을 이전에 구성한 적이 없다면 임의로 수정하지 않는다.

---

# 5.9 웹 서비스 확인

### 실행 위치 → Linux 호스트

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

정상이라면 TCP 8080에서 LISTEN 상태가 확인되어야 한다.

없다면 우선 기존 웹 서버 로그나 프로세스를 확인한다.

임의의 Python 프로세스를 모두 종료하면 안 된다.

웹 서비스가 전혀 없다면 A는 실습용 페이지를 준비한다.

```bash
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week4
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week4/www
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week4/logs
```

```bash
sudo tee /srv/pbl/week4/www/index.html >/dev/null <<'EOF'
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>PBL Week 4</title>
</head>
<body>
  <h1>PBL Week 4 OK</h1>
  <p>Subnet mask experiment web server.</p>
</body>
</html>
EOF
```

8080을 사용하는 다른 실습 서버가 없는 것을 확인한 뒤 실행한다.

```bash
sudo bash -c '
nohup ip netns exec pbl-web \
  python3 -m http.server 8080 \
  --bind 192.168.10.20 \
  --directory /srv/pbl/week4/www \
  >/srv/pbl/week4/logs/web.log 2>&1 < /dev/null &
echo $! >/run/pbl-w4-web.pid
'
```

재확인:

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

---

# 5.10 정상 통신 자체 점검

### 실행 위치 → Linux 호스트

```bash
sudo ip netns exec pbl-c1 ping -n -c 2 192.168.10.11
```

```bash
sudo ip netns exec pbl-c1 ping -n -c 2 192.168.10.20
```

```bash
sudo ip netns exec pbl-c1 \
  curl --noproxy '*' -fsS --max-time 5 \
  http://192.168.10.20:8080/
```

정상이라면:

```text
.11 ping 성공
.20 ping 성공
HTTP 응답 성공
```

을 확인한다.

이 단계에서 실패한다면 마스크 오류 실험으로 넘어가지 않는다.

---

# 6. 참여자용 4주차 보조 스크립트 설치

공유 서버에서 A·B·C·D가 임의의 관리자 명령을 실행하게 하지 않고 이번 실습에 필요한 작업만 허용한다.

이번 스크립트는 계정명을 `pbl-a`, `pbl-b`처럼 하드코딩하지 않는다.

대신 **`pbl` 그룹에 속한 실제 실습 계정**을 허용한다.

---

## 6.1 디렉터리 준비

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week4
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week4/captures
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week4/logs
```

---

# 6.2 실습 계정 그룹 확인

각 참여자의 실제 Linux 계정을 확인한다.

예:

```bash
id <실습계정>
```

`pbl` 그룹이 없다면 A가 추가한다.

```bash
sudo usermod -aG pbl <실습계정>
```

그룹 변경은 기존 로그인 세션에 즉시 적용되지 않을 수 있다.

해당 사용자는 SSH에서 로그아웃한 뒤 다시 로그인한다.

```bash
id
```

에서 `pbl` 그룹을 확인한다.

---

# 6.3 pbl-w4-lab 저장

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w4-lab
```

다음 내용을 전체 저장한다.

```bash
#!/usr/bin/env bash
set -euo pipefail

ACTION="${1:-}"
ARG="${2:-}"

BASE="/srv/pbl/week4"
CAPDIR="${BASE}/captures"
LOGDIR="${BASE}/logs"

LOCKDIR="/run/lock/pbl-w4-capture"
OWNERFILE="${LOCKDIR}/owner"
PIDFILE="${LOCKDIR}/pid"
FILEFILE="${LOCKDIR}/file"

if [[ ${EUID} -ne 0 ]]; then
    echo "ERROR: sudo를 통해 실행하십시오." >&2
    exit 1
fi

REAL_USER="${SUDO_USER:-root}"

if [[ "${REAL_USER}" != "root" ]]; then
    if ! id -nG "${REAL_USER}" 2>/dev/null \
        | tr ' ' '\n' \
        | grep -Fxq pbl; then
        echo "ERROR: ${REAL_USER} 계정은 pbl 그룹에 속하지 않습니다." >&2
        exit 1
    fi
fi

for cmd in ip ping curl tcpdump; do
    if ! command -v "${cmd}" >/dev/null 2>&1; then
        echo "ERROR: 필요한 명령이 없습니다: ${cmd}" >&2
        exit 1
    fi
done

mkdir -p "${CAPDIR}" "${LOGDIR}"

need_ns() {
    local ns="$1"

    if ! ip netns list | awk '{print $1}' | grep -Fxq "${ns}"; then
        echo "ERROR: 네임스페이스가 없습니다: ${ns}" >&2
        exit 1
    fi
}

need_baseline() {
    need_ns pbl-c1
    need_ns pbl-c2
    need_ns pbl-web

    if ! ip link show pbl-br-staff >/dev/null 2>&1; then
        echo "ERROR: pbl-br-staff가 없습니다." >&2
        exit 1
    fi

    if ! ip link show pbl-c1-h >/dev/null 2>&1; then
        echo "ERROR: pbl-c1-h가 없습니다." >&2
        exit 1
    fi
}

soft_run() {
    set +e
    "$@"
    rc=$?
    set -e

    echo
    echo "명령 종료 코드: ${rc}"
    return 0
}

check_extra_c1_addr() {
    extras="$(
        ip -n pbl-c1 -o -4 addr show dev eth0 \
        | awk '{print $4}' \
        | grep -vE '^192\.168\.10\.10/(26|28)$' \
        || true
    )"

    if [[ -n "${extras}" ]]; then
        echo "ERROR: pbl-c1 eth0에 예상하지 못한 IPv4 주소가 있습니다." >&2
        echo "${extras}" >&2
        echo "임의로 삭제하지 말고 A가 확인하십시오." >&2
        exit 1
    fi
}

set_prefix() {
    local prefix="$1"

    if [[ -d "${LOCKDIR}" ]]; then
        echo "ERROR: 캡처 실행 중에는 주소 상태를 변경하지 마십시오." >&2
        exit 1
    fi

    need_baseline
    check_extra_c1_addr

    ip -n pbl-c1 addr del 192.168.10.10/26 dev eth0 \
        2>/dev/null || true

    ip -n pbl-c1 addr del 192.168.10.10/28 dev eth0 \
        2>/dev/null || true

    ip -n pbl-c1 addr add "192.168.10.10/${prefix}" dev eth0

    echo "=== pbl-c1 주소 ==="
    ip -n pbl-c1 -br addr show dev eth0

    echo
    echo "=== pbl-c1 main routing table ==="
    ip -n pbl-c1 route
}

case "${ACTION}" in

check)
    need_baseline

    echo "=== pbl-c1 ==="
    ip -n pbl-c1 -br addr
    echo
    ip -n pbl-c1 route

    echo
    echo "=== pbl-c1 default route ==="
    ip -n pbl-c1 route show default

    echo
    echo "=== pbl-c1 route rules ==="
    ip -n pbl-c1 rule

    echo
    echo "=== pbl-c2 ==="
    ip -n pbl-c2 -br addr

    echo
    echo "=== pbl-web ==="
    ip -n pbl-web -br addr

    echo
    echo "=== pbl-web TCP 8080 ==="
    ip netns exec pbl-web ss -lntp | grep ':8080' || {
        echo "ERROR: TCP 8080 웹 서비스가 보이지 않습니다." >&2
        exit 1
    }

    echo
    echo "=== bridge 연결 ==="
    ip -br link show master pbl-br-staff || true

    echo
    if [[ -d "${LOCKDIR}" ]]; then
        echo -n "현재 캡처 사용자: "
        cat "${OWNERFILE}" 2>/dev/null || echo "unknown"
    else
        echo "현재 활성 캡처 없음"
    fi
    ;;

normal)
    echo "pbl-c1을 정상 /26 상태로 설정합니다."
    set_prefix 26
    ;;

bad-mask)
    echo "pbl-c1만 오류 /28 상태로 설정합니다."
    set_prefix 28
    ;;

route-get)
    need_baseline

    case "${ARG}" in
        192.168.10.11|192.168.10.20)
            ;;
        *)
            echo "ERROR: 허용 목적지는 192.168.10.11 또는 192.168.10.20입니다." >&2
            exit 2
            ;;
    esac

    echo "ROUTE GET TIME: $(date --iso-8601=seconds)"
    echo "목적지: ${ARG}"
    echo

    soft_run ip -n pbl-c1 route get "${ARG}"
    ;;

neigh-show)
    need_baseline
    ip -n pbl-c1 neigh show dev eth0
    ;;

neigh-flush)
    need_baseline

    echo "pbl-c1 eth0의 이웃 캐시만 비웁니다."
    ip -n pbl-c1 neigh flush dev eth0 >/dev/null 2>&1 || true

    echo "=== flush 후 ==="
    ip -n pbl-c1 neigh show dev eth0
    ;;

ping)
    need_baseline

    case "${ARG}" in
        192.168.10.11|192.168.10.20)
            ;;
        *)
            echo "ERROR: 허용 목적지는 192.168.10.11 또는 192.168.10.20입니다." >&2
            exit 2
            ;;
    esac

    echo "PING TIME: $(date --iso-8601=seconds)"
    echo "목적지: ${ARG}"
    echo

    soft_run \
        ip netns exec pbl-c1 \
        ping -n -c 2 -W 1 "${ARG}"
    ;;

http20)
    need_baseline

    echo "HTTP TIME: $(date --iso-8601=seconds)"
    echo

    soft_run \
        ip netns exec pbl-c1 \
        curl \
        --noproxy '*' \
        -v \
        --connect-timeout 2 \
        --max-time 5 \
        http://192.168.10.20:8080/
    ;;

capture-start)
    need_baseline

    phase="${ARG}"

    case "${phase}" in
        normal|bad|recovery)
            ;;
        *)
            echo "ERROR: phase는 normal, bad, recovery 중 하나입니다." >&2
            exit 2
            ;;
    esac

    if ! mkdir "${LOCKDIR}" 2>/dev/null; then
        echo "ERROR: 다른 참여자의 캡처가 이미 실행 중입니다." >&2
        echo -n "현재 사용자: " >&2
        cat "${OWNERFILE}" 2>/dev/null || echo "unknown" >&2
        exit 1
    fi

    echo "${REAL_USER}" > "${OWNERFILE}"

    stamp="$(date +%Y%m%d-%H%M%S)"
    outfile="${CAPDIR}/${REAL_USER}-${phase}-${stamp}.pcap"
    logfile="${LOGDIR}/${REAL_USER}-${phase}-${stamp}.tcpdump.log"

    nohup tcpdump \
        -i pbl-c1-h \
        -U \
        -s 0 \
        -w "${outfile}" \
        >"${logfile}" 2>&1 < /dev/null &

    pid=$!

    echo "${pid}" > "${PIDFILE}"
    echo "${outfile}" > "${FILEFILE}"

    sleep 0.5

    if ! kill -0 "${pid}" 2>/dev/null; then
        echo "ERROR: tcpdump 시작 실패" >&2
        cat "${logfile}" >&2 || true
        rm -rf "${LOCKDIR}"
        exit 1
    fi

    echo "CAPTURE STARTED"
    echo "사용자: ${REAL_USER}"
    echo "단계: ${phase}"
    echo "시각: $(date --iso-8601=seconds)"
    echo "파일: ${outfile}"
    ;;

capture-stop)
    if [[ ! -d "${LOCKDIR}" ]]; then
        echo "ERROR: 현재 활성 캡처가 없습니다." >&2
        exit 1
    fi

    owner="$(cat "${OWNERFILE}" 2>/dev/null || true)"

    if [[ "${owner}" != "${REAL_USER}" &&
          "${REAL_USER}" != "root" ]]; then
        echo "ERROR: 현재 캡처는 ${owner} 사용자의 것입니다." >&2
        exit 1
    fi

    if [[ ! -f "${PIDFILE}" || ! -f "${FILEFILE}" ]]; then
        echo "ERROR: 캡처 상태 파일이 없습니다." >&2
        exit 1
    fi

    pid="$(cat "${PIDFILE}")"
    outfile="$(cat "${FILEFILE}")"

    if [[ "${outfile}" != "${CAPDIR}/"*.pcap ]]; then
        echo "ERROR: 예상하지 못한 캡처 파일 경로입니다." >&2
        exit 1
    fi

    if [[ "${pid}" =~ ^[0-9]+$ ]] &&
       kill -0 "${pid}" 2>/dev/null; then

        cmdline="$(
            tr '\0' ' ' < "/proc/${pid}/cmdline" 2>/dev/null || true
        )"

        if [[ "${cmdline}" == *"tcpdump"* &&
              "${cmdline}" == *"pbl-c1-h"* ]]; then

            kill -INT "${pid}" 2>/dev/null || true

            for _ in {1..30}; do
                kill -0 "${pid}" 2>/dev/null || break
                sleep 0.1
            done

            if kill -0 "${pid}" 2>/dev/null; then
                kill -TERM "${pid}" 2>/dev/null || true
            fi
        else
            echo "ERROR: PID가 예상한 tcpdump가 아닙니다." >&2
            exit 1
        fi
    fi

    chgrp pbl "${outfile}" || true
    chmod 0640 "${outfile}" || true

    rm -rf "${LOCKDIR}"

    echo "CAPTURE STOPPED"
    echo "종료 시각: $(date --iso-8601=seconds)"
    echo "PCAP 파일:"
    echo "${outfile}"
    ;;

*)
    cat <<'EOF'
사용법:
  sudo /usr/local/sbin/pbl-w4-lab check

  sudo /usr/local/sbin/pbl-w4-lab normal
  sudo /usr/local/sbin/pbl-w4-lab bad-mask

  sudo /usr/local/sbin/pbl-w4-lab route-get 192.168.10.11
  sudo /usr/local/sbin/pbl-w4-lab route-get 192.168.10.20

  sudo /usr/local/sbin/pbl-w4-lab neigh-show
  sudo /usr/local/sbin/pbl-w4-lab neigh-flush

  sudo /usr/local/sbin/pbl-w4-lab ping 192.168.10.11
  sudo /usr/local/sbin/pbl-w4-lab ping 192.168.10.20

  sudo /usr/local/sbin/pbl-w4-lab http20

  sudo /usr/local/sbin/pbl-w4-lab capture-start normal
  sudo /usr/local/sbin/pbl-w4-lab capture-start bad
  sudo /usr/local/sbin/pbl-w4-lab capture-start recovery
  sudo /usr/local/sbin/pbl-w4-lab capture-stop
EOF
    exit 2
    ;;
esac
```

저장 후:

```bash
sudo chown root:root /usr/local/sbin/pbl-w4-lab
sudo chmod 0755 /usr/local/sbin/pbl-w4-lab
```

---

# 6.4 sudo 권한 제한

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo visudo -f /etc/sudoers.d/pbl-week4
```

다음을 저장한다.

```text
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w4-lab *
```

이 규칙은 wrapper 명령에 인수를 전달할 수 있도록 허용하지만, wrapper 내부에서 허용된 동작과 목적지를 다시 제한한다.

저장 후:

```bash
sudo chmod 0440 /etc/sudoers.d/pbl-week4
sudo visudo -cf /etc/sudoers.d/pbl-week4
```

---

# 6.5 A의 최종 사전 점검

### 실행 위치 → Linux 호스트

```bash
sudo /usr/local/sbin/pbl-w4-lab normal
sudo /usr/local/sbin/pbl-w4-lab check
```

반드시 확인한다.

```text
pbl-c1  = 192.168.10.10/26
pbl-c2  = 192.168.10.11/26
pbl-web = 192.168.10.20/26

default route 없음
불필요한 수동 route 없음
TCP 8080 LISTEN
활성 capture 없음
```

---

# 7. 실습 참여자 A·B·C·D의 활동

이 절은 네 명 모두 직접 수행한다.

공유 `pbl-c1`의 설정을 변경하므로:

```text
A → 정상/오류/복구 전체 완료
B → 정상/오류/복구 전체 완료
C → 정상/오류/복구 전체 완료
D → 정상/오류/복구 전체 완료
```

처럼 **한 명씩 순서대로** 수행한다.

PCAP을 개인 PC에 내려받은 뒤 Wireshark 분석은 동시에 할 수 있다.

각 참여자는 총 세 개의 캡처를 만든다.

```text
normal
bad
recovery
```

---

# 8. 개인 실험 시작

## 8.1 Windows에서 SSH 접속

### 실행 위치 → 개인 Windows PowerShell

```powershell
ssh <본인_실습계정>@<SERVER_MANAGEMENT_IP>
```

`192.168.10.10`, `.11`, `.20`은 SSH용 관리 IP가 아니다.

---

## 8.2 현재 사용자와 시간 기록

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
whoami
date --iso-8601=seconds
```

개인 보고서에 기록한다.

---

# 9. 1단계 — 정상 /26 실험

## 9.1 실행 전에 예상 작성

명령을 실행하기 전 다음을 작성한다.

```text
[정상 /26 예상]

192.168.10.11
- route 판단:
- ping 예상:
- 예상 패킷:

192.168.10.20
- route 판단:
- ping 예상:
- HTTP 예상:
- 예상 패킷:
```

---

# 9.2 정상 /26으로 맞춘다

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w4-lab normal
```

예상:

```text
pbl-c1 주소:
192.168.10.10/26
```

라우팅 테이블에는 다음과 비슷한 직접 연결 경로가 보인다.

```text
192.168.10.0/26 dev eth0 ...
```

---

# 9.3 주소와 경로 확인

```bash
sudo /usr/local/sbin/pbl-w4-lab check
```

특히:

```text
pbl-c1 = /26
default route 없음
```

을 확인한다.

---

# 9.4 정상 상태 캡처 시작

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-start normal
```

`CAPTURE STARTED`가 표시된 뒤 다음 단계로 진행한다.

---

# 9.5 캐시의 영향을 줄인다

ARP가 이미 학습되어 있으면 새로운 ARP가 나타나지 않을 수 있다.

이번 비교에서는 각 단계 시작 시 pbl-c1의 `eth0` 이웃 캐시만 비운다.

```bash
sudo /usr/local/sbin/pbl-w4-lab neigh-flush
```

이것은:

```text
IP 주소 변경 X
라우팅 경로 변경 X
Linux 호스트의 ARP 캐시 변경 X
```

이며 `pbl-c1` 내부의 `eth0` 이웃 정보만 비우는 것이다.

---

# 9.6 .11 경로 조회

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  route-get 192.168.10.11
```

정상 `/26`에서는 다음과 비슷한 정보가 예상된다.

```text
192.168.10.11 dev eth0 src 192.168.10.10
```

출력 형식은 iproute2 버전에 따라 일부 달라질 수 있으므로 자신의 실제 값을 기록한다.

---

# 9.7 .20 경로 조회

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  route-get 192.168.10.20
```

역시 `eth0`을 통해 직접 연결된 목적지로 해석되는 결과가 예상된다.

---

# 9.8 .11 ping

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.11
```

예상:

```text
ICMP Echo Request
ICMP Echo Reply
```

처음 통신하는 상태라면 그 전에 `.11`의 MAC 주소를 알아내기 위한 ARP가 관찰될 수 있다.

---

# 9.9 .20 ping

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.20
```

예상:

```text
ICMP Echo Request
ICMP Echo Reply
```

캐시를 비운 뒤 첫 `.20` 통신이라면 ARP도 나타날 가능성이 높다.

---

# 9.10 .20 HTTP 요청

```bash
sudo /usr/local/sbin/pbl-w4-lab http20
```

예상:

```text
Trying 192.168.10.20:8080...
Connected ...
GET /
HTTP ... 200 OK
```

실제 HTTP 버전·헤더·출력 순서는 달라질 수 있다.

중요한 것은:

```text
TCP 연결 발생
HTTP 요청 발생
HTTP 응답 발생
```

여부다.

---

# 9.11 이웃 정보 확인

```bash
sudo /usr/local/sbin/pbl-w4-lab neigh-show
```

예를 들어 `.11`과 `.20`에 대한 MAC 정보가 나타날 수 있다.

실제 MAC 주소는 미리 정해진 예시가 아니므로 자신의 값을 기록한다.

---

# 9.12 정상 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-stop
```

출력된 PCAP 파일 이름을 기록한다.

예:

```text
/srv/pbl/week4/captures/<계정>-normal-....pcap
```

---

# 10. 2단계 — pbl-c1만 /28로 변경

## 10.1 먼저 예상한다

**아직 설정을 변경하지 말고 먼저 기록한다.**

```text
[오류 /28 예상]

pbl-c1 = 192.168.10.10/28

192.168.10.11
- 같은 subnet인가?
- route-get 예상:
- ping 예상:
- ARP 예상:

192.168.10.20
- 같은 subnet인가?
- route-get 예상:
- ping 예상:
- HTTP 예상:
- ARP/ICMP/TCP 패킷 예상:
```

---

# 10.2 기존 /26 주소를 제거하고 /28을 추가

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w4-lab bad-mask
```

스크립트 내부에서는 핵심적으로 다음과 같은 변경이 이루어진다.

```bash
ip -n pbl-c1 addr del 192.168.10.10/26 dev eth0
ip -n pbl-c1 addr add 192.168.10.10/28 dev eth0
```

즉 IP 자체:

```text
192.168.10.10
```

은 그대로이고 prefix만:

```text
/26 → /28
```

로 바뀐다.

---

# 10.3 주소 변화 확인

```bash
sudo /usr/local/sbin/pbl-w4-lab check
```

확인:

```text
pbl-c1 = 192.168.10.10/28
```

동시에 pbl-c2와 pbl-web은 그대로여야 한다.

```text
pbl-c2  = 192.168.10.11/26
pbl-web = 192.168.10.20/26
```

---

# 10.4 연결 경로가 어떻게 바뀌었는지 확인

오류 상태의 `ip route`에서 핵심은 다음 변화다.

정상:

```text
192.168.10.0/26 dev eth0 ...
```

오류:

```text
192.168.10.0/28 dev eth0 ...
```

즉 직접 연결 범위가:

```text
192.168.10.0 ~ 63
```

에서:

```text
192.168.10.0 ~ 15
```

로 줄어든다.

---

# 10.5 오류 상태 캡처 시작

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-start bad
```

---

# 10.6 이웃 캐시 비우기

```bash
sudo /usr/local/sbin/pbl-w4-lab neigh-flush
```

---

# 10.7 .11 경로 조회

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  route-get 192.168.10.11
```

`192.168.10.11`은 여전히 `192.168.10.0/28` 범위 안에 있다.

따라서 직접 연결된 `eth0` 경로를 얻는 것이 예상된다.

---

# 10.8 .20 경로 조회

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  route-get 192.168.10.20
```

이번에는 중요한 차이가 예상된다.

`192.168.10.20`은 직접 연결된 `/28`에 포함되지 않고 기본 경로도 없기 때문에 경로 조회가 실패할 수 있다.

환경에 따라 메시지 표현은 다를 수 있으나 예를 들어:

```text
Network is unreachable
```

형태가 나타날 수 있다.

**오류 메시지를 추측해서 보고서에 쓰지 말고 실제 출력과 종료 코드를 기록한다.**

---

# 10.9 .11 ping

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.11
```

이번 실험의 중요한 결과 중 하나다.

마스크가 잘못되었지만 `.11`은 같은 `/28`이므로 여전히 성공할 수 있다.

따라서:

> 잘못된 서브넷 마스크 = 모든 통신이 무조건 실패

라고 판단하면 안 된다.

---

# 10.10 .20 ping

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.20
```

실제 화면의 오류를 그대로 기록한다.

정상 `/26` 때처럼 ICMP Echo Request가 반드시 발생한다고 가정하지 않는다.

운영체제가 먼저 경로를 찾지 못하면 `pbl-c1-h`로 ICMP 패킷이 나오지 않을 수 있다.

---

# 10.11 .20 HTTP

```bash
sudo /usr/local/sbin/pbl-w4-lab http20
```

`curl` 화면의 실제 오류를 기록한다.

여기에서도 중요한 것은:

```text
HTTP 응답 실패
```

만 보는 것이 아니다.

다음을 함께 확인한다.

```text
ip route get 결과
curl 오류
캡처에서 TCP SYN 존재 여부
ARP 존재 여부
ICMP 존재 여부
```

---

# 10.12 이웃 정보 확인

```bash
sudo /usr/local/sbin/pbl-w4-lab neigh-show
```

예상상 `.11`은 통신했으므로 이웃 정보가 생성될 수 있다.

반면 `.20`에 대해 실제 ARP까지 진행하지 않았다면 `.20` 이웃 항목이 새로 만들어지지 않을 수 있다.

자신의 실제 결과를 기록한다.

---

# 10.13 오류 상태 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-stop
```

PCAP 이름을 기록한다.

```text
<계정>-bad-....pcap
```

---

# 11. 3단계 — /26으로 복구

## 11.1 복구 전에 예상

```text
[복구 예상]

pbl-c1을 /26으로 되돌리면:

.11 route:
.20 route:
.11 ping:
.20 ping:
HTTP:
```

---

# 11.2 /28 제거 후 /26 추가

```bash
sudo /usr/local/sbin/pbl-w4-lab normal
```

실질적으로:

```text
192.168.10.10/28 제거
192.168.10.10/26 추가
```

가 이루어진다.

---

# 11.3 복구 상태 확인

```bash
sudo /usr/local/sbin/pbl-w4-lab check
```

확인:

```text
pbl-c1 = 192.168.10.10/26
```

경로:

```text
192.168.10.0/26 dev eth0 ...
```

---

# 11.4 복구 캡처 시작

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-start recovery
```

---

# 11.5 캐시 비우기

```bash
sudo /usr/local/sbin/pbl-w4-lab neigh-flush
```

---

# 11.6 같은 조건으로 다시 시험

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  route-get 192.168.10.11
```

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  route-get 192.168.10.20
```

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.11
```

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.20
```

```bash
sudo /usr/local/sbin/pbl-w4-lab http20
```

---

# 11.7 복구 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-stop
```

이제 개인별로:

```text
normal PCAP
bad PCAP
recovery PCAP
```

세 개가 있어야 한다.

---

# 12. Windows PC로 PCAP 내려받기

Linux SSH 세션에서 각 파일의 정확한 이름을 기록한다.

Windows에서 폴더 생성:

### 실행 위치 → 개인 Windows PowerShell

```powershell
New-Item -ItemType Directory -Force "$HOME\pbl-week4"
```

각 파일을 내려받는다.

```powershell
scp <본인_실습계정>@<SERVER_MANAGEMENT_IP>:/srv/pbl/week4/captures/<normal파일>.pcap "$HOME\pbl-week4\"
```

```powershell
scp <본인_실습계정>@<SERVER_MANAGEMENT_IP>:/srv/pbl/week4/captures/<bad파일>.pcap "$HOME\pbl-week4\"
```

```powershell
scp <본인_실습계정>@<SERVER_MANAGEMENT_IP>:/srv/pbl/week4/captures/<recovery파일>.pcap "$HOME\pbl-week4\"
```

예시 파일명을 그대로 사용하지 않는다.

---

# 13. Wireshark 분석

### 실행 위치 → 개인 Windows PC

Wireshark에서 세 PCAP을 각각 연다.

이번에는 **정상 → 오류 → 복구**를 비교하는 것이 핵심이다.

Wireshark의 Display Filter는 캡처 파일에서 패킷을 삭제하지 않고 화면에 보이는 패킷만 제한한다. 따라서 필터 결과가 비었다면 필터를 지워 전체 패킷도 함께 확인한다.

---

# 13.1 전체 관련 프로토콜 보기

Display Filter:

```text
arp || icmp || tcp.port == 8080
```

확인:

```text
ARP
ICMP
TCP 8080
```

이 어떤 단계에서 나타나는지 비교한다.

---

# 13.2 .11 통신 확인

```text
ip.addr == 192.168.10.11
```

오류 `/28` 상태에서도 `.11`과의 ICMP 통신이 보이는지 확인한다.

---

# 13.3 .20 통신 확인

```text
ip.addr == 192.168.10.20
```

정상·복구 상태에서는:

```text
ICMP
TCP
HTTP
```

관련 패킷이 나타날 수 있다.

오류 `/28` 상태에서는 결과가 달라질 수 있다.

---

# 13.4 ARP 따로 확인

```text
arp
```

각 ARP Frame에서 Packet Details의:

```text
Address Resolution Protocol
```

을 펼친다.

다음을 확인한다.

```text
Sender MAC address
Sender IP address
Target MAC address
Target IP address
```

실제 MAC 주소를 기록한다.

---

# 13.5 ICMP 확인

```text
icmp
```

정상 상태에서 `.11`과 `.20`의:

```text
Echo request
Echo reply
```

를 찾아본다.

오류 상태에서 `.11`과 `.20` 결과가 어떻게 달라지는지 비교한다.

---

# 13.6 TCP 연결 시도 확인

```text
tcp.port == 8080
```

정상·복구 PCAP에서는 pbl-c1과 pbl-web 사이 TCP 연결을 찾는다.

TCP 연결의 첫 SYN만 보고 싶다면:

```text
tcp.dstport == 8080 &&
tcp.flags.syn == 1 &&
tcp.flags.ack == 0
```

정상 상태에서 다음 방향의 SYN을 찾는다.

```text
192.168.10.10 → 192.168.10.20
```

---

# 13.7 HTTP 확인

```text
http
```

또는:

```text
http.request
```

정상·복구 파일에서 HTTP로 해석된다면 요청을 확인한다.

8080을 HTTP로 자동 해석하지 않는 환경에서는 먼저:

```text
tcp.port == 8080
```

으로 TCP 패킷 자체의 존재를 확인한다.

`http` 필터에 아무것도 없다는 이유만으로 통신 실패라고 결론내리지 않는다.

---

# 14. “패킷이 없다”를 증거로 사용하는 방법

오류 `/28` PCAP에서 `.20` 관련 패킷이 보이지 않았다고 하자.

그 사실 하나만으로:

```text
"패킷이 네트워크 중간에서 유실되었다."
```

라고 결론내리면 안 된다.

이번 실험에서는 다음 증거가 함께 있을 때 더 강한 판단이 가능하다.

```text
1. 캡처가 요청보다 먼저 정상적으로 시작됨.
2. 캡처 인터페이스가 pbl-c1-h임.
3. 명령 실행 시각이 캡처 구간 안에 있음.
4. ip route get 192.168.10.20이 경로 실패를 보임.
5. ping 또는 curl도 로컬 경로 관련 오류를 보임.
6. 같은 PCAP에서 .11 통신은 실제로 잡힘.
7. .20에 대한 ICMP/TCP가 보이지 않음.
```

이 경우:

> 캡처가 실패한 것이 아니라 pbl-c1의 IP 계층에서 `.20`으로 내보낼 경로를 선택하지 못해 패킷이 해당 인터페이스로 나오지 않았다는 해석

을 근거와 함께 제시할 수 있다.

---

# 15. 정상·오류·복구 결과 비교

아래는 **실험 후 비교용 기준**이다.

실험 전에 그대로 베껴서 예상 결과로 제출하지 않는다.

| 항목 | 정상 `/26` | 오류 `/28` | 복구 `/26` |
|---|---|---|---|
| pbl-c1 주소 | `.10/26` | `.10/28` | `.10/26` |
| 직접 연결 경로 | `192.168.10.0/26` | `192.168.10.0/28` | `192.168.10.0/26` |
| `.11`의 위치 판단 | 로컬 | 로컬 | 로컬 |
| `.20`의 위치 판단 | 로컬 | 다른 subnet | 로컬 |
| `.11 route get` | eth0 직접 | eth0 직접 | eth0 직접 |
| `.20 route get` | eth0 직접 | 경로 없음 예상 | eth0 직접 |
| `.11 ping` | 성공 예상 | 성공 예상 | 성공 예상 |
| `.20 ping` | 성공 예상 | 실패 예상 | 성공 예상 |
| `.20 HTTP` | 성공 예상 | 실패 예상 | 성공 예상 |
| `.11 관련 패킷` | 관찰 가능 | 관찰 가능 | 관찰 가능 |
| `.20 ICMP/TCP` | 관찰 가능 | 발생하지 않을 수 있음 | 관찰 가능 |
| `.20 ARP` | 첫 통신 시 관찰 가능 | 반드시 발생한다고 할 수 없음 | 첫 통신 시 관찰 가능 |

---

# 16. 이번 실험에서 특히 설명해야 하는 비대칭

오류 상태에서:

```text
pbl-c1 = 192.168.10.10/28
pbl-web = 192.168.10.20/26
```

이다.

pbl-c1의 관점:

```text
.20은 내 /28 밖에 있다.
```

pbl-web의 관점:

```text
.10은 내 /26 안에 있다.
```

즉 두 장치가 동일한 상대방을 보고 사용하는 subnet 판단이 서로 다를 수 있다.

하지만 이번 상황에서는 pbl-c1이 처음부터 `.20`으로 보낼 경로를 찾지 못할 수 있으므로 pbl-web의 반환 판단까지 도달하지 않을 수 있다.

따라서:

```text
"서버는 .10을 로컬로 보니까 통신이 될 것이다."
```

라고 단순하게 결론내리면 안 된다.

요청을 **처음 보내는 pbl-c1의 경로 판단부터** 확인해야 한다.

---

# 17. 예상과 다른 결과가 나올 때 점검

## 17.1 `/28`인데 .20 통신이 된다

가장 먼저:

```bash
sudo /usr/local/sbin/pbl-w4-lab check
```

확인한다.

특히:

```bash
sudo ip -n pbl-c1 -4 addr show dev eth0
sudo ip -n pbl-c1 route
sudo ip -n pbl-c1 rule
```

을 A가 확인한다.

가능한 원인:

```text
/26 주소가 동시에 남아 있음
default route가 존재함
.20으로 가는 host route가 존재함
192.168.10.16/28 경로가 추가되어 있음
정책 라우팅이 존재함
실제로는 다른 namespace에서 요청함
```

이다.

---

# 17.2 `/28`에서 .11도 실패한다

`.11`은 `/28` 안에 있으므로 마스크 하나만으로 설명하기 어렵다.

확인:

```bash
sudo ip -n pbl-c2 -br addr
sudo ip -n pbl-c2 link show eth0
sudo ip -br link show master pbl-br-staff
```

확인할 것:

```text
pbl-c2 eth0 UP
192.168.10.11/26 존재
pbl-c2-h가 bridge에 연결됨
```

Wireshark에서는:

```text
arp
```

와:

```text
icmp
```

를 확인한다.

---

# 17.3 정상 상태에서도 .20 HTTP가 실패한다

먼저 ping과 HTTP를 분리한다.

```bash
sudo /usr/local/sbin/pbl-w4-lab \
  ping 192.168.10.20
```

ping은 성공하지만 HTTP가 실패한다면:

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

확인한다.

TCP 8080 서비스가 정지한 문제일 수 있다.

마스크 문제라고 단정하지 않는다.

---

# 17.4 `Connection refused`가 나온다

이는 일반적으로 목적지까지 아무 통신도 못 했다는 의미와 같지 않다.

캡처에:

```text
TCP SYN
TCP RST
```

같은 패킷이 있는지 확인한다.

해당 서비스가 LISTEN 중인지도 함께 확인한다.

---

# 17.5 `/28`에서 .20 관련 ARP가 보인다

먼저 그 ARP가 정말 pbl-c1이 발생시킨 `.20` 대상 ARP인지 확인한다.

확인할 것:

```text
Frame 시각
Sender IP
Target IP
캡처 파일 단계
캡처 인터페이스
```

또한:

```bash
sudo ip -n pbl-c1 route
sudo ip -n pbl-c1 route get 192.168.10.20
```

을 확인한다.

추가 경로가 있다면 실험 조건이 달라진 것이다.

---

# 17.6 정상 파일에서 ARP가 보이지 않는다

바로 오류라고 판단하지 않는다.

가능한 원인:

```text
이웃 캐시에 MAC 주소가 이미 존재함
cache flush를 실행하지 않음
캡처보다 먼저 통신함
잘못된 PCAP을 열었음
Display Filter가 ARP를 숨김
```

`ip neigh`와 전체 PCAP을 함께 확인한다.

---

# 17.7 PCAP에 아무것도 없다

다음을 순서대로 확인한다.

```text
1. capture-start 성공 메시지를 봤는가?
2. 요청 전에 capture-start했는가?
3. pbl-c1-h가 실제 캡처 인터페이스인가?
4. capture-stop으로 정상 저장했는가?
5. 올바른 PCAP 파일을 열었는가?
6. Wireshark Display Filter를 모두 지워 보았는가?
7. tcpdump 로그에 오류가 없는가?
```

A가 확인:

```bash
ls -lh /srv/pbl/week4/captures/
ls -lh /srv/pbl/week4/logs/
```

필요하면 해당 tcpdump 로그를 읽는다.

---

# 17.8 `.20 ping`이 timeout인데 패킷은 나간다

이 결과는 이번 표준 `/28 + route 없음` 조건과 다르다.

패킷이 실제로 나갔다면:

```text
경로 선택은 이루어졌다는 증거
```

이다.

그다음:

```text
어느 MAC으로 전송했는가?
ARP는 누구에게 했는가?
응답 패킷이 돌아왔는가?
```

를 확인한다.

즉:

```text
timeout
```

이라는 단어만 보고 원인을 정하지 않는다.

---

# 18. 세 PCAP 비교 방법

각 참여자는 정상·오류·복구 파일을 동시에 비교한다.

다음 네 항목을 반드시 기록한다.

## A. 주소 상태

```text
normal:
bad:
recovery:
```

## B. route 판단

```text
normal .11:
normal .20:

bad .11:
bad .20:

recovery .11:
recovery .20:
```

## C. 실제 패킷

```text
ARP:
ICMP:
TCP:
HTTP:
```

## D. 실행 결과

```text
ping .11:
ping .20:
HTTP .20:
```

---

# 19. 개인 보고서 양식

## 19.1 기본 정보

```text
참여자:
Linux 계정:
실험 날짜:

정상 실험 시작:
오류 실험 시작:
복구 실험 시작:
```

---

## 19.2 사전 주소 계산

```text
[192.168.10.10/26]

Subnet mask:
Network address:
Host range:
Broadcast:

192.168.10.11이 같은 subnet인가?
이유:

192.168.10.20이 같은 subnet인가?
이유:
```

```text
[192.168.10.10/28]

Subnet mask:
Network address:
Host range:
Broadcast:

192.168.10.11이 같은 subnet인가?
이유:

192.168.10.20이 같은 subnet인가?
이유:
```

---

## 19.3 정상 /26 예상

```text
.11 route:
.11 ping:
.11 예상 패킷:

.20 route:
.20 ping:
.20 HTTP:
.20 예상 패킷:
```

---

## 19.4 정상 /26 실제 결과

```text
ip addr 실제 출력 요약:

ip route 실제 출력:

ip route get .11:

ip route get .20:

ping .11:

ping .20:

HTTP .20:

PCAP 파일:
```

---

## 19.5 정상 /26 Wireshark 근거

```text
근거 Frame 1:
프로토콜:
Source:
Destination:
관찰 필드:
이 패킷을 근거로 선택한 이유:

근거 Frame 2:
프로토콜:
Source:
Destination:
관찰 필드:
이 패킷을 근거로 선택한 이유:

근거 Frame 3:
프로토콜:
Source:
Destination:
관찰 필드:
이 패킷을 근거로 선택한 이유:
```

---

# 19.6 오류 /28 예상

```text
.11 route:
.11 ping:
.11 예상 패킷:

.20 route:
.20 ping:
.20 HTTP:
.20 예상 패킷:
```

---

# 19.7 오류 /28 실제 결과

```text
ip addr 실제 출력 요약:

ip route 실제 출력:

ip route get .11:

ip route get .20:

ping .11:

ping .20:

HTTP .20:

PCAP 파일:
```

---

# 19.8 오류 상태 Wireshark 근거

```text
.11 관련 패킷이 있었는가?

있었다면 Frame 번호:
프로토콜:
관찰 내용:


.20 관련 패킷이 있었는가?

있었다면 Frame 번호:
프로토콜:
관찰 내용:

없었다면 다음을 기록:

캡처 시작 시각:
요청 실행 시각:
route-get 결과:
ping/curl 오류:
같은 PCAP에서 .11 패킷 관찰 여부:
```

---

# 19.9 예상과 실제가 달랐던 점

```text
내 예상:

실제 결과:

둘이 달랐던 부분:

가능한 이유:

추가로 확인한 명령 또는 패킷:

확인 후 판단:
```

예상과 실제가 같았더라도:

```text
예상과 일치함.
근거:
```

처럼 증거를 적는다.

---

# 19.10 복구 결과

```text
복구 후 주소:

복구 후 route:

route-get .11:

route-get .20:

ping .11:

ping .20:

HTTP .20:

복구 PCAP:

정상화되었다고 판단한 근거:
```

---

# 19.11 확인하지 못한 사항

```text
이번 실습에서 확인하지 못한 점:

추가로 확인하려면 필요한 명령 또는 증거:
```

무리하게 원인을 확정하기보다 확인 범위를 명확하게 작성한다.

---

# 20. 강사용 해설 — 실습 후 확인

이 절은 개인 예상과 분석을 작성한 뒤 확인한다.

핵심 논리는 다음과 같다.

```text
192.168.10.10/26
        │
        ├─ .11 → 같은 subnet
        └─ .20 → 같은 subnet
```

따라서 둘 모두 직접 연결 경로로 처리된다.

반면:

```text
192.168.10.10/28
        │
        ├─ .11 → 192.168.10.0/28 내부
        │          → 직접 연결
        │
        └─ .20 → 192.168.10.0/28 외부
                   → 다른 network
                   → gateway/default/static route 필요
```

그런데 이번 구성은:

```text
gateway 없음
default route 없음
static route 없음
```

이다.

따라서 `.20`으로 보낼 경로가 없는 것이 의도된 실험 조건이다.

이 상황에서는:

```text
.11 → 정상 통신 가능
.20 → 로컬에서 경로 판단 실패 가능
```

이라는 차이가 나타난다.

---

# 21. 평가 기준

이번 주 평가는 빠르게 정답을 맞히는 것보다 **계산 → 판단 → 패킷 근거 → 복구**의 연결을 평가한다.

총 100점 기준 예시는 다음과 같다.

## 21.1 주소 계산 — 25점

확인할 내용:

```text
/26 mask 계산
/26 network/host/broadcast 계산
/28 mask 계산
/28 network/host/broadcast 계산
.11과 .20의 포함 여부 판단
```

단순히:

```text
.11 된다
.20 안 된다
```

만 쓰고 계산 근거가 없으면 충분하지 않다.

---

## 21.2 경로 판단 — 25점

확인할 내용:

```text
ip addr 확인
ip route 확인
ip route get .11
ip route get .20

직접 연결 경로와 기본 경로 구분
```

특히 오류 `/28`에서 `.20`이 왜 달라졌는지 routing table과 연결해 설명해야 한다.

---

## 21.3 패킷 증거 — 25점

확인할 내용:

```text
ARP
ICMP
TCP
HTTP

근거 Frame 번호
Source/Destination
필요한 필드
```

패킷이 없었던 경우에도:

```text
캡처가 정상임을 확인한 근거
route-get 결과
명령 오류
다른 정상 패킷의 존재
```

를 함께 제시하면 증거로 평가한다.

---

## 21.4 복구 확인 — 25점

단순히 `/26`을 다시 입력한 것만으로 완료가 아니다.

확인해야 한다.

```text
pbl-c1 = .10/26
연결 route = 192.168.10.0/26
.11 route 정상
.20 route 정상
.11 ping 정상
.20 ping 정상
HTTP 정상
복구 PCAP 존재
```

같은 시험 조건으로 재검증해야 한다.

---

# 22. 정상화 최종 확인

각 참여자가 실습을 마친 뒤 반드시 `/26` 상태로 돌려놓는다.

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w4-lab normal
```

확인:

```bash
sudo /usr/local/sbin/pbl-w4-lab check
```

A는 마지막 참여자가 끝난 뒤 직접 다시 확인한다.

```bash
sudo ip -n pbl-c1 -4 addr show dev eth0
sudo ip -n pbl-c1 route
sudo ip -n pbl-c1 route show default
```

최종 정상 기준:

```text
pbl-c1 = 192.168.10.10/26
pbl-c2 = 192.168.10.11/26
pbl-web = 192.168.10.20/26

pbl-c1 직접 연결:
192.168.10.0/26 dev eth0

default route 없음
의도하지 않은 static route 없음
```

최종 통신 확인:

```bash
sudo ip netns exec pbl-c1 \
  ping -n -c 2 192.168.10.11
```

```bash
sudo ip netns exec pbl-c1 \
  ping -n -c 2 192.168.10.20
```

```bash
sudo ip netns exec pbl-c1 \
  curl --noproxy '*' -fsS --max-time 5 \
  http://192.168.10.20:8080/
```

세 항목이 정상이어야 한다.

---

# 23. 남은 캡처 프로세스 확인

A가 실습 종료 후 확인한다.

```bash
if sudo test -d /run/lock/pbl-w4-capture; then
    echo "WARNING: 아직 캡처가 남아 있습니다."
    sudo cat /run/lock/pbl-w4-capture/owner
else
    echo "활성 캡처 없음"
fi
```

어떤 참여자의 캡처가 남아 있다면 그 참여자가 다시 로그인해:

```bash
sudo /usr/local/sbin/pbl-w4-lab capture-stop
```

을 실행한다.

불필요하게:

```bash
killall tcpdump
pkill tcpdump
```

같은 전체 프로세스 종료를 사용하지 않는다.

---

# 24. 이번 주 완료 체크리스트

각 참여자가 아래를 모두 완료해야 한다.

- [ ] `/26`에서 network·host·broadcast 범위를 직접 계산했다.
- [ ] `/28`에서 network·host·broadcast 범위를 직접 계산했다.
- [ ] `.10/28` 기준으로 `.11`은 로컬, `.20`은 다른 subnet임을 계산했다.
- [ ] 정상 실험 전에 결과를 예상했다.
- [ ] `ip addr`에서 실제 prefix를 확인했다.
- [ ] `ip route`에서 연결 경로를 확인했다.
- [ ] `ip route get .11`을 확인했다.
- [ ] `ip route get .20`을 확인했다.
- [ ] 정상 `/26` PCAP을 만들었다.
- [ ] pbl-c1만 `/28`로 변경했다.
- [ ] 오류 `/28` PCAP을 만들었다.
- [ ] `/28`에서도 `.11` 결과를 직접 확인했다.
- [ ] `/28`에서 `.20` 결과를 직접 확인했다.
- [ ] 오류 메시지를 실제 출력 기준으로 기록했다.
- [ ] ARP가 반드시 발생한다고 가정하지 않았다.
- [ ] ICMP가 반드시 발생한다고 가정하지 않았다.
- [ ] TCP SYN이 실제 발생했는지 확인했다.
- [ ] 패킷이 없었다면 캡처 실패 가능성부터 배제했다.
- [ ] `/26`으로 복구했다.
- [ ] 같은 시험으로 정상화를 재검증했다.
- [ ] 복구 PCAP을 만들었다.
- [ ] 세 PCAP의 근거 Frame 번호를 기록했다.
- [ ] 예상과 실제가 달랐다면 그 이유를 작성했다.

---

# 25. 이번 주 핵심 정리

이번 실습에서 외워야 할 문장은:

```text
같은 LAN에 꽂혀 있다고 해서
호스트가 모든 주소를 로컬로 판단하는 것은 아니다.
```

이다.

실제 판단 과정은 다음처럼 연결해서 이해한다.

```text
내 IP와 subnet mask 확인
        ↓
목적지 IP와 비교
        ↓
routing table 조회
        ↓
직접 보낼 것인지
gateway로 보낼 것인지
경로가 없는지 결정
        ↓
필요한 경우 ARP
        ↓
IP 패킷 전송
        ↓
ICMP 또는 TCP/HTTP 진행
```

따라서 장애 분석에서도:

```text
ping 실패
```

만 보는 것이 아니라:

```text
주소가 맞는가?
        ↓
mask가 맞는가?
        ↓
route는 무엇인가?
        ↓
route get은 무엇을 선택하는가?
        ↓
실제로 ARP가 발생했는가?
        ↓
ICMP/TCP가 인터페이스까지 나왔는가?
        ↓
응답은 돌아왔는가?
```

순서로 증거를 좁힐 수 있어야 한다.

4주차에서 이 차이를 확인한 뒤 5주차에는 실제로 직원망과 서버망을 분리하고 R1과 기본 게이트웨이를 추가한다. 그때는 이번 주에 “경로가 없어서 갈 수 없었던 다른 네트워크”를 **라우터를 통해 실제로 전달하는 과정**으로 확장한다.