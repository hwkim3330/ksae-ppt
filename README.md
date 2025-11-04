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
- **발표**: KSAE 모빌리티 플랫폼 컨퍼런스 2025
- **논문**: [25AKSAEK0402.pdf](25AKSAEK0402.pdf)

---

## 🎯 프로젝트 개요

이 저장소는 **IEEE 802.1CB FRER (Frame Replication and Elimination for Reliability)** 기술을 자율주행 차량의 In-Vehicle Network에 적용한 연구 성과를 정리한 발표 자료입니다.

### 핵심 성과

✅ **FRER 실제 구현 및 검증**
- Microchip LAN9662 평가보드 기반 4-스위치 토폴로지
- VCAP 규칙을 이용한 스트림 식별 및 복제/제거
- Wireshark를 통한 R-TAG (0xF1C1) 및 시퀀스 번호 실증

✅ **IEEE 802.1CB 표준 준수**
- 시퀀스 번호 기반 프레임 복제
- 하드웨어 가속 중복 제거
- 제로 패킷 손실, 즉시 장애 전환 (이론적)

✅ **Fail-Operational 설계**
- ISO 26262 ASIL-D 요구사항 대응
- SOTIF (ISO/PAS 21448) 불확실성 고려
- 물리적으로 독립된 다중 경로 구성

---

## 📂 디렉토리 구조

```
ksae-ppt/
├── README.md                      # 본 문서
├── 25AKSAEK0402.pdf              # 논문 원본
├── index.html                     # 발표 슬라이드 (메인)
├── docs/                          # 기술 문서
│   ├── FRER_TECHNICAL.md         # FRER 기술 상세 설명
│   ├── 5G_FRER_INTEGRATION.md    # 5G-URLLC와 FRER 통합
│   └── AUTOMOTIVE_NETWORK.md     # 자율주행 네트워크 아키텍처
└── slides/                        # 구버전 슬라이드 (백업)
```

---

## 🚀 빠른 시작

### 발표 슬라이드 보기

**온라인 버전** (GitHub Pages):
```
https://hwkim3330.github.io/ksae-ppt/
```

**로컬에서 실행**:
```bash
# 저장소 클론
git clone https://github.com/hwkim3330/ksae-ppt.git
cd ksae-ppt

# 웹 서버 실행 (Python)
python3 -m http.server 8000

# 브라우저에서 열기
open http://localhost:8000/
```

### 논문 보기

- **원본 PDF**: [25AKSAEK0402.pdf](25AKSAEK0402.pdf)
- **발표 슬라이드**: 10개 슬라이드 (논문 내용 기반)

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

### Wireshark 패킷 캡처 검증

본 연구는 **Wireshark 패킷 분석**을 통해 FRER 동작을 실증적으로 검증하였습니다:

```
✅ R-TAG EtherType 0xF1C1 확인
   - 4바이트 태그가 정상적으로 삽입됨

✅ Sequence Number 단조 증가 (0, 1, 2, ...)
   - 16-bit 카운터가 프레임마다 1씩 증가

✅ 동일 Seq가 2개 포트에서 수신
   - 프레임 복제 기능 정상 동작 확인

✅ S4에서 1개만 전달
   - 중복 제거 기능 정상 동작 확인
```

**논문 Photo 1 참조**: R-TAG 및 Sequence Number 검증 화면

---

## 🎯 핵심 기여

1. **FRER 실제 구현** - Microchip LAN9662 평가보드 기반 4-스위치 토폴로지 설계
2. **Wireshark 실증 검증** - R-TAG, 시퀀스 번호, 복제/제거 기능 확인
3. **VCAP 규칙 기반 구현** - 하드웨어 가속을 통한 스트림 식별 및 처리
4. **표준 준수** - IEEE 802.1CB, ISO 26262 ASIL-D, ISO/PAS 21448 SOTIF

---

## 📈 향후 연구 계획

본 연구는 FRER 동작 검증에 중점을 두었으며, 향후 다음 연구를 수행할 예정입니다:

### 정량적 성능 평가
- **RFC 2544**: 처리율, 지연, 프레임 손실률 측정
- **RFC 2889**: 브리지 포워딩 성능 평가
- **RFC 3918**: 멀티캐스트 성능 측정

### Fail-Operational 특성 검증
- **단일 링크 단선**: S1-S2 경로 장애 시나리오
- **스위치 노드 고장**: S3 전체 장애 시나리오
- **다중 경로 동시 장애**: S1-S2 및 S1-S3 동시 장애 시나리오

각 시나리오에서 **패킷 손실률**, **복구 시간**, **지연 변동성**을 측정할 계획입니다.

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