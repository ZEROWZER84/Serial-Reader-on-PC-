# Serial-Reader-on-PC

#include <SoftwareSerial.h>

#define GAS_SENSOR A0
#define TEMP_SENSOR A1
#define SENSOR_3 A2

SoftwareSerial esp(10, 11); // RX, TX

unsigned long lastRead = 0;
const unsigned long interval = 2000; // Interval for sensor readings (2 seconds)

void setup() {
  Serial.begin(9600);
  esp.begin(9600);
  delay(1000); // Give ESP module time to initialize
  
  Serial.println("Starting setup...");
  if (setupWiFi()) {
    Serial.println("WiFi setup completed successfully!");
  } else {
    Serial.println("WiFi setup failed!");
  }
}

void loop() {
  if (millis() - lastRead >= interval) {
    // Read sensor values
    int gas = analogRead(GAS_SENSOR);
    int temp = analogRead(TEMP_SENSOR);
    int s3 = analogRead(SENSOR_3);

    // Print sensor values for PC to read
    Serial.print("Sensor values: gas=");
    Serial.print(gas);
    Serial.print(", temp=");
    Serial.print(temp);
    Serial.print(", s3=");
    Serial.println(s3);

    // Send data to server
    if (sendDataToServer(gas, temp, s3)) {
      Serial.println("Data sent successfully!");
    } else {
      Serial.println("Failed to send data!");
    }

    lastRead = millis();
  }
}

// Send AT command and check response
bool sendAT(const char* cmd, unsigned long timeout = 2000) {
  esp.println(cmd);
  Serial.print("Sent AT: ");
  Serial.println(cmd);

  String response;
  unsigned long startTime = millis();
  while (millis() - startTime < timeout) {
    while (esp.available()) {
      char c = esp.read();
      response += c;
      Serial.write(c); // Send to PC for debugging
    }
    if (response.indexOf("OK") >= 0) {
      return true;
    }
    if (response.indexOf("ERROR") >= 0) {
      return false;
    }
    delay(10);
  }
  Serial.println("No valid response for command: " + String(cmd));
  return false;
}

// Set up WiFi connection
bool setupWiFi() {
  if (!sendAT("AT", 1000)) {
    Serial.println("ESP module not responding!");
    return false;
  }
  if (!sendAT("AT+CWMODE=1", 1000)) {
    Serial.println("Failed to set WiFi mode!");
    return false;
  }
  if (!sendAT("AT+CWJAP=\"YourSSID\",\"YourPassword\"", 10000)) {
    Serial.println("Failed to connect to WiFi!");
    return false;
  }
  return true;
}

// Send sensor data to server
bool sendDataToServer(int gas, int temp, int s3) {
  char cmd[100];
  char data[150];

  snprintf(cmd, sizeof(cmd), "AT+CIPSTART=\"TCP\",\"192.168.1.100\",80");
  if (!sendAT(cmd, 5000)) {
    Serial.println("Failed to connect to server!");
    return false;
  }

  snprintf(data, sizeof(data), "GET /log?gas=%d&temp=%d&s3=%d HTTP/1.1\r\nHost: 192.168.1.100\r\n\r\n", gas, temp, s3);
  snprintf(cmd, sizeof(cmd), "AT+CIPSEND=%d", strlen(data));
  if (!sendAT(cmd, 1000)) {
    Serial.println("Failed to prepare data send!");
    return false;
  }

  esp.print(data);
  Serial.print("Sent HTTP: ");
  Serial.println(data);

  String response;
  unsigned long startTime = millis();
  while (millis() - startTime < 5000) {
    while (esp.available()) {
      char c = esp.read();
      response += c;
      Serial.write(c);
    }
    if (response.indexOf("HTTP/1.1") >= 0) {
      break;
    }
    delay(10);
  }

  if (!sendAT("AT+CIPCLOSE", 1000)) {
    Serial.println("Failed to close TCP connection!");
    return false;
  }

  if (response.indexOf("200 OK") >= 0) {
    return true;
  } else {
    Serial.println("Server response not OK: " + response);
    return false;
  }
}
