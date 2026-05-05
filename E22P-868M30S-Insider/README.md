
# Why use E22P-868M30S

A completely new generation, coming out at the end of 2025 !<br>

This module adds a true RF front end module (RF FEM) comprising :
- A power amplifier (PA) up to 32 dBm at 868 MHz (see datasheet)
- A low-noise amplifier (LNA)
- Several surface acoustic wave (SAW) filters
- RF Ceramic Chips
- A high-precision active temperature compensated crystal oscillator

Compared to the previous old generation E22-900M30S,<br>
these modules significantly improve out-of-band spurious signal suppression capabilities,<br>
and offer enhanced anti-interference capabilities as well as increased receive sensitivity.<br>

<br>

Change of pin name from E22-900M30S to E22P-868M30S :
- RXEN pin is now EN pin
- TXEN pin is now T/R CTRL pin

The EN pin must now always be set to ON.<br>
However, to ensure good confuguration, full power and reduced floor noise,<br><b>it must be grounded for 3 seconds during startup !</b><br>
<br>

## Power Output @868MHz (dBm) - SX1262 Power Setting (dBm) - Power Supply (V) 
<table>
  <thead>
    <tr>
      <th>Pset</th>
      <th>5,0 V</th>
      <th>4,2 V</th>
      <th>3,7 V</th>
      <th>3,3 V</th>
    </tr>
  </thead>
  <tbody>
    <tr><th>1</th><td>21,7</td><td>20,7</td><td>20,7</td><td>20,7</td></tr>
    <tr><th>2</th><td>22,7</td><td>21,7</td><td>21,7</td><td>21,7</td></tr>
    <tr><th>3</th><td>23,7</td><td>22,7</td><td>22,7</td><td>22,7</td></tr>
    <tr><th>4</th><td>24,7</td><td>23,7</td><td>23,7</td><td>23,7</td></tr>
    <tr><th>5</th><td>25,7</td><td>24,7</td><td>24,7</td><td>24,7</td></tr>
    <tr><th>6</th><td>26,6</td><td>25,6</td><td>25,6</td><td>25,6</td></tr>
    <tr><th>7</th><td>27,6</td><td>26,6</td><td>26,6</td><td>26,6</td></tr>
    <tr><th>8</th><td>28,5</td><td>27,5</td><td>27,4</td><td>27,2</td></tr>
    <tr><th>9</th><td>29,4</td><td>28,4</td><td>28,1</td><td>27,8</td></tr>
    <tr><th>10</th><td>30,3</td><td>29,2</td><td>28,7</td><td>28,2</td></tr>
    <tr><th>11</th><td>30,7</td><td>29,6</td><td>29,1</td><td>28,5</td></tr>
    <tr><th>12</th><td>31,1</td><td>29,9</td><td>29,4</td><td>28,8</td></tr>
    <tr><th>13</th><td>31,4</td><td>30,2</td><td>29,6</td><td>28,9</td></tr>
    <tr><th>14</th><td>31,6</td><td>30,4</td><td>29,8</td><td>29,0</td></tr>
    <tr><th>15</th><td>31,8</td><td>30,6</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>16</th><td>31,9</td><td>30,7</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>17</th><td>32,0</td><td>30,8</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>18</th><td>32,0</td><td>30,9</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>19</th><td>32,1</td><td>30,9</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>20</th><td>32,1</td><td>31,0</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>21</th><td>32,1</td><td>31,0</td><td>30,0</td><td>29,0</td></tr>
    <tr><th>22</th><td>32,0</td><td>31,0</td><td>30,0</td><td>29,0</td></tr>
  </tbody>
</table>

<br>

## SF8301

The SF8301 is a Surface Acoustic Wave (SAW) filter, commonly used for high-frequency signal conditioning :
- Center Frequency: 868 MHz
- Bandwidth: 10 MHz

SAW Filters provide high-performance frequency selection, effectively separating and filtering signals from different frequency bands.<br>
This ensures smooth communication between communication devices while suppressing interference.<br>
This is essential for use in urban areas with Cellular mobile networks.<br>
<br>
Several surface acoustic wave (SAW) filters in E22P-868M30S is a game changer !

<br>

## KCT8102L

KCT8102L is a highly-integrated, RF Front-End Integrated Circuit which incorporates key RF functionality needed for a TDD-mode operating in the 860M to 930MHz ISM frequency bands. <br>
KCT8102L is integrated with a power amplifier (PA), a low noise amplifier (LNA) with bypass, and a single-pole, double-throw (SPDT) switch all in one device. <br>
KCT8102L is designed for ease of use and maximum flexibility, with different matching circuits, KCT8102L could achieve high power and high efficiency two operation status.<br>
KCT8102L has simple and low-voltage 1.6 to 3.6V control logic, and wide supply voltage range from 3.0 to 5.25V allowing the KCT8102L to be used in battery powered applications.<br>
KCT8102L is assembled in a compact, low-profile 3x3x0.77mm 16-lead LGA package. KCT8102L is the ideal RF front-end solution for implementing sub-GHz from 860M to 930MHz supporting multiple standards including LP-WAN (LoRa, Sigfox, etc.), Wi-SUN, and other loT applications.<br>
<br>
Integrated high-performance 860 to 930MHz Front-End Module<br>
Nominated voltage at 3.3V & 4.2V & 5V<br>
<br>
<b>+32.5dBm saturated output power at 5.0V High Power Matching</b><br>
+31.0dBm saturated output power at 5.0V High Efficiency Matching<br>
+31.0dBm saturated output power at 4.2V High Power Matching<br>
+29.5dBm saturated output power at 4.2V High Efficiency Matching<br>
+29.0dBm saturated output power at 3.3V High Power Matching<br>
+27.5dBm saturated output power at 3.3V High Efficiency Matching<br>
<br>
Integrated LNA with noise figure of 1.7dB at 5V & 4.2V & 3.3V<br>
Receive bypass path with low loss: 2.0dB at 5V & 4.2V & 3.3V<br>
Support external Receive filter<br>
ESD Protection Circuitry on All Ports<br>
Small package: LGA-16L, 3×3×0.77mm (MSL3, 260°C per JEDEC J-STD-020)<br>
RoHS and REACH Compliant<br>

<br>
<br>

<img src="images/E22P-868M30S - Outside.jpg" height="1000">

<img src="images/E22P-868M30S - Inside 1.jpg" height="1000">

<img src="images/E22P-868M30S - Inside Closer 1.jpg" height="1000">

<img src="images/E22P-868M30S - Inside Closer 2.jpg" height="1000">

<img src="images/E22P-868M30S - RealOutput versus SX1262 settings.jpg" height="1000">

<img src="images/E22P-868M30S - Comparison of the three BOX.jpg" height="1000">

