**Hướng dẫn này tạo node trên Docker container với quyền chi sẻ Folder giữa Ubuntu và container (Trên Jetson nano)**
# Tạo docker container với full quyền truy cập phần cứng và share_foler

Giả sử thư mục trên Ubuntu là _```/home/jetson/human_detection_yolo3tiny/tmp```_ và trên container là _```/root/shared_folder```_, khi đó lệnh tạo container:
```
sudo docker run -it \
  --name ros-humble-container \
  -v /dev:/dev \
  --privileged \
  --runtime nvidia \
  --network host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $XAUTHORITY:/root/.Xauthority \
  -v /home/jetson/human_detection_yolo3tiny/tmp:/root/shared_folder \
  dustynv/ros:humble-desktop-l4t-r36.4.0
```
...sau đó vào container và kiểm tra thư mục liên kết:

```
docker exec -it ros-humble-container /bin/bash
cd
ls
cd shared_folder
```
lúc này file ```person_detection.json``` có tồn tại.

# create-a-ROS2-node

create a ROS2 node for read Json file output from **```person_detector_with_file_output.py```** and publish /cmd_vel topic to control my 2 wheels robot.

1. Tạo workspace và package
Tạo một workspace (nếu chưa có):
```
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
source install/setup.bash
```

Tạo một package mới trong workspace:
```
cd ~/ros2_ws/src
ros2 pkg create my_python_package --build-type ament_python --dependencies rclpy
```

2. Thêm mã nguồn Python
Di chuyển vào thư mục package:
```
cd ~/ros2_ws/src/my_python_package

```
Tạo thư mục my_python_package (chứa mã nguồn Python):
```
mkdir my_python_package

```

Tạo file Python (ví dụ simple_node.py) bên trong thư mục my_python_package:
```
touch my_python_package/simple_node.py
chmod +x my_python_package/simple_node.py

```
Viết mã Python trong file simple_node.py:
```
# it read data from a Json file
# output of json like: {"timestamp": 1736010539.8562458, "detected": false, "tracking": false, "distance": null, "x_center": 0.0, "search_command": 0.0}

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import json
import time
from pathlib import Path

class SimpleFollowerNode(Node):
    def __init__(self):
        super().__init__('simple_follower')
        
        # Create publisher for robot velocity commands
        self.publisher_ = self.create_publisher(Twist, 'cmd_vel', 10)
        
        # Create timer for checking detection data
        self.timer = self.create_timer(0.1, self.timer_callback)  # 10Hz
        
        # Parameters
        self.target_distance = 1.0  # meters
        self.max_linear_speed = 0.5  # m/s
        self.max_angular_speed = 1.82  # rad/s
        self.search_angular_speed = 1.0  # rad/s for search mode
        
        # Update file path to use absolute path in Docker container
        self.detection_file_path = Path('/root/shared_folder/tmp/person_detection.json')
        
        self.get_logger().info(f'Simple follower node initialized. Watching file: {self.detection_file_path}')
        
        # Verify file exists at startup
        if self.detection_file_path.exists():
            self.get_logger().info('Detection file found successfully!')
        else:
            self.get_logger().error(f'Detection file not found at startup. Please check path: {self.detection_file_path}')

    def timer_callback(self):
        try:
            # Check file existence
            if not self.detection_file_path.exists():
                self.get_logger().warn(f'Detection file not found at: {self.detection_file_path}')
                self.stop_robot()
                return
                
            # Read detection data from file
            with open(self.detection_file_path, 'r') as f:
                data = json.load(f)
            
            # Log raw data occasionally
            self.get_logger().info(f'Detection data: {data}')
            
            msg = Twist()
            
            # Check if person is detected with valid distance
            if data['detected'] and data['distance'] is not None:
                # Calculate linear velocity
                distance_error = data['distance'] - self.target_distance
                
                if abs(distance_error) > 0.1:  # 10cm deadband
                    linear_speed = self.max_linear_speed * min(abs(distance_error), 1.0)
                    msg.linear.x = linear_speed if distance_error > 0 else -linear_speed
                
                # Calculate angular velocity based on person position
                if abs(data['x_center']) > 0.1:
                    msg.angular.z = -self.max_angular_speed * data['x_center']
                
            else:
                # No person detected, use search_command if available
                if 'search_command' in data:
                    # search_command ranges from -1 to 1
                    search_velocity = data['search_command'] * self.search_angular_speed
                    msg.angular.z = search_velocity
                    self.get_logger().info(f'Searching with angular velocity: {search_velocity:.2f}')
                else:
                    self.get_logger().info('No valid detection and no search command, stopping robot')
                    msg.angular.z = 0.0
                
                # Stop forward motion when searching
                msg.linear.x = 0.0
            
            # Publish command
            self.publisher_.publish(msg)
            self.get_logger().info(
                f'cmd_vel: linear={msg.linear.x:.2f}, angular={msg.angular.z:.2f}'
            )
                
        except Exception as e:
            self.get_logger().error(f'Error in timer callback: {str(e)}')
            self.stop_robot()

    def stop_robot(self):
        msg = Twist()
        msg.linear.x = 0.0
        msg.angular.z = 0.0
        self.publisher_.publish(msg)

def main(args=None):
    rclpy.init(args=args)
    node = SimpleFollowerNode()
    
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

3. Chỉnh sửa file setup.py
Mở file setup.py và chỉnh sửa như sau:

```
from setuptools import setup

package_name = 'my_python_package'

setup(
    name=package_name,
    version='0.0.0',
    packages=[package_name],
    py_modules=[],
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='your_name',
    maintainer_email='your_email@example.com',
    description='Example Python package for ROS 2',
    license='Apache License 2.0',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            'simple_node = my_python_package.simple_node:main',
        ],
    },
)

```
4. Build package
Quay lại thư mục workspace:

```
cd ~/ros2_ws
colcon build --packages-select my_python_package
source install/setup.bash
```

5. Chạy node
Chạy node simple_node:
```
ros2 run my_python_package simple_node

```
Nếu bạn muốn source tự động mỗi khi mở terminal, hãy thêm dòng này vào file ~/.bashrc:

```
echo "source /opt/ros/foxy/setup.bash" >> ~/.bashrc
source ~/.bashrc
```
or 
```
echo "source /opt/ros/foxy/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```
Để tự động source workspace, thêm vào file ~/.bashrc:

```
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

Xử lý khi không tìm thấy lệnh ros2
Nếu vẫn gặp lỗi sau khi source, hãy kiểm tra các vấn đề sau:

```
echo $PATH
export PATH=/opt/ros/foxy/bin:$PATH
```

