# 如何通过TCP端口30000获取实时数据

本文以端口30000为例



## 1.介绍

我们提供 TCP 端口 30000 以实时获取机械臂数据，频率为 250HZ（使用 FT 传感器时为 200HZ） 

下面简要描述如何使用 30000 端口进行数据报告。

[TCP 端口数据说明](data-description-of-tcp-port.md)
## 2.Python示例

```python
'''
Base on TCP port 30000, 250HZ.
Firmware version should be 2.1.101 or later.
Here is an example to get actual joint currents by TCP 30000 port, unit: Ampere.
'''

import socket
import struct

def bytes_to_fp32(bytes_data, is_big_endian=False):
    return struct.unpack('>f' if is_big_endian else '<f', bytes_data)[0]

def bytes_to_fp32_list(bytes_data, n=0, is_big_endian=False):
    ret = []
    count = n if n > 0 else len(bytes_data) // 4
    for i in range(count):
        ret.append(bytes_to_fp32(bytes_data[i * 4: i * 4 + 4], is_big_endian))
    return ret


def bytes_to_u32(data):
    data_u32 = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3]
    return data_u32


robot_ip = '192.168.1.77'
robot_port = 30000

# create socket connection
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.setblocking(True)
sock.settimeout(1)
sock.connect((robot_ip, robot_port))

buffer = sock.recv(4)
print(buffer)
while len(buffer) < 4:
    buffer += sock.recv(4 - len(buffer))
size = bytes_to_u32(buffer[:4])

cnt=0
while cnt<1000:
    buffer += sock.recv(size - len(buffer))
    if len(buffer) < size:
        continue
    cnt+=1
    data = buffer[:size]
    buffer = buffer[size:]
    actual_joint_currents=bytes_to_fp32_list(data[200:228])
    print("counter={},actual_joint_currents={}".format(cnt,actual_joint_currents))
```

## 3.C++示例
```cpp
#include <iostream>
#include <vector>
#include <array>
#include <cstring>
#include <arpa/inet.h>
#include <unistd.h>

float bytesToFloat(const std::array<uint8_t, 4>& bytes, bool isBigEndian = false) {
    float value;
    uint32_t intValue = isBigEndian ?
        (bytes[0] << 24 | bytes[1] << 16 | bytes[2] << 8 | bytes[3]) :
        (bytes[3] << 24 | bytes[2] << 16 | bytes[1] << 8 | bytes[0]);
    std::memcpy(&value, &intValue, sizeof(float));
    return value;
}

uint32_t bytesToUInt32(const std::array<uint8_t, 4>& bytes) {
    return (bytes[0] << 24) | (bytes[1] << 16) | (bytes[2] << 8) | bytes[3];
}

std::vector<float> bytesToFloatList(const std::vector<uint8_t>& data, size_t start, size_t count, bool isBigEndian = false) {
    std::vector<float> values;
    for (size_t i = 0; i < count; ++i) {
        std::array<uint8_t, 4> floatBytes = {data[start + i * 4], data[start + i * 4 + 1], data[start + i * 4 + 2], data[start + i * 4 + 3]};
        values.push_back(bytesToFloat(floatBytes, isBigEndian));
    }
    return values;
}

int main() {
    const char* robotIp = "192.168.1.77";
    const int robotPort = 30000;
    
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        std::cerr << "Failed to create socket" << std::endl;
        return -1;
    }
    
    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(robotPort);
    inet_pton(AF_INET, robotIp, &serverAddr.sin_addr);
    
    if (connect(sock, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) < 0) {
        std::cerr << "Connection failed" << std::endl;
        close(sock);
        return -1;
    }
    
    std::vector<uint8_t> buffer(4);
    if (recv(sock, buffer.data(), 4, 0) < 4) {
        std::cerr << "Failed to receive size data" << std::endl;
        close(sock);
        return -1;
    }
    
    uint32_t size = bytesToUInt32({buffer[0], buffer[1], buffer[2], buffer[3]});
    buffer.resize(size);
    
    int cnt = 0;
    while (cnt < 1000) {
        ssize_t received = recv(sock, buffer.data(), size, 0);
        if (received < size) continue;
        
        cnt++;
        std::vector<float> actualJointCurrents = bytesToFloatList(buffer, 200, 7);
        std::cout << "Counter=" << cnt << ", actual_joint_currents=[";
        for (size_t i = 0; i < actualJointCurrents.size(); ++i) {
            std::cout << actualJointCurrents[i] << (i < actualJointCurrents.size() - 1 ? ", " : "");
        }
        std::cout << "]" << std::endl;
    }
    
    close(sock);
    return 0;
}

```

## 4.解析数据

get_joint_angle_30000.py:  通过端口30000获取实际关节角度并保存为csv文件

read_csv.py:  读取数据并通过“matplolib”画图



* get_joint_angle_30000.py

```python
import time
import math
import socket
import struct
import csv
import io

def bytes_to_fp32(bytes_data, is_big_endian=False):
    return struct.unpack('>f' if is_big_endian else '<f', bytes_data)[0]

def bytes_to_fp32_list(bytes_data, n=0, is_big_endian=False):
    ret = []
    count = n if n > 0 else len(bytes_data) // 4
    for i in range(count):
        ret.append(bytes_to_fp32(bytes_data[i * 4: i * 4 + 4], is_big_endian))
    return ret

def bytes_to_u8(data):
    data_u8_0= int.from_bytes(data[:],byteorder='little')
    return data_u8_0

def bytes_to_u16(data):
    data_u16 = data[0] << 8 | data[1]
    return data_u16


def bytes_to_u32(data):
    data_u32 = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3]
    return data_u32


def bytes_to_u64(data):
    if len(data) != 8:
        raise ValueError("Input data must be exactly 8 bytes long")
    u64 = struct.unpack(">Q", data)[0]
    return u64

robot_ip = '192.168.1.80'
robot_port = 30000

# socket connection
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.setblocking(True)
sock.settimeout(1)
sock.connect((robot_ip, robot_port))

buffer = sock.recv(4)
while len(buffer) < 4:
    buffer += sock.recv(4 - len(buffer))
size = bytes_to_u32(buffer[:4])

cnt=0
t0=time.time()

output = io.StringIO()
csv_writer = csv.writer(output)

while cnt < 15000:
    buffer += sock.recv(size - len(buffer))
    if len(buffer) < size:
        continue
    cnt += 1
    data = buffer[:size]
    buffer = buffer[size:]
    # actual joint angles
    actual_joints_in_rad = bytes_to_fp32_list(data[116:144])
    print(actual_joints_in_rad)

    actual_joints_in_deg = []
    for i in actual_joints_in_rad:
        j = math.degrees(i)
        actual_joints_in_deg.append(j)
    print('{},{}'.format(cnt, actual_joints_in_deg))
    print(cnt)
    csv_writer.writerow(actual_joints_in_deg)

with open('joint_pos.csv', 'a+', newline='') as f:
    f.write(output.getvalue())

t1 = time.time()
dur = t1 - t0
print(dur)

```



* read_csv.py

```python
import csv
import pandas as pd
import matplotlib.pyplot as plt

data = pd.read_csv('joint_pos.csv')
print('test=', type(data))

# print('test================', data.iloc[:, 0])
plt.plot(data.iloc[:, 0], label='joint1', color='red')
plt.plot(data.iloc[:, 1], label='joint2', color='green')
plt.plot(data.iloc[:, 2], label='joint3', color='blue')
plt.plot(data.iloc[:, 3], label='joint4', color='yellow')
plt.plot(data.iloc[:, 4], label='joint5', color='pink')
plt.plot(data.iloc[:, 5], label='joint6',color='black')

plt.xlabel('time')
plt.ylabel('Joint Angle')
plt.legend()
plt.show()

```



例如：

[joint_pose.csv](https://share.weiyun.com/0tuCPdU7)



结果如图：
![image](../assets/image(59).png)
