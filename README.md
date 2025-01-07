# TR-CDA
Real Time Communication and Data Aquisition (RT-CDA) Methodology

## Table of Contents
1. [Communication and Control System Setup](#communication-and-control-system-setup)  
   - [Overview of Full Setup](#overview-of-full-setup)  
   - [MAVLink](#mavlink) 
   - [MQTT](#mqtt)  
   - [Camera Streaming](#camera-streaming)  
2. [Scripts Documentation](#scripts-documentation)  
   - [MAVLink Script Arguments](#mavlink-script-arguments)  
   - [Summary of MAVLink Scripts](#summary-of-mavlink-scripts)  

## Communication and Control System Setup

### Overview of Full Setup
During initial setup, an SSH connection is used for accessing the command prompt of the Raspberry Pi from the GCBS. Then, MAVLink and MQTT are used for communication between the SysML model and the Raspberry Pi. For MAVLink, the Raspberry is connected to the Pixhawk via UART. MAVProxy, running on the Raspberry Pi, forwards MAVLink messages from the Pixhawk to the ground control base station. The GCBS can send MAVLink messages back to the Pixhawk through the Raspberry Pi. Lastly, the Raspberry Pi receives camera input and can host a webserver that allows the camera to be displayed in Mission Planner using a URL.
![B1 image](https://github.com/clarc10/TR-CDA/blob/a05636617ff4b3ed106697e5c72e0068f18a77ee/B1.png?raw=true)

### MAVLink 
The main method of communication used between the vehicle and the Ground Control Base Station (GCBS) is the MAVLink protocol. MAVLink is a lightweight protocol for communication with drones. In our setup, the MAVLink protcol can be used for direction communication between the GCBS and Pixhawk over radio. It can be used over WiFi with the Raspberry Pi. We will cover setup instructions for the latter, where the Raspberry Pi is physically wired to the Pixhawk and the GCBS is on the same network as the Raspberry Pi.

#### Raspberry Pi UART Connection to Pixhawk
The following link contains instructions on connecting the Raspberry Pi to the telem2 port on the Pixhawk: https://docs.px4.io/main/en/companion_computer/pixhawk_rpi.html. Ignoring the setup of ROS, these instructions will enable the Raspberry Pi to receive MAVLink messages over UART from the Pixhawk.

#### MAVProxy Forwarding to GCBS
MAVProxy is a command line base station. It uses MAVLink to communicate between the Raspberry Pi and Pixhawk. It allows us to forward the MAVLink messages received by the Raspberry Pi to the GCBS computer. MAVProxy was mainly used as a middleman between the Pixhawk and GCBS to allow MAVLink messages to be sent over the network. To install MAVProxy on the Raspberry Pi, run the following command: `python3 –m pip install MAVProxy`

To run MAVProxy on the Raspberry Pi, first determine the serial port corresponding the UART connection with the Pixhawk. This is typically either /dev/ttyS0 or /dev/ttyAMA0. The baud rate of the connection is also required. To find the baud rate, check the SERIAL2_BAUD parameter on the Pixhawk (assuming the telemetry 2 port is used). Confirm that the SERIAL2_PROTOCOL parameter is set to 2, for MAVLink 2. Once the port and the baud rate are found, start MAVProxy with the following command: 
`sudo mavproxy.py --master=/dev/ttyS0 --baudrate 921600`

Note that /dev/ttyS0 and 921600 will need to be replaced with the correct port and baud rate. To run MAVProxy with MAVLink forwarding to the GCBS, find the IP of the GCBS. Then, on the Raspberry Pi, rerun MAVLink with the following command, replacing <GCBS_IP> with the IP of the GCBS:
`sudo mavproxy.py --master=/dev/ttyS0 --baudrate 921600 –-out <GCBS_IP>:14550 <GCBS_IP>:14551`

The above comamnd sends the MAVLink messages over UDP to ports 14550 and 14551 of the GCBS. Port 14550 is typically used for MAVLink connections and is used by autopilot software in the GCBS such as Mission Planner by default. for the Communication and Control System Model, port 14551 will be used. This allows running both Mission Planner and the SysML model at the same time. 

#### MAVLink in Python
To utilize the MAVLink protocol in Python scripts on the GCBS, we use the pymavlink library. It can be installed using pip: python3 –m pip install pymavlink. 

### MQTT 
One method of communication between the Raspberry Pi (on the rover) and the Ground Control Base Station (GCBS) is the MQTT (Message Queuing Telemetry Transport) protocol. MQTT is a publish-subscribe protcol that requires a broker. In our setup, we run the Eclipse Mosquitto MQTT Broker on the Raspberry Pi. Both the Raspberry Pi and the GCBS then run MQTT clients which can publish and subscribe to topics on the broker.

#### Steps to install and configure Mosquitto on the Raspberry Pi
- Update System
    - `sudo apt update && sudo apt upgrade`
-	Install Mosquitto
    - `sudo apt install -y mosquitto mosquitto-clients`
- To Enable Auto-start on Boot
    - `sudo systemctl enable mosquitto.service`
-	To allow remote access without authentication, add the following lines to `/etc/mosquitto/mosquitto.conf`
    -	`listener 1883`
    - `allow_anonymous true`

#### Steps to setup MQTT client with Python
To interact with the MQTT broker through Python scripts, we use the paho-mqtt Python library. This must be installed on both the GCBS and Raspberry Pi. It can be installed using pip:
`python3 –m pip install paho-mqtt`

### Camera Streaming
The Raspberry Pi can stream live camera feed to the GCBS. For this, we use `mjpg-streamer`. It can be installed following the instructions from https://github.com/jacksonliam/mjpg-streamer. 

#### Using mjpg-streamer
Note that these instructions were written in 2023, so they may be outdated. 
Once you have installed mjpg-streamer, `cd` into the  mjpg-streamer-experimental directory. 
To start the stream, run: 

`./mjpg_streamer -o "output_http.so -w www -p 8080" -i "input_uvc.so -rot 180 -r 1280x720 -fps 15"`, 
where -p is for port number, -rot is for rotation, -r is for resolution, -fps is for frames per second.

To access the live stream from the browser of any computer on the network, open the following link: http://<pi_ip_address>:8080?action=stream. Substitute <pi_ip_address> for the actual IP address of the Raspberry Pi. On the Raspberry Pi itself use http://localhost:8080?action=stream.


#### Camera Configuration in Mission Planner
Once the section above is completed, the stream can be setup in the Mission Planner HUD. Right click on the HUD, click video, and then click Set MJPEG source. A popup will ask for the URL. Enter the URL provided above, and the stream should start in Mission Planner.

## Scripts Documentation

### MAVLink Script Arguments
All scripts accept the following arguments:  
- **`-s/--source`**: MAVLink connection source string.
     - For UDP connection on port 14551, `udpin:0.0.0.0:14551`
     - For TCP connection on port 14551, `tcpin:0.0.0.0:14551`
     - For USB radio on Windows, this will be a com port, such as `com3`
- **`-b/--baudrate`**: Baud rate for the MAVLink connection.
     - Only required when using MAVLink over a USB radio

The additional arguments for each script are covered in the following section.
Furthermore, any script can be provided `-h` to view the full documentation. 

### Summary of MAVLink Scripts

| **Script Name**                | **Description**                                                                                                                                                                                                                 | **Key Arguments**                                  |
|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| **`arm.py`**                   | Arms or disarms the eUAV by sending `MAV_CMD_COMPONENT_ARM_DISARM`. Exits with code `0` if `MAV_RESULT_ACCEPTED` is received, else exits with code `1`. Prints command acknowledgment in JSON format with a timestamp.       | `-d/--disarm` (disarm instead of arm)            |
| **`mavlink_recv_till_mission_complete.py`** | Receives MAVLink messages until the mission is complete. Waits for `MISSION_ITEM_REACHED` message. Each message is printed in JSON format.                                                                            | `msg_types` (specific message types), `-a` (add timestamp) |
| **`mavlink_recv.py`**          | Receives specified MAVLink messages, printing each as a JSON line. Can limit by number of messages or time duration.                                                                                                         | `msg_types` (specific message types), `-n` (max messages), `-t` (max time), `-a` (add timestamp) |
| **`set_mode.py`**              | Sets the mode of the eUAV by sending `MAV_CMD_DO_SET_MODE`. Accepts mode names (e.g., `AUTO`) or numbers. Exits with code `0` if successful, else `1`. Prints acknowledgment with a timestamp in JSON format.               | `mode` (name or number)                          |
| **`set_pos_target_ned.py`**    | Sends `set_position_target_local_ned_message` for relative movement in the North-East direction using `LOCAL_OFFSET_NED` frame.                                                                                             | `N` (meters North), `E` (meters East)            |
| **`upload_waypoints.py`**      | Uploads waypoints from a file to the eUAV over MAVLink. Exits with code `0` if mission is accepted (`MAV_MISSION_ACCEPTED`), else `1`.                                                                                       | `WP_FILE` (path to waypoint file)                |
| **`verify_mavlink.py`**        | Verifies MAVLink source and baudrate by waiting for a heartbeat. Exits with code `0` if a heartbeat is received within 3 seconds, else exits with code `1`.                                                                 | None                                             |




