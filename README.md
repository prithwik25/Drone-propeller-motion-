# Drone-propeller-motion-



    #include <Servo.h>

// Create servo objects for each ESC (Electronic Speed Controller)
Servo motor1;  // Front Left
Servo motor2;  // Front Right  
Servo motor3;  // Back Left
Servo motor4;  // Back Right

// Pin definitions
const int MOTOR1_PIN = 3;
const int MOTOR2_PIN = 5;
const int MOTOR3_PIN = 6;
const int MOTOR4_PIN = 9;

// Motor speed variables (1000-2000 microseconds)
int motor1Speed = 1000;
int motor2Speed = 1000;
int motor3Speed = 1000;
int motor4Speed = 1000;

// Throttle and control variables
int baseThrottle = 1000;    // Base throttle (idle)
int maxThrottle = 2000;     // Maximum throttle
int minThrottle = 1000;     // Minimum throttle

// Lift calculation constants
const float DRONE_WEIGHT = 1.5;  // kg
const float GRAVITY = 9.81;      // m/s²
const float THRUST_TO_WEIGHT_RATIO = 2.0;  // Safety margin

void setup() {
  Serial.begin(9600);
  
  // Initialize ESCs
  motor1.attach(MOTOR1_PIN);
  motor2.attach(MOTOR2_PIN);
  motor3.attach(MOTOR3_PIN);
  motor4.attach(MOTOR4_PIN);
  
  // Initialize motors to minimum throttle
  initializeMotors();
  
  Serial.println("Drone Propeller Control System Initialized");
  Serial.println("Commands: 'a' = arm, 'd' = disarm, '+' = increase throttle, '-' = decrease throttle");
  
  delay(2000);
}

void loop() {
  // Check for serial commands
  if (Serial.available() > 0) {
    char command = Serial.read();
    handleCommand(command);
  }
  
  // Update motor speeds
  updateMotors();
  
  // Calculate and display current thrust
  displayThrustInfo();
  
  delay(100);
}

void initializeMotors() {
  // Send minimum throttle to all ESCs for initialization
  motor1.writeMicroseconds(1000);
  motor2.writeMicroseconds(1000);
  motor3.writeMicroseconds(1000);
  motor4.writeMicroseconds(1000);
  
  delay(3000);  // Wait for ESC initialization
  Serial.println("Motors initialized");
}

void handleCommand(char command) {
  switch (command) {
    case 'a':  // Arm motors
      armMotors();
      break;
    case 'd':  // Disarm motors
      disarmMotors();
      break;
    case '+':  // Increase throttle
      increaseThrottle();
      break;
    case '-':  // Decrease throttle
      decreaseThrottle();
      break;
    case 'h':  // Hover throttle
      setHoverThrottle();
      break;
    case 's':  // Stop/idle
      setIdleThrottle();
      break;
  }
}

void armMotors() {
  baseThrottle = 1100;  // Set to armed idle speed
  updateAllMotors(baseThrottle);
  Serial.println("Motors ARMED - Ready for flight");
}

void disarmMotors() {
  baseThrottle = 1000;  // Set to disarmed speed
  updateAllMotors(1000);
  Serial.println("Motors DISARMED - Safe");
}

void increaseThrottle() {
  baseThrottle += 50;
  if (baseThrottle > maxThrottle) {
    baseThrottle = maxThrottle;
  }
  updateAllMotors(baseThrottle);
  Serial.print("Throttle increased to: ");
  Serial.println(baseThrottle);
}

void decreaseThrottle() {
  baseThrottle -= 50;
  if (baseThrottle < minThrottle) {
    baseThrottle = minThrottle;
  }
  updateAllMotors(baseThrottle);
  Serial.print("Throttle decreased to: ");
  Serial.println(baseThrottle);
}

void setHoverThrottle() {
  // Calculate approximate hover throttle based on drone weight
  int hoverThrottle = calculateHoverThrottle();
  baseThrottle = hoverThrottle;
  updateAllMotors(baseThrottle);
  Serial.print("Hover throttle set to: ");
  Serial.println(baseThrottle);
}

void setIdleThrottle() {
  baseThrottle = 1100;  // Safe idle speed
  updateAllMotors(baseThrottle);
  Serial.println("Throttle set to idle");
}

int calculateHoverThrottle() {
  // Simplified hover throttle calculation
  // This would need calibration for your specific drone
  float requiredThrust = DRONE_WEIGHT * GRAVITY;
  float throttlePercent = requiredThrust / (DRONE_WEIGHT * GRAVITY * THRUST_TO_WEIGHT_RATIO);
  int hoverThrottle = 1000 + (int)(throttlePercent * 1000);
  
  // Clamp to valid range
  if (hoverThrottle < 1100) hoverThrottle = 1100;
  if (hoverThrottle > 1800) hoverThrottle = 1800;
  
  return hoverThrottle;
}

void updateAllMotors(int throttle) {
  motor1Speed = throttle;
  motor2Speed = throttle;
  motor3Speed = throttle;
  motor4Speed = throttle;
}

void updateMotors() {
  // Send PWM signals to ESCs
  motor1.writeMicroseconds(motor1Speed);
  motor2.writeMicroseconds(motor2Speed);
  motor3.writeMicroseconds(motor3Speed);
  motor4.writeMicroseconds(motor4Speed);
}

void displayThrustInfo() {
  static unsigned long lastDisplay = 0;
  if (millis() - lastDisplay > 1000) {  // Display every second
    float throttlePercent = (float)(baseThrottle - 1000) / 1000.0 * 100.0;
    float estimatedThrust = calculateEstimatedThrust();
    Serial.print("Throttle: ");
    Serial.print(throttlePercent);
    Serial.print("% | Estimated Thrust: ");
    Serial.print(estimatedThrust);
    Serial.print("N | Status: ");
    
    if (estimatedThrust < DRONE_WEIGHT * GRAVITY * 0.8) {
      Serial.println("INSUFFICIENT LIFT");
    } else if (estimatedThrust >= DRONE_WEIGHT * GRAVITY * 0.8 && 
               estimatedThrust <= DRONE_WEIGHT * GRAVITY * 1.2) {
      Serial.println("HOVER RANGE");
    } else {
      Serial.println("CLIMBING");
    }
    
    lastDisplay = millis();
  }
}

float calculateEstimatedThrust() {
  // Simplified thrust estimation based on throttle
  // This would need calibration with actual thrust measurements
  float throttlePercent = (float)(baseThrottle - 1000) / 1000.0;
  
  // Quadratic relationship approximation for propeller thrust
  float thrustCoefficient = 40.0;  // Adjust based on your propellers
  float estimatedThrust = thrustCoefficient * throttlePercent * throttlePercent;
  
  return estimatedThrust;
}

// Safety function - emergency stop
void emergencyStop() {
  motor1.writeMicroseconds(1000);
  motor2.writeMicroseconds(1000);
  motor3.writeMicroseconds(1000);
  motor4.writeMicroseconds(1000);
  baseThrottle = 1000;
  Serial.println("EMERGENCY STOP ACTIVATED");
}