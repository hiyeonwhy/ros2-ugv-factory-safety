<div align="center">

# 🤖 ros2-ugv-factory-safety

**ROS2 기반 UGV로 공장 내부를 자율주행하며 위험 상황을 감지해 관리자에게 전달하는 실내 안전 순찰 시스템**

![ROS2](https://img.shields.io/badge/ROS2-Humble-22314E?style=flat-square&logo=ros&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![C++](https://img.shields.io/badge/C%2B%2B-17-00599C?style=flat-square&logo=cplusplus&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=flat-square&logo=python&logoColor=white)
![Nav2](https://img.shields.io/badge/Navigation-Nav2-4A90D9?style=flat-square)
![License](https://img.shields.io/badge/License-TBD-lightgrey?style=flat-square)

<!-- 로고 이미지가 준비되면 이 자리에 추가하세요 -->
<!-- 빌드 상태 배지: CI 구성 후 아래 주석을 해제하세요 -->
<!-- ![Build](https://img.shields.io/github/actions/workflow/status/<user>/ros2-ugv-factory-safety/ci.yml?style=flat-square) -->

</div>

---

## 📖 프로젝트 개요

공장 현장의 안전 점검은 대부분 사람이 직접 순회하는 방식에 의존합니다. 야간이나 비가동 시간대에는 점검 공백이 생기고, 연기·화재나 작업자의 쓰러짐 같은 위험 상황은 발견이 늦어질수록 피해가 커집니다.

이 프로젝트는 **자율주행 UGV가 공장 내부를 스스로 순회하면서 위험 상황을 실시간으로 감지하고 관리자에게 전달하는 것**을 목표로 합니다. LiDAR 기반 SLAM으로 실내 지도를 만들고 Nav2로 목표 지점까지 자율주행하며, 카메라와 YOLO 모델로 사람의 자세와 연기를 인식합니다.

개발 과정에서 특히 집중한 문제는 **고속 주행·회전 시 발생하는 위치 추정 오차**였습니다. 순찰 로봇은 넓은 구역을 빠르게 돌아야 실용적이지만, 속도를 올리면 odometry drift가 커지면서 SLAM 지도가 틀어지는 문제가 있었습니다. IMU를 추가하고 EKF로 센서를 융합한 뒤, ROS2 bag으로 적용 전후 데이터를 정량 비교하며 개선했습니다.

---

## ✨ 주요 기능

### 1. 🗺️ LiDAR 기반 SLAM 및 자율주행

`slam_toolbox`로 실내 점유 격자 지도를 작성하고, Nav2와 AMCL을 이용해 지도 위에서 목표 지점까지 경로를 계획하고 주행합니다. global/local costmap을 튜닝해 장애물 회피 반응 속도를 확보했습니다.

### 2. 🧭 IMU + EKF 센서 융합 위치 추정

`robot_localization`의 `ekf_filter_node`로 wheel odometry와 IMU를 융합해 `/odom/filtered`를 발행합니다. 2D 주행 특성에 맞춰 yaw와 yaw rate 중심으로 설정하여, 고속 회전 시 방향 추정 오차를 줄였습니다.

### 3. 🔍 YOLO 기반 안전 상황 감지

USB 카메라 영상을 입력으로 YOLOv11 Pose 모델이 작업자의 자세를(쓰러짐 등) 인식하고, 별도 YOLO 모델이 연기를 감지합니다. 프레임 단위 오탐을 줄이기 위해 최근 N개 프레임의 결과를 누적 판단하는 필터링을 적용했습니다.

### 4. 📊 ROS2 Bag 기반 주행 데이터 분석

주행 중 센서 토픽을 bag으로 기록하고 재생하며 IMU 적용 전후의 yaw 변화, odometry drift, localization 안정성을 RMSE/RPE 기준으로 비교 분석합니다.

---

## 🛠️ 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| **Language** | C++17, Python 3.10 |
| **Framework** | ROS2 Humble Hawksbill |
| **SLAM / Mapping** | slam_toolbox (async) |
| **Navigation** | Nav2, AMCL, DWB Controller |
| **Sensor Fusion** | robot_localization (EKF) |
| **Robot Control** | ros2_control, diff_drive_controller, twist_mux |
| **LiDAR** | Slamtec RPLIDAR / SLLIDAR SDK |
| **Perception** | Ultralytics YOLOv11 (Pose / Smoke), OpenCV, cv_bridge |
| **Hardware I/F** | Arduino Serial (libserial) |
| **Tools** | RViz2, rqt_image_view, ros2 bag, colcon |

---

## 🚀 시작하기

### 사전 요구 사항

* **Ubuntu 22.04 LTS**
* **ROS2 Humble** ([설치 가이드](https://docs.ros.org/en/humble/Installation.html))
* colcon 빌드 도구

### 1. 의존 패키지 설치

```bash
sudo apt update
sudo apt install -y \
  ros-humble-slam-toolbox \
  ros-humble-navigation2 \
  ros-humble-nav2-bringup \
  ros-humble-robot-localization \
  ros-humble-twist-mux \
  ros-humble-ros2-control \
  ros-humble-ros2-controllers \
  ros-humble-robot-state-publisher \
  ros-humble-v4l2-camera \
  ros-humble-rqt-image-view \
  libserial-dev
```

### 2. 워크스페이스 클론 및 빌드

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/<your-username>/ros2-ugv-factory-safety.git factory_ugv

cd ~/ros2_ws
colcon build --packages-select factory_ugv
source install/setup.bash
```

매번 source 하는 것이 번거롭다면 `~/.bashrc`에 등록합니다.

```bash
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

> [!IMPORTANT]
> `1.robot.launch.py`는 `description/factory-ugv.urdf.xacro`를 참조합니다. 현재 저장소에는 로봇 URDF(`description/`)가 포함되어 있지 않으므로, 사용 중인 로봇 사양에 맞는 xacro 파일을 해당 경로에 추가한 뒤 실행하세요.

> [!NOTE]
> YOLO 기반 perception 노드(`yolo_pose_node`, `smoke_detection_node`)는 **이 저장소에 포함되어 있지 않습니다.** 별도의 Python 패키지로 구현되어 `/v4l2_camera/image_raw` 토픽을 구독하는 방식으로 연동됩니다.

---

## 💻 사용법

각 launch 파일은 **실행 순서대로 번호가 매겨져 있습니다.** 터미널을 나눠 순서대로 실행하세요.

### Step 1. 로봇 본체 구동

`robot_state_publisher`, `ros2_control`, `twist_mux`, diff drive 컨트롤러를 함께 올립니다.

```bash
ros2 launch factory_ugv 1.robot.launch.py
```

### Step 2. LiDAR 구동

```bash
ros2 launch factory_ugv 2.sllidar_a1.launch.py
```

USB 포트 번호가 재부팅 때마다 바뀌는 문제를 피하기 위해 기본값으로 `/dev/serial/by-path/...` 경로를 사용합니다. 사용 환경이 다르면 인자로 지정하세요.

```bash
ros2 launch factory_ugv 2.sllidar_a1.launch.py \
  serial_port:=/dev/ttyUSB0 \
  serial_baudrate:=115200 \
  frame_id:=laser_frame
```

### Step 3. SLAM (지도 작성)

```bash
ros2 launch factory_ugv 3.slam_toolbox.launch.py use_sim_time:=false
```

RViz2로 지도가 그려지는 것을 확인하며 로봇을 주행시킵니다. 완성된 지도는 slam_toolbox의 `serialize_map` 서비스로 저장합니다.

```bash
rviz2
```

### Step 4. Navigation (자율주행)

```bash
ros2 launch factory_ugv 4.navigation_launch.py use_sim_time:=false
```

RViz2의 **2D Goal Pose**로 목표 지점을 지정하면 로봇이 경로를 계획해 주행합니다.

<details>
<summary>📌 저장된 지도로 localization 모드 실행하기</summary>

`params/slam_toolbox_params.yaml`의 `map_file_name`에 저장된 지도의 **절대 경로**(확장자 제외)를 지정하고 `mode: localization`으로 설정합니다.

```yaml
slam_toolbox:
  ros__parameters:
    mode: localization
    map_file_name: /home/<user>/ros2_ws/src/factory_ugv/map/test_1021
    map_start_at_dock: true
```

</details>

### 상태 확인 명령어

```bash
# 노드 및 토픽 확인
ros2 node list
ros2 topic list

# 센서 데이터 확인
ros2 topic echo /scan
ros2 topic echo /imu/data
ros2 topic echo /odom/filtered

# TF 트리 확인
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo odom base_link

# 카메라 영상 확인
ros2 run v4l2_camera v4l2_camera_node
rqt_image_view
```

### 주행 데이터 기록 및 재생

```bash
# 기록
ros2 bag record /scan /tf /tf_static /odom /odom/filtered /imu/data /cmd_vel

# 재생
ros2 bag play <bag_directory>
```

---

## 🧩 시스템 구조

### TF 트리

```text
map
└── odom
    └── base_link
        ├── laser_frame
        ├── imu_link
        └── camera_link
```

### 주요 토픽

| Topic | 설명 |
| --- | --- |
| `/scan` | LiDAR LaserScan 데이터 |
| `/imu/data` | IMU 센서 데이터 |
| `/odom` | 휠 기반 odometry |
| `/odom/filtered` | EKF로 보정된 odometry |
| `/cmd_vel` | 로봇 속도 명령 |
| `/map` | SLAM 또는 저장된 지도 |
| `/amcl_pose` | AMCL 기반 추정 위치 |
| `/v4l2_camera/image_raw` | 카메라 원본 이미지 |

---

## 📁 폴더 구조

```text
ros2-ugv-factory-safety/
├── CMakeLists.txt              # 빌드 설정 (ament_cmake)
├── package.xml                 # ROS2 패키지 매니페스트
│
├── launch/                     # 실행 순서대로 번호가 매겨진 launch 파일
│   ├── 1.robot.launch.py       #   로봇 본체 (ros2_control, twist_mux)
│   ├── 2.sllidar_a1.launch.py  #   LiDAR 드라이버
│   ├── 3.slam_toolbox.launch.py#   SLAM
│   └── 4.navigation_launch.py  #   Nav2 자율주행
│
├── params/                     # 노드별 파라미터
│   ├── controller_manager_params.yaml
│   ├── ekf.yaml                #   IMU + odometry 센서 융합
│   ├── nav2_params.yaml        #   AMCL, costmap, controller
│   ├── slam_toolbox_params.yaml
│   └── twist_mux_params.yaml   #   cmd_vel 입력 우선순위
│
├── hardware/                   # ros2_control 하드웨어 인터페이스
│   ├── diffbot_system.cpp      #   Arduino 시리얼 통신 기반 diff drive
│   ├── diffdrive_arduino.xml   #   pluginlib 플러그인 선언
│   └── include/diffdrive_arduino/
│
├── src/                        # LiDAR ROS2 노드
│   ├── sllidar_node.cpp        #   /scan 토픽 발행
│   └── sllidar_client.cpp      #   스캔 데이터 확인용 클라이언트
│
├── sdk/                        # Slamtec LiDAR SDK (외부 라이브러리)
│
└── map/                        # 저장된 점유 격자 지도
    ├── *.pgm / *.yaml          #   지도 이미지 및 메타데이터
    └── *.data / *.posegraph    #   slam_toolbox 직렬화 데이터
```

---

## 🔧 트러블슈팅

개발 중 실제로 마주친 문제와 해결 방법입니다.

<details>
<summary><b>USB 시리얼 포트 번호가 계속 바뀝니다</b></summary>

Linux는 USB 장치 연결 순서에 따라 `/dev/ttyUSB0`, `/dev/ttyUSB1`이 바뀝니다. 고정 경로인 `/dev/serial/by-path/`를 사용하세요.

```bash
ls -l /dev/serial/by-path/
```

</details>

<details>
<summary><b>고속 회전 시 SLAM 지도가 찌그러집니다</b></summary>

odometry만으로는 회전 시 yaw 추정 오차가 누적됩니다. IMU를 추가하고 EKF(`params/ekf.yaml`)로 융합해 `/odom/filtered`를 사용하세요. 함께 `max_vel_theta`와 AMCL의 particle 관련 파라미터도 조정이 필요합니다.

</details>

<details>
<summary><b>YOLO 노드의 subscription count가 0입니다</b></summary>

실제 카메라 토픽 이름과 코드에서 구독하는 토픽이 다른 경우입니다. `ros2 topic list`로 실제 토픽명을 확인한 뒤 맞추세요. QoS 설정 불일치도 원인이 될 수 있습니다.

</details>

<details>
<summary><b>Package / node not found 오류가 납니다</b></summary>

```bash
cd ~/ros2_ws
colcon build --packages-select factory_ugv
source install/setup.bash
ros2 pkg list | grep factory_ugv
```

</details>

<details>
<summary><b>Message dropping이 발생합니다</b></summary>

카메라·LiDAR·YOLO 추론이 동시에 돌면 처리 부하로 메시지가 유실됩니다. 카메라 frame rate 조정, YOLO 모델 경량화, QoS queue size 조정, 불필요한 토픽 발행 제거를 검토하세요.

</details>


---
