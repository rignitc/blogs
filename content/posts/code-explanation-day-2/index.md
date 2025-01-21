---
title: Code Explanation Day 02
date: 2025-01-10T18:08:19+05:30
lastmod: 2025-01-10T18:08:19+05:30
author: ORIGO
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: cover.png
images:
  - cover.png
categories:
  - origo
tags:
  - yolo
  - code
  - python
  - basics
  - handout
  - rignitc
# nolastmod: true
draft: false
---

<!-- Summary -->
Day 02: Building a Person-Following Robot with YOLO and PID Control

<!--more-->

# CODE EXPLANATION - DAY 2

We are going to integrate YOLO into our robot so that it can detect a person and follow them around. 

We are going to accomplish this in this way:

- Place our phone on the robot and use its camera to get video feed
- Identify a person in the video stream using YOLO and get their position
- Rotate the robot so that the center of the robot (i.e center of the video feed) coincides with the position of the person detected
- Once the center coincides, move forward

So we need to:

1. Get video stream from our phone
2. Get position of person using YOLO
3. Use a PID Controller to get the required PWM signals needed to rotate the wheels so as to make the center of the video feed and the position of the person coincident.

## Running the code
1. Start Hotspot on your laptop and connect your phone to it. Make sure the network band is set to 2.4 Ghz.
2. Add the WiFi credential to the NodeMCU .ino file and upload it.
3. Start the NodeMCU and wait for it to connect to your hotspot.
4. Note down the IP shown in the serial monitor and edit the `ESP_IP` in the `person_detection.py` file to this IP.
5. Start video server in IPWebcam.
6. Note down the IPv4 address shown and change the `VIDSTREAM` in `person_detection.py` file to this address + '/video'
7. Start the .py file with `py person_detection.py`

## Code Explanation
 
```py
import cv2
import json
import socket
import signal
import sys
from simple_pid import PID
import torch
```

- cv2: OpenCV - For capturing the video stream and displaying it.
- json: For converting dictionaries into JSON (will be explained later)
- socket: For establishing TCP communication
- signal, sys: In our case, only used to ensure clean exit
- simple_pid: Easy to use PID library
- torch: A ML Library to use the YOLO model    

```py
model = torch.hub.load('ultralytics/yolov5', 'yolov5s')  
```

YOLO is an deep learning-based object detection model. PyTorch is a machine learning library used to build, train and deploy deep learning models. So we use PyTorch to actually use the YOLO model.

```py
ESP_IP = "192.168.137.57"  
ESP_PORT = 80         

client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_socket.connect((ESP_IP, ESP_PORT))   
print(f"Connected to ESP at {ESP_IP}:{ESP_PORT}")   
```
This is used to establish TCP communication between the NodeMCU and our laptop. 
`ESP_IP` should be set according to the IP Of the NodeMCU connection.


```py
def send_over_socket(param_dict): 
    json_data = json.dumps(param_dict) + '\n'
    encoded_data = json_data.encode()
    client_socket.sendall(encoded_data)
    print(f"Sent: {json_data.strip()}")
```
This is a helper function to send data over TCP. We cannot send data directly from dictionary format through TCP. Only bytes can be sent.

> DICTIONARY > JSON > BYTE > TCP

#### JSON?

JSON is Javascript Object Notation. It's a text-based data format for representing structured data, widely used for data exchange between systems. It is a universal format that can be understood by any language. JSON is stored as a **string**. 

We can only send bytes over TCP. Dictionaries cannot be directly encoded to bytes. But strings can be. Since JSON is stored as a string we can encode that into bytes and send it over using TCP. 

`json.dumps()` is used to convert a dictionary into json. We add the new line character `\n` to this so that we can identify end of the message which receive the bytes in the NodeMCU. 

`json_data.encode()` is used to convert the JSON string into bytes

`client_socket.sendall()` is used to actually send the data across the TCP connection.

```py
def signal_handler(signal, frame):
    send_over_socket({"pwmR": 0, "pwmL": 0})
    sys.exit(0)

signal.signal(signal.SIGINT, signal_handler) 
```
This is a function that is run when the program exits (which is represented by `signal.SIGINT` i.e Signal Interrupt). It just tells the robot to stop moving when the server (python file in our laptop) stops since we don't want the robot to move even when we have shutdown our python file.


```py
# Initialize PID controller
pid = PID(0, 0, 0, setpoint=0)

# Slider functions for PID tuning
def slider_kp(val):
    pid.Kp = round(val * 0.1, 3)

def slider_ki(val):
    pid.Ki = round(val * 0.01, 3)

def slider_kd(val):
    pid.Kd = round(val * 0.1, 3)

# Create a window with trackbars for PID tuning
Winname = 'PID Slider'
cv2.namedWindow(Winname)
cv2.createTrackbar('Kp', Winname, 0, 100, slider_kp)
cv2.createTrackbar('Ki', Winname, 0, 100, slider_ki)
cv2.createTrackbar('Kd', Winname, 0, 100, slider_kd)
```

This initializes the PID Controller. Also, it defines the sliders neccessary to adjust the gains. 

```py
# For video capture
VIDSTREAM = "http://192.168.137.154:8080/video"
cap = cv2.VideoCapture(VIDSTREAM)

# If there is video available
if cap.isOpened():
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    pid.setpoint = width // 2 # Set the setpoint to the center of the frame
    pid.output_limits = (-250, 250)
```

This starts the capture of video from our phone. Then we calculate the width and height of the image. 

The PID setpoint is set as the center of the frame. This is because we want to rotate the robot until the position of the person detected coincides with the center of the video frame.

We set the PID output limits as -250, 250 since the output is PWM values and it has a maximum of 255. We are not going for 255 since it is not that good for the safe working of the motors.

```py
while cap.isOpened():
    success , frame = cap.read() # Get the video frame
    if not success:
        continue

    # YOLOv5 inference
    results = model(frame)
```

With this we are reading each frame of the video and getting the results of the YOLO model for each of those frames.

```py
    labels, coords = results.xyxyn[0][:, -1], results.xyxyn[0][:, :-1]
```

What exactly does this `results.xyxyn` return. `xyxyn` denotes the **normalized coordinates** of the bounding boxes of the objects detected by the model. 

What are normalized coordinates? We can define it as coordinates that have been scaled down to be between  0 and 1. 
For example, let's say we are considering a frame of 1000 pixels width and 500 pixels height. Let's consider a point denoted by the pixel coordinates (500,100). This would be denoted in normalized coordinates as (0.5,0.2) since 500 is 0.5\*1000 and 100 is 0.2\*500.

Why do we use `results.xyxyn[0]`? Actually, we can give multiple frames at a time in `model(frame)`. But since in our case, we have given only one frame, we want to access only that first (0 index) frame. 

`results.xyxyn[0]` is of a form:
```
[ [x1, y1, x2, y2, conf, class], # For the first object detected
  [x1, y1, x2, y2, conf, class], # For the second object detected 
  ...
  ]
```

where `x1, y1` represents the normalized coordinates of the top left of the bounding box and `x2, y2` represents the bottom right point. 
`conf` represents the confidence of the YOLO model's prediction.
`class` denotes what sort of object YOLO thinks it is i.e the label of the object. 

So when we access `results.xyxyn[0]` with [:,-1], we getting the labels/classes of all the objects that were detected and with [:,:-1] we are getting everything except the last element (the class) of all the objects.

```py
    for i in range(len(labels)):
        if labels[i] == 0:  
            x1, y1, x2, y2, conf = coords[i] 
            x1, y1, x2, y2 = int(x1 * width), int(y1 * height), int(x2 * width), int(y2 * height)
            cx, cy = (x1 + x2) // 2, (y1 + y2) // 2
            
            cv2.rectangle(frame, (x1, y1), (x2, y2), color, 2)
            cv2.circle(frame, (cx, cy), 5, color, -1)
            cv2.putText(frame, f"Person ({conf})", (x1, y1 - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, color, 2)
```

We are looping through all the objects detected and checking if they are a person (since the label of a person = 0 in the pre-trained YOLOv5 model we are using). 
We are converting the normalized coordinates into pixel coordinates of the bounding box and calculating its center. We are drawing the box, its center and adding text showing its confidence level. 

```py
            if (width//2)-100 < cx < (width//2)+100:
                param_dict["pwmL"] = param_dict["pwmR"] = 255
                
            else:
                pwm = int(pid(cx))
                param_dict["pwmL"] = -pwm
                param_dict["pwmR"] = pwm
```

`(width//2)-100 < cx < (width//2)+100` basically tells the robot that if the center of the person detected is at the center of the image (with a passable error of about 100 pixels to the left and right), the robot should move forward at full speed.

If it is not at the center, we use PID to set the rotation of the robot. Based on the center of the person detected, the PID controller gives an output pwm (based on the gains we had chosen) and we give one motor positive pwm and the other negative pwm to rotate it. 

```py
            threshold = 150
            if pwm > 0:
                pwm += threshold
            else: 
                pwm -= threshold
```

We have to give a minimun threshold to the motors since all non-high end motors will have the issue of not rotating at low PWM values.


```py
else:
        # If nothing is detected
        param_dict["pwmL"] = 0
        param_dict["pwmR"] = 0
```

This is the case when nothing has been detected. The robot should not move in that situation, hence we are giving the pwm as 0. 

## NodeMCU Code 

Most of the code is same as the Day 1 NodeMCU code.

The difference arises in handing the JSON data. We use a library called `ArduinoJSON` to handle JSON.

```cpp
        String jsonString = client.readStringUntil('\n');
        jsonString.trim();

        DynamicJsonDocument doc(200);

        DeserializationError error = deserializeJson(doc, jsonString);
        if (error) {
          Serial.println("parseObject() failed");
        } else {
          int pwmL = doc["pwmL"];
          int pwmR = doc["pwmR"];
          Serial.printf("\n L - %d , R - %d\n", pwmL, pwmR); 
          motorL(pwmL);
          motorR(pwmR);
        }
```

`client.readStringUntil('\n')` is used to read the bytes over TCP until we reach the new line character. And `jsonString.trim()` is used to remove the newline character. 

We define a `DynamicJsonDocument`. This acts like the dictionary we use in python. 

We use the`deserializeJson` function to convert the jsonString into a dictionary. In case there is any error, it is stored in `error`. Otherwise, the parsed JSON document is stored in `doc`. Once we get the dictionary in `doc`, we can access the pwm simply like we did in python itself.

Once we get the pwm values needed, we give it to the left and right motors.

--- 


