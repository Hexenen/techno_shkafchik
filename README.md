#include <SPI.h>
#include <MFRC522.h>
#include <ESP32Servo.h>  // Используем ESP32Servo вместо Servo
#include <EEPROM.h>

// Пины для RFID - ESP32
#define RST_PIN 4     // GPIO 4
#define SS_PIN 5      // GPIO 5

// Пины для RGB светодиода (общий катод)
#define RED_PIN 25    // GPIO 25
#define GREEN_PIN 26  // GPIO 26
#define BLUE_PIN 27   // GPIO 27

// Пин для кнопки
#define BUTTON_PIN 32 // GPIO 32

// Пины для сервоприводов
#define SERVO1_PIN 13 // GPIO 13
#define SERVO2_PIN 14 // GPIO 14

// Пин для бузера (если вернете)
#define BUZZER_PIN 15 // GPIO 15

// Адреса в EEPROM
#define CARD_COUNT_ADDR 0
#define FIRST_CARD_ADDR 10
#define EEPROM_SIZE 512  // Размер EEPROM для ESP32

MFRC522 mfrc522(SS_PIN, RST_PIN);
Servo servo1;
Servo servo2;

bool lockerOpen = false;
bool learningMode = false;
bool lastButtonState = HIGH;
bool buttonPressed = false;
unsigned long buttonPressStartTime = 0;
unsigned long learningModeStartTime = 0;
const unsigned long LEARNING_TIMEOUT = 10000;

// Переменные для антиспам RFID
unsigned long lastRFIDCheck = 0;
const unsigned long RFID_CHECK_INTERVAL = 500;

void setup() {
  Serial.begin(115200);  // ESP32 использует 115200
  Serial.println(F("\n=== УМНЫЙ ШКАФЧИК на ESP32 ==="));
  
  // Инициализация EEPROM для ESP32
  EEPROM.begin(EEPROM_SIZE);
  
  SPI.begin(18, 19, 23, 5); // SCK, MISO, MOSI, SS для ESP32
  
  // Инициализация RFID
  mfrc522.PCD_Init();
  delay(4);
  
  // Вывод версии RFID модуля
  Serial.print(F("RFID Module: "));
  mfrc522.PCD_DumpVersionToSerial();
  
  // Настройка светодиода
  pinMode(RED_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BLUE_PIN, OUTPUT);
  
  // Настройка кнопки
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  // Разрешить ESP32 использовать все таймеры для сервоприводов
  ESP32PWM::allocateTimer(0);
  ESP32PWM::allocateTimer(1);
  ESP32PWM::allocateTimer(2);
  ESP32PWM::allocateTimer(3);
  
  // Подключение сервоприводов
  servo1.setPeriodHertz(50);      // Стандартная частота 50Hz для сервоприводов
  servo1.attach(SERVO1_PIN, 500, 2400); // Микросекунды для 0-180 градусов
  
  servo2.setPeriodHertz(50);
  servo2.attach(SERVO2_PIN, 500, 2400);
  
  // Закрыть шкафчик
  closeLocker();
  
  // Инициализация EEPROM если нужно
  if (EEPROM.read(CARD_COUNT_ADDR) == 255) {
    EEPROM.write(CARD_COUNT_ADDR, 0);
    EEPROM.commit();
    Serial.println(F("EEPROM инициализирована"));
  }
  
  printCardCount();
  setColor(0, 0, 255); // Синий
  
  // Тестовый сигнал светом (без звука)
  Serial.println(F("Тест системы:"));
  delay(500);
  setColor(255, 0, 0); delay(300);  // Красный
  setColor(0, 255, 0); delay(300);  // Зеленый
  setColor(0, 0, 255); delay(300);  // Синий
  
  Serial.println(F("\n=== СИСТЕМА ГОТОВА ==="));
  Serial.println(F("1. Приложите карту для открытия"));
  Serial.println(F("2. Удерживайте кнопку 3 сек для записи карты"));
  Serial.println(F("=========================================="));
}

void loop() {
  // Обработка кнопки
  handleButton();
  
  // Обработка режима обучения
  if (learningMode) {
    handleLearningMode();
  }
  
  // Проверка RFID карты (с антиспамом)
  if (millis() - lastRFIDCheck > RFID_CHECK_INTERVAL) {
    checkRFID();
    lastRFIDCheck = millis();
  }
  
  // Небольшая задержка для стабильности
  delay(10);
}

// Проверка RFID карты
void checkRFID() {
  // Сбрасываем состояние ридера
  mfrc522.PICC_HaltA();
  mfrc522.PCD_StopCrypto1();
  delay(50);
  
  // Проверяем новую карту
  if (!mfrc522.PICC_IsNewCardPresent()) {
    return;
  }
  
  // Читаем карту
  if (!mfrc522.PICC_ReadCardSerial()) {
    return;
  }
  
  Serial.print(F("Карта обнаружена! UID: "));
  dump_byte_array(mfrc522.uid.uidByte, mfrc522.uid.size);
  Serial.println();
  
  if (learningMode) {
    // Режим записи
    saveCardToEEPROM(mfrc522.uid.uidByte, mfrc522.uid.size);
  } else {
    // Обычный режим
    bool accessGranted = checkCardInEEPROM(mfrc522.uid.uidByte, mfrc522.uid.size);
    
    if (accessGranted) {
      Serial.println(F("✓ Доступ разрешен!"));
      visualBeep(1, 200);  // Визуальная индикация вместо звука
      
      if (lockerOpen) {
        closeLocker();
      } else {
        openLocker();
      }
    } else {
      Serial.println(F("✗ Доступ запрещен!"));
      setColor(255, 0, 0); // Красный
      visualBeep(3, 100);
      delay(2000);
      if (!lockerOpen) setColor(0, 0, 255); // Возвращаем синий
    }
  }
  
  // Останавливаем чтение
  mfrc522.PICC_HaltA();
  mfrc522.PCD_StopCrypto1();
}

// Визуальный сигнал (мигание светом вместо звука)
void visualBeep(int count, int duration) {
  for (int i = 0; i < count; i++) {
    // Мигаем белым светом
    setColor(255, 255, 255);
    delay(duration);
    // Возвращаем нормальный цвет
    if (lockerOpen) {
      setColor(0, 255, 0); // Зеленый если открыто
    } else {
      setColor(0, 0, 255); // Синий если закрыто
    }
    if (i < count - 1) delay(100);
  }
}

// Обработка кнопки
void handleButton() {
  int reading = digitalRead(BUTTON_PIN);
  
  if (reading == LOW && !buttonPressed) {
    buttonPressed = true;
    buttonPressStartTime = millis();
    Serial.println(F("Кнопка нажата"));
  }
  
  if (reading == HIGH && buttonPressed) {
    buttonPressed = false;
    
    unsigned long pressDuration = millis() - buttonPressStartTime;
    
    if (pressDuration >= 3000) {
      if (!learningMode) {
        enterLearningMode();
      } else {
        exitLearningMode();
      }
    } else if (pressDuration > 50 && pressDuration < 1000) {
      Serial.println(F("Короткое нажатие кнопки"));
      visualBeep(1, 50);
      if (lockerOpen) {
        closeLocker();
      } else {
        openLocker();
      }
    }
  }
  
  if (buttonPressed && !learningMode) {
    unsigned long currentTime = millis();
    if (currentTime - buttonPressStartTime >= 2900 && 
        currentTime - buttonPressStartTime < 3000) {
      visualBeep(1, 50);
    }
  }
}

// Вход в режим обучения
void enterLearningMode() {
  learningMode = true;
  learningModeStartTime = millis();
  setColor(255, 255, 0); // Желтый
  visualBeep(2, 200);
  Serial.println(F("\n=== РЕЖИМ ЗАПИСИ КАРТЫ ==="));
  Serial.println(F("Приложите новую карту для записи"));
  Serial.println(F("Режим автоматически отключится через 10 секунд"));
}

// Обработка режима обучения
void handleLearningMode() {
  // Мигание желтым
  unsigned long currentTime = millis();
  if ((currentTime / 500) % 2 == 0) {
    setColor(255, 255, 0); // Яркий желтый
  } else {
    setColor(150, 150, 0); // Тусклый желтый
  }
  
  // Проверка таймаута
  if (currentTime - learningModeStartTime > LEARNING_TIMEOUT) {
    Serial.println(F("Таймаут! Режим записи отключен."));
    visualBeep(3, 100);
    exitLearningMode();
  }
}

// Выход из режима обучения
void exitLearningMode() {
  learningMode = false;
  if (lockerOpen) {
    setColor(0, 255, 0); // Зеленый
  } else {
    setColor(0, 0, 255); // Синий
  }
  Serial.println(F("Режим записи отключен"));
}

// Сохранение карты в EEPROM
void saveCardToEEPROM(byte* uid, byte size) {
  if (size != 4) {
    Serial.println(F("Ошибка: поддерживаются только 4-байтные UID"));
    return;
  }
  
  byte cardCount = EEPROM.read(CARD_COUNT_ADDR);
  
  // Проверка на дубликат
  for (byte i = 0; i < cardCount; i++) {
    int addr = FIRST_CARD_ADDR + i * 5;
    bool duplicate = true;
    for (byte j = 0; j < 4; j++) {
      if (EEPROM.read(addr + j) != uid[j]) {
        duplicate = false;
        break;
      }
    }
    if (duplicate) {
      Serial.println(F("Эта карта уже записана!"));
      visualBeep(3, 100);
      return;
    }
  }
  
  // Сохранение карты
  int addr = FIRST_CARD_ADDR + cardCount * 5;
  for (byte i = 0; i < 4; i++) {
    EEPROM.write(addr + i, uid[i]);
  }
  EEPROM.write(addr + 4, 1); // Статус активна
  
  cardCount++;
  EEPROM.write(CARD_COUNT_ADDR, cardCount);
  
  // Сохраняем в EEPROM (важно для ESP32!)
  EEPROM.commit();
  
  Serial.print(F("Карта успешно записана! UID: "));
  dump_byte_array(uid, size);
  Serial.println();
  Serial.print(F("Всего карт: "));
  Serial.println(cardCount);
  
  visualBeep(1, 500); // Длинный визуальный сигнал успеха
  exitLearningMode();
}

// Проверка карты в EEPROM
bool checkCardInEEPROM(byte* uid, byte size) {
  if (size != 4) return false;
  
  byte cardCount = EEPROM.read(CARD_COUNT_ADDR);
  
  for (byte i = 0; i < cardCount; i++) {
    int addr = FIRST_CARD_ADDR + i * 5;
    bool match = true;
    
    for (byte j = 0; j < 4; j++) {
      if (EEPROM.read(addr + j) != uid[j]) {
        match = false;
        break;
      }
    }
    
    if (match && EEPROM.read(addr + 4) == 1) {
      return true;
    }
  }
  
  return false;
}

// Открыть шкафчик
void openLocker() {
  lockerOpen = true;
  servo1.write(90);
  servo2.write(90);
  setColor(0, 255, 0); // Зеленый
  visualBeep(2, 150);
  Serial.println(F(">>> Шкафчик ОТКРЫТ"));
}

// Закрыть шкафчик
void closeLocker() {
  lockerOpen = false;
  servo1.write(0);
  servo2.write(0);
  setColor(0, 0, 255); // Синий
  visualBeep(1, 200);
  Serial.println(F("<<< Шкафчик ЗАКРЫТ"));
}

// Управление RGB светодиодом (общий катод)
void setColor(int red, int green, int blue) {
  analogWrite(RED_PIN, 255 - red);
  analogWrite(GREEN_PIN, 255 - green);
  analogWrite(BLUE_PIN, 255 - blue);
}

// Вывод количества карт
void printCardCount() {
  byte cardCount = EEPROM.read(CARD_COUNT_ADDR);
  Serial.print(F("Зарегистрировано карт: "));
  Serial.println(cardCount);
}

// Вывод UID карты в Serial
void dump_byte_array(byte *buffer, byte bufferSize) {
  for (byte i = 0; i < bufferSize; i++) {
    Serial.print(buffer[i] < 0x10 ? " 0" : " ");
    Serial.print(buffer[i], HEX);
  }
}