#include <Servo.h>

// Create servo objects for each motor/propeller unit
Servo motor1;  // Front Left Propeller/Wheel
Servo motor2;  // Front Right Propeller/Wheel
Servo motor3;  // Back Left Propeller/Wheel
Servo motor4;  // Back Right Propeller/Wheel

// Additional servos for propeller positioning (tilt mechanism)
Servo tilt1;   // Front Left Tilt Servo
Servo tilt2;   // Front Right Tilt Servo
Servo tilt3;   // Back Left Tilt Servo
Servo tilt4;   // Back Right Tilt Servo

// Pin definitions
const int MOTOR1_PIN = 3;
const int MOTOR2_PIN = 5;
const int MOTOR3_PIN = 6;
const int MOTOR4_PIN = 9;
const int TILT1_PIN = 10;
const int TILT2_PIN = 11;
const int TILT3_PIN = 12;
const int TILT4_PIN = 13;

// Motor speed variables (1000-2000 microseconds)
int motor1Speed = 1000;
int motor2Speed = 1000;
int motor3Speed = 1000;
int motor4Speed = 1000;

// Tilt positions (0-180 degrees)
int tilt1Pos = 90;  // 90 = horizontal (wheel mode), 0 = vertical up (drone mode)
int tilt2Pos = 90;
int tilt3Pos = 90;
int tilt4Pos = 90;

// Control variables
int baseThrottle = 1000;
int maxThrottle = 2000;
int minThrottle = 1000;

// Operation modes
enum OperationMode {
  DRONE_MODE,    // Propellers vertical for flight
  WHEEL_MODE,    // Propellers horizontal for ground movement
  TRANSITION     // Moving between modes
};

OperationMode currentMode = WHEEL_MODE;
bool isArmed = false;

// Vehicle parameters
const float VEHICLE_WEIGHT = 2.5;  // kg (heavier due to hybrid design)
const float WHEEL_DIAMETER = 0.3;  // meters (propeller acting as wheel)
const float MAX_GROUND_SPEED = 5.0; // m/s

void setup() {
  Serial.begin(9600);
  
  // Initialize motor ESCs
  motor1.attach(MOTOR1_PIN);
  motor2.attach(MOTOR2_PIN);
  motor3.attach(MOTOR3_PIN);
  motor4.attach(MOTOR4_PIN);
  
  // Initialize tilt servos
  tilt1.attach(TILT1_PIN);
  tilt2.attach(TILT2_PIN);
  tilt3.attach(TILT3_PIN);
  tilt4.attach(TILT4_PIN);
  
  // Initialize to wheel mode
  initializeSystem();
  setWheelMode();
  
  Serial.println("=== HYBRID DRONE-ROVER CONTROL SYSTEM ===");
  Serial.println("Commands:");
  Serial.println("'a' = arm system    'd' = disarm system");
  Serial.println("'f' = drone mode    'w' = wheel mode");
  Serial.println("'+' = increase      '-' = decrease");
  Serial.println("'u' = move forward  'b' = move backward");
  Serial.println("'l' = turn left     'r' = turn right");
  Serial.println("'h' = hover (drone) 's' = stop");
  
  delay(2000);
}

void loop() {
  // Check for serial commands
  if (Serial.available() > 0) {
    char command = Serial.read();
    handleCommand(command);
  }
  
  // Update motor speeds and positions
  updateSystem();
  
  // Display system status
  displaySystemStatus();
  
  delay(100);
}

void initializeSystem() {
  // Initialize motors to minimum throttle
  motor1.writeMicroseconds(1000);
  motor2.writeMicroseconds(1000);
  motor3.writeMicroseconds(1000);
  motor4.writeMicroseconds(1000);
  
  // Initialize tilt servos to horizontal (wheel mode)
  tilt1.write(90);
  tilt2.write(90);
  tilt3.write(90);
  tilt4.write(90);
  
  delay(3000);
  Serial.println("System initialized in WHEEL MODE");
}

void handleCommand(char command) {
  switch (command) {
    case 'a':  // Arm system
      armSystem();
      break;
    case 'd':  // Disarm system
      disarmSystem();
      break;
    case 'f':  // Switch to flight/drone mode
      setDroneMode();
      break;
    case 'w':  // Switch to wheel mode
      setWheelMode();
      break;
    case '+':  // Increase power
      increasePower();
      break;
    case '-':  // Decrease power
      decreasePower();
      break;
    case 'u':  // Move forward
      moveForward();
      break;
    case 'b':  // Move backward
      moveBackward();
      break;
    case 'l':  // Turn left
      turnLeft();
      break;
    case 'r':  // Turn right
      turnRight();
      break;
    case 'h':  // Hover (drone mode only)
      hover();
      break;
    case 's':  // Stop/idle
      stopMovement();
      break;
  }
}

void armSystem() {
  if (!isArmed) {
    baseThrottle = 1100;
    isArmed = true;
    Serial.println("SYSTEM ARMED - Ready for operation");
    if (currentMode == WHEEL_MODE) {
      Serial.println("Ground movement enabled");
    } else {
      Serial.println("Flight controls enabled");
    }
  }
}

void disarmSystem() {
  isArmed = false;
  baseThrottle = 1000;
  updateAllMotors(1000);
  Serial.println("SYSTEM DISARMED - Safe state");
}

void setDroneMode() {
  if (currentMode != DRONE_MODE) {
    Serial.println("Transitioning to DRONE MODE...");
    currentMode = TRANSITION;
    
    // Gradually tilt propellers to vertical position
    for (int angle = 90; angle >= 0; angle -= 5) {
      tilt1.write(angle);
      tilt2.write(angle);
      tilt3.write(angle);
      tilt4.write(angle);
      delay(50);
    }
    
    currentMode = DRONE_MODE;
    tilt1Pos = tilt2Pos = tilt3Pos = tilt4Pos = 0;
    Serial.println("DRONE MODE ACTIVE - Propellers vertical for flight");
    
    if (isArmed) {
      Serial.println("Ready for takeoff - Use 'h' for hover, '+/-' for altitude");
    }
  }
}

void setWheelMode() {
  if (currentMode != WHEEL_MODE) {
    Serial.println("Transitioning to WHEEL MODE...");
    currentMode = TRANSITION;
    
    // Gradually tilt propellers to horizontal position
    for (int angle = tilt1Pos; angle <= 90; angle += 5) {
      tilt1.write(angle);
      tilt2.write(angle);
      tilt3.write(angle);
      tilt4.write(angle);
      delay(50);
    }
    
    currentMode = WHEEL_MODE;
    tilt1Pos = tilt2Pos = tilt3Pos = tilt4Pos = 90;
    Serial.println("WHEEL MODE ACTIVE - Propellers horizontal for ground movement");
    
    if (isArmed) {
      Serial.println("Ready to roll - Use 'u/b' for forward/back, 'l/r' for turns");
    }
  }
}

void increasePower() {
  if (!isArmed) {
    Serial.println("System must be armed first!");
    return;
  }
  
  baseThrottle += 50;
  if (baseThrottle > maxThrottle) {
    baseThrottle = maxThrottle;
  }
  
  if (currentMode == DRONE_MODE) {
    updateAllMotors(baseThrottle);
    Serial.print("Flight power increased to: ");
  } else {
    Serial.print("Ground power increased to: ");
  }
  Serial.println(baseThrottle);
}

void decreasePower() {
  if (!isArmed) {
    Serial.println("System must be armed first!");
    return;
  }
  
  baseThrottle -= 50;
  if (baseThrottle < 1100) {
    baseThrottle = 1100;
  }
  
  if (currentMode == DRONE_MODE) {
    updateAllMotors(baseThrottle);
    Serial.print("Flight power decreased to: ");
  } else {
    Serial.print("Ground power decreased to: ");
  }
  Serial.println(baseThrottle);
}

void moveForward() {
  if (!isArmed || currentMode != WHEEL_MODE) {
    Serial.println("Must be armed and in WHEEL MODE for ground movement!");
    return;
  }
  
  // All wheels spin forward
  motor1Speed = baseThrottle;
  motor2Speed = baseThrottle;
  motor3Speed = baseThrottle;
  motor4Speed = baseThrottle;
  Serial.println("Moving forward");
}

void moveBackward() {
  if (!isArmed || currentMode != WHEEL_MODE) {
    Serial.println("Must be armed and in WHEEL MODE for ground movement!");
    return;
  }
  
  // All wheels spin backward (reverse motor direction)
  motor1Speed = 1000 + (1000 - (baseThrottle - 1000)); // Reverse calculation
  motor2Speed = 1000 + (1000 - (baseThrottle - 1000));
  motor3Speed = 1000 + (1000 - (baseThrottle - 1000));
  motor4Speed = 1000 + (1000 - (baseThrottle - 1000));
  Serial.println("Moving backward");
}

void turnLeft() {
  if (!isArmed || currentMode != WHEEL_MODE) {
    Serial.println("Must be armed and in WHEEL MODE for ground movement!");
    return;
  }
  
  // Left wheels slower, right wheels faster
  motor1Speed = baseThrottle - 200;  // Front left
  motor2Speed = baseThrottle + 100;  // Front right
  motor3Speed = baseThrottle - 200;  // Back left
  motor4Speed = baseThrottle + 100;  // Back right
  
  // Ensure speeds stay in valid range
  motor1Speed = constrain(motor1Speed, 1100, 2000);
  motor2Speed = constrain(motor2Speed, 1100, 2000);
  motor3Speed = constrain(motor3Speed, 1100, 2000);
  motor4Speed = constrain(motor4Speed, 1100, 2000);
  
  Serial.println("Turning left");
}

void turnRight() {
  if (!isArmed || currentMode != WHEEL_MODE) {
    Serial.println("Must be armed and in WHEEL MODE for ground movement!");
    return;
  }
  
  // Right wheels slower, left wheels faster
  motor1Speed = baseThrottle + 100;  // Front left
  motor2Speed = baseThrottle - 200;  // Front right
  motor3Speed = baseThrottle + 100;  // Back left
  motor4Speed = baseThrottle - 200;  // Back right
  
  // Ensure speeds stay in valid range
  motor1Speed = constrain(motor1Speed, 1100, 2000);
  motor2Speed = constrain(motor2Speed, 1100, 2000);
  motor3Speed = constrain(motor3Speed, 1100, 2000);
  motor4Speed = constrain(motor4Speed, 1100, 2000);
  
  Serial.println("Turning right");
}

void hover() {
  if (!isArmed || currentMode != DRONE_MODE) {
    Serial.println("Must be armed and in DRONE MODE for hovering!");
    return;
  }
  
  // Calculate hover throttle based on weight
  int hoverThrottle = 1000 + (int)(0.6 * 1000); // Approximately 60% throttle
  baseThrottle = hoverThrottle;
  updateAllMotors(baseThrottle);
  Serial.println("Hovering mode engaged");
}

void stopMovement() {
  if (isArmed) {
    baseThrottle = 1100;  // Armed idle
    updateAllMotors(baseThrottle);
    Serial.println("Movement stopped - Idling");
  } else {
    updateAllMotors(1000);
    Serial.println("System stopped");
  }
}

void updateAllMotors(int throttle) {
  motor1Speed = throttle;
  motor2Speed = throttle;
  motor3Speed = throttle;
  motor4Speed = throttle;
}

void updateSystem() {
  // Send PWM signals to motors
  motor1.writeMicroseconds(motor1Speed);
  motor2.writeMicroseconds(motor2Speed);
  motor3.writeMicroseconds(motor3Speed);
  motor4.writeMicroseconds(motor4Speed);
  
  // Update tilt servo positions if needed
  tilt1.write(tilt1Pos);
  tilt2.write(tilt2Pos);
  tilt3.write(tilt3Pos);
  tilt4.write(tilt4Pos);
}

void displaySystemStatus() {
  static unsigned long lastDisplay = 0;
  if (millis() - lastDisplay > 2000) {  // Display every 2 seconds
    
    Serial.println("=== SYSTEM STATUS ===");
    Serial.print("Mode: ");
    switch (currentMode) {
      case DRONE_MODE:
        Serial.println("FLIGHT (Propellers Vertical)");
        break;
      case WHEEL_MODE:
        Serial.println("GROUND (Propellers Horizontal)");
        break;
      case TRANSITION:
        Serial.println("TRANSITIONING...");
        break;
    }
    
    Serial.print("Armed: ");
    Serial.println(isArmed ? "YES" : "NO");
    
    Serial.print("Base Power: ");
    float powerPercent = (float)(baseThrottle - 1000) / 1000.0 * 100.0;
    Serial.print(powerPercent);
    Serial.println("%");
    
    Serial.print("Motor Speeds: [");
    Serial.print(motor1Speed);
    Serial.print(", ");
    Serial.print(motor2Speed);
    Serial.print(", ");
    Serial.print(motor3Speed);
    Serial.print(", ");
    Serial.print(motor4Speed);
    Serial.println("]");
    
    if (currentMode == WHEEL_MODE && isArmed) {
      float estimatedSpeed = calculateGroundSpeed();
      Serial.print("Est. Ground Speed: ");
      Serial.print(estimatedSpeed);
      Serial.println(" m/s");
    }
    
    Serial.println("==================");
    
    lastDisplay = millis();
  }
}

float calculateGroundSpeed() {
  // Simplified ground speed estimation
  float avgMotorSpeed = (motor1Speed + motor2Speed + motor3Speed + motor4Speed) / 4.0;
  float throttlePercent = (avgMotorSpeed - 1000) / 1000.0;
  return throttlePercent * MAX_GROUND_SPEED;
}

// Emergency stop function
void emergencyStop() {
  motor1.writeMicroseconds(1000);
  motor2.writeMicroseconds(1000);
  motor3.writeMicroseconds(1000);
  motor4.writeMicroseconds(1000);
  baseThrottle = 1000;
  isArmed = false;
  Serial.println("!!! EMERGENCY STOP ACTIVATED !!!");
}