# WSN-based-Weather-Monitoring-system-using-GLYPH-C3
//Slave node
#include <SocketIOclient.h>
#include <WebSockets.h>
#include <WebSockets4WebServer.h>
#include <WebSocketsClient.h>
#include <WebSocketsServer.h>
#include <WebSocketsVersion.h>

#include <HTTP_Method.h>
#include <Middlewares.h>
#include <Uri.h>
#include <WebServer.h>

#include <WiFi.h>
#include <WebSocketsClient.h>
#include <ArduinoJson.h>
#include <DHT.h>

// WiFi credentials for master node
const char* ssid = "GLYPHC3_Master";
const char* password = "12345678";

// Master node IP and port
const char* masterIP = "192.168.7.12";
const uint16_t webSocketPort = 81;

// Unique node identifier - change this for each slave node
String nodeId = "Node01";  // Change to "Node02" for the second slave, etc.

 #include <ESP8266WebServer.h>
// DHT sensor setup
#define DHTPIN 2          // DHT sensor pin
#define DHTTYPE DHT22     // DHT 22 (AM2302)
DHT dht(DHTPIN, DHTTYPE);

// LDR setup
#define LDR_PIN 3         // LDR analog pin

// WebSocket client
WebSocketsClient webSocket;

// Sensor reading interval - updated to 1 second
unsigned long previousMillis = 0;
const long interval = 1000;  // 1 second

void setup() {
  Serial.begin(115200);
  
  Serial.println("ESP32-C3 Slave Node Starting");
  Serial.print("Node ID: ");
  Serial.println(nodeId);
  
  // Initialize DHT sensor
  dht.begin();
  
  // LDR pin as input
  pinMode(LDR_PIN, INPUT);
  
  // Connect to WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
  
  // Set up WebSocket connection
  webSocket.begin(masterIP, webSocketPort, "/");
  webSocket.onEvent(webSocketEvent);
  webSocket.setReconnectInterval(5000);
  
  Serial.println("WebSocket client started");
}

void loop() {
  webSocket.loop();
  
  // Read sensors and send data every 1 second
  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;
    sendSensorData();
  }
}

void webSocketEvent(WStype_t type, uint8_t * payload, size_t length) {
  switch(type) {
    case WStype_DISCONNECTED:
      Serial.println("WebSocket disconnected");
      break;
    
    case WStype_CONNECTED:
      Serial.println("WebSocket connected");
      // Send initial data immediately after connection
      sendSensorData();
      break;
    
    case WStype_TEXT:
      Serial.printf("Received text: %s\n", payload);
      break;
  }
}

void sendSensorData() {
  // Read DHT sensor
  float h = dht.readHumidity();
  float t = dht.readTemperature();
  
  // Read LDR sensor
  int lightValue = analogRead(LDR_PIN);
  
  // Check if readings are valid
  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    // Still send the node ID with default values
    h = 0;
    t = 0;
  }
  
  // Create JSON object
  DynamicJsonDocument doc(256);
  doc["nodeId"] = nodeId;  // Add the node ID to the data
  doc["temperature"] = t;
  doc["humidity"] = h;
  doc["light"] = lightValue;
  
  // Serialize JSON to string
  String jsonString;
  serializeJson(doc, jsonString);
  
  // Send data via WebSocket
  webSocket.sendTXT(jsonString);
  
  Serial.println("Sent sensor data: " + jsonString);
}
