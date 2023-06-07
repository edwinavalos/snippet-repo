# Docs

```go
package main

import (
	"github.com/d2r2/go-dht"
	"github.com/d2r2/go-logger"
	"github.com/stianeikeland/go-rpio/v4"
	"log"
)

func main() {
	log.Println("Opening rpio")
	err := rpio.Open() // Open up our access to the GPIO pins (requires sudo)
	if err != nil {
		log.Panicf("could not open rpio: %s", err)
	}
	defer rpio.Close()

	err = logger.ChangePackageLogLevel("dht", logger.FatalLevel) // This turns off very noisy default logging
	if err != nil {                                              // in the dht-11 library we are using
		log.Fatalf("could not turn off dht logger: %s", err)
	}

	pinNumber := 18 // This is the pin that we are going to be interacting with, it is what we connected the sensor to
	tries := 10     // The reading of the sensor is unreliable, and the library has retries built in

	temperature, humidity, _, err := dht.ReadDHTxxWithRetry(dht.DHT11, int(pinNumber), false, tries)
	if err != nil {
		log.Fatalf("unable to read sensor after: %d tries err: %s", tries, err)
	}

	log.Printf("temperature: %0.2f°C humidity: %0.2f%%\n", temperature, humidity) // Print the information we gathered
}
```

```console
amos-labs@raspberrypi:~/repos/go-dht11-example $ sudo ./go-dht11-example 
2023/06/02 20:42:18 Opening rpio
2023/06/02 20:42:20 temperature: 24.00°C humidity: 78.00%
```

```console
amos-labs@raspberrypi:~ $ sudo raspi-config nonint do_i2c 0
amos-labs@raspberrypi:~ $ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- 36 -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

```go
package main

import (
	"fmt"
	"github.com/asssaf/stemma-soil-go/soil"
	"log"
	"periph.io/x/conn/v3/i2c/i2creg"
	"periph.io/x/host/v3"
)

func main() {
	if _, err := host.Init(); err != nil { // We need to load drivers to work with i2c stuff
		panic(err)
	}

	devAddr := 0x36                           // This is the register that we found when we ran i2cscan
	i2cPort, err := i2creg.Open("/dev/i2c-1") // This is the filepath that our device is mounted to
	if err != nil {
		log.Panicf("could not open i2c device: %s", err)
	}

	opts := soil.DefaultOpts // This instantiates default settings for the soil library, the default is already 0x36, but
	// could change based on if we want to hook up multiple of these, we would need to jump the sensor,
	// and update this code
	if devAddr != 0 {
		if devAddr < 0x36 || devAddr > 0x39 {
			panic(fmt.Sprintf("given address not supported by device: %x", devAddr))
		}
		opts.Addr = uint16(devAddr)
	}

	dev, err := soil.NewI2C(i2cPort, &opts) // instantiate our soil libraries i2c connection
	if err != nil {
		panic(err)
	}
	defer dev.Halt() // Close out the connection we create when this function finishes

	values := soil.SensorValues{} // Create the sensor value struct we will populate in the next line
	if err := dev.Sense(&values); err != nil {
		log.Fatalf("unable to sense moisture: %s", err)
	}
	floatCap := float32(values.Capacitance) // Turn the Capacitance into a float that we can report with float formatting
	log.Printf("temperature: %0.2f°C capacitance : %0.2f\n", values.Temperature.Celsius(), floatCap)
}
```


```console
amos-labs@raspberrypi:~/repos/go-soil-sensor-example $ ./go-soil-sensor-example 
2023/06/02 20:44:43 temperature: 24.00°C capacitance : 365.00
```

```go
package main

import (
	"github.com/stianeikeland/go-rpio/v4"
	"log"
)

func main() {
	log.Println("Opening rpio")
	err := rpio.Open() // Open access to the GPIO
	if err != nil {
		log.Panicf("could not open rpio: %s", err)
	}
	defer rpio.Close() // Close the connection once we are done with this function

	relay1Pin := 18 // This is the pin that we connected to our first relay
	log.Printf("connecting to pin %d\n", relay1Pin)
	relay1 := rpio.Pin(18) // instantiate the pin

	state := relay1.Read() // Read its current status
	if state == rpio.Low {
		log.Println("relay is currently 'off'")
	} else {
		log.Println("relay is currently 'on'")
	}

	relay1.Output() // Prepare the pin to be written to
	log.Println("toggling the relay")
	relay1.Toggle() // Toggle the relay, it switches from high to low and low to high
}
```


```console
amos-labs@raspberrypi:~/repos/go-pi-relay-example $ ./go-pi-relay-example 
2023/06/02 23:18:38 Opening rpio
2023/06/02 23:18:38 connecting to pin 18
2023/06/02 23:18:38 relay is currently 'on'
2023/06/02 23:18:38 toggling the relay
amos-labs@raspberrypi:~/repos/go-pi-relay-example $ ./go-pi-relay-example 
2023/06/02 23:18:45 Opening rpio
2023/06/02 23:18:45 connecting to pin 18
2023/06/02 23:18:45 relay is currently 'off'
2023/06/02 23:18:45 toggling the relay
amos-labs@raspberrypi:~/repos/go-pi-relay-example $ ./go-pi-relay-example 
2023/06/02 23:18:46 Opening rpio
2023/06/02 23:18:46 connecting to pin 18
2023/06/02 23:18:46 relay is currently 'on'
2023/06/02 23:18:46 toggling the relay
```

```go
package main

import (
	"github.com/pimvanhespen/go-pi-lcd1602"
	"github.com/pimvanhespen/go-pi-lcd1602/synchronized"
	"log"
	"time"
)

func main() {
	lcdi := lcd1602.New(18, 23, []int{16, 12, 25, 24}, 16)
	syncedLCD := synchronized.NewSynchronizedLCD(lcdi)
	syncedLCD.Initialize()
	defer syncedLCD.Close()

	log.Println("Resetting screen")
	syncedLCD.WriteLines()

	log.Println("Printing a message")
	syncedLCD.WriteLines("Hello world!")
	log.Println("Sleeping for 5 seconds")
	time.Sleep(time.Second * 5)
	log.Println("Resetting the screen")
	syncedLCD.WriteLines()
	log.Println("Writing another message")
	syncedLCD.WriteLines("from Amos Labs")
	time.Sleep(time.Second * 2)
	log.Println("Resetting the screen")
	syncedLCD.WriteLines()
	log.Println("Writing multiple lines")
	syncedLCD.WriteLines("This is a ", "test of the lcd")
}
```

```console
amos-labs@raspberrypi:~/repos/go-lcd1602-example $ ./go-lcd1602-example
2023/06/06 19:49:18 Resetting screen
2023/06/06 19:49:18 Printing a message
2023/06/06 19:49:18 Sleeping for 5 seconds
2023/06/06 19:49:23 Resetting the screen
2023/06/06 19:49:23 Writing another message
2023/06/06 19:49:25 Resetting the screen
2023/06/06 19:49:25 Writing multiple lines
```
