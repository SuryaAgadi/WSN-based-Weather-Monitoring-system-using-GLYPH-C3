#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <ArduinoJson.h>

// WiFi credentials for SoftAP
const char* ssid = "GLYPHC3_Master";
const char* password = "12345678";  // at least 8 characters

// IP Configuration
IPAddress local_ip(192, 168, 7, 12);
IPAddress gateway(192, 168, 7, 1);
IPAddress subnet(255, 255, 255, 0);

// Server and WebSocket
WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

// Sensor data storage structure - one for each node
struct NodeData {
  String nodeId = "Unknown";
  float temperature = 0.0;
  float humidity = 0.0;
  int light = 0;
  bool isActive = false;
  unsigned long lastUpdate = 0;
};

// Array to store data for multiple nodes
NodeData nodeData[2]; // For two nodes

// WebSocket event handler
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch(type) {
    case WStype_DISCONNECTED:
      Serial.printf("[%u] Disconnected!\n", num);
      break;
    
    case WStype_CONNECTED:
      {
        IPAddress ip = webSocket.remoteIP(num);
        Serial.printf("[%u] Connected from %d.%d.%d.%d\n", num, ip[0], ip[1], ip[2], ip[3]);
      }
      break;
    
    case WStype_TEXT:
      {
        Serial.printf("[%u] Received text: %s\n", num, payload);
        
        // Parse JSON data
        DynamicJsonDocument doc(256);
        DeserializationError error = deserializeJson(doc, payload);
        
        if (error) {
          Serial.print("deserializeJson() failed: ");
          Serial.println(error.c_str());
          return;
        }
        
        // Check if the message contains a node ID
        if (doc.containsKey("nodeId")) {
          String nodeId = doc["nodeId"].as<String>();
          
          // Find the proper node to update, or use the first inactive node
          int nodeIndex = -1;
          
          // First check if we already have this node
          for (int i = 0; i < 2; i++) {
            if (nodeData[i].isActive && nodeData[i].nodeId == nodeId) {
              nodeIndex = i;
              break;
            }
          }
          
          // If node not found, use the first inactive slot
          if (nodeIndex == -1) {
            for (int i = 0; i < 2; i++) {
              if (!nodeData[i].isActive) {
                nodeIndex = i;
                nodeData[i].isActive = true;
                nodeData[i].nodeId = nodeId;
                break;
              }
            }
          }
          
          // If we found a slot to use
          if (nodeIndex != -1) {
            // Extract sensor data
            if (doc.containsKey("temperature")) nodeData[nodeIndex].temperature = doc["temperature"];
            if (doc.containsKey("humidity")) nodeData[nodeIndex].humidity = doc["humidity"];
            if (doc.containsKey("light")) nodeData[nodeIndex].light = doc["light"];
            nodeData[nodeIndex].lastUpdate = millis();
            
            // Prepare data for broadcast
            DynamicJsonDocument broadcastDoc(512);
            
            for (int i = 0; i < 2; i++) {
              if (nodeData[i].isActive) {
                JsonObject node = broadcastDoc.createNestedObject("node" + String(i+1));
                node["nodeId"] = nodeData[i].nodeId;
                node["temperature"] = nodeData[i].temperature;
                node["humidity"] = nodeData[i].humidity;
                node["light"] = nodeData[i].light;
                node["isActive"] = true;
              } else {
                JsonObject node = broadcastDoc.createNestedObject("node" + String(i+1));
                node["isActive"] = false;
              }
            }
            
            String broadcastJson;
            serializeJson(broadcastDoc, broadcastJson);
            webSocket.broadcastTXT(broadcastJson);
          }
        }
      }
      break;
  }
}

// Check for inactive nodes
void checkInactiveNodes() {
  unsigned long currentTime = millis();
  bool changed = false;
  
  for (int i = 0; i < 2; i++) {
    // If no update for 10 seconds, mark as inactive
    if (nodeData[i].isActive && (currentTime - nodeData[i].lastUpdate > 10000)) {
      nodeData[i].isActive = false;
      changed = true;
    }
  }
  
  // If any node status changed, broadcast update
  if (changed) {
    DynamicJsonDocument broadcastDoc(512);
    
    for (int i = 0; i < 2; i++) {
      JsonObject node = broadcastDoc.createNestedObject("node" + String(i+1));
      if (nodeData[i].isActive) {
        node["nodeId"] = nodeData[i].nodeId;
        node["temperature"] = nodeData[i].temperature;
        node["humidity"] = nodeData[i].humidity;
        node["light"] = nodeData[i].light;
        node["isActive"] = true;
      } else {
        node["isActive"] = false;
      }
    }
    
    String broadcastJson;
    serializeJson(broadcastDoc, broadcastJson);
    webSocket.broadcastTXT(broadcastJson);
  }
}

void setup() {
  Serial.begin(115200);
  
  // Configure SoftAP
  WiFi.softAP(ssid, password);
  WiFi.softAPConfig(local_ip, gateway, subnet);
  
  Serial.println("SoftAP started");
  Serial.print("IP address: ");
  Serial.println(WiFi.softAPIP());
  
  // Start WebSocket server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  Serial.println("WebSocket server started");
  
  // Define Web server routes
  server.on("/", handleRoot);
  server.onNotFound(handleNotFound);
  
  // Start server
  server.begin();
  Serial.println("HTTP server started");
  
  // Initialize node data
  for (int i = 0; i < 2; i++) {
    nodeData[i].isActive = false;
  }
}

unsigned long previousMillis = 0;
const long checkInterval = 1000;  // Check for inactive nodes every second

void loop() {
  webSocket.loop();
  server.handleClient();
  
  // Periodically check for inactive nodes
  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= checkInterval) {
    previousMillis = currentMillis;
    checkInactiveNodes();
  }
}

// HTML content for the web page
void handleRoot() {
  String html = R"(
<!DOCTYPE html>
<html>
<head>
  <title>ESP32-C3 Master Node</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: Arial, sans-serif; margin: 0; padding: 20px; text-align: center; }
    .container { display: flex; flex-direction: column; align-items: center; }
    .node-container { 
      width: 100%; 
      max-width: 800px; 
      margin: 10px 0; 
      padding: 15px;
      border: 1px solid #ccc;
      border-radius: 10px;
    }
    .node-heading {
      font-size: 18px;
      font-weight: bold;
      margin-bottom: 10px;
      padding-bottom: 5px;
      border-bottom: 1px solid #eee;
    }
    .data-blocks {
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
    }
    .data-block { 
      width: 28%; 
      margin: 5px; 
      padding: 10px; 
      border-radius: 10px; 
      box-shadow: 0 2px 5px rgba(0,0,0,0.2); 
      text-align: center;
    }
    .temperature { background-color: #FFE0E0; }
    .humidity { background-color: #E0E0FF; }
    .light { background-color: #FFFFE0; }
    h2 { margin-top: 0; font-size: 16px; }
    .value { font-size: 20px; font-weight: bold; }
    .inactive { 
      text-align: center; 
      padding: 20px; 
      color: #999; 
      font-style: italic;
    }
  </style>
</head>
<body>
  <h1>ESP32-C3 Wireless Sensor Network</h1>
  
  <div class="container">
    <!-- Node 1 -->
    <div class="node-container" id="node1-container">
      <div class="node-heading">Node 1: <span id="node1-id">-</span></div>
      <div id="node1-active">
        <div class="data-blocks">
          <div class="data-block temperature">
            <h2>Temperature</h2>
            <div class="value" id="node1-temp">-- °C</div>
          </div>
          <div class="data-block humidity">
            <h2>Humidity</h2>
            <div class="value" id="node1-hum">-- %</div>
          </div>
          <div class="data-block light">
            <h2>Light</h2>
            <div class="value" id="node1-light">--</div>
          </div>
        </div>
      </div>
      <div id="node1-inactive" class="inactive" style="display:none;">
        Node is offline or not connected
      </div>
    </div>
    
    <!-- Node 2 -->
    <div class="node-container" id="node2-container">
      <div class="node-heading">Node 2: <span id="node2-id">-</span></div>
      <div id="node2-active">
        <div class="data-blocks">
          <div class="data-block temperature">
            <h2>Temperature</h2>
            <div class="value" id="node2-temp">-- °C</div>
          </div>
          <div class="data-block humidity">
            <h2>Humidity</h2>
            <div class="value" id="node2-hum">-- %</div>
          </div>
          <div class="data-block light">
            <h2>Light</h2>
            <div class="value" id="node2-light">--</div>
          </div>
        </div>
      </div>
      <div id="node2-inactive" class="inactive" style="display:none;">
        Node is offline or not connected
      </div>
    </div>
  </div>

  <script>
    var gateway = `ws://${window.location.hostname}:81`;
    var websocket;
    
    window.addEventListener('load', onLoad);

    function onLoad(event) {
      initWebSocket();
    }

    function initWebSocket() {
      console.log('Trying to open a WebSocket connection...');
      websocket = new WebSocket(gateway);
      websocket.onopen = onOpen;
      websocket.onclose = onClose;
      websocket.onmessage = onMessage;
    }

    function onOpen(event) {
      console.log('Connection opened');
    }

    function onClose(event) {
      console.log('Connection closed');
      setTimeout(initWebSocket, 2000);
    }

    function onMessage(event) {
      var data = JSON.parse(event.data);
      
      // Update Node 1
      if (data.node1) {
        if (data.node1.isActive) {
          document.getElementById('node1-id').innerHTML = data.node1.nodeId;
          document.getElementById('node1-temp').innerHTML = data.node1.temperature.toFixed(1) + ' °C';
          document.getElementById('node1-hum').innerHTML = data.node1.humidity.toFixed(1) + ' %';
          document.getElementById('node1-light').innerHTML = data.node1.light;
          document.getElementById('node1-active').style.display = 'block';
          document.getElementById('node1-inactive').style.display = 'none';
        } else {
          document.getElementById('node1-active').style.display = 'none';
          document.getElementById('node1-inactive').style.display = 'block';
        }
      }
      
      // Update Node 2
      if (data.node2) {
        if (data.node2.isActive) {
          document.getElementById('node2-id').innerHTML = data.node2.nodeId;
          document.getElementById('node2-temp').innerHTML = data.node2.temperature.toFixed(1) + ' °C';
          document.getElementById('node2-hum').innerHTML = data.node2.humidity.toFixed(1) + ' %';
          document.getElementById('node2-light').innerHTML = data.node2.light;
          document.getElementById('node2-active').style.display = 'block';
          document.getElementById('node2-inactive').style.display = 'none';
        } else {
          document.getElementById('node2-active').style.display = 'none';
          document.getElementById('node2-inactive').style.display = 'block';
        }
      }
    }
  </script>
</body>
</html>
)";

  server.send(200, "text/html", html);
}

void handleNotFound() {
  server.send(404, "text/plain", "404: Not found");
}
