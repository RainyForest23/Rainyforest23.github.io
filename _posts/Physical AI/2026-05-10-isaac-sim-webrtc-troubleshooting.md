---
published: true
title: "Isaac Sim 4.2.0 WebRTC 원격 접속 트러블슈팅 정리"
last_modified_at: 2026-05-10
categories: [Physical AI]
tags:
    - Isaac Sim
    - WebRTC
    - Docker
    - ROS2
    - NVIDIA
toc: true
toc_sticky: true
toc_label: "목차"
---

> 환경: Ubuntu 22.04 / Docker / RTX 3060 x2 / Driver 580.142 / Isaac Sim 4.2.0
> 목표: Mac에서 브라우저로 Isaac Sim GUI 원격 접속 (WebRTC)

---

## I. 이슈 트래킹

### Issue 1: `[Warning] [omni.graph.core.plugin]` 무한 출력 + 1,700,000ms 대기

- **증상**: 컨테이너 실행 후 약 28분간 경고 로그만 출력됨
- **원인**: Isaac Sim 최초 실행 시 RTX shader, CUDA kernel, OmniGraph 플러그인을 전부 컴파일하는 정상적인 초기화 과정
- **해결**: 정상임. 기다리면 됨. 캐시가 쌓이면 두 번째 실행부터 빨라짐

---

### Issue 2: GLCache / ComputeCache 비어있음

- **증상**: `/opt/isaac_cache/glcache`, `/opt/isaac_cache/compute` 디렉토리가 비어있음
- **원인**: 캐시 디렉토리가 `root` 소유라 권한 설정이 불일치
- **해결**:

```bash
sudo chmod -R 777 /opt/isaac_cache/
sudo chown -R $USER:$USER /opt/isaac_cache/
```

---

### Issue 3: `librtx.scenedb.plugin.so` Segmentation Fault (Crash)

- **증상**:

```
[Fatal] [carb.crashreporter-breakpad.plugin] librtx.scenedb.plugin.so
munmap_chunk(): invalid pointer
Segmentation fault (core dumped)
```

- **원인 후보 1**: Vulkan ICD 충돌 — `/usr/share/vulkan/icd.d/`에 Intel, AMD, NVIDIA ICD가 공존
- **원인 후보 2**: 드라이버 595와 Isaac Sim 4.2.0 간 호환성 문제 (GitHub Issue #537 확인)
- **해결**: 드라이버가 이미 **580.142**로 되어 있었음 확인 후 crash 해소됨

| 드라이버 버전 | Isaac Sim 4.2.0 | 비고 |
|---|---|---|
| 595.x | ❌ Crash | `librtx.scenedb` segfault |
| 580.x | ✅ 정상 | 권장 버전 |
| 525+ | ✅ 최소 요구사항 | |

---

### Issue 4: `DISPLAY` 미설정 → X11 관련 옵션 불필요

- **증상**: `xhost: unable to open display ""`
- **원인**: WebRTC 원격 접속 목적이므로 X11 디스플레이 불필요
- **해결**: docker run 명령어에서 X11 관련 옵션 전부 제거
  - `-e DISPLAY=$DISPLAY`
  - `-v /tmp/.X11-unix:/tmp/.X11-unix`
  - `-v /etc/vulkan/icd.d:/etc/vulkan/icd.d:ro`
  - `-v /usr/share/nvidia:/usr/share/nvidia:ro`

---

### Issue 5: Streaming Server 포트 바인딩 실패 (0x800B1002)

- **증상**:

```
[Error] [carb.livestream.plugin] Stream Server: starting the server failed, 0x800B1002
[Warning] [omni.kvdb.plugin] Disabling key-value database because another kit process is locking it
```

- **원인**: 이전 Isaac Sim 프로세스가 종료되지 않아 lock 파일 충돌
- **해결**: 컨테이너 완전 종료 후 재시작

```bash
docker stop rainy_isaac
docker rm rainy_isaac 2>/dev/null
```

---

### Issue 6: `runheadless.webrtc.sh`가 아닌 `runheadless.native.sh`로 실행됨

- **증상**: 로그에 `Isaac Sim Headless Native App is loaded` 출력, 8211 포트 미오픈
- **원인**: 컨테이너 이미지의 기본 ENTRYPOINT가 `runheadless.native.sh`로 설정되어 있어서, `runheadless.webrtc.sh --allow-root`가 native 스크립트의 인자로 전달됨

```
/bin/sh -c /isaac-sim/runheadless.native.sh /isaac-sim/runheadless.webrtc.sh --allow-root
```

- **해결**: `--entrypoint` 플래그로 명시적 덮어쓰기

```bash
--entrypoint /isaac-sim/runheadless.webrtc.sh
```

> Docker 이미지에 `ENTRYPOINT`가 설정된 경우, `docker run` 마지막 명령어는 ENTRYPOINT의 **인자**로 전달된다. `--entrypoint` 플래그로 명시적으로 덮어써야 원하는 스크립트가 실행된다.

---

### Issue 7: 8211 포트 오픈됐으나 WebRTC 무한 로딩

- **증상**: `http://192.168.0.51:8211/streaming/webrtc-client/` 무한 로딩
- **원인**: Chrome 보안 정책 — HTTP에서 WebRTC 차단
- **해결**: `chrome://flags/#unsafely-treat-insecure-origin-as-secure`에 해당 URL 추가

---

### Issue 8: WebRTC 클라이언트가 `127.0.1.1`로 리디렉션 ✅ 최종 해결

- **증상**: Chrome Network 탭에서 `webrtc-demo/?server=127.0.1.1 → ERR_CONNECTION_TIMED_OUT`
- **원인**: `/etc/hosts`에 hostname이 `127.0.1.1`로 매핑되어 있어 Isaac Sim이 자신의 IP를 잘못 인식

```
127.0.1.1    rainyforest23-Alienware-Aurora-R12
```

- **해결**:

```bash
sudo sed -i 's/127.0.1.1\trainyforest23-Alienware-Aurora-R12/192.168.0.51\trainyforest23-Alienware-Aurora-R12/' /etc/hosts
```

> Ubuntu는 자신의 hostname을 IP로 변환할 때 `/etc/hosts`를 먼저 참조한다. 기본 설치 시 `127.0.1.1 <hostname>` 매핑이 생성되어 있어, Isaac Sim이 자신의 IP를 `127.0.1.1`로 반환 → 클라이언트(Mac)가 자기 자신의 loopback으로 해석 → 연결 실패.

---

### Issue 9: ROS2 Bridge startup failed

- **증상**: `[Error] [omni.isaac.ros2_bridge.scripts.extension] ROS2 Bridge startup failed`
- **원인**: ROS2 관련 환경변수 3개 누락
  - `RMW_IMPLEMENTATION`: RMW 구현체 지정
  - `LD_LIBRARY_PATH`: ROS2 humble 라이브러리 경로
  - `AMENT_PREFIX_PATH`: ament 패키지 경로 (없으면 `rmw_init_options_init` 심볼 못 찾음)
- **해결**:

```bash
-e "RMW_IMPLEMENTATION=rmw_fastrtps_cpp" \
-e "LD_LIBRARY_PATH=/isaac-sim/exts/omni.isaac.ros2_bridge/humble/lib" \
-e "AMENT_PREFIX_PATH=/isaac-sim/exts/omni.isaac.ros2_bridge/humble" \
```

---

## II. 최종 작동 Docker 명령어

```bash
docker run --name rainy_isaac \
  --gpus '"device=0"' \
  -it --rm \
  --network=host \
  --shm-size=8g \
  --privileged \
  --entrypoint /isaac-sim/runheadless.webrtc.sh \
  -e "ACCEPT_EULA=Y" \
  -e "PRIVACY_CONSENT=Y" \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  -e NVIDIA_VISIBLE_DEVICES=0 \
  -e VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json \
  -e "RMW_IMPLEMENTATION=rmw_fastrtps_cpp" \
  -e "LD_LIBRARY_PATH=/isaac-sim/exts/omni.isaac.ros2_bridge/humble/lib" \
  -e "AMENT_PREFIX_PATH=/isaac-sim/exts/omni.isaac.ros2_bridge/humble" \
  -v /usr/share/vulkan/icd.d:/usr/share/vulkan/icd.d:ro \
  -v ~/my_workspace:/workspace \
  -v /opt/isaac_cache/ov:/root/.local/share/ov/data \
  -v /opt/isaac_cache/glcache:/root/.cache/nvidia/GLCache \
  -v /opt/isaac_cache/compute:/root/.nv/ComputeCache \
  nvcr.io/nvidia/isaac-sim:4.2.0 \
  --allow-root
```

### 접속 URL (Mac 브라우저)

```
http://192.168.0.51:8211/streaming/webrtc-client/
```

Chrome 최초 설정: `chrome://flags/#unsafely-treat-insecure-origin-as-secure` → `http://192.168.0.51:8211` 추가 후 재시작

---

## III. Isaac Sim WebRTC Streaming Client (Mac 전용 앱 권장)

브라우저 대신 **전용 데스크톱 앱** 사용을 권장.
브라우저는 HTTP에서 WebRTC 보안 제한 + UDP 포트 접근 제한이 있어 불안정하다.

### 설치 (Mac)

1. 공식 다운로드 페이지에서 Mac용 dmg 다운로드
2. DMG 열고 **Isaac Sim WebRTC Streaming Client** → **Applications** 폴더로 드래그
3. 앱 실행 후 IP 입력: `192.168.0.51` → **Connect**

### Mac 사용 시 주의사항

- 앱 내 복사/붙여넣기: `Ctrl+C` / `Ctrl+V`
- 호스트 → 클라이언트 복사: `⌘C` / `Ctrl+V`
- 빈 화면: **View** 메뉴 → **Reload**

### 성공 확인 로그 (서버 측)

```
Isaac Sim Headless WebRTC App is loaded.
rclpy loaded
```

---

## IV. 유용한 디버깅 명령어

```bash
# 컨테이너 종료
docker stop rainy_isaac

# 컨테이너 안으로 진입
docker exec -it rainy_isaac bash

# 8211 포트 열렸는지 확인
ss -tlnp | grep 8211

# GPU 상태 실시간 모니터링
watch -n 2 nvidia-smi

# 캐시 용량 모니터링
watch -n 5 'du -sh /opt/isaac_cache/*'

# 스트리밍 관련 로그 필터
docker exec -it rainy_isaac bash -c \
  "grep -i 'stream\|8211\|error\|port' /root/.nvidia-omniverse/logs/Kit/Isaac-Sim/4.2/kit_*.log | tail -30"

# 권한 설정 + lock 파일 제거
sudo chmod -R 777 /opt/isaac_cache/
sudo chown -R $USER:$USER /opt/isaac_cache/
rm -rf /opt/isaac_cache/ov/Kit/Isaac-Sim/4.2/*.lock 2>/dev/null
```

---

## V. 다음 실행 체크리스트

```
□ /etc/hosts에 hostname → 192.168.0.51 매핑 확인
□ /opt/isaac_cache/ 권한 777 확인
□ nvidia-smi로 드라이버 580.x 확인
□ --entrypoint /isaac-sim/runheadless.webrtc.sh 옵션 포함
□ ROS2 환경변수 3개 포함
□ Chrome flags에 http://192.168.0.51:8211 예외 추가 확인
□ 실행 후 "Isaac Sim Headless WebRTC App is loaded" 확인
□ 실행 후 "rclpy loaded" 확인 (ROS2 정상)
□ ss -tlnp | grep 8211 로 포트 오픈 확인
```
