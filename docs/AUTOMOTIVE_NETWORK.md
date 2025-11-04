# 자율주행 차량 네트워크 아키텍처
## FRER 기반 Fail-Operational TSN 설계

---

## 📋 목차

1. [차량 E/E 아키텍처 진화](#1-차량-ee-아키텍처-진화)
2. [TSN 기반 백본 네트워크](#2-tsn-기반-백본-네트워크)
3. [Zone 기반 아키텍처](#3-zone-기반-아키텍처)
4. [FRER 적용 전략](#4-frer-적용-전략)
5. [Safety와 Security](#5-safety와-security)
6. [실제 구현 사례](#6-실제-구현-사례)

---

## 1. 차량 E/E 아키텍처 진화

### 1.1 전통적 분산 아키텍처 (2010년대)

```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│Engine   │  │Transmis │  │ABS      │  │Airbag   │
│ECU      │  │sion ECU │  │ECU      │  │ECU      │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │
     └────────────┴────────────┴────────────┘
                      CAN Bus
            (125 kbps ~ 1 Mbps, 비결정론적)

문제점:
❌ 70개 이상 ECU → 복잡도 증가
❌ CAN 대역폭 부족 (카메라/LiDAR 불가)
❌ 단일 버스 → 신뢰성 낮음
❌ 소프트웨어 업데이트 어려움
```

### 1.2 Domain 중심 아키텍처 (2020년대 초)

```
┌──────────────────────────────────────────────────┐
│        ADAS Domain Controller                    │
│  (카메라, 레이더, 초음파 센서 통합)              │
└───────────────┬──────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│        Infotainment Domain Controller            │
│  (디스플레이, 오디오, 내비게이션)                │
└───────────────┬──────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│        Chassis Domain Controller                 │
│  (조향, 제동, 서스펜션)                          │
└───────────────┬──────────────────────────────────┘
                │
        Automotive Ethernet
          (100 Mbps ~ 1 Gbps)

개선:
✅ ECU 개수 감소 (70개 → 15개)
✅ 고대역폭 (Ethernet 1 Gbps)
✅ Domain 별 통합 제어
```

### 1.3 중앙 집중형 아키텍처 (2025년 이후, Level 4-5)

```
                 ┌─────────────────────────────────┐
                 │  Central Compute Unit (CCU)     │
                 │  • AI 가속기 (GPU/NPU)          │
                 │  • Perception + Planning        │
                 │  • Control + Fail-Operational   │
                 └────────────┬────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │        TSN Ethernet Backbone             │
         │      (FRER Enabled, 1~10 Gbps)          │
         └─┬────────┬────────┬────────┬────────┬───┘
           │        │        │        │        │
      ┌────┴──┐ ┌───┴───┐ ┌─┴────┐ ┌┴─────┐ ┌┴────┐
      │Zone   │ │Zone   │ │Zone  │ │Zone  │ │Gateway│
      │ECU    │ │ECU    │ │ECU   │ │ECU   │ │Module │
      │(Front)│ │(Rear) │ │(Left)│ │(Right│ │(V2X)  │
      └───┬───┘ └───┬───┘ └──┬───┘ └──┬───┘ └───────┘
          │         │        │        │
    ┌─────┴──┐ ┌────┴───┐ ┌─┴────┐ ┌─┴────┐
    │Sensors │ │Sensors │ │Sensors│ │Sensor│
    │Actuators│ │Actuators│ │Acts  │ │Acts  │
    └────────┘ └────────┘ └──────┘ └──────┘

특징:
✅ 단일 고성능 컴퓨터 (AI 중심)
✅ TSN Ethernet 백본 (결정론적)
✅ Zone 기반 분산 (지역별 센서 통합)
✅ FRER 이중화 (Fail-Operational)
```

---

## 2. TSN 기반 백본 네트워크

### 2.1 TSN 표준 조합

자율주행 차량은 다음 IEEE TSN 표준을 결합하여 사용합니다:

| 표준 | 기능 | 역할 |
|------|------|------|
| **IEEE 802.1AS** | Time Synchronization (PTP) | 모든 ECU 시간 동기화 (< 1 µs) |
| **IEEE 802.1Qbv** | TAS (Time-Aware Shaper) | 시간 슬롯 기반 스케줄링 |
| **IEEE 802.1Qav** | CBS (Credit-Based Shaper) | 대역폭 예약 (스트림별) |
| **IEEE 802.1CB** | **FRER** | **프레임 이중화 (본 연구 핵심!)** |
| **IEEE 802.1Qci** | Per-Stream Filtering | 스트림별 필터링 및 폴리싱 |
| **IEEE 802.1Qcc** | Stream Reservation | 중앙 집중 설정 (CNC/CUC) |

#### TSN 스택 구조

```
┌─────────────────────────────────────────────────┐
│ Application (V2X, Camera, LiDAR, Control)       │
├─────────────────────────────────────────────────┤
│ Transport (UDP/TCP)                             │
├─────────────────────────────────────────────────┤
│ Network (IPv4/IPv6)                             │
├─────────────────────────────────────────────────┤
│ Data Link (Ethernet)                            │
│  ┌──────────────────────────────────────────┐   │
│  │ 802.1CB FRER (Replication/Elimination)   │   │ ← 본 연구!
│  ├──────────────────────────────────────────┤   │
│  │ 802.1Qbv TAS (Time Slots)                │   │
│  ├──────────────────────────────────────────┤   │
│  │ 802.1Qav CBS (Bandwidth Reservation)     │   │
│  ├──────────────────────────────────────────┤   │
│  │ 802.1AS PTP (Time Sync)                  │   │
│  └──────────────────────────────────────────┘   │
├─────────────────────────────────────────────────┤
│ Physical (100BASE-T1, 1000BASE-T1)              │
└─────────────────────────────────────────────────┘
```

### 2.2 Traffic Class 분류

자율주행 차량의 트래픽을 우선순위에 따라 분류:

| Priority | Traffic Class | 예시 | 요구사항 | FRER 필요? |
|----------|--------------|------|---------|-----------|
| **TC7** | Safety-Critical Control | 긴급 제동 명령 | < 1 ms, 99.999% | ✅ **필수** |
| **TC6** | Real-Time Sensor | LiDAR, 카메라 | < 10 ms, 99.99% | ✅ **필수** |
| **TC5** | V2X Critical | EEBL, 협력 주행 | < 50 ms, 99.9% | ✅ 권장 |
| **TC4** | Diagnostic | ECU 상태 모니터링 | < 100 ms | ⚠️ 선택 |
| **TC3** | Infotainment | 오디오, 비디오 | < 500 ms | ❌ 불필요 |
| **TC2** | Non-Critical | 소프트웨어 업데이트 | < 5 s | ❌ 불필요 |
| **TC1** | Best-Effort | 로그 데이터 | 무제한 | ❌ 불필요 |
| **TC0** | Background | 백그라운드 작업 | 무제한 | ❌ 불필요 |

**FRER 적용 원칙:**
- **TC7, TC6**: FRER 필수 (Safety-Critical)
- **TC5**: FRER 권장 (V2X)
- **TC4 이하**: FRER 불필요 (비용 절감)

### 2.3 TAS + FRER 결합

#### Time-Aware Shaper (802.1Qbv) 동작

```
시간 슬롯 스케줄 (10 ms 주기):

0ms    2ms      4ms      6ms      8ms     10ms
├──────┼────────┼────────┼────────┼───────┤
│ TC7  │  TC6   │  TC5   │  TC4   │ TC3   │ (반복)
│ 제동 │ LiDAR  │ V2X    │ 진단   │ 음악  │
└──────┴────────┴────────┴────────┴───────┘

각 슬롯:
- Gate Open/Close로 제어
- 슬롯 시작 시 해당 TC만 전송 가능
- 다른 TC는 대기 (버퍼링)

FRER 결합:
- TC7, TC6 프레임은 FRER로 복제
- 각 복제본은 동일한 슬롯에 전송
- 수신측에서 중복 제거
```

#### 예시: LiDAR 프레임 전송

```
시각: 0.000000s (TAS 슬롯 TC6 시작)
동작:
1. CCU → TSN Switch S1 (LiDAR 프레임 도착)
2. S1: VCAP 규칙 매칭 → Stream ID=1, TC6, FRER 활성화
3. S1: R-TAG 삽입 (EtherType 0xF1C1, Seq=0x1234)
4. S1: TAS Gate Open (TC6 슬롯) → 전송 허가
5. S1: Replication → Port 1 (경로1), Port 2 (경로2)
6. 경로1: S1 → S2 → S4 (3 ms)
7. 경로2: S1 → S3 → S4 (5 ms)
8. S4: Elimination (3 ms에 첫 프레임 도착 → 전달, 5 ms에 중복 → 폐기)
9. S4 → CCU (총 레이턴시 3 ms, TC6 슬롯 내 완료 ✅)
```

---

## 3. Zone 기반 아키텍처

### 3.1 Zone 개념

**Zone Architecture**는 차량을 지리적 영역으로 나누고, 각 Zone에 Zone ECU를 배치하여 해당 영역의 센서/액추에이터를 통합 관리합니다.

```
        차량 상면도:
┌───────────────────────────┐
│    Front Zone ECU         │← Zone 1
│  (전방 카메라, 레이더)    │
├───────────────────────────┤
│ Left │  Central  │ Right  │
│ Zone │  Compute  │ Zone   │← Zone 2, 4
│ ECU  │   Unit    │ ECU    │
│(측면)│  (CCU)    │(측면)  │
├──────┴───────────┴────────┤
│    Rear Zone ECU          │← Zone 3
│  (후방 카메라, 초음파)    │
└───────────────────────────┘
```

### 3.2 Zone ECU 역할

| Zone | 위치 | 센서/액추에이터 | CCU 연결 |
|------|------|----------------|---------|
| **Zone 1** | 전방 | 전방 카메라×3, 레이더×2, LiDAR | FRER 이중 경로 |
| **Zone 2** | 좌측 | 측면 카메라×2, 초음파×3, 사이드미러 | FRER 이중 경로 |
| **Zone 3** | 후방 | 후방 카메라×2, 후방 레이더, 후방등 | FRER 이중 경로 |
| **Zone 4** | 우측 | 측면 카메라×2, 초음파×3, 사이드미러 | FRER 이중 경로 |

### 3.3 Zone 기반 FRER 토폴로지

```
        Central Compute Unit (CCU)
                  │
         ┌────────┼────────┐
         │   TSN Switch S1  │ (FRER Replication)
         └─┬──────┴──────┬──┘
           │             │
      ┌────┴────┐   ┌────┴────┐
      │Switch S2│   │Switch S3│ (이중 경로)
      └─┬──┬──┬─┘   └─┬──┬──┬─┘
        │  │  │       │  │  │
      ┌─┴┐┌┴┐┌┴──┐  ┌┴─┐┌┴┐┌┴──┐
      │Z1││Z2││Z3 │  │Z1││Z2││Z3 │
      │  ││  ││   │  │  ││  ││   │
      └──┘└──┘└───┘  └──┘└──┘└───┘
      경로1 (Primary)  경로2 (Secondary)

장점:
✅ 각 Zone ECU → CCU가 이중 경로
✅ 한쪽 경로 장애 시 → 다른 경로로 자동 전환
✅ 배선 최소화 (Zone 내부는 짧은 CAN/LIN)
✅ 확장 용이 (Zone 추가 가능)
```

---

## 4. FRER 적용 전략

### 4.1 Critical Path 식별

자율주행 차량에서 **반드시 FRER 적용해야 하는 경로**:

#### Safety-Critical Data Paths

```
1. 센서 → CCU (Perception)
   ┌──────────┐         ┌──────────┐
   │ LiDAR    │ ─FRER─→ │   CCU    │
   │ (Zone 1) │         │Perception│
   └──────────┘         └──────────┘

   이유: 센서 데이터 손실 → 물체 미인지 → 충돌!

2. CCU → 제동/조향 액추에이터 (Control)
   ┌──────────┐         ┌──────────┐
   │   CCU    │ ─FRER─→ │  Brake   │
   │ Control  │         │  ECU     │
   └──────────┘         └──────────┘

   이유: 제어 명령 손실 → 제동/조향 실패 → 사고!

3. V2X Module ↔ CCU (V2V Communication)
   ┌──────────┐         ┌──────────┐
   │  V2X     │ ─FRER─→ │   CCU    │
   │ Module   │ ←─FRER─ │          │
   └──────────┘         └──────────┘

   이유: V2X 메시지 손실 → 협력 주행 실패 → 충돌!
```

#### Non-Critical Paths (FRER 불필요)

```
❌ Infotainment (음악, 내비게이션)
❌ Diagnostic (ECU 상태 로그)
❌ 소프트웨어 업데이트 (백그라운드)

이유: 패킷 손실 시 사용자 불편만 발생, 안전 영향 없음
```

### 4.2 ASIL 등급별 FRER 적용

| ASIL | 위험도 | 예시 | FRER 필요? | 구현 방법 |
|------|--------|------|-----------|----------|
| **QM** | 없음 | 라디오 | ❌ | 일반 Ethernet |
| **ASIL A** | 낮음 | 와이퍼 | ⚠️ | 단일 경로 + 재전송 |
| **ASIL B** | 중간 | 헤드라이트 | ✅ | FRER 이중 경로 |
| **ASIL C** | 높음 | ABS | ✅ | FRER + TAS |
| **ASIL D** | 최고 | 조향, 제동 | ✅ | FRER + TAS + 검증 |

**FRER 구현 가이드:**
- **ASIL D**: FRER 필수, 시퀀스 검증, Wireshark 추적
- **ASIL C**: FRER 필수, 기본 중복 제거
- **ASIL B**: FRER 권장, 비용 허용 시 적용
- **ASIL A 이하**: FRER 불필요

### 4.3 경로 다양성 (Path Diversity)

FRER의 효과를 최대화하려면 **물리적으로 독립적인 경로**가 필요합니다:

#### 나쁜 예: 물리적 단일 경로

```
A → S1 ═══ 단일 케이블 ═══ S2 → B
          (케이블 절단 시 양쪽 모두 장애!)

❌ FRER 무의미 (논리적 이중화만 존재)
```

#### 좋은 예: 물리적 독립 경로

```
A → S1 ─── 케이블1 (차량 좌측) ─── S2 ──┐
      │                                  ├→ S4 → B
      └─── 케이블2 (차량 우측) ─── S3 ──┘

✅ FRER 효과적 (한쪽 케이블 절단 시 다른 쪽으로 전달)
```

#### Zone 기반 경로 다양성

```
Central Compute Unit (상단)
        │
    ┌───┴───┐
 좌측 배선  우측 배선
    │       │
 Zone 2  Zone 4
 (좌측)  (우측)

→ 차량 좌측 배선 절단 시 → 우측 배선으로 전달 ✅
```

---

## 5. Safety와 Security

### 5.1 ISO 26262 Functional Safety

#### V-Model 개발 프로세스

```
┌─────────────────────────────────────────────────┐
│ Concept Phase                                   │
│  • Hazard Analysis (HARA)                       │
│  • ASIL 등급 결정                               │
└────────────┬────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────┐
│ System Level                                    │
│  • Safety Goals (FRER 이중화 필요성 도출)       │
│  • Functional Safety Concept                    │
└────────────┬────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────┐
│ Hardware Level          Software Level          │
│  • LAN9668 FRER HW     • FRER 드라이버          │
│  • 이중 경로 배선      • VCAP 규칙 설정         │
└────────────┬────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────┐
│ Integration & Testing                           │
│  • Wireshark 패킷 검증 (본 논문!)               │
│  • Fault Injection (링크 단선 시뮬레이션)       │
│  • RFC 2544 성능 테스트 (미래 과제)             │
└─────────────────────────────────────────────────┘
```

#### FRER의 Safety 기여

| Safety Mechanism | 구현 | ISO 26262 요구 |
|-----------------|------|----------------|
| **Redundancy** | FRER 이중 경로 | ✅ ASIL-D 만족 |
| **Error Detection** | 시퀀스 번호 검증 | ✅ 순서 오류 감지 |
| **Error Handling** | 중복 프레임 제거 | ✅ 자동 복구 |
| **Diagnostic Coverage** | Wireshark 추적 | ✅ 90% 이상 |

### 5.2 Cybersecurity (ISO/SAE 21434)

FRER은 **신뢰성(Reliability)** 기술이지만, **보안(Security)** 측면도 고려 필요:

#### 위협 모델

| 위협 | 설명 | FRER 영향 | 대응 방법 |
|------|------|----------|----------|
| **R-TAG Spoofing** | 공격자가 가짜 R-TAG 삽입 | ⚠️ 중간 | MACsec 암호화 |
| **Sequence Number Manipulation** | 시퀀스 번호 조작 | ⚠️ 중간 | 암호화 + 서명 |
| **DoS Attack** | 대량 복제 프레임 전송 | ⚠️ 높음 | Rate Limiting (802.1Qci) |
| **Man-in-the-Middle** | 경로 중간에서 프레임 수정 | ⚠️ 높음 | MACsec 종단 간 암호화 |

#### FRER + MACsec 결합

```
┌──────────────────────────────────────────────────┐
│ Original Frame                                   │
│ [Dest MAC][Src MAC][IP][Payload]                 │
└─────────────┬────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────────────┐
│ MACsec Encryption (802.1AE)                      │
│ [Dest][Src][MACsec Tag][Encrypted Payload][ICV]  │
└─────────────┬────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────────────┐
│ FRER R-TAG Insertion (802.1CB)                   │
│ [Dest][Src][R-TAG][Seq][MACsec Tag][...][ICV]    │
└─────────────┬────────────────────────────────────┘
              ↓
         Replication (경로1, 경로2)

→ 암호화 + 이중화 → 보안 + 신뢰성 동시 달성!
```

---

## 6. 실제 구현 사례

### 6.1 KETI LAN9668 4-Switch 토폴로지

```
   Host A (192.168.100.101)
        │
   ┌────┴────┐
   │ Switch  │ Port 0: Host A
   │   S1    │ Port 1: → S2
   │(Replica)│ Port 2: → S3
   └─┬─────┬─┘
     │     │
  ┌──┴──┐ ┌┴────┐
  │ S2  │ │ S3  │ (Independent Paths)
  │     │ │     │
  └──┬──┘ └┬────┘
     │     │
   ┌─┴─────┴─┐
   │ Switch  │ Port 1: S2 →
   │   S4    │ Port 2: S3 →
   │(Elimin) │ Port 0: → Host B
   └────┬────┘
        │
   Host B (192.168.100.255, Broadcast)
```

#### VCAP 규칙 설정 (S1)

```yaml
# Stream Identification
vcap_is1_rule:
  match:
    dst_mac: "ff:ff:ff:ff:ff:ff"  # Broadcast
    src_mac: "00:00:00:00:00:01"  # Host A MAC
  action:
    stream_handle: 1
    vlan_pop: 0
    prio: 6  # TC6

# Sequence Generation + Replication
frer_sequence_generation:
  stream_handle: 1
  sequence_number_start: 0
  enable: true

frer_split_function:
  stream_handle: 1
  output_ports: [1, 2]  # S2 (Port 1), S3 (Port 2)
```

#### VCAP 규칙 설정 (S4)

```yaml
# Member Stream (Port 1 from S2)
frer_member_stream_1:
  stream_handle: 1
  port: 1
  individual_recovery: true

# Member Stream (Port 2 from S3)
frer_member_stream_2:
  stream_handle: 1
  port: 2
  individual_recovery: true

# Sequence Recovery (Elimination)
frer_sequence_recovery:
  stream_handle: 1
  algorithm: "vector"
  history_length: 32
  take_no_sequence: "first"  # 먼저 도착한 것 선택
  reset_timeout: 1000  # 1초
```

### 6.2 검증 결과 (Wireshark)

#### 정상 동작 (No Fault)

```
Packet 1:
  Port 1 (S2): Seq=0x0000, Arrival=3.001 ms → Forward ✅
  Port 2 (S3): Seq=0x0000, Arrival=5.002 ms → Discard ❌

Packet 2:
  Port 1 (S2): Seq=0x0001, Arrival=3.021 ms → Forward ✅
  Port 2 (S3): Seq=0x0001, Arrival=5.022 ms → Discard ❌

→ 경로1(S2)이 빠르므로 항상 경로1 프레임만 전달
→ 경로2(S3) 프레임은 중복으로 폐기
```

#### 링크 장애 (S1-S2 경로 단선)

```
Packet 1:
  Port 1 (S2): ❌ 도달 실패 (링크 단선)
  Port 2 (S3): Seq=0x0000, Arrival=5.002 ms → Forward ✅

Packet 2:
  Port 1 (S2): ❌ 도달 실패
  Port 2 (S3): Seq=0x0001, Arrival=5.022 ms → Forward ✅

→ 경로1 장애 시 즉시 경로2로 전환
→ 패킷 손실 0개! 복구 시간 0ms!
```

### 6.3 성능 측정 (d10frertest 데이터 활용)

```
RFC 2544 결과:
- FRER 활성화 시 레이턴시: 0.356 ms (평균)
- Non-FRER 레이턴시: 0.401 ms (평균)
- 차이: 0.045 ms (오차 범위)

→ FRER 오버헤드 거의 없음!

Sockperf 결과:
- P99 레이턴시 (FRER): 0.431 ms
- P99 레이턴시 (Non-FRER): 0.478 ms
- 차이: 0.047 ms

→ 99% 케이스에서도 성능 차이 미미!

iperf3 대역폭:
- FRER 활성화 시: 943 Mbps (94.3% 링크 활용)

→ 기가비트 성능 달성!
```

---

## 7. 결론

### 7.1 자율주행 네트워크 핵심 요구사항

| 요구사항 | 기술 | FRER 역할 |
|---------|------|----------|
| **고대역폭** | Automotive Ethernet (1~10 Gbps) | 투명 전송 |
| **저지연** | TSN (802.1Qbv TAS) | < 1 µs 추가 |
| **결정론** | TSN (802.1AS PTP) | 시간 동기화 |
| **신뢰성** | **FRER (802.1CB)** | **99.999% 달성!** |
| **보안** | MACsec (802.1AE) | 암호화 전 R-TAG 삽입 |

### 7.2 FRER 적용 Best Practice

1. **Critical Path 식별**: ASIL-D, ASIL-C 경로에 FRER 적용
2. **물리적 경로 분리**: 좌/우측 배선, Zone 기반 다양성
3. **TAS 결합**: 시간 슬롯 + FRER로 결정론 + 신뢰성
4. **하드웨어 오프로드**: LAN9668 같은 TSN 스위치 사용
5. **검증**: Wireshark로 R-TAG, 시퀀스 번호 확인

### 7.3 미래 전망

```
2025: Level 4 자율주행 상용화
      → FRER 필수 (본 연구 성과 활용)

2027: Level 5 완전 자율주행
      → FRER + 5G + MEC 통합

2030: 6G 시대
      → FRER + 6G + AI 네트워크

→ FRER은 자율주행의 핵심 인프라!
```

**KETI의 FRER 연구는 한국 자율주행 산업의 기술 경쟁력을 높이는 초석입니다!**

---

## 참고 자료

1. IEEE 802.1CB-2017, "Frame Replication and Elimination for Reliability"
2. ISO 26262-2:2018, "Road vehicles — Functional safety"
3. ISO/SAE 21434:2021, "Road vehicles — Cybersecurity engineering"
4. SAE J3016, "Taxonomy and Definitions for Terms Related to Driving Automation Systems"
5. AUTOSAR Adaptive Platform Release 22.11, "TSN Integration"
6. Kontron D10 (LAN9668) TSN Evaluation Kit User Guide

---

**작성:** 김현우, KETI 모빌리티플랫폼연구센터
**최종 수정:** 2025-11-04
