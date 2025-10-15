+++
title = "Retrospective: GDOOR"
date = 2025-10-09

emoji = "⚙️💻📟"
banner_c = "Device diagram listing the used components and the layout."

tags = ["programming", "embedded", "cxx", "oop"]
draft = false
+++

The point of this small embedded project was to get familiar with object-oriented programming (oop)
in the context of embedded devices. Before this I'd only used basic Micropython or C for embedded
programming, so I spent most of the effort from my two brain cells on trying to designing sensible
interfaces and inheritance for the components present in the system.

I also did some more reading on cmake build systems and how to better configure them, which directly
affected the general project source structure. Overall, I'm quite happy with how it turned out, but there were
some issues with passing the correct board type to pico-sdk that I didn't have time to fully resolve.
Something to do with my cmake fragments and their active scope, I suspect...

## Source Structure

```
GDOOR
├── module
│   ├── paho-mqtt
│   └── pico-sdk
├── include
│   ├── components.hpp
│   ├── config.hpp
│   ├── gdoor.hpp
└── src
    ├── components.cpp
    ├── gdoor.cpp
    └── main.cpp
```

Above is the relevant source structure for the code in this project. I split the SDKs and other
external modules into a separate *module* directory, to leave the *src* (implementations), and
*include* (definitions) clean for project-specific code only.

Beginning from the `main.cpp`, we simply instantiate the device object and call the statemachine-style
member function on an endless loop.

```c++
#include <stdio.h>
#include "gdoor.hpp"

int main() {
	stdio_init_all();
	printf("\n<BOOT>\n");
	
	GarageDoor gdoor;
	while (true) gdoor.run();
	
	return 0;
}
```

## Definitions

![GDOOR and Component interfaces](class-overview.png)

### GarageDoor

Next would be appropriate to go over the definition of the machine itself, as well as the
namespaces that are used for configuration and tuning of the device and its components.

Here's the `gdoor.hpp` which defines the device interfaces and data fields:

```c++
#pragma once

#include "components.hpp"
#include "config.hpp"

class GarageDoor {
	private:
		// Components
		Components::StepperMotor motor;
		Components::Button button_a, button_b;
		Components::Limit floor, ceil;
		Components::RotaryEncoder rot;
		Components::Eeprom storage;
		Components::WifiMQTT remote_control;
		Components::Led led1;
		Components::Led led2;
		Components::Led led3;
		
		// States and Data fields
		uint8_t magic = Config::GDOOR_MAGIC;
		Config::GDOOR_STATE state = Config::GDOOR_STATE::UNKNOWN;
		Config::GDOOR_STATE state_prev = Config::GDOOR_STATE::UNKNOWN;
		Config::GDOOR_MOV movement = Config::GDOOR_MOV::NONE;
		uint32_t last_input = 0;
		uint8_t stuck_counter = 0;
		bool calibrated = false;
		uint8_t calib_steps = 0;
		uint8_t current_step = 0;
		bool remote_needs_informing = false;
	public:
		// Constructor
		GarageDoor();
		
		// Member functions
		bool at_floor();
		bool at_ceil();
		bool check_stuck(int rotary_initial_value);
		
		Config::GDOOR_STATE close(bool in_calib);
		Config::GDOOR_STATE open(bool in_calib);
		Config::GDOOR_STATE calibrate();
		Config::GDOOR_STATE stuck();
		Config::GDOOR_STATE change_state();
		
		void mqtt_send_state(Config::GDOOR_STATE new_state); // TODO: obs notify?
		
		bool load_data();
		bool save_data();
		
		void print_data();
		void run();
};
```

This is quite self-explanatory as well. The machine definition depends on the definition of
its components in `components.hpp` and the namespaces available in `config.hpp`.

There are some data fields that could --and should-- be simplified away, but I made some
shortsighted decisions in the implementation that neccessitated the availability of some
data outside their original scope in member functions. I would additionally restrict some
of the member functions to be private, because they realistically don't need to be called
from the outside (e.g. `at_floor()`, `at_ceil()`).

### Components

The `components.hpp` defines the elementary level components used in the actual device.
Most of these are really simple GPIO-pin derived classes that just have a consistent
constructor to configure a specific combination of pullups, io, or inversion (see: Button, Led, Limit, Driver).

This file, naturally, depends on the actual SDK implementations for configuring the hardware
via the official pico-sdk as RP2040 was used for this project.

I wasn't in charge of the Wifi and MQTT implementation used in this project, so I can't comment on it too much,
but I feel like that component on its own has enough dependencies that it could've maybe been separated into its
own files.

```c++
#pragma once

#include <stdio.h>

#include "pico/stdlib.h"
#include "hardware/gpio.h"

#include "config.hpp"

#include "MQTTClient.h"
#include "MQTTConnect.h"
#include "MQTTPacket.h"

#include "mqtt/IPStack.h"
#include "mqtt/Countdown.h"


namespace Components {
	class GPIOPin {
		private:
			uint pin;
			bool output;
			bool pullup;
			bool invert;
		public:
			// Constructor
			GPIOPin(uint pin0, bool output0, bool pullup0, bool invert0);

			// Member functions
			bool read();
			void write(bool value);
			uint pin_nr();

			// Operators
			bool operator() ();
			void operator() (bool value);
			operator int();
	};

	class Button : public GPIOPin {
		public:
			// Constructor
			Button(uint pin0);

			// Member functions
			bool pressed();
	};

	class Led : public GPIOPin {
		private:
			uint32_t state_last_changed = 0;
		public:
			Led(uint pin0);
			void blinks();
	};

	class Limit : public GPIOPin {
		public: Limit(uint pin0);
	};

	class Driver : public GPIOPin {
		public: Driver(uint pin0);
	};

	class StepperMotor {
		private:
			Driver a, b, c, d;
			uint16_t drive_pos;
		public:
			// Constructor
			StepperMotor(uint pin_a, uint pin_b, uint pin_c, uint pin_d);

			// Member functions
			void drive(bool reverse);
	};

	class RotaryEncoder {
		private:
			GPIOPin pin_a, pin_b;
			static RotaryEncoder* instance;
			uint32_t last_active;
			volatile int counter = 0; // TODO: use pico que?
		public:
			// Constructor
			RotaryEncoder(uint pin_a0, uint pin_b0);
			
			// Member functions
			void react(int x);
			static void trampoline(uint gpio, uint32_t event_mask);
			void handler(uint gpio, uint32_t event_mask);

			int get_count();
			void reset_count();
			uint32_t get_last_active();
	};

	class Eeprom {
		private:
			uint address;
			uint op_delay_ms;
		public:
			// Constructor
			Eeprom(uint pin_sda, uint pin_scl, uint address0, uint op_delay_ms0);

			// Member Functions
			int split_u32_to_8(uint32_t src, uint8_t *dst, int pos);
			int split_u16_to_8(uint16_t src, uint8_t *dst, int pos);
			uint32_t merge_u32_from_8(uint8_t *src, int *pos);
			uint16_t merge_u16_from_8(uint8_t *src, int *pos);
			uint16_t crc16(const uint8_t *pData, size_t len);
			
			bool read(uint16_t address, uint8_t *buffer, uint count);
			bool write(uint16_t address, uint8_t *buffer, uint count);
	};


	class WifiMQTT{
	// As the name implies, this class could do
	// with splitting into two, but time constraints
	// are secondary to functionality for now.
		private:
			const char *ssid;
			const char *pass;
			const char *broker_ip;
			const int broker_port;
			
			static WifiMQTT* instance;

			bool wifi_connection = false;
			bool mqtt_connection = false;
			
			char last_mqtt_msg[128];
			int last_mqtt_msg_len = 0;
			bool msg_handled = false;
			
			Countdown countdown;
			IPStack ipstack;
			MQTT::Client<IPStack, Countdown> client;
			MQTTPacket_connectData data;
		public:
			// Constructor
			WifiMQTT(const char *ssid0, const char *pass0,
					const char *broker_ip0, int broker_port0);

			// Member functions
			bool connect_wifi();
			bool connect_mqtt();
			bool is_connected();

			bool mqtt_subscribe(const char* topic);
			bool mqtt_send_msg(const char* msg);
			bool new_msg();
			Config::GDOOR_STATE mqtt_get_cmd();

			void loop(int timeout_ms);

			void messageArrived(MQTT::MessageData &md);
			static void messageArrivedWrapper(MQTT::MessageData &md);
	};
};
```

Most of the definitions here look okay to me. If I had more time I would've split some functionality
off from the components (serialisation functions from Eeprom, e.g.). I just jammed it into one
when I did testing to speed things up, as I didn't want to spend time deliberating on what kind of
additional namespaces would've been needed.

The RotaryEncoder class has a notable issue in it, not because it doesn't work, but because its a pain
to use in practice. It doesn't use a queue for tracking the inputs from the hardware irq, instead
it has a counter. This is kind of okay for this project since I just need to differentiate between
the door moving up and down (I don't care about the switch button in the rotary). But crucially
this makes the counter itself live in the RotaryEncoder object, making tracking it a little more
cumbersome than needed in the main device object itself. This is very evident when states need to
be saved and loaded from the Eeprom! Effectively requiring another counter in the main device anyhow.

### Configuration

The `config.hpp` is the file that contains our general configuration values (Config::),
and pin assignments (Pins::).

A larger project would probably benefit from better separation, specially between component
and device specific values, but this was completely workable for a project this small.

```c++
#pragma once

#include <cstdint>

#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "hardware/i2c.h"


namespace Config {
	constexpr bool DEBUG_MODE = true;
	constexpr uint32_t DEBOUNCE_MS = 75;
	constexpr uint32_t DEBOUNCE_US = DEBOUNCE_MS * 1000;
	constexpr uint32_t INPUT_CD_MS = 200;
	constexpr uint32_t INPUT_CD_US = INPUT_CD_MS * 1000;
	constexpr uint32_t LED_BLINK_INTERVAL_MS = 750;
	constexpr uint32_t LED_BLINK_INTERVAL_US = LED_BLINK_INTERVAL_MS * 1000;
	
	// Stepper Motor
	constexpr int STEPPER_INCR = 128;
	constexpr int STEPPER_STUCK_DELTA = 1;
	constexpr int STEPPER_STUCK_CTR_MAX = 3; // belt wobble makes this a pain
	constexpr uint8_t STEPPER_ROWS = 8;
	constexpr uint8_t STEPPER_TABLE[8][4] = {
		{1, 0, 0, 0},
		{1, 1, 0, 0},
		{0, 1, 0, 0},
		{0, 1, 1, 0},
		{0, 0, 1, 0},
		{0, 0, 1, 1},
		{0, 0, 0, 1},
		{1, 0, 0, 1} 
	};
	constexpr uint16_t STEPPER_STEPS = 4096;
	constexpr uint16_t STEPPER_SLEEP_US = 1250;
	constexpr int STEPPER_CALIB_DELTA = 3; // shrinks calibration span from limits

	// Eeprom
	constexpr auto I2C_UNIT = i2c0;
	constexpr int I2C_BAUD_HZ = 100 * 1000;
	constexpr uint EEPROM_DEV_ADDR = 0x50;
	constexpr uint EEPROM_DATA_ADDR = 8 * 64;
	constexpr uint EEPROM_WRITE_DELAY_MS = 5;
	constexpr uint EEPROM_PAGES = 512;
	constexpr uint EEPROM_PAGE_SIZE = 64;
	constexpr uint EEPROM_MEMBER_SIZE = 8;

	// Wifi & MQTT
	constexpr const char* WIFI_SSID = "MySSID";
	constexpr const char* WIFI_PASS = "MySecurePassword";

	constexpr const char* MQTT_BROKER_IP = "192.168.1.123";
	constexpr int MQTT_BROKER_PORT = 1883;

	// Garage Door
	constexpr int GDOOR_MAGIC = 0x2A;
	constexpr uint8_t GDOOR_DATA_LEN = 8+2; // BYTE length, depends on saved fields!
    
	enum class GDOOR_STATE : uint8_t {
		UNKNOWN,     // door is not calibrated
		CALIBRATING, // door is calibrating
		CEIL,        // door is at ceiling
		FLOOR,       // door is at floor
		CLOSING,     // door was last seen as closing
		OPENING,     // door was last seen as opening
		STOPPED,     // door was stopped during movement
		BAD_CMD,     // remote controller sent bad command
		ERROR        // door encountered error
	};
	enum class GDOOR_MOV : uint8_t {
		NONE,
		UP,
		DOWN,
	};
}

namespace Pins {
	// Stepper Motor
	constexpr uint STEPPER0 = 2;
	constexpr uint STEPPER1 = 3;
	constexpr uint STEPPER2 = 6;
	constexpr uint STEPPER3 = 13;
	
	// Buttons
	constexpr uint BUTTON_A = 7;
	constexpr uint BUTTON_B = 9;

	// Leds
	constexpr uint LED0 = 20;
	constexpr uint LED1 = 21;
	constexpr uint LED2 = 22;

	// Rotary Encoder
	constexpr uint ROT_A = 14;
	constexpr uint ROT_B = 15;
	
	// Limit Switches
	constexpr uint LIM_FLOOR = 4;
	constexpr uint LIM_CEIL = 5;

	// Eeprom
	constexpr uint EEPROM_SDA = 16;
	constexpr uint EEPROM_SCL = 17;
}
```

## Closing Thoughts

During this project/assignment I focused above all on OOP, interfaces, namespaces, and source
structure in the build system. I think I found a good balance between separation, readability,
and maintainability.

The more complex WifiMQTT element should've definitely been a separate
file with its own definitions for easier testing and development, and this is something I will definitely consider
in the future when I see a component requiring more unique dependencies to others.

### Miscellaneous

As an aside... A thing I spent an ungodly amount of time on (somehow) was the local controls. Namely the
*button_a* and *button_b* in the local controls of the door. I had a hell of a time making them
behave exactly how I wanted (non-blocking, triggering in a way that felt good to use, and proper debounce).

It felt like when I fixed one aspect, a new problem cropped up. I mean how damn difficult could a simple switch
button really be? It's literally about as simple as it gets... In hindsight I should've designed the button class
with a couple different press detection member functions (press, long press, double press, falling edge, ...),
which would've allowed me to just choose the implementation I want to use at different parts of the
device code when polling for user input.

Combining these multiple press detection functions with a global input cooldown on the device side
would've been the perfect solution with easy to maintain interfaces and granular component specific limits.

The more you know...
