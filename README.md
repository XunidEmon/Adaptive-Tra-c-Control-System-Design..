This project presents a four-way Adaptive Traffic Control System designed using Arduino, IR
sensors, counter ICs, and 7-segment displays. The system intelligently controls traffic
signals by detecting the presence and number of vehicles on each road. If fewer than three
vehicles are detected, the light does not switch to that road, reducing unnecessary waiting
time. The Arduino acts as the data collector and decision maker, controlling counter ICs that
drive 7-segment displays through a BCD-to-7-segment decoder. The design improves traffic
flow efficiency, minimizes fuel consumption, and ensures pedestrian and emergency safety.

#define H HIGH
#define L LOW

// A side pins
int aPins[] = {4, 5, 6, 7};       
int aExtraPins[] = {13, 12, 11};  
int aStartX[] = {9, 5};
int aEndX[] = {0, 0};    
int aStartY[] = {9, 5};
int aEndY[] = {0, 0};    

// B side pins
int bPins[] = {0, 1, 2, 3};       
int bExtraPins[] = {8, 9, 10};     
int bStartX[] = {9, 5};
int bEndX[] = {0, 0};    
int bStartY[] = {9, 5};
int bEndY[] = {0, 0};    

// Button pins
int buttonX = A4;
int buttonY = A5;

// IR Sensor pins
int irPins[] = {A0, A1, A2, A3};
int vehicleCount[] = {0, 0, 0, 0};
int totalVehicleCount[] = {0, 0, 0, 0};
bool lastIRState[] = {false, false, false, false};
int irThreshold = 500;

bool xActive = true;
bool previousXActive = true;
unsigned long lastButtonCheck = 0;
const unsigned long buttonCheckInterval = 100;

void setup() {
  Serial.begin(9600);
  
  for (int i = 0; i < 4; i++) pinMode(aPins[i], OUTPUT);
  for (int i = 0; i < 3; i++) pinMode(aExtraPins[i], OUTPUT);
  for (int i = 0; i < 4; i++) pinMode(bPins[i], OUTPUT);
  for (int i = 0; i < 3; i++) pinMode(bExtraPins[i], OUTPUT);
  
  pinMode(buttonX, INPUT_PULLUP);
  pinMode(buttonY, INPUT_PULLUP);
  
  Serial.println("System Started - Press A4 for X, A5 for Y");
  Serial.println("IR Sensors: A0, A1, A2, A3");
  Serial.println("Auto switch: After function completion + Data analysis");
  Serial.println("Switch logic: Compare A0+A2 vs A1+A3");
  
  // Initial data reset
  resetCycleData();
}

void loop() {
  checkButtons();
  
  // Read IR sensors continuously
  readIRSensors();
  
  if (xActive) {
    functionX();
  } else {
    functionY();
  }
}

void checkAutoSwitch() {
  Serial.println("=== AUTO SWITCH ANALYSIS ===");
  
  // Calculate traffic density for both sides
  int trafficXside = vehicleCount[0] + vehicleCount[2]; // A0 + A2
  int trafficYside = vehicleCount[1] + vehicleCount[3]; // A1 + A3
  
  Serial.print("Traffic Analysis - X-side (A0+A2): ");
  Serial.print(trafficXside);
  Serial.print(" | Y-side (A1+A3): ");
  Serial.println(trafficYside);
  
  bool shouldSwitch = false;
  
  if (xActive) {
    // Currently in X, check if should switch to Y
    if (trafficYside > trafficXside) {
      Serial.println("Decision: More traffic on Y-side - Switching to Y");
      shouldSwitch = true;
    } else if (trafficYside == trafficXside && trafficYside > 0) {
      Serial.println("Decision: Equal traffic - Alternating to Y");
      shouldSwitch = true;
    } else if (trafficXside == 0 && trafficYside == 0) {
      Serial.println("Decision: No traffic - Alternating to Y");
      shouldSwitch = true;
    } else {
      Serial.println("Decision: More traffic on X-side - Staying in X");
    }
  } else {
    // Currently in Y, check if should switch to X
    if (trafficXside > trafficYside) {
      Serial.println("Decision: More traffic on X-side - Switching to X");
      shouldSwitch = true;
    } else if (trafficXside == trafficYside && trafficXside > 0) {
      Serial.println("Decision: Equal traffic - Alternating to X");
      shouldSwitch = true;
    } else if (trafficXside == 0 && trafficYside == 0) {
      Serial.println("Decision: No traffic - Alternating to X");
      shouldSwitch = true;
    } else {
      Serial.println("Decision: More traffic on Y-side - Staying in Y");
    }
  }
  
  // DEBUG: Print current vehicleCount before reset
  Serial.print("DEBUG - Before reset: [");
  for(int i = 0; i < 4; i++) {
    Serial.print(vehicleCount[i]);
    if(i < 3) Serial.print(",");
  }
  Serial.println("]");
  
  // ALWAYS RESET DATA - whether switching or not
  resetCycleData();
  
  if (shouldSwitch) {
    Serial.println("=== AUTO SWITCH EXECUTED ===");
    
    // Toggle the function
    xActive = !xActive;
    
    if (xActive) {
      Serial.println("Switched to X FUNCTION");
    } else {
      Serial.println("Switched to Y FUNCTION");
    }
  } else {
    Serial.println("=== NO SWITCH - CONTINUING CURRENT FUNCTION ===");
  }
  
  // DEBUG: Print current vehicleCount after reset
  Serial.print("DEBUG - After reset: [");
  for(int i = 0; i < 4; i++) {
    Serial.print(vehicleCount[i]);
    if(i < 3) Serial.print(",");
  }
  Serial.println("]");
  
  Serial.println("=======================");
}

void checkButtons() {
  if (millis() - lastButtonCheck >= buttonCheckInterval) {
    lastButtonCheck = millis();
    
    if (digitalRead(buttonX) == LOW) {
      Serial.println("=== BUTTON PRESS ===");
      Serial.println("X Button Pressed - Switching to X");
      xActive = true;
      resetCycleData(); // Reset data when manually switching
      delay(300);
    }
    
    if (digitalRead(buttonY) == LOW) {
      Serial.println("=== BUTTON PRESS ===");
      Serial.println("Y Button Pressed - Switching to Y");
      xActive = false;
      resetCycleData(); // Reset data when manually switching
      delay(300);
    }
  }
  
  // Check if function has changed and print message
  if (xActive != previousXActive) {
    Serial.println("=== FUNCTION CHANGED ===");
    if (xActive) {
      Serial.println("Now running: X FUNCTION");
    } else {
      Serial.println("Now running: Y FUNCTION");
    }
    previousXActive = xActive;
  }
}

void readIRSensors() {
  for (int i = 0; i < 4; i++) {
    int irValue = analogRead(irPins[i]);
    bool currentState = (irValue < irThreshold);
    
    // Detect rising edge (vehicle passing)
    if (currentState && !lastIRState[i]) {
      vehicleCount[i]++;
      totalVehicleCount[i]++;
      Serial.print("Vehicle detected on IR");
      Serial.print(i);
      Serial.print(" (A");
      Serial.print(i);
      Serial.print(") - Cycle: ");
      Serial.print(vehicleCount[i]);
      Serial.print(" | Total: ");
      Serial.println(totalVehicleCount[i]);
    }
    
    lastIRState[i] = currentState;
  }
}

void printCycleSummary() {
  Serial.println("=== CYCLE COMPLETED ===");
  Serial.print("Cycle Counts - A0:");
  Serial.print(vehicleCount[0]);
  Serial.print(" A1:");
  Serial.print(vehicleCount[1]);
  Serial.print(" A2:");
  Serial.print(vehicleCount[2]);
  Serial.print(" A3:");
  Serial.println(vehicleCount[3]);
  
  Serial.print("Traffic Summary - X-side (A0+A2): ");
  Serial.print(vehicleCount[0] + vehicleCount[2]);
  Serial.print(" | Y-side (A1+A3): ");
  Serial.println(vehicleCount[1] + vehicleCount[3]);
  
  Serial.print("Total Counts - A0:");
  Serial.print(totalVehicleCount[0]);
  Serial.print(" A1:");
  Serial.print(totalVehicleCount[1]);
  Serial.print(" A2:");
  Serial.print(totalVehicleCount[2]);
  Serial.print(" A3:");
  Serial.println(totalVehicleCount[3]);
  Serial.println("=======================");
}

void resetCycleData() {
  Serial.print("*** RESETTING CYCLE DATA - Before: [");
  for(int i = 0; i < 4; i++) {
    Serial.print(vehicleCount[i]);
    if(i < 3) Serial.print(",");
  }
  Serial.print("]");
  
  for (int i = 0; i < 4; i++) {
    vehicleCount[i] = 0;
  }
  
  Serial.print(" After: [");
  for(int i = 0; i < 4; i++) {
    Serial.print(vehicleCount[i]);
    if(i < 3) Serial.print(",");
  }
  Serial.println("] ***");
}

void functionX() {
  Serial.println("X Function Running...");
  
  for (int seq = 0; seq < 2 && xActive; seq++) {
    int aExtra = aExtraPins[seq];
    int bExtra = bExtraPins[seq];

    int aStep = (aStartX[seq] >= aEndX[seq]) ? -1 : 1;
    int bStep = (bStartX[seq] >= bEndX[seq]) ? -1 : 1;

    int aSteps = abs(aStartX[seq] - aEndX[seq]);
    int bSteps = abs(bStartX[seq] - bEndX[seq]);
    int maxSteps = max(aSteps, bSteps);

    for (int step = 0; step <= maxSteps && xActive; step++) {
      checkButtons();
      if (!xActive) {
        Serial.println("*** X Function Interrupted ***");
        return;
      }
      
      int aNum = aStartX[seq] + step * aStep;
      if (aStep > 0) aNum = min(aNum, aEndX[seq]);
      else aNum = max(aNum, aEndX[seq]);

      int bNum = bStartX[seq] + step * bStep;
      if (bStep > 0) bNum = min(bNum, bEndX[seq]);
      else bNum = max(bNum, bEndX[seq]);

      // Update pins
      for (int i = 0; i < 4; i++) digitalWrite(aPins[i], (aNum & (1 << i)) ? H : L);
      for (int i = 0; i < 3; i++) digitalWrite(aExtraPins[i], L);
      digitalWrite(aExtra, H);

      for (int i = 0; i < 4; i++) digitalWrite(bPins[i], (bNum & (1 << i)) ? H : L);
      for (int i = 0; i < 3; i++) digitalWrite(bExtraPins[i], L);
      digitalWrite(bExtra, H);

      // Non-blocking delay
      unsigned long startTime = millis();
      while (millis() - startTime < 1000 && xActive) {
        checkButtons();
        readIRSensors();
        delay(100);
      }
    }
  }
  
  // X cycle completed - NOW DO AUTO SWITCH ANALYSIS
  if (xActive) {
    printCycleSummary();
    checkAutoSwitch(); // Data analysis and decision making
  }
}

void functionY() {
  Serial.println("Y Function Running...");
  
  for (int seq = 0; seq < 2 && !xActive; seq++) {
    int aExtra = aExtraPins[2-seq];
    int bExtra = bExtraPins[2-seq];

    int aStep = (aStartY[seq] >= aEndY[seq]) ? -1 : 1;
    int bStep = (bStartY[seq] >= bEndY[seq]) ? -1 : 1;

    int aSteps = abs(aStartY[seq] - aEndY[seq]);
    int bSteps = abs(bStartY[seq] - bEndY[seq]);
    int maxSteps = max(aSteps, bSteps);

    for (int step = 0; step <= maxSteps && !xActive; step++) {
      checkButtons();
      if (xActive) {
        Serial.println("*** Y Function Interrupted ***");
        return;
      }
      
      int aNum = aStartY[seq] + step * aStep;
      if (aStep > 0) aNum = min(aNum, aEndY[seq]);
      else aNum = max(aNum, aEndY[seq]);

      int bNum = bStartY[seq] + step * bStep;
      if (bStep > 0) bNum = min(bNum, bEndY[seq]);
      else bNum = max(bNum, bEndY[seq]);

      // Update pins
      for (int i = 0; i < 4; i++) digitalWrite(aPins[i], (aNum & (1 << i)) ? H : L);
      for (int i = 0; i < 3; i++) digitalWrite(aExtraPins[i], L);
      digitalWrite(aExtra, H);

      for (int i = 0; i < 4; i++) digitalWrite(bPins[i], (bNum & (1 << i)) ? H : L);
      for (int i = 0; i < 3; i++) digitalWrite(bExtraPins[i], L);
      digitalWrite(bExtra, H);

      // Non-blocking delay
      unsigned long startTime = millis();
      while (millis() - startTime < 1000 && !xActive) {
        checkButtons();
        readIRSensors();
        delay(100);
      }
    }
  }
  
  // Y cycle completed - NOW DO AUTO SWITCH ANALYSIS
  if (!xActive) {
    printCycleSummary();
    checkAutoSwitch(); // Data analysis and decision making
  }
}
