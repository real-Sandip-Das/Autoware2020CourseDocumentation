Various Software "Design Patterns" concepts that motivate ROS Concepts:

| ROS Concept             | Design Patterns equivalent                                            |
| ----------------------- | --------------------------------------------------------------------- |
| ROS Nodes               | Processes/Threads                                                     |
| ROS Topics              | Buses/PubSub                                                          |
| ROS Messages / ROS Bags | Serialization                                                         |
| ROS Params              | Black Board Pattern (like Redis)                                      |
| ROS Services            | Synchronous Remote Procedure Call (RPC)                               |
| ROS Actions             | Asynchronous Remote Procedure Call (RPC)                              |
| ROS Life Cycles         | State Machines                                                        |
| URDF and TF             | (Matrix Math for 3D Operations, no Design Patterns concepts involved) |

## Setting up ROS 2 Environment and Installing some Packages

```bash
cd ~/adehome/AutowareAuto
ade start
ade enter
# Instead of sourcing once, let's to ~/.bashrc and source that
echo "source /opt/ros/foxy/setup.bash" >> ~/.bashrc
source ~/.bashrc
sudo apt update
sudo apt install ros-foxy-turtlesim
sudo apt install ros-foxy-rqt-*
sudo apt install byobu # some may prefer: tmux (none of them are actually required)
```

## Some Nomenclature/Terms

- **Overlay:** A second workspace with more/different/new packages. If there are multiple versions of a package/code then the one at the bottom is used.
- **Underlay:** The workspace(s) that is(are) underneath an overlay. To include a workspace as an underlay, we must source its `install/setup.bash`.
- **Colcon:** The ROS 2 build tool. It can be thought of as a layer above CMake/Make/SetupTools that helps these tools work together smoothly in, say, a large Projects with multiple workspaces.

## Setting up an Example Workspace and Building it

Although I like to use `tmux` as a terminal manager I would prefer opening multiple host terminals and executing `ade enter` in them.

```bash
mkdir -p ~/ros2_example_ws/src && cd ~/ros2_example_ws/src
git clone https://github.com/ros2/examples
cd examples
git checkout foxy
cd ../../
colcon build --symlink-install
```

## Messages, Nodes, Publishers, Subscribers, Topics, Bagging, Serialization, Deserialization

The ideas and working principles of Messages, Nodes, Publishers, Subscribers, Topics, Bagging, Serialization and Deserialization are ROS 2 is analogous to ROS 1

## Running a ROS Node

- **ROS Best Practice: It's recommended to ALWAYS build and execute in different terminals.**

```bash
# In a separate terminal for execution, execute:
source ~/ros2_example_ws/install/setup.bash
# An usual ros2 run command:
# ros2 run <package_name> <executable_name>
ros2 run examples_rclcpp_minimal_publisher publisher_lambda
```
![[publisher_lambda ros2 run.png]]
Example output:
```plaintext
1723303401.253725 [0] publisher_: using network interface wlo1 (udp/10.145.151.172) selected arbitrarily from: wlo1, docker0
[INFO] [1723303401.764791243] [minimal_publisher]: Publishing: 'Hello, world! 0'
[INFO] [1723303402.264747313] [minimal_publisher]: Publishing: 'Hello, world! 1'
[INFO] [1723303402.764742345] [minimal_publisher]: Publishing: 'Hello, world! 2'
[INFO] [1723303403.264739685] [minimal_publisher]: Publishing: 'Hello, world! 3'
[INFO] [1723303403.764767422] [minimal_publisher]: Publishing: 'Hello, world! 4'
[INFO] [1723303404.264771982] [minimal_publisher]: Publishing: 'Hello, world! 5'
[INFO] [1723303404.764629367] [minimal_publisher]: Publishing: 'Hello, world! 6'
[INFO] [1723303405.264615869] [minimal_publisher]: Publishing: 'Hello, world! 7'
[INFO] [1723303405.764722051] [minimal_publisher]: Publishing: 'Hello, world! 8'
^C[INFO] [1723303406.057073515] [rclcpp]: signal_handler(signal_value=2)
```

![[Topic CLI tools.png]]
## Other common CLI Tools

```bash
# If you know about ROS 1, these do what you think these would do
ros2 topic list
ros2 topic echo /topic
ros2 topic hz /topic
ros2 node list
```

## Elaborating on the Publisher Node's Source Code

**Obstacle Encountered:** The path to the source code(`ros2_example_ws/src/examples/rclcpp/minimal_publisher/member_function.cpp`) for this publisher node given in the tutorial doesn't match in ROS 2 Foxy version
**Solution:** A hack for finding the relevant source file, in case we don't know, would be to use the `find` tool e.g. running the command `find ./src -name *minimal_publisher*`(from the root of the workspace), we get the following locations:
```plaintext
./src/examples/rclpy/topics/minimal_publisher
./src/examples/rclpy/topics/minimal_publisher/resource/examples_rclpy_minimal_publisher
./src/examples/rclpy/topics/minimal_publisher/examples_rclpy_minimal_publisher
./src/examples/rclcpp/topics/minimal_publisher
```
Clearly, what we want must be located in the directory `ros2_example_ws/src/examples/rclcpp/topics/minimal_publisher`
After `cd`-ing the directory, we find out that the relevant path is: `ros2_example_ws/src/examples/rclcpp/topics/minimal_publisher/member_function.cpp`

- `rclcpp` is an abbreviation of "ROS Client Library C++", a.k.a. the ROS C++ API
- The specific include directive used for `rclcpp` is `#include "rclcpp/rclcpp.hpp"`
- This node's source code uses the approach of having a separate class, and using its member functions as Callback methods

```cpp
class MinimalPublisher : public rclcpp::Node
{
public:
  MinimalPublisher()
  : Node("minimal_publisher"), count_(0)
  {
    publisher_ = this->create_publisher<std_msgs::msg::String>("topic", 10);
    timer_ = this->create_wall_timer(
      500ms, std::bind(&MinimalPublisher::timer_callback, this));
  }
```

- The class `MinimalPublisher` inherits from the `rclcpp::Node` base class
- To specify the Node's name(`minimal_publisher`), the Constructor uses its parent class's constructor in `Node("minimal_publisher")`
- Creation of a publisher object:

```cpp
    publisher_ = this->create_publisher<std_msgs::msg::String>("topic", 10);
```

- Setting up a Member function as a Callback method to be called repeatedly at a specific interval

```cpp
    timer_ = this->create_wall_timer(
      500ms, std::bind(&MinimalPublisher::timer_callback, this));
```

- The callback function is the one that actually publishes those messages (at intervals of length specified in the last code snippet) along with logging

```cpp
private:
  void timer_callback()
  {
    auto message = std_msgs::msg::String();
    message.data = "Hello, world! " + std::to_string(count_++);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", message.data.c_str());
    publisher_->publish(message);
  }
```

- Creating the message object and assigning proper message data:

```cpp
    auto message = std_msgs::msg::String();
    message.data = "Hello, world! " + std::to_string(count_++);
```

- Logging:

```cpp
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", message.data.c_str());
```

- Publishing the message(using the member publisher object `publisher_`):

```cpp
    publisher_->publish(message);
```

- The `int main` function contains the main entry point code for running a node using the Class

```cpp
int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<MinimalPublisher>());
  rclcpp::shutdown();
  return 0;
}
```

- Initialization with the ROS Client Library a.k.a. the ROS C++ API is done by the first line: `rclcpp::init(argc, argv);`.
- The line `rclcpp::spin(std::make_shared<MinimalPublisher>());` creates a new object of the aforementioned class with an associated Smart Shared Pointer, which is passed to the ROS C++ API for running it as a Node until a `SIGTERM` is given.
- After Spinning the Node terminates, the `rclcpp::shutdown()` function call cleans everything up and Shuts down.

## Exercise: Modify and Build this Node

### Modifying

The following modifications should be made for the desired results:

1. Make it run at 10 Hz instead of 2 Hz: modification in Constructor of `MinimalPublisher`: `timer_ = this->create_wall_timer(100ms, std::bind(&MinimalPublisher::timer_callback, this));`
2. Change the topic name from "topic" to "greetings": modification in Constructor of `MinimalPublisher`: `publisher_ = this->create_publisher<std_msgs::msg::String>("greetings", 10);`
3. Change the message to "Hello Open Road": modification in `timer_callback()` method: `message.data = "Hello, Open Road! " + std::to_string(count_++);`
4. Change the node name to `revenge_of_minimal_publisher`: modification in member object constructor calls of Constructor of `MinimalPublisher`: `: Node("revenge_of_minimal_publisher"), count_(0)`

### Building and Executing

```bash
# In build terminal session
cd ../../../../../
colcon build --symlink-install
# In execution terminal session
ros2 run examples_rclcpp_minimal_publisher publisher_member_function
```

![[publisher_member_function ros2 run.png]]

![[publisher_member_function ros2 topic.png]]

## An Example node for Subscribing

With the same "examples" packages, also comes an example subscriber node, whose source code is located at: `src/examples/rclcpp/topics/minimal_subscriber/member_function.cpp`

- The overall pattern and approach here is analogous to what was done in `MinimalPublisher`
- Creating a Subscriber object:

```cpp
  MinimalSubscriber()
  : Node("minimal_subscriber")
  {
    subscription_ = this->create_subscription<std_msgs::msg::String>(
      "topic", 10, std::bind(&MinimalSubscriber::topic_callback, this, _1));
  }
```

- Instead of a publisher callback method, here we pass a Member function(that logs every message published to a specific topic) to a subscriber object (whose constructor decides which topic to subscribe to for logging, it also decides the queue size for this object).

```cpp
  void topic_callback(const std_msgs::msg::String::SharedPtr msg) const
  {
    RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());
  }
```

- The `main` function follows the exact same boilerplate here.

### Changing the subscriber's topic

To change the subscriber's topic, modify the following line in the constructor of `MinimalSubscriber`:

```cpp
    subscription_ = this->create_subscription<std_msgs::msg::String>(
      "greetings", 10, std::bind(&MinimalSubscriber::topic_callback, this, _1));
```

To build, execute:

```bash
cd ../../../../../
colcon build --symlink-install
```

Screenshots of running:

![[publisher_member_function ros2 run for subscriber.png]]

![[subscriber_member_function ros2 run.png]]
## Services

### Running and Calling a pre-built service node

![[minimal_service ros2 run.png]]

![[minimal_service ros2 service call.png]]

Screenshots: running and calling a service(that adds two integers)

```bash
ros2 run examples_rclcpp_minimal_service service_main
# In a different terminal
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 1, b: 2}"
```

### Source code for providing a service(from a Node)

`src/examples/rclcpp/services/minimal_service/main.cpp`:

```cpp
// Copyright 2016 Open Source Robotics Foundation, Inc.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

#include <inttypes.h>
#include <memory>
#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using AddTwoInts = example_interfaces::srv::AddTwoInts;
rclcpp::Node::SharedPtr g_node = nullptr;

void handle_service(
  const std::shared_ptr<rmw_request_id_t> request_header,
  const std::shared_ptr<AddTwoInts::Request> request,
  const std::shared_ptr<AddTwoInts::Response> response)
{
  (void)request_header;
  RCLCPP_INFO(
    g_node->get_logger(),
    "request: %" PRId64 " + %" PRId64, request->a, request->b);
  response->sum = request->a + request->b;
}

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  g_node = rclcpp::Node::make_shared("minimal_service");
  auto server = g_node->create_service<AddTwoInts>("add_two_ints", handle_service);
  rclcpp::spin(g_node);
  rclcpp::shutdown();
  g_node = nullptr;
  return 0;
}
```

## Source code for calling a Service

`src/examples/rclcpp/services/minimal_client/main.cpp`:

```cpp
// Copyright 2016 Open Source Robotics Foundation, Inc.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

#include <chrono>
#include <cinttypes>
#include <memory>
#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

using AddTwoInts = example_interfaces::srv::AddTwoInts;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = rclcpp::Node::make_shared("minimal_client");
  auto client = node->create_client<AddTwoInts>("add_two_ints");
  while (!client->wait_for_service(std::chrono::seconds(1))) {
    if (!rclcpp::ok()) {
      RCLCPP_ERROR(node->get_logger(), "client interrupted while waiting for service to appear.");
      return 1;
    }
    RCLCPP_INFO(node->get_logger(), "waiting for service to appear...");
  }
  auto request = std::make_shared<AddTwoInts::Request>();
  request->a = 41;
  request->b = 1;
  auto result_future = client->async_send_request(request);
  if (rclcpp::spin_until_future_complete(node, result_future) !=
    rclcpp::FutureReturnCode::SUCCESS)
  {
    RCLCPP_ERROR(node->get_logger(), "service call failed :(");
    return 1;
  }
  auto result = result_future.get();
  RCLCPP_INFO(
    node->get_logger(), "result of %" PRId64 " + %" PRId64 " = %" PRId64,
    request->a, request->b, result->sum);
  rclcpp::shutdown();
  return 0;
}
```

### Running the Service Client Node

(and thus making it call the service):

```bash
ros2 run examples_rclcpp_minimal_service service_main
# In a different terminal session:
ros2 run examples_rclcpp_minimal_client client_main
```

![[Pasted image 20240811000849.png]]

![[Pasted image 20240811000940.png]]

## ROS 2 Actions

ROS 2 Actions are very similar to ROS 1 Actions
The main difference between a Service and an Action is that Services are like Synchronous RPCs and Actions are like Asynchronous RPCs i.e. Services are supposed to return the result almost immediately(e.g. Turning on a light, Opening/Closing a door etc), whereas Actions are supposed to take some amount of time to return the result(e.g. Navigating to a specific location), the Action server may decline to an action request(e.g. that location is unreachable according to path finding algorithms or maybe battery too low) and the client can ask for feedback while the action request is being processed or possibly cancel the action midway.

### Timeline/Stages of an Action being processed

- First step: find the action and action server(action server may be down)
- The action being accepted
- The client might change its mind and cancel the action
- Exchange of action "feedback"
- Return of the result (can either correspond to success or failure) to the client

**The action definition files are similar in structure to the ones in ROS 1**

## Running an Action and Calling it

```bash
ros2 run 
```