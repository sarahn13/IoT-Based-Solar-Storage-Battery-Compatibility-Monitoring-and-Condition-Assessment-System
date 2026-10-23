# IoT-Based-Solar-Storage-Battery-Compatibility-Monitoring-and-Condition-Assessment-System
README.md.
#include <WiFi.h>
#include <PubSubClient.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// ==================== CONFIGURATION ====================
const char* WIFI_SSID     = "YOUR_WIFI_SSID";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
const char* MQTT_SERVER   = "broker.hivemq.com"; // Or your custom MQTT broker IP
const int   MQTT_PORT     = 1883;

// Pin Definitions
#define PIN_TEMP_SENSOR  4   // DS18B20 OneWire Data Pin
#define PIN_RELAY_CUTOFF 26  // Relay output to disconnect load on critical failure
#define PIN_VOLTAGE_ADC  34  // Battery Voltage Divider
#define PIN_BATT_CURR    35  // Main Battery Current (ACS712)
#define PIN_LAMP_CURR    32  // Lamp Branch Current (ACS712)
#define PIN_LEAKAGE_ADC  33  // DC Residual / Leakage Current Sensor

// Sensor Calibration Factors
const float ADC_REF_VOLTAGE = 3.3;
const float VOLTAGE_DIVIDER_RATIO = 5.0; // Scaled for 0-15V measuring range
const float ACS712_SENSITIVITY = 0.185;  // 185mV/A for ACS712-5A model

// Objects
OneWire oneWire(PIN_TEMP_SENSOR);
DallasTemperature tempSensors(&oneWire);
WiFiClient espClient;
PubSubClient mqttClient(espClient);

// State Variables
bool lampCommandedState = false; // false = OFF, true = ON
bool loadCutoffState = false;

// Setup Function
void setup() {
  Serial.begin(115200);
  pinMode(PIN_RELAY_CUTOFF, OUTPUT);
  digitalWrite(PIN_RELAY_CUTOFF, HIGH); // Normal Operation: Closed Relay

  tempSensors.begin();
  setupWiFi();
  mqttClient.setServer(MQTT_SERVER, MQTT_PORT);
}

void setupWiFi() {
  delay(10);
  Serial.print("\nConnecting to WiFi: ");
  Serial.println(WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected! IP: " + WiFi.localIP().toString());
}

void reconnectMQTT() {
  while (!mqttClient.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32_SolarMonitor_" + String(random(0xffff), HEX);
    if (mqttClient.connect(clientId.c_str())) {
      Serial.println("Connected to MQTT Broker.");
    } else {
      Serial.print("Failed, rc=");
      Serial.print(mqttClient.state());
      Serial.println(" Retrying in 5 seconds...");
      delay(5000);
    }
  }
}

// Calculate State of Charge (SoC) based on standard 12V Lead-Acid curve
int calculateSoC(float voltage) {
  if (voltage >= 12.7) return 100;
  if (voltage >= 12.5) return 80;
  if (voltage >= 12.2) return 50;
  if (voltage >= 12.0) return 30;
  if (voltage >= 11.6) return 10;
  return 0;
}

void loop() {
  if (!mqttClient.connected()) {
    reconnectMQTT();
  }
  mqttClient.loop();

  // 1. Read Raw ADC Values
  int rawVoltage = analogRead(PIN_VOLTAGE_ADC);
  int rawBattCurr = analogRead(PIN_BATT_CURR);
  int rawLampCurr = analogRead(PIN_LAMP_CURR);
  int rawLeakage  = analogRead(PIN_LEAKAGE_ADC);

  // 2. Convert ADC to Calculated Values
  float batteryVoltage = (rawVoltage / 4095.0) * ADC_REF_VOLTAGE * VOLTAGE_DIVIDER_RATIO;
  
  // ACS712 zero point is VCC/2 (approx 1.65V at 3.3V power/divider logic)
  float lampCurrent = abs(((rawLampCurr / 4095.0) * ADC_REF_VOLTAGE - 1.65) / ACS712_SENSITIVITY);
  float battCurrent = ((rawBattCurr / 4095.0) * ADC_REF_VOLTAGE - 1.65) / ACS712_SENSITIVITY;
  float leakageCurrent = (rawLeakage / 4095.0) * ADC_REF_VOLTAGE; // Adjust for IMD module scaling

  tempSensors.requestTemperatures();
  float battTemp = tempSensors.getTempCByIndex(0);

  int batterySoC = calculateSoC(batteryVoltage);

  // 3. Condition Assessment & Rule-Based Fault Logic
  String conditionStatus = "NORMAL";
  bool alertTriggered = false;

  // Temperature Threshold Check
  if (battTemp > 45.0) {
    conditionStatus = "CRITICAL: OVERTEMPERATURE";
    alertTriggered = true;
  } 
  // Voltage Threshold Check
  else if (batteryVoltage < 11.2) {
    conditionStatus = "CRITICAL: LOW VOLTAGE CUTOFF";
    loadCutoffState = true;
    digitalWrite(PIN_RELAY_CUTOFF, LOW); // Cut off load to protect battery
    alertTriggered = true;
  }
  // Unexpected Drain Detection (Lamp Commanded OFF but Current > 0.05A)
  else if (!lampCommandedState && lampCurrent > 0.05) {
    conditionStatus = "WARNING: UNEXPECTED OFF-STATE DRAIN";
    alertTriggered = true;
  }
  // Electrical Leakage Detection
  else if (leakageCurrent > 0.02) { // Example threshold: 20mA
    conditionStatus = "WARNING: ELECTRICAL LEAKAGE DETECTED";
    alertTriggered = true;
  }

  // 4. Construct JSON Payload
  String payload = "{";
  payload += "\"battery_voltage\":" + String(batteryVoltage, 2) + ",";
  payload += "\"battery_current\":" + String(battCurrent, 2) + ",";
  payload += "\"lamp_current\":" + String(lampCurrent, 2) + ",";
  payload += "\"leakage_current\":" + String(leakageCurrent, 3) + ",";
  payload += "\"temperature\":" + String(battTemp, 1) + ",";
  payload += "\"soc\":" + String(batterySoC) + ",";
  payload += "\"cutoff_status\":" + String(loadCutoffState ? "true" : "false") + ",";
  payload += "\"status\":\"" + conditionStatus + "\"";
  payload += "}";

  // 5. Publish to MQTT
  mqttClient.publish("gb11/solar_monitoring/telemetry", payload.c_str());
  Serial.println("Published Telemetry: " + payload);

  delay(5000); // 5-second sample interval
}
