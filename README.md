
#include "DxlMaster.h"

#include <TrackingCamDxlUart.h>

TrackingCamDxlUart trackingCam;
unsigned long previousMillis = 0; // stores last time cam was updated

DynamixelMotor motor(1);

#define M1_dir 45
#define M1_Speed 44

#define start_button_pin 35 // кнопка паузы

#define echoPin1 6  //ультразвук 1
#define trigPin1 7

#define echoPin2 4  //ультразвук 2
#define trigPin2 5

bool start_tr = true;

bool count = true;

int chetchik;

uint8_t n;

int duration1, cml, duration2, cmr;     // инициализация переменных расстояния 
int cm0_left, cm0_right;

int a0 = 808;     // угол альфа 0

int k = -1;     // коэфицент направленности

int c = false;

int chetchik_vremeni = 0;      // счетчик времени(расстояния) в начале

void moving(bool a, int times, int delta_alfa, int napravlenie, bool b) {
  if (a) {
    digitalWrite(M1_dir, b);
    analogWrite(M1_Speed, 150);
    motor.goalPosition(a0 + delta_alfa*napravlenie);      // поворот руля
    delay(times);
    analogWrite(M1_Speed, 0);
  }
  else {
    analogWrite(M1_Speed, 0);
  }
}

void ultra_wave1() {
  digitalWrite(trigPin1, LOW);       // волна ультразвука 1
  delayMicroseconds(2);
  digitalWrite(trigPin1, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin1, LOW);

  duration1 = pulseIn(echoPin1, HIGH);      // расстояние 1
  cml = duration1/58;     // расстояние в см 1
}

void ultra_wave2() {
  digitalWrite(trigPin2, LOW);       // волна ультразвука 1
  delayMicroseconds(2);
  digitalWrite(trigPin2, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin2, LOW);
  
  duration2 = pulseIn(echoPin2, HIGH);      // расстояние 2
  cmr = duration2/58;     // расстояние в см 2
}

void setup() {
  
  DxlMaster.begin(1000000);

  motor.enableTorque();
  motor.jointMode(500, 1000);
  motor.speed(100);

  pinMode(M1_dir, OUTPUT);
  pinMode(M1_Speed, OUTPUT);

  Serial.begin(9600);

  pinMode(start_button_pin, INPUT);
  
  pinMode(trigPin1, OUTPUT);     // ультразвук 1
  pinMode(echoPin1, INPUT);

  pinMode(trigPin2, OUTPUT);     // ултразвук 2
  pinMode(echoPin2, INPUT);

  
  ultra_wave1();      // волна ультразвука 1

  ultra_wave2();      // волна ультразвука 2
  
  

  int cm0_left = cml;
  int cm0_right = cmr;

  Serial.print(motor.currentPosition());
  while (digitalRead(start_button_pin) == HIGH) {

  }

  trackingCam.init(51, 3, 115200, 30);
  Serial.begin(9600);
  delay(5000);
}

void loop() {

  if (start_tr) {
    start();  
  }

  while (chetchik < 12) {
    osnova();
    chetchik += 1;
  }

  if (k == 1) {
    
  }
  
}

void start() {
  ultra_wave1();
  ultra_wave2();

  if (cml - cmr > 20) {
    k = 1;
  }
  
  Serial.print(cmr);
  Serial.print(" ");
  Serial.print(cml);
  Serial.print(" ");
  if (abs(cml - cmr) < 20) {
    k = 1;
  }
  else {
    moving(count, 1000, 50, 1, LOW);
    uint8_t n;
    n = trackingCam.readBlobs(); // read data about first 5 blobs
    Serial.println(n); // print numbers of blobs 
    
    if (n > 0 and trackingCam.blob[0].type == 0 and k == 1) {
      moving(count, 720, 150, -1, HIGH);
      moving(count, 100, 20, 1, HIGH);
      moving(count, 900, 150, 1, HIGH);
      moving(count, 700, 0, 1, HIGH);      
      //moving(count, 900, 50, 1, HIGH);
    }
    else if (n > 0 and trackingCam.blob[0].type == 1 and k == -1) {
      moving(count, 720, 150, -1, HIGH);
      moving(count, 100, 20, 1, HIGH);
      moving(count, 900, 150, 1, HIGH);
      moving(count, 1050, 50, 1, HIGH);
    }
    else {
      moving(count, 800, 150, -1, HIGH);
      moving(count, 100, 20, 1, HIGH);
      moving(count, 1100, 150, 1, HIGH);
      moving(count, 800, 100, -1, HIGH);
      motor.goalPosition(a0);
    }
  }

  

  ultra_wave1();      // волна ультразвука 1
  ultra_wave2();      // волна ультразвука 2
  

  while (cml + cmr < 90) {
    ultra_wave1();      // волна ультразвука 1
    ultra_wave2();      // волна ультразвука 2
    if (cmr > (cmr+cml)/2+2) {
      moving(count, 200,  30, 1, HIGH);
    }
    if (cml > (cmr+cml)/2+2) {
      moving(count, 200, 30, -1, HIGH);
    }
    moving(count, 100, 0, 1, HIGH);
    n = trackingCam.readBlobs();
    if (n) {
      if (trackingCam.blob[0].type == 0 and trackingCam.blob[0].area > 3000) {
        moving(count, 400, 100, 1, HIGH);
        moving(count, 780, 100, -1, HIGH);
      }
      if (trackingCam.blob[0].type == 1 and trackingCam.blob[0].area > 3000) {
        moving(count, 400, 100, -1, HIGH);
        moving(count, 930, 100, 1, HIGH);
      }
    }
    chetchik_vremeni += 1;
  }

  if (cml > 70) {
    moving(count, 400, 0, -1, HIGH);
    moving(count, 900, 200, -1, HIGH);
    moving(count, 1000, 0, -1, HIGH);
  }
  if (cmr > 70) {
    moving(count, 400, 0, -1, HIGH);
    moving(count, 900, 200, 1, HIGH);
    moving(count, 1000, 0, -1, HIGH);
  }
  
  
  analogWrite(M1_Speed, 0);
  
  start_tr = false;
}



void osnova() {
  ultra_wave1();      // волна ультразвука 1
  ultra_wave2();      // волна ультразвука 2

  while (cml + cmr < 90) {
    ultra_wave1();      // волна ультразвука 1
    ultra_wave2();      // волна ультразвука 2
    if (cmr > (cmr+cml)/2+2) {
      moving(count, 200,  30, 1, HIGH);
    }
    if (cml > (cmr+cml)/2+2) {
      moving(count, 200, 30, -1, HIGH);
    }
    moving(count, 100, 0, 1, HIGH);
    n = trackingCam.readBlobs();
    if (n) {
      if (trackingCam.blob[0].type == 0 and trackingCam.blob[0].area > 3000) {
        moving(count, 400, 100, 1, HIGH);
        moving(count, 1000, 100, -1, HIGH);
        moving(count, 1100, 50, 1, HIGH);
      }
      if (trackingCam.blob[0].type == 1 and trackingCam.blob[0].area > 3000) {
        moving(count, 400, 100, -1, HIGH);
        moving(count, 930, 100, 1, HIGH);
        moving(count, 1100, 50, -1, HIGH);
      }
      if (cml > 70) {
        moving(count, 50, 0, -1, HIGH);
        moving(count, 650, 200, -1, HIGH);
        c = false;
      }
      if (cmr > 70) {
        moving(count, 900, 200, 1, HIGH);
        moving(count, 500, 0, -1, HIGH);
        moving(count, 600, 0, -1, LOW);
        c = false;
      }
    }
  }
  analogWrite(M1_Speed, 0);

  ultra_wave1();      // волна ультразвука 1
  ultra_wave2();      // волна ультразвука 2

  
  
  if (cml > 70 and c) {
    moving(count, 500, 0, -1, HIGH);
    moving(count, 1000, 200, -1, HIGH);
    moving(count, 1000, 0, -1, HIGH);
  }
  if (cmr > 70 and c) {
    moving(count, 500, 0, -1, HIGH);
    moving(count, 1000, 200, 1, HIGH);
    moving(count, 1000, 0, -1, HIGH);
  }
  
  c = true;
  
  n = trackingCam.readBlobs();
  if (n) {
    if (trackingCam.blob[0].type == 0 and trackingCam.blob[0].area > 3000) {
      moving(count, 400, 0, -1, HIGH);
      moving(count, 900, 200, -1, HIGH);
      moving(count, 1000, 0, -1, HIGH);
    }
    if (trackingCam.blob[0].type == 1 and trackingCam.blob[0].area > 3000) {
      moving(count, 400, 0, -1, HIGH);
      moving(count, 900, 200, 1, HIGH);
      moving(count, 1000, 0, -1, HIGH);
    }
  }
  else {
    moving(count, 1000, 0, -1, HIGH);
  }
}
