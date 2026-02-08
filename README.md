# CAMT: Conductor's Adaptive Movement Translator

**Featured in:** [ASU Engineering News - January 2025](https://news.engineering.asu.edu/2025/01/integrated-engineering-on-display/)

Assistive technology helping visually impaired musicians follow conductor directions through real-time haptic feedback.

**Team:** Matthew Arroyo, Hans Inocentes, Mert Isik, Adrian Rodriguez, Mia Sedano  
**Course:** FSE 100 - Introduction to Engineering (Fall 2024)  
**Institution:** Arizona State University, School of Integrated Engineering

---

## Overview

CAMT translates conducting gestures into haptic feedback signals, enabling visually impaired musicians to "feel" conductor directions in real-time. The system detects motion patterns from a wearable conducting wand and transmits wireless signals to trigger haptic feedback worn by musicians.

### The Problem
Visually impaired musicians struggle to follow traditional conductor cues, limiting their ability to participate in ensemble performances. Existing assistive technologies are expensive, complex, or require specialized training.

### Existing Solutions
Professional solutions like the [Haptic Baton](https://www.humaninstruments.co.uk/haptic-baton) by Human Instruments offer sophisticated conducting-to-haptic translation systems. However, these commercial products are typically expensive, complex, or not widely available. While the Haptic Baton represents the state-of-the-art in professional conducting assistance, the high cost creates a significant barrier to accessibility for the musicians who need it most.

### Our Solution
CAMT addresses the **affordability gap** by demonstrating that haptic feedback can be translated from movement using low-cost hardware. As part of the class project specifications, the total budget remained **under $100** while possessing the following core features:
- Gesture detection using motion sensors (MPU-6050)
- Wireless signal transmission over local network
- Haptic feedback via vibration motors
- 80-90% gesture recognition accuracy for common movements

**Our goal:** Prove that accessible conducting assistance technology is achievable within the budget constraints of music education programs, community ensembles, and individual musicians.

---

## System Architecture
```
[Conductor Wand]                    [Musician Wearable]
┌─────────────────┐                 ┌──────────────────┐
│ Raspberry Pi    │                 │  Laptop Server   │
│ Zero 2W         │──── WiFi ───────▶  (TCP Relay)     │
│                 │   192.168.x.x   │                  │
│ MPU-6050 (I2C)  │                 │  Arduino         │
│ Accelerometer   │                 │  + Vibration     │
└─────────────────┘                 │    Motor         │
                                    └──────────────────┘
```

**Communication Flow:**
1. MPU-6050 reads 6-axis motion data via I2C protocol
2. Raspberry Pi processes accelerometer vectors (x, y, z)
3. Gesture detection algorithm identifies conducting patterns
4. TCP/IP socket transmits signal to laptop server (port 12345)
5. Laptop relays to Arduino via serial (115200 baud)
6. Arduino triggers vibration motor for haptic feedback

---

## Technical Implementation

### Hardware Components
- **Raspberry Pi Zero 2W**: Main processing unit on conductor wand, runs Python gesture detection
- **MPU-6050**: 6-axis accelerometer/gyroscope (I2C address: 0x68)
- **Arduino Uno**: Haptic feedback controller
- **Vibration Motor**: Tactile output device
- **Breadboard + Components**: Circuit prototyping
- **Power Supply**: Portable battery pack

**Total Cost: < $100**

### Software Stack
- **Python 3.x**: Core programming language
- **Libraries**:
  - `mpu6050`: Custom library for MPU-6050 sensor interface
  - `socket`: TCP/IP wireless communication
  - `math`: Vector magnitude calculations
  - `serial` (pySerial): Arduino communication
  - `time`: Timing and debounce logic

### Gesture Detection Algorithm

**Core Logic:**
```python
# Read 3-axis accelerometer data
accelerometer_data = mpu6050.get_accel_data()
xMove = accelerometer_data['x']
yMove = accelerometer_data['y']
zMove = accelerometer_data['z']

# Calculate magnitude of acceleration vector
accelMagnitude = math.sqrt(xMove**2 + yMove**2 + zMove**2)

# Threshold-based gesture detection
if (accelMagnitude >= 25):  # Tuned threshold for conducting gestures
    send_signal_to_laptop()  # Trigger haptic feedback
    time.sleep(0.1)  # Debounce delay

previousAccel = accelMagnitude
time.sleep(0.005)  # 200Hz sampling rate
```

The core logic accounts for multi-directional movement with safeguards to limit recognition overload. This is done by using (1) a threshold of 25 m/s^2 (anything slower is not counted), and (2) a 100ms delay between valid inputs to prevent back-to-back vibrations. This implementation was sufficient to create a presentable prototype that can reliably detect simple-medium complexity gestures, but complex high-speed gestures lose accuracy.

### Network Communication

**Raspberry Pi → Laptop Server (TCP/IP):**
```python
def send_signal_to_laptop():
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client_socket.connect((LAPTOP_IP, 12345))
    
    try:
        client_socket.sendall(str(1).encode("utf-8"))
    finally:
        client_socket.close()
```

**Laptop Server → Arduino (Serial):**
```python
def start_server():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.bind(("0.0.0.0", 12345))
    server_socket.listen(1)
    
    while True:
        client_socket, _ = server_socket.accept()
        data = client_socket.recv(1024).decode("utf-8")
        signal = float(data)
        
        if signal == 1:
            arduino.write(b'1')  # Trigger haptic feedback
        
        client_socket.close()
```

---

## Performance Testing

We conducted rigorous testing with a team member experienced in conducting:

**Test Protocol:**
- Conductor performed gestures with wand
- Blindfolded + deafened team member responded to haptic feedback by tapping a table
- Measured accuracy across gesture complexity and tempo variations

**Results:**

| Gesture Type | Tempo | Accuracy | Notes |
|-------------|-------|----------|-------|
| Simple (up/down, left/right) | Low BPM | **80-90%** | Reliable for basic patterns |
| Complex (arcs, diagonals) | Low BPM | **60-70%** | Requires threshold tuning |
| Complex + High tempo | High BPM | **~50%** | Challenges with rapid movement |

**Key Insights:**
- System excels at detecting distinct, deliberate gestures
- High-tempo complex movements require adaptive threshold algorithms
- Latency remained under 100ms across all test conditions
- False positive rate < 5% with proper debounce implementation

**Latency Breakdown:**
- Sensor reading: ~5ms (200Hz sampling)
- Processing: <1ms (vector calculation)
- Network transmission: 10-20ms (local WiFi)
- Arduino response: <5ms
- **Total: 20-30ms end-to-end**

---

## Code Structure
```
camt-project/
├── mpu6050_test.py           # Main Raspberry Pi script (gesture detection)
├── project1_laptop_server.py # TCP server + Arduino relay
├── accel_experiment.py       # Early prototyping/testing
├── extra.py                  # Experimental differential detection logic
└── README.md
```

**Note:** Complete source code cannot be shared per course policy. Key implementation details and architecture are documented in this README for educational purposes.

---

## My Contributions (Mert Isik - Embedded Software Engineer)

### 1. I2C Protocol Research & Hardware Integration
- Researched MPU-6050 datasheet and I2C communication protocol specifications
- **Learned soldering** specifically for this project to connect MPU-6050 to Raspberry Pi GPIO pins
- Configured I2C address (0x68) and verified sensor initialization
- Debugged hardware connectivity issues using I2C detection tools

### 2. Circuit Design & Prototyping
- Designed breadboard circuit for vibration motor control
- Used **Fritzing** software to create wiring diagrams before physical assembly
- Prevented wiring errors through careful planning and documentation
- Assembled and tested complete hardware system

### 3. Software Development
**Raspberry Pi Script (`mpu6050_test.py`):**
- Real-time accelerometer data acquisition from MPU-6050
- Vector magnitude calculation for gesture detection
- Threshold-based pattern recognition algorithm
- TCP/IP wireless transmission implementation

**Laptop Server (`project1_laptop_server.py`):**
- Socket-based signal reception (TCP server on port 12345)
- Serial communication with Arduino (115200 baud)
- Signal relay logic for haptic feedback triggering

### 4. Development Workflow & File Management
- Initially used **SSH** to upload scripts to Raspberry Pi
- Migrated to **FileZilla** for more efficient file transfer and management
- Managed development workflow across multiple devices (laptop, Raspberry Pi, Arduino)
- Version control and iterative testing on embedded hardware

### 5. Project Budgeting & Component Research
- Researched component costs to meet strict $100 budget constraint
- Sourced affordable alternatives: Raspberry Pi Zero ($15), MPU-6050 ($5), Arduino Uno ($25)
- Balanced cost vs. functionality for optimal prototype within budget
- *(Detailed budget spreadsheet unavailable per course policy)*

---

## Recognition & Presentation

**ASU Engineering Project Showcase (December 2024)**
- Presented at **Changemaker Central**, ASU West Valley Campus
- Reverse career-fair format: industry experts, faculty, and guests visited student stations
- Demonstrated live system functionality and answered technical questions from diverse audience
- Received positive feedback on cost-effectiveness and accessibility focus

**Press Coverage:**

Featured as one of three lead projects in **ASU Engineering News** (January 16, 2025):

> "An array of simple but versatile devices that make it possible for musicians who are visually impaired to follow an orchestra conductor's directions. It includes programmable tools that interact with hardware devices using software programs."

> "A project to help the visually impaired play musical instruments was the work of computer science students Matthew Arroyo, Hans Inocentes and **Mert Isik**, Adrian Rodriguez and microelectronics student Mia Sedano."

[Read full article →](https://news.engineering.asu.edu/2025/01/integrated-engineering-on-display/)

---

## Learning Outcomes

### Technical Skills Gained

**Embedded Systems:**
- Raspberry Pi GPIO programming and Linux command-line tools
- I2C protocol specification and hardware interfacing
- Cross-compilation and remote debugging techniques

**Network Programming:**
- TCP/IP socket communication and client-server architecture
- Protocol design for real-time signal transmission
- Error handling and connection management

**Signal Processing:**
- Accelerometer data filtering and noise reduction
- Threshold-based pattern recognition algorithms
- Real-time data processing optimization (200Hz sampling)

**Hardware Development:**
- Circuit design and breadboard prototyping
- Soldering and physical component integration
- Using Fritzing for circuit documentation

**Cross-Platform Development:**
- Python on embedded Linux (Raspberry Pi)
- Arduino C/C++ for microcontroller programming
- Serial communication protocols (UART at 115200 baud)

### Project Management

**Budget Constraints:**
- Delivered functional prototype under $100 (10x cheaper than commercial solutions)
- Made strategic trade-offs between cost and functionality
- Researched affordable component alternatives

**Team Collaboration:**
- 5-person multidisciplinary team (CS, microelectronics, engineering science)
- Division of responsibilities (hardware, software, testing, presentation)
- Weekly milestone planning and progress tracking

**Time Management:**
- Completed project within 3-month semester timeline
- Balanced with other coursework and commitments
- Iterative development with regular testing checkpoints

**Public Communication:**
- Presented technical work to non-technical audience at showcase
- Answered questions from industry experts and faculty
- Explained accessibility impact and cost-benefit analysis

---

## Acknowledgments

- **Professor Joana Sipe**: Course instructor, project advisor, and advocate for integrated engineering
- **ASU School of Integrated Engineering**: Resources, facilities, and support
- **Changemaker Central**: Venue and platform for public demonstration
- **Team Members**: Matthew Arroyo, Hans Inocentes, Adrian Rodriguez, Mia Sedano
- **ASU Engineering News**: Coverage and recognition of our work
- **Human Instruments**: Inspiration from the professional Haptic Baton system

---

## References & Related Work

- [Haptic Baton by Human Instruments](https://www.humaninstruments.co.uk/haptic-baton) - Professional conducting assistance system
- [ASU Engineering News Feature](https://news.engineering.asu.edu/2025/01/integrated-engineering-on-display/) - January 2025
- MPU-6050 Datasheet - InvenSense 6-axis Motion Tracking Device
- Accessibility in Music Performance - Research on assistive technologies for musicians with disabilities

---

## License

Educational project completed for FSE 100 at Arizona State University.  
Code samples and documentation provided for portfolio purposes.  
Hardware design and software architecture may be adapted for non-commercial, educational use.

---

## Contact

**Mert Isik**  
Computer Science Student, Arizona State University  

mertisik329@gmail.com  
[LinkedIn](https://linkedin.com/in/mert-c-isik) | [GitHub](https://github.com/packageIncoming)

---

*This project demonstrates the intersection of embedded systems, accessibility technology, and music education. By making conducting gestures tangible through haptic feedback at a fraction of commercial costs, we hope to inspire more inclusive and affordable assistive technologies for musicians.*

---

**Built with:** Python • Raspberry Pi • Arduino • MPU-6050 • I2C • TCP/IP • Haptic Feedback  
**Cost:** <$100 • **Accuracy:** 80-90% • **Latency:** <100ms
