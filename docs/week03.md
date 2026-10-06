# 3주차 실습 가이드
## IP 주소와 서브넷 설계

프로젝트: **「소규모 사내망 설계 및 Wireshark 기반 통신 검증·장애 진단」**

---

# 1. 이번 주의 위치

이번 주에는 아직 라우터나 서버망을 만들지 않는다.

현재 실습 환경은 다음과 같다.

```text
개인 Windows PC
      │
      │ SSH / SCP
      │ 실제 관리망
      ▼
공용 Ubuntu 서버
      │
      └─ pbl-br-staff
            │
            ├─ pbl-c1
            │   192.168.10.10/26
            │
            ├─ pbl-c2
            │   192.168.10.11/26
            │
            └─ pbl-web
                192.168.10.20/26
                TCP 8080
```

**기본 게이트웨이는 없다.**

1주차부터 `pbl-c1`과 `pbl-web`은 공용 서버 안의 단일 LAN에서 통신하도록 구성했고, 요청은 Windows PC가 아니라 `pbl-c1`에서 발생시키는 방식을 사용했다.

3주차에도 이 구조를 유지한다.

이번 주에 **하지 않는 것**은 다음과 같다.

- `192.168.20.0/27` 서버망 실제 생성
- `pbl-web`의 `192.168.20.20/27` 이동
- 라우터 `pbl-r1` 생성
- 기본 게이트웨이 설정
- DHCP 서비스 시작
- DNS 서비스 시작
- 고의적인 잘못된 마스크 실험

서버망으로 실제 이동하고 라우팅을 구성하는 것은 **5주차**다.

---

# 2. 이번 주 학습 목표

A·B·C·D 전원이 이번 주 종료 시 다음을 수행할 수 있어야 한다.

1. IPv4 주소가 32비트이며 8비트씩 네 옥텟으로 표현된다는 것을 설명한다.
2. `/26`, `/27`을 서브넷 마스크로 변환한다.
3. 네트워크 주소, 브로드캐스트 주소, 사용 가능한 호스트 범위를 계산한다.
4. `/26`과 `/27`의 일반적인 사용 가능 호스트 수를 계산한다.
5. 두 IP가 같은 서브넷인지 마스크를 이용해 판단한다.
6. 필요한 호스트 수를 보고 `/26`과 `/27` 중 적절한 크기를 선택한다.
7. 고정 주소, DHCP 동적 할당 범위, 예약 주소, DHCP 동적 할당 제외 주소를 구분한다.
8. 자신이 계산한 주소를 실제 `pbl-c1`에 적용하고 Linux의 연결 경로를 확인한다.
9. 실제 패킷에서 자신이 설정한 출발지 IP를 확인한다.
10. 다음 질문에 계산과 관찰 근거를 이용해 답한다.

> **“이 주소가 이 마스크에서 왜 192.168.10.20과 같은 네트워크인가?”**

IPv4 CIDR 표기에서는 `/` 뒤 숫자가 32비트 주소 중 네트워크 프리픽스로 사용하는 상위 비트 수를 의미한다. RFC 4632도 `/26`은 64개 주소, `/27`은 32개 주소로 구성되는 블록으로 설명한다.

---

# 3. 개념 학습

## 3.1 IPv4는 32비트다

예를 들어:

```text
192.168.10.10
```

은 네 개의 숫자로 나뉜다.

```text
192  .  168  .  10  .  10
```

각 부분은 **옥텟(octet)**이라고 하며 8비트다.

따라서:

```text
8비트 × 4 = 32비트
```

이다.

`192.168.10.10`을 2진수로 쓰면:

```text
192       168       10        10

11000000.10101000.00001010.00001010
```

이번 주에는 이 32비트 중 어디까지가 **네트워크 부분**이고 어디부터가 **호스트 부분**인지를 판단한다.

---

# 3.2 `/26`의 의미

`/26`은 앞의 26비트가 네트워크 부분이라는 뜻이다.

마스크를 2진수로 쓰면:

```text
11111111.11111111.11111111.11000000
```

10진수로 바꾸면:

```text
255.255.255.192
```

이다.

즉:

```text
/26
=
255.255.255.192
```

마지막 옥텟만 보면:

```text
마스크 192 = 11000000
```

앞의 2비트까지 네트워크 부분이고 뒤의 6비트가 호스트 부분이다.

```text
NNHHHHHH
```

따라서 호스트 비트 수는:

```text
32 - 26 = 6비트
```

전체 주소 수는:

```text
2^6 = 64
```

이다.

일반적인 브로드캐스트 LAN에서 네트워크 주소 1개와 브로드캐스트 주소 1개를 제외하면:

```text
64 - 2 = 62
```

따라서 `/26`의 일반적인 사용 가능 호스트 수는 **62개**다.

---

# 3.3 `/26`을 2진수로 직접 확인

현재 클라이언트:

```text
192.168.10.10/26
```

마지막 옥텟만 계산한다.

```text
IP 10       = 00001010
Mask 192    = 11000000
              --------
AND           00000000
```

따라서 네트워크 주소의 마지막 옥텟은 `0`이다.

```text
192.168.10.0/26
```

웹 서버 `192.168.10.20`도 계산한다.

```text
IP 20       = 00010100
Mask 192    = 11000000
              --------
AND           00000000
```

역시:

```text
192.168.10.0
```

이 나온다.

즉:

```text
192.168.10.10/26 → 네트워크 192.168.10.0
192.168.10.20/26 → 네트워크 192.168.10.0
```

이므로 두 주소는 같은 `/26` 네트워크에 속한다.

---

# 3.4 `/26`을 블록 크기로 계산

2진수 계산에 익숙해진 뒤에는 블록 크기로 빠르게 검산할 수 있다.

마지막 마스크 옥텟은:

```text
192
```

이므로:

```text
256 - 192 = 64
```

블록 크기는 64다.

따라서 마지막 옥텟의 네트워크 시작점은:

```text
0
64
128
192
```

이다.

`192.168.10.10`은 `0~63` 구간 안에 있다.

따라서:

```text
네트워크 주소       192.168.10.0
첫 사용 가능 주소   192.168.10.1
마지막 사용 가능    192.168.10.62
브로드캐스트        192.168.10.63
```

이다.

---

# 3.5 `/27`의 의미

`/27`은 앞의 27비트가 네트워크 부분이다.

2진수 마스크:

```text
11111111.11111111.11111111.11100000
```

10진수 마스크:

```text
255.255.255.224
```

즉:

```text
/27
=
255.255.255.224
```

호스트 비트는:

```text
32 - 27 = 5비트
```

전체 주소 수:

```text
2^5 = 32
```

일반적인 LAN에서 사용 가능한 호스트 수:

```text
32 - 2 = 30
```

따라서 `/27`에서는 일반적으로 **30개 호스트 주소**를 사용할 수 있다.

---

# 3.6 `/27` 2진수 계산

향후 서버망에서 사용할:

```text
192.168.20.20/27
```

을 계산한다.

마지막 옥텟:

```text
IP 20       = 00010100
Mask 224    = 11100000
              --------
AND           00000000
```

따라서 네트워크 주소는:

```text
192.168.20.0/27
```

이다.

브로드캐스트에서는 호스트 5비트를 모두 `1`로 만든다.

```text
00011111 = 31
```

따라서:

```text
네트워크 주소       192.168.20.0
첫 사용 가능 주소   192.168.20.1
마지막 사용 가능    192.168.20.30
브로드캐스트        192.168.20.31
```

이다.

---

# 3.7 `/27` 블록 크기 계산

마스크 마지막 옥텟은:

```text
224
```

이다.

따라서:

```text
256 - 224 = 32
```

블록 크기는 32다.

네트워크 시작점은:

```text
0
32
64
96
128
160
192
224
```

가 된다.

`192.168.20.0/27`은 첫 번째 블록이므로:

```text
0 ~ 31
```

을 사용한다.

---

# 3.8 이번 주 사용할 계산 공식

`/26`, `/27`과 같은 일반적인 LAN 서브넷에서는:

```text
호스트 비트 수
= 32 - 프리픽스 길이
```

```text
전체 주소 수
= 2^(호스트 비트 수)
```

```text
일반적인 사용 가능 호스트 수
= 전체 주소 수 - 2
```

로 계산한다.

따라서:

| 프리픽스 | 마스크 | 호스트 비트 | 전체 주소 | 일반적인 사용 가능 호스트 |
|---|---|---:|---:|---:|
| /26 | 255.255.255.192 | 6 | 64 | 62 |
| /27 | 255.255.255.224 | 5 | 32 | 30 |

---

# 3.9 `/31`과 `/32` 주의

**`2^(호스트 비트)-2`를 모든 프리픽스에 무조건 적용한다고 외우면 안 된다.**

`/31`은 point-to-point 링크에서 두 주소를 호스트 주소로 사용할 수 있는 특별한 경우가 RFC 3021에 정의되어 있다. `/32`는 하나의 IPv4 주소를 나타내며 호스트 라우트 등에 사용된다.

따라서 이번 주의:

```text
전체 주소 - 네트워크 주소 - 브로드캐스트 주소
```

계산 규칙은 **일반적인 `/26`, `/27` LAN 계산을 학습하기 위한 규칙**으로 사용한다.

`/31`, `/32`의 실제 활용은 이번 계산 문제의 범위에서 제외한다.

---

# 3.10 같은 서브넷인지 판단하는 원칙

두 주소가 같은 서브넷인지 판단할 때는 단순히:

```text
192.168.10.x니까 같다
```

라고 판단하면 안 된다.

올바른 기준은 각각:

```text
IP 주소 AND 서브넷 마스크
```

를 계산하여 **네트워크 주소가 같은지 비교하는 것**이다.

예:

```text
192.168.10.12/26
192.168.10.20/26
```

둘 다 계산 결과:

```text
192.168.10.0/26
```

이면 같은 서브넷이다.

반대로 주소 앞부분이 비슷해 보여도 마스크가 다르면 판단 결과가 달라질 수 있다.

그 차이는 4주차의 통제된 마스크 변경 실험에서 직접 확인한다.

---

# 4. 주소를 어떤 용도로 사용할 것인가

주소 범위를 계산한 뒤에는 모든 사용 가능한 주소를 DHCP에 넣는 것이 아니다.

이번 프로젝트에서는 주소를 다음처럼 구분한다.

### 고정 주소

관리자가 장치에 특정 주소를 직접 정한다.

예:

```text
라우터       192.168.10.1
DHCP 서비스  192.168.10.2
DNS          192.168.20.10
웹 서버      192.168.20.20
```

### DHCP 동적 할당 범위

DHCP 서버가 클라이언트에게 자동으로 빌려줄 수 있도록 정한 범위다.

직원망에서는 향후:

```text
192.168.10.30 ~ 192.168.10.50
```

을 사용한다.

DHCP는 8주차에 실제 구성한다.

### 예약 주소

특정 장비나 용도를 위해 미리 사용 계획을 잡아 둔 주소다.

예:

```text
192.168.10.1 → 향후 R1 직원망 인터페이스
192.168.10.2 → 향후 DHCP 서비스
```

3주차에는 아직 이 장비를 만들지 않는다.

### DHCP 동적 할당 제외 주소

DHCP 서버가 동적으로 나눠 주지 않도록 하는 영역이다.

이번 설계에서는 동적 풀이 `.30~.50`뿐이므로 그 밖의 주소는 동적 할당 대상으로 사용하지 않는다.

단, **“DHCP에서 제외됐다”와 “사용할 수 없는 주소다”는 같은 말이 아니다.**

예를 들어 `.10`은 DHCP 풀에서는 제외되어 있지만 현재 `pbl-c1`의 고정 주소로 정상 사용한다.

---

# 5. 개인 주소 설계 활동

## 실행 위치

**개인 기록지 또는 개인 문서**

아직 명령을 실행하지 않는다.

A·B·C·D 모두 서로 상의하기 전에 먼저 작성한다.

---

## 문제 1 — 직원망

요구 조건:

```text
기반 서비스와 게이트웨이를 포함하여 최대 50개 호스트 주소 필요
```

다음을 작성한다.

```text
필요한 최소 호스트 비트:
선택한 프리픽스:
서브넷 마스크:
전체 주소 수:
사용 가능한 호스트 수:
네트워크 주소:
호스트 범위:
브로드캐스트 주소:
```

생각할 것:

```text
/27 → 30개 사용 가능
/26 → 62개 사용 가능
```

어느 것이 50개 요구 조건을 충족하는가?

---

## 문제 2 — 서버망

요구 조건:

```text
게이트웨이를 포함하여 최대 20개 호스트 주소 필요
```

다음을 작성한다.

```text
/28의 일반적인 사용 가능 호스트 수:
 /27의 일반적인 사용 가능 호스트 수:

선택한 프리픽스:
선택 이유:
```

---

# 6. 작성 후 확인하는 기준 설계안

개인 답안을 먼저 작성한 뒤 비교한다.

## 직원망

| 항목 | 기준안 |
|---|---|
| 네트워크 | `192.168.10.0/26` |
| 마스크 | `255.255.255.192` |
| 전체 주소 | 64 |
| 일반적인 사용 가능 호스트 | 62 |
| 호스트 범위 | `192.168.10.1 ~ 192.168.10.62` |
| 브로드캐스트 | `192.168.10.63` |
| 게이트웨이 예약 | `192.168.10.1` |
| 기반 서비스 고정 주소 | DHCP 서비스 `192.168.10.2` |
| 현재 클라이언트 고정 주소 | `pbl-c1=.10`, `pbl-c2=.11` |
| 현재 3주차 웹 주소 | `pbl-web=.20` |
| DHCP 예정 범위 | `192.168.10.30 ~ 192.168.10.50` |
| DHCP 동적 할당 제외 영역 | `.1~.29`, `.51~.62` |

`192.168.10.20`의 `pbl-web`은 **현재 단일 LAN 실습용 주소**다.

5주차에는 웹 서버를:

```text
192.168.20.20/27
```

으로 이동시키고 기존 `192.168.10.20`은 제거한다.

---

## 서버망

| 항목 | 기준안 |
|---|---|
| 네트워크 | `192.168.20.0/27` |
| 마스크 | `255.255.255.224` |
| 전체 주소 | 32 |
| 일반적인 사용 가능 호스트 | 30 |
| 호스트 범위 | `192.168.20.1 ~ 192.168.20.30` |
| 브로드캐스트 | `192.168.20.31` |
| 게이트웨이 예약 | `192.168.20.1` |
| 서버 고정 주소 | DNS `192.168.20.10`, Web `192.168.20.20` |
| DHCP 범위 | 없음 |

서버망은 **설계만 한다.**

3주차에 `pbl-br-server`나 `pbl-r1`을 생성해서는 안 된다.

---

# 7. 왜 이 크기를 선택했는가

직원망은 50개 주소가 필요하다.

```text
/27 → 30개
```

이므로 부족하다.

```text
/26 → 62개
```

이므로 요구 조건을 충족한다.

따라서:

```text
192.168.10.0/26
```

을 사용한다.

서버망은 20개 주소가 필요하다.

```text
/28
호스트 비트 4
2^4 - 2 = 14
```

이므로 부족하다.

```text
/27
호스트 비트 5
2^5 - 2 = 30
```

이면 충분하다.

따라서:

```text
192.168.20.0/27
```

을 사용한다.

---

# 8. 수작업 답안 작성 후 Python으로 검산

Python은 **답을 대신 구하는 도구가 아니라 수작업 계산을 확인하는 도구**로 사용한다.

먼저 종이에 계산을 완료한다.

그다음 실행한다.

## 실행 위치

**Linux 호스트 / 본인 SSH 세션**

```bash
python3 - <<'PY'
import ipaddress

for text in ("192.168.10.0/26", "192.168.20.0/27"):
    n = ipaddress.ip_network(text)
    hosts = list(n.hosts())

    print(f"Network   : {n}")
    print(f"Netmask   : {n.netmask}")
    print(f"Broadcast : {n.broadcast_address}")
    print(f"First host: {hosts[0]}")
    print(f"Last host : {hosts[-1]}")
    print(f"Addresses : {n.num_addresses}")
    print(f"Hosts     : {len(hosts)}")
    print()
PY
```

## 예상 관찰

```text
Network   : 192.168.10.0/26
Netmask   : 255.255.255.192
Broadcast : 192.168.10.63
First host: 192.168.10.1
Last host : 192.168.10.62
Addresses : 64
Hosts     : 62

Network   : 192.168.20.0/27
Netmask   : 255.255.255.224
Broadcast : 192.168.20.31
First host: 192.168.20.1
Last host : 192.168.20.30
Addresses : 32
Hosts     : 30
```

## 이유

자신이 직접 계산한 네트워크 주소·브로드캐스트·범위가 맞는지 확인한다.

결과가 다르다면 Python 값을 그대로 복사하기 전에 **어느 계산 단계에서 잘못됐는지 다시 찾는다.**

---

# 9. 셋업 담당자 A의 활동

이제 실제 실습 환경을 준비한다.

A가 아래 작업을 주도하지만, **A도 이후 실습자 활동을 별도로 수행한다.**

---

# 9.1 관리망 상태 백업

## 실행 위치

**Linux 호스트 / A의 관리자 SSH 세션**

## 관리자 권한

일부 명령에 필요.

먼저:

```bash
whoami
echo "$SSH_CONNECTION"
date --iso-8601=seconds
ip -br addr
ip route
```

## 예상 관찰

공용 서버의 실제 관리 인터페이스와 관리 IP, 기본 경로가 나타난다.

## 이유

3주차에 바꾸려는 것은 `pbl-c1`의 가상 주소이지 공용 서버의 관리 주소가 아니다.

**호스트의 실제 NIC, 관리 IP, 기본 경로는 수정하지 않는다.**

---

# 9.2 3주차 기록 디렉터리 생성

먼저 기존 `pbl` 그룹을 확인한다.

```bash
getent group pbl
```

기존 주차에서 사용하던 그룹이 확인되어야 한다.

그다음:

```bash
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week3
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week3/captures
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week3/logs
sudo install -d -o root -g pbl -m 2770 /srv/pbl/week3/backup
```

---

# 9.3 현재 상태를 파일로 백업

```bash
STAMP="$(date +%Y%m%d-%H%M%S)"

{
    echo "=== TIME ==="
    date --iso-8601=seconds

    echo
    echo "=== HOST ADDR ==="
    ip -br addr

    echo
    echo "=== HOST ROUTE ==="
    ip route

    echo
    echo "=== NETNS ==="
    sudo ip netns list

    echo
    echo "=== pbl-c1 ==="
    sudo ip -n pbl-c1 -br addr
    sudo ip -n pbl-c1 route

    echo
    echo "=== pbl-c2 ==="
    sudo ip -n pbl-c2 -br addr
    sudo ip -n pbl-c2 route

    echo
    echo "=== pbl-web ==="
    sudo ip -n pbl-web -br addr
    sudo ip -n pbl-web route
} | sudo tee "/srv/pbl/week3/backup/pre-${STAMP}.txt" >/dev/null
```

확인:

```bash
sudo cat "/srv/pbl/week3/backup/pre-${STAMP}.txt"
```

## 예상 관찰

최소한 다음 주소가 확인되어야 한다.

```text
pbl-c1   192.168.10.10/26
pbl-c2   192.168.10.11/26
pbl-web  192.168.10.20/26
```

그리고:

```bash
sudo ip -n pbl-c1 route show default
sudo ip -n pbl-c2 route show default
sudo ip -n pbl-web route show default
```

에서는 출력이 없어야 한다.

## 실패 점검

`pbl-c2`가 없거나 주소가 다르면 3주차 실습을 계속하지 않는다.

앞 주차의 단일 LAN 정상 상태부터 복구한다.

3주차 가이드에서 임의로 새로운 구조를 만들어 문제를 덮지 않는다.

---

# 9.4 링크와 웹 서비스 확인

```bash
sudo ip link show pbl-br-staff
sudo ip link show pbl-c1-h
sudo ip netns exec pbl-web ss -lnt
```

`8080` 확인:

```bash
sudo ip netns exec pbl-web ss -lnt | grep ':8080'
```

정상 HTTP 확인:

```bash
sudo ip netns exec pbl-c1 \
  curl --noproxy '*' \
  -fsS -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  --max-time 5 \
  http://192.168.10.20:8080/
```

## 예상 관찰

정상 상태라면:

```text
HTTP 200
```

이 예상된다.

## 의미

주소를 바꾸기 **전** 정상 상태를 확인한 것이다.

나중에 실험이 실패했을 때 원래부터 고장 나 있었던 환경과 주소 변경 때문에 발생한 문제를 구분할 수 있다.

---

# 9.5 필요한 명령 확인

```bash
command -v ip
command -v tcpdump
command -v curl
command -v ping
command -v python3
```

모두 경로가 나오는지 확인한다.

`ping`만 없다면 Ubuntu에서는 일반적으로 `iputils-ping` 패키지가 필요하다.

```bash
sudo apt install -y iputils-ping
```

없는 프로그램만 설치한다.

---

# 9.6 참여자별 3주차 임시 주소

이번 주에 실제 주소를 바꿀 대상은 **`pbl-c1` 하나뿐**이다.

참여자는 한 명씩 순서대로 실습한다.

| 참여자 | 임시 `pbl-c1` 주소 |
|---|---|
| A | `192.168.10.12/26` |
| B | `192.168.10.13/26` |
| C | `192.168.10.14/26` |
| D | `192.168.10.15/26` |

이 주소들은:

```text
pbl-c2 = .11
pbl-web = .20
DHCP 예정 범위 = .30~.50
```

과 겹치지 않는다.

또 모두:

```text
192.168.10.1 ~ 192.168.10.62
```

범위 안에 있다.

각 실습 종료 후 반드시:

```text
pbl-c1 = 192.168.10.10/26
```

으로 복구한다.

---

# 9.7 실제 Linux 계정과 참여자 역할 매핑

실제 계정 이름은 이 문서가 알 수 없으므로 A가 확인한다.

각 참여자 SSH 세션에서:

```bash
whoami
```

를 실행한다.

기존 계정이 정확히:

```text
pbl-a
pbl-b
pbl-c
pbl-d
```

라면 아래 기본값을 그대로 사용할 수 있다.

## 실행 위치

**Linux 호스트 / A 관리자 세션**

```bash
sudo install -d -o root -g pbl -m 0750 /etc/pbl
```

```bash
sudo tee /etc/pbl/week3-users.tsv >/dev/null <<'EOF'
pbl-a A 192.168.10.12
pbl-b B 192.168.10.13
pbl-c C 192.168.10.14
pbl-d D 192.168.10.15
EOF
```

```bash
sudo chown root:pbl /etc/pbl/week3-users.tsv
sudo chmod 0640 /etc/pbl/week3-users.tsv
```

### 실제 계정 이름이 다른 경우

예를 들어 A의 실제 계정이 `student1`이라면:

```text
student1 A 192.168.10.12
```

처럼 **첫 번째 열만 실제 SSH 계정명으로 바꾼다.**

스크립트 안에 알 수 없는 사용자 이름을 임의로 추측해서 넣지 않는다.

확인:

```bash
sudo cat /etc/pbl/week3-users.tsv
```

각 실제 참여자 계정이 `pbl` 그룹에 속하는지도 확인한다.

```bash
id <실제계정명>
```

필요하고 서버 운영 정책상 허용된다면 A가:

```bash
sudo usermod -aG pbl <실제계정명>
```

으로 추가한 뒤 해당 사용자가 다시 로그인한다.

---

# 9.8 임시 주소가 이미 사용 중이지 않은지 확인

격리된 실습망 안의 네임스페이스 주소를 확인한다.

```bash
for ns in $(sudo ip netns list | awk '{print $1}'); do
    echo "=== ${ns} ==="
    sudo ip -n "${ns}" -o -4 addr show
done
```

다음 주소가 이미 다른 장치에 없어야 한다.

```text
192.168.10.12
192.168.10.13
192.168.10.14
192.168.10.15
```

이미 사용 중인 주소가 있다면 그대로 진행하지 않는다.

---

# 9.9 주소 복구 스크립트 준비

Linux의 `ip address` 명령은 인터페이스에 프리픽스 길이가 포함된 주소를 추가·삭제할 수 있다. Ubuntu 24.04의 `ip-address(8)` 문서에도 `add`, `delete`와 `ADDRESS/prefix` 형식이 명시되어 있다.

실습 중 SSH가 끊기거나 참여자가 중간에 종료해도 `pbl-c1`을 `.10/26`으로 되돌릴 수 있도록 복구 스크립트를 먼저 만든다.

## 실행 위치

**Linux 호스트**

## 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w3-restore
```

다음 내용을 전체 저장한다.

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ ${EUID} -ne 0 ]]; then
    echo "ERROR: root 권한이 필요합니다." >&2
    exit 1
fi

NS="pbl-c1"
BASE_ADDR="192.168.10.10/26"

mapfile -t IFS_LIST < <(
    ip -n "${NS}" -o link show |
    awk -F': ' '$2 !~ /^lo(@|$)/ {
        n=$2
        sub(/@.*/, "", n)
        print n
    }'
)

if [[ ${#IFS_LIST[@]} -ne 1 ]]; then
    echo "ERROR: pbl-c1 인터페이스를 하나로 확정할 수 없습니다." >&2
    exit 1
fi

DEV="${IFS_LIST[0]}"

mapfile -t CURRENT < <(
    ip -n "${NS}" -o -4 addr show dev "${DEV}" scope global |
    awk '{print $4}'
)

# 이 실습에서 예상한 주소만 자동으로 처리한다.
# 알 수 없는 주소가 있으면 flush하지 않고 중단한다.
for a in "${CURRENT[@]}"; do
    case "${a}" in
        192.168.10.10/26|\
        192.168.10.12/26|\
        192.168.10.13/26|\
        192.168.10.14/26|\
        192.168.10.15/26)
            ;;
        *)
            echo "ERROR: 예상하지 못한 IPv4 주소 ${a}가 있습니다." >&2
            echo "자동 복구하지 않고 중단합니다." >&2
            exit 1
            ;;
    esac
done

# 정상 주소를 먼저 확보한다.
if ! ip -n "${NS}" -o -4 addr show dev "${DEV}" |
     awk '{print $4}' |
     grep -Fxq "${BASE_ADDR}"; then

    ip -n "${NS}" addr add "${BASE_ADDR}" dev "${DEV}"
fi

# 3주차에서 사용하는 임시 주소만 제거한다.
for a in \
    192.168.10.12/26 \
    192.168.10.13/26 \
    192.168.10.14/26 \
    192.168.10.15/26
do
    if ip -n "${NS}" -o -4 addr show dev "${DEV}" |
       awk '{print $4}' |
       grep -Fxq "${a}"; then

        ip -n "${NS}" addr del "${a}" dev "${DEV}"
    fi
done

ip netns exec "${NS}" \
    ip neigh flush dev "${DEV}" >/dev/null 2>&1 || true

echo "=== 복구 후 pbl-c1 ==="
ip -n "${NS}" -br addr
ip -n "${NS}" route

if ip -n "${NS}" route show default | grep -q .; then
    echo "ERROR: 예상하지 않은 default route가 있습니다." >&2
    echo "자동으로 삭제하지 않습니다." >&2
    exit 1
fi

if ip netns exec pbl-c1 \
    curl --noproxy '*' \
    -fsS -o /dev/null \
    --max-time 5 \
    http://192.168.10.20:8080/
then
    echo "RESTORE CHECK: OK"
else
    echo "ERROR: 주소는 복구했지만 HTTP 정상 확인에 실패했습니다." >&2
    exit 1
fi
```

권한:

```bash
sudo chown root:root /usr/local/sbin/pbl-w3-restore
sudo chmod 0755 /usr/local/sbin/pbl-w3-restore
```

구문 검사:

```bash
sudo bash -n /usr/local/sbin/pbl-w3-restore
```

오류가 없어야 한다.

### 중요한 이유

다음과 같은 광범위한 명령을 복구 방법으로 사용하지 않는다.

```text
ip addr flush ...
전체 네트워크 재시작
호스트 기본 경로 삭제
호스트 방화벽 초기화
```

이번 복구 스크립트는 **3주차에 사용한 알려진 `pbl-c1` 주소만 처리한다.**

---

# 9.10 실습자용 제한 스크립트

참여자가 호스트 전체 네트워크를 자유롭게 변경하지 않고 이번 주 필요한 동작만 수행하도록 한다.

## 실행 위치

**Linux 호스트**

## 관리자 권한 필요

```bash
sudo nano /usr/local/sbin/pbl-w3-lab
```

다음 내용을 전체 저장한다.

```bash
#!/usr/bin/env bash
set -euo pipefail

ACTION="${1:-}"
REAL_USER="${SUDO_USER:-}"

BASE="/srv/pbl/week3"
CAPDIR="${BASE}/captures"
LOGDIR="${BASE}/logs"

USERMAP="/etc/pbl/week3-users.tsv"
LOCKDIR="/run/lock/pbl-w3-lab"

die() {
    echo "ERROR: $*" >&2
    exit 1
}

if [[ ${EUID} -ne 0 ]]; then
    die "sudo를 통해 실행하십시오."
fi

if [[ -z "${REAL_USER}" || "${REAL_USER}" == "root" ]]; then
    die "실습자 계정에서 sudo로 실행하십시오."
fi

[[ -r "${USERMAP}" ]] || die "${USERMAP} 파일이 없습니다."

read -r ROLE TEMP_IP < <(
    awk -v u="${REAL_USER}" \
        '$1==u {print $2, $3; exit}' \
        "${USERMAP}"
)

if [[ -z "${ROLE:-}" || -z "${TEMP_IP:-}" ]]; then
    die "허용된 실습 계정이 아닙니다: ${REAL_USER}"
fi

case "${TEMP_IP}" in
    192.168.10.12|\
    192.168.10.13|\
    192.168.10.14|\
    192.168.10.15)
        ;;
    *)
        die "허용되지 않은 임시 주소입니다: ${TEMP_IP}"
        ;;
esac

c1_if() {
    local ifs

    mapfile -t ifs < <(
        ip -n pbl-c1 -o link show |
        awk -F': ' '$2 !~ /^lo(@|$)/ {
            n=$2
            sub(/@.*/, "", n)
            print n
        }'
    )

    [[ ${#ifs[@]} -eq 1 ]] ||
        die "pbl-c1의 비-loopback 인터페이스를 하나로 확정할 수 없습니다."

    printf '%s\n' "${ifs[0]}"
}

has_c1_addr() {
    local cidr="$1"
    local dev="$2"

    ip -n pbl-c1 -o -4 addr show dev "${dev}" |
        awk '{print $4}' |
        grep -Fxq "${cidr}"
}

require_owner() {
    [[ -d "${LOCKDIR}" ]] ||
        die "먼저 capture-start를 실행하십시오."

    local owner
    owner="$(cat "${LOCKDIR}/owner" 2>/dev/null || true)"

    [[ "${owner}" == "${REAL_USER}" ]] ||
        die "현재 실습 잠금은 ${owner:-unknown} 사용자의 것입니다."
}

require_capture_alive() {
    require_owner

    local pid
    pid="$(cat "${LOCKDIR}/pid" 2>/dev/null || true)"

    [[ "${pid}" =~ ^[0-9]+$ ]] ||
        die "캡처 PID 정보가 올바르지 않습니다."

    kill -0 "${pid}" 2>/dev/null ||
        die "캡처 프로세스가 실행 중이 아닙니다."
}

show_core_state() {
    echo "=== pbl-c1 ==="
    ip -n pbl-c1 -br addr
    ip -n pbl-c1 route

    echo
    echo "=== pbl-c2 ==="
    ip -n pbl-c2 -br addr

    echo
    echo "=== pbl-web ==="
    ip -n pbl-web -br addr

    echo
    echo "=== TCP 8080 ==="
    ip netns exec pbl-web ss -lnt |
        grep ':8080' || true
}

require_baseline() {
    local dev
    dev="$(c1_if)"

    has_c1_addr "192.168.10.10/26" "${dev}" ||
        die "pbl-c1이 192.168.10.10/26 정상 상태가 아닙니다. A가 복구하십시오."

    for t in \
        192.168.10.12/26 \
        192.168.10.13/26 \
        192.168.10.14/26 \
        192.168.10.15/26
    do
        ! has_c1_addr "${t}" "${dev}" ||
            die "이전 임시 주소 ${t}가 남아 있습니다. A가 복구하십시오."
    done

    if ip -n pbl-c1 route show default | grep -q .; then
        die "pbl-c1에 예상하지 않은 default route가 있습니다."
    fi
}

case "${ACTION}" in

check)
    show_core_state
    echo

    require_baseline

    ip -n pbl-c2 -o -4 addr show |
        awk '{print $4}' |
        grep -Fxq '192.168.10.11/26' ||
        die "pbl-c2의 192.168.10.11/26을 확인하지 못했습니다."

    ip -n pbl-web -o -4 addr show |
        awk '{print $4}' |
        grep -Fxq '192.168.10.20/26' ||
        die "pbl-web의 192.168.10.20/26을 확인하지 못했습니다."

    ip netns exec pbl-web ss -lnt |
        grep -q ':8080' ||
        die "pbl-web의 TCP 8080 수신 상태를 확인하지 못했습니다."

    if [[ -d "${LOCKDIR}" ]]; then
        echo "현재 실습 사용자: $(
            cat "${LOCKDIR}/owner" 2>/dev/null || echo unknown
        )"
    else
        echo "현재 활성 실습 없음"
    fi

    echo "CHECK: OK"
    ;;

capture-start)
    require_baseline

    ip link show pbl-c1-h >/dev/null 2>&1 ||
        die "호스트의 pbl-c1-h 인터페이스가 없습니다."

    if ! mkdir "${LOCKDIR}" 2>/dev/null; then
        die "다른 참여자의 3주차 실습이 이미 진행 중입니다: $(
            cat "${LOCKDIR}/owner" 2>/dev/null || echo unknown
        )"
    fi

    trap 'rm -rf "${LOCKDIR}"' ERR INT TERM

    echo "${REAL_USER}" > "${LOCKDIR}/owner"
    echo "${ROLE}" > "${LOCKDIR}/role"
    echo "${TEMP_IP}" > "${LOCKDIR}/temp_ip"

    stamp="$(date +%Y%m%d-%H%M%S)"
    outfile="${CAPDIR}/${REAL_USER}-${stamp}.pcap"
    logfile="${LOGDIR}/${REAL_USER}-${stamp}.tcpdump.log"

    nohup tcpdump \
        -i pbl-c1-h \
        -U \
        -s 0 \
        -w "${outfile}" \
        >"${logfile}" 2>&1 < /dev/null &

    pid=$!

    echo "${pid}" > "${LOCKDIR}/pid"
    echo "${outfile}" > "${LOCKDIR}/file"
    echo "${logfile}" > "${LOCKDIR}/log"

    sleep 0.5

    if ! kill -0 "${pid}" 2>/dev/null; then
        cat "${logfile}" >&2 || true
        die "tcpdump 시작 실패"
    fi

    trap - ERR INT TERM

    echo "CAPTURE STARTED"
    echo "참여자: ${ROLE} (${REAL_USER})"
    echo "임시 주소: ${TEMP_IP}/26"
    echo "시각: $(date --iso-8601=seconds)"
    echo "파일: ${outfile}"
    ;;

set-temp)
    require_capture_alive

    dev="$(c1_if)"
    require_baseline

    # 정상 주소가 사라지는 시간을 최소화하기 위해
    # 임시 주소를 먼저 추가한 뒤 기존 주소만 정확히 삭제한다.
    ip -n pbl-c1 addr add "${TEMP_IP}/26" dev "${dev}"
    ip -n pbl-c1 addr del "192.168.10.10/26" dev "${dev}"

    ip netns exec pbl-c1 \
        ip neigh flush dev "${dev}" >/dev/null 2>&1 || true

    echo "TEMP ADDRESS APPLIED"
    ip -n pbl-c1 -br addr

    echo
    ip -n pbl-c1 route

    echo
    echo "=== 192.168.10.20 경로 선택 ==="
    ip netns exec pbl-c1 ip route get 192.168.10.20
    ;;

test)
    require_capture_alive

    dev="$(c1_if)"

    has_c1_addr "${TEMP_IP}/26" "${dev}" ||
        die "본인 임시 주소 ${TEMP_IP}/26이 적용되지 않았습니다."

    # ARP를 다시 관찰하기 쉽게 이웃 캐시를 비운다.
    ip netns exec pbl-c1 \
        ip neigh flush dev "${dev}" >/dev/null 2>&1 || true

    echo "=== ICMP 확인 ==="
    ip netns exec pbl-c1 \
        ping -c 2 -W 1 192.168.10.20

    echo
    echo "=== HTTP 확인 ==="
    ip netns exec pbl-c1 \
        curl --noproxy '*' \
        -fsS -o /dev/null \
        --max-time 5 \
        http://192.168.10.20:8080/

    echo "HTTP: OK"
    echo "TEST: OK"
    ;;

restore)
    require_owner
    /usr/local/sbin/pbl-w3-restore
    ;;

capture-stop)
    require_capture_alive

    # 원래 주소로 복구되지 않았다면 캡처 종료를 허용하지 않는다.
    require_baseline

    pid="$(cat "${LOCKDIR}/pid")"
    outfile="$(cat "${LOCKDIR}/file")"
    logfile="$(cat "${LOCKDIR}/log")"

    cmdline="$(
        tr '\0' ' ' < "/proc/${pid}/cmdline" 2>/dev/null || true
    )"

    if [[ "${cmdline}" != *"tcpdump"* ||
          "${cmdline}" != *"pbl-c1-h"* ]]; then
        die "PID ${pid}가 예상한 tcpdump가 아닙니다."
    fi

    kill -INT "${pid}" 2>/dev/null || true

    for _ in {1..30}; do
        kill -0 "${pid}" 2>/dev/null || break
        sleep 0.1
    done

    if kill -0 "${pid}" 2>/dev/null; then
        kill -TERM "${pid}" 2>/dev/null || true
    fi

    chgrp pbl "${outfile}" "${logfile}" 2>/dev/null || true
    chmod 0640 "${outfile}" "${logfile}" 2>/dev/null || true

    rm -rf "${LOCKDIR}"

    echo "CAPTURE STOPPED"
    echo "종료 시각: $(date --iso-8601=seconds)"
    echo "PCAP 파일: ${outfile}"
    ;;

*)
    echo "사용법:"
    echo "  sudo /usr/local/sbin/pbl-w3-lab check"
    echo "  sudo /usr/local/sbin/pbl-w3-lab capture-start"
    echo "  sudo /usr/local/sbin/pbl-w3-lab set-temp"
    echo "  sudo /usr/local/sbin/pbl-w3-lab test"
    echo "  sudo /usr/local/sbin/pbl-w3-lab restore"
    echo "  sudo /usr/local/sbin/pbl-w3-lab capture-stop"
    exit 2
    ;;
esac
```

저장 후:

```bash
sudo chown root:root /usr/local/sbin/pbl-w3-lab
sudo chmod 0755 /usr/local/sbin/pbl-w3-lab
sudo bash -n /usr/local/sbin/pbl-w3-lab
```

구문 오류가 없어야 한다.

---

# 9.11 제한된 sudo 권한

## 실행 위치

**Linux 호스트**

## 관리자 권한 필요

```bash
sudo visudo -f /etc/sudoers.d/pbl-week3
```

다음을 저장한다.

```text
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w3-lab check
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w3-lab capture-start
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w3-lab set-temp
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w3-lab test
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w3-lab restore
%pbl ALL=(root) NOPASSWD: /usr/local/sbin/pbl-w3-lab capture-stop
```

확인:

```bash
sudo chmod 0440 /etc/sudoers.d/pbl-week3
sudo visudo -cf /etc/sudoers.d/pbl-week3
```

문법 오류가 없어야 한다.

---

# 9.12 셋업 최종 점검

A 자신의 실습 계정으로 다시 로그인하여 확인한다.

```bash
sudo /usr/local/sbin/pbl-w3-lab check
```

## 예상 관찰

핵심적으로:

```text
pbl-c1  192.168.10.10/26
pbl-c2  192.168.10.11/26
pbl-web 192.168.10.20/26

현재 활성 실습 없음
CHECK: OK
```

이 나타나야 한다.

### `허용된 실습 계정이 아닙니다`가 나온다면

네트워크 문제로 판단하지 않는다.

확인:

```bash
whoami
sudo cat /etc/pbl/week3-users.tsv
```

현재 계정이 매핑 파일 첫 번째 열과 일치하는지 확인한다.

---

# 10. 실습 참여자 A·B·C·D의 활동

이제부터는 **네 명 모두 개별 수행**한다.

공유 `pbl-c1`의 주소를 변경하므로:

```text
A 전체 실습 완료
→ B 전체 실습 완료
→ C 전체 실습 완료
→ D 전체 실습 완료
```

처럼 순서대로 수행한다.

PCAP을 내려받은 뒤 Wireshark 분석은 동시에 수행해도 된다.

---

# 10.1 자신의 주소를 계산부터 한다

명령을 실행하기 전에 본인에게 배정된 임시 주소를 확인한다.

예를 들어 A라면:

```text
192.168.10.12/26
```

이다.

먼저 개인 기록에 직접 계산한다.

```text
주소:
프리픽스:
마스크:
네트워크 주소:
브로드캐스트:
사용 가능한 호스트 범위:
pbl-web 192.168.10.20과 같은 네트워크인가:
그렇게 판단한 계산 근거:
```

A의 예를 계산하면:

```text
12  = 00001100
192 = 11000000
      --------
AND   00000000
```

따라서:

```text
192.168.10.12/26 → 192.168.10.0/26
```

웹 서버:

```text
20  = 00010100
192 = 11000000
      --------
AND   00000000
```

따라서:

```text
192.168.10.20/26 → 192.168.10.0/26
```

두 네트워크 주소가 같으므로 같은 서브넷이라고 예상한다.

B·C·D는 자신의 `.13`, `.14`, `.15`로 직접 계산한다.

---

# 10.2 실험 예상 작성

명령 실행 전에 다음 세 가지를 반드시 예상한다.

예:

```text
1. 주소 변경 후 pbl-c1은 192.168.10.12/26을 사용할 것이다.

2. 192.168.10.12/26과 192.168.10.20/26은
   둘 다 192.168.10.0/26에 속하므로
   게이트웨이 없이 직접 통신할 것으로 예상한다.

3. 캡처에서는 192.168.10.12가 출발지 IP인
   ICMP 또는 TCP 패킷을 볼 것으로 예상한다.
```

아직 관찰하지 않았으므로 **예상**이라고 적는다.

---

# 10.3 준비 상태 확인

## 실행 위치

**Linux 호스트 / 본인 SSH 세션**

```bash
sudo /usr/local/sbin/pbl-w3-lab check
```

## 예상 관찰

```text
CHECK: OK
```

## 이유

다른 사람이 남긴 임시 주소가 없는 정상 상태에서 실험을 시작하기 위함이다.

## 실패 점검

`pbl-c1이 192.168.10.10/26 정상 상태가 아닙니다`가 나온다면 임의로 수정하지 않는다.

A가:

```bash
sudo /usr/local/sbin/pbl-w3-restore
```

를 실행하고 원인을 확인한다.

---

# 10.4 캡처 시작

주소를 바꾸기 **전에** 캡처부터 시작한다.

```bash
sudo /usr/local/sbin/pbl-w3-lab capture-start
```

## 예상 관찰

A 예:

```text
CAPTURE STARTED
참여자: A (pbl-a)
임시 주소: 192.168.10.12/26
시각: ...
파일: /srv/pbl/week3/captures/pbl-a-....pcap
```

실제 계정명과 파일명은 환경마다 다르다.

## 이유

주소 변경 이후 발생하는 ARP·ICMP·TCP를 빠뜨리지 않기 위해서다.

---

# 10.5 자신의 임시 주소 적용

```bash
sudo /usr/local/sbin/pbl-w3-lab set-temp
```

## 예상 관찰

A의 예:

```text
TEMP ADDRESS APPLIED

pbl-c1 ...
192.168.10.12/26
```

경로에서는 다음과 비슷한 항목이 예상된다.

```text
192.168.10.0/26 dev eth0 proto kernel scope link src 192.168.10.12
```

또한:

```text
=== 192.168.10.20 경로 선택 ===
192.168.10.20 dev eth0 src 192.168.10.12 ...
```

와 비슷하게 보일 수 있다.

실제 인터페이스명과 부가 필드는 자신의 출력을 기록한다.

## 핵심 관찰

`192.168.10.20`로 가는 경로에:

```text
via <gateway>
```

가 아니라:

```text
dev <실습 인터페이스>
```

형태의 **직접 연결 경로**가 선택되는지를 본다.

## 이유

수작업 계산에서:

```text
192.168.10.12
AND /26
=
192.168.10.0
```

이라고 계산한 결과가 Linux의 실제 경로 판단에도 연결되는지 확인하기 위해서다.

---

# 10.6 직접 통신 발생

```bash
sudo /usr/local/sbin/pbl-w3-lab test
```

스크립트가 실제로 수행하는 핵심 동작은:

```text
pbl-c1
  ↓
ping 192.168.10.20
  ↓
HTTP 요청 TCP 8080
```

이다.

## 예상 관찰

ICMP에서:

```text
2 packets transmitted
2 received
```

와 비슷한 결과가 예상된다.

HTTP가 성공하면:

```text
HTTP: OK
TEST: OK
```

가 나타난다.

## 의미

자신이 설정한 새로운 주소에서도 같은 `/26` 내부의 웹 서버에 통신했다.

단, 이것만으로 “모든 네트워크가 정상”이라고 일반화하지 않는다.

현재 확인한 조건은:

```text
같은 단일 LAN
+
같은 /26
+
웹 서버 192.168.10.20
```

이다.

---

# 10.7 바로 정상 주소로 복구

테스트가 끝나면 다음 실습자에게 넘기기 전에 복구한다.

```bash
sudo /usr/local/sbin/pbl-w3-lab restore
```

## 예상 관찰

```text
=== 복구 후 pbl-c1 ===
...
192.168.10.10/26
...
RESTORE CHECK: OK
```

## 이유

공유 환경을 정상 상태로 돌려놓는다.

**`.12~.15`가 남아 있는 상태로 다음 사람에게 넘기지 않는다.**

---

# 10.8 캡처 종료

복구 성공을 확인한 뒤:

```bash
sudo /usr/local/sbin/pbl-w3-lab capture-stop
```

## 예상 관찰

```text
CAPTURE STOPPED
종료 시각: ...
PCAP 파일:
/srv/pbl/week3/captures/...
```

정확한 파일명을 기록한다.

### 복구하지 않고 종료하려는 경우

스크립트가 중단하도록 구성되어 있다.

먼저:

```bash
sudo /usr/local/sbin/pbl-w3-lab restore
```

를 수행한다.

---

# 10.9 PCAP을 Windows로 다운로드

SSH 세션에서 파일명을 기록하고 Windows PowerShell로 돌아간다.

## 실행 위치

**개인 Windows PowerShell**

```powershell
New-Item -ItemType Directory -Force "$HOME\pbl-week3"
```

그다음:

```powershell
scp <본인계정>@<SERVER_MANAGEMENT_IP>:/srv/pbl/week3/captures/<실제파일명>.pcap "$HOME\pbl-week3\"
```

예시 이름을 그대로 입력하지 말고 `capture-stop`에서 나온 실제 파일명을 사용한다.

Windows PC의 IP·서브넷 마스크·게이트웨이는 변경하지 않는다.

---

# 11. Wireshark 분석

Wireshark에서 자신의 PCAP을 연다.

```text
File
→ Open
→ 본인의 3주차 PCAP
```

Wireshark의 `ip.addr`은 출발지 또는 목적지 IPv4 주소를, `tcp.port`는 출발지 또는 목적지 TCP 포트를 대상으로 사용할 수 있다.

---

# 11.1 전체 실습 흐름 먼저 보기

Display Filter:

```text
arp || icmp || tcp.port == 8080
```

## 예상 관찰

대략 다음 종류가 보일 수 있다.

```text
ARP
↓
ICMP Echo Request / Reply
↓
TCP 연결
↓
HTTP 요청 / 응답
```

모든 파일에서 정확히 같은 Frame 번호가 나오는 것은 아니다.

---

# 11.2 자신의 임시 주소 찾기

A라면:

```text
ip.addr == 192.168.10.12
```

B:

```text
ip.addr == 192.168.10.13
```

C:

```text
ip.addr == 192.168.10.14
```

D:

```text
ip.addr == 192.168.10.15
```

## 확인할 것

IPv4 패킷에서 자신의 임시 주소가 실제 Source 또는 Destination으로 나타나는지 확인한다.

---

# 11.3 클라이언트 → 웹 서버 패킷

A 예:

```text
ip.src == 192.168.10.12 && ip.dst == 192.168.10.20
```

패킷을 선택한다.

Packet Details에서:

```text
Internet Protocol Version 4
```

를 펼친다.

기록:

```text
Frame 번호:
Source Address:
Destination Address:
```

A라면 클라이언트 → 서버 패킷에서:

```text
Source      192.168.10.12
Destination 192.168.10.20
```

가 예상된다.

---

# 11.4 ARP 확인

필터:

```text
arp
```

Wireshark는 ARP의 sender/target IPv4 주소와 opcode 등의 필드를 제공한다.

캡처를 주소 변경 전에 시작했고 테스트 전에 이웃 캐시를 비웠으므로, 다음과 같은 의미의 ARP를 관찰할 가능성이 높다.

```text
누가 192.168.10.20을 가지고 있는가?
```

그리고 `pbl-web`이 자신의 MAC 주소를 알려주는 응답이 뒤따를 수 있다.

## 확인할 필드

Packet Details:

```text
Address Resolution Protocol
```

을 펼쳐:

```text
Sender IP address
Target IP address
Sender MAC address
Target MAC address
```

를 확인한다.

## 중요한 의미

이번 격리된 실습 구성에서는 `pbl-c1`이 `192.168.10.20`을 직접 연결된 상대라고 판단했기 때문에 **웹 서버 주소 자체에 대한 링크 계층 주소를 알아내려는 ARP**가 발생한다.

다만 “ARP가 보였다” 하나만으로 모든 환경에서 같은 서브넷이라고 단정하지 않는다.

이번 판단에는 함께 다음 증거를 사용한다.

```text
① 수작업 AND 계산
② Linux 연결 경로
③ ip route get 결과
④ ARP
⑤ 실제 IP 패킷
```

---

# 11.5 ICMP 확인

필터:

```text
icmp
```

또는 A의 경우:

```text
icmp && ip.addr == 192.168.10.12
```

Echo Request와 Echo Reply를 확인한다.

기록:

```text
요청 Frame:
요청 Source IP:
요청 Destination IP:

응답 Frame:
응답 Source IP:
응답 Destination IP:
```

---

# 11.6 TCP와 HTTP 확인

필터:

```text
tcp.port == 8080
```

자신의 임시 주소까지 포함하려면 A 예:

```text
tcp.port == 8080 && ip.addr == 192.168.10.12
```

클라이언트 → 서버 패킷에서는:

```text
Destination Port: 8080
```

서버 → 클라이언트 패킷에서는:

```text
Source Port: 8080
```

이 예상된다.

HTTP로 해석되는 경우:

```text
http
```

필터도 사용한다.

8080이 자동으로 HTTP로 해석되지 않으면 TCP 8080 패킷이 존재하는지 먼저 확인한다.

필요하면:

```text
우클릭
→ Decode As...
→ HTTP
```

를 사용한다.

`http` 필터에 아무것도 없다는 이유만으로 통신 실패라고 결론내리지 않는다.

---

# 12. 반드시 작성할 핵심 설명

각 참여자는 다음 질문에 자기 말로 답한다.

> **내가 사용한 임시 IP가 `192.168.10.20`과 `/26`에서 왜 같은 네트워크인가?**

답안에는 최소한 다음 세 종류의 근거가 들어가야 한다.

### 1. 계산 근거

A 예:

```text
192.168.10.12 AND 255.255.255.192
= 192.168.10.0

192.168.10.20 AND 255.255.255.192
= 192.168.10.0
```

따라서 네트워크 주소가 같다.

### 2. Linux 경로 근거

실제 출력에서:

```text
192.168.10.0/26 dev ...
```

직접 연결 경로가 있었고:

```text
ip route get 192.168.10.20
```

에서도 별도 게이트웨이의 `via` 없이 인터페이스가 선택되었다.

### 3. 패킷 근거

캡처에서:

```text
Source IP      = 자신의 임시 주소
Destination IP = 192.168.10.20
```

인 실제 ICMP 또는 TCP 패킷을 확인했다.

그리고 필요하면 ARP에서 `192.168.10.20`의 MAC 주소를 알아내는 과정도 근거로 제시한다.

---

# 13. 좋은 답안과 부족한 답안

부족한 답:

```text
앞의 숫자가 비슷해서 같은 네트워크다.
```

또는:

```text
ping이 되니까 같은 네트워크다.
```

이것만으로는 부족하다.

더 적절한 답:

```text
/26은 255.255.255.192이고 블록 크기는 64이다.
따라서 192.168.10.0~63은 같은 /26 블록이다.

내 주소 192.168.10.12와 서버 주소 192.168.10.20에
각각 /26 마스크를 적용하면 두 주소 모두
192.168.10.0이라는 네트워크 주소가 나온다.

실제 Linux에서도 192.168.10.0/26이 직접 연결 경로로
표시되었고, 192.168.10.20에 대한 route get 결과에
별도 gateway가 없었다.

캡처에서도 192.168.10.12에서 192.168.10.20으로
직접 발생한 패킷을 확인했다.
```

처럼 **계산과 실제 관찰을 연결**해야 한다.

---

# 14. 실패 시 점검 순서

| 증상 | 먼저 확인 | 의미 또는 다음 조치 |
|---|---|---|
| `허용된 실습 계정이 아닙니다` | `whoami`, `/etc/pbl/week3-users.tsv` | 계정 매핑 문제 |
| `pbl-c1 ... 정상 상태가 아닙니다` | `ip -n pbl-c1 -br addr` | 이전 실험 복구 여부 확인 |
| `.12~.15`가 이미 존재 | 전체 네임스페이스 주소 조회 | 중복 주소 원인 확인 후 진행 |
| `capture-start`에서 다른 사용자 표시 | 현재 owner | 공유 실습이 아직 끝나지 않음 |
| `set-temp` 실패 | pbl-c1 주소, 실제 인터페이스 | 무작정 `addr flush` 금지 |
| `ping` 실패 | 주소 → route → ARP 순서 | 바로 웹 서비스 장애라고 단정 금지 |
| ping 성공, HTTP 실패 | `pbl-web ss -lnt` | TCP 8080 서비스 확인 |
| `capture-stop`이 복구 요구 | pbl-c1 주소 | `restore` 먼저 실행 |
| Wireshark에 ARP 없음 | 올바른 PCAP인지, 캡처 시작 시점 | 패킷 부재만으로 네트워크 유실 단정 금지 |
| TCP는 있으나 HTTP 표시 없음 | `tcp.port == 8080` | 필요 시 Decode As |
| 복구 스크립트가 알 수 없는 주소 때문에 중단 | 실제 주소 확인 | 자동 flush하지 말고 A가 원인 파악 |

---

# 15. `pbl-c1`이 임시 주소에 남았을 때 A의 복구

## 실행 위치

**Linux 호스트 / A 관리자 세션**

먼저 조회한다.

```bash
sudo ip -n pbl-c1 -br addr
sudo ip -n pbl-c1 route
```

주소가 이번 주의 알려진 `.12~.15` 중 하나라면:

```bash
sudo /usr/local/sbin/pbl-w3-restore
```

## 예상

```text
192.168.10.10/26
RESTORE CHECK: OK
```

## 복구 스크립트가 중단되는 경우

`예상하지 못한 IPv4 주소`가 나타나면 그 주소를 자동 삭제하지 않는다.

A가:

```bash
sudo ip -n pbl-c1 addr
sudo ip -n pbl-c1 route
```

를 보고 변경 원인을 확인한다.

**`ip addr flush`로 모든 주소를 지워서 맞추지 않는다.**

---

# 16. 팀 비교 활동

네 명의 PCAP 분석이 끝난 뒤 비교한다.

팀 토의 전에 각자 먼저 작성한다.

| 항목 | 개인 기록 |
|---|---|
| 임시 IP | |
| `/26` 마스크 | |
| 계산한 네트워크 주소 | |
| Linux에서 관찰한 연결 경로 | |
| `route get`의 source IP | |
| ARP 근거 Frame | |
| ICMP 근거 Frame | |
| TCP/HTTP 근거 Frame | |
| 예상과 달랐던 점 | |
| 아직 확정하지 못한 점 | |

그다음 팀에서 비교한다.

비교할 질문:

1. A·B·C·D의 임시 주소는 서로 달랐는데 왜 모두 같은 `/26` 네트워크였는가?
2. 모든 사람의 네트워크 주소 계산 결과는 무엇이었는가?
3. 출발지 IP는 바뀌었지만 목적지 서버 주소는 왜 그대로였는가?
4. 왜 게이트웨이를 추가하지 않아도 통신할 수 있었는가?
5. 계산 결과와 Linux의 `route` 출력은 어떻게 연결되는가?
6. 계산 결과와 ARP 패킷은 어떻게 연결되는가?
7. ping 성공과 HTTP 성공은 각각 무엇을 보여 주는가?

---

# 17. 개인 제출물

A·B·C·D 각각 제출한다.

## ① 수작업 계산

```text
/26
2진수 마스크:
10진수 마스크:
블록 크기:
전체 주소 수:
사용 가능 호스트 수:

/27
2진수 마스크:
10진수 마스크:
블록 크기:
전체 주소 수:
사용 가능 호스트 수:
```

## ② 주소 설계표

직원망과 서버망 각각:

```text
네트워크:
마스크:
호스트 범위:
브로드캐스트:
게이트웨이 예약:
고정 주소:
DHCP 예정 범위:
DHCP 동적 할당 제외 주소:
```

## ③ 자신의 실제 실험 조건

```text
참여자:
SSH 계정:
실험 시각:
원래 pbl-c1 주소:
임시 주소:
마스크:
웹 서버 주소:
```

## ④ 예상

주소를 변경하기 전에 작성한 내용을 제출한다.

## ⑤ Linux 관찰

다음 출력의 핵심 부분을 기록한다.

```text
ip addr
ip route
ip route get 192.168.10.20
```

## ⑥ PCAP

자신의 실제 `.pcap` 파일.

## ⑦ 근거 패킷

최소 세 개를 선택한다.

```text
근거 1 — ARP

Frame:
관찰:
의미:

근거 2 — ICMP 또는 IPv4

Frame:
Source IP:
Destination IP:
의미:

근거 3 — TCP 또는 HTTP

Frame:
Source IP:
Destination IP:
Source Port:
Destination Port:
의미:
```

## ⑧ 핵심 설명

반드시 자기 말로 답한다.

```text
“내 주소와 192.168.10.20이 /26에서
왜 같은 네트워크인가?”
```

## ⑨ 확인하지 못한 사항

예:

```text
HTTP 세부 헤더 전체는 이번 분석에서 확인하지 않았다.
ARP의 모든 필드 의미는 아직 설명하지 못한다.
```

관찰하지 않은 내용을 측정값처럼 작성하지 않는다.

---

# 18. 이번 주 종료 상태 확인

마지막 참여자의 실습까지 끝난 뒤 A가 확인한다.

```bash
sudo ip -n pbl-c1 -br addr
sudo ip -n pbl-c2 -br addr
sudo ip -n pbl-web -br addr
```

반드시:

```text
pbl-c1   192.168.10.10/26
pbl-c2   192.168.10.11/26
pbl-web  192.168.10.20/26
```

이어야 한다.

기본 경로:

```bash
sudo ip -n pbl-c1 route show default
sudo ip -n pbl-c2 route show default
sudo ip -n pbl-web route show default
```

모두 출력이 없어야 한다.

HTTP 재확인:

```bash
sudo ip netns exec pbl-c1 \
  curl --noproxy '*' \
  -fsS -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  --max-time 5 \
  http://192.168.10.20:8080/
```

정상이라면:

```text
HTTP 200
```

이 예상된다.

---

# 19. 3주차 최종 체크리스트

- [ ] IPv4가 32비트임을 설명할 수 있다.
- [ ] 옥텟이 8비트임을 설명할 수 있다.
- [ ] `/26 = 255.255.255.192`를 계산했다.
- [ ] `/27 = 255.255.255.224`를 계산했다.
- [ ] `/26`의 블록 크기 64를 계산했다.
- [ ] `/27`의 블록 크기 32를 계산했다.
- [ ] `/26`의 일반적인 사용 가능 호스트 수 62를 계산했다.
- [ ] `/27`의 일반적인 사용 가능 호스트 수 30을 계산했다.
- [ ] `192.168.10.0/26`의 네트워크·브로드캐스트·호스트 범위를 계산했다.
- [ ] `192.168.20.0/27`의 네트워크·브로드캐스트·호스트 범위를 계산했다.
- [ ] 직원망이 왜 `/26`이어야 하는지 설명했다.
- [ ] 서버망이 왜 `/27`이면 충분한지 설명했다.
- [ ] 고정·DHCP·예약·DHCP 제외 주소를 구분했다.
- [ ] 수작업 후 Python으로 검산했다.
- [ ] 본인이 `pbl-c1` 주소를 직접 변경했다.
- [ ] 주소 변경 전에 캡처를 시작했다.
- [ ] 직접 연결 경로를 확인했다.
- [ ] `ip route get` 결과를 확인했다.
- [ ] ICMP와 HTTP 요청을 직접 발생시켰다.
- [ ] 자신의 임시 IP가 포함된 패킷을 Wireshark에서 확인했다.
- [ ] 근거 Frame 번호를 기록했다.
- [ ] “왜 같은 네트워크인가?”를 계산과 패킷으로 설명했다.
- [ ] `pbl-c1`을 `.10/26`으로 복구했다.
- [ ] 서버망·라우터·DHCP·DNS를 미리 만들지 않았다.

---

# 20. 이번 주의 핵심 연결

이번 주에 암기해야 할 문장은:

```text
/26 = 255.255.255.192
```

하나가 아니다.

실제로 이해해야 할 흐름은 다음이다.

```text
필요 호스트 수 확인
        ↓
프리픽스 선택
        ↓
마스크 계산
        ↓
네트워크·브로드캐스트·호스트 범위 계산
        ↓
두 주소의 네트워크 주소 비교
        ↓
실제 인터페이스에 주소 설정
        ↓
Linux가 만든 직접 연결 경로 확인
        ↓
실제 통신 발생
        ↓
ARP / IP / ICMP / TCP 패킷 확인
        ↓
계산 결과와 실제 통신을 연결하여 설명
```

3주차가 정상적으로 완료되면 4주차에는 이 정상 기준을 바탕으로 **마스크를 의도적으로 잘못 설정했을 때 로컬/원격 판단이 어떻게 달라지는지** 비교할 수 있다.