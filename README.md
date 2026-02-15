
# Commands to look around

Check active topics:

```bash
ros2 topic list
```

Check topic info:

```bash
ros2 topic info /topic
```

Check topic activity:

```bash
ros2 topic hz /topic
```

Echo topic:

```bash
ros2 topic echo /topic
```

Run vizualization
```bash
rviz2
```

Add by type any `LaserScan` topic, modify fixed frame to visualize the LIDAR data. 





# Create your own ROS 2 Workspace

In a terminal (shortcut `Ctrl-Alt-t`)

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
```

Run the setup script (source it):

```bash
source install/setup.bash
```

# Create a Python Package

Go to the `src` folder:

```bash
cd ~/ros2_ws/src
```

Create a Python package:

```bash
ros2 pkg create --build-type ament_python py_pubsub
```

# Create the Publisher Node

Create the publisher file and open it in an editor (suggested: vscode):

```bash
cd py_pubsub/py_pubsub
touch simple_publisher.py
chmod +x simple_publisher.py
```

Copy the following code into `simple_publisher.py` (replace `topic` with any unique name!):

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class SimplePublisher(Node):

    def __init__(self):
        super().__init__('simple_publisher')
        self.publisher_ = self.create_publisher(String, 'topic', 10)
        self.timer = self.create_timer(1.0, self.timer_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello ROS 2: {self.count}'
        self.publisher_.publish(msg)
        self.get_logger().info(f'Publishing: "{msg.data}"')
        self.count += 1


def main(args=None):
    rclpy.init(args=args)
    node = SimplePublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Save the file.

Run in the terminal with
```bash
python3 simple_publisher.py
```

In a second terminal inspect the topic with the commands from the first section.

# Create the Subscriber Node

Stop the publisher (`Ctrl-C`) and in the same place create the subscriber file:

```bash
touch simple_subscriber.py
chmod +x simple_subscriber.py
```

Edit the file, adding the code:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class SimpleSubscriber(Node):

    def __init__(self):
        super().__init__('simple_subscriber')
        self.subscription = self.create_subscription(
            String,
            'topic',
            self.listener_callback,
            10)
        self.subscription  # prevent unused variable warning

    def listener_callback(self, msg):
        self.get_logger().info(f'I heard: "{msg.data}"')


def main(args=None):
    rclpy.init(args=args)
    node = SimpleSubscriber()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Run the publisher in one terminal and the subscriber in the other, verify the communication works.


# Turn the scripts into full ros package

Go to base folder of the package:

```bash
cd ~/ros2_ws/src/py_pubsub
```

Modify in `setup.py` the `entry_points`  section:

```python
entry_points={
    'console_scripts': [
        'publisher = py_pubsub.simple_publisher:main',
        'subscriber = py_pubsub.simple_subscriber:main',
    ],
},
```

Build the Package (Important! always build package in workspace root!):

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

# Run the publisher and subscriber as nodes

In two separate terminal (remember to source `~/ros2_ws/install/setup.bash` in each!)

```bash
ros2 run py_pubsub publisher
ros2 run py_pubsub subscriber
```

# Redo the same examples in C++

Create a C++ package (note the change in the command!)

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake cpp_pubsub --dependencies rclcpp std_msgs
cd cpp_pubsub/src
touch simple_publisher.cpp
touch simple_subscriber.cpp
```

Edit `simple_publisher.cpp` (use the same topic name you used in the python example!)

```cpp
#include <chrono>
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class SimplePublisher : public rclcpp::Node
{
public:
    SimplePublisher()
    : Node("simple_publisher_cpp"), count_(0)
    {
        publisher_ = this->create_publisher<std_msgs::msg::String>("topic", 10);
        timer_ = this->create_wall_timer(
            1s,
            std::bind(&SimplePublisher::timer_callback, this));
    }

private:
    void timer_callback()
    {
        auto message = std_msgs::msg::String();
        message.data = "Hello ROS 2 C++: " + std::to_string(count_++);
        RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", message.data.c_str());
        publisher_->publish(message);
    }

    rclcpp::TimerBase::SharedPtr timer_;
    rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
    size_t count_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<SimplePublisher>());
    rclcpp::shutdown();
    return 0;
}
```

And similarly for the subscriber:

```cpp
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class SimpleSubscriber : public rclcpp::Node
{
public:
    SimpleSubscriber()
    : Node("simple_subscriber_cpp")
    {
        subscription_ = this->create_subscription<std_msgs::msg::String>(
            "topic",
            10,
            std::bind(&SimpleSubscriber::topic_callback, this, std::placeholders::_1));
    }

private:
    void topic_callback(const std_msgs::msg::String::SharedPtr msg) const
    {
        RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());
    }

    rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<SimpleSubscriber>());
    rclcpp::shutdown();
    return 0;
}
```

# Configure the build for the nodes

```bash
cd ~/ros2_ws/src/cpp_pubsub
```

Edit `CMakeLists.txt` to contain:

```cmake
cmake_minimum_required(VERSION 3.8)
project(cpp_pubsub)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(publisher src/simple_publisher.cpp)
ament_target_dependencies(publisher rclcpp std_msgs)

add_executable(subscriber src/simple_subscriber.cpp)
ament_target_dependencies(subscriber rclcpp std_msgs)

install(TARGETS
  publisher
  subscriber
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```

When completed. build:

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

And now you are ready to run

```bash
ros2 run cpp_pubsub publisher
ros2 run cpp_pubsub subscriber
```

Verify they communicate with each other and that they also communicate with the python nodes.

# Modify the examples to work with more complex messages

The message type `geometry_msgs/msg/Vector3` from the package `geometry_msgs` represents a 3D vector.
It is used to describe:
- 3D position
- Velocity (linear or angular)
- Acceleration
- Force

Message Definition
```bash
ros2 interface show geometry_msgs/msg/Vector3
```

```text
float64 x
float64 y
float64 z
```

Now, let us replace the publishers and subscribers to send vectors instead of strings. 
You may choose python or C++ whichever you prefer.


##  Changes for python 

In both scripts in the import section enter the proper message type

```python
from geometry_msgs.msg import Vector3
import math
```

In the publisher, change the topic type:

```python
self.publisher_ = self.create_publisher(Vector3, 'topic', 10)
```

and replce the callback function with:
```python
def timer_callback(self):
    msg = Vector3()
    msg.x = float(self.count)
    msg.y = float(self.count * 2)
    msg.z = math.sin(self.count)

    self.publisher_.publish(msg)

    self.get_logger().info(
        f'Publishing: x={msg.x:.2f}, y={msg.y:.2f}, z={msg.z:.2f}'
    )

    self.count += 1
```

In subscriber defince the subscriber as:

```python
self.subscription = self.create_subscription(
    Vector3,
    'vector_topic',
    self.listener_callback,
    10)
```

and replace the callback function with:

```python
def listener_callback(self, msg):
    self.get_logger().info(
        f'I heard: x={msg.x:.2f}, y={msg.y:.2f}, z={msg.z:.2f}'
    )
```

Finally, modify the package configuration in `package.xml` ensure dependency exists:
```xml
<depend>geometry_msgs</depend>
```

In the build terminal, build, source and run, validating the results.

## Changes for C++ 

Add the message headers
```cpp
#include "geometry_msgs/msg/vector3.hpp"
#include <cmath>
```

Change topic type:

```cpp
rclcpp::Publisher<geometry_msgs::msg::Vector3>::SharedPtr publisher_;
// ...
publisher_ = this->create_publisher<geometry_msgs::msg::Vector3>("vector_topic", 10);
```

Replace the callback function with:

```cpp
void timer_callback()
{
    auto message = geometry_msgs::msg::Vector3();
    message.x = static_cast<double>(count_);
    message.y = static_cast<double>(count_ * 2);
    message.z = std::sin(count_);

    RCLCPP_INFO(
        this->get_logger(),
        "Publishing: x=%.2f y=%.2f z=%.2f",
        message.x, message.y, message.z
    );

    publisher_->publish(message);
    count_++;
}
```

And in the subscriber

```cpp
rclcpp::Subscription<geometry_msgs::msg::Vector3>::SharedPtr subscription_;
// ...
subscription_ = this->create_subscription<geometry_msgs::msg::Vector3>(
    "vector_topic",
    10,
    std::bind(&SimpleSubscriber::topic_callback, this, std::placeholders::_1));
```

Replace the callback function:

```cpp
void topic_callback(const geometry_msgs::msg::Vector3::SharedPtr msg) const
{
    RCLCPP_INFO(
        this->get_logger(),
        "I heard: x=%.2f y=%.2f z=%.2f",
        msg->x, msg->y, msg->z
    );
}
```

Then in `CMakeLists.txt` modify proper lines

```cmake
find_package(geometry_msgs REQUIRED)

ament_target_dependencies(publisher rclcpp std_msgs geometry_msgs)
ament_target_dependencies(subscriber rclcpp std_msgs geometry_msgs)
```

Build and source. Test the result.

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

# Working with own messages

Note: avoid creating new messages without a need, as it will forbid easy interfacing to existing packages.
Check first if no existing message type fits.

Sometimes there is nothing that matches the needs, then...


We want to create a type `ColorNumber.msg`
With the following structure
```cpp
string color
float64 number
```

We will use interface package to define a message that will be compiled into C++ and Python code (and DDS) and then you may use it with any other package.


Create interface package
```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake my_interfaces
cd my_interfaces
mkdir msg
touch msg/ColorNumber.msg
```

Edit `msg/ColorNumber.msg` file to contain
```text
string color
float64 number
```

Configure the interface, editing `package.xml` to contain `rosidl` references:
```xml
<buildtool_depend>ament_cmake</buildtool_depend>

<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>

<member_of_group>rosidl_interface_packages</member_of_group>
```

Modify CMakeLists.txt to contain:

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/ColorNumber.msg"
)

ament_export_dependencies(rosidl_default_runtime)

ament_package()
```

Build only this package:

```bash
cd ~/ros2_ws
colcon build --packages-select my_interfaces
source install/setup.bash
```

Verify Message Was Generated

```bash
ros2 interface show my_interfaces/msg/ColorNumber
```

Now - modify publisher and subsciber to use the new message type

Hints

## in python

```python
from my_interfaces.msg import ColorNumber
#....
msg = ColorNumber()
#....
msg.color = "red"
msg.number = 3.14
```

Subscriber:

```python
def listener_callback(self, msg):
    self.get_logger().info(
        f'Received: color={msg.color}, number={msg.number}'
    )
```

Add dependency in `package.xml`

```xml
<depend>my_interfaces</depend>
```

## in C++

```cpp
#include "my_interfaces/msg/color_number.hpp"
//...
rclcpp::Publisher<my_interfaces::msg::ColorNumber>::SharedPtr publisher_;
```


Subscriber:

```cpp
void topic_callback(const my_interfaces::msg::ColorNumber::SharedPtr msg)
{
    RCLCPP_INFO(
        this->get_logger(),
        "Received: color=%s number=%.2f",
        msg->color.c_str(),
        msg->number
    );
}
```

Add dependency in `package.xml`

```xml
<depend>my_interfaces</depend>
```

And in  `CMakeLists.txt`:

```cmake
find_package(my_interfaces REQUIRED)


ament_target_dependencies(publisher rclcpp my_interfaces)
```

Now update the packages and test.


# Communicate with the robot

## LIDAR - read

Check the structure of the robot `sensor_msgs/LaserScan` message.
Try to modify the subscriber to read it.


## Motor - write

Check the structure of the velocity message type (`geometry_msgs/Twist`).
Write a  publisher sending angular `z` value.


