# 5주차 실습 가이드
## 두 서브넷 연결과 기본 게이트웨이

프로젝트: **「소규모 사내망 설계 및 Wireshark 기반 통신 검증·장애 진단」**

---

# 1. 이번 주에 무엇이 달라지는가

4주차까지는 `pbl-c1`, `pbl-c2`, `pbl-web`이 같은 직원망에 있다고 생각할 수 있었다.

5주차부터는 웹 서버를 별도의 **서버망**으로 이동하고, 두 네트워크 사이를 **라우터 `pbl-r1`**으로 연결한다.

이번 주 최종 구조는 다음과 같다.

```text
                         공용 Ubuntu Linux 호스트
              ─────────────────────────────────────────

                     직원망 192.168.10.0/26

 pbl-c1                     pbl-c2
 eth0                       eth0
 192.168.10.10/26           192.168.10.11/26
 default via .10.1           default via .10.1
     │                           │
     │ veth                      │ veth
     ▼                           ▼
 ┌───────────────────────────────────────────┐
 │             pbl-br-staff                  │
 └──────────────────┬────────────────────────┘
                    │
                    │
             pbl-r1 / staff0
             192.168.10.1/26
                    │
             [ IPv4 forwarding ]
                    │
             pbl-r1 / server0
             192.168.20.1/27
                    │
                    │
 ┌──────────────────┴────────────────────────┐
 │             pbl-br-server                 │
 └──────────────────┬────────────────────────┘
                    │
                    │ veth
                    ▼
                 pbl-web
                 eth0
                 192.168.20.20/27
                 default via 192.168.20.1
                 TCP 8080
```

`pbl-web`의 이전 주소:

```text
192.168.10.20/26
```

은 제거한다.

이번 구성에는 NAT가 없다.

따라서 `pbl-c1 → pbl-web` 통신에서 IP 주소가 다음처럼 유지된다.

```text
출발지 IP: 192.168.10.10
목적지 IP: 192.168.20.20
```

라우터가 목적지 IP를 자신의 `192.168.10.1` 또는 `192.168.20.1`로 바꾸는 것이 아니다.

라우터는 **다음 네트워크로 전달할 때 Ethernet 헤더를 새로 구성**한다.

---

# 2. 이번 주 학습 목표

A·B·C·D 전원이 실습 후 다음을 설명할 수 있어야 한다.

1. 직접 연결된 목적지와 원격 네트워크 목적지를 구분한다.
2. 기본 게이트웨이가 언제 사용되는지 설명한다.
3. `ip route`에서 직접 연결 경로와 `default` 경로를 구분한다.
4. 같은 LAN 통신에서는 목적지 호스트의 MAC 주소를 사용하는 것을 확인한다.
5. 다른 LAN 통신에서는 첫 Ethernet 프레임의 목적지 MAC이 **게이트웨이 MAC**임을 확인한다.
6. 이 경우에도 IP 목적지는 최종 웹 서버 `192.168.20.20`으로 유지됨을 확인한다.
7. `pbl-c1`의 기본 경로를 삭제했을 때 같은 직원망 통신은 유지되지만 서버망 통신은 실패하는 이유를 설명한다.
8. 요청 경로뿐 아니라 서버의 응답이 돌아오기 위한 경로도 필요함을 설명한다.
9. 기본 경로를 복원하고 같은 요청으로 정상화를 검증한다.
10. 패킷과 라우팅 테이블을 함께 근거로 사용한다.

---

# 3. 핵심 개념

## 3.1 직접 연결 경로

`pbl-c1`의 주소가 다음이라고 하자.

```text
192.168.10.10/26
```

정상 구성에서는 라우팅 테이블에 대략 다음 두 종류의 경로가 있다.

```text
192.168.10.0/26 dev eth0 ...
default via 192.168.10.1 dev eth0
```

첫 번째가 **직접 연결 경로**다.

따라서 `192.168.10.11`인 `pbl-c2`로 갈 때는 기본 게이트웨이를 사용할 필요가 없다.

Linux의 `ip route`는 라우팅 테이블을 관리하며, `default via <게이트웨이>`는 다른 더 구체적인 경로가 선택되지 않는 목적지를 전달하는 기본 경로를 만든다.

---

## 3.2 기본 게이트웨이

`pbl-c1`이 다음 주소로 통신한다고 하자.

```text
192.168.20.20
```

이는 `192.168.10.0/26` 밖에 있다.

`pbl-c1`에는 `192.168.20.0/27`에 대한 개별 경로가 없으므로 다음 기본 경로를 사용한다.

```text
default via 192.168.10.1 dev eth0
```

따라서 첫 번째 전달 대상은:

```text
192.168.10.1
```

인 `pbl-r1`이다.

그러나 여기서 매우 중요한 구분이 있다.

```text
IPv4 Destination = 192.168.20.20
Ethernet Destination = pbl-r1 직원망 측 MAC
```

즉,

> 원격 IP로 가는 패킷의 IP 목적지를 게이트웨이 주소로 바꾸는 것이 아니다.

---

## 3.3 Ethernet 주소는 홉마다 달라질 수 있다

`pbl-c1 → pbl-web` 요청의 직원망 구간에서는:

```text
Ethernet Source      = pbl-c1 MAC
Ethernet Destination = pbl-r1 staff0 MAC

IPv4 Source          = 192.168.10.10
IPv4 Destination     = 192.168.20.20
```

라우터가 서버망으로 전달할 때는 새 Ethernet 프레임이 만들어진다.

개념적으로:

```text
Ethernet Source      = pbl-r1 server0 MAC
Ethernet Destination = pbl-web MAC

IPv4 Source          = 192.168.10.10
IPv4 Destination     = 192.168.20.20
```

이번 주에는 `pbl-c1` 쪽에서 캡처한다.

라우터 양쪽 캡처를 직접 나란히 비교하는 활동은 6주차에서 수행한다.

---

## 3.4 요청만 갈 수 있어서는 부족하다

`pbl-c1`이 서버에 요청을 보내더라도 `pbl-web`이 다음 목적지로 돌아갈 방법을 알아야 한다.

```text
192.168.10.10
```

`pbl-web`의 정상 라우팅 테이블에는:

```text
192.168.20.0/27 dev eth0 ...
default via 192.168.20.1 dev eth0
```

이 있어야 한다.

따라서 서버의 응답은:

```text
pbl-web
→ 192.168.20.1 pbl-r1
→ 직원망
→ pbl-c1
```

방향으로 돌아온다.

---

# 4. 자원 이름과 주소표

| 역할 | 자원/인터페이스 | 주소 | 기본 게이트웨이 |
|---|---|---|---|
| 직원망 bridge | `pbl-br-staff` | 호스트 IPv4 없음 | 없음 |
| 클라이언트 1 | `pbl-c1 / eth0` | `192.168.10.10/26` | `192.168.10.1` |
| 클라이언트 2 | `pbl-c2 / eth0` | `192.168.10.11/26` | `192.168.10.1` |
| R1 직원망 | `pbl-r1 / staff0` | `192.168.10.1/26` | 없음 |
| R1 서버망 | `pbl-r1 / server0` | `192.168.20.1/27` | 없음 |
| 서버망 bridge | `pbl-br-server` | 호스트 IPv4 없음 | 없음 |
| 웹 서버 | `pbl-web / eth0` | `192.168.20.20/27` | `192.168.20.1` |
| 웹 서비스 | `pbl-web` | TCP `8080` | - |

호스트에 보이는 veth 이름:

| 용도 | Linux 호스트 쪽 이름 |
|---|---|
| pbl-c1 | `pbl-c1-h` |
| pbl-c2 | `pbl-c2-h` |
| pbl-web | `pbl-web-h` |
| R1 직원망 | `pbl-r1-st-h` |
| R1 서버망 | `pbl-r1-sv-h` |

Linux의 veth는 항상 서로 연결된 쌍으로 생성되며 한쪽을 네트워크 네임스페이스에 옮길 수 있다.

---

# 5. 공유 서버 운영 원칙

A가 환경 구성을 주도한다.

그러나 A도 셋업을 마친 뒤 B·C·D와 똑같이 개인 실습을 수행한다.

공유 자원인 `pbl-c1`의 기본 경로를 변경하기 때문에 실제 실험은:

```text
A 완료
→ 정상 상태 복구
→ B 완료
→ 정상 상태 복구
→ C 완료
→ 정상 상태 복구
→ D 완료
```

처럼 한 명씩 수행한다.

PCAP을 Windows PC로 내려받은 뒤의 Wireshark 분석은 동시에 진행해도 된다.

---

# 6. 셋업 담당자 A — 변경 전 상태 백업

## 6.1 관리망부터 확인

### 실행 위치 → Linux 호스트
### 관리자 권한 불필요

```bash
whoami
echo "$SSH_CONNECTION"
ip -br addr
ip -4 route
ip netns list
```

### 예상 관찰

실제 관리 NIC, SSH 접속 정보, 기존 네임스페이스가 표시된다.

### 의미

5주차 구성을 만들기 전 관리망 상태를 남긴다.

### 주의

다음은 변경하지 않는다.

```text
실제 관리 NIC
실제 관리 IP
호스트 기본 경로
호스트의 전체 방화벽 정책
```

---

## 6.2 주소 충돌 확인

### 실행 위치 → Linux 호스트

```bash
ip -4 addr
ip -4 route
```

VPN이 별도로 있다면 VPN 연결 상태에서도 경로를 확인한다.

### 확인 대상

실제 관리망이나 VPN이 다음 주소 계획을 사용하고 있지 않은지 확인한다.

```text
192.168.10.0/26
192.168.20.0/27
```

충돌한다면 일부 IP만 즉석에서 바꾸지 않는다.

**프로젝트 전체 주소 계획을 일관되게 다시 정한 후 진행한다.**

---

# 7. 필요한 프로그램 확인

### 실행 위치 → Linux 호스트

```bash
command -v ip
command -v bridge
command -v python3
command -v curl
command -v ping
command -v tcpdump
command -v ss
command -v sysctl
```

없는 것이 있다면 A가 설치한다.

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping tcpdump python3 curl procps
```

전체 시스템 업그레이드는 필요하지 않다.

---

# 8. 실습 그룹 확인

기존 가이드에서 `pbl` 그룹과 실습 계정을 사용했다면 그대로 활용한다.

### 실행 위치 → Linux 호스트

```bash
getent group pbl
```

없다면:

### 관리자 권한 필요

```bash
sudo groupadd -f pbl
```

실습용 계정만 그룹에 등록한다.

예를 들어 기존 계정이 `pbl-a`~`pbl-d`라면:

```bash
sudo usermod -aG pbl pbl-a
sudo usermod -aG pbl pbl-b
sudo usermod -aG pbl pbl-c
sudo usermod -aG pbl pbl-d
```

그룹 추가 후에는 해당 사용자가 로그아웃했다가 다시 로그인해야 할 수 있다.

확인:

```bash
id
```

이번 5주차 보조 스크립트는 사용자명을 `pbl-a`처럼 하드코딩하지 않고, **`pbl` 그룹 구성원인지 확인**한다.

따라서 실습용 계정명이 다른 환경에서도 사용할 수 있다.

---

# 9. 5주차 전체 네트워크 구성 스크립트

## 9.1 저장

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w5-setup
```

아래 내용을 전체 저장한다.

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ ${EUID} -ne 0 ]]; then
    echo "ERROR: sudo 또는 root 권한으로 실행해야 합니다." >&2
    exit 1
fi

BASE="/srv/pbl/week5"
BACKUP_ROOT="${BASE}/backups"
CAPDIR="${BASE}/captures"
LOGDIR="${BASE}/logs"
WWW="${BASE}/www"

WEB_PID="/run/pbl-w5-web.pid"
WEB_LOG="${LOGDIR}/web.log"

BR_STAFF="pbl-br-staff"
BR_SERVER="pbl-br-server"

for cmd in ip bridge python3 curl ping tcpdump ss sysctl \
           getent awk grep sort ps; do
    if ! command -v "${cmd}" >/dev/null 2>&1; then
        echo "ERROR: 필요한 명령이 없습니다: ${cmd}" >&2
        exit 1
    fi
done

if ! getent group pbl >/dev/null 2>&1; then
    echo "ERROR: pbl 그룹이 없습니다." >&2
    echo "먼저 sudo groupadd -f pbl 을 실행하십시오." >&2
    exit 1
fi

install -d -o root -g pbl -m 2770 "${BASE}"
install -d -o root -g pbl -m 2770 "${BACKUP_ROOT}"
install -d -o root -g pbl -m 2770 "${CAPDIR}"
install -d -o root -g pbl -m 2770 "${LOGDIR}"
install -d -o root -g pbl -m 2770 "${WWW}"

ns_exists() {
    ip netns list | awk '{print $1}' | grep -Fxq "$1"
}

echo "=== [1/9] 변경 전 상태 백업 ==="

stamp="$(date +%Y%m%d-%H%M%S)"
BACKUP="${BACKUP_ROOT}/${stamp}"
mkdir -p "${BACKUP}"

ip -br addr > "${BACKUP}/host-ip-br-addr.txt"
ip -br link > "${BACKUP}/host-ip-br-link.txt"
ip -4 route > "${BACKUP}/host-ipv4-route.txt"
ip netns list > "${BACKUP}/netns-list.txt"
bridge link show > "${BACKUP}/bridge-link.txt" 2>&1 || true
sysctl net.ipv4.ip_forward > "${BACKUP}/host-ip-forward.txt"

for ns in pbl-c1 pbl-c2 pbl-web pbl-r1; do
    if ns_exists "${ns}"; then
        {
            echo "=== LINK ==="
            ip -n "${ns}" -br link
            echo
            echo "=== ADDRESS ==="
            ip -n "${ns}" -br addr
            echo
            echo "=== ROUTE ==="
            ip -n "${ns}" route
            echo
            echo "=== PIDS ==="
            ip netns pids "${ns}" || true
        } > "${BACKUP}/${ns}.txt" 2>&1
    fi
done

HOST_FWD_BEFORE="$(sysctl -n net.ipv4.ip_forward)"

echo "백업 위치: ${BACKUP}"

echo
echo "=== [2/9] 기존 프로젝트 프로세스 안전 확인 ==="

# c1, c2, r1에는 지속 실행 프로세스가 없어야 한다.
for ns in pbl-c1 pbl-c2 pbl-r1; do
    if ns_exists "${ns}"; then
        pids="$(ip netns pids "${ns}" 2>/dev/null | xargs || true)"

        if [[ -n "${pids}" ]]; then
            echo "ERROR: ${ns}에 실행 중인 프로세스가 있습니다." >&2
            echo "PID: ${pids}" >&2
            echo "임의로 종료하지 않고 셋업을 중단합니다." >&2
            exit 1
        fi
    fi
done

# pbl-web에서는 기존 실습용 Python HTTP 서버만 종료 허용.
if ns_exists pbl-web; then
    allowed_pids=()
    unknown_found=0

    for pid in $(ip netns pids pbl-web 2>/dev/null || true); do
        cmdline="$(tr '\0' ' ' < "/proc/${pid}/cmdline" 2>/dev/null || true)"

        if [[ "${cmdline}" == *"python3"* &&
              "${cmdline}" == *"-m http.server"* &&
              "${cmdline}" == *"8080"* ]]; then
            allowed_pids+=("${pid}")
        else
            echo "ERROR: pbl-web에서 예상하지 못한 프로세스 발견" >&2
            echo "PID ${pid}: ${cmdline}" >&2
            unknown_found=1
        fi
    done

    if [[ ${unknown_found} -ne 0 ]]; then
        echo "알 수 없는 프로세스를 강제 종료하지 않습니다." >&2
        exit 1
    fi

    for pid in "${allowed_pids[@]}"; do
        echo "기존 실습용 웹 서버 종료: PID ${pid}"
        kill "${pid}" 2>/dev/null || true

        for _ in {1..20}; do
            kill -0 "${pid}" 2>/dev/null || break
            sleep 0.1
        done
    done
fi

rm -f "${WEB_PID}"

echo
echo "=== [3/9] 기존 프로젝트 bridge 포트 확인 ==="

check_bridge_ports() {
    local br="$1"
    shift
    local allowed="$*"

    if ! ip link show "${br}" >/dev/null 2>&1; then
        return
    fi

    while IFS= read -r dev; do
        [[ -z "${dev}" ]] && continue

        case " ${allowed} " in
            *" ${dev} "*)
                ;;
            *)
                echo "ERROR: ${br}에 알 수 없는 인터페이스가 연결되어 있습니다: ${dev}" >&2
                echo "bridge를 삭제하지 않고 중단합니다." >&2
                exit 1
                ;;
        esac
    done < <(
        ip -o link show master "${br}" \
        | awk -F': ' '{print $2}' \
        | cut -d@ -f1
    )
}

check_bridge_ports \
    "${BR_STAFF}" \
    pbl-c1-h pbl-c2-h pbl-web-h pbl-r1-st-h

check_bridge_ports \
    "${BR_SERVER}" \
    pbl-web-h pbl-r1-sv-h

echo
echo "=== [4/9] 기존 프로젝트 가상 네트워크 제거 ==="

for ns in pbl-c1 pbl-c2 pbl-web pbl-r1; do
    if ns_exists "${ns}"; then
        ip netns del "${ns}"
    fi
done

for dev in \
    pbl-c1-h \
    pbl-c2-h \
    pbl-web-h \
    pbl-r1-st-h \
    pbl-r1-sv-h
do
    if ip link show "${dev}" >/dev/null 2>&1; then
        ip link del "${dev}"
    fi
done

for br in "${BR_STAFF}" "${BR_SERVER}"; do
    if ip link show "${br}" >/dev/null 2>&1; then
        ip link del "${br}"
    fi
done

echo
echo "=== [5/9] 네임스페이스와 bridge 생성 ==="

ip netns add pbl-c1
ip netns add pbl-c2
ip netns add pbl-web
ip netns add pbl-r1

ip link add "${BR_STAFF}" type bridge
ip link add "${BR_SERVER}" type bridge

ip link set "${BR_STAFF}" up
ip link set "${BR_SERVER}" up

ip -n pbl-c1 link set lo up
ip -n pbl-c2 link set lo up
ip -n pbl-web link set lo up
ip -n pbl-r1 link set lo up

# IP forwarding은 pbl-r1 네임스페이스에서만 활성화한다.
ip netns exec pbl-r1 \
    sysctl -q -w net.ipv4.ip_forward=1

echo
echo "=== [6/9] 직원망 구성 ==="

# pbl-c1
ip link add pbl-c1-h type veth peer name pbl-c1-n
ip link set pbl-c1-h master "${BR_STAFF}"
ip link set pbl-c1-h up
ip link set pbl-c1-n netns pbl-c1

ip -n pbl-c1 link set pbl-c1-n name eth0
ip -n pbl-c1 link set eth0 up
ip -n pbl-c1 addr add 192.168.10.10/26 dev eth0
ip -n pbl-c1 route replace \
    default via 192.168.10.1 dev eth0

# pbl-c2
ip link add pbl-c2-h type veth peer name pbl-c2-n
ip link set pbl-c2-h master "${BR_STAFF}"
ip link set pbl-c2-h up
ip link set pbl-c2-n netns pbl-c2

ip -n pbl-c2 link set pbl-c2-n name eth0
ip -n pbl-c2 link set eth0 up
ip -n pbl-c2 addr add 192.168.10.11/26 dev eth0
ip -n pbl-c2 route replace \
    default via 192.168.10.1 dev eth0

# pbl-r1 직원망 측
ip link add pbl-r1-st-h type veth peer name pbl-r1-st-n
ip link set pbl-r1-st-h master "${BR_STAFF}"
ip link set pbl-r1-st-h up
ip link set pbl-r1-st-n netns pbl-r1

ip -n pbl-r1 link set pbl-r1-st-n name staff0
ip -n pbl-r1 link set staff0 up
ip -n pbl-r1 addr add 192.168.10.1/26 dev staff0

echo
echo "=== [7/9] 서버망 구성 ==="

# pbl-web
ip link add pbl-web-h type veth peer name pbl-web-n
ip link set pbl-web-h master "${BR_SERVER}"
ip link set pbl-web-h up
ip link set pbl-web-n netns pbl-web

ip -n pbl-web link set pbl-web-n name eth0
ip -n pbl-web link set eth0 up
ip -n pbl-web addr add 192.168.20.20/27 dev eth0
ip -n pbl-web route replace \
    default via 192.168.20.1 dev eth0

# pbl-r1 서버망 측
ip link add pbl-r1-sv-h type veth peer name pbl-r1-sv-n
ip link set pbl-r1-sv-h master "${BR_SERVER}"
ip link set pbl-r1-sv-h up
ip link set pbl-r1-sv-n netns pbl-r1

ip -n pbl-r1 link set pbl-r1-sv-n name server0
ip -n pbl-r1 link set server0 up
ip -n pbl-r1 addr add 192.168.20.1/27 dev server0

echo
echo "=== [8/9] 웹 페이지와 HTTP 서비스 시작 ==="

cat > "${WWW}/index.html" <<'EOF'
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>PBL Week 5</title>
</head>
<body>
  <h1>PBL Week 5 OK</h1>
  <p>pbl-web is now on 192.168.20.20/27.</p>
</body>
</html>
EOF

chown root:pbl "${WWW}/index.html"
chmod 0644 "${WWW}/index.html"

nohup ip netns exec pbl-web \
    python3 -m http.server 8080 \
    --bind 192.168.20.20 \
    --directory "${WWW}" \
    >"${WEB_LOG}" 2>&1 < /dev/null &

echo $! > "${WEB_PID}"

sleep 0.7

webpid="$(cat "${WEB_PID}")"

if ! kill -0 "${webpid}" 2>/dev/null; then
    echo "ERROR: 웹 서비스가 시작되지 않았습니다." >&2
    cat "${WEB_LOG}" >&2 || true
    exit 1
fi

echo
echo "=== [9/9] 최종 자체 점검 ==="

# pbl-web의 구 주소가 남아 있으면 실패.
if ip -n pbl-web -4 addr show dev eth0 \
    | grep -q '192\.168\.10\.20'; then
    echo "ERROR: pbl-web의 이전 주소 192.168.10.20이 남아 있습니다." >&2
    exit 1
fi

# bridge 자체에는 실습 IPv4 주소를 주지 않는다.
for br in "${BR_STAFF}" "${BR_SERVER}"; do
    if ip -4 -o addr show dev "${br}" | grep -q .; then
        echo "ERROR: ${br}에 예상하지 못한 IPv4 주소가 있습니다." >&2
        ip -4 addr show dev "${br}" >&2
        exit 1
    fi
done

# bridge 포트가 의도한 구성인지 확인.
mapfile -t staff_ports < <(
    ip -o link show master "${BR_STAFF}" \
    | awk -F': ' '{print $2}' \
    | cut -d@ -f1 \
    | sort
)

actual_staff="${staff_ports[*]}"
expected_staff="pbl-c1-h pbl-c2-h pbl-r1-st-h"

if [[ "${actual_staff}" != "${expected_staff}" ]]; then
    echo "ERROR: 직원망 bridge 포트 구성이 예상과 다릅니다." >&2
    echo "실제: ${actual_staff}" >&2
    echo "예상: ${expected_staff}" >&2
    exit 1
fi

mapfile -t server_ports < <(
    ip -o link show master "${BR_SERVER}" \
    | awk -F': ' '{print $2}' \
    | cut -d@ -f1 \
    | sort
)

actual_server="${server_ports[*]}"
expected_server="pbl-r1-sv-h pbl-web-h"

if [[ "${actual_server}" != "${expected_server}" ]]; then
    echo "ERROR: 서버망 bridge 포트 구성이 예상과 다릅니다." >&2
    echo "실제: ${actual_server}" >&2
    echo "예상: ${expected_server}" >&2
    exit 1
fi

R1_FWD="$(ip netns exec pbl-r1 \
    sysctl -n net.ipv4.ip_forward)"

HOST_FWD_AFTER="$(sysctl -n net.ipv4.ip_forward)"

if [[ "${R1_FWD}" != "1" ]]; then
    echo "ERROR: pbl-r1 IPv4 forwarding이 활성화되지 않았습니다." >&2
    exit 1
fi

if [[ "${HOST_FWD_AFTER}" != "${HOST_FWD_BEFORE}" ]]; then
    echo "ERROR: 호스트 net.ipv4.ip_forward 값이 변경되었습니다." >&2
    echo "변경 전: ${HOST_FWD_BEFORE}" >&2
    echo "변경 후: ${HOST_FWD_AFTER}" >&2
    exit 1
fi

echo "직원망 내부 확인..."
ip netns exec pbl-c1 \
    ping -c 1 -W 2 192.168.10.11 >/dev/null

echo "직원망 -> 서버망 HTTP 확인..."
ip netns exec pbl-c1 \
    curl --noproxy '*' -fsS --max-time 5 \
    http://192.168.20.20:8080/ \
    | grep -q 'PBL Week 5 OK'

echo "서버망 -> 직원망 반환 경로 확인..."
ip netns exec pbl-web \
    ping -c 1 -W 2 192.168.10.10 >/dev/null

echo
echo "=== pbl-c1 ==="
ip -n pbl-c1 -br addr
ip -n pbl-c1 route

echo
echo "=== pbl-c2 ==="
ip -n pbl-c2 -br addr
ip -n pbl-c2 route

echo
echo "=== pbl-r1 ==="
ip -n pbl-r1 -br addr
ip -n pbl-r1 route
echo "net.ipv4.ip_forward = ${R1_FWD}"

echo
echo "=== pbl-web ==="
ip -n pbl-web -br addr
ip -n pbl-web route

echo
echo "=== HTTP LISTEN ==="
ip netns exec pbl-web ss -lntp | grep ':8080'

echo
echo "=== bridge ports ==="
echo "[${BR_STAFF}]"
ip -o link show master "${BR_STAFF}"
echo
echo "[${BR_SERVER}]"
ip -o link show master "${BR_SERVER}"

echo
echo "SETUP CHECK: OK"
echo "백업 위치: ${BACKUP}"
echo "NAT 규칙은 구성하지 않았습니다."
echo "호스트 실제 NIC와 기본 경로를 수정하지 않았습니다."
```

저장 후:

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo chmod 0755 /usr/local/sbin/pbl-w5-setup
sudo /usr/local/sbin/pbl-w5-setup
```

---

# 10. 셋업 정상 결과 읽기

정상이라면 마지막에:

```text
SETUP CHECK: OK
```

가 나타나야 한다.

`pbl-c1`에는 대략:

```text
192.168.10.0/26 dev eth0 ...
default via 192.168.10.1 dev eth0
```

`pbl-web`에는:

```text
192.168.20.0/27 dev eth0 ...
default via 192.168.20.1 dev eth0
```

`pbl-r1`에는:

```text
192.168.10.0/26 dev staff0 ...
192.168.20.0/27 dev server0 ...
```

가 나타난다.

`pbl-r1`에는 이번 주 별도의 default route가 없어도 된다.

두 네트워크가 모두 라우터에 **직접 연결**되어 있기 때문이다.

---

# 11. IP forwarding 위치 확인

네트워크 네임스페이스는 네트워크 장치뿐 아니라 IPv4/IPv6 프로토콜 스택, 라우팅 테이블, 방화벽 규칙과 여러 `/proc/sys/net` 값을 격리한다.

Linux 커널의 `net.ipv4.ip_forward`는 인터페이스 사이 IPv4 패킷 전달을 활성화한다.

### 실행 위치 → Linux 호스트

```bash
sysctl net.ipv4.ip_forward
sudo ip netns exec pbl-r1 \
    sysctl net.ipv4.ip_forward
```

### 의미

두 출력은 서로 별개의 네트워크 네임스페이스 상태다.

이번 프로젝트에서 중요한 것은:

```text
pbl-r1:
net.ipv4.ip_forward = 1
```

이다.

호스트 값을 실습 때문에 `1`로 변경해서는 안 된다.

---

# 12. 웹 서버의 이전 주소 제거 확인

### 실행 위치 → Linux 호스트

```bash
sudo ip -n pbl-web -4 addr show dev eth0
```

정상:

```text
192.168.20.20/27
```

확인 대상:

```text
192.168.10.20
```

은 없어야 한다.

직원망에서 옛 주소를 계속 사용하면 안 된다.

---

# 13. 우회 경로가 없는지 확인

## 13.1 직원망 bridge

### 실행 위치 → Linux 호스트

```bash
ip -o link show master pbl-br-staff
```

정상 멤버:

```text
pbl-c1-h
pbl-c2-h
pbl-r1-st-h
```

`pbl-web-h`가 직원망에 있으면 안 된다.

---

## 13.2 서버망 bridge

```bash
ip -o link show master pbl-br-server
```

정상 멤버:

```text
pbl-web-h
pbl-r1-sv-h
```

`pbl-c1-h`, `pbl-c2-h`가 서버망에 직접 연결되면 안 된다.

---

## 13.3 bridge IPv4 주소 확인

```bash
ip -4 addr show dev pbl-br-staff
ip -4 addr show dev pbl-br-server
```

실습용 `inet` 주소가 없어야 한다.

즉, Linux 호스트 자신이:

```text
192.168.10.x
192.168.20.x
```

주소를 가진 라우터가 되는 구성이 아니다.

---

# 14. NAT를 사용하지 않았는지 확인

이번 셋업 스크립트에는 다음 종류의 설정이 없다.

```text
MASQUERADE
SNAT
DNAT
```

따라서 내부 통신은 주소 변환 없이 이루어진다.

`pbl-c1 → pbl-web`에서 최종 목적지 IP는 계속:

```text
192.168.20.20
```

이다.

---

# 15. 호스트 방화벽 관련 주의

실습을 위해 다음과 같은 조치를 하지 않는다.

```bash
sudo iptables -F
sudo iptables -P FORWARD ACCEPT
sudo nft flush ruleset
```

Docker 등 다른 소프트웨어 때문에 호스트의 FORWARD 정책이 이미 존재한다면 이를 무작정 초기화해서는 안 된다.

셋업 자체 점검에서 통신이 실패하는 경우:

1. 주소
2. bridge 멤버십
3. 네임스페이스 경로
4. `pbl-r1` forwarding
5. 캡처에서 실제 프레임이 어느 지점까지 보이는지

순서로 확인한다.

공유 호스트의 기존 방화벽 정책이 실습용 Linux bridge까지 실제로 간섭하는 환경이라면, 기존 정책을 삭제하는 대신 **실습용 VM처럼 다른 서비스와 분리된 환경을 사용하는 것이 가장 단순하고 재현성이 높다.**

---

# 16. 실습자용 보조 명령 설치

A·B·C·D가 호스트 전체에 root 명령을 자유롭게 실행할 필요는 없다.

이번 주 필요한 작업만 제한한다.

## 저장

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w5-lab
```

전체 코드:

```bash
#!/usr/bin/env bash
set -euo pipefail

ACTION="${1:-}"
ARG="${2:-}"
REAL_USER="${SUDO_USER:-}"

BASE="/srv/pbl/week5"
CAPDIR="${BASE}/captures"

LOCKDIR="/run/lock/pbl-w5-lab"
PIDFILE="${LOCKDIR}/tcpdump.pid"
FILEFILE="${LOCKDIR}/capture.file"
OWNERFILE="${LOCKDIR}/owner"

if [[ ${EUID} -ne 0 ]]; then
    echo "ERROR: sudo를 통해 실행하십시오." >&2
    exit 1
fi

if [[ -z "${REAL_USER}" ]]; then
    echo "ERROR: SUDO_USER를 확인할 수 없습니다." >&2
    exit 1
fi

if ! id -nG "${REAL_USER}" \
    | tr ' ' '\n' \
    | grep -Fxq pbl; then
    echo "ERROR: pbl 실습 그룹 구성원이 아닙니다: ${REAL_USER}" >&2
    exit 1
fi

require_owner() {
    if [[ ! -d "${LOCKDIR}" ]]; then
        echo "ERROR: 먼저 capture-start를 실행하십시오." >&2
        exit 1
    fi

    owner="$(cat "${OWNERFILE}" 2>/dev/null || true)"

    if [[ "${owner}" != "${REAL_USER}" ]]; then
        echo "ERROR: 현재 실습 사용자는 ${owner}입니다." >&2
        exit 1
    fi
}

case "${ACTION}" in

check)
    echo "=== pbl-c1 ==="
    ip -n pbl-c1 -br addr
    ip -n pbl-c1 route

    echo
    echo "=== pbl-c2 ==="
    ip -n pbl-c2 -br addr

    echo
    echo "=== pbl-r1 ==="
    ip -n pbl-r1 -br addr
    ip -n pbl-r1 route
    echo -n "IPv4 forwarding: "
    ip netns exec pbl-r1 \
        sysctl -n net.ipv4.ip_forward

    echo
    echo "=== pbl-web ==="
    ip -n pbl-web -br addr
    ip -n pbl-web route

    echo
    echo "=== TCP 8080 ==="
    ip netns exec pbl-web \
        ss -lnt | grep ':8080' || {
            echo "ERROR: TCP 8080 LISTEN을 확인하지 못했습니다." >&2
            exit 1
        }

    echo
    if [[ -d "${LOCKDIR}" ]]; then
        echo -n "현재 실습 사용자: "
        cat "${OWNERFILE}" 2>/dev/null || echo unknown
    else
        echo "현재 활성 실습 없음"
    fi

    if ! ip -n pbl-c1 route show default | grep -q .; then
        echo
        echo "WARNING: pbl-c1 기본 경로가 없습니다."
        echo "실험 종료 상태라면 default-restore가 필요합니다."
    fi
    ;;

macs)
    echo "=== 실제 MAC 주소 ==="

    printf "pbl-c1 eth0       : "
    ip netns exec pbl-c1 \
        cat /sys/class/net/eth0/address

    printf "pbl-c2 eth0       : "
    ip netns exec pbl-c2 \
        cat /sys/class/net/eth0/address

    printf "pbl-r1 staff0     : "
    ip netns exec pbl-r1 \
        cat /sys/class/net/staff0/address

    printf "pbl-r1 server0    : "
    ip netns exec pbl-r1 \
        cat /sys/class/net/server0/address

    printf "pbl-web eth0      : "
    ip netns exec pbl-web \
        cat /sys/class/net/eth0/address
    ;;

capture-start)
    case "${ARG}" in
        normal|no-default|recovered)
            ;;
        *)
            echo "ERROR: 캡처 이름은 normal, no-default, recovered 중 하나입니다." >&2
            exit 2
            ;;
    esac

    if ! mkdir "${LOCKDIR}" 2>/dev/null; then
        echo "ERROR: 다른 참여자가 이미 실습 중입니다." >&2
        echo -n "현재 사용자: " >&2
        cat "${OWNERFILE}" 2>/dev/null || echo unknown >&2
        exit 1
    fi

    echo "${REAL_USER}" > "${OWNERFILE}"

    stamp="$(date +%Y%m%d-%H%M%S)"
    outfile="${CAPDIR}/${REAL_USER}-${ARG}-${stamp}.pcap"
    logfile="${CAPDIR}/${REAL_USER}-${ARG}-${stamp}.tcpdump.log"

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
    echo "조건: ${ARG}"
    echo "시각: $(date --iso-8601=seconds)"
    echo "파일: ${outfile}"
    ;;

routes)
    require_owner

    echo "=== pbl-c1 route ==="
    ip -n pbl-c1 route

    echo
    echo "=== route get: 같은 직원망 pbl-c2 ==="
    ip netns exec pbl-c1 \
        ip route get 192.168.10.11 || true

    echo
    echo "=== route get: 서버망 pbl-web ==="
    ip netns exec pbl-c1 \
        ip route get 192.168.20.20 || true

    echo
    echo "=== pbl-r1 route ==="
    ip -n pbl-r1 route

    echo
    echo "=== pbl-web route ==="
    ip -n pbl-web route
    ;;

same-lan)
    require_owner

    # ARP를 매 실험에서 직접 관찰할 수 있도록
    # pbl-c1 자신의 이웃 캐시만 비운다.
    ip netns exec pbl-c1 \
        ip neigh flush dev eth0 >/dev/null 2>&1 || true

    echo "=== SAME LAN TEST ==="
    echo "pbl-c1 -> pbl-c2"
    echo "192.168.10.10 -> 192.168.10.11"

    ip netns exec pbl-c1 \
        ping -c 2 -W 2 192.168.10.11
    ;;

web)
    require_owner

    ip netns exec pbl-c1 \
        ip neigh flush dev eth0 >/dev/null 2>&1 || true

    echo "=== REMOTE WEB TEST ==="
    echo "pbl-c1 -> pbl-web"
    echo "192.168.10.10 -> 192.168.20.20:8080"
    echo "REQUEST TIME: $(date --iso-8601=seconds)"
    echo

    if ip netns exec pbl-c1 \
        curl \
        --noproxy '*' \
        -v \
        --max-time 5 \
        http://192.168.20.20:8080/
    then
        echo
        echo "WEB RESULT: SUCCESS"
    else
        rc=$?
        echo
        echo "WEB RESULT: FAILED"
        echo "curl exit code: ${rc}"
        exit "${rc}"
    fi
    ;;

default-remove)
    require_owner

    if ip -n pbl-c1 route show default | grep -q .; then
        ip -n pbl-c1 route del default
    fi

    echo "pbl-c1 기본 경로 제거 완료"
    ip -n pbl-c1 route
    ;;

default-restore)
    if [[ -d "${LOCKDIR}" ]]; then
        require_owner
    fi

    ip -n pbl-c1 route replace \
        default via 192.168.10.1 dev eth0

    echo "pbl-c1 기본 경로 복구 완료"
    ip -n pbl-c1 route
    ;;

capture-stop)
    require_owner

    pid="$(cat "${PIDFILE}" 2>/dev/null || true)"
    outfile="$(cat "${FILEFILE}" 2>/dev/null || true)"

    if [[ ! "${pid}" =~ ^[0-9]+$ ]]; then
        echo "ERROR: 정상적인 tcpdump PID가 아닙니다." >&2
        exit 1
    fi

    if kill -0 "${pid}" 2>/dev/null; then
        cmdline="$(tr '\0' ' ' < "/proc/${pid}/cmdline" 2>/dev/null || true)"

        if [[ "${cmdline}" != *"tcpdump"* ||
              "${cmdline}" != *"pbl-c1-h"* ]]; then
            echo "ERROR: PID가 예상한 tcpdump가 아닙니다." >&2
            exit 1
        fi

        kill -INT "${pid}" 2>/dev/null || true

        for _ in {1..30}; do
            kill -0 "${pid}" 2>/dev/null || break
            sleep 0.1
        done
    fi

    if [[ "${outfile}" != "${CAPDIR}/"*.pcap ]]; then
        echo "ERROR: 예상하지 못한 캡처 파일입니다." >&2
        exit 1
    fi

    chown "${REAL_USER}:pbl" "${outfile}"
    chmod 0640 "${outfile}"

    rm -rf "${LOCKDIR}"

    echo "CAPTURE STOPPED"
    echo "시각: $(date --iso-8601=seconds)"
    echo "파일: ${outfile}"

    if ! ip -n pbl-c1 route show default | grep -q .; then
        echo
        echo "WARNING: pbl-c1 기본 경로가 아직 제거된 상태입니다."
        echo "다음 명령으로 정상화하십시오:"
        echo "sudo /usr/local/sbin/pbl-w5-lab default-restore"
    fi
    ;;

*)
    echo "사용법:"
    echo "  sudo /usr/local/sbin/pbl-w5-lab check"
    echo "  sudo /usr/local/sbin/pbl-w5-lab macs"
    echo "  sudo /usr/local/sbin/pbl-w5-lab capture-start normal"
    echo "  sudo /usr/local/sbin/pbl-w5-lab capture-start no-default"
    echo "  sudo /usr/local/sbin/pbl-w5-lab capture-start recovered"
    echo "  sudo /usr/local/sbin/pbl-w5-lab routes"
    echo "  sudo /usr/local/sbin/pbl-w5-lab same-lan"
    echo "  sudo /usr/local/sbin/pbl-w5-lab web"
    echo "  sudo /usr/local/sbin/pbl-w5-lab default-remove"
    echo "  sudo /usr/local/sbin/pbl-w5-lab default-restore"
    echo "  sudo /usr/local/sbin/pbl-w5-lab capture-stop"
    exit 2
    ;;
esac
```

설치:

```bash
sudo chown root:root /usr/local/sbin/pbl-w5-lab
sudo chmod 0755 /usr/local/sbin/pbl-w5-lab
```

---

# 17. 실습자 sudo 권한 제한

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo visudo -f /etc/sudoers.d/pbl-week5
```

다음 내용을 저장한다.

```text
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab check
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab macs
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab capture-start normal
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab capture-start no-default
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab capture-start recovered
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab routes
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab same-lan
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab web
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab default-remove
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab default-restore
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w5-lab capture-stop
```

검사:

```bash
sudo chmod 0440 /etc/sudoers.d/pbl-week5
sudo visudo -cf /etc/sudoers.d/pbl-week5
```

실습자에게 호스트 전체 `sudo` 권한을 주지 않는다.

---

# 18. A의 최종 셋업 점검

### 실행 위치 → Linux 호스트

```bash
sudo /usr/local/sbin/pbl-w5-lab check
```

확인할 상태:

```text
pbl-c1
192.168.10.10/26
default via 192.168.10.1

pbl-c2
192.168.10.11/26
default via 192.168.10.1

pbl-r1
staff0  = 192.168.10.1/26
server0 = 192.168.20.1/27
ip_forward = 1

pbl-web
192.168.20.20/27
default via 192.168.20.1

TCP 8080 LISTEN
```

---

# 19. 웹 서비스만 재시작해야 할 때

네트워크 전체를 다시 만들 필요가 없는 경우다.

먼저 PID를 확인한다.

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo cat /run/pbl-w5-web.pid
```

PID가 예를 들어 `12345`라면 실제 명령을 확인한다.

```bash
sudo tr '\0' ' ' < /proc/12345/cmdline
echo
```

반드시 다음 실습 서버임을 확인한다.

```text
python3 -m http.server
8080
192.168.20.20
```

확인 후에만:

```bash
sudo kill 12345
```

다시 시작:

```bash
sudo sh -c '
nohup ip netns exec pbl-web \
  python3 -m http.server 8080 \
  --bind 192.168.20.20 \
  --directory /srv/pbl/week5/www \
  >/srv/pbl/week5/logs/web.log 2>&1 < /dev/null &
echo $! >/run/pbl-w5-web.pid
'
```

검증:

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

전체 Python 프로세스를 일괄 종료하지 않는다.

---

# 20. 실습 참여자 A·B·C·D 활동

이제부터는 A도 B·C·D와 동일하다.

각자 독립적으로:

```text
예측
→ 정상 상태 측정
→ 정상 PCAP
→ 기본 경로 제거
→ 같은 실험 반복
→ 장애 PCAP
→ 기본 경로 복원
→ 복구 PCAP
→ Wireshark 분석
→ 개인 설명
```

을 수행한다.

---

# 21. 개인 실습 1단계 — 상태와 실제 MAC 주소 기록

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
date --iso-8601=seconds
sudo /usr/local/sbin/pbl-w5-lab check
sudo /usr/local/sbin/pbl-w5-lab macs
```

MAC 주소는 문서에서 미리 정하지 않는다.

Linux가 실제 생성한 값을 사용한다.

개인 기록에 다음 형태로 작성한다.

```text
pbl-c1 MAC:
pbl-c2 MAC:
pbl-r1 staff0 MAC:
pbl-r1 server0 MAC:
pbl-web MAC:
```

---

# 22. 패킷을 보기 전에 먼저 예측

각자 팀원과 상의하기 전에 작성한다.

## 상황 A — pbl-c1 → pbl-c2

예측 항목:

```text
IP 목적지:
ARP 대상 IP:
Ethernet 목적지 MAC:
기본 게이트웨이 사용 여부:
```

## 상황 B — pbl-c1 → pbl-web

```text
IP 목적지:
ARP 대상 IP:
Ethernet 목적지 MAC:
기본 게이트웨이 사용 여부:
```

정답을 먼저 공유하지 않는다.

---

# 23. 정상 상태 캡처

## 23.1 캡처 시작

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab capture-start normal
```

`CAPTURE STARTED`를 확인한 후에만 요청을 발생시킨다.

---

# 24. 정상 상태 라우팅 테이블 기록

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab routes \
  | tee "$HOME/pbl-w5-normal-routes.txt"
```

중요하게 볼 부분은 `pbl-c1`이다.

예상:

```text
192.168.10.0/26 dev eth0 ...
default via 192.168.10.1 dev eth0
```

그리고:

```text
ip route get 192.168.10.11
```

결과에서는 직접 `eth0`을 사용하는 경로가 선택되어야 한다.

반면:

```text
ip route get 192.168.20.20
```

은 대략:

```text
192.168.20.20 via 192.168.10.1 dev eth0 ...
```

처럼 나타난다.

---

# 25. 같은 직원망의 pbl-c2와 통신

이번 가이드에서 “같은 직원망의 다른 클라이언트에 접속”은 **ICMP ping 통신**으로 수행한다.

HTTP 서비스를 c2에 추가하지 않는다.

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab same-lan
```

### 예상

```text
192.168.10.11 ... bytes from ...
```

와 비슷한 정상 ping 응답이 나타난다.

### 의미

`pbl-c1`과 `pbl-c2`는 직접 연결된 `192.168.10.0/26`에 속한다.

이 통신에는 기본 게이트웨이가 필요하지 않다.

---

# 26. 서버망의 웹 서버 접속

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab web
```

### 예상

`curl -v`에:

```text
Trying 192.168.20.20:8080...
Connected to 192.168.20.20 ...
```

와 HTTP 응답이 나타난다.

본문:

```text
PBL Week 5 OK
```

확인.

### 의미

서로 다른 두 IP 네트워크 사이 통신이 `pbl-r1`을 통해 정상적으로 수행되었다.

---

# 27. 정상 캡처 종료

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab capture-stop
```

출력되는 실제 `.pcap` 파일 이름을 기록한다.

---

# 28. 기본 경로 제거 실험 시작

새 캡처를 시작한다.

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab capture-start no-default
```

이제 `pbl-c1`의 기본 경로만 제거한다.

```bash
sudo /usr/local/sbin/pbl-w5-lab default-remove
```

---

# 29. 기본 경로 제거 상태 기록

```bash
sudo /usr/local/sbin/pbl-w5-lab routes \
  | tee "$HOME/pbl-w5-no-default-routes.txt"
```

`pbl-c1`에서는:

```text
192.168.10.0/26 dev eth0 ...
```

은 남아 있지만:

```text
default via 192.168.10.1
```

은 없어야 한다.

`192.168.20.20`에 대한 `ip route get`은 경로를 찾지 못한다.

환경에 따라 다음과 비슷한 오류가 나타날 수 있다.

```text
RTNETLINK answers: Network is unreachable
```

정확한 문구는 실제 출력을 기록한다.

---

# 30. 기본 경로 없이 같은 LAN 통신 반복

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab same-lan
```

### 예상

`pbl-c2` ping은 여전히 성공한다.

### 왜 그런가

`192.168.10.11`은 다음 직접 연결 경로와 일치한다.

```text
192.168.10.0/26 dev eth0
```

따라서 `default`가 없어도 된다.

---

# 31. 기본 경로 없이 서버망 통신 반복

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab web
```

### 예상

실패한다.

`curl`의 정확한 오류 문구는 환경에 따라 조금 다를 수 있다.

중요한 근거는 단순히:

```text
curl이 실패했다
```

가 아니다.

함께 확인한:

```text
pbl-c1의 default route 없음
+
ip route get 192.168.20.20 실패
+
해당 시점의 PCAP
```

이다.

---

# 32. 장애 상태 캡처 종료

```bash
sudo /usr/local/sbin/pbl-w5-lab capture-stop
```

파일 이름을 기록한다.

---

# 33. 기본 경로 복구

### 실행 위치 → Linux 호스트 / 본인 SSH 세션

```bash
sudo /usr/local/sbin/pbl-w5-lab default-restore
```

정상:

```text
default via 192.168.10.1 dev eth0
```

가 다시 표시되어야 한다.

---

# 34. 복구 후 재검증

캡처 시작:

```bash
sudo /usr/local/sbin/pbl-w5-lab capture-start recovered
```

라우팅 상태 저장:

```bash
sudo /usr/local/sbin/pbl-w5-lab routes \
  | tee "$HOME/pbl-w5-recovered-routes.txt"
```

같은 LAN:

```bash
sudo /usr/local/sbin/pbl-w5-lab same-lan
```

웹 서버:

```bash
sudo /usr/local/sbin/pbl-w5-lab web
```

종료:

```bash
sudo /usr/local/sbin/pbl-w5-lab capture-stop
```

### 복구 완료 기준

단순히 경로를 추가했다는 사실이 아니라:

```text
기본 경로 복구
→ 동일 요청 재실행
→ HTTP 정상 응답
→ 복구 PCAP 확보
```

까지 완료해야 한다.

---

# 35. PCAP을 Windows PC로 다운로드

Linux에서 기록한 실제 파일명을 사용한다.

예:

```text
/srv/pbl/week5/captures/pbl-a-normal-....pcap
/srv/pbl/week5/captures/pbl-a-no-default-....pcap
/srv/pbl/week5/captures/pbl-a-recovered-....pcap
```

### 실행 위치 → 개인 Windows PowerShell

```powershell
New-Item -ItemType Directory -Force "$HOME\pbl-week5"
```

예:

```powershell
scp pbl-a@<SERVER_MANAGEMENT_IP>:/srv/pbl/week5/captures/pbl-a-normal-20261006-180000.pcap "$HOME\pbl-week5\"
```

장애/복구 파일도 같은 방법으로 가져온다.

라우팅 테이블 파일:

```powershell
scp pbl-a@<SERVER_MANAGEMENT_IP>:~/pbl-w5-normal-routes.txt "$HOME\pbl-week5\"
scp pbl-a@<SERVER_MANAGEMENT_IP>:~/pbl-w5-no-default-routes.txt "$HOME\pbl-week5\"
scp pbl-a@<SERVER_MANAGEMENT_IP>:~/pbl-w5-recovered-routes.txt "$HOME\pbl-week5\"
```

본인의 실제 계정과 파일명을 사용한다.

---

# 36. Wireshark 분석 — 같은 LAN

정상 PCAP을 연다.

Wireshark의 Display Filter는 이미 수집된 패킷에서 원하는 필드를 기준으로 표시 대상을 좁히는 기능이다.

## ARP 확인

필터:

```text
arp
```

`pbl-c1`이 `pbl-c2`의 MAC을 알아내는 ARP Request를 찾는다.

Packet Details에서 ARP의 Target Protocol Address를 확인한다.

예상 대상 IP:

```text
192.168.10.11
```

즉:

```text
누가 192.168.10.11을 가지고 있는가?
```

에 해당하는 ARP이다.

실제 표시 문구는 Wireshark 버전과 언어 설정에 따라 달라질 수 있으므로 **필드 값**을 기준으로 확인한다.

---

# 37. 같은 LAN Ethernet 목적지 확인

필터:

```text
icmp && ip.dst == 192.168.10.11
```

`pbl-c1 → pbl-c2` 방향 Echo Request 하나를 선택한다.

Packet Details:

```text
Ethernet II
Internet Protocol Version 4
```

를 각각 펼친다.

기록:

```text
Frame 번호:
Ethernet Source:
Ethernet Destination:
IPv4 Source:
IPv4 Destination:
```

정상 예상:

```text
IPv4 Source      = 192.168.10.10
IPv4 Destination = 192.168.10.11

Ethernet Destination = 실제 pbl-c2 MAC
```

Wireshark에는 Ethernet 목적지 주소를 나타내는 `eth.dst` 필드가 있다.

실제 MAC은 `pbl-w5-lab macs`에서 조회한 값과 비교한다.

---

# 38. Wireshark 분석 — 원격 웹 서버

우선 다음으로 웹 연결을 찾는다.

```text
ip.addr == 192.168.20.20 && tcp.port == 8080
```

클라이언트가 연결을 시작하는 첫 SYN을 더 정확히 찾으려면:

```text
ip.dst == 192.168.20.20
&& tcp.dstport == 8080
&& tcp.flags.syn == 1
&& tcp.flags.ack == 0
```

해당 프레임을 클릭한다.

---

# 39. 가장 중요한 비교

Packet Details에서:

```text
Ethernet II
Internet Protocol Version 4
```

를 펼친다.

정상적으로는 다음 관계를 관찰한다.

```text
Ethernet Destination
    = pbl-r1 staff0 실제 MAC

IPv4 Destination
    = 192.168.20.20
```

즉:

```text
MAC 목적지는 라우터
IP 목적지는 웹 서버
```

이다.

이를 혼동하지 않는 것이 이번 주의 핵심이다.

실제 MAC을 알고 있다면 다음과 같이도 필터링할 수 있다.

```text
eth.dst == <실제_R1_staff0_MAC>
&& ip.dst == 192.168.20.20
```

`<실제_R1_staff0_MAC>`은 글자를 그대로 입력하는 것이 아니라 실제 조회값으로 교체한다.

---

# 40. 원격 목적지의 ARP 대상 확인

정상 PCAP에서 다시:

```text
arp
```

를 사용한다.

웹 요청 직전에 발생한 `pbl-c1`의 ARP를 찾는다.

예상 ARP 대상 IP는:

```text
192.168.10.1
```

이다.

다음이 아니다.

```text
192.168.20.20
```

### 이유

`pbl-c1`은 `192.168.20.20`을 자신의 직접 연결 LAN에 있는 장치라고 판단하지 않는다.

따라서 웹 서버 MAC을 직원망에서 찾지 않는다.

대신 다음 홉인:

```text
192.168.10.1
```

의 MAC을 알아낸다.

---

# 41. 정상 상태 비교표

본인의 실제 캡처를 채운다.

| 항목 | pbl-c1 → pbl-c2 | pbl-c1 → pbl-web |
|---|---|---|
| IP 목적지 | `192.168.10.11` | `192.168.20.20` |
| 같은 서브넷인가 |  |  |
| 선택된 경로 |  |  |
| ARP 대상 IP |  |  |
| Ethernet 목적지 MAC |  |  |
| 기본 게이트웨이 사용 |  |  |
| 근거 Frame 번호 |  |  |

표의 MAC은 문서의 예시값이 아니라 실제 값으로 작성한다.

---

# 42. 응답 패킷도 확인

정상 웹 통신에서 서버 → 클라이언트 방향 패킷을 찾는다.

필터:

```text
ip.src == 192.168.20.20
&& ip.dst == 192.168.10.10
&& tcp.srcport == 8080
```

`pbl-c1` 쪽 캡처에서 정상적으로 돌아온 프레임을 선택한다.

예상:

```text
IPv4 Source      = 192.168.20.20
IPv4 Destination = 192.168.10.10

Ethernet Source      = pbl-r1 staff0 MAC
Ethernet Destination = pbl-c1 MAC
```

### 이 패킷이 알려주는 것

서버의 응답이 직원망으로 들어올 때도 R1을 거쳐 왔음을 확인할 수 있다.

다만 이번 캡처 지점은 `pbl-c1` 쪽이므로 서버망 구간에서 사용했던 Ethernet 주소까지 이 PCAP 하나만으로 직접 볼 수는 없다.

그 비교는 6주차의 라우터 양쪽 캡처에서 수행한다.

---

# 43. 기본 경로 제거 PCAP 분석

`no-default` PCAP을 연다.

먼저 같은 LAN ping을 찾는다.

```text
icmp && ip.addr == 192.168.10.11
```

정상적으로 보일 것이다.

반면 웹 요청:

```text
ip.addr == 192.168.20.20 && tcp.port == 8080
```

은 해당 실패 시점에 전송 패킷이 나타나지 않을 수 있다.

그렇다고 다음처럼 단독으로 결론 내리면 안 된다.

> “패킷이 안 보였으니 중간 라우터에서 버려졌다.”

이번 실험에서 더 강한 근거는:

```text
1. pbl-c1에서 default route가 제거되어 있음
2. 192.168.10.0/26 직접 연결 경로는 남아 있음
3. ip route get 192.168.10.11은 성공
4. ip route get 192.168.20.20은 실패
5. 같은 LAN ping은 성공
6. 서버망 HTTP 요청은 실패
7. 해당 HTTP 패킷이 pbl-c1-h에서 전송되지 않음
```

의 조합이다.

---

# 44. 복구 PCAP 분석

`recovered` 파일에서도 다시:

```text
ip.dst == 192.168.20.20
&& tcp.dstport == 8080
&& tcp.flags.syn == 1
&& tcp.flags.ack == 0
```

을 사용한다.

정상 상태와 똑같이:

```text
IP 목적지 = 192.168.20.20
Ethernet 목적지 = pbl-r1 staff0 MAC
```

인지 확인한다.

이것이 복구 검증이다.

---

# 45. 반드시 함께 제출할 라우팅 근거

각 참여자는 최소 다음 세 상태의 라우팅 결과를 제출한다.

```text
정상
기본 경로 제거
기본 경로 복구
```

특히 비교할 것:

## 정상

```text
192.168.10.0/26 dev eth0 ...
default via 192.168.10.1 dev eth0
```

## 장애

```text
192.168.10.0/26 dev eth0 ...
```

`default` 없음.

## 복구

```text
192.168.10.0/26 dev eth0 ...
default via 192.168.10.1 dev eth0
```

---

# 46. 이번 주 판단에서 주의할 표현

잘못된 설명:

> 웹 서버 IP가 192.168.20.20이지만 게이트웨이를 거치므로 목적지 IP가 192.168.10.1로 바뀐다.

수정된 설명:

> pbl-c1은 192.168.20.20이 원격 네트워크임을 판단하고 기본 경로의 다음 홉인 192.168.10.1을 사용한다. 따라서 직원망에서 전송하는 Ethernet 프레임의 목적지 MAC은 R1의 직원망 측 MAC이지만, IPv4 목적지 주소는 192.168.20.20으로 유지된다.

잘못된 설명:

> 기본 게이트웨이가 없으면 네트워크 통신을 전혀 할 수 없다.

수정:

> 기본 게이트웨이가 없어도 직접 연결 경로에 속하는 192.168.10.11과는 통신할 수 있다. 다른 네트워크인 192.168.20.20으로 갈 경로가 없기 때문에 서버망 통신이 실패한다.

---

# 47. 장애 발생 시 점검 순서

## 증상: pbl-c1 → pbl-c2도 실패

먼저:

### 실행 위치 → Linux 호스트

```bash
sudo ip -n pbl-c1 -br addr
sudo ip -n pbl-c2 -br addr
ip -o link show master pbl-br-staff
```

확인:

```text
192.168.10.10/26
192.168.10.11/26
pbl-c1-h
pbl-c2-h
```

기본 게이트웨이를 먼저 의심하지 않는다.

둘은 같은 LAN이다.

---

## 증상: c2는 되는데 웹 서버가 안 됨

확인 순서:

```bash
sudo ip -n pbl-c1 route
sudo ip netns exec pbl-c1 ip route get 192.168.20.20

sudo ip -n pbl-r1 route
sudo ip netns exec pbl-r1 sysctl net.ipv4.ip_forward

sudo ip -n pbl-web route
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
```

각각 질문한다.

```text
c1은 R1으로 가는 경로를 아는가?
R1은 두 망을 직접 연결하고 있는가?
R1 forwarding은 켜져 있는가?
웹 서버는 직원망으로 돌아갈 기본 경로를 갖는가?
TCP 8080 서비스가 실제 LISTEN 중인가?
```

---

## 증상: ping은 되는데 HTTP가 Connection refused

라우팅만으로 웹 서비스 정상이라고 결론 내리지 않는다.

확인:

```bash
sudo ip netns exec pbl-web ss -lntp | grep ':8080'
sudo cat /srv/pbl/week5/logs/web.log
```

`Connection refused`는 대상까지 도달했지만 해당 TCP 포트에서 서비스가 수신하지 않는 상황과 연결될 수 있다.

---

## 증상: HTTP 요청이 시간 초과

패킷이 어디까지 갔는지 캡처와 라우팅 테이블을 함께 확인한다.

시간 초과라는 사실 하나만으로:

```text
웹 서버 중단
라우터 문제
방화벽 문제
```

중 하나를 확정하지 않는다.

---

## 증상: Wireshark에서 HTTP가 안 보임

먼저:

```text
tcp.port == 8080
```

으로 TCP 자체를 확인한다.

필요하면 Wireshark의 `Decode As...`로 TCP 8080을 HTTP로 해석한다.

HTTP 디코딩이 안 보인다는 이유만으로 연결 실패라고 판단하지 않는다.

---

# 48. 기본 경로를 복구하지 않고 다음 사람이 시작했을 때

### 실행 위치 → Linux 호스트

```bash
sudo /usr/local/sbin/pbl-w5-lab check
```

다음 경고가 있으면:

```text
WARNING: pbl-c1 기본 경로가 없습니다.
```

복원:

```bash
sudo /usr/local/sbin/pbl-w5-lab default-restore
```

재확인:

```bash
sudo /usr/local/sbin/pbl-w5-lab check
```

---

# 49. 셋업 담당자 A — 하루 종료 점검

### 실행 위치 → Linux 호스트

```bash
sudo /usr/local/sbin/pbl-w5-lab check
```

캡처 목록:

```bash
ls -lh /srv/pbl/week5/captures/
```

웹 로그:

```bash
tail -n 30 /srv/pbl/week5/logs/web.log
```

디스크 사용량:

```bash
du -sh /srv/pbl/week5/captures
```

중요한 최종 상태:

```text
pbl-c1 default via 192.168.10.1
pbl-c2 default via 192.168.10.1
pbl-web default via 192.168.20.1
pbl-r1 ip_forward = 1
pbl-web TCP 8080 LISTEN
```

---

# 50. 5주차 환경 제거 스크립트

공용 서버에서 가상 환경을 완전히 제거해야 할 때만 사용한다.

PCAP과 백업은 삭제하지 않는다.

## 저장

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w5-teardown
```

전체 코드:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ ${EUID} -ne 0 ]]; then
    echo "ERROR: sudo 또는 root로 실행해야 합니다." >&2
    exit 1
fi

BASE="/srv/pbl/week5"
WEB_PID="/run/pbl-w5-web.pid"
LOCKDIR="/run/lock/pbl-w5-lab"

ns_exists() {
    ip netns list | awk '{print $1}' | grep -Fxq "$1"
}

if [[ -d "${LOCKDIR}" ]]; then
    echo "ERROR: 현재 참여자 실습이 진행 중입니다." >&2
    echo -n "사용자: " >&2
    cat "${LOCKDIR}/owner" 2>/dev/null || echo unknown >&2
    echo "capture-stop 후 다시 실행하십시오." >&2
    exit 1
fi

echo "[1/4] 웹 서버 확인"

if [[ -f "${WEB_PID}" ]]; then
    pid="$(cat "${WEB_PID}" 2>/dev/null || true)"

    if [[ "${pid}" =~ ^[0-9]+$ ]] && \
       kill -0 "${pid}" 2>/dev/null; then

        cmdline="$(tr '\0' ' ' < "/proc/${pid}/cmdline" 2>/dev/null || true)"

        if [[ "${cmdline}" == *"python3"* &&
              "${cmdline}" == *"-m http.server"* &&
              "${cmdline}" == *"8080"* ]]; then
            kill "${pid}" 2>/dev/null || true
            sleep 0.5
        else
            echo "ERROR: 웹 PID가 예상한 프로세스가 아닙니다." >&2
            echo "${cmdline}" >&2
            exit 1
        fi
    fi

    rm -f "${WEB_PID}"
fi

echo "[2/4] 남은 네임스페이스 프로세스 확인"

for ns in pbl-c1 pbl-c2 pbl-web pbl-r1; do
    if ns_exists "${ns}"; then
        pids="$(ip netns pids "${ns}" 2>/dev/null | xargs || true)"

        if [[ -n "${pids}" ]]; then
            echo "ERROR: ${ns}에 실행 중인 프로세스가 남아 있습니다." >&2
            echo "PID: ${pids}" >&2
            echo "강제로 종료하지 않고 중단합니다." >&2
            exit 1
        fi
    fi
done

echo "[3/4] 프로젝트 네임스페이스 제거"

for ns in pbl-c1 pbl-c2 pbl-web pbl-r1; do
    if ns_exists "${ns}"; then
        ip netns del "${ns}"
    fi
done

echo "[4/4] 프로젝트 bridge와 veth 제거"

for dev in \
    pbl-c1-h \
    pbl-c2-h \
    pbl-web-h \
    pbl-r1-st-h \
    pbl-r1-sv-h
do
    if ip link show "${dev}" >/dev/null 2>&1; then
        ip link del "${dev}"
    fi
done

for br in pbl-br-staff pbl-br-server; do
    if ip link show "${br}" >/dev/null 2>&1; then
        ip link del "${br}"
    fi
done

echo
echo "5주차 실습 네트워크 제거 완료."
echo "삭제하지 않은 항목:"
echo "  ${BASE}/backups"
echo "  ${BASE}/captures"
echo "  ${BASE}/logs"
echo "  ${BASE}/www"
echo
echo "호스트 실제 NIC, 관리 IP, 기본 경로는 변경하지 않았습니다."
```

설치:

```bash
sudo chmod 0755 /usr/local/sbin/pbl-w5-teardown
```

실행:

```bash
sudo /usr/local/sbin/pbl-w5-teardown
```

### 주의

이 스크립트는 5주차 환경을 제거할 뿐 **4주차 환경을 자동 복원하지 않는다.**

4주차로 돌아가야 한다면 4주차의 정상 구성 절차를 사용한다.

변경 전 백업 파일은 비교 자료이지 자동 복원 스크립트가 아니다.

---

# 51. 5주차 정상 상태로 다시 초기화

잘못된 실험 상태가 많이 남았거나 정확한 상태를 알 수 없다면 A가:

### 실행 위치 → Linux 호스트
### 관리자 권한 필요

```bash
sudo /usr/local/sbin/pbl-w5-setup
```

을 다시 실행한다.

이 과정은 새 백업을 만든 뒤 프로젝트 자원을 다시 구성한다.

알 수 없는 프로세스나 예상하지 못한 bridge 포트가 있으면 강제로 제거하지 않고 중단하도록 설계했다.

---

# 52. 개인 제출물

A·B·C·D 각각 제출한다.

## ① 실제 환경 정보

```text
참여자:
실습 계정:
실험 날짜:
정상 캡처 파일:
기본 경로 제거 캡처 파일:
복구 캡처 파일:

pbl-c1 실제 MAC:
pbl-c2 실제 MAC:
pbl-r1 staff0 실제 MAC:
pbl-r1 server0 실제 MAC:
pbl-web 실제 MAC:
```

## ② 실험 전 예측

```text
같은 LAN에서 예상 ARP 대상:
같은 LAN에서 예상 Ethernet 목적지:

서버망 통신에서 예상 ARP 대상:
서버망 통신에서 예상 Ethernet 목적지:
서버망 통신에서 예상 IP 목적지:
```

## ③ 정상 라우팅 테이블

`pbl-w5-normal-routes.txt`

## ④ 기본 경로 제거 라우팅 테이블

`pbl-w5-no-default-routes.txt`

## ⑤ 복구 라우팅 테이블

`pbl-w5-recovered-routes.txt`

## ⑥ 같은 LAN 근거 패킷

```text
Frame 번호:
ARP 대상 IP:
Ethernet Source:
Ethernet Destination:
IPv4 Source:
IPv4 Destination:
```

## ⑦ 원격 웹 서버 근거 패킷

```text
Frame 번호:
ARP 대상 IP:
Ethernet Source:
Ethernet Destination:
IPv4 Source:
IPv4 Destination:
TCP Destination Port:
```

## ⑧ 응답 근거 패킷

```text
Frame 번호:
Ethernet Source:
Ethernet Destination:
IPv4 Source:
IPv4 Destination:
TCP Source Port:
```

## ⑨ 장애 분석

```text
관찰 사실:
원인 후보:
원인 판단 근거:
추가로 확인한 명령:
PCAP에서 확인한 내용:
확인하지 못한 사항:
```

## ⑩ 복구 검증

```text
수정한 설정:
복구 후 route:
복구 후 HTTP 결과:
복구 PCAP 근거 Frame:
```

---

# 53. 개인 이해도 질문

팀 토의 전에 각자 답한다.

### 질문 1

`pbl-c1`의 다음 두 경로는 각각 어떤 역할을 하는가?

```text
192.168.10.0/26 dev eth0
default via 192.168.10.1 dev eth0
```

### 질문 2

`pbl-c1 → pbl-c2`에서 ARP 대상이 왜 `192.168.10.1`이 아닌가?

### 질문 3

`pbl-c1 → pbl-web`에서는 왜 `192.168.20.20`의 MAC 주소를 직원망에서 ARP하지 않는가?

### 질문 4

웹 서버로 가는 TCP SYN에서:

```text
IPv4 Destination = 192.168.20.20
Ethernet Destination = R1 MAC
```

이 동시에 성립하는 이유는 무엇인가?

### 질문 5

라우터가 해당 패킷을 전달할 때 IPv4 목적지를 `192.168.20.1`로 바꾸는가?

그렇지 않다면 어떤 정보가 홉마다 달라지는가?

### 질문 6

`pbl-c1`의 default route를 삭제해도 `pbl-c2` ping이 성공한 이유는 무엇인가?

### 질문 7

같은 상태에서 웹 접속이 실패한 이유는 무엇인가?

### 질문 8

웹 서버의 기본 게이트웨이가 잘못되어 있다면 요청이 서버에 도착했더라도 어떤 문제가 발생할 수 있는가?

### 질문 9

PCAP에서 웹 요청 패킷이 없다는 사실만으로 “R1에서 패킷을 버렸다”고 결론 내릴 수 없는 이유는 무엇인가?

### 질문 10

이번 환경에서 NAT를 사용하지 않았다는 것은 `pbl-c1 → pbl-web` 패킷의 Source/Destination IP 관점에서 어떤 의미인가?

---

# 54. 정답 확인용 핵심 논리

이 부분은 각자가 먼저 답한 뒤 확인한다.

## 같은 직원망

```text
pbl-c1
192.168.10.10
        │
        │ 목적지 192.168.10.11
        ▼
직접 연결 경로 192.168.10.0/26 선택
        │
        ▼
ARP: 192.168.10.11의 MAC은?
        │
        ▼
Ethernet Destination = pbl-c2 MAC
```

## 서버망

```text
pbl-c1
192.168.10.10
        │
        │ 목적지 192.168.20.20
        ▼
직접 연결 경로와 불일치
        │
        ▼
default via 192.168.10.1 선택
        │
        ▼
ARP: 192.168.10.1의 MAC은?
        │
        ▼
Ethernet Destination = pbl-r1 staff0 MAC
IPv4 Destination     = 192.168.20.20
```

## 기본 경로 제거

```text
192.168.10.11
→ 직접 연결 경로 존재
→ 통신 가능

192.168.20.20
→ 직접 연결 경로 없음
→ default 없음
→ 사용할 경로 없음
→ 통신 실패
```

---

# 55. 5주차 최종 체크리스트

셋업 담당 A:

- [ ] 변경 전 관리망과 프로젝트 상태를 백업했다.
- [ ] `pbl-c1`, `pbl-c2`, `pbl-web`, `pbl-r1`을 확인했다.
- [ ] `pbl-br-staff`와 `pbl-br-server`를 구성했다.
- [ ] `pbl-web`의 `192.168.10.20/26`을 제거했다.
- [ ] `pbl-web=192.168.20.20/27`을 설정했다.
- [ ] `pbl-r1`에 `192.168.10.1/26`, `192.168.20.1/27`을 설정했다.
- [ ] IP forwarding은 `pbl-r1` 네임스페이스에서만 활성화했다.
- [ ] Linux 호스트의 관리 NIC와 기본 경로를 변경하지 않았다.
- [ ] NAT를 구성하지 않았다.
- [ ] 직원망과 서버망 사이의 우회 연결이 없음을 확인했다.
- [ ] TCP 8080 서비스를 정상화했다.
- [ ] c1→c2, c1→web, web→c1 통신을 자체 점검했다.

A·B·C·D 각자:

- [ ] 실제 MAC 주소를 직접 조회했다.
- [ ] 패킷을 보기 전에 결과를 예측했다.
- [ ] 정상 상태 라우팅 테이블을 저장했다.
- [ ] 요청 전에 정상 PCAP 캡처를 시작했다.
- [ ] `pbl-c1 → pbl-c2`를 직접 실행했다.
- [ ] `pbl-c1 → pbl-web:8080`을 직접 실행했다.
- [ ] 같은 LAN에서 ARP 대상이 c2임을 확인했다.
- [ ] 원격 LAN에서 ARP 대상이 R1임을 확인했다.
- [ ] 웹 패킷의 IP 목적지가 `192.168.20.20`임을 확인했다.
- [ ] 같은 패킷의 Ethernet 목적지가 R1 직원망 MAC임을 확인했다.
- [ ] pbl-c1의 기본 경로를 직접 제거했다.
- [ ] 기본 경로가 없어도 c2 통신이 되는 것을 확인했다.
- [ ] 기본 경로가 없으면 웹 서버 통신이 실패하는 것을 확인했다.
- [ ] 장애 상태 라우팅 테이블과 PCAP을 저장했다.
- [ ] 기본 경로를 복구했다.
- [ ] 같은 HTTP 요청으로 정상화를 재검증했다.
- [ ] 복구 PCAP을 저장했다.
- [ ] 서버의 응답에도 반환 경로가 필요하다고 설명했다.
- [ ] 관찰 사실과 자신의 추정을 구분해 작성했다.

---

# 56. 이번 주의 핵심 문장

5주차에서 가장 중요하게 기억할 문장은 다음이다.

```text
같은 LAN의 목적지
→ 목적지 호스트의 MAC을 찾는다.

다른 LAN의 목적지
→ 다음 홉인 기본 게이트웨이의 MAC을 찾는다.

그러나 IP 목적지는 최종 목적지 IP로 유지된다.
```

그리고 장애 분석에서는:

```text
패킷
+
라우팅 테이블
+
실행 결과
```

를 함께 본다.

하나의 증거만으로 원인을 단정하지 않는다.