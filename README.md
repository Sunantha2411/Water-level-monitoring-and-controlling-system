# ESP8266 Water Level Monitoring and Control System

This project uses an ESP8266 microcontroller to monitor the water level in a tank and control a relay based on the water level. The system uses an ultrasonic sensor to measure the water level and Blynk for remote monitoring and control.

## Features

- Monitor water level using an ultrasonic sensor.
- Display water level percentage on an OLED screen.
- Control a relay based on water level to automatically fill or stop filling the tank.
- Switch between automatic and manual modes.
- Remote monitoring and control via Blynk.

## Technologies Used

- **ESP8266**: Microcontroller used for the project.
- **Arduino IDE**: Software used for programming the ESP8266.
- **Blynk**: IoT platform for remote monitoring and control.
- **Adafruit SSD1306**: Library for the OLED display.
- **AceButton**: Library for handling button inputs.

## Requirements

- ESP8266 microcontroller.
- Ultrasonic sensor (e.g., HC-SR04).
- Relay module.
- OLED display (128x32).
- Push buttons.
- Blynk account and project setup.

## Installation

1. **Clone the Repository:**

    ```sh
    git clone https://github.com/yourusername/ESP8266_Water_Level_Monitor
    cd ESP8266_Water_Level_Monitor
    ```

2. **Install Dependencies:**

    Make sure you have the required libraries installed in your Arduino IDE:
    - `ESP8266WiFi`
    - `BlynkSimpleEsp8266`
    - `Adafruit_SSD1306`
    - `AceButton`

3. **Configure Blynk:**

    - Create a new Blynk project.
    - Note the `Auth Token` provided by Blynk.
    - Configure virtual pins `V1` for water level percentage, `V3` for mode selection, and `V4` for relay control.

4. **Update the Code:**

    Replace the placeholder values with your actual WiFi credentials and Blynk authentication token.

    ```cpp
    #define BLYNK_TEMPLATE_ID "TMPL6gRVFZngK"
    #define BLYNK_TEMPLATE_NAME "Sunantha"
    #define BLYNK_AUTH_TOKEN "YvjMyP1EfHoHaHia_KXscGKa2kokRc73"
    
    const char* ssid = "YourSSID";
    const char* pass = "YourPassword";
    ```

5. **Upload the Code:**

    - Connect your ESP8266 to your computer.
    - Select the correct board and port in the Arduino IDE.
    - Upload the code to the ESP8266.

## Usage

1. **Power the ESP8266:**

    - Connect the ESP8266 to a power source.

2. **Monitor and Control:**

    - Use the Blynk app to monitor the water level and control the relay.
    - Switch between automatic and manual modes using the buttons or the Blynk app.
    - View the current water level percentage and system status on the OLED display.

## Code

```cpp
#define BLYNK_TEMPLATE_ID "TMPL6gRVFZngK"
#define BLYNK_TEMPLATE_NAME "Sunantha"
#define BLYNK_AUTH_TOKEN "YvjMyP1EfHoHaHia_KXscGKa2kokRc73"

#include <Adafruit_SSD1306.h>
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <AceButton.h>

char ssid[] = "YourSSID";
char pass[] = "YourPassword";

int emptyTankDistance = 160;
int fullTankDistance = 20;
int triggerPer = 20;

using namespace ace_button;

#define TRIG 12                //D6
#define ECHO 13                //D7
#define Relay 14           //D5
#define BP1 2              //D0
#define BP2 13             //D3
#define BP3 15             //D4

#define V_B_1 V1
#define V_B_3 V3
#define V_B_4 V4

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32

#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

float duration;
float distance;
int waterLevelPer;

bool toggleRelay = false;
bool modeFlag = true;
String currMode;

char auth[] = BLYNK_AUTH_TOKEN;

ButtonConfig config1;
AceButton button1(&config1);
ButtonConfig config2;
AceButton button2(&config2);
ButtonConfig config3;
AceButton button3(&config3);

void handleEvent1(AceButton*, uint8_t, uint8_t);
void handleEvent2(AceButton*, uint8_t, uint8_t);
void handleEvent3(AceButton*, uint8_t, uint8_t);

BlynkTimer timer;

void checkBlynkStatus() {

  bool isconnected = Blynk.connected();
  if (isconnected == false) {
  }
  if (isconnected == true) {
  }
}

BLYNK_WRITE(VPIN_BUTTON_3) {
  modeFlag = param.asInt();
  if (!modeFlag && toggleRelay) {
    digitalWrite(Relay, LOW);
    toggleRelay = false;
  }
  currMode = modeFlag ? "AUTO" : "MANUAL";
}

BLYNK_WRITE(VPIN_BUTTON_4) {
  if (!modeFlag) {
    toggleRelay = param.asInt();
    digitalWrite(Relay, toggleRelay);
  } else {
    Blynk.virtualWrite(V_B_4, toggleRelay);
  }
}

BLYNK_CONNECTED() {
  Blynk.syncVirtual(V_B_1);
  Blynk.virtualWrite(V_B_3, modeFlag);
  Blynk.virtualWrite(V_B_4, toggleRelay);
}

void displayData() {
  display.clearDisplay();
  display.setTextSize(3);
  display.setCursor(30, 0);
  display.print(waterLevelPer);
  display.print(" ");
  display.print("%");
  display.setTextSize(1);
  display.setCursor(20, 25);
  display.print(currMode);
  display.setCursor(95, 25);
  display.print(toggleRelay ? "ON" : "OFF");
  display.display();
}

void measureDistance() {

  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(20);
  digitalWrite(TRIG, LOW);
  duration = pulseIn(ECHO, HIGH);
  distance = ((duration / 2) * 0.343) / 10;
  if (distance > (fullTankDistance - 15) && distance < emptyTankDistance) {
    waterLevelPer = map((int)distance, emptyTankDistance, fullTankDistance, 0, 100);
    Blynk.virtualWrite(V_B_1, waterLevelPer);
    if (waterLevelPer < triggerPer) {
      if (modeFlag) {
        if (!toggleRelay) {
          digitalWrite(Relay, HIGH);
          toggleRelay = true;
          Blynk.virtualWrite(V_B_4, toggleRelay);
        }
      }
    }
    if (distance < fullTankDistance) {
      if (modeFlag) {
        if (toggleRelay) {
          digitalWrite(Relay, LOW);
          toggleRelay = false;
          Blynk.virtualWrite(V_B_4, toggleRelay);
        }
      }
    }
  }
  displayData();
  delay(100);
}

void setup() {
  Serial.begin(9600);
  pinMode(ECHO, INPUT);
  pinMode(TRIG, OUTPUT);
  pinMode(Relay, OUTPUT);

  pinMode(BP1, INPUT_PULLUP);
  pinMode(BP2, INPUT_PULLUP);
  pinMode(BP3, INPUT_PULLUP);

  digitalWrite(Relay, HIGH);

  config1.setEventHandler(button1Handler);
  config2.setEventHandler(button2Handler);
  config3.setEventHandler(button3Handler);

  button1.init(BP1);
  button2.init(BP2);
  button3.init(BP3);

  currMode = modeFlag ? "AUTO" : "MANUAL";

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;)
      ;
  }
  delay(1000);
  display.setTextSize(1);
  display.setTextColor(WHITE);
  display.clearDisplay();

  WiFi.begin(ssid, pass);
  timer.setInterval(2000L, checkBlynkStatus);
  timer.setInterval(1000L, measureDistance);
  Blynk.config(auth);
  delay(1000);

  Blynk.virtualWrite(V_B_3, modeFlag);
  Blynk.virtualWrite(V_B_4, toggleRelay);

  delay(500);
}

void loop() {
  Blynk.run();
  timer.run();
  button1.check();
  button3.check();

  if (!modeFlag) {
    button2.check();
  }
}

void button1Handler(AceButton* button, uint8_t eventType, uint8_t buttonState) {
  Serial.println("EVENT1");
  switch (eventType) {
    case AceButton::kEventReleased:
      if (modeFlag && toggleRelay) {
        digitalWrite(Relay, LOW);
        toggleRelay = false;
      }
      modeFlag = !modeFlag;
      currMode = modeFlag ? "AUTO" : "MANUAL";
      Blynk.virtualWrite(V_B_3, modeFlag);
      break;
  }
}

void button2Handler(AceButton* button, uint8_t eventType, uint8_t buttonState) {
  Serial.println("EVENT2");
  switch (eventType) {
    case AceButton::kEventReleased:
      if (toggleRelay) {
        digitalWrite(Relay, LOW);
        toggleRelay = false;
      } else {
        digitalWrite(Relay, HIGH);
        toggleRelay = true;
      }
      Blynk.virtualWrite(V_B_4, toggleRelay);
      delay(1000);
      break;
  }
}

void button3Handler(AceButton* button, uint8_t eventType, uint8_t buttonState) {
  Serial.println("EVENT3");
  switch (eventType) {
    case AceButton::kEventReleased:
      break;
  }
}
