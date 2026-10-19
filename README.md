# 😜✌️ FaceGesture AI Twister
### A Multimodal AI Physical Puzzle of Faces and Gestures 🤖🧩

FaceGesture AI Twister is an interactive physical computing game built on a Raspberry Pi. Each round, the player must match a randomly generated combination of a facial expression, a gesture with each hand, and the distance of each hand from a sensor, all within a strict time limit. ⏱️ Think Twister, but for your face and fingers!

The game combines on-device computer vision models (for facial expressions and hand gestures) with ultrasonic distance sensing and spoken feedback from a large language model, turning face, hands, and space into simultaneous inputs for one puzzle.

🎓 **Course:** AI in the Physical World, Duke University (Spring 2026)  
👥 **Team:** Diya Mirji, Ishan Madan, Shuangting Yao  
🛠️ **Shuangting's role:** Overall game design, hardware design & integration

---

## ✨ Key Features

### 👀 Pretrained vision models for faces and hands
- 😊 **Facial expressions:** an EfficientNet-B0 model, pretrained and fine-tuned for facial emotion recognition, then quantized to int8 TensorFlow Lite for on-device inference (see `facial_emotion_efficientnetb0_tflite.ipynb`).
- 🖐️ **Hand gestures:** MediaPipe hand-landmark detection, used to classify each hand's gesture.
- 🍓 Both models run locally on the Raspberry Pi.

### 🗣️ Spoken LLM feedback
After each round, the game's scoring results (which elements the player got right or wrong) are sent to a cloud LLM via LiteLLM. The LLM's short reaction is spoken aloud through Piper text-to-speech, which runs locally. Expect some cheering… or some roasting. 🔊😂

### 📏 Distance as a spatial constraint
Two HC-SR04 ultrasonic sensors measure how far each hand is from its sensor. The readings are mapped to color-coded zones (e.g., Pink for ≤ 20 cm, Red for > 20 cm), so the player has to control hand distance while also holding an expression and two gestures. 🌈

---

## 🎮 How to Play

Each round, the LCD displays and the speaker announces a target combination:

| Input | Options | Detected by |
|---|---|---|
| 🙂 Face | 😄 Happy, 😲 Surprise, 🤢 Disgust | EfficientNet-B0 (TF Lite) |
| 🤚 Left hand gesture | 👍 Thumbs Up, ✌️ Peace, 👎 Thumbs Down | MediaPipe |
| ✋ Right hand gesture | 👍 Thumbs Up, ✌️ Peace, 👎 Thumbs Down | MediaPipe |
| ⬅️ Left color (distance) | 🩷 Pink (close ≤ 20 cm) / ❤️ Red (far > 20 cm) | HC-SR04 |
| ➡️ Right color (distance) | 💙 Blue (close ≤ 20 cm) / 💚 Green (far > 20 cm) | HC-SR04 |

⏳ You have **5 seconds** to get into position, then **hold for 3 seconds**. 📸 The camera takes a snapshot at the end of the hold, and the result is scored. 🏆 Win **2 out of 3** rounds to win the game.

---

## 🔧 Hardware

| Component | Details |
|---|---|
| Board | Raspberry Pi 4 or 5 |
| Camera | Pi Camera Module (picamzero) |
| Distance sensors | 2× HC-SR04 ultrasonic sensors |
| Display | 16×2 I2C LCD (`lcd_i2c`, address `39` / 0x27) |
| Speaker | USB audio device (device index 1, 48000 Hz) |
| Button | Momentary push button — GPIO21 |

### 🔌 Wiring

**HC-SR04 ultrasonic sensors (×2)**
```
Sensor Pin → Pi Pin
VCC        → 5V
GND        → GND
TRIG       → GPIO16 (Pin 36)  ← left sensor
ECHO       → GPIO20 (Pin 38)  ← left sensor  [voltage divider: 1kΩ + 2kΩ]
TRIG       → GPIO23 (Pin 16)  ← right sensor
ECHO       → GPIO24 (Pin 18)  ← right sensor [voltage divider: 1kΩ + 2kΩ]
```
> ⚠️ HC-SR04 ECHO outputs 5 V — use a voltage divider before connecting to the Pi.

**16×2 I2C LCD**
```
LCD Pin → Pi Pin
VCC     → 5V
GND     → GND
SDA     → GPIO2 / SDA (Pin 3)
SCL     → GPIO3 / SCL (Pin 5)
```

**Button**
```
One leg → GPIO21 (Pin 40)
Other   → GND
```
Uses the internal pull-up, so no external resistor is needed.

---

## 📁 Project Structure

```
final_pi_puzzle/
├── robust_game.py   # Main game loop with gesture detection and LLM feedback
├── game.py          # Simple initial version (no gesture detection or LLM)
├── display.py       # LCDDisplay class
├── config.py        # All configuration and game options
├── facial_emotion_efficientnetb0_tflite.ipynb  # Emotion model fine-tuning + TFLite export
├── models/
│   ├── emotionnet_int8.tflite
│   ├── emotion_model_config.json
│   ├── en_US-libritts-high.onnx
│   └── en_US-libritts-high.onnx.json
├── tests/
│   ├── test_button.py    # Verify button wiring
│   ├── test_lcd.py       # Verify LCD display
│   ├── test_sensors.py   # Verify both ultrasonic sensors
│   ├── test_camera.py    # Verify camera + emotion model
│   └── test_speaker.py   # Verify Piper TTS + speaker
└── README.md
```

---

## 🚀 Setup

### 1. Enable I2C on the Pi
```bash
sudo raspi-config   # Interface Options → I2C → Enable
i2cdetect -y 1      # Confirm the LCD shows up at address 0x27 (= 39)
```

### 2. Install dependencies
```bash
pip install picamzero gpiozero lcd_i2c piper-tts sounddevice numpy opencv-python tflite-runtime mediapipe requests
```

### 3. Download models
Download the Piper voice model:
```bash
wget -O models/en_US-libritts-high.onnx \
  'https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/libritts/high/en_US-libritts-high.onnx?download=true'
wget -O models/en_US-libritts-high.onnx.json \
  'https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/libritts/high/en_US-libritts-high.onnx.json?download=true'
```
Then copy `emotionnet_int8.tflite` and `emotion_model_config.json` into `models/`.

### 4. Set the LiteLLM API token
The spoken feedback calls a remote LLM, so the Pi needs an internet connection and a token:
```bash
export LITELLM_TOKEN="your_token_here"
```
To persist across sessions:
```bash
echo 'export LITELLM_TOKEN="your_token_here"' >> ~/.bashrc
source ~/.bashrc
```

### 5. Test each component
Run these from the `final_pi_puzzle/` directory before starting the full game:
```bash
python tests/test_button.py     # press button → "Button pressed!" prints
python tests/test_lcd.py        # messages appear on LCD
python tests/test_sensors.py    # distances update as you move your hands
python tests/test_camera.py     # emotion label prints for each snapshot
python tests/test_speaker.py    # three phrases spoken through the speaker
```

### 6. Run the game 🎉
```bash
python robust_game.py   # full game with gesture detection + LLM feedback
python game.py          # simple version without gesture detection or LLM
```

---

## ⚙️ Configuration

All tunable values are in `config.py`:

| Key | Default | Description |
|---|---|---|
| `dist_threshold` | `20` cm | Below = Pink/Blue, above = Red/Green |
| `prepare_seconds` | `5` | Countdown duration per round |
| `hold_seconds` | `3` | Hold duration before snapshot |
| `total_rounds` | `3` | Rounds per game |
| `pass_threshold` | `2` | Minimum rounds correct to win |
| `emotion_confidence` | `0.20` | Minimum confidence for an emotion prediction |

---

Now go make a weird face and throw up some peace signs. ✌️😲✌️
