# Emotion Pilot — Điều khiển TurtleBot bằng biểu cảm

Giao diện desktop PyQt5, webcam OpenCV, nhận diện bằng DeepFace và điều khiển
ROS 2 Humble qua `geometry_msgs/msg/Twist` trên `/cmd_vel`.
Đây là package ROS 2 độc lập `facial_expression_control`, nằm tại
`~/ros2_ws/src/facial_expression_control`.

## 1. Cài môi trường

Dùng Ubuntu 22.04 / Python 3.10 / ROS 2 Humble. Các lệnh sau chạy tại thư mục gốc dự án:

```bash
cd ~/ros2_ws/src/facial_expression_control
sudo apt install python3-venv python3-pyqt5 python3-opencv
python3 -m venv --system-site-packages .venv-emotion
source .venv-emotion/bin/activate
export PYTHONNOUSERSITE=1
python -m pip install -r requirements.txt
```

DeepFace/TensorFlow là các thư viện lớn. Lần khởi động đầu mô hình Emotion sẽ được tải
về cache `~/.deepface/weights`, cần kết nối Internet. Trong lúc tải, camera vẫn hiển thị,
robot chờ kết quả hợp lệ. Không cài các gói này vào môi trường giọng nói cũ nếu muốn giữ
hai bộ phụ thuộc riêng biệt.

## 2. Nhận diện và xem giao diện, chưa kết nối robot

```bash
cd ~/ros2_ws/src/facial_expression_control
source .venv-emotion/bin/activate
export PYTHONNOUSERSITE=1
python -m facial_expression_control --preview
```

`--preview` dùng nhận diện thật nhưng không tạo node ROS và không publish lệnh.
Đổi webcam bằng `--camera 1` nếu cần. Webcam phải sẵn sàng và không bị ứng dụng khác chiếm.

Có thể dùng `--demo` thay `--preview` để xem giao diện với các xác suất giả lập luân phiên,
không cần DeepFace. Chế độ này vẫn dùng webcam và luôn tắt kết nối ROS.
Dữ liệu demo được ghi rõ trên hình và thanh trạng thái.

## 3. Chạy Gazebo với môi trường trống và điều khiển robot

Tất cả launch, world và cấu hình Gazebo của bài nằm trong package này.
Không gọi `tb4_voice_navigation`, không mở căn nhà, Nav2, SLAM hoặc RViz.
World chỉ có mặt sàn 200 × 200 m, ánh sáng và TurtleBot 4; không có tường,
đồ vật hoặc dock. Robot khởi đầu tại gốc tọa độ; không cần gửi lệnh undock.
Các package TurtleBot/Gazebo chuẩn vẫn là phụ thuộc mô phỏng cần cài sẵn.

Đóng các phiên Gazebo/điều khiển trước đó trước khi mở phiên mới.

**Build package mới một lần, hoặc sau khi sửa launch/world:**

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build --packages-select facial_expression_control --symlink-install
```

**Terminal 1 — Gazebo và robot:**

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
export ROS_DOMAIN_ID=24
ros2 launch facial_expression_control empty_sim.launch.py
```

Đợi robot xuất hiện và mô phỏng đang chạy (nút Play). Có thể kéo/zoom góc nhìn
để quan sát đường đi. Nếu chỉ cần server, thêm `headless:=true`.

**Terminal 2 — giao diện nhận diện và điều khiển:**

```bash
cd ~/ros2_ws/src/facial_expression_control
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
source .venv-emotion/bin/activate
export PYTHONNOUSERSITE=1
export ROS_DOMAIN_ID=24
python -m facial_expression_control --cmd-vel /cmd_vel --odom /odom
```

Chờ DeepFace sẵn sàng, nhấn **BẬT ĐIỀU KHIỂN** và giữ biểu cảm.
Quan sát robot trong Gazebo; nhấn **Space** trong cửa sổ Emotion Pilot để dừng.

Hai terminal phải có cùng `ROS_DOMAIN_ID`. Dùng Python của venv để nhận DeepFace;
không cần build lại khi chỉ chạy trực tiếp bằng `python -m` sau khi sửa mã Python.
Với robot thật, thay topic bằng `--cmd-vel /robot1/cmd_vel --odom /robot1/odom`
nếu có namespace, và undock theo yêu cầu của robot thật.
Ứng dụng publish `Twist`; hệ thống dùng `TwistStamped` cần bộ chuyển đổi.

## 4. Thao tác trên giao diện

1. Chờ camera và dòng **DeepFace • Sẵn sàng**.
2. Chọn tốc độ tiến/lùi (mặc định 0,10 m/s) và ngưỡng tin cậy (mặc định 60%).
3. Nhấn **BẬT ĐIỀU KHIỂN**. Robot chưa chạy cho tới khi nhận biểu cảm ổn định.
4. Giữ biểu cảm ít nhất 0,7 giây qua các lần suy luận liên tiếp.
5. Sau mỗi động tác, giữ **Neutral** ổn định rồi thực hiện biểu cảm tiếp theo.
6. Nhấn **DỪNG ROBOT**, phím **Space**, hoặc thoát bằng **Esc** để dừng phiên.
   Sau khi dừng, cần nhấn bật điều khiển lại.

| Biểu cảm | Lệnh mặc định | Kết thúc |
|---|---|---|
| Happy | Tiến 0,10 m/s; xoay trái 0,45 rad/s | Sau 3 giây |
| Sad | Tiến 0,10 m/s; xoay phải −0,45 rad/s | Sau 3 giây |
| Neutral | Vận tốc bằng 0 | Cho phép nhận động tác mới |
| Angry | Lùi −0,10 m/s | Sau 1,2 giây |
| Surprise | Xoay trái tại chỗ 0,45 rad/s | Tích lũy góc odometry khoảng 360° |
| Fear | Tiến 0,10 m/s | Sau 1,2 giây |

DeepFace trả thêm `disgust`; biểu cảm này không phát lệnh. Khi có nhiều mặt, chỉ chọn mặt lớn nhất nếu diện tích lớn hơn 1,8 lần mặt đứng thứ hai; nếu không thì không phát lệnh. Robot dừng khi kết quả dưới ngưỡng tin cậy, không có mặt,
đổi biểu cảm đang chờ xác nhận, hoặc kết quả nhận diện cũ quá 2 giây. Camera ngắt hoặc
không có hình mới trong 0,7 giây cũng khiến robot dừng.

Xoay 360° dựa trên góc yaw tích lũy từ odometry, bao gồm xử lý vượt biên ±π;
không thực hiện khi odometry cũ hơn 0,5 giây. Có giới hạn thời gian nếu robot bị kẹt.
Sai số thực tế phụ thuộc odometry, chu kỳ điều khiển và quán tính robot.

Lệnh vận tốc được publish 20 Hz từ luồng riêng. Khi thoát bình thường, ứng dụng gửi
lệnh 0. Nếu tiến trình bị kill hoặc máy mất nguồn, cần watchdog vận tốc phía robot;
ứng dụng không thể gửi lệnh khi tiến trình đã bị kết thúc cưỡng bức.
Không có tránh vật cản trong module này: chạy thử trên khoảng trống với tốc độ thấp.

## Cấu trúc

- `launch/empty_sim.launch.py`: mở Gazebo Fortress và robot trong world trống.
- `launch/spawn_robot.launch.py`: khởi tạo TurtleBot, không spawn dock.
- `worlds/emotion_arena.sdf`: mặt sàn, ánh sáng và plugin mô phỏng.
- `gui/arena.config`: giao diện Gazebo với góc nhìn toàn cảnh.

- `app.py`: giao diện tiếng Việt, bảng xác suất, camera, trạng thái, nút điều khiển.
- `vision.py`: camera và suy luận ở hai luồng nền, xử lý khung hình mới nhất.
- `control.py`: xác nhận biểu cảm, ánh xạ chuyển động, thời hạn dữ liệu và đo vòng quay.
- `ros_bridge.py`: node ROS 2, publisher Twist, subscriber Odometry và vòng điều khiển.
- `requirements.txt`: bộ phụ thuộc riêng cho bài biểu cảm.

Chỉnh thời gian động tác, tốc độ xoay và thời gian xác nhận trong `Settings` ở `facial_expression_control/control.py`.
Camera và nhận diện chạy cục bộ; ứng dụng không gửi hình ảnh tới API bên ngoài.

API tham khảo: [DeepFace](https://github.com/serengil/deepface/tree/v0.0.93).

## Trạng thái bàn giao

Chưa chạy thử ứng dụng, mô hình, webcam hoặc ROS/robot theo yêu cầu của bạn.
Cần xác nhận độ trễ nhận diện, topic thực tế và chiều quay trên môi trường sử dụng.

## Khắc phục lỗi giao diện / Gazebo / pandas

Cột điều khiển có thanh cuộn khi màn hình thấp; thông báo lỗi được xuống dòng.
Rê chuột lên thông báo để xem đầy đủ. Lỗi tải mô hình được in cả traceback ở terminal.

World không khai báo thêm Sensors vì robot description đã nạp plugin này.
Gazebo server và GUI chạy thành hai tiến trình: GUI thoát không tắt controller/server;
Ctrl+C ở terminal launch để đóng cả phiên. Log chi tiết bật ở mức 4.
Không mở thêm phiên Gazebo cho cùng world trong lúc phiên cũ còn chạy.

Nếu báo `C extension: None not built` từ pandas, đóng Emotion Pilot rồi cài lại
cặp wheel NumPy/pandas trong venv, không gỡ thư viện hệ thống:

```bash
cd ~/ros2_ws/src/facial_expression_control
source .venv-emotion/bin/activate
export PYTHONNOUSERSITE=1
python -m pip install --only-binary=:all: --force-reinstall --no-deps numpy==1.26.4 pandas==2.2.3
python -m facial_expression_control --preview
```

Nếu vẫn lỗi, gửi toàn bộ traceback ở terminal, từ `Traceback` đến dòng cuối.
Sau khi cập nhật launch/world, build lại package trước khi mở Gazebo.
Các sửa đổi này chưa được chạy thử theo yêu cầu của bạn.

## Cài đầy đủ thư viện và chạy bằng script

Đóng ứng dụng trước khi cập nhật thư viện. Script luôn tắt user-site từ lúc khởi động Python,
tránh lấy nhầm phụ thuộc từ `~/.local` rồi mất chúng khi bật `PYTHONNOUSERSITE`.

```bash
cd ~/ros2_ws/src/facial_expression_control
bash install.sh
bash run.sh --preview
```

Điều khiển ROS: source ROS và workspace, đặt ROS_DOMAIN_ID như mục 3, rồi `bash run.sh`.
Nút bật điều khiển chỉ sáng khi mô hình đã tải và camera có hình mới.
Nếu có người ở phía sau, ứng dụng ưu tiên mặt tiền cảnh lớn hơn rõ rệt; hai mặt có
kích thước gần nhau sẽ dừng để tránh nhận nhầm người điều khiển.

### TensorFlow không tìm thấy tensorflow.keras

DeepFace 0.0.93 đặt `TF_USE_LEGACY_KERAS=1` trước khi import TensorFlow.
Môi trường TF 2.15.1 cần có `tf-keras==2.15.1` để đường dẫn legacy tồn tại.
Gói này đã được thêm vào requirements, cùng SciPy 1.11.4 tương thích NumPy 1.26.4.
Không cài tf-keras phiên bản mới nhất vì có thể kéo TensorFlow sang phiên bản khác.
Chạy `bash install.sh` để đồng bộ môi trường, rồi đóng/mở lại ứng dụng bằng `bash run.sh`.
