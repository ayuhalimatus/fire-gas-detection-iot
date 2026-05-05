#include <ESP8266WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <LiquidCrystal_I2C.h>
#include <Wire.h>

// ================= 1. WIFI & TELEGRAM CONFIGURATION =================
#define WIFI_SSID     "YOUR_WIFI_SSID"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"
#define BOT_TOKEN     "YOUR_TELEGRAM_BOT_TOKEN"
#define CHAT_ID       "YOUR_TELEGRAM_CHAT_ID"

// ================= 2. PIN CONFIGURATION =================
// D5 = Fire Sensor
// D6 = Buzzer
// D7 = Gas Sensor (MQ-2)
#define PIN_FIRE_SENSOR  D5
#define PIN_GAS_SENSOR   D7
#define PIN_BUZZER       D6

// Communication Objects
WiFiClientSecure client;
UniversalTelegramBot bot(BOT_TOKEN, client);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Control Variables
int botRequestDelay = 1000;
unsigned long lastTimeBotRan;
unsigned long lastAlertTime = 0;
bool alarmState = false;

// ================= SETUP (Runs Once) =================
void setup() {
  Serial.begin(115200);

  // Set Pin Modes
  pinMode(PIN_FIRE_SENSOR, INPUT);
  pinMode(PIN_GAS_SENSOR, INPUT);
  pinMode(PIN_BUZZER, OUTPUT);
  digitalWrite(PIN_BUZZER, LOW); // Turn off buzzer initially

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("System Starting");

  // Connect to WiFi
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected Successfully!");

  client.setInsecure(); // Required for Telegram

  // Display Ready Status
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("WiFi Connected!");
  lcd.setCursor(0, 1);
  lcd.print("Sensors Ready...");
  delay(2000);
  lcd.clear();

  // Send Initial Telegram Message
  bot.sendMessage(CHAT_ID, "Hello! Your smart fire detector is ready. I am now watching over the area and will alert you immediately if any smoke or fire is detected.", "");
}

// ================= HANDLE INCOMING TELEGRAM MESSAGES =================
void handleNewMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);
    String text = bot.messages[i].text;

    if (chat_id == CHAT_ID) { // Only respond to authorized user

      if (text == "/status") {
        // Read current sensor values
        int fire = digitalRead(PIN_FIRE_SENSOR);
        int gas = digitalRead(PIN_GAS_SENSOR);

        String message = "HOME STATUS REPORT:\n\n";

        // Check Fire Sensor
        if (fire == LOW) message += "FIRE: DETECTED (DANGER!)\n";
        else message += "Fire: Safe.\n";

        // Check Gas Sensor
        if (gas == LOW) message += "GAS/SMOKE: LEAK DETECTED (DANGER!)\n";
        else message += "Gas/Smoke: Safe.\n";

        bot.sendMessage(chat_id, message, "");
      }
      else if (text == "/start") {
        bot.sendMessage(chat_id, "Hello! Fire and Gas Monitoring System is ready.\nType /status to check current conditions.", "");
      }
    }
  }
}

// ================= MAIN LOOP (Runs Continuously) =================
void loop() {
  // 1. Read Sensors (LOW means danger detected)
  int isFire = digitalRead(PIN_FIRE_SENSOR);
  int isGas = digitalRead(PIN_GAS_SENSOR);

  // 2. Alarm Logic
  if (isFire == LOW || isGas == LOW) {
    // === DANGER DETECTED (Fire OR Gas) ===

    digitalWrite(PIN_BUZZER, HIGH); // Activate alarm
    alarmState = true;

    lcd.setCursor(0, 0);
    lcd.print("!! WARNING !!   ");

    // Determine specific alert message
    String alertMessage = "";

    if (isFire == LOW && isGas == LOW) {
      lcd.setCursor(0, 1);
      lcd.print("FIRE & GAS LEAK!");
      alertMessage = "CRITICAL EMERGENCY!\n\nDANGER: Both FIRE and a GAS LEAK have been detected simultaneously!\nThe risk of explosion is high. Evacuate immediately!";
    }
    else if (isFire == LOW) {
      lcd.setCursor(0, 1);
      lcd.print("FIRE DETECTED!! ");
      alertMessage = "DANGER: FIRE DETECTED IN THE HOUSE!\nThe sensor has picked up a flame signal. Please check the location immediately!";
    }
    else if (isGas == LOW) {
      lcd.setCursor(0, 1);
      lcd.print("GAS LEAK !!!    ");
      alertMessage = "DANGER: LPG GAS LEAK / SMOKE DETECTED!\nPlease check the ventilation and the source immediately!";
    }

    // Send Telegram notification (every 5 seconds to prevent overload)
    if (millis() - lastAlertTime > 5000) {
      bot.sendMessage(CHAT_ID, alertMessage, "");
      lastAlertTime = millis();
    }
  }
  else {
    // === SAFE CONDITIONS ===
    digitalWrite(PIN_BUZZER, LOW); // Deactivate alarm

    // Notify if conditions have returned to normal
    if (alarmState == true) {
      lcd.clear();
      alarmState = false;
      bot.sendMessage(CHAT_ID, "INFO: The danger has passed. Conditions have returned to NORMAL.", "");
    }

    // Standby Display
    lcd.setCursor(0, 0);
    lcd.print("Fire: SAFE      ");
    lcd.setCursor(0, 1);
    lcd.print("Gas : SAFE      ");
  }

  // 3. Check Incoming Telegram Messages
  if (millis() > lastTimeBotRan + botRequestDelay) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }
    lastTimeBotRan = millis();
  }
}
