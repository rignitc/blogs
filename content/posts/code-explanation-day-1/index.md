---
title: Code Explanation Day 01
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
  - opencv
  - code
  - python
  - basics
  - handout
  - rignitc
# nolastmod: true
draft: false
---

<!-- Summary -->
Day 01: Gesture Controlled Robot with OpenCV and ESP

<!--more-->

# **Gesture-Controlled Robot**

This Python script captures hand gestures using MediaPipe, classifies them using a custom-trained model, and sends the gesture index to an ESP device over a TCP socket connection.

The robot is gesture-controlled, meaning it performs specific actions based on the hand gestures detected by a camera. It uses an ESP-based microcontroller to receive commands over Wi-Fi. The ESP processes these commands (gesture indices) and controls the robot's movements or actions accordingly.

---

## **How It Works**

1. **Hand Gesture Detection:**
   - Uses MediaPipe Hands to detect hand landmarks (e.g., fingertips, knuckles).
   - These landmarks are processed by a pre-trained model (`KeyPointClassifier`) to classify the current gesture.

2. **Data Transmission:**
   - The recognized gesture index (a number representing the gesture) is sent to the ESP microcontroller over a TCP socket connection.

3. **ESP Response:**
   - The ESP processes the received index and controls the robot based on the predefined gesture mappings.

---

## **Libraries and Tools**

```python
import cv2
import mediapipe as mp 
import socket 
from model import KeyPointClassifier 
import landmark_utils as u
```

### **Key Libraries:**
- `cv2`: OpenCV library for video capture and image processing.
- `mediapipe`: Library for detecting and tracking hand landmarks.
- `socket`: Provides support for network communication to send data to the ESP device.
- `KeyPointClassifier`: A custom model that classifies hand gestures.
- `landmark_utils`: A custom utility module to process hand landmarks into the required format for classification.

---

## **Gesture Mappings**

```python
gestures = { 
    0: "Open Hand", 
    1: "Thumb up", 
    2: "OK", 
    3: "Peace", 
    4: "No Hand Detected"
}
```

- Maps numerical gesture indices (output of `KeyPointClassifier`) to readable gesture names.
- Defaults to `4` ("No Hand Detected") if no hand is detected.

---

## **Connection Setup**

```python
ESP_IP = "192.168.137.89"  # Replace with your ESP's IP address
ESP_PORT = 80  # Replace with your ESP's listening port

client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_socket.connect((ESP_IP, ESP_PORT))
print(f"Connected to ESP at {ESP_IP}:{ESP_PORT}")
```

- Sets up a TCP socket for communication with the ESP microcontroller.
- Replace `ESP_IP` and `ESP_PORT` with your ESP's IP address and port.

---

## **Hand Detection and Gesture Classification**

### **Initialize MediaPipe Hands**

```python
mp_drawing = mp.solutions.drawing_utils
mp_drawing_styles = mp.solutions.drawing_styles
mp_hands = mp.solutions.hands
cap = cv2.VideoCapture(0)

with mp_hands.Hands(
    model_complexity=0,
    min_detection_confidence=0.5,
    min_tracking_confidence=0.5
) as hands:
```

- `mp_hands.Hands`: Detects hands in the video feed.
- Parameters:
  - `model_complexity`: Determines model complexity (lower = faster).
  - `min_detection_confidence`: Threshold for detecting a hand.
  - `min_tracking_confidence`: Threshold for tracking detected hands.

---

### **Process Video Feed**

```python
while cap.isOpened():
    success, frame = cap.read()
    if not success:
        print("Ignoring empty camera frame.")
        continue

    image.flags.writeable = False
    rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    results = hands.process(rgb_frame)

    frame.flags.writeable = True
    bgr_frame = cv2.cvtColor(rgb_frame, cv2.COLOR_RGB2BGR)
    gesture_index = 4  # Default to "No Hand Detected"
```

- Reads frames from the webcam.
- Converts each frame to RGB format and processes it with MediaPipe.
- Initializes `gesture_index` as `4` (No Hand Detected).

---

### **Detect and Classify Gestures**

```python
    if results.multi_hand_landmarks:
        for hand_landmarks in results.multi_hand_landmarks:
            landmark_list = u.calc_landmark_list(bgr_frame, hand_landmarks)
            keypoints = u.pre_process_landmark(landmark_list)
            gesture_index = kpclf(keypoints)

            mp_drawing.draw_landmarks(
                bgr_frame,
                hand_landmarks,
                mp_hands.HAND_CONNECTIONS,
                mp_drawing_styles.get_default_hand_landmarks_style(),
                mp_drawing_styles.get_default_hand_connections_style()
            )
```

- **Landmark Processing:**
  - `calc_landmark_list`: Extracts landmark coordinates.
  - `pre_process_landmark`: Prepares data for the classifier.
  - `kpclf`: Classifies the landmarks into gestures.

- Draws landmarks and connections on the video feed.

---

### **Send Gesture to ESP**

```python
    print(f"Detected Gesture Index: {gesture_index}")
    gesture_data = (str(gesture_index) + "\n").encode()
    client_socket.sendall(gesture_data)
```

- Prints the detected gesture index.
- Sends the index to the ESP device via the TCP socket.

---

### **Display Video Feed**

```python
    final = cv2.flip(bgr_frame, 1)
    cv2.putText(final, gestures[gesture_index],
                (10, 30), cv2.FONT_HERSHEY_DUPLEX, 1, 255)
    cv2.imshow('MediaPipe Hands', final)
    if cv2.waitKey(5) & 0xFF == 27:
        break
```

- Flips the frame horizontally for a selfie view.
- Displays the gesture name on the frame.
- Terminates the loop when the **Esc** key is pressed.

---

## **Cleanup**

```python
cap.release()
client_socket.close()
cv2.destroyAllWindows()
print("Connection closed.")
```

- Releases the camera and closes the socket connection.
- Destroys all OpenCV windows.

---

## **Instructions for Running the Code**

1. **Turn On Hotspot:**
   - Enable your device's hotspot.
   - Set the SSID and PASSWORD.

2. **Set Hotspot Bandwidth:**
   - Use the 2.4GHz band for NodeMCU communication.

3. **Update the Arduino Code:**
   - Open the Arduino code in the Gesture folder.
   - Replace placeholder SSID and PASSWORD with your credentials.

4. **Navigate to the Correct Folder:**
   - Ensure you're inside the `Gesture_control` folder for accessing the `KeyPointClassifier`.

5. **Run the Python Code:**
   - Execute the Python script for gesture detection.

6. **Perform Hand Gestures:**
   - Supported gestures:
     - Thumbs up
     - Open hand
     - Peace sign
     - OK sign

7. **Control the Robot:**
   - The robot responds to your gestures as per the predefined mappings.

--- 
```
