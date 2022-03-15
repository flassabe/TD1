# TD1 - WiFi data

This session's goal consists in understanding and managing Wi-Fi data focused on positioning systems. We'll apply these data in Python programs.

## Context

Since we'll be using regular hardware, we'll rely on RSSI measurements for providing positioning. Data acquisition will be considered later, this session is focusing on how to store, access and use RSSI data.

## RSSI data

In the following document, a frame is the data unit transmitted on the OSI link layer, without consideration for its content on the upper layers.

On most of Wi-Fi devices, the Wi-Fi network interface card is able to measure the power of any received frame (it also knows its transmitting power). This power, usually measured in dBm (decibel based on milliwatts) is provided as RSSI by the Wi-Fi NIC's driver.

Notice that one RSSI value relates to a given frame, so that **each individual frame** (whatever its source) **has its own RSSI value**.

In a typical measurement process, a device will listen to the channel in monitoring mode and retrieve RSSI values for each frame received. Frames headers contain the frame's source MAC address, which should be unique for each device (we don't consider MAC address spoofing).

Technical goals within a positioning system are :

 - associating devices to their frames
 - extracting RSSI values for each frame
 - associating RSSI samples to their measuring source (the device which performed the measurement)

Performing these goals allows, for each frame, to be able to tell *"This frame was sent by that device and measured by that receiver"*

Given some choices in the architecture, sources and probes (measuring devices) may be varying devices on the network. This will be covered during lectures.

Since locating devices relies either :
- on associating probes or transmitters to locations,
- or on associating sets of RSSI samples to locations

this must be addressed by our data structure.

## RSSI sample

The base data structure we will be using is the RSSI sample. It is a RSSI value associated to a reference MAC address (the reference is a known device, such as an access point or a wireless LAN probe). In Python, this will be the following data structure:

```python
class RSSISample:
	def __init__(self, mac_address: str, rssi: float) -> None:
		self.mac_address = mac_address
		self.rssi = rssi
```

RSSI samples values may be related to one measurement, or to a sequence of measurements computed as an average value.

## Fingerprint sample

In fingerprinting systems, a set of RSSI samples is associated to a location. The set of RSSI samples is composed of values for each reference MAC address received from the current fingerprint location. Such set is also used in association with a mobile device before being compared to fingerprints. In Python, we'll use the following definition:

```python
class FingerprintSample:
	def __init__(self, samples: list[RSSISample]) -> None:
		self.samples = samples
```

## Location

Since we aim at solving device's locations, we must process locations. We will define a simple location with its x, y and z coordinate. Further work might rely on more data for a location, in a child class.

```python
class SimpleLocation:
	def __init__(self, x: float, y: float, z: float) -> None:
		self.x = x
		self.y = y
		self.z = z
```

## Fingerprint

For a fingerprint, we associate a Fingerprint sample to a location.

```python
class Fingerprint:
	def __init__(self, position: SimpleLocation, sample: FingerprintSample) -> None:
		self.position = position
		self.sample = sample
```

## Fingerprint database

A fingerprint database is composed of a collection of Fingerprints. We initialize it with an empty fingerprints list.

```python
class FingerprintDatabase:
	def __init__(self) -> None:
		self.db = []
```

# Work: manipulating RSSI and location data (distance based)

## RSSI values averaging

First, we want to be able to collate several RSSI values on a given sampling phase into a single value. We will use the averaging to get a single value. Since dBm are not a linear unit, we must convert dBm to mW, average the converted values, then convert the result back to dBm. Add and implement the following function to the FingerprintSample class:

```python
class FingerprintSample:
	"""
	Other class members
	"""
	def get_average_rssi(self) -> float:
		"""
		Your code here, remove pass instruction below and return the average value.
		"""
		pass
```

Here are the formulas to convert between both units and scales:

* From mW to dBm: P(dBm) = 10.0 log (P(mW))
* From dBm to mW: P(mW) = 10.0 ^ (P(dBm)/10.0)

## Reading RSSI data from a CSV file

You are provided with a CSV file containing calibration data for a positioning system. The calibration was performed in the Numerica building (Montbéliard university site, FEMTO-ST offices) by walking from calibration point to calibration point. At each of these points, we stopped, faced north and triggered a measurements (several seconds), then repeated this process facing east, then south, then west. Then, we went to the next point and repeated the 4 measurements.

Each line of the file is structured as follows:

* First field: x coordinate of the calibration point
* Second field: y coordinate of the calibration point
* Third field: z coordinate of the calibration point
* Fourth field: mobile device orientation, with 4 values possible (ignored in this session):
	* 1: facing north
	* 2: facing east
	* 3: facing south
	* 4: facing west
* Remaining fields as pairs (so, the count of remaining fields is even):
	* First field of a pair: reference MAC address of the frame whose RSSI is measured during calibration (as 6 groups of hexadecimal digits separated by ':')
	* Second field of a pair: RSSI or the frame, as measured during the calibration process

In this exercise, you have to import RSSI data from the CSV file and aggregate RSSI for duplicate MAC addresses values. The aggregated value is the average value of the signals received for a given location as described above.

The content of the file must be loaded into memory using classes defined in previous sections. You may add members to the classes to meet your requirements. Then you must output the resulting structured data into a CSV file named 'result.csv', with 1 line for each point (orientations shall be collapsed, i.e. you must merge the 4 corresponding lines of a given x,y,z location, and set its orientation to zero). Therefore, each line of the resulting file must ensure the following properties:

* Its x,y,z set of values is unique over the entire file
* Its orientation value is 0
* Its pairs of RSSI samples (defined by a MAC address and its associated RSSI) are ordered by MAC addresses (ascending order)

## Example

This data extract will be used for a more thorough explanation of the expectations about what you have to do (I added line returns and blank lines for a better understanding):
```
5.40,29.59,1.20,1,00:13:ce:8f:77:43,-63,00:13:ce:97:78:79,-80, \
00:13:ce:8f:78:d9,-29,00:13:ce:95:de:7e,-73,00:13:ce:8f:78:d9,-41 \
,00:13:ce:97:78:79,-76,00:13:ce:95:e1:6f,-58,00:13:ce:8f:77:43,-61

5.40,29.59,1.20,2,00:13:ce:95:e1:6f,-57,00:13:ce:95:de:7e,-77, \
00:13:ce:8f:78:d9,-37,00:13:ce:8f:78:d9,-38,00:13:ce:95:e1:6f,-61, \
00:13:ce:95:e1:6f,-57,00:13:ce:8f:78:d9,-35,00:13:ce:95:e1:6f,-57, \
00:13:ce:8f:78:d9,-33

5.40,29.59,1.20,3,00:13:ce:97:78:79,-84,00:13:ce:8f:77:43,-61, \
00:13:ce:95:e1:6f,-57,00:13:ce:8f:78:d9,-24,00:13:ce:95:de:7e,-73, \
00:13:ce:97:78:79,-83

5.40,29.59,1.20,4,00:13:ce:8f:77:43,-64,00:13:ce:8f:78:d9,-29, \
00:13:ce:95:de:7e,-77,00:13:ce:97:78:79,-75,00:13:ce:8f:78:d9,-29, \
00:13:ce:95:e1:6f,-61,00:13:ce:8f:77:43,-63

...
```

These four lines define measurements for a single point (coordinate: 5.4, 29.59, 1.2) facing the four cardinal directions. As stated previously in the instructions, we want to group everything to get only one line for a point, and only one value for a given access point.

Here we see 5 access points, whose MAC addresses are:
 - 00:13:ce:8f:77:43
 - 00:13:ce:97:78:79
 - 00:13:ce:8f:78:d9
 - 00:13:ce:95: de:7e
 - 00:13:ce:95:e1:6f

For each MAC address, we have a couple RSSI measurements (for all four lines):

| MAC Address | Values	|
| ------ | ------------- |
| 00:13:ce:8f:77:43 | -63, -61, -61, -64, -63 |
| 00:13:ce:97:78:79 | -80, -76, -84, -83, -75 |
| 00:13:ce:8f:78:d9 | -29, -41, -37, -38, -35, -33, -24, -29, -29 |
| 00:13:ce:95: de:7e | -73, -77, -73, -77 |
| 00:13:ce:95:e1:6f | -58, -57, -61, -57, -57, -57, -61 |

As dBm cannot be averaged as is (they are not linear), we must convert them back to mW, then compute their average value, then convert the average value back to dBm, meaning we apply for each source dBm value the following formula:

```python
power_mW = 10 ** (power_dBm / 10.)
```
Then, we compute the average value, and convert this average value to dBm, like this:
```python
from math import log10
avg_dBm = 10. * log10(avg_mW)
```

When applied on the data above, one shall find (rounded to two decimals):

| MAC Address | Average RSSI (dBm)			|
|------ |---------------|
|00:13:ce:8f:77:43 | -62.23 |
|00:13:ce:97:78:79 | -78.20 |
|00:13:ce:8f:78:d9 | -30.00 |
|00:13:ce:95: de:7e | -74.55 |
|00:13:ce:95:e1:6f | -57.98 |

This results in the following line in the output file:
```
5.40,29.59,1.20,0,00:13:ce:8f:77:43,-62.23,00:13:ce:97:78:79, \
-78.20,00:13:ce:8f:78:d9,-30.00,00:13:ce:95:de:7e,-74.55, \
00:13:ce:95:e1:6f,-57.98
```
