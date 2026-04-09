0. Created by Electro-Tech
1. The Power Architecture (Electrical Design)
Managing multiple voltages is the most critical part of this "Electro-Tech" build.

The 12V High-Power Rail: This powers the water pump. It must remain isolated from the ESP32 to prevent electrical noise or surges from destroying the microcontroller.

The 5V/3.3V Logic Rail: The ESP32 runs on 3.3V. The relay module usually requires 5V to trigger the internal coil. Using a common ground (GND) across all power supplies is essential for the signal to travel correctly.

The Relay Isolation: The relay acts as a "bridge." When the ESP32 sends a signal to Pin 26, the relay closes an internal switch, allowing the 12V current to reach the pump.

2. Physical Sensor Integration
DHT22 (Atmospheric Sensing): Unlike the cheaper DHT11, the DHT22 provides decimal-point accuracy for temperature and humidity. The code uses the DHT.h library to parse the digital signal coming into GPIO 4.

Capacitive Soil Moisture Sensor: This sensor measures the dielectric constant of the soil. In the code, map() and constrain() functions translate raw analog values (which range from 0 to 4095 on the ESP32) into a user-friendly 0% to 100% scale.

Calibration: The SOIL_DRY and SOIL_WET constants are vital. You must test the sensor in a cup of water and in bone-dry soil to set these numbers accurately for your specific sensor.

3. Software Logic & Decision Making
The code is designed to be "Proactive" rather than "Reactive."

The Weather Filter: The fetchWeather() function calls the OpenWeatherMap API using your latitude and longitude. It looks at the pop (Probability of Precipitation). If the chance of rain is high, the system enters a "Protection Mode," refusing to pump water even if the soil is dry, because it knows nature will do the job for free.

The Non-Blocking Loop: Notice the use of millis() instead of delay(). This allows the ESP32 to check for Telegram messages every second while simultaneously monitoring sensor data and managing irrigation timers without ever "freezing."

The Security Layer: The WiFiClientSecure library is used because both Telegram and the Groq AI API require encrypted HTTPS connections. This ensures your control commands cannot be intercepted on your local network.

4. AI Vision & Telegram Interaction
This is where the project moves beyond a standard Arduino build into the realm of "Smart Technology."

Image Processing: When you send a photo of your plant (e.g., a tomato leaf with spots), the Telegram bot captures the file_path.

LLM Integration: The code sends this image URL to the Groq API (Meta-Llama-4-Scout). The AI is prompted to act as an "Expert Botanist." It analyzes the pixels to identify the plant species and suggest if the plant needs more nitrogen, has a fungal infection, or requires a different watering schedule.

Remote Control: The bot acts as a global remote. Whether you are in the next room or another country, you can send /relayon to start the pump or /status to see exactly what the sensors are seeing in real-time.

5. Maintenance and Logging
Activity Ledger: The activityLog is a circular buffer. It records every event (Pump Start, Rain Alert, Manual Override). This is stored in the ESP32's RAM, providing a history you can call up via the /log command to see if the system has been working correctly while you were away.

Error Handling: The initTime() function ensures the ESP32 has the correct local time from an NTP server. This is necessary for the logs to show the correct hour and minute of each event.

By following these points, you create a robust, data-informed ecosystem that protects your plants while conserving water resources efficiently.


Created by Electro-Tech
