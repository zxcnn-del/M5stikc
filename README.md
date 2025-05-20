# M5stikc
I can't find the error




#include <dummy.h>
#include <M5StickCPlus2.h>

#define MAX_DEVIATION 0.15    // Предельное отклонение в градусах
#define SAMPLE_SIZE 5         // Размер буфера для усреднения
#define IN_RANGE_COLOR GREEN  // Цвет в допуске
#define OUT_OF_RANGE_COLOR RED // Цвет вне допуска

float calibrationX = 0;
float calibrationY = 0;
float rollBuffer[SAMPLE_SIZE] = {0};
float pitchBuffer[SAMPLE_SIZE] = {0};
int bufferIndex = 0;
bool calibrated = false;

void calibrateLevel() {
  float sumX = 0, sumY = 0;
  for(int i=0; i<100; i++) {
    float ax, ay, az;
    M5.Imu.getAccelData(&ax, &ay, &az);
    sumX += ax;
    sumY += ay;
    delay(10);
  }
  calibrationX = sumX/100;
  calibrationY = sumY/100;
  calibrated = true;
}

float getFilteredRoll() {
  float ax, ay, az;
  M5.Imu.getAccelData(&ax, &ay, &az);
  ax -= calibrationX;
  ay -= calibrationY;
  
  // Добавляем новое значение в буфер
  rollBuffer[bufferIndex] = atan2(ay, az) * 180 / PI;
  float avg = 0;
  for(int i=0; i<SAMPLE_SIZE; i++) avg += rollBuffer[i];
  return avg / SAMPLE_SIZE;
}

float getFilteredPitch() {
  float ax, ay, az;
  M5.Imu.getAccelData(&ax, &ay, &az);
  ax -= calibrationX;
  ay -= calibrationY;
  
  // Добавляем новое значение в буфер
  pitchBuffer[bufferIndex] = atan2(-ax, sqrt(ay*ay + az*az)) * 180 / PI;
  float avg = 0;
  for(int i=0; i<SAMPLE_SIZE; i++) avg += pitchBuffer[i];
  return avg / SAMPLE_SIZE;
}

void drawLevel(float roll, float pitch) {
  M5.Lcd.fillScreen(BLACK);
  M5.Lcd.setTextSize(2);
  
  // Проверка на соответствие допуску
  bool inRange = (abs(roll) <= MAX_DEVIATION) && (abs(pitch) <= MAX_DEVIATION);
  uint16_t bubbleColor = inRange ? IN_RANGE_COLOR : OUT_OF_RANGE_COLOR;
  
  // Отрисовка сетки
  int centerX = M5.Lcd.width()/2;
  int centerY = M5.Lcd.height()/2;
  
  // Горизонтальная и вертикальная линии
  M5.Lcd.drawLine(0, centerY, M5.Lcd.width(), centerY, WHITE);
  M5.Lcd.drawLine(centerX, 0, centerX, M5.Lcd.height(), WHITE);
  
  // Зона допуска (прозрачный прямоугольник)
  int toleranceZone = 10; // 10 пикселей от центра
  M5.Lcd.drawRect(centerX - toleranceZone, 
                centerY - toleranceZone, 
                toleranceZone*2, 
                toleranceZone*2, 
                inRange ? GREEN : DARKGREY);
  
  // Пузырек уровня
  int bubbleX = centerX + map(pitch, -15, 15, -50, 50);
  int bubbleY = centerY + map(roll, -15, 15, -50, 50);
  bubbleX = constrain(bubbleX, 10, M5.Lcd.width()-10);
  bubbleY = constrain(bubbleY, 10, M5.Lcd.height()-10);
  
  M5.Lcd.fillCircle(bubbleX, bubbleY, 10, bubbleColor);
  
  // Отображение углов с цветовой индикацией
  M5.Lcd.setCursor(10, 10);
  M5.Lcd.setTextColor(abs(roll) <= MAX_DEVIATION ? GREEN : RED);
  M5.Lcd.printf("Roll: %5.1f°", roll);
  
  M5.Lcd.setCursor(10, 40);
  M5.Lcd.setTextColor(abs(pitch) <= MAX_DEVIATION ? GREEN : RED);
  M5.Lcd.printf("Pitch:%5.1f°", pitch);
  
  // Статус уровня
  M5.Lcd.setTextSize(1);
  M5.Lcd.setCursor(centerX - 30, M5.Lcd.height() - 25);
  M5.Lcd.setTextColor(inRange ? GREEN : RED, BLACK);
  M5.Lcd.printf(inRange ? "[  LEVEL  ]" : "[ADJUST]");
}

void setup() {
  M5.begin();
  M5.Lcd.setRotation(1);
  M5.Imu.Init();
  
  M5.Lcd.setTextColor(WHITE);
  M5.Lcd.setTextSize(1);
  M5.Lcd.println("Calibrating... Keep device flat");
  calibrateLevel();
  
  // Инициализация буферов
  for(int i=0; i<SAMPLE_SIZE; i++) {
    rollBuffer[i] = getFilteredRoll();
    pitchBuffer[i] = getFilteredPitch();
  }
}

void loop() {
  M5.update();
  
  if(M5.BtnA.wasPressed()) {
    calibrateLevel();
  }
  
  bufferIndex = (bufferIndex + 1) % SAMPLE_SIZE;
  float roll = getFilteredRoll();
  float pitch = getFilteredPitch();
  
  drawLevel(roll, pitch);
  delay(50);
}

