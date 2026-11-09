# a603-build-image

Seeed A603 캐리어 보드에 Jetson Orin NX를 설치할 **플래시 이미지를 만드는** 스크립트입니다. 보드에 쓰지는 않습니다. 쓰기는 [a603-flash-image](https://github.com/JunghanShin/a603-flash-image)가 합니다.

이미지에는 **JetPack 6.2 런타임과 개발 패키지**(CUDA·nvcc, TensorRT, cuDNN, VPI, OpenCV 4.8 헤더 포함, gcc·cmake), **기본 사용자 계정**, **부팅 시 CAN(can0, 1 Mbps) 활성화**가 들어 있어, 플래시하고 전원만 켜면 바로 SSH로 접속해 쓸 수 있습니다. 패키지 구성과 버전은 크로스 컴파일 환경 [a603-dev-env](https://github.com/JunghanShin/a603-dev-env)와 같은 목록(`jetpack-6.2.list`)으로 고정되어, PC에서 빌드한 프로그램이 그대로 실행되고 보드에서 직접 빌드해도 같은 라이브러리를 씁니다.

생성과 쓰기를 나눈 이유는 USB 연결 시간입니다. 한 번에 생성하고 쓰면 보드를 USB로 연결한 채 오래 기다려야 하고, 그동안 USB write 에러가 납니다. 이 스크립트는 **보드 없이** 이미지를 만들고, 보드는 마지막 쓰기 단계에서만 연결합니다.

```text
a603-build-image.sh   (보드 없이, 오래 걸림)
  다운로드 → 압축 해제 → Seeed A603 패치 → NVIDIA 바이너리 적용
  → JetPack 런타임·개발 패키지 설치 → 언어·시간대·키보드 → CAN → 기본 사용자 → 이미지 생성
        │ 작업 폴더/Linux_for_Tegra/tools/kernel_flash/images/
        ▼
a603-flash-image.sh   (보드 REC 모드, USB 연결)
  생성된 이미지를 NVMe와 QSPI에 쓰기
```

| 항목 | 값 |
|---|---|
| 모듈 | Jetson Orin NX |
| 캐리어 보드 | Seeed A603 |
| 소프트웨어 | JetPack 6.2 (L4T 36.4.3), Ubuntu 22.04. Seeed 패치(`603_jp62`)가 지원하는 JetPack 6 중 최신 |
| L4T 대상 이름 | `jetson-orin-nano-devkit-super` (Orin NX도 이 이름 사용) |
| 설치 위치 | NVMe SSD (`nvme0n1p1`), 부트로더는 QSPI |

Holybro 보드용 [holybro-build-image](https://github.com/JunghanShin/holybro-build-image)(JetPack 6.2.2)와 구조가 같습니다. 보드마다 지원하는 JetPack이 달라 저장소를 나눴습니다.

## 준비

| 항목 | 내용 |
|---|---|
| Linux | **x86_64 Ubuntu 22.04** PC 또는 가상 머신. 그 밖의 버전은 NVIDIA가 지원하지 않음. 가상 머신 USB 설정은 [holybro-build-image README](https://github.com/JunghanShin/holybro-build-image#가상-머신-virtualbox-vmware)와 같음 |
| 권한 | `sudo` (스크립트가 필요할 때 스스로 `sudo`로 다시 실행) |
| 디스크 | 작업 폴더에 여유 공간 약 50GB |
| 인터넷 | NVIDIA BSP, 루트 파일 시스템과 JetPack 런타임·개발 패키지(수 GB) 다운로드 |
| **Seeed 패치** | `603_jp62.tar.gz`를 **직접 받아 작업 폴더에 둠** (아래) |

### Seeed 패치 받기

Seeed는 A603 드라이버를 SharePoint로 배포해서 스크립트가 자동으로 받을 수 없습니다.

1. [Seeed A603 플래시 문서](https://wiki.seeedstudio.com/reComputer_A603_Flash_System/)의 드라이버 표에서 **JP6.2** 행의 `603_jp62.tar.gz`를 받습니다.
2. 작업 폴더(예: `~/a603-work`)에 그대로 둡니다. 압축은 풀지 않습니다.
3. 같은 문서의 SHA256 값으로 확인하려면 `PATCH_SHA256`을 넘깁니다. 넘기지 않으면 계산한 값만 출력합니다.
   ```bash
   PATCH_SHA256=<Seeed가 공개한 값> ./a603-build-image/a603-build-image.sh -w ~/a603-work
   ```

## 설치

```bash
git clone https://github.com/JunghanShin/a603-build-image.git
```
```bash
git clone https://github.com/JunghanShin/a603-flash-image.git
```

## 작업 폴더

다운로드 파일, `Linux_for_Tegra`, 생성된 이미지는 모두 **작업 폴더**에 저장됩니다. 두 스크립트가 같은 작업 폴더를 써야 쓰기 스크립트가 이미지를 찾습니다.

- `-w 폴더`로 지정합니다.
- 지정하지 않으면 환경 변수 `WORK_DIR`, 그것도 없으면 **명령을 실행한 현재 폴더**를 씁니다.

## 사용법

### 1. 이미지 생성 (보드 없이)

```bash
./a603-build-image/a603-build-image.sh -w ~/a603-work
```

- 이미지가 이미 있으면 아무것도 하지 않고 끝납니다.
- 다시 만들려면 `--force`를 붙입니다. 다운로드한 파일은 다시 받지 않습니다.
- 전체 출력은 작업 폴더의 `build-image.log`에 저장됩니다.

### 2. 보드에 쓰기

[a603-flash-image](https://github.com/JunghanShin/a603-flash-image)에서 같은 작업 폴더를 지정합니다.

```bash
./a603-flash-image/a603-flash-image.sh -w ~/a603-work
```

### 3. 첫 부팅과 접속

초기 설정 화면 없이 바로 부팅되고 자동 로그인됩니다.

| 항목 | 값 |
|---|---|
| 사용자 / 비밀번호 | `dev` / `roqkf` |
| 호스트 이름 | `jetson` |
| 언어 / 키보드 | 영어 `en_US.UTF-8` / US |
| 시간대 | `Asia/Seoul` |
| CAN | `can0` 1 Mbps, 부팅 시 활성화 |

- **이더넷:** 기본은 DHCP입니다. 공유기에서 받은 IP로 접속합니다.
- **USB 원격:** 플래시할 때 쓴 micro-USB를 그대로 두면 `192.168.55.1`로 접속될 것으로 예상합니다. A603에서는 아직 확인하지 않았습니다(아래 [상태](#상태)).
  ```bash
  ssh dev@192.168.55.1
  ```
- 런타임과 개발 패키지가 들어갔는지 확인합니다. 둘 다 `6.2+b77`이 나와야 합니다.
  ```bash
  dpkg-query -W nvidia-jetpack-runtime nvidia-jetpack-dev
  ```

### CAN (`can-setup`)

A603에는 CAN 트랜시버가 보드에 있지만, 모듈의 CAN 핀이 부팅 후 CAN 기능으로 설정되어 있지 않습니다. Seeed 문서의 절차(pinmux 레지스터를 `devmem2`로 쓰고, `mttcan`을 올리고, 비트레이트를 정해 `up`)는 재부팅하면 사라지므로, 이미지에 부팅 서비스(`can-setup.service`)와 명령(`can-setup`)을 넣었습니다.

```bash
can-setup
```
```bash
sudo can-setup bitrate 500000
```
```bash
sudo can-setup off
```

| 명령 | 내용 |
|---|---|
| `can-setup` | 지금 상태(`can state`, bitrate)와 저장된 설정 보기 |
| `sudo can-setup on` | 지금 켜고 부팅할 때도 켬 |
| `sudo can-setup off` | 지금 끄고 부팅할 때도 끔 |
| `sudo can-setup bitrate N` | 비트레이트 변경 후 바로 적용 |

- 설정은 `/etc/default/can-setup`에 남습니다(`IFACE`, `BITRATE`, `RESTART_MS`, `PINMUX`).
- 버스 오프가 되면 100ms 뒤 자동으로 복구합니다(`RESTART_MS=100`).
- `can-utils`가 들어 있어 `candump can0`, `cansend can0 123#1122`로 바로 확인할 수 있습니다.
- 이미지를 만들 때 비트레이트를 바꾸려면 `CAN_BITRATE=500000`을 넘깁니다.

### 개발 패키지 (`jetson-install-dev.sh`)

[holybro-build-image](https://github.com/JunghanShin/holybro-build-image#개발-패키지-jetson-install-devsh)와 같고 버전만 6.2(`6.2+b77`)입니다. 고정 파일은 `/etc/apt/preferences.d/jetpack-6.2.pref`입니다.

### 다른 모듈을 쓸 때: 보드 식별값 다시 확인 (`--with-board`)

식별값은 캐리어가 아니라 **모듈** 값이라, 기본값은 holybro-build-image와 같은 Orin NX 16GB 값입니다. 다른 모듈(예: Orin NX 8GB)이거나 처음 쓰는 모듈이면 한 번 읽어 확인합니다. 보드에는 아무것도 쓰지 않습니다.

1. 보드 전원을 끄고 A603의 **REC 핀과 GND를 쇼트**합니다.
2. PC와 보드의 micro-USB를 연결하고 전원을 켭니다.
3. NVIDIA 장치(`0955:7323` 등)가 `480M`으로 보이는지 확인합니다.
   ```bash
   lsusb -t
   ```
4. 보드를 읽으며 생성합니다.
   ```bash
   ./a603-build-image/a603-build-image.sh -w ~/a603-work --with-board --force
   ```
5. 끝에 출력되는 식별값(`BOARDID` ~ `RAMCODE_ID`) 중 기본값과 다른 값을 스크립트 위쪽에 고쳐 적거나 환경 변수로 넘깁니다.

## 옵션

| 옵션 | 설명 |
|---|---|
| `-w 폴더` | 작업 폴더 (기본: `WORK_DIR`, 없으면 현재 폴더) |
| `--force` | 기존 이미지를 지우고 다시 생성 |
| `--with-board` | REC 모드 보드(480Mbps)에서 식별값을 읽으며 생성 |
| `-h` | 사용법 출력 |

| 환경 변수 | 기본값 | 설명 |
|---|---|---|
| `BOARDID` | `3767` | 모듈 ID |
| `FAB` | `303` | 모듈 리비전 |
| `BOARDSKU` | `0000` | 모듈 SKU |
| `BOARDREV` | `F.1` | 보드 리비전 |
| `CHIP_SKU` | `00:00:00:D3` | 칩 SKU (Orin NX 16GB) |
| `FUSELEVEL` | `fuselevel_production` | 퓨즈 레벨 |
| `RAMCODE_ID` | `00:00:00:01` | DRAM 종류 (RAMCODE 1) |
| `JETSON_USER` | `dev` | 기본 사용자 |
| `JETSON_PASSWORD` | `roqkf` | 기본 사용자 비밀번호 |
| `JETSON_HOSTNAME` | `jetson` | 호스트 이름 |
| `JETSON_TIMEZONE` | `Asia/Seoul` | 시간대 |
| `CAN_BITRATE` | `1000000` | 부팅 시 can0 비트레이트 |
| `PATCH_SHA256` | (없음) | `603_jp62.tar.gz` 확인용 SHA256 |

## 스크립트가 하는 일

1. 작업 폴더에 `603_jp62.tar.gz`가 있는지 먼저 확인합니다(없으면 받는 곳을 안내하고 멈춤).
2. 필요한 패키지를 설치합니다(`wget`, `lbzip2`, `libxml2`, `python3`, `qemu-user-static` 등).
3. NVIDIA BSP와 루트 파일 시스템(r36.4.3)을 받아 풉니다. 이미 있으면 다시 받지 않습니다.
4. Seeed 패치를 풀어 `603_jp62/Linux_for_Tegra/`를 `Linux_for_Tegra`에 덮어씁니다. Seeed 문서와 같이 `apply_binaries.sh`보다 먼저 합니다.
5. `l4t_flash_prerequisites.sh`, `apply_binaries.sh`를 실행합니다.
6. 루트 파일 시스템 안으로 chroot해서 `nvidia-jetpack-runtime=6.2+b77`을 설치합니다. `jetpack-6.2.list`로 모든 JetPack 패키지를, L4T 패키지는 36.4.3으로 고정합니다(저장소에 JetPack 6.2.1용 36.4.4·36.4.7이 있음). 고정 설정은 보드의 `/etc/apt/preferences.d/jetpack-6.2.pref`에 남습니다.
7. `jetson-install-dev.sh`로 개발 패키지를 설치합니다.
8. 언어, 키보드, 시간대를 설정합니다.
9. `devmem2`, `can-utils`와 CAN 부팅 서비스(`can-setup.service`)를 넣습니다.
10. 기본 사용자를 만들고 자동 로그인을 켭니다. 첫 부팅 초기 설정 화면이 나오지 않습니다.
11. `l4t_initrd_flash.sh --no-flash`로 NVMe 설치용 이미지를 만들고 `system.img`가 생겼는지 확인합니다.

## `jetpack-6.2.list`를 만든 방법

NVIDIA r36.4 저장소(`common`, `t234`)의 패키지 목록에서 `nvidia-jetpack=6.2+b77`의 의존성을 풀었습니다. 패키지 구성은 holybro-build-image의 `jetpack-6.2.2.list`와 같게 맞췄고, 6.2.2와 버전이 다른 것은 JetPack 메타 패키지(`nvidia-*`, `6.2+b77`)와 L4T 패키지(`36.4.3-20250107174145`)뿐입니다. CUDA 12.6, TensorRT 10.3, cuDNN 9.3, VPI 3.2, OpenCV 4.8은 같은 버전입니다. a603-dev-env의 목록과 **같은 파일**이어야 합니다.

## 문제 해결

| 증상 | 원인과 조치 |
|---|---|
| `Missing .../603_jp62.tar.gz` | Seeed 패치를 작업 폴더에 두지 않았습니다. [Seeed 패치 받기](#seeed-패치-받기)를 따릅니다. |
| `603_jp62.tar.gz did not unpack to 603_jp62/Linux_for_Tegra` | Seeed가 패치 구조를 바꿨습니다. 출력된 최상위 목록을 보고 스크립트의 `PATCH_DIR`을 고칩니다. |
| `sha256sum: WARNING ... did NOT match` | 받은 파일이 Seeed가 공개한 값과 다릅니다. 다시 받습니다. |
| `L4T in the image is ... but ... expects ...` | BSP와 `jetpack-6.2.list`의 L4T 버전이 다릅니다. |
| `devmem2` 설치 실패 | 루트 파일 시스템의 apt 저장소에 universe가 없는 경우입니다. `build-image.log`에서 apt 오류를 확인합니다. |
| 부팅 후 `can0: not present` | `sudo journalctl -u can-setup`으로 오류를 봅니다. Seeed 패치가 pinmux를 바꿨다면 `/etc/default/can-setup`의 `PINMUX`를 비우고 `sudo can-setup on`을 해 봅니다. |
| 런타임 설치 중 `exec format error` | `sudo systemctl restart systemd-binfmt` 후 다시 실행합니다. |

## 상태

- 2026-11-09 작성. **아직 한 번도 실행하지 않았습니다.** holybro-build-image(Ubuntu 22.04 PC에서 생성·플래시·부팅 확인)의 흐름을 그대로 쓰고 아래만 바꿨습니다.
  - BSP r36.5.0 → r36.4.3, Holybro 패치 → Seeed `603_jp62`(적용 순서도 Seeed 문서대로 `apply_binaries.sh` 전)
  - `usb-role` 제거, CAN 부팅 서비스 추가
- 확인할 것
  - `603_jp62.tar.gz`의 내부 구조(`603_jp62/Linux_for_Tegra/`)와, 패치에 CAN pinmux가 이미 들어 있는지. 들어 있으면 `devmem2` 단계를 빼고 디바이스 트리 설정만 씁니다.
  - A603 micro-USB로 USB 원격(`192.168.55.1`)이 바로 되는지. Holybro는 USB-C 역할 칩이 없어 `usb-role`이 필요했습니다.
  - r36.4.3의 `bootloader/chkbdinfo`(`--with-board` 출력에 사용) 존재 여부.
