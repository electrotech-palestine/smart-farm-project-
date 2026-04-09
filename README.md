#include <DHT.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>
#include <time.h>

// --- الإعدادات الأساسية ---
#define WIFI_SSID     "use your own "
#define WIFI_PASS     "use your own "
#define BOT_TOKEN     "use your own :"
#define CHAT_ID       "use your own " 
#define WEATHER_KEY   "use your own "
#define LATITUDE      "use your own "
#define LONGITUDE     "use your own "
#define GROQ_KEY      "use your own "

#define DHT_PIN       4
#define SOIL_PIN      34
#define RELAY_PIN     26
#define DHT_TYPE      DHT22
#define SOIL_DRY      3000
#define SOIL_WET      1000
#define SOIL_THRESH   40
#define AUTO_IRRIGATE_DURATION 60000
#define MAX_LOG       15

DHT dht(DHT_PIN, DHT_TYPE);
WiFiClientSecure client;
WiFiClientSecure groqClient;
WiFiClientSecure telegramFileClient;
UniversalTelegramBot bot(BOT_TOKEN, client);

unsigned long lastBotCheck      = 0;
unsigned long lastSensorRead    = 0;
unsigned long lastAlert         = 0;
unsigned long lastWeatherCheck  = 0;
unsigned long irrigationStarted = 0;
const unsigned long BOT_INTERVAL     = 1000;
const unsigned long SENSOR_INTERVAL  = 30000;
const unsigned long ALERT_INTERVAL   = 300000;
const unsigned long WEATHER_INTERVAL = 3600000;
bool relayState     = false;
bool autoIrrigating = false;
bool rainExpected   = false;
float rainChance    = 0;
String weatherDesc  = "";
float outsideTemp   = 0;

String activityLog[MAX_LOG];
int logIndex = 0;
int logCount = 0;

// --- وظائف الوقت والنظام ---

void initTime() {
  configTime(7200, 0, "pool.ntp.org", "time.nist.gov");
  struct tm timeinfo;
  int attempts = 0;
  while (!getLocalTime(&timeinfo) && attempts < 20) {
    delay(500);
    attempts++;
  }
}

String getTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) return "??:??";
  char buf[20];
  strftime(buf, sizeof(buf), "%d/%m %H:%M", &timeinfo);
  return String(buf);
}

void addLog(String action) {
  String entry = "[" + getTime() + "] " + action;
  activityLog[logIndex] = entry;
  logIndex = (logIndex + 1) % MAX_LOG;
  if (logCount < MAX_LOG) logCount++;
}

String buildLogMessage() {
  if (logCount == 0) return "No activity recorded yet.";
  String msg = "Last " + String(logCount) + " activities:\n--------------------\n";
  int start = (logCount < MAX_LOG) ? 0 : logIndex;
  for (int i = 0; i < logCount; i++) {
    msg += activityLog[(start + i) % MAX_LOG] + "\n";
  }
  return msg;
}

// --- جلب الطقس ---
void fetchWeather() {
  if (WiFi.status() != WL_CONNECTED) return;
  HTTPClient http;
  String url = "http://api.openweathermap.org/data/2.5/forecast?lat=" + String(LATITUDE) + "&lon=" + String(LONGITUDE) + "&appid=" + String(WEATHER_KEY) + "&units=metric&cnt=8";
  http.begin(url);
  if (http.GET() == 200) {
    DynamicJsonDocument doc(8192);
    deserializeJson(doc, http.getString());
    rainExpected = false; rainChance = 0;
    for (int i = 0; i < 8; i++) {
      float pop = doc["list"][i]["pop"].as<float>() * 100;
      if (pop > rainChance) rainChance = pop;
      String desc = doc["list"][i]["weather"][0]["main"].as<String>();
      if (desc == "Rain" || desc == "Drizzle" || desc == "Thunderstorm") rainExpected = true;
    }
    outsideTemp = doc["list"][0]["main"]["temp"].as<float>();
    weatherDesc = doc["list"][0]["weather"][0]["description"].as<String>();
  }
  http.end();
}

int getSoilPercent() {
  return constrain(map(analogRead(SOIL_PIN), SOIL_DRY, SOIL_WET, 0, 100), 0, 100);
}

// --- بناء الرسائل كما في الصورة [cite: 40, 41, 42, 43, 48, 51, 52, 53] ---

String buildStatusMessage() {
  float temp = dht.readTemperature();
  float hum  = dht.readHumidity();
  int   soil = getSoilPercent();
  
  String msg = "Farm Status\n";
  msg += "--------------------\n";
  msg += "Temperature: " + String(temp, 1) + " C\n";
  msg += "Air Humidity: " + String(hum, 1) + " %\n";
  msg += "Soil Moisture: " + String(soil) + "%\n";
  
  if (soil < 20) msg += " -> Very Dry!\n";
  else if (soil < 40) msg += " -> Dry\n";
  else if (soil < 70) msg += " -> Good\n";
  else msg += " -> Very Wet\n";
  
  msg += "--------------------\n";
  msg += "Irrigation: " + String(relayState ? "ON" : "OFF") + "\n";
  msg += "Auto Mode: " + String(autoIrrigating ? "Running" : "Standby") + "\n";
  msg += "Rain expected: " + String(rainExpected ? "YES" : "NO") + "\n";
  msg += "Time: " + getTime();
  
  return msg;
}

String buildWeatherMessage() {
  String msg = "Weather - Nablus\n";
  msg += "--------------------\n";
  msg += "Outside Temp: " + String(outsideTemp, 1) + " C\n";
  msg += "Condition: " + weatherDesc + "\n";
  msg += "Rain chance: " + String(rainChance, 0) + "%\n\n";
  
  if (rainExpected) msg += "Rain expected - Irrigation disabled!";
  else msg += "No rain - Irrigation enabled!";
  
  return msg;
}

// --- التحكم بالري ---
void setRelay(bool state, String chat_id) {
  autoIrrigating = false;
  relayState = state;
  digitalWrite(RELAY_PIN, state ? HIGH : LOW);
  addLog(state ? "Manual ON by " + chat_id : "Manual OFF by " + chat_id);
  bot.sendMessage(chat_id, state ? "Irrigation ON" : "Irrigation OFF", "");
}

void checkAutoAlert() {
  int soil = getSoilPercent();
  unsigned long now = millis();
  if (soil < SOIL_THRESH && !autoIrrigating && (now - lastAlert >= ALERT_INTERVAL)) {
    lastAlert = now;
    if (rainExpected) {
      bot.sendMessage(CHAT_ID, "ALERT: Soil dry! Rain expected - irrigation blocked.", "");
    } else {
      relayState = true;
      autoIrrigating = true;
      irrigationStarted = now;
      digitalWrite(RELAY_PIN, HIGH);
      bot.sendMessage(CHAT_ID, "AUTO: Soil dry! Irrigation ON for 1 min.", "");
    }
  }
}

// --- تحليل الصور والرسائل ---
void analyzePlantImage(String chat_id, String file_url) {
  bot.sendMessage(chat_id, "Analyzing plant image...", "");
  String prompt = "Expert botanist analysis. Be concise.";
  String reqBody = "{\"model\":\"meta-llama/llama-4-scout-17b-16e-instruct\",\"messages\":[{\"role\":\"user\",\"content\":[{\"type\":\"image_url\",\"image_url\":{\"url\":\"" + file_url + "\"}},{\"type\":\"text\",\"text\":\"" + prompt + "\"}]}],\"max_tokens\":500}";

  HTTPClient httpGroq;
  httpGroq.begin(groqClient, "https://api.groq.com/openai/v1/chat/completions");
  httpGroq.addHeader("Content-Type", "application/json");
  httpGroq.addHeader("Authorization", "Bearer " + String(GROQ_KEY));
  int groqCode = httpGroq.POST(reqBody);

  if (groqCode == 200) {
    DynamicJsonDocument resDoc(8192);
    deserializeJson(resDoc, httpGroq.getString());
    String aiResponse = resDoc["choices"][0]["message"]["content"].as<String>();
    bot.sendMessage(chat_id, "Plant Diagnosis:\n" + aiResponse, "");
  }
  httpGroq.end();
}

void checkTelegramMessages() {
  int msgCount = bot.getUpdates(bot.last_message_received + 1);
  while (msgCount) {
    for (int i = 0; i < msgCount; i++) {
      String chat_id  = bot.messages[i].chat_id;
      String text     = bot.messages[i].text;

      if (bot.messages[i].hasDocument || bot.messages[i].type == "photo") {
        analyzePlantImage(chat_id, bot.messages[i].file_path);
      }
      else if (text == "/start" || text == "/help") {
        bot.sendMessage(chat_id, "Welcome! Commands:\n/status\n/weather\n/relayon\n/relayoff\n/log", "");
      }
      else if (text == "/status")   bot.sendMessage(chat_id, buildStatusMessage(), "");
      else if (text == "/weather")  bot.sendMessage(chat_id, buildWeatherMessage(), "");
      else if (text == "/relayon")  setRelay(true, chat_id);
      else if (text == "/relayoff") setRelay(false, chat_id);
      else if (text == "/log")      bot.sendMessage(chat_id, buildLogMessage(), "");
    }
    msgCount = bot.getUpdates(bot.last_message_received + 1);
  }
}

void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);

  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) { delay(500); }
  
  client.setCACert(TELEGRAM_CERTIFICATE_ROOT);
  groqClient.setInsecure();
  telegramFileClient.setInsecure();

  initTime();
  fetchWeather();
  bot.sendMessage(CHAT_ID, "Smart Farm Online!", "");
}

void loop() {
  unsigned long now = millis();
  if (now - lastBotCheck     >= BOT_INTERVAL)     { lastBotCheck = now;     checkTelegramMessages(); }
  if (now - lastSensorRead   >= SENSOR_INTERVAL)  { lastSensorRead = now;   checkAutoAlert(); }
  if (now - lastWeatherCheck >= WEATHER_INTERVAL) { lastWeatherCheck = now; fetchWeather(); }

  if (autoIrrigating && (now - irrigationStarted >= AUTO_IRRIGATE_DURATION)) {
    autoIrrigating = false;
    relayState = false;
    digitalWrite(RELAY_PIN, LOW);
    bot.sendMessage(CHAT_ID, "AUTO: Irrigation finished.", "");
  }
}
