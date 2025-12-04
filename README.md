# FLIR Lepton ROS2 
**Gemini 3**로 작성되었습니다.
### ✏️ 준비물
1. **HW** : FLIR Lepton 2.5 + Pure Thermal 2 보드

2. **개발환경**: Ubuntu 22.04 + ROS 2 Humble

### 💾 환경 설정 및 필수 패키지 설치
먼저 시스템에 필요한 도구와 ROS 2, 카메라 관련 라이브러리를 설치합니다.
터미널을 열고 다음 명령어들을 순서대로 입력하세요:
```bash
# 1. 시스템 업데이트
sudo apt update && sudo apt upgrade -y

# 2. ROS 2 Humble 설치 확인 (아직 설치 안 했다면)
# (이미 설치되어 있다면 이 단계는 건너뛰셔도 됩니다)
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt update
sudo apt install ros-humble-desktop -y

# 3. 빌드 도구 및 이미지 처리 라이브러리 설치
sudo apt install python3-colcon-common-extensions python3-pip -y
sudo apt install ros-humble-cv-bridge ros-humble-vision-opencv -y
pip3 install opencv-python numpy

# 4. v4l2 도구 설치 (카메라 장치 확인용)
sudo apt install v4l2-utils -y
```

### 🎥 카메라 연결 확인
FLIR Lepton(PureThermal)을 USB에 연결한 후 다음 명령어로 장치를 확인합니다.
```bash
ls /dev/video*
```
보통 `/dev/video0` 또는 `/dev/video2` 등으로 잡힙니다. 정확히 Lepton인지 확인하려면 다음 명령어를 사용하세요 (video0이라 가정):
```bash
v4l2-ctl -d /dev/video0 --list-formats-ext
```
출력 결과에 `'Y16 ' (16-bit Greyscale)` 포맷이 보이면 Lepton 카메라가 맞습니다.

### 📂 ROS2 패키지 생성 (빌드 파일 작성)
ROS 2 워크스페이스를 만들고 Python 기반의 패키지를 생성합니다.
```bash
# 워크스페이스 생성
mkdir -p ~/lepton_ws/src
cd ~/lepton_ws/src

# 패키지 생성 (이름: flir_lepton_ros)
ros2 pkg create --build-type ament_python flir_lepton_ros --dependencies rclpy sensor_msgs cv_bridge
```
이제 생성된 패키지 안에 코드를 작성합니다.

**▶ `package.xml` 수정:** 

`~/lepton_ws/src/flir_lepton_ros/package.xml` 파일을 열어 아래 의존성을 확인(없으면 추가)합니다.
```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>flir_lepton_ros</name>
  <version>0.0.0</version>
  <description>FLIR Lepton ROS2 Driver for Ubuntu</description>
  <maintainer email="user@todo.todo">user</maintainer>
  <license>TODO: License declaration</license>

  <depend>rclpy</depend>
  <depend>sensor_msgs</depend>
  <depend>cv_bridge</depend>

  <test_depend>ament_copyright</test_depend>
  <test_depend>ament_flake8</test_depend>
  <test_depend>ament_pep257</test_depend>
  <test_depend>python3-pytest</test_depend>

  <export>
    <build_type>ament_python</build_type>
  </export>
</package>
```

**▶ 노드 코드 작성 (`lepton_node.py`):**

Lepton의 **16-bit Raw 데이터(온도 정보 포함)**와 8-bit 시각화용 데이터(컬러맵 적용) 두 가지 토픽을 발행하도록 작성했습니다.

`~/lepton_ws/src/flir_lepton_ros/flir_lepton_ros/lepton_node.py` 파일을 생성하고 아래 내용을 붙여넣으세요.
```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
import cv2
import numpy as np

class LeptonPublisher(Node):
    def __init__(self):
        super().__init__('lepton_publisher')
        
        # 파라미터 선언 (기본값: /dev/video0)
        self.declare_parameter('device_id', 0)
        self.device_id = self.get_parameter('device_id').value

        # 토픽 발행자 생성
        # 1. 시각화용 8비트 컬러 이미지 (RViz에서 보기 좋음)
        self.publisher_color = self.create_publisher(Image, 'thermal/image_color', 10)
        # 2. 알고리즘용 16비트 Raw 이미지 (실제 온도 데이터 분석용)
        self.publisher_raw = self.create_publisher(Image, 'thermal/image_raw', 10)
        
        self.timer = self.create_timer(0.1, self.timer_callback) # 10Hz
        self.bridge = CvBridge()
        
        # OpenCV 비디오 캡처 초기화
        self.cap = cv2.VideoCapture(self.device_id, cv2.CAP_V4L2)
        
        # Lepton은 Y16 포맷(16bit)을 사용합니다.
        # 이 설정이 중요합니다. RGB 변환을 끄고 Raw 데이터를 받습니다.
        self.cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc('Y', '1', '6', ' '))
        self.cap.set(cv2.CAP_PROP_CONVERT_RGB, 0)

        if not self.cap.isOpened():
            self.get_logger().error(f'Could not open video device: /dev/video{self.device_id}')
            exit()
        else:
            self.get_logger().info(f'Lepton camera opened on /dev/video{self.device_id}')

    def timer_callback(self):
        ret, frame = self.cap.read()
        
        if ret:
            # frame은 현재 (Height, Width, 1)의 uint16 형태입니다.
            
            # 1. Raw Data (16-bit) 발행
            # Odometry 등에 사용할 때는 이 데이터를 써야 정보 손실이 없습니다.
            raw_msg = self.bridge.cv2_to_imgmsg(frame, encoding="mono16")
            raw_msg.header.stamp = self.get_clock().now().to_msg()
            raw_msg.header.frame_id = "lepton_frame"
            self.publisher_raw.publish(raw_msg)

            # 2. Color Data (8-bit Visualization) 발행
            # 16비트 데이터를 0~255 범위로 정규화(Normalize)합니다.
            # cv2.normalize를 사용하여 최소값(차가운 부분)~최대값(뜨거운 부분)을 0~255로 매핑
            frame_norm = cv2.normalize(frame, None, 0, 255, cv2.NORM_MINMAX)
            frame_uint8 = np.uint8(frame_norm)
            
            # 컬러맵 적용 (INFERNO가 열화상에 보기 좋음, JET도 많이 사용)
            frame_color = cv2.applyColorMap(frame_uint8, cv2.COLORMAP_INFERNO)
            
            color_msg = self.bridge.cv2_to_imgmsg(frame_color, encoding="bgr8")
            color_msg.header.stamp = self.get_clock().now().to_msg()
            color_msg.header.frame_id = "lepton_frame"
            self.publisher_color.publish(color_msg)
            
        else:
            self.get_logger().warn('Failed to capture frame')

    def __del__(self):
        if self.cap.isOpened():
            self.cap.release()

def main(args=None):
    rclpy.init(args=args)
    node = LeptonPublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

**▶ 빌드 과정:**

작성한 패키지를 빌드합니다.
```bash
cd ~/lepton_ws

# 의존성 해결 (필수는 아니지만 권장)
rosdep install -i --from-path src --rosdistro humble -y

# 빌드 수행
colcon build --symlink-install
```
빌드가 성공적으로 끝나면 `Summary: 1 package finished` 메시지가 뜹니다.

### 🖼️ 실행 및 RViz 시각화

USB 카메라 장치에 접근 권한이 필요할 수 있습니다. (장치를 뺐다 끼면 초기화될 수 있음)
```bash
sudo chmod 666 /dev/video0  # 본인의 장치 번호에 맞게 수정
```

터미널을 열고 환경 변수를 로드한 뒤 실행합니다.
```bash
cd ~/lepton_ws
source install/setup.bash

# 노드 실행 (기본 /dev/video0)
ros2 run flir_lepton_ros lepton_node
```

만약 `/dev/video1` 등 다른 포트라면: `ros2 run flir_lepton_ros lepton_node --ros-args -p device_id:=1`

새로운 터미널을 열고 RViz2를 실행합니다.

```bash
ros2 run rviz2 rviz2
```

**RViz 설정 방법:**

1. 왼쪽 하단 **[Add]** 버튼 클릭.

2. **By topic** 탭 선택.

3. `/thermal/image_color` → Image를 선택하고 OK 클릭.

   - 화면에 화려한 색상(INFERNO 컬러맵)의 열화상 영상이 나와야 합니다.

4. `/thermal/image_raw` → Image를 선택하고 OK 클릭.

   - 화면에 회색조의 이미지가 나옵니다.

   - 주의: 16-bit 이미지는 RViz 기본 설정에서 매우 어둡게 보일 수 있습니다. (설정 패널에서 `Normalize Range`를 체크하면 잘 보일 수 있습니다.)

5. Global Options의 Fixed Frame에 `lepton_frame` 이라고 직접 타이핑해 넣으세요 (경고 메시지를 없애기 위함).
