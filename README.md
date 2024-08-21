CODE

TRANSMITTER (AMBULANCE)

#include <RCSwitch.h>

RCSwitch mySwitch1 = RCSwitch();
RCSwitch mySwitch2 = RCSwitch();

const int buttonPin1 = 5; 
const int buttonPin2 = 4; 

void setup() {
  Serial.begin(9600);
  mySwitch1.enableTransmit(14); 
  mySwitch2.enableTransmit(12); 
  pinMode(buttonPin1, INPUT_PULLUP);
  pinMode(buttonPin2, INPUT_PULLUP);
  
  Serial.println("System Initialized");
}

void loop() {
  // Read button states (LOW means pressed due to pull-up resistor)
  int buttonState1 = digitalRead(buttonPin1);
  int buttonState2 = digitalRead(buttonPin2);

  Serial.print("Button 1 State: ");
  Serial.println(buttonState1);
  Serial.print("Button 2 State: ");
  Serial.println(buttonState2);
  if (buttonState1 == LOW) {
    Serial.println("Button 1 Pressed - Sending Signal 1234");
    mySwitch1.send("1234"); 
    delay(2000); 
  }

  if (buttonState2 == LOW) {
    Serial.println("Button 2 Pressed - Sending Signal 5678");
    mySwitch2.send("5678"); 
    delay(2000); 
  }
}


RECEIVER (TRAFFIC SIGNAL)

#define BLYNK_TEMPLATE_ID "XXXXXXXXX"
#define BLYNK_TEMPLATE_NAME "Traffic Light Controller"

#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <RCSwitch.h>

char auth[] = "XXXXXXXXXXXXXXXXXX";
char ssid[] = "XXXXX";
char pass[] = "XXXXX";

RCSwitch mySwitch = RCSwitch(); 

int signal1[] = {D3, D4}; 
int signal2[] = {D5, D6}; 

int triggerpin1 = D1; 
int echopin1 = D2;  
int triggerpin2 = D7; 
int echopin2 = D8; 

int rfReceiverPin = D0; 

int S1, S2; 
int t = 5;  

bool manualControl = false;
bool signal1Red = false;
bool signal1Green = false;
bool signal2Red = false;
bool signal2Green = false;

unsigned long lastChangeTime = 0;
unsigned long greenDuration = 5000; 

bool signal1Priority = true; 

bool emergencySignal1 = false; 
bool emergencySignal2 = false;

void setup()
{
  Serial.begin(115200);
  Blynk.begin(auth, ssid, pass);
  mySwitch.enableReceive(rfReceiverPin); 

  for (int i = 0; i < 2; i++)
  {
    pinMode(signal1[i], OUTPUT);
    pinMode(signal2[i], OUTPUT);
  }

  pinMode(triggerpin1, OUTPUT);
  pinMode(echopin1, INPUT);
  pinMode(triggerpin2, OUTPUT);
  pinMode(echopin2, INPUT);

  lowAll();
  lastChangeTime = millis();
}

void loop()
{
  Blynk.run();
  if (mySwitch.available()) {
    int value = mySwitch.getReceivedValue();
    if (value == 1234) {
      emergencySignal1 = true;
      emergencySignal2 = false;
      handleEmergencySignal1();
    } else if (value == 5678) {
      emergencySignal2 = true;
      emergencySignal1 = false;
      handleEmergencySignal2();
    }
    mySwitch.resetAvailable();
  }
  if (!emergencySignal1 && !emergencySignal2) {
    if (!manualControl) {
      S1 = readDistance(triggerpin1, echopin1);
      S2 = readDistance(triggerpin2, echopin2);

      Serial.print("Distance from signal1: ");
      Serial.print(S1);
      Serial.println(" cm");
      Serial.print("Distance from signal2: ");
      Serial.print(S2);
      Serial.println(" cm");

      handleTrafficSignal();
    } else {
      digitalWrite(signal1[0], signal1Red ? HIGH : LOW);
      digitalWrite(signal1[1], signal1Green ? HIGH : LOW);
      digitalWrite(signal2[0], signal2Red ? HIGH : LOW);
      digitalWrite(signal2[1], signal2Green ? HIGH : LOW);
    }
  } else if (emergencySignal1 && emergencySignal2) {
    resolveEmergencyConflict();
  }

  updateBlynk();
  delay(1000);
}

int readDistance(int triggerPin, int echoPin)
{
  long duration;
  digitalWrite(triggerPin, LOW);
  delayMicroseconds(2);
  digitalWrite(triggerPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(triggerPin, LOW);
  duration = pulseIn(echoPin, HIGH);
  return duration * 0.034 / 2;
}

void handleTrafficSignal()
{
  if (S1 < t && S2 >= t)
  {
    switchToSignal1();
  }
  else if (S2 < t && S1 >= t)
  {
    switchToSignal2();
  }
  else if (S1 < t && S2 < t)
  {
    // Both sides have traffic
    if (millis() - lastChangeTime > greenDuration)
    {
      if (signal1Priority)
      {
        switchToSignal1();
      }
      else
      {
        switchToSignal2();
      }
      signal1Priority = !signal1Priority; 
      lastChangeTime = millis();
    }
  }
  else
  {
    lowAll();
  }
}

void switchToSignal1()
{
  lowAll();
  digitalWrite(signal1[1], HIGH); 
  digitalWrite(signal1[0], LOW); 
  Blynk.virtualWrite(V3, 255);
  Blynk.virtualWrite(V1, 0);
  delay(5000);
}

void switchToSignal2()
{
  lowAll();
  digitalWrite(signal2[1], HIGH); 
  digitalWrite(signal2[0], LOW); 
  Blynk.virtualWrite(V5, 255);
  Blynk.virtualWrite(V4, 0);
  delay(5000);
}

void lowAll()
{
  digitalWrite(signal1[0], HIGH); 
  digitalWrite(signal1[1], LOW); 
  digitalWrite(signal2[0], HIGH); 
  digitalWrite(signal2[1], LOW); 

  Blynk.virtualWrite(V1, 255);
  Blynk.virtualWrite(V3, 0);
  Blynk.virtualWrite(V4, 255);
  Blynk.virtualWrite(V5, 0);
}

void handleEmergencySignal1()
{
  lowAll();
  digitalWrite(signal1[1], HIGH);
  digitalWrite(signal2[0], HIGH); 
  Blynk.virtualWrite(V3, 255);
  Blynk.virtualWrite(V1, 0);
  Blynk.virtualWrite(V4, 255);
  Blynk.virtualWrite(V5, 0);
}

void handleEmergencySignal2()
{
  lowAll();
  digitalWrite(signal2[1], HIGH); 
  digitalWrite(signal1[0], HIGH); 
  Blynk.virtualWrite(V5, 255);
  Blynk.virtualWrite(V4, 0);
  Blynk.virtualWrite(V1, 255);
  Blynk.virtualWrite(V3, 0);
}

void resolveEmergencyConflict()
{
  if (signal1Priority) {
    handleEmergencySignal1();
  } else {
    handleEmergencySignal2();
  }
  signal1Priority = !signal1Priority; 
  lastChangeTime = millis();
}

void updateBlynk()
{
  Blynk.virtualWrite(V1, digitalRead(signal1[0]) == HIGH ? 255 : 0); 
  Blynk.virtualWrite(V3, digitalRead(signal1[1]) == HIGH ? 255 : 0); 

  Blynk.virtualWrite(V4, digitalRead(signal2[0]) == HIGH ? 255 : 0); 
  Blynk.virtualWrite(V5, digitalRead(signal2[1]) == HIGH ? 255 : 0); 
}

BLYNK_WRITE(V0)
{
  manualControl = param.asInt();
  Serial.print("Manual Control: ");
  Serial.println(manualControl);
}

BLYNK_WRITE(V1)
{
  signal1Red = param.asInt();
  Serial.print("Signal 1 Red: ");
  Serial.println(signal1Red);
}

BLYNK_WRITE(V3)
{
  signal1Green = param.asInt();
  Serial.print("Signal 1 Green: ");
  Serial.println(signal1Green);
}

BLYNK_WRITE(V4)
{
  signal2Red = param.asInt();
  Serial.print("Signal 2 Red: ");
  Serial.println(signal2Red);
}

BLYNK_WRITE(V5)
{
  signal2Green = param.asInt();
  Serial.print("Signal 2 Green: ");
  Serial.println(signal2Green);
}



