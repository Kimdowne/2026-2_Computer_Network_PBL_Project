
# 6주차 실습 가이드
## 다중 지점 캡처와 반환 경로 분석

프로젝트: **「소규모 사내망 설계 및 Wireshark 기반 통신 검증·장애 진단」**

---

# 1. 이번 주의 핵심 질문

이번 주에는 같은 통신을 라우터 `pbl-r1`의 **직원망 측**과 **서버망 측**에서 동시에 수집한다.

관찰하려는 핵심은 다음과 같다.

```text
pbl-c1
192.168.10.10/26
    |
    | 직원망
    |
192.168.10.1/26
pbl-r1
192.168.20.1/27
    |
    | 서버망
    |
pbl-web
192.168.20.20/27
TCP 8080
```

정상 상태에서는 요청과 응답이 모두 R1을 통과한다.

이번 주에는 다음 질문에 답할 수 있어야 한다.

1. 같은 IP 패킷을 R1 양쪽 캡처에서 어떻게 찾아낼 수 있는가?
2. 라우터를 지날 때 무엇이 그대로이고 무엇이 바뀌는가?
3. Ethernet MAC 주소가 왜 양쪽 캡처에서 달라지는가?
4. IP TTL은 왜 한쪽보다 다른 쪽에서 1 작게 보이는가?
5. 서버까지 가는 길이 있어도 서버에서 돌아오는 길이 없으면 어떤 현상이 발생하는가?
6. 패킷이 보이지 않을 때 어디까지 사실로 말할 수 있는가?

이번 주에는 **패킷 하나의 Frame 번호가 다른 PCAP에서도 같을 것이라고 생각하지 않는 것**이 특히 중요하다.

---

# 2. 이번 주 완료 기준

A·B·C·D 전원이 각각 다음 활동을 직접 수행한다.

```text
선행 상태 확인
→ R1 양쪽 캡처 시작
→ 정상 ping 실행
→ 양쪽 캡처 종료
→ 두 PCAP 다운로드
→ 같은 ICMP 패킷 대응
→ MAC과 TTL 비교

→ 다시 양쪽 캡처 시작
→ 정상 HTTP 연결 실행
→ TCP 흐름 대응
→ 양쪽 캡처 종료

→ 반환 경로 장애용 캡처 시작
→ pbl-web 기본 경로 제거
→ ping 및 HTTP 연결 시도
→ 서버 경로 조회
→ 기본 경로 복구
→ 캡처 종료

→ 복구 확인용 양쪽 캡처
→ 동일 요청 재실행
→ 정상 복구 확인
```

한 명이 캡처하고 나머지 세 명이 그 파일만 분석하는 것은 개인 완료로 취급하지 않는다.

공유 환경이므로 **실제 캡처와 설정 변경은 A → B → C → D처럼 한 명씩 순서대로 수행**한다.

PCAP을 Windows PC로 내려받은 뒤 Wireshark 분석은 동시에 수행해도 된다.

---

# 3. 이번 주 정상 구성

6주차는 다음 정상 상태를 사용한다.

| 항목 | 정상 값 |
|---|---|
| 직원망 | `192.168.10.0/26` |
| pbl-c1 | `192.168.10.10/26` |
| pbl-c2 | `192.168.10.11/26` |
| 직원망 게이트웨이 | `192.168.10.1` |
| 서버망 | `192.168.20.0/27` |
| pbl-web | `192.168.20.20/27` |
| 서버망 게이트웨이 | `192.168.20.1` |
| 웹 서비스 | TCP 8080 |
| NAT | 사용하지 않음 |
| 라우터 | `pbl-r1` |
| R1 전달 기능 | IPv4 forwarding 활성화 |

이번 주에는 R1 내부의 실제 인터페이스 이름을 `eth0`, `eth1`이라고 가정하지 않는다.

주소를 기준으로 다음 두 인터페이스를 찾는다.

```text
192.168.10.1/26이 설정된 R1 인터페이스
    → 직원망 측 캡처 지점

192.168.20.1/27이 설정된 R1 인터페이스
    → 서버망 측 캡처 지점
```

Linux 네트워크 네임스페이스마다 네트워크 장치, IP 스택, 라우팅 테이블 등이 분리되므로 `pbl-r1`의 forwarding과 라우팅 상태도 해당 네임스페이스 내부에서 확인해야 한다.

---

# 4. 먼저 이해할 개념

## 4.1 라우터는 Ethernet 프레임을 그대로 반대편에 복사하지 않는다

예를 들어 클라이언트가 서버에 ICMP Echo Request를 보낸다고 하자.

직원망에서 R1이 받는 프레임은 개념적으로 다음과 같다.

```text
Ethernet
Source MAC      = pbl-c1 MAC
Destination MAC = R1 직원망 측 MAC

IPv4
Source IP       = 192.168.10.10
Destination IP  = 192.168.20.20
TTL             = T
```

R1이 라우팅한 뒤 서버망에 내보내는 프레임은 다음처럼 예상할 수 있다.

```text
Ethernet
Source MAC      = R1 서버망 측 MAC
Destination MAC = pbl-web MAC

IPv4
Source IP       = 192.168.10.10
Destination IP  = 192.168.20.20
TTL             = T - 1
```

즉, NAT가 없는 정상 라우팅에서는 **출발지·목적지 IP는 유지**되지만 Ethernet의 MAC 주소 조합은 다음 링크에 맞게 달라진다.

TTL도 라우터 하나를 통과하면서 감소한다.

실습에서는 `T=64`라고 미리 정답을 써놓지 않는다.

실제 PCAP에서 값을 확인하고:

```text
직원망 측 TTL = ?
서버망 측 TTL = ?
차이 = ?
```

형태로 기록한다.

---

# 4.2 반환 경로도 필요하다

클라이언트가 서버까지 패킷을 보낼 수 있다는 사실만으로 통신 전체가 정상이라는 뜻은 아니다.

정상 상태:

```text
pbl-c1
   ↓ 요청

pbl-r1
   ↓

pbl-web

pbl-web
   ↓ 응답

pbl-r1
   ↓

pbl-c1
```

이번 주 장애에서는 **pbl-web의 기본 경로만 제거**한다.

따라서 pbl-web에는 여전히:

```text
192.168.20.0/27
```

에 대한 직접 연결 경로가 남아 있다.

그러나:

```text
192.168.10.10
```

은 다른 네트워크이므로 서버가 클라이언트 방향으로 응답을 보내려면 정상적인 반환 경로가 필요하다.

---

# 4.3 HTTP 장애에서 특히 주의할 표현

기본 경로가 없는 서버에 클라이언트가 TCP 8080 연결을 시도하면 서버 방향으로 **TCP SYN**이 전달될 수 있다.

하지만 서버가 SYN-ACK을 클라이언트로 돌려보내지 못하면 TCP 연결이 완성되지 않는다.

그 결과 클라이언트는 보통 HTTP의:

```text
GET /
```

까지 전송하지 못한다.

따라서 장애 실험 결과를 다음처럼 쓰면 부정확할 수 있다.

```text
X: HTTP 요청이 서버에 도착했지만 응답이 없었다.
```

PCAP에서 실제로 GET을 확인하지 못했다면 더 정확한 표현은 다음과 같다.

```text
O: 클라이언트의 TCP SYN이 R1의 서버망 측 캡처 지점까지
   전달된 것을 확인했다.

O: 이후 SYN-ACK은 관찰하지 못했다.

O: TCP 연결이 완성되지 않았기 때문에
   HTTP GET은 관찰하지 못했다.
```

그리고 R1 서버망 측에서 SYN을 봤다는 사실만으로도 엄밀하게는:

> `pbl-web`의 애플리케이션까지 전달되었다.

라고 확정하지 않는다.

확실하게 말할 수 있는 범위는:

> R1을 통과해 서버망 측 인터페이스까지 전달되었다.

이다.

이런 식으로 **확인한 범위와 아직 확인하지 못한 범위**를 구분하는 것이 이번 주의 중요한 평가 항목이다.

---

# 4.4 두 PCAP의 Frame 번호는 관계가 없다

직원망 PCAP:

```text
Frame 18
```

과 서버망 PCAP:

```text
Frame 11
```

이 같은 실제 IP 패킷일 수 있다.

각 캡처 프로세스는 자신이 본 패킷에 독립적으로 Frame 번호를 붙인다.

따라서:

```text
staff Frame 18 = server Frame 18
```

처럼 Frame 번호만 맞추면 안 된다.

---

# 4.5 시각만으로 대응시키지 않는다

이번 실습에서는 같은 Linux 호스트 안에서 두 `tcpdump`를 실행하지만 **두 개의 독립된 캡처 프로세스**다.

같은 패킷이라도 캡처된 시각에 미세한 차이가 있을 수 있다.

또한 짧은 시간 동안 비슷한 패킷이 여러 번 발생할 수 있다.

따라서 패킷 대응은:

```text
프로토콜
IP 주소
ICMP identifier / sequence
TCP 포트
TCP flags
TCP sequence/acknowledgment 정보
payload 길이
시각
```

을 함께 사용한다.

**Timestamp는 보조 근거이지 단독 식별자 아니다.**

---

# 5. 셋업 담당자 A의 활동

이 단락은 A가 주도한다.

단, 준비를 마친 뒤 A도 B·C·D와 같은 개인 실습을 직접 수행한다.

---

# 5.1 5주차 상태가 남아 있는지 먼저 확인

## 실행 위치 → Linux 호스트  
## 관리자 권한 필요

네임스페이스 목록:

```bash
sudo ip netns list
```

정상 상태라면 최소한 다음 세 네임스페이스가 있어야 한다.

```text
pbl-c1
pbl-r1
pbl-web
```

`pbl-c2`도 6주차 정상 구성에 포함된다.

주소 확인:

```bash
sudo ip -n pbl-c1 -br addr
sudo ip -n pbl-c2 -br addr
sudo ip -n pbl-r1 -br addr
sudo ip -n pbl-web -br addr
```

확인할 주소:

```text
pbl-c1   192.168.10.10/26
pbl-c2   192.168.10.11/26

pbl-r1   192.168.10.1/26
pbl-r1   192.168.20.1/27

pbl-web  192.168.20.20/27
```

주소가 다르거나 네임스페이스가 없다면 **6주차 실습을 진행하지 않는다.**

먼저 5주차 정상 구성을 복구한다.

6주차 준비 과정에서 누락된 전체 네트워크를 임의의 다른 주소로 새로 만들지 않는다.

---

# 5.2 기본 경로 확인

## pbl-c1

```bash
sudo ip -n pbl-c1 route
```

정상 상태에는 다음 의미의 경로가 있어야 한다.

```text
default via 192.168.10.1
```

정확한 `dev` 이름은 환경의 실제 값을 따른다.

## pbl-web

```bash
sudo ip -n pbl-web route
```

정상 상태에는:

```text
default via 192.168.20.1
```

가 있어야 한다.

Linux의 `ip route`는 커널 라우팅 테이블을 조회·추가·삭제하는 명령이며 `default via <gateway>` 형태로 기본 경로를 지정할 수 있다.

---

# 5.3 R1 forwarding 확인

## 실행 위치 → Linux 호스트에서 pbl-r1 내부 조회

```bash
sudo ip netns exec pbl-r1 sysctl -n net.ipv4.ip_forward
```

정상 예상:

```text
1
```

`0`이면 이번 주 정상 상태가 아니다.

이번 프로젝트에서는 forwarding을 **호스트 전체가 아니라 `pbl-r1` 네임스페이스 내부에서 설정**하는 것이 원칙이다.

5주차 정상 설정을 복구한 뒤 계속한다.

---

# 5.4 R1 양쪽 실제 인터페이스 이름 찾기

## 실행 위치 → Linux 호스트

```bash
sudo ip -n pbl-r1 -o -4 addr show
```

예를 들어 다음처럼 보일 수 있다.

```text
2: r1-staff ... inet 192.168.10.1/26 ...
3: r1-server ... inet 192.168.20.1/27 ...
```

단, `r1-staff`, `r1-server`라는 이름 자체는 예시다.

실제 이름을 확인한다.

자동 조회:

```bash
STAFF_IF="$(
  sudo ip -n pbl-r1 -o -4 addr show |
  awk '$4=="192.168.10.1/26" {print $2; exit}'
)"

SERVER_IF="$(
  sudo ip -n pbl-r1 -o -4 addr show |
  awk '$4=="192.168.20.1/27" {print $2; exit}'
)"

printf 'R1 직원망 측: %s\n' "$STAFF_IF"
printf 'R1 서버망 측: %s\n' "$SERVER_IF"
```

두 값 중 하나가 비어 있으면 진행하지 않는다.

---

# 5.5 웹 서비스 상태 확인

## 실행 위치 → Linux 호스트에서 pbl-web 내부 조회

```bash
sudo ip netns exec pbl-web ss -lntp
```

TCP `8080`이 LISTEN 상태인지 찾는다.

예상되는 의미:

```text
LISTEN ... 192.168.20.20:8080
```

또는 구현에 따라 적절한 8080 LISTEN 항목이 나타난다.

8080이 열려 있지 않다면 이번 주에 임의의 다른 웹 서버를 추가하지 말고 **5주차에서 사용하던 정상 웹 서비스를 복구**한다.

---

# 5.6 정상 통신 사전 확인

## ping

```bash
sudo ip netns exec pbl-c1 ping -c 2 -W 1 192.168.20.20
```

정상 환경이라면 응답이 예상되지만, 실제 결과를 확인한다.

## HTTP

```bash
sudo ip netns exec pbl-c1 \
  curl --noproxy '*' -v --max-time 5 \
  http://192.168.20.20:8080/
```

이 단계에서 실패한다면 장애 실험을 시작하지 않는다.

정상 상태를 먼저 복구한다.

---

# 6. 6주차 실습 디렉터리 준비

## 실행 위치 → Linux 호스트  
## 관리자 권한 필요

```bash
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week6
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week6/captures
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week6/logs
```

확인:

```bash
ls -ld /srv/pbl/week6
ls -ld /srv/pbl/week6/captures
ls -ld /srv/pbl/week6/logs
```

---

# 7. 양쪽 캡처·장애 실험용 제한 스크립트 설치

이 스크립트의 목적은 다음과 같다.

- R1의 실제 인터페이스 이름을 주소로 찾는다.
- 직원망 측과 서버망 측 `tcpdump`를 따로 실행한다.
- 두 캡처 프로세스의 PID를 따로 보관한다.
- 두 출력 PCAP 이름을 따로 보관한다.
- 다른 참여자의 캡처와 겹치지 않게 한다.
- `killall`, `pkill tcpdump` 등을 사용하지 않는다.
- 자신이 시작한 PID가 실제 예상한 `tcpdump`인지 확인한 뒤 종료한다.
- pbl-web의 기본 경로 하나만 통제된 방식으로 제거·복구한다.

`tcpdump -w`는 캡처 파일을 기록하며 `-U`를 사용하면 수집된 패킷이 파일에 보다 즉시 기록되도록 할 수 있다.

## 파일 생성

### 실행 위치 → Linux 호스트  
### 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w6-lab
```

아래 전체 내용을 저장한다.

```bash
#!/usr/bin/env bash
set -euo pipefail

ACTION="${1:-}"
ARG="${2:-}"

BASE="/srv/pbl/week6"
CAPDIR="${BASE}/captures"
LOGDIR="${BASE}/logs"

LOCKDIR="/run/lock/pbl-w6"
STATEDIR="/run/pbl-w6"

REAL_USER="${SUDO_USER:-}"

if [[ ${EUID} -ne 0 ]]; then
    echo "ERROR: sudo 또는 root 권한으로 실행해야 합니다." >&2
    exit 1
fi

case "${REAL_USER}" in
    pbl-a|pbl-b|pbl-c|pbl-d)
        ;;
    *)
        echo "ERROR: 허용된 실습 계정이 아닙니다: ${REAL_USER:-unknown}" >&2
        exit 1
        ;;
esac

mkdir -p "${CAPDIR}" "${LOGDIR}" "${STATEDIR}"

get_if_by_cidr() {
    local ns="$1"
    local cidr="$2"

    ip -n "${ns}" -o -4 addr show |
        awk -v target="${cidr}" '$4 == target {print $2; exit}'
}

ns_exists() {
    local ns="$1"

    ip netns list |
        awk '{print $1}' |
        grep -Fxq "${ns}"
}

require_namespaces() {
    local ns

    for ns in pbl-c1 pbl-c2 pbl-r1 pbl-web; do
        if ! ns_exists "${ns}"; then
            echo "ERROR: 필요한 네임스페이스가 없습니다: ${ns}" >&2
            exit 1
        fi
    done
}

discover_interfaces() {
    STAFF_IF="$(get_if_by_cidr pbl-r1 192.168.10.1/26)"
    SERVER_IF="$(get_if_by_cidr pbl-r1 192.168.20.1/27)"
    C1_IF="$(get_if_by_cidr pbl-c1 192.168.10.10/26)"
    WEB_IF="$(get_if_by_cidr pbl-web 192.168.20.20/27)"

    if [[ -z "${STAFF_IF}" ]]; then
        echo "ERROR: pbl-r1의 192.168.10.1/26 인터페이스를 찾지 못했습니다." >&2
        exit 1
    fi

    if [[ -z "${SERVER_IF}" ]]; then
        echo "ERROR: pbl-r1의 192.168.20.1/27 인터페이스를 찾지 못했습니다." >&2
        exit 1
    fi

    if [[ -z "${C1_IF}" ]]; then
        echo "ERROR: pbl-c1의 192.168.10.10/26 인터페이스를 찾지 못했습니다." >&2
        exit 1
    fi

    if [[ -z "${WEB_IF}" ]]; then
        echo "ERROR: pbl-web의 192.168.20.20/27 인터페이스를 찾지 못했습니다." >&2
        exit 1
    fi
}

require_owner() {
    if [[ ! -d "${LOCKDIR}" ]]; then
        echo "ERROR: 현재 활성 캡처가 없습니다." >&2
        exit 1
    fi

    owner="$(cat "${LOCKDIR}/owner" 2>/dev/null || true)"

    if [[ "${owner}" != "${REAL_USER}" ]]; then
        echo "ERROR: 현재 실습은 ${owner:-unknown} 사용자의 것입니다." >&2
        exit 1
    fi
}

show_route_status() {
    echo "=== pbl-c1 route ==="
    ip -n pbl-c1 route

    echo
    echo "=== pbl-web route ==="
    ip -n pbl-web route

    echo
    echo "=== pbl-web -> 192.168.10.10 route lookup ==="

    set +e
    ip -n pbl-web route get 192.168.10.10
    rc=$?
    set -e

    echo "route-get 종료 코드: ${rc}"
}

stop_capture_pid() {
    local pid="$1"
    local ifname="$2"

    if [[ ! "${pid}" =~ ^[0-9]+$ ]]; then
        echo "ERROR: 올바르지 않은 PID 값: ${pid}" >&2
        return 1
    fi

    if ! kill -0 "${pid}" 2>/dev/null; then
        echo "INFO: PID ${pid}는 이미 종료되어 있습니다."
        return 0
    fi

    cmdline="$(
        tr '\0' ' ' < "/proc/${pid}/cmdline" 2>/dev/null || true
    )"

    if [[ "${cmdline}" != *"tcpdump"* ||
          "${cmdline}" != *"${ifname}"* ]]; then

        echo "ERROR: PID ${pid}가 예상한 tcpdump가 아닙니다." >&2
        echo "명령행: ${cmdline}" >&2
        return 1
    fi

    kill -INT "${pid}" 2>/dev/null || true

    for _ in {1..30}; do
        kill -0 "${pid}" 2>/dev/null || break
        sleep 0.1
    done

    if kill -0 "${pid}" 2>/dev/null; then
        echo "WARNING: PID ${pid}가 SIGINT 후에도 실행 중입니다." >&2
        echo "SIGTERM을 이 PID에만 보냅니다." >&2

        kill -TERM "${pid}" 2>/dev/null || true

        for _ in {1..20}; do
            kill -0 "${pid}" 2>/dev/null || break
            sleep 0.1
        done
    fi

    if kill -0 "${pid}" 2>/dev/null; then
        echo "ERROR: PID ${pid}를 안전하게 종료하지 못했습니다." >&2
        return 1
    fi
}

require_namespaces
discover_interfaces

case "${ACTION}" in

check)
    echo "=== 실제 인터페이스 ==="
    echo "R1 직원망 측 : ${STAFF_IF}"
    echo "R1 서버망 측 : ${SERVER_IF}"
    echo "pbl-c1       : ${C1_IF}"
    echo "pbl-web      : ${WEB_IF}"

    echo
    echo "=== 주소 ==="
    ip -n pbl-c1 -br addr
    ip -n pbl-c2 -br addr
    ip -n pbl-r1 -br addr
    ip -n pbl-web -br addr

    echo
    echo "=== R1 링크/MAC ==="
    ip -n pbl-r1 link show "${STAFF_IF}"
    ip -n pbl-r1 link show "${SERVER_IF}"

    echo
    echo "=== pbl-c1 링크/MAC ==="
    ip -n pbl-c1 link show "${C1_IF}"

    echo
    echo "=== pbl-web 링크/MAC ==="
    ip -n pbl-web link show "${WEB_IF}"

    echo
    echo "=== R1 forwarding ==="

    forwarding="$(
        ip netns exec pbl-r1 \
        sysctl -n net.ipv4.ip_forward
    )"

    echo "${forwarding}"

    if [[ "${forwarding}" != "1" ]]; then
        echo "ERROR: pbl-r1 forwarding이 활성화되지 않았습니다." >&2
        exit 1
    fi

    echo
    show_route_status

    echo
    echo "=== 기본 경로 검사 ==="

    c1_default="$(ip -n pbl-c1 route show default)"
    web_default="$(ip -n pbl-web route show default)"

    if [[ "${c1_default}" != *"via 192.168.10.1"* ]]; then
        echo "ERROR: pbl-c1 기본 경로가 예상과 다릅니다." >&2
        exit 1
    fi

    if [[ "${web_default}" != *"via 192.168.20.1"* ]]; then
        echo "ERROR: pbl-web 기본 경로가 예상과 다릅니다." >&2
        exit 1
    fi

    echo "기본 경로: OK"

    echo
    echo "=== TCP 8080 LISTEN 확인 ==="

    if ! ip netns exec pbl-web \
        ss -lnt |
        grep -q ':8080'; then

        echo "ERROR: pbl-web에서 TCP 8080 LISTEN을 찾지 못했습니다." >&2
        exit 1
    fi

    echo "TCP 8080: LISTEN 확인"

    echo
    echo "=== 활성 실습 확인 ==="

    if [[ -d "${LOCKDIR}" ]]; then
        echo -n "현재 사용자: "
        cat "${LOCKDIR}/owner" 2>/dev/null || echo "unknown"

        echo -n "현재 라벨: "
        cat "${LOCKDIR}/label" 2>/dev/null || echo "unknown"
    else
        echo "활성 캡처 없음"
    fi
    ;;

capture-start)
    case "${ARG}" in
        normal-ping|normal-http|broken|restored)
            ;;
        *)
            echo "ERROR: 캡처 라벨은 다음 중 하나여야 합니다." >&2
            echo "normal-ping | normal-http | broken | restored" >&2
            exit 1
            ;;
    esac

    if ! mkdir "${LOCKDIR}" 2>/dev/null; then
        echo "ERROR: 다른 참여자의 실습이 이미 진행 중입니다." >&2
        echo -n "현재 사용자: " >&2
        cat "${LOCKDIR}/owner" 2>/dev/null || echo "unknown" >&2
        exit 1
    fi

    echo "${REAL_USER}" > "${LOCKDIR}/owner"
    echo "${ARG}" > "${LOCKDIR}/label"

    stamp="$(date +%Y%m%d-%H%M%S)"

    staff_file="${CAPDIR}/${REAL_USER}-${ARG}-${stamp}-staff.pcap"
    server_file="${CAPDIR}/${REAL_USER}-${ARG}-${stamp}-server.pcap"

    staff_log="${LOGDIR}/${REAL_USER}-${ARG}-${stamp}-staff.log"
    server_log="${LOGDIR}/${REAL_USER}-${ARG}-${stamp}-server.log"

    echo "${STAFF_IF}" > "${LOCKDIR}/if.staff"
    echo "${SERVER_IF}" > "${LOCKDIR}/if.server"

    echo "${staff_file}" > "${LOCKDIR}/file.staff"
    echo "${server_file}" > "${LOCKDIR}/file.server"

    nohup ip netns exec pbl-r1 \
        tcpdump \
        -i "${STAFF_IF}" \
        -n \
        -U \
        -s 0 \
        -w "${staff_file}" \
        '(icmp or tcp port 8080)' \
        >"${staff_log}" 2>&1 < /dev/null &

    staff_pid=$!

    echo "${staff_pid}" > "${LOCKDIR}/pid.staff"

    nohup ip netns exec pbl-r1 \
        tcpdump \
        -i "${SERVER_IF}" \
        -n \
        -U \
        -s 0 \
        -w "${server_file}" \
        '(icmp or tcp port 8080)' \
        >"${server_log}" 2>&1 < /dev/null &

    server_pid=$!

    echo "${server_pid}" > "${LOCKDIR}/pid.server"

    sleep 0.7

    failed=0

    if ! kill -0 "${staff_pid}" 2>/dev/null; then
        echo "ERROR: 직원망 측 tcpdump 시작 실패" >&2
        cat "${staff_log}" >&2 || true
        failed=1
    fi

    if ! kill -0 "${server_pid}" 2>/dev/null; then
        echo "ERROR: 서버망 측 tcpdump 시작 실패" >&2
        cat "${server_log}" >&2 || true
        failed=1
    fi

    if [[ "${failed}" -ne 0 ]]; then
        kill -INT "${staff_pid}" 2>/dev/null || true
        kill -INT "${server_pid}" 2>/dev/null || true
        rm -rf "${LOCKDIR}"
        exit 1
    fi

    echo "CAPTURE STARTED"
    echo "사용자: ${REAL_USER}"
    echo "라벨: ${ARG}"
    echo "시작 시각: $(date --iso-8601=seconds)"
    echo
    echo "직원망 측:"
    echo "  인터페이스: ${STAFF_IF}"
    echo "  PID: ${staff_pid}"
    echo "  파일: ${staff_file}"
    echo
    echo "서버망 측:"
    echo "  인터페이스: ${SERVER_IF}"
    echo "  PID: ${server_pid}"
    echo "  파일: ${server_file}"
    echo
    echo "이 메시지를 확인한 뒤 트래픽을 발생시키십시오."
    ;;

ping)
    require_owner

    echo "PING START: $(date --iso-8601=seconds)"
    echo

    set +e

    ip netns exec pbl-c1 \
        ping -c 3 -W 1 192.168.20.20

    rc=$?

    set -e

    echo
    echo "PING 종료 코드: ${rc}"
    ;;

http)
    require_owner

    echo "HTTP START: $(date --iso-8601=seconds)"
    echo

    set +e

    ip netns exec pbl-c1 \
        curl \
        --noproxy '*' \
        -v \
        --connect-timeout 3 \
        --max-time 5 \
        http://192.168.20.20:8080/

    rc=$?

    set -e

    echo
    echo "CURL 종료 코드: ${rc}"
    ;;

route-check)
    require_owner

    show_route_status
    ;;

fault-remove)
    require_owner

    label="$(cat "${LOCKDIR}/label")"

    if [[ "${label}" != "broken" ]]; then
        echo "ERROR: fault-remove은 broken 캡처에서만 실행합니다." >&2
        exit 1
    fi

    if [[ -f "${STATEDIR}/fault-owner" ]]; then
        echo "ERROR: 반환 경로 장애 상태가 이미 기록되어 있습니다." >&2
        echo -n "장애 주입 사용자: " >&2
        cat "${STATEDIR}/fault-owner" >&2
        exit 1
    fi

    mapfile -t defaults < <(
        ip -n pbl-web route show default
    )

    if [[ "${#defaults[@]}" -ne 1 ]]; then
        echo "ERROR: pbl-web 기본 경로가 정확히 하나가 아닙니다." >&2
        printf '%s\n' "${defaults[@]}" >&2
        exit 1
    fi

    if [[ "${defaults[0]}" != *"via 192.168.20.1"* ]]; then
        echo "ERROR: 정상 기본 경로가 아닙니다." >&2
        echo "${defaults[0]}" >&2
        exit 1
    fi

    printf '%s\n' "${defaults[0]}" \
        > "${STATEDIR}/saved-web-default"

    ip -n pbl-web \
        route del default \
        via 192.168.20.1 \
        dev "${WEB_IF}"

    echo "${REAL_USER}" > "${STATEDIR}/fault-owner"

    echo "FAULT INJECTED"
    echo "변경 대상: pbl-web"
    echo "제거 항목: default via 192.168.20.1"
    echo "시각: $(date --iso-8601=seconds)"

    echo
    echo "=== 변경 후 pbl-web route ==="
    ip -n pbl-web route
    ;;

fault-restore)
    require_owner

    if [[ ! -f "${STATEDIR}/fault-owner" ]]; then
        echo "ERROR: 기록된 반환 경로 장애가 없습니다." >&2
        exit 1
    fi

    fault_owner="$(
        cat "${STATEDIR}/fault-owner"
    )"

    if [[ "${fault_owner}" != "${REAL_USER}" ]]; then
        echo "ERROR: 장애 주입 사용자는 ${fault_owner}입니다." >&2
        exit 1
    fi

    ip -n pbl-web \
        route replace default \
        via 192.168.20.1 \
        dev "${WEB_IF}"

    echo
    echo "=== 복구 후 pbl-web route ==="
    ip -n pbl-web route

    echo
    echo "=== 192.168.10.10 경로 조회 ==="

    if ! ip -n pbl-web \
        route get 192.168.10.10; then

        echo "ERROR: 기본 경로 복구 후에도 경로 조회가 실패합니다." >&2
        exit 1
    fi

    rm -f "${STATEDIR}/fault-owner"
    rm -f "${STATEDIR}/saved-web-default"

    echo
    echo "FAULT RESTORED"
    echo "시각: $(date --iso-8601=seconds)"
    ;;

capture-stop)
    require_owner

    if [[ -f "${STATEDIR}/fault-owner" ]]; then
        echo "ERROR: pbl-web 기본 경로가 아직 장애 상태입니다." >&2
        echo "먼저 다음을 실행하십시오:" >&2
        echo "sudo /usr/local/sbin/pbl-w6-lab fault-restore" >&2
        exit 1
    fi

    staff_pid="$(cat "${LOCKDIR}/pid.staff")"
    server_pid="$(cat "${LOCKDIR}/pid.server")"

    staff_if="$(cat "${LOCKDIR}/if.staff")"
    server_if="$(cat "${LOCKDIR}/if.server")"

    staff_file="$(cat "${LOCKDIR}/file.staff")"
    server_file="$(cat "${LOCKDIR}/file.server")"

    stop_capture_pid "${staff_pid}" "${staff_if}"
    stop_capture_pid "${server_pid}" "${server_if}"

    for outfile in "${staff_file}" "${server_file}"; do
        if [[ "${outfile}" != "${CAPDIR}/"*.pcap ]]; then
            echo "ERROR: 예상하지 못한 캡처 파일 경로입니다." >&2
            exit 1
        fi

        if [[ -f "${outfile}" ]]; then
            chgrp pbl "${outfile}"
            chmod 0640 "${outfile}"
        fi
    done

    rm -rf "${LOCKDIR}"

    echo "CAPTURE STOPPED"
    echo "종료 시각: $(date --iso-8601=seconds)"
    echo
    echo "직원망 측 PCAP:"
    echo "${staff_file}"
    echo
    echo "서버망 측 PCAP:"
    echo "${server_file}"
    ;;

*)
    echo "사용법:"
    echo "  sudo /usr/local/sbin/pbl-w6-lab check"
    echo
    echo "  sudo /usr/local/sbin/pbl-w6-lab capture-start normal-ping"
    echo "  sudo /usr/local/sbin/pbl-w6-lab capture-start normal-http"
    echo "  sudo /usr/local/sbin/pbl-w6-lab capture-start broken"
    echo "  sudo /usr/local/sbin/pbl-w6-lab capture-start restored"
    echo
    echo "  sudo /usr/local/sbin/pbl-w6-lab ping"
    echo "  sudo /usr/local/sbin/pbl-w6-lab http"
    echo "  sudo /usr/local/sbin/pbl-w6-lab route-check"
    echo "  sudo /usr/local/sbin/pbl-w6-lab fault-remove"
    echo "  sudo /usr/local/sbin/pbl-w6-lab fault-restore"
    echo "  sudo /usr/local/sbin/pbl-w6-lab capture-stop"
    exit 2
    ;;
esac
```

저장 후:

```bash
sudo chown root:root /usr/local/sbin/pbl-w6-lab
sudo chmod 0755 /usr/local/sbin/pbl-w6-lab
```

---

# 8. sudo 권한 제한

## 실행 위치 → Linux 호스트  
## 관리자 권한 필요

```bash
sudo visudo -f /etc/sudoers.d/pbl-week6
```

다음을 저장한다.

```text
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab check
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab capture-start normal-ping
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab capture-start normal-http
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab capture-start broken
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab capture-start restored
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab ping
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab http
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab route-check
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab fault-remove
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab fault-restore
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w6-lab capture-stop
```

권한과 문법 확인:

```bash
sudo chmod 0440 /etc/sudoers.d/pbl-week6
sudo visudo -cf /etc/sudoers.d/pbl-week6
```

오류가 나오면 수정한 뒤 실습자에게 넘긴다.

---

# 9. A의 최종 셋업 점검

실습자 계정 중 하나로 테스트하거나 A 본인 계정에서 다음을 실행한다.

```bash
sudo /usr/local/sbin/pbl-w6-lab check
```

확인할 내용:

```text
pbl-c1   = 192.168.10.10/26
pbl-c2   = 192.168.10.11/26
pbl-r1   = 192.168.10.1/26
pbl-r1   = 192.168.20.1/27
pbl-web  = 192.168.20.20/27

pbl-c1 default via 192.168.10.1
pbl-web default via 192.168.20.1

pbl-r1 ip_forward = 1

TCP 8080 LISTEN
활성 캡처 없음
```

실제 MAC 주소도 이 출력에서 확인할 수 있다.

MAC 주소는 예를 들어:

```text
7a:3c:91:2f:8b:10
```

같은 형태지만 **이 문서의 예시 값을 실제 값으로 복사하지 않는다.**

실제 `link/ether` 값을 기록한다.

---

# 10. 실습 참여자 A·B·C·D의 활동

아래는 네 명 모두 개인별로 수행한다.

한 참여자가 `capture-start`를 실행하면 `capture-stop`까지 마친 후 다음 참여자가 시작한다.

---

# 11. 실험 1 — 정상 ping을 R1 양쪽에서 동시에 수집

## 11.1 정상 상태 확인

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w6-lab check
```

오류가 있으면 실험을 시작하지 않는다.

---

# 11.2 먼저 예상 작성

명령 실행 전에 개인 기록에 쓴다.

예:

```text
예상:

pbl-c1에서 pbl-web으로 ICMP Echo Request가 전송될 것이다.

직원망 측과 서버망 측 캡처 모두에서 같은 요청을 찾을 수 있을 것으로 예상한다.

IP 출발지와 목적지는 같게 유지될 것으로 예상한다.

서버망 측에서 요청 TTL이 직원망 측보다 1 작을 것으로 예상한다.

Ethernet Source/Destination MAC은 양쪽에서 다를 것으로 예상한다.
```

이 내용은 **예상**이며 측정 결과가 아니다.

---

# 11.3 양쪽 캡처 시작

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-start normal-ping
```

출력 예:

```text
CAPTURE STARTED
사용자: pbl-a
라벨: normal-ping

직원망 측:
  인터페이스: ...
  PID: ...
  파일: ...-staff.pcap

서버망 측:
  인터페이스: ...
  PID: ...
  파일: ...-server.pcap
```

반드시 두 PID와 두 파일이 모두 출력된 것을 확인한다.

---

# 11.4 ping 실행

```bash
sudo /usr/local/sbin/pbl-w6-lab ping
```

스크립트 내부에서는:

```bash
ip netns exec pbl-c1 ping -c 3 -W 1 192.168.20.20
```

이 실행된다.

정상 상태라면 Echo Reply가 나타날 것으로 예상되지만, 실제 출력값을 기록한다.

---

# 11.5 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-stop
```

두 파일명을 모두 기록한다.

예:

```text
pbl-a-normal-ping-...-staff.pcap
pbl-a-normal-ping-...-server.pcap
```

---

# 12. Windows PC로 두 PCAP 내려받기

먼저 SSH 세션에서 실제 파일명을 복사해 둔다.

## 실행 위치 → 개인 Windows PowerShell

폴더 생성:

```powershell
New-Item -ItemType Directory -Force "$HOME\pbl-week6"
```

예를 들어 A라면:

```powershell
scp pbl-a@<SERVER_MANAGEMENT_IP>:/srv/pbl/week6/captures/<STAFF_FILE>.pcap "$HOME\pbl-week6\"
```

```powershell
scp pbl-a@<SERVER_MANAGEMENT_IP>:/srv/pbl/week6/captures/<SERVER_FILE>.pcap "$HOME\pbl-week6\"
```

`<STAFF_FILE>`과 `<SERVER_FILE>`에는 실제 `capture-stop` 출력값을 사용한다.

확인:

```powershell
Get-ChildItem "$HOME\pbl-week6"
```

---

# 13. 정상 ping PCAP 대응 분석

두 PCAP을 Wireshark의 별도 창에 열어 나란히 비교하면 편하다.

파일 이름에:

```text
-staff.pcap
-server.pcap
```

이 명확히 구분되어 있는지 먼저 확인한다.

---

# 13.1 ICMP만 표시

두 창 모두 Display Filter:

```text
icmp
```

또는 Echo Request만 보려면:

```text
icmp.type == 8
```

---

# 13.2 첫 번째 Echo Request 선택

직원망 측 PCAP에서:

```text
Source      192.168.10.10
Destination 192.168.20.20
```

인 Echo Request 하나를 고른다.

다음 값을 기록한다.

```text
Frame 번호:
Capture 시각:
Source IP:
Destination IP:
ICMP Identifier:
ICMP Sequence Number:
Ethernet Source:
Ethernet Destination:
IP TTL:
```

---

# 13.3 서버망 측에서 같은 요청 찾기

**같은 Frame 번호를 찾지 않는다.**

다음 조합으로 찾는다.

```text
Source IP          = 192.168.10.10
Destination IP     = 192.168.20.20
ICMP Identifier    = 직원망 PCAP과 동일
ICMP Sequence      = 직원망 PCAP과 동일
Capture 시각       = 매우 가까운 값
```

ICMP Identifier와 Sequence는 Packet Details의 ICMP 부분에서 확인한다.

---

# 13.4 대응표 작성

실제 값을 채운다.

| 항목 | 직원망 측 PCAP | 서버망 측 PCAP |
|---|---|---|
| Frame 번호 |  |  |
| 시각 |  |  |
| Source IP |  |  |
| Destination IP |  |  |
| ICMP Identifier |  |  |
| ICMP Sequence |  |  |
| Ethernet Source MAC |  |  |
| Ethernet Destination MAC |  |  |
| IP TTL |  |  |

그 아래에 판단을 적는다.

```text
같은 패킷이라고 판단한 근거:
1.
2.
3.
```

---

# 14. MAC 주소 비교

Packet Details에서:

```text
Ethernet II
```

를 펼친다.

확인:

```text
Source
Destination
```

정상 구성에서 예상되는 관계는 다음과 같다.

## 직원망 측 — 요청

```text
Source MAC
    pbl-c1

Destination MAC
    R1 직원망 측
```

## 서버망 측 — 같은 요청

```text
Source MAC
    R1 서버망 측

Destination MAC
    pbl-web
```

즉, IP 패킷이 라우팅되어 다른 LAN으로 나갈 때 새 Ethernet 프레임으로 전달되므로 MAC 주소 조합이 달라진다.

---

# 15. TTL 비교

Packet Details:

```text
Internet Protocol Version 4
→ Time to Live
```

두 캡처의 같은 Echo Request 값을 비교한다.

작성:

```text
직원망 측 TTL = ______

서버망 측 TTL = ______

차이 = ______
```

정상 라우팅이라면 서버망 측에서 1 감소한 값을 기대할 수 있다.

하지만 실제 PCAP을 확인하기 전에:

```text
64 → 63
```

처럼 결과를 미리 측정값으로 쓰지 않는다.

---

# 16. Echo Reply도 반대 방향으로 비교

서버 → 클라이언트 Echo Reply를 하나 선택한다.

이번에는 서버망 측에서 먼저 들어오는 응답을 찾은 뒤 직원망 측에서 대응시킨다.

예상 관계:

```text
서버망 측
pbl-web MAC
   →
R1 서버망 MAC

IP:
192.168.20.20
   →
192.168.10.10
```

R1을 지난 뒤 직원망 측:

```text
R1 직원망 MAC
   →
pbl-c1 MAC

IP:
192.168.20.20
   →
192.168.10.10
```

TTL 역시 R1 통과 전후를 비교한다.

---

# 17. 실험 2 — 정상 HTTP 연결 양쪽 캡처

## 17.1 캡처 시작

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-start normal-http
```

---

# 17.2 HTTP 요청 실행

```bash
sudo /usr/local/sbin/pbl-w6-lab http
```

정상 상태라면 `curl -v`에서 TCP 연결과 HTTP 응답이 관찰될 것으로 예상한다.

실제 상태 코드와 출력은 직접 기록한다.

---

# 17.3 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-stop
```

두 PCAP을 Windows PC로 내려받는다.

---

# 18. TCP 흐름 찾기

Wireshark Display Filter:

```text
tcp.port == 8080
```

TCP 연결 앞부분을 찾는다.

일반적으로 다음 유형의 패킷을 찾을 수 있다.

```text
SYN
SYN, ACK
ACK
...
HTTP 요청
HTTP 응답
```

실제로 존재하는 패킷을 기준으로 분석한다.

---

# 18.1 TCP SYN 대응시키기

직원망 PCAP에서 클라이언트 → 서버 SYN을 선택한다.

기록:

```text
Source IP:
Destination IP:

Source TCP Port:
Destination TCP Port:

SYN:
ACK flag:

Sequence Number:
TCP payload length:

IP TTL:
시각:
```

Destination Port는 정상 구성이라면 `8080`일 것으로 예상한다.

Source Port는 운영체제가 선택하는 임시 포트이므로 고정값으로 외우지 않는다.

서버망 PCAP에서 다음 조건으로 대응되는 SYN을 찾는다.

```text
Source IP 동일
Destination IP 동일

Source TCP Port 동일
Destination TCP Port 동일

SYN 플래그 동일

TCP 데이터 길이 동일

시각이 가까움
```

그리고 TTL과 MAC 주소를 비교한다.

---

# 18.2 4-tuple을 이용한다

하나의 TCP 연결을 구분할 때 중요한 네 값은:

```text
클라이언트 IP
클라이언트 TCP 포트
서버 IP
서버 TCP 포트
```

이다.

예:

```text
192.168.10.10 : 51842
       ↕
192.168.20.20 : 8080
```

여기서 `51842`는 예시일 뿐이다.

본인의 실제 값을 사용한다.

---

# 18.3 TCP Stream 번호도 PCAP 간 절대 식별자가 아니다

Wireshark에:

```text
tcp.stream
```

번호가 표시될 수 있다.

예:

```text
staff PCAP  tcp.stream == 0
server PCAP tcp.stream == 1
```

이어도 같은 실제 연결일 수 있다.

`tcp.stream` 번호도 각 캡처 파일 안에서 Wireshark가 관리하는 값이므로 **두 PCAP에서 같은 번호여야 한다고 생각하지 않는다.**

---

# 19. TCP 상대 시퀀스 번호 주의점

Wireshark는 기본적으로 TCP Sequence/Acknowledgment 값을 사람이 읽기 쉽게 **상대 시퀀스 번호**로 표시할 수 있다.

즉, 실제 32비트 원래 번호 대신 해당 캡처에서 관찰한 연결을 기준으로 `0`, `1`, `...` 같은 값으로 다시 표현할 수 있다.

따라서 서로 다른 두 PCAP이 다음 조건이면:

- 한 파일은 SYN부터 캡처됨
- 다른 파일은 연결 중간부터 캡처됨
- 분석 설정이 다름

상대 Sequence 표시가 서로 다를 수 있다.

따라서:

```text
상대 Sequence가 다르다
→ 다른 패킷이다
```

라고 즉시 판단하지 않는다.

우선:

```text
IP
포트
TCP flag
payload 길이
시각
방향
```

을 함께 비교한다.

필요할 때만 Wireshark의:

```text
Edit
→ Preferences
→ Protocols
→ TCP
```

에서 Relative sequence numbers 설정을 확인할 수 있다.

입문 단계에서는 설정을 무조건 바꿀 필요가 없다.

---

# 20. 체크섬 경고와 오프로딩 주의

Linux에서 캡처할 때 Wireshark가 TCP 또는 IP 체크섬을:

```text
incorrect
unverified
partial
```

등으로 표시하는 경우가 있을 수 있다.

이 표시만 보고:

```text
패킷이 실제 네트워크에서 손상되었다.
```

라고 결론내리면 안 된다.

운영체제나 네트워크 장치가 checksum 계산 일부를 나중 단계에서 처리하는 **checksum offloading** 때문에 송신 측 캡처에서 아직 완성되지 않은 값을 Wireshark가 볼 수 있기 때문이다. Wireshark 공식 문서도 오프로딩 때문에 송신 패킷의 체크섬이 잘못된 것처럼 보일 수 있다고 설명한다. 최신 Wireshark는 일부 partial checksum도 별도로 인식한다.

이번 주 원칙:

```text
체크섬 경고를 봄
        ↓
실제 손상이라고 즉시 판단하지 않음
        ↓
패킷 방향과 캡처 위치 확인
        ↓
상대편 응답이 있었는지 확인
        ↓
다른 근거와 함께 판단
```

**이번 입문 실습에서는 오프로딩을 자동으로 끄지 않는다.**

환경 설정 자체를 변경하는 대신 먼저 관찰 결과를 해석하는 연습을 한다.

---

# 21. 실험 3 — pbl-web 반환 경로 제거

이제 한 번에 하나의 원인만 변경한다.

변경 대상:

```text
pbl-web의 default route
```

변경하지 않는 것:

```text
pbl-c1 기본 경로
R1 라우팅
R1 forwarding
pbl-web IP 주소
웹 서비스
호스트 관리망
호스트 기본 경로
호스트 실제 NIC
```

---

# 21.1 장애 결과를 먼저 예상

개인 기록에 먼저 답한다.

### 질문 1

pbl-c1이 보내는 ICMP Echo Request가 R1 직원망 측에 보일 것인가?

### 질문 2

같은 Echo Request가 R1 서버망 측에도 보일 것인가?

### 질문 3

pbl-web의 Echo Reply가 보일 것인가?

### 질문 4

TCP SYN은 서버망 측까지 전달될 것인가?

### 질문 5

SYN-ACK은 돌아올 것인가?

### 질문 6

HTTP `GET /`까지 관찰할 수 있을 것인가?

아직 실행하지 않았으므로 이 단계 답은 **예상**이다.

---

# 21.2 장애용 양쪽 캡처 먼저 시작

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-start broken
```

캡처가 시작되었다는 메시지를 확인한다.

---

# 21.3 pbl-web 기본 경로만 제거

```bash
sudo /usr/local/sbin/pbl-w6-lab fault-remove
```

이 명령은 정상 기본 경로가 정확히 하나이고:

```text
via 192.168.20.1
```

인 것을 확인한 후에만 삭제한다.

`ip route del`은 지정한 라우팅 항목을 제거하는 동작이다.

---

# 21.4 장애 상태의 경로를 직접 확인

```bash
sudo /usr/local/sbin/pbl-w6-lab route-check
```

특히:

```text
=== pbl-web route ===
```

와:

```text
pbl-web -> 192.168.10.10 route lookup
```

을 확인한다.

정상 기본 경로를 제거했다면 `192.168.10.10`에 대한 경로 조회가 실패할 수 있다.

실제 메시지를 그대로 기록한다.

예상되는 의미는:

```text
pbl-web은 자신의 192.168.20.0/27에는 직접 연결되어 있지만
192.168.10.10으로 나가는 경로는 현재 찾지 못한다.
```

이다.

---

# 22. 장애 상태에서 ping

```bash
sudo /usr/local/sbin/pbl-w6-lab ping
```

성공/실패 여부를 실제로 기록한다.

실패하더라도 이번 실험에서는 예상할 수 있는 결과이므로 스크립트는 캡처를 강제로 종료하지 않는다.

---

# 23. 장애 상태에서 HTTP 연결 시도

```bash
sudo /usr/local/sbin/pbl-w6-lab http
```

다음 중 실제 출력된 내용을 기록한다.

```text
연결 성공 여부
timeout 여부
HTTP 요청 출력 여부
HTTP 상태 코드 출력 여부
curl 종료 코드
```

---

# 24. 여기서 바로 원인을 단정하지 않는다

현재까지 얻은 것은 두 종류의 증거다.

### 패킷 증거

```text
직원망 측에 무엇이 보였는가?
서버망 측에 무엇이 보였는가?
어느 방향의 응답이 없었는가?
```

### 설정 증거

```text
pbl-web 라우팅 테이블에
192.168.10.10으로 가는 경로가 있는가?
```

둘을 함께 사용한다.

---

# 25. 기본 경로 복구

장애 실험 후 반드시 복구한다.

```bash
sudo /usr/local/sbin/pbl-w6-lab fault-restore
```

정상 상태에서는 다시:

```text
default via 192.168.20.1
```

이 나타나야 한다.

그리고:

```text
ip route get 192.168.10.10
```

에 해당하는 조회가 다시 유효한 경로를 보여야 한다.

---

# 26. 장애 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-stop
```

`fault-restore`를 하지 않았다면 스크립트는 종료를 거부한다.

이는 다음 참여자에게 장애 상태를 넘기는 것을 막기 위한 것이다.

두 broken PCAP도 Windows로 내려받는다.

---

# 27. 장애 PCAP에서 먼저 ICMP 확인

Display Filter:

```text
icmp
```

또는:

```text
icmp.type == 8
```

직원망 측과 서버망 측에서 Echo Request를 대응시킨다.

확인표:

| 관찰 항목 | 직원망 측 | 서버망 측 |
|---|---:|---:|
| Echo Request 관찰 |  |  |
| Source IP |  |  |
| Destination IP |  |  |
| Identifier |  |  |
| Sequence |  |  |
| TTL |  |  |
| Echo Reply 관찰 |  |  |

---

# 28. 장애 PCAP에서 TCP 확인

Display Filter:

```text
tcp.port == 8080
```

또는 SYN을 보기 쉽게:

```text
tcp.port == 8080 && tcp.flags.syn == 1
```

클라이언트 → 서버 방향 SYN을 찾는다.

확인:

```text
직원망 측 SYN 있음/없음:
서버망 측 대응 SYN 있음/없음:

서버 → 클라이언트 SYN-ACK 있음/없음:

HTTP GET 있음/없음:
```

TCP 연결이 완성되지 않았다면 `GET /`이 없을 수 있다.

이 경우:

```text
HTTP 요청이 사라졌다
```

가 아니라:

```text
TCP 연결 자체가 완성되지 않아
애플리케이션 계층 HTTP 요청 단계까지 진행되지 않았다.
```

라고 설명하는 것이 더 정확하다.

---

# 29. “응답이 없다”의 정확한 표현

예를 들어 서버망 PCAP에서 클라이언트 SYN을 봤고 서버 응답을 못 봤다고 하자.

말할 수 있는 것:

```text
클라이언트가 전송한 SYN이 R1 서버망 측 캡처 지점까지
도달한 것은 확인했다.

해당 연결에 대한 SYN-ACK은 이 캡처에서 관찰하지 못했다.
```

그 PCAP만으로 말할 수 없는 것:

```text
서버 프로그램이 SYN을 받지 않았다.
서버 NIC가 패킷을 버렸다.
서버 운영체제가 패킷을 받지 않았다.
웹 프로세스가 고장났다.
```

추가로 pbl-web 라우팅 테이블을 조회했을 때 반환 경로가 없다는 사실이 확인되면:

```text
응답 방향으로 사용할 경로가 없는 상태가 확인되었고,
관찰된 응답 부재와 일치한다.
```

라는 식으로 근거를 연결한다.

---

# 30. 실험 4 — 복구 후 동일 조건 재검증

장애를 수정했다고 끝내지 않는다.

**같은 조건으로 다시 요청하여 복구를 증명**한다.

---

# 30.1 복구 캡처 시작

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-start restored
```

---

# 30.2 ping 재실행

```bash
sudo /usr/local/sbin/pbl-w6-lab ping
```

---

# 30.3 HTTP 재실행

```bash
sudo /usr/local/sbin/pbl-w6-lab http
```

---

# 30.4 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w6-lab capture-stop
```

PCAP 두 개를 내려받는다.

---

# 31. 정상·장애·복구 비교

최종적으로 최소 다음 세 상태를 비교한다.

```text
정상
반환 경로 누락
복구
```

예시 표:

| 관찰 | 정상 | 반환 경로 누락 | 복구 |
|---|---|---|---|
| ICMP Request 직원망 측 |  |  |  |
| ICMP Request 서버망 측 |  |  |  |
| ICMP Reply 서버망 측 |  |  |  |
| ICMP Reply 직원망 측 |  |  |  |
| TCP SYN 직원망 측 |  |  |  |
| TCP SYN 서버망 측 |  |  |  |
| TCP SYN-ACK |  |  |  |
| TCP 연결 완료 |  |  |  |
| HTTP 요청 |  |  |  |
| HTTP 응답 |  |  |  |
| pbl-web 기본 경로 |  |  |  |

실제 PCAP과 실행 기록을 보고 작성한다.

---

# 32. Wireshark에서 유용한 필터

## ICMP 전체

```text
icmp
```

## Echo Request

```text
icmp.type == 8
```

## Echo Reply

```text
icmp.type == 0
```

## 클라이언트 관련 IPv4

```text
ip.addr == 192.168.10.10
```

## 서버 관련 IPv4

```text
ip.addr == 192.168.20.20
```

## TCP 8080

```text
tcp.port == 8080
```

## SYN 포함

```text
tcp.port == 8080 && tcp.flags.syn == 1
```

## 클라이언트에서 서버 방향

```text
ip.src == 192.168.10.10 &&
ip.dst == 192.168.20.20
```

## 서버에서 클라이언트 방향

```text
ip.src == 192.168.20.20 &&
ip.dst == 192.168.10.10
```

Display Filter는 화면에 보이는 패킷을 좁히는 것이며 원래 PCAP에서 패킷을 삭제하는 기능이 아니다.

---

# 33. 비교할 필드를 열로 추가하는 방법

패킷을 반복해서 비교할 때 Wireshark Packet Details에서 원하는 필드를 우클릭한 뒤:

```text
Apply as Column
```

을 이용하면 편하다.

예를 들어:

```text
ip.ttl
eth.src
eth.dst
icmp.ident
icmp.seq
tcp.srcport
tcp.dstport
```

등을 관찰한다.

Wireshark 버전에 따라 메뉴 표현은 조금 다를 수 있다.

---

# 34. 캡처에 패킷이 없을 때 진단 순서

패킷이 없다고 즉시:

```text
중간 네트워크에서 유실됐다.
```

라고 쓰지 않는다.

먼저 다음을 확인한다.

1. 올바른 PCAP을 열었는가?
2. `-staff`와 `-server` 파일을 뒤바꾸지 않았는가?
3. 요청 전에 캡처를 시작했는가?
4. 두 tcpdump PID가 실제 실행 중이었는가?
5. 올바른 R1 인터페이스를 캡처했는가?
6. Display Filter가 패킷을 숨기고 있지 않은가?
7. 캡처 파일 자체에 패킷이 있는가?
8. 실제 요청 명령을 실행했는가?

Linux에서 A가 파일 자체를 점검할 때:

```bash
sudo tcpdump -nn -r /srv/pbl/week6/captures/<FILE>.pcap -c 20
```

저장 PCAP을 `tcpdump -r`로 다시 읽을 수 있다.

---

# 35. 캡처 프로세스가 이상할 때

전체 `tcpdump`를 종료하는 다음 명령은 사용하지 않는다.

```text
pkill tcpdump
killall tcpdump
```

다른 사람이나 다른 실험의 캡처까지 종료할 수 있다.

A는 먼저 저장된 PID를 확인한다.

```bash
sudo cat /run/lock/pbl-w6/pid.staff
sudo cat /run/lock/pbl-w6/pid.server
```

그 PID가 어떤 프로세스인지 확인한다.

```bash
ps -fp <PID>
```

또는:

```bash
sudo tr '\0' ' ' < /proc/<PID>/cmdline
echo
```

예상한 tcpdump임을 확인한 경우에만 해당 PID 하나를 처리한다.

---

# 36. 참여자가 장애 상태에서 연결이 끊긴 경우 A의 복구

실습자가 `fault-remove` 후 SSH 연결을 종료하거나 명령 진행이 꼬였다면 **가장 먼저 pbl-web의 반환 경로부터 복구**한다.

## 실행 위치 → Linux 호스트 / A 관리자 세션

실제 pbl-web 인터페이스 찾기:

```bash
WEB_IF="$(
  sudo ip -n pbl-web -o -4 addr show |
  awk '$4=="192.168.20.20/27" {print $2; exit}'
)"

printf '%s\n' "$WEB_IF"
```

비어 있지 않은 것을 확인한다.

기본 경로 복구:

```bash
sudo ip -n pbl-web \
  route replace default \
  via 192.168.20.1 \
  dev "$WEB_IF"
```

검증:

```bash
sudo ip -n pbl-web route
sudo ip -n pbl-web route get 192.168.10.10
```

그 뒤 캡처 PID를 개별 확인한다.

```bash
sudo cat /run/lock/pbl-w6/pid.staff
sudo cat /run/lock/pbl-w6/pid.server
```

각 PID의 명령행을 확인한 뒤, **실제로 이번 6주차 tcpdump인 경우에만** 해당 PID에 `SIGINT`를 보낸다.

```bash
sudo kill -INT <확인한_PID>
```

두 PID를 각각 확인한다.

전체 호스트의 네트워크 서비스나 모든 tcpdump를 일괄 종료하지 않는다.

---

# 37. 이번 주 제출물 1 — 양쪽 캡처 대응표

정상 ping에서 최소:

- Echo Request 1개
- Echo Reply 1개

를 대응시킨다.

## Echo Request

| 항목 | 직원망 측 | 서버망 측 |
|---|---|---|
| PCAP 파일 |  |  |
| Frame 번호 |  |  |
| 캡처 시각 |  |  |
| Source IP |  |  |
| Destination IP |  |  |
| ICMP Identifier |  |  |
| ICMP Sequence |  |  |
| Source MAC |  |  |
| Destination MAC |  |  |
| TTL |  |  |

같은 패킷이라고 판단한 근거:

```text
1.
2.
3.
```

## Echo Reply

같은 형식으로 별도 작성한다.

---

# 38. 제출물 2 — HTTP 흐름 대응표

정상 HTTP 연결에서 최소 다음을 고른다.

```text
클라이언트 SYN
서버 SYN-ACK
HTTP 요청 또는 실제 확인 가능한 애플리케이션 패킷
HTTP 응답 또는 실제 확인 가능한 애플리케이션 패킷
```

각 패킷마다:

| 항목 | 직원망 측 | 서버망 측 |
|---|---|---|
| Frame 번호 |  |  |
| Source IP |  |  |
| Destination IP |  |  |
| Source Port |  |  |
| Destination Port |  |  |
| TCP Flags |  |  |
| Sequence 표시 |  |  |
| ACK 표시 |  |  |
| TCP Length |  |  |
| TTL |  |  |
| Source MAC |  |  |
| Destination MAC |  |  |
| 시각 |  |  |

을 작성한다.

상대 Sequence 번호가 서로 다르면 그 사실도 기록한다.

---

# 39. 제출물 3 — 정상과 반환 경로 누락 비교

다음 형식을 사용한다.

```text
[정상 상태]

pbl-web 기본 경로:
관찰한 요청:
관찰한 응답:
ping 결과:
TCP 연결 결과:
HTTP 결과:


[반환 경로 누락]

변경한 설정:
pbl-web 기본 경로:
관찰한 요청:
관찰한 응답:
ping 결과:
TCP SYN:
TCP SYN-ACK:
HTTP GET:
HTTP 응답:


[복구]

복구한 설정:
pbl-web 기본 경로:
ping 재검증:
HTTP 재검증:
복구를 확인한 패킷:
```

---

# 40. 제출물 4 — “어디까지 확인했는가?”

반드시 별도 문단으로 작성한다.

형식:

```text
확인한 사실:
-
-
-

이 사실을 확인한 근거:
-
-
-

아직 확인하지 못한 것:
-
-
-

현재 가능한 원인 후보:
-
-

후보를 좁히기 위해 추가로 필요한 증거:
-
-
```

예를 들어 다음처럼 구분할 수 있다.

```text
확인:
클라이언트 SYN이 R1의 서버망 측 인터페이스까지
전달된 것을 PCAP으로 확인했다.

확인:
pbl-web의 기본 경로가 없는 것을
ip route 출력으로 확인했다.

확인:
같은 연결에 대한 SYN-ACK을
두 PCAP에서 관찰하지 못했다.

미확인:
R1 서버망 측 캡처만으로
pbl-web 애플리케이션이 패킷을 받았다고 직접 확인하지는 못했다.
```

이 구분이 이번 주 핵심이다.

---

# 41. 개인 이해도 질문

각자 먼저 답한 뒤 팀과 비교한다.

### 질문 1

같은 ICMP 패킷인데 두 PCAP의 Frame 번호가 다른 이유는 무엇인가?

### 질문 2

같은 Echo Request를 찾기 위해 Frame 번호 대신 어떤 필드를 사용할 수 있는가?

### 질문 3

직원망 측과 서버망 측에서 Source/Destination IP가 유지되는 이유는 무엇인가?

### 질문 4

MAC 주소는 왜 달라지는가?

### 질문 5

TTL은 왜 감소하는가?

### 질문 6

정상 요청에서 서버망 측 TTL이 반드시 `63`이라고 미리 단정할 수 없는 이유는 무엇인가?

### 질문 7

pbl-web 기본 경로를 제거해도 클라이언트의 TCP SYN이 R1 서버망 측까지 보일 수 있는 이유는 무엇인가?

### 질문 8

그 상태에서 HTTP GET이 아예 보이지 않을 수 있는 이유는 무엇인가?

### 질문 9

R1 서버망 측에서 패킷을 관찰했다는 것과 pbl-web 웹 애플리케이션이 그 패킷을 받았다는 것은 동일한 사실인가?

### 질문 10

두 캡처의 timestamp가 매우 비슷하다는 것만으로 같은 패킷이라고 확정하면 안 되는 이유는 무엇인가?

### 질문 11

두 PCAP의 TCP 상대 Sequence 번호가 다르게 표시될 수 있는 이유는 무엇인가?

### 질문 12

Wireshark의 `incorrect checksum` 표시만으로 실제 전송 손상을 단정하면 안 되는 이유는 무엇인가?

### 질문 13

응답 패킷이 보이지 않을 때 가장 먼저 확인해야 할 캡처 조건은 무엇인가?

### 질문 14

경로를 수정한 뒤 왜 반드시 같은 ping과 HTTP 요청으로 다시 검증해야 하는가?

---

# 42. 실습 종료 전 개인 체크리스트

각 참여자가 모두 확인한다.

- [ ] 정상 상태를 먼저 확인했다.
- [ ] 본인 계정으로 캡처를 직접 시작했다.
- [ ] R1 직원망 측과 서버망 측을 동시에 캡처했다.
- [ ] 두 tcpdump PID가 별개임을 확인했다.
- [ ] 정상 ping을 직접 실행했다.
- [ ] 두 PCAP을 직접 내려받았다.
- [ ] ICMP Identifier와 Sequence로 같은 Echo Request를 대응시켰다.
- [ ] Frame 번호가 서로 달라도 같은 패킷일 수 있음을 확인했다.
- [ ] 직원망 측과 서버망 측 MAC 주소를 비교했다.
- [ ] TTL을 비교했다.
- [ ] 정상 HTTP 연결을 직접 캡처했다.
- [ ] IP와 TCP 포트를 이용해 같은 TCP 흐름을 대응시켰다.
- [ ] 상대 TCP Sequence 표시의 한계를 이해했다.
- [ ] 체크섬 경고를 곧바로 실제 손상이라고 단정하지 않았다.
- [ ] 반환 경로 장애 전에 캡처를 시작했다.
- [ ] pbl-web의 기본 경로만 제거했다.
- [ ] 장애 중 pbl-web 경로 조회를 직접 확인했다.
- [ ] 장애 상태의 ping을 직접 실행했다.
- [ ] 장애 상태의 HTTP 연결을 직접 실행했다.
- [ ] SYN과 SYN-ACK 존재 여부를 각각 확인했다.
- [ ] HTTP GET 존재 여부를 별도로 확인했다.
- [ ] 기본 경로를 복구했다.
- [ ] 같은 조건으로 다시 ping을 실행했다.
- [ ] 같은 조건으로 다시 HTTP 요청을 실행했다.
- [ ] 복구 PCAP을 저장했다.
- [ ] 확인한 사실과 확인하지 못한 사실을 구분했다.

---

# 43. 하루 실습 종료 시 A의 확인

## 반환 경로 정상 여부

```bash
sudo ip -n pbl-web route
```

반드시 정상 기본 경로를 확인한다.

```text
default via 192.168.20.1
```

## R1 forwarding

```bash
sudo ip netns exec pbl-r1 \
  sysctl -n net.ipv4.ip_forward
```

확인값:

```text
1
```

## 활성 캡처 여부

```bash
if sudo test -d /run/lock/pbl-w6; then
    echo "WARNING: 6주차 캡처 상태가 남아 있습니다."
    sudo cat /run/lock/pbl-w6/owner
else
    echo "활성 6주차 캡처 없음"
fi
```

## 캡처 목록

```bash
ls -lh /srv/pbl/week6/captures/
```

캡처 파일을 자동 삭제하지 않는다.

---

# 44. 이번 주에 하면 안 되는 것

다음 방식으로 문제를 해결하지 않는다.

```text
호스트 실제 NIC 주소 변경
호스트 기본 경로 변경
호스트 전체 방화벽 초기화

systemctl restart NetworkManager
systemctl restart systemd-networkd

pkill tcpdump
killall tcpdump

모든 Python 프로세스 종료
모든 네임스페이스 삭제

오프로딩을 이유 없이 일괄 비활성화
NAT 추가
호스트 우회 경로 추가
```

특히 반환 경로 장애를 해결하기 위해:

```text
호스트에 임시 route를 추가한다.
NAT를 켠다.
```

같은 우회 해결책을 사용하지 않는다.

원래 정상 구성은:

```text
pbl-web
default via 192.168.20.1
```

이므로 그 설정만 최소 수정하여 복구한다.

---

# 45. 6주차 핵심 정리

이번 주의 핵심은 “패킷이 있다/없다”만 보는 것이 아니다.

다음 사고 순서를 연습한다.

```text
같은 통신을 서로 다른 지점에서 캡처
        ↓
같은 패킷을 프로토콜 필드로 대응
        ↓
IP와 TCP/ICMP 정보 비교
        ↓
MAC과 TTL처럼 경유 과정에서 달라지는 값 확인
        ↓
정상 양방향 통신 확인
        ↓
서버의 반환 경로만 제거
        ↓
어디까지 요청이 관찰되는지 확인
        ↓
어느 방향부터 응답이 관찰되지 않는지 확인
        ↓
서버 라우팅 테이블 조회
        ↓
패킷 증거와 설정 증거를 연결
        ↓
최소 수정
        ↓
같은 조건으로 재검증
```

그리고 보고서에서는 반드시:

```text
관찰 사실
≠
추정
≠
확인하지 못한 것
```

을 구분한다.

이번 주의 최종 학습 목표는 단순히:

> “기본 게이트웨이를 지우면 통신이 안 된다.”

를 외우는 것이 아니다.

다음처럼 설명할 수 있어야 한다.

> 클라이언트가 보낸 요청이 R1의 직원망 측과 서버망 측에서 어떻게 관찰되는지 비교했고, 정상 상태에서는 Ethernet 헤더가 링크별로 바뀌고 IP TTL이 라우터 통과 과정에서 감소하는 것을 확인하였다. 반환 경로 장애에서는 클라이언트 방향 요청이 R1 서버망 측까지 전달되는지와 서버 방향 응답이 관찰되는지를 별도로 조사하였다. 이후 pbl-web의 라우팅 테이블을 조회하여 반환 경로 상태를 확인하고, 정상 기본 경로를 복구한 뒤 같은 ping과 HTTP 통신으로 정상화 여부를 다시 검증하였다.

이 과정을 근거 PCAP의 실제 Frame 번호와 설정 조회 결과로 뒷받침하면 6주차 실습이 완료된다.