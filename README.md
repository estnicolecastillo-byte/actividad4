# 🖐️ Sistema de Control de Iluminación Basado en Gestos con ESP32

**Universidad Militar Nueva Granada**  
**Proyecto:** Sistema de control de iluminación mediante visión artificial y microcontrolador ESP32.

---

## 📄 Descripción del Proyecto
Este proyecto implementa un sistema en tiempo real para el control de iluminación mediante gestos de la mano. Utiliza la cámara web de una laptop para capturar el video, procesa las imágenes con la librería **MediaPipe Gesture Recognizer** de Google en Python y transmite comandos serie a **115200 baudios** hacia una tarjeta **ESP32**, la cual regula la intensidad de los LEDs mediante PWM y ejecuta secuencias de luces.

---

## 🎯 Mapeo de Gestos y Acciones

| Gesto | Reconocedor MediaPipe | Carácter Serial | Acción en la ESP32 |
| :--- | :--- | :---: | :--- |
| ✊ **Puño Cerrado** | `Closed_Fist` | `'A'` | **30% de Intensidad** |
| ✌️ **Victoria / Paz** | `Victory` | `'B'` | **70% de Intensidad** |
| 🖐️ **Palma Abierta** | `Open_Palm` | `'C'` | **100% de Intensidad** |
| 👎 **Pulgar Abajo** | `Thumb_Down` | `'D'` | **Primera Interrupción:** Secuencia de luces (Modo 1) |
| 👍 **Pulgar Arriba** | `Thumb_Up` | `'E'` | **Segunda Interrupción:** Secuencia de luces (Modo 2) |

---

## 🖐️ Puntos de Referencia de la Mano (Landmarks)

El modelo de MediaPipe detecta los 21 puntos clave (*landmarks*) de la mano para clasificar cada gesto:

- **0:** WRIST (Muñeca)
- **1 - 4:** THUMB (CMC, MCP, IP, TIP)
- **5 - 8:** INDEX FINGER (MCP, PIP, DIP, TIP)
- **9 - 12:** MIDDLE FINGER (MCP, PIP, DIP, TIP)
- **13 - 16:** RING FINGER (MCP, PIP, DIP, TIP)
- **17 - 20:** PINKY (MCP, PIP, DIP, TIP)

---

## ⚡ Optimización de Rendimiento
Para evitar el congelamiento de la cámara y la saturación del búfer serial durante la ejecución en vivo, se implementaron las siguientes mejoras en Python:

1. **Resolución Reducida:** Ajuste de la captura de la webcam a $320 \times 240$ píxeles.
2. **Procesamiento de Cuadros (Frame Skipping):** Evaluación de 1 de cada 3 fotogramas (`frame_count % 3 == 0`).
3. **Gestión de Puerto Serial:** Limpieza constante del búfer (`reset_output_buffer()`) y envío restringido únicamente al detectar un cambio de gesto o transcurrido un lapso seguro de tiempo (0.3s).

---

## 💻 Código Fuente

### 1. Script en Python (`camara.py`)

```python
import cv2
import mediapipe as mp
import serial
import time

# 1. Configurar puerto Serial a 115200 baudios para la ESP32
try:
    esp32 = serial.Serial('COM3', 115200, timeout=0.01)
    esp32.reset_input_buffer()
    esp32.reset_output_buffer()
    print("ESP32 Conectado a 115200 baudios!")
    time.sleep(1.5)
except Exception as e:
    print(f"Error Serial: {e}")
    exit()

# 2. Configurar MediaPipe Gesture Recognizer
BaseOptions = mp.tasks.BaseOptions
GestureRecognizer = mp.tasks.vision.GestureRecognizer
GestureRecognizerOptions = mp.tasks.vision.GestureRecognizerOptions
VisionRunningMode = mp.tasks.vision.RunningMode

options = GestureRecognizerOptions(
    base_options=BaseOptions(model_asset_path='gesture_recognizer.task'),
    running_mode=VisionRunningMode.VIDEO,
    min_hand_detection_confidence=0.5,
    min_hand_presence_confidence=0.5,
    min_tracking_confidence=0.5,
    num_hands=1
)
recognizer = GestureRecognizer.create_from_options(options)

# 3. Inicializar Cámara en resolución ligera (320x240)
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 320)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 240)

ultimo_gesto = ""
ultimo_envio = 0
timestamp = 0
frame_count = 0

print("¡Listo! Control por gestos activado.")

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break

    frame_count += 1
    
    # Procesar 1 de cada 3 cuadros para aligerar carga
    if frame_count % 3 == 0:
        timestamp += 1
        
        mp_image = mp.Image(
            image_format=mp.ImageFormat.SRGB, 
            data=cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        )
        
        result = recognizer.recognize_for_video(mp_image, timestamp)
        tiempo_actual = time.time()

        if result.gestures:
            gesto_obj = result.gestures[0][0]
            gesto_actual = gesto_obj.category_name
            
            if gesto_obj.score > 0.5:
                if (gesto_actual != ultimo_gesto) or (tiempo_actual - ultimo_envio > 0.3):
                    ultimo_gesto = gesto_actual
                    ultimo_envio = tiempo_actual
                    
                    esp32.reset_output_buffer()
                    
                    if gesto_actual == 'Closed_Fist':
                        esp32.write(b'A')
                        print("Gesto: Puño (A) -> 30% Intensidad")
                    elif gesto_actual == 'Victory':
                        esp32.write(b'B')
                        print("Gesto: Victoria (B) -> 70% Intensidad")
                    elif gesto_actual == 'Open_Palm':
                        esp32.write(b'C')
                        print("Gesto: Palma (C) -> 100% Intensidad")
                    elif gesto_actual == 'Thumb_Down':
                        esp32.write(b'D')
                        print("Gesto: Pulgar Abajo (D) -> Interrupción Modo 1")
                    elif gesto_actual == 'Thumb_Up':
                        esp32.write(b'E')
                        print("Gesto: Pulgar Arriba (E) -> Interrupción Modo 2")

    cv2.imshow('Control por Gestos - ESP32', frame)
    
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
esp32.close()
## Código C++ para la ESP32
#define LED_PIN_1 18
#define LED_PIN_2 19
#define LED_PIN_3 21

// Canales PWM
#define PWM_CHANNEL_1 0
#define PWM_CHANNEL_2 1
#define PWM_CHANNEL_3 2

#define PWM_FREQ 5000
#define PWM_RES 8 // Resolución de 0 a 255

void setup() {
  Serial.begin(115200);

  // Configurar canales PWM
  ledcSetup(PWM_CHANNEL_1, PWM_FREQ, PWM_RES);
  ledcSetup(PWM_CHANNEL_2, PWM_CHANNEL_2, PWM_RES);
  ledcSetup(PWM_CHANNEL_3, PWM_CHANNEL_3, PWM_RES);

  ledcAttachPin(LED_PIN_1, PWM_CHANNEL_1);
  ledcAttachPin(LED_PIN_2, PWM_CHANNEL_2);
  ledcAttachPin(LED_PIN_3, PWM_CHANNEL_3);

  apagarLeds();
}

void apagarLeds() {
  ledcWrite(PWM_CHANNEL_1, 0);
  ledcWrite(PWM_CHANNEL_2, 0);
  ledcWrite(PWM_CHANNEL_3, 0);
}

void fijarIntensidad(int porcentaje) {
  int valorDuty = map(porcentaje, 0, 100, 0, 255);
  ledcWrite(PWM_CHANNEL_1, valorDuty);
  ledcWrite(PWM_CHANNEL_2, valorDuty);
  ledcWrite(PWM_CHANNEL_3, valorDuty);
}

void secuenciaModo1() {
  for (int i = 0; i < 3; i++) {
    apagarLeds();
    ledcWrite(PWM_CHANNEL_1, 255); delay(150);
    ledcWrite(PWM_CHANNEL_2, 255); delay(150);
    ledcWrite(PWM_CHANNEL_3, 255); delay(150);
  }
}

void secuenciaModo2() {
  for (int i = 0; i < 3; i++) {
    fijarIntensidad(100); delay(200);
    apagarLeds(); delay(200);
  }
}

void loop() {
  if (Serial.available() > 0) {
    char comando = Serial.read();

    switch (comando) {
      case 'A': // Puño cerrado -> 30%
        fijarIntensidad(30);
        break;
      case 'B': // Victoria -> 70%
        fijarIntensidad(70);
        break;
      case 'C': // Palma abierta -> 100%
        fijarIntensidad(100);
        break;
      case 'D': // Pulgar abajo -> Interrupción Modo 1
        secuenciaModo1();
        break;
      case 'E': // Pulgar arriba -> Interrupción Modo 2
        secuenciaModo2();
        break;
    }
  }
}
<img width="435" height="723" alt="image" src="https://github.com/user-attachments/assets/dd19526b-50f2-4aec-abac-e3734e73c8ef" />
<img width="1884" height="987" alt="image" src="https://github.com/user-attachments/assets/3c9af381-eeeb-4b9f-9fe4-11af4b8b26dd" />


