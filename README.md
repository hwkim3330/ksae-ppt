# FRER 기반 TSN 이중화 기법 - KSAE 발표 자료

[![GitHub Pages](https://img.shields.io/badge/GitHub-Pages-blue)](https://hwkim3330.github.io/ksae-ppt/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**자동차 이더넷의 신뢰성 확보를 위한 FRER 기반 TSN 이중화 기법 적용 및 성능 검증: Fail-Operational TSN 관점**

---

## 📄 논문 정보

- **제목**: 자동차 이더넷의 신뢰성 확보를 위한 FRER 기반 TSN 이중화 기법 적용 및 성능 검증
- **부제**: Fail-Operational TSN 관점
- **저자**: 김현우, 박부식
- **소속**: 한국전자기술연구원 (KETI) 모빌리티플랫폼연구센터
- **발표**: KSAE 모빌리티 플랫폼 컨퍼런스

---

## 🎯 프로젝트 개요

이 저장소는 **IEEE 802.1CB FRER (Frame Replication and Elimination for Reliability)** 기술을 자율주행 차량의 In-Vehicle Network에 적용한 연구 성과를 정리한 발표 자료입니다.

### 핵심 성과

✅ **FRER 실제 구현 및 검증**
- Microchip LAN9662 기반 4-스위치 토폴로지
- VCAP 규칙을 이용한 스트림 식별 및 복제/제거
- Wireshark를 통한 R-TAG 및 시퀀스 번호 실증

✅ **Fail-Operational 특성 입증**
- 단일 장애 시 제로 패킷 손실 (0%)
- 복구 시간 0ms (즉시 경로 전환)
- ISO 26262 ASIL-D 요구사항 만족

✅ **성능 영향 최소화**
- 레이턴시 거의 동일 (차이 < 0.5 ms)
- 하드웨어 오프로드로 오버헤드 < 1 µs
- 기가비트 성능 달성 (943 Mbps)

---

## 📂 디렉토리 구조

```
ksae-ppt/
├── README.md                      # 본 문서
├── docs/                          # 기술 문서
│   ├── FRER_TECHNICAL.md         # FRER 기술 상세 설명
│   ├── 5G_FRER_INTEGRATION.md    # 5G-URLLC와 FRER 통합
│   └── AUTOMOTIVE_NETWORK.md     # 자율주행 네트워크 아키텍처
├── slides/                        # 발표 자료
│   └── presentation.html         # reveal.js 발표 슬라이드
├── scripts/                       # 발표 대본
├── images/                        # 다이어그램 및 이미지
├── typst/                         # Typst 기술 문서
└── assets/                        # 기타 자산
    └── diagrams/                  # Mermaid/PlantUML 다이어그램
```

---

## 🚀 빠른 시작

### 발표 슬라이드 보기

**온라인 버전** (GitHub Pages):
```
https://hwkim3330.github.io/ksae-ppt/slides/presentation.html
```

**로컬에서 실행**:
```bash
# 저장소 클론
git clone https://github.com/hwkim3330/ksae-ppt.git
cd ksae-ppt

# 웹 서버 실행 (Python)
python3 -m http.server 8000

# 브라우저에서 열기
open http://localhost:8000/slides/presentation.html
```

### 기술 문서 읽기

1. **FRER 기술 상세**: [docs/FRER_TECHNICAL.md](docs/FRER_TECHNICAL.md)
2. **5G-FRER 통합**: [docs/5G_FRER_INTEGRATION.md](docs/5G_FRER_INTEGRATION.md)
3. **차량 네트워크 아키텍처**: [docs/AUTOMOTIVE_NETWORK.md](docs/AUTOMOTIVE_NETWORK.md)

---

## 🔬 연구 배경

### 자율주행 Level 4-5의 신뢰성 요구사항

| 표준 | 요구사항 | 목표 신뢰성 |
|------|---------|-----------|
| **SAE J3016** | Level 4-5 완전 자율주행 | 99.9999% (Six Nines) |
| **ISO 26262** | 기능 안전 (ASIL-D) | Fail-Operational |
| **ISO/PAS 21448** | SOTIF (의도된 기능의 안전) | 예측 불가능 상황 대응 |

---

## 🛠️ FRER 기술 핵심

### IEEE 802.1CB 표준

**FRER (Frame Replication and Elimination for Reliability)**는 TSN (Time-Sensitive Networking) 표준군의 일부로, 네트워크 이중화를 통한 Fail-Operational 특성을 제공합니다.

### 4단계 동작 메커니즘

| 단계 | 기능 | 위치 | 동작 |
|------|------|------|------|
| **1️⃣** | 스트림 식별 | 송신측 (S1) | VCAP 규칙으로 MAC/VLAN 매칭 |
| **2️⃣** | 시퀀스 생성 | 송신측 (S1) | 16-bit 카운터 (0~65535) |
| **3️⃣** | 프레임 복제 | 송신측 (S1) | R-TAG 삽입 후 2개 경로로 전송 |
| **4️⃣** | 중복 제거 | 수신측 (S4) | 시퀀스 번호 기반 중복 폐기 |

---

## 📊 검증 결과

### Wireshark 패킷 캡처

```
✅ R-TAG EtherType 0xF1C1 확인
✅ Sequence Number 단조 증가 (0, 1, 2, ...)
✅ 동일 Seq가 2개 포트에서 수신 (복제 검증)
✅ S4에서 1개만 전달 (중복 제거 검증)
```

### 성능 측정 (FRER vs Non-FRER)

| 지표 | Non-FRER | FRER | 차이 |
|------|----------|------|------|
| **평균 레이턴시** | 0.401 ms | 0.356 ms | **-0.045 ms** |
| **P99 레이턴시** | 0.478 ms | 0.431 ms | **-0.047 ms** |
| **대역폭 (iperf3)** | - | **943 Mbps** | 94.3% 링크 활용 ✅ |

**결론**: 레이턴시 거의 동일, 기가비트 성능 달성!

---

## 🌐 5G-URLLC와 FRER 통합

### End-to-End 신뢰성

```
차량 내부 (FRER) × 무선 (5G-URLLC) × MEC (FRER)
   99.999%     ×     99.99%      ×  99.999%
           = 99.998%+ 달성!
```

---

## 🎯 핵심 기여

1. **FRER 실제 구현 및 검증** - LAN9662 기반 4-스위치 토폴로지
2. **Fail-Operational 입증** - 제로 패킷 손실, 0ms 복구
3. **성능 영향 최소화** - 레이턴시 < 0.5 ms 차이
4. **5G-URLLC 통합** - End-to-End 신뢰성 아키텍처

---

## 📈 향후 연구

- RFC 2544/2889/3918 표준 기반 성능 평가
- 장애 시나리오 확대 (링크 단선, 스위치 고장)
- 실차 환경 적용 (센서 데이터 전송, V2X)

---

## 👥 저자

- **김현우** (KETI 모빌리티플랫폼연구센터)
- **박부식** (KETI 모빌리티플랫폼연구센터)
  - 📧 pusik.park@keti.re.kr

---

## 📜 라이선스

MIT License - Copyright (c) 2025 KETI

---

**Last Updated**: 2025-11-04