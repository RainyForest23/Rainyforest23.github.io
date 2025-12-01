---
title: "VM별 AI 워크로드 에너지 측정 연구 여정"
date: 2024-12-01
categories: [PJT, Capstone]
tags: [VM, Energy Measurement, RAPL, KVM, AI Workload]
---

# VM별 AI 워크로드 에너지 측정 연구 여정

## 🎯 연구 동기: 불공정한 클라우드 과금의 문제

클라우드 컴퓨팅 서비스를 사용해본 적 있나요? 대부분의 클라우드 서비스는 **시간 기반 과금 모델**을 사용합니다. VM을 1시간 돌렸으면 1시간 치를 내는 거죠. 그런데 여기엔 큰 문제가 있습니다.

같은 1시간이라도:
- **CPU 집약적 워크로드** (예: 웹 서버, 데이터베이스)
- **GPU 집약적 AI 워크로드** (예: 딥러닝 훈련, 추론)

이 두 워크로드가 **실제로 소비하는 에너지는 완전히 다릅니다**. GPU는 때로 CPU보다 10배 이상의 전력을 소비하는데, 동일한 시간에 대해 같은 비용을 청구받는다면 이게 과연 공정한 걸까요?

이런 문제의식에서 출발한 저희 캡스톤 프로젝트 **"OptimusPrime"**은 **VM별 실제 에너지 소비량을 측정하고 예측하는 모델**을 개발하고 있습니다.

---

## 🤔 왜 VM별 에너지 측정이 어려운가?

### 문제 1: VM은 하드웨어에 직접 접근할 수 없다

가상화 환경에서 VM은 하이퍼바이저(KVM, VMware 등)를 통해 하드웨어를 공유합니다. 이 때문에 **VM 내부에서는 자신이 실제로 얼마나 에너지를 소비했는지 알 수 없습니다**.

```
Host Machine (Physical Server)
├── Hypervisor (KVM)
│   ├── VM1 (AI Training) ← "내가 얼마나 전력을 썼지?"
│   ├── VM2 (Web Server) ← "나도 모르겠어"
│   └── VM3 (Database) ← "측정할 방법이 없어"
└── Hardware (CPU, GPU, Memory)
    └── Power Sensors (RAPL, NVML) ← 여기서만 측정 가능!
```

### 문제 2: Energy Attribution (에너지 귀속) 문제

호스트 머신 전체의 에너지는 측정할 수 있습니다. 하지만 **이 에너지를 각 VM에 어떻게 공정하게 할당할 것인가?**

단순히 CPU 사용률로 나누면 될까요? 아닙니다. 전력 소비는 **비선형적**입니다:
- CPU 사용률 50% ≠ 전력 소비 50%
- 메모리 접근 패턴, 캐시 미스율, GPU 활용도 등 복잡한 요소들이 영향을 미칩니다

---

## 🔬 연구 접근: 측정 가능한 것부터 시작하기

### 하드웨어 환경

저희는 **Dell Alienware Aurora R12** 게이밍 PC를 연구용 서버로 활용했습니다.

**스펙:**
- **CPU:** Intel Core i7-11700KF (8코어, 16스레드, 3.6GHz)
  - Sandy Bridge 이후 세대 → **RAPL 지원 ✅**
- **GPU:** NVIDIA GeForce RTX 3060 Ti (8GB VRAM)
  - NVIDIA-ML 라이브러리로 GPU 전력 측정 가능 ✅
- **RAM:** 32GB DDR4
- **OS:** Ubuntu 24.04 (Linux 기반 → KVM 지원)

게이밍 PC가 연구용으로 적합한 이유:
1. **고성능 GPU** 탑재 (AI 워크로드 실행 가능)
2. **RAPL 지원** CPU (에너지 측정 가능)
3. **충분한 메모리** (다중 VM 동시 실행)
4. **비용 효율적** (서버급 워크스테이션 대비)

---

## ⚡ 핵심 기술 1: RAPL (Running Average Power Limit)

### RAPL이란?

Intel이 Sandy Bridge (2011년) 이후 CPU에 내장한 **하드웨어 전력 측정 인터페이스**입니다. CPU 내부의 센서를 통해 실시간으로 에너지 소비를 측정할 수 있습니다.

### RAPL이 측정하는 도메인

```bash
/sys/class/powercap/intel-rapl/
├── intel-rapl:0/              # Package (CPU 소켓 전체)
│   ├── energy_uj              # 총 에너지 (micro-Joules)
│   ├── intel-rapl:0:0/        # Core (CPU 코어들)
│   │   └── energy_uj
│   ├── intel-rapl:0:1/        # Uncore (LLC 캐시, I/O)
│   │   └── energy_uj
│   └── intel-rapl:0:2/        # DRAM (메모리)
│       └── energy_uj
```

### 실제 측정 코드

```python
def read_rapl_energy():
    """RAPL로 CPU 패키지 에너지 읽기 (단위: Joules)"""
    path = '/sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj'
    with open(path) as f:
        energy_uj = int(f.read())
    return energy_uj / 1_000_000  # micro-Joules → Joules

# 1초 간격으로 전력 측정
prev_energy = read_rapl_energy()
time.sleep(1)
curr_energy = read_rapl_energy()

energy_consumed = curr_energy - prev_energy  # Joules
power = energy_consumed  # Watts (1초 간격이므로)

print(f"CPU Power: {power:.2f}W")
```

### RAPL의 정확도

ACM TOMPECS 2018 논문 "RAPL in Action"에 따르면:
- **실제 전력계 대비 오차 2-5%**
- 매우 신뢰할 수 있는 측정 방법
- 무료, 실시간, 세부 도메인별 측정 가능

---

## 🎮 핵심 기술 2: GPU 전력 측정 (nvidia-smi)

NVIDIA GPU는 **NVML (NVIDIA Management Library)**를 통해 전력 측정을 지원합니다.

### 실시간 GPU 전력 모니터링

```bash
# 1초마다 GPU 전력, 온도, 사용률 출력
watch -n 1 nvidia-smi --query-gpu=power.draw,power.limit,temperature.gpu,utilization.gpu --format=csv

# 로그 파일로 저장 (데이터 수집용)
nvidia-smi dmon -s pucvmet -o DT > gpu_power.log
```

### Python으로 GPU 전력 읽기

```python
import pynvml

pynvml.nvmlInit()
handle = pynvml.nvmlDeviceGetHandleByIndex(0)  # GPU 0번

# 실시간 전력 (mW 단위)
power_mw = pynvml.nvmlDeviceGetPowerUsage(handle)
power_w = power_mw / 1000

# 온도
temp = pynvml.nvmlDeviceGetTemperature(handle, 0)

print(f"GPU Power: {power_w:.2f}W, Temp: {temp}°C")

pynvml.nvmlShutdown()
```

---

## 🖥️ 핵심 기술 3: VM별 자원 사용률 추적 (cgroup)

Linux의 **cgroup (Control Groups)**을 활용하면 각 VM의 자원 사용량을 추적할 수 있습니다.

### KVM + libvirt 환경에서 VM 추적

```bash
# 실행 중인 VM 목록
virsh list

# VM의 cgroup 경로 확인
systemctl status libvirtd | grep cgroup
ls /sys/fs/cgroup/machine.slice/

# VM의 CPU 사용 시간 (나노초 단위)
cat /sys/fs/cgroup/machine.slice/machine-qemu*.scope/cpu.stat

# VM의 메모리 사용량
cat /sys/fs/cgroup/machine.slice/machine-qemu*.scope/memory.current
```

---

## 🔄 에너지 귀속 (Energy Attribution) 문제 해결

### 접근 방법: 다층 측정 + 머신러닝 모델

```
┌─────────────────────────────────────────┐
│  Host-Level Measurement                 │
│  (RAPL + nvidia-smi)                    │
│  → Total System Energy: E_total         │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  VM-Level Resource Metrics              │
│  (cgroup + libvirt stats)               │
│  → CPU usage, Memory BW, GPU util, ...  │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  Attribution Algorithm                  │
│  (ML Model: Gradient Boosting)          │
│  E_VM = f(cpu_util, mem_bw, gpu_util...)│
└─────────────┬───────────────────────────┘
              │
              ▼
        Per-VM Energy
```

### 2024년 최신 연구 참고

"Data-Driven Energy Estimation for Virtual Servers" (arXiv 2024)에서 제시한 **Gradient Boosting Regressor**를 활용하면:
- **R² 0.90-0.97**의 높은 예측 정확도
- 게스트 VM에서 호스트 접근 없이 에너지 추정 가능

---

## 🧪 실험 설계: 워크로드별 에너지 프로파일링

### Phase 1: 베이스라인 측정 (단일 VM)

1. **Idle Baseline**
   - VM 부팅 후 아무 작업 안 함
   - 목적: 최소 에너지 소비량 파악

2. **CPU-Intensive Workload**
   - 컴파일 작업, 압축, 암호화
   - 예: Linux 커널 컴파일

3. **Memory-Intensive Workload**
   - In-memory 데이터베이스, 캐싱
   - 예: Redis 벤치마크

4. **GPU-Intensive AI Workload**
   - PyTorch로 ResNet 훈련
   - 배치 크기 변화 (512 → 4096): 에너지 소비 4배 차이 확인

### Phase 2: 다중 VM 간섭 효과 측정

- VM 2개 동시 실행: AI Training + Web Server
- VM 3개 동시 실행: AI + DB + Idle
- **목표:** 워크로드 간섭이 에너지 소비에 미치는 영향 분석

### Phase 3: 예측 모델 학습 및 검증

- 수집한 데이터로 ML 모델 훈련
- Ground Truth: 단일 VM을 베어메탈에서 실행 → RAPL 측정
- 비교: 다중 VM 환경에서 attribution 결과와 실측값 비교

---

## 📊 전체 시스템 전력 측정 (하드웨어)

소프트웨어 측정(RAPL + nvidia-smi)만으로는 **PSU 효율, 마더보드 전력** 등을 놓칩니다.

### 해결책: 스마트 플러그

**TP-Link Tapo P110** (약 2-3만원):
- WiFi 연결, 실시간 전력 확인
- Python 라이브러리: `PyP100`
- 1초 간격 측정 가능

```python
from PyP100 import PyP110

plug = PyP110.P110("192.168.x.x", "email@gmail.com", "password")
plug.handshake()
plug.login()
energy_usage = plug.getEnergyUsage()
print(f"Total System Power: {energy_usage['result']['current_power']}W")
```

### 측정값 비교로 보정 계수 도출

```
Total System Power (스마트 플러그): 200W
- RAPL (CPU): 65W
- nvidia-smi (GPU): 120W
- 나머지: 15W (PSU 손실, 마더보드, 팬, SSD 등)
```

---

## 🚀 향후 연구 방향

### 1. 실시간 에너지 기반 과금 시스템 프로토타입

- Prometheus + Grafana로 실시간 대시보드 구축
- VM별 에너지 소비량 시각화
- 공정한 과금 시뮬레이션

### 2. 이기종 하드웨어 환경으로 확장

- AMD CPU + NVIDIA GPU
- Intel CPU + AMD GPU
- ARM 기반 서버 (AWS Graviton 등)

### 3. Bursting 워크로드 대응

- 급격한 워크로드 변화 시 예측 정확도 향상
- Transformer 기반 시계열 예측 모델 적용

### 4. 오픈소스 기여

- 측정 도구 및 데이터셋 공개
- Scaphandre, Kepler 같은 기존 프로젝트에 기여

---

## 📚 참고 자료

### 주요 논문
- "Data-Driven Energy Estimation for Virtual Servers" (arXiv 2024)
- "RAPL in Action" (ACM TOMPECS 2018)
- "Efficient Energy Consumption in Hybrid Cloud Environment" (Nature Scientific Reports 2024)

### 오픈소스 도구
- **Scaphandre:** KVM 특화 VM 에너지 측정 도구
- **Kepler:** Kubernetes 환경 에너지 모니터링 (CNCF Sandbox)
- **PowerAPI:** Python 기반 전력 측정 프레임워크

### 학회 및 커뮤니티
- ACM ICPE (성능 평가)
- IEEE CLOUD (클라우드 컴퓨팅)
- CNCF TAG Environmental Sustainability

---

## 💭 마치며

이 연구의 궁극적인 목표는 **더 공정하고 투명한 클라우드 컴퓨팅 과금 체계**를 만드는 것입니다. 시간이 아닌 **실제 에너지 소비량**을 기준으로 과금한다면, 사용자는 더 효율적인 코드를 작성하게 되고, 클라우드 제공자는 에너지 효율적인 인프라를 구축하게 될 것입니다.

이는 단순히 비용 절감의 문제가 아닙니다. 데이터센터의 에너지 소비가 전 세계 전력의 1-2%를 차지하는 현재, **Green IT와 탄소 중립**을 위한 필수적인 접근입니다.

---

**프로젝트:** Team 37 "OptimusPrime" Capstone Research
**지도교수:** 반효경 교수님
**연구기간:** 2024년 9월 - 진행 중
**연구기관:** 이화여자대학교 컴퓨터공학과
