# 학습 진행 기록 — RPi3로 *Mastering Embedded Linux Development* 따라하기

> **상세 단계별 로그**: GitHub **Issue #1** (명령어·결과·막힌 점이 시간순으로 누적)
> **이 문서**: 다른 컴퓨터에서도 이어갈 수 있는 **포터블 요약 + 재개 지점**

## 목표 / 환경
- 보유 하드웨어는 **Raspberry Pi 3 (aarch64, Cortex-A53)** 뿐. 책/저장소의 RPi4·BeaglePlay 예제를 RPi3로 적응(자세한 매핑은 issue #1 본문).
- 호스트: **x86-64 Ubuntu 22.04**, Bootlin aarch64 툴체인(`~/aarch64--glibc--stable-2024.02-1`), QEMU.
- 시리얼 콘솔: **FTDI FT232 (3.3V)** → 호스트 `/dev/ttyUSB0`, `picocom -b 115200`. (RPi3 GPIO 8/10/6에 TX↔RX 교차)
- SD카드 = **/dev/sdc** (29.7G). 굽기 전 항상 재확인. (`/dev/sda`=외장 데이터, `/dev/sdb`=시스템 — 절대 금지)

## 진행 현황 (Phase)
| Phase | 내용 | 상태 |
|---|---|---|
| 0 | 호스트 셋업 (툴체인·QEMU·빌드 의존성) | ✅ |
| 1 | Ch2 툴체인 / Ch4-5 커널+rootfs (QEMU 부팅) | ✅ |
| 2 | Ch4 RPi3 커널 빌드 + 실물 부팅 (시리얼) | ✅ |
| 3 | Ch6 Buildroot / Ch7 Yocto (둘 다 실물 부팅) | ✅ |
| 4 | Ch9 read-only rootfs ✅ / **overlayfs 진행 중**, Ch10 OTA 남음 | 🔄 |
| 4-5 | Ch11~21 (디바이스 드라이버·init·프로세스·디버깅 등) | ⬜ |

보류 항목: DSI 7" 터치 디스플레이(콘솔 출력은 됨, KMS/터치 미완), Ch8 Yocto 심화.

## 핵심 환경 메모 (다른 PC에서 재구축용)
- **빌드 결과물·캐시는 git에 안 들어옵니다.** 새 PC에선 Phase 0부터 재구축 필요. 단 외장 드라이브 `/mnt/shared`(Yocto 캐시 46GB + `tmp-meld`)를 물리적으로 옮기면 캐시 재사용 가능.
- **Yocto 작업 디렉토리**: `~/projects/yocto_tutorial/build-meld` (기존 moya 작업 `build`와 분리, `TMPDIR=/mnt/shared/yocto/tmp-meld`, DL_DIR/SSTATE 공유). 재개:
  ```bash
  cd ~/projects/yocto_tutorial && source poky/oe-init-build-env build-meld
  bitbake core-image-minimal
  ```
- `build-meld/conf/local.conf` 주요 설정: `MACHINE="raspberrypi3-64"`, systemd, `ENABLE_UART="1"`(시리얼 콘솔), `EXTRA_IMAGE_FEATURES += "read-only-rootfs"`.
- **굽기**: `sudo bmaptool copy <...wic.bz2> /dev/sdc` (Yocto 정석, .bmap 검증).
- ⚠️ **bitbake용 clean shell 함정**: Bootlin 툴체인 bin의 python3.11(sqlite3 없음)이 시스템 python3(3.10, sqlite3 있음)을 가리면 `bitbake-layers`가 `No module named 'sqlite3'`로 실패. → Bootlin PATH 없는 **새 터미널**에서 작업.

## ▶ 다음 재개 지점: Ch9 정석형 overlayfs (영구 쓰기 계층)
read-only 루트 위에 overlayfs로 영구 쓰기 계층을 얹는다. 레이어: `marcusfolkesson/meta-readonly-rootfs-overlay` (scarthgap).

1. 레이어 클론 + 추가
   ```bash
   cd ~/projects/yocto_tutorial
   git clone https://github.com/marcusfolkesson/meta-readonly-rootfs-overlay.git -b scarthgap
   cd build-meld   # (clean 터미널에서 source poky/oe-init-build-env build-meld 먼저)
   bitbake-layers add-layer ../meta-readonly-rootfs-overlay
   ```
2. **쓰기 파티션(mmcblk0p3) 추가**: 커스텀 wks 작성(메타-raspberrypi `wic/sdimage-raspberrypi.wks` = boot+root 기반, `part --ondisk mmcblk0 --fstype=ext4 --label data --align 4096 --size 512` 추가). 커스텀 레이어(`meta-meld/wic/...`)에 넣고 `WKS_FILE` 지정.
3. **local.conf**: `read-only-rootfs` 제거(`sed -i '/read-only-rootfs/d' conf/local.conf`), `IMAGE_INSTALL:append = " initscripts-readonly-rootfs-overlay"`, `CMDLINE:append = " rootrw=/dev/mmcblk0p3 init=/init"` 추가.
4. `bitbake core-image-minimal` → bmaptool로 /dev/sdc → 부팅 검증: 루트 ro인데 `/etc`·`/var` 쓰기가 **재부팅 후에도 유지**, 전원 막 빼도 안전.

## 남은 작업
- Ch9 overlayfs 마무리 → **Ch10 OTA (Mender)**
- Phase 4: Ch11-12(디바이스 드라이버·애드온 보드), Ch13-14(init·전원)
- Phase 5: Ch15-16(Python·컨테이너), Ch17-18(프로세스·메모리), Ch19-21(디버깅·프로파일링·실시간)
- 보류: DSI 터치 디스플레이 (레거시 펌웨어 방식 또는 커널 빌트인 재빌드)
