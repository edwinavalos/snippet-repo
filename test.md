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
	if _, err := host.Init(); err != nil {
		panic(err)
	}

	devAddr := 0x36
	i2cPort, err := i2creg.Open("/dev/i2c-1")
	if err != nil {
		log.Panicf("could not open i2c device: %s", err)
	}

	opts := soil.DefaultOpts
	if devAddr != 0 {
		if devAddr < 0x36 || devAddr > 0x39 {
			panic(fmt.Sprintf("given address not supported by device: %x", devAddr))
		}
		opts.Addr = uint16(devAddr)
	}

	dev, err := soil.NewI2C(i2cPort, &opts)
	if err != nil {
		panic(err)
	}
	defer dev.Halt()

	values := soil.SensorValues{}
	if err := dev.Sense(&values); err != nil {
		log.Fatalf("unable to sense moisture: %s", err)
	}
	floatCap := float32(values.Capacitance)
	log.Printf("temperature: %0.2f°C capacitance : %0.2f\n", values.Temperature.Celsius(), floatCap)
}
```


```console
amos-labs@raspberrypi:~/repos/go-soil-sensor-example $ ./go-soil-sensor-example 
2023/06/02 20:44:43 temperature: 24.00°C capacitance : 365.00
```