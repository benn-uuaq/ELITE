# ELITE Robot ROS 2 Humble Controller

본 패키지는 ELITE 로봇의 3대 핵심 통신 포트(Modbus 502, Primary 30001, Dashboard 29999)에 동시 접속하여 실시간 상태 모니터링 및 원격 제어를 수행하는 통합 제어 허브 노드입니다.

---

## 📌 필수 사전 작업 (공통)

로봇 노드가 모드버스 통신을 정상적으로 수행하기 위해 시스템 전역 환경에 아래 라이브러리가 반드시 설치되어 있어야 합니다.

```bash
sudo python3 -m pip install pyModbusTCP
```

---

## 💡 사용 방식에 따른 환경 세팅 (2가지 버전)

원하는 작업 형태에 따라 **[버전 A]** 또는 **[버전 B]** 중 하나를 선택하여 세팅하세요.

### 🔹 [버전 A] 단독(Standalone) 실행 환경 구축
이 레포지토리만 단독으로 띄워 로봇 데이터만 모니터링하거나 단독 테스트를 진행할 때 사용하는 방법입니다.

```bash
# 1. 독립 워크스페이스 생성 및 이동
mkdir -p ~/ws_isolated/src
cd ~/ws_isolated/src

# 2. 레포지토리 다운로드 (Clone)
git clone git@github.com:benn-uuaq/ELITE.git

# 3. 워크스페이스 루트로 이동 후 빌드 (Colcon이 하위 패키지를 자동 탐색합니다)
cd ~/ws_isolated
colcon build --symlink-install

# 4. ROS 2 환경 로드 후 실행
source install/setup.bash
ros2 run robot_controller robot_control_node
```

---

### 🔹 [버전 B] 기존 ROS 2 프로젝트/워크스페이스 통합 구축
이미 비전 처리, 상위 시퀀서 등 다른 기능이 돌고 있는 기존 워크스페이스(예: `~/my_robot_ws`)에 이 엘리트 로봇 노드를 얹어서 함께 빌드하고 연동할 때 사용하는 방법입니다.

```bash
# 1. 기존에 사용 중이던 워크스페이스의 src 폴더로 이동
cd ~/my_robot_ws/src

# 2. 소스 패키지 안으로 레포지토리 클론
git clone git@github.com:benn-uuaq/ELITE.git

# 3. 메인 프로그램과의 패키지 경로 정리를 위해 패키지 폴더만 복사 후 정리 (선택 사항)
# ※ 레포지토리 내부에 중첩된 패키지를 꺼내어 기존 src 최상단에 배치합니다.
cp -r ELITE/ros/robot_controller .
rm -rf ELITE

# 4. 워크스페이스 루트로 복귀 후 전체 통합 빌드
cd ~/my_robot_ws
colcon build --symlink-install

# 5. 환경 로드
source install/setup.bash
```
> **💡 통합 운영 팁:** 기존 메인 프로그램 노드에서 본 노드가 던져주는 데이터를 받아 쓰려면, 하단의 **[ROS 2 인터페이스 명세]** 토픽을 구독(Subscribe)하도록 코드를 작성하시면 됩니다.

---

## 🚀 실행 및 로봇 IP 매개변수 설정

실행 시 로봇의 실제 IP를 파라미터로 넘겨줄 수 있어 소스 코드를 수정할 필요가 없습니다.

```bash
# 기본 IP(192.168.227.134)로 실행 시
ros2 run robot_controller robot_control_node

# 다른 로봇 IP로 변경하여 실행 시
ros2 run robot_controller robot_control_node --ros-args -p robot_ip:="192.168.1.100"
```

---

## 📊 ROS 2 인터페이스 명세 (명령어 사전)

노드를 켜둔 상태에서 **새 터미널 창을 열고 환경을 로드(`source install/setup.bash`)한 뒤** 아래 명령어들로 제어 및 모니터링을 실시간 검증합니다.
*(※ 가상 환경 충돌 방지를 위해 `direnv` 등 파이썬 가상환경이 켜지지 않은 홈 디렉토리(`cd ~`) 등에서 실행하는 것을 권장합니다.)*

### 1. 실시간 읽기 파이프라인 (Topics)

#### 🔹 토픽 목록 및 데이터 타입
| 토픽명 | 데이터 타입 | 설명 |
| :--- | :--- | :--- |
| `/robot/status/tcp_pose` | `Float32MultiArray` | **TCP 실시간 공간 좌표 및 자세**<br>· [X, Y, Z, Rx, Ry, Rz] 배열<br>· XYZ 단위: mm / 회전 단위: mRad (음수 완벽 보정) |
| `/robot/status/robot_mode` | `Int32` | **로봇 현재 상태 모드** (66번 레지스터 상태 상세 참조) |
| `/robot/status/control_method` | `Int32` | **제어 권한 상태** (71번 레지스터 상태 상세 참조) |
| `/robot/status/operation_mode` | `Int32` | **동작 모드** (72번 레지스터 상태 상세 참조) |
| `/robot/status/alarms` | `String` | **30001 포트 실시간 에러/알람 원인 스트림**<br>· 충돌 및 세이프티 정지 발생 시 즉시 감지 후 텍스트 발행 |

#### 🔹 레지스터 상태 코드 상세 명세 (전수 기재)
*   **현재 로봇 상태 모드 (`/robot/status/robot_mode`)**
    *   `0`: ROBOT_MODE_DISCONNECTED (연결 끊김)
    *   `1`: ROBOT_MODE_CONFIRM_SAFETY (안전 확인 필요)
    *   `2`: ROBOT_MODE_BOOTING (부팅 중)
    *   `3`: ROBOT_MODE_POWER_OFF (제어 전원 꺼짐)
    *   `4`: ROBOT_MODE_POWER_ON (제어 전원 켜짐)
    *   `5`: ROBOT_MODE_IDLE (대기 상태)
    *   `6`: ROBOT_MODE_BACKDRIVE (백드라이브 모드 활성화)
    *   `7`: ROBOT_MODE_RUNNING (태스크 프로그램 실행 중)
    *   `8`: ROBOT_MODE_UPDATING_FIRMWARE (펌웨어 업데이트 중)
    *   `9`: ROBOT_MODE_WAITING_CALIBRATION (캘리브레이션 대기 중)
*   **제어 권한 상태 (`/robot/status/control_method`)**
    *   `0`: Not open remote control (원격 제어 미개방)
    *   `1`: Local control (티치펜던트 로컬 제어)
    *   `2`: Remote control (원격 통신 제어 활성화)
*   **동작 모드 (`/robot/status/operation_mode`)**
    *   `-1`: None (지정 안 됨)
    *   `0`: Automatic (자동 모드)
    *   `1`: Manual (수동 모드)

#### 🔹 실시간 모니터링 터미널 명령어 예시
```bash
# 실시간 로봇 6축 공간 좌표 확인
ros2 topic echo /robot/status/tcp_pose

# 실시간 로봇 에러/알람 트래킹 대기
ros2 topic echo /robot/status/alarms
```

---

### 2. 원격 제어 및 명령어 전송 파이프라인 (Services)

29999 대시보드 포트를 트리거하는 채널입니다. 명령 결과를 성공/실패 구조(`success: true/false`, `message: "문자열"`)로 응답받습니다. 아래는 구현된 모든 서비스 목록입니다.

*   **로봇 제어 박스 전체 시스템 컨디션 상태 조회 (`status`)**
    ```bash
    ros2 service call /robot/dashboard/status std_srvs/srv/Trigger
    ```
*   **로봇 현재 모드 문자열 조회 (`robotMode`)**
    ```bash
    ros2 service call /robot/dashboard/robot_mode std_srvs/srv/Trigger
    ```
*   **로봇 하드웨어 브레이크 해제 명령 (`brakeRelease`)**
    ```bash
    ros2 service call /robot/dashboard/brake_release std_srvs/srv/Trigger
    ```
*   **로봇 제어 전원 원격 ON (`robotControl -on`)**
    ```bash
    ros2 service call /robot/dashboard/power_on std_srvs/srv/Trigger
    ```
*   **로봇 제어 전원 원격 OFF (`robotControl -off`)**
    ```bash
    ros2 service call /robot/dashboard/power_off std_srvs/srv/Trigger
    ```
*   **적재된 태스크 프로그램 시작 및 재생 (`play`)**
    ```bash
    ros2 service call /robot/dashboard/play std_srvs/srv/Trigger
    ```
*   **동작 중인 태스크 프로그램 일시 정지 (`pause`)**
    ```bash
    ros2 service call /robot/dashboard/pause std_srvs/srv/Trigger
    ```
*   **동작 중인 태스크 프로그램 완전 정지 (`stop`)**
    ```bash
    ros2 service call /robot/dashboard/stop std_srvs/srv/Trigger
    ```
