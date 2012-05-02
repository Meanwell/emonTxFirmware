                          _____    
                         |_   _|   
  ___ _ __ ___   ___  _ __ | |_  __
 / _ \ '_ ` _ \ / _ \| '_ \| \ \/ /
|  __/ | | | | | (_) | | | | |>  < 
 \___|_| |_| |_|\___/|_| |_\_/_/\_\
           Interrupt Driven Version

http://openenergymonitor.org/emon/emontx
***********************************

This code is based on EmonTx Ofiginal version by OpenEnergyMonitor.org
This code was inspired by Atmel´s Doc AVR465: Energy Meter using tinyAVR and megaAVR devices <http://www.atmel.com/Images/doc2566.pdf> 

The goal of this fork is to provide a 1-3Phase high precise wireless energy monitor.

==WARNING===
This code is not energy efficient. It uses the processor 100% of its time and it does never go to sleep.
So, do not run it from battery.
============

The code is split in 3 parts:

1 - Configuration Management
2 - Interrupt Service Routine that do nerly all the power calculations
3 - Main loop - That do the sums of the data gathered by the ISR, and send it to the RFM12

===Defines===

Do you want serial debug? set this to 1
#define SERIAL 0

//in seconds - time wich arduino will do power calculations. this must be kept low.
#define CALC_INTERVAL 2 

#define FREQUENCY 60 // IN HZ

//How many CTS for the main breaker (may not be on the same phase)
#define  PHASE_SENSORS  3

// how many voltage sensors do you have (Open energy monitor dafaults to 1)
#define  VOLTAGE_SENSORS 1

//Phase shift angle between phases (defaults to 120 for 3 phase circuits) not in use now.
#define  PHASE_SHIFT  = 120

//MAX_SHIFT_DEGREE = 240
#define ADC_BUFFER_SIZE  33 // see below

//phase2 offset in samples
#define PHASE1_OFFSET 16
//phase3 offset in samples
#define PHASE2_OFFSET 32


//see the ISR below but this is in resume the order of the pins that will be probed. You have to put here the arduino ports of the sensors.
//the order below is from Emontx documentation. Voltage1, Current1, Current2, Current3.
unsigned char input_analog_pins[PHASE_SENSORS + VOLTAGE_SENSORS] = {2,3,0,1};


===Remote Config Commands===

There was intruducted EEPROm config of user defineable variables:

config.band = 4;				//Radio Band 4=433Mhz (see Jeelabs)
config.group = 210;				//Radio Group (RFM12B Only, RFM12 is always 210)
config.nodeID = 10;				//My Node ID
config.sendTo = 0;				//Note that EmonTx will sens it´s measurement payload. 0 For broadcast so that GLCD and BASE can receive it.
config.sendTo = 3;			//Intervals of Calc Interval(2 seconds) that the payload will be transmited
config.Phase_Calibration[0] = 1.7;		//Phase Shift calibration of 1 phase
config.Phase_Calibration[1] = 1.7;		//Phase Shift calibration of 2 phase
config.Phase_Calibration[2] = 1.7;		//Phase Shift calibration of 3 phase
config.Current_Calibration[0] = 0.1080;		//Current Calibration of 1 phase
config.Current_Calibration[1] = 0.1080;		//Current Calibration of 2 phase
config.Current_Calibration[2] = 0.1080;		//Current Calibration of 3 phase
config.Voltage_Calibration[0] = 0.4581;		//Voltage Calibration of 1 phase
config.Voltage_Calibration[1] = 0.4581;		//Voltage Calibration of 2 phase
config.Voltage_Calibration[2] = 0.4581;		//Voltage Calibration of 3 phase
config.Voltage = 127.0;				//is you dont have voltage sensor, put here your grid voltage.

on the Github you will find a DOC containing all possible RF commands that this node wikk respond to.

==ISR===
The code uses an ISR to monitor the channels in a specified order:
Considering you have 3 current sensors:

If we have 0 voltage sensors the order will be:

Current Channel1
Current Channel2
Current Channel3
Loop

If we have 1 voltage sensor the order will be: - The voltage sensor must be connected to the line of the first current sensor.

Voltage Channel1
Current Channel1
Current Channel2
Current Channel3
Loop

If we have 3 voltage sensors the order will be:

Voltage Channel1
Current Channel1
Voltage Channel2
Current Channel2
Voltage Channel3
Current Channel3
Loop

===ADC Conversions===
My version of EmonTx uses 20Mhz Crystal for better accuracy but you can use it with the original 16Mhz ones.

Since the Atmega´s internal ADC is a 10 bit one capeable of continuous samples with auto trigger interrupt, the code monitor the AC grid in a fixed time interval defined by the following formula:

(Processor Clock) / (ACD Prescaler) / (Conversion Cycles (13)) ( / Num CTs / Num Voltage Sensors)

For 16mhz Crystal: 16000000 / 128 / 13 = 9615 Samples Per Second
For 20Mhz Crystal: 20000000 / 128 / 13 = 12019 Samples Per Second

I have 3 Cts and 1 voltage sensor (the maximun emonTx hardware) butis is possible to as 2 more voltage sensors with a few components.

For 3 cts and 1 voltage sensor you have to measure 4 channels in a constant form, so you have:

For 16mhz Crystal: 9615 / 4 = 2403 Samples Per Channel per Second (SPCPS)
For 20Mhz Crystal: 12019 / 4 = 3004 Samples Per Channel per Second

Lets consider 60Hz power grid, you will then have SPCPS / 60:

For 16mhz Crystal: 2403 / 60 = 40 Samples per Waveform
For 20Mhz Crystal: 3004 / 60 = 50 Samples per Waveform

In my oppinion it´s a good precision.

If you are monitoring only a single phase with one CT on 16Mhz at 60hz there will be 80 samples per waveform.


===ADC Buffering===
Adc Buffering is a name i gave to the algorytm i´ve written to deal with the real power / power factor of 3 phases with only 1 voltage sensor.

The code uses an array to store the last n voltage measurements so it can shift the voltage by a certain angle to apply the correct Voltage x Current sample. It is not 100% precise because it will apply the current reading to the last sampled voltage reading to give the power that instant. Since the voltage does not varies so much, i think the code is ok.


max shift degree will be the total diference angle between phase 1 and 3. The phases must be in order.

#define ADC_BUFFER_SIZE  33 // ((((SAMPLES_PER_SECOND/FREQUENCY) * MAX_SHIFT_DEGREE) / 360)  +1);

for 16mhz and 60hz this shoud be: 

40 * 240 / 360 + 1 = 27 but this number did not worked for me. i stll did not figured out why. i´m using 33 here for 60hz 50hz will be a higer number.



TODO: i have to draw this...


===Payload===
Is a 62 Bytes data transmitted to config.sendTo every 2*config.sendTo seconds.

typedef struct { 
  byte nodeId;             //1 Byte	//sender node id
  byte command;            //1 Byte	//Command to transmit
  unsigned int txCount;    //2 bytes	//transmission count
  float totalP;            //4 bytes	//Computed Sum of real power of the 3 phases
  float cummKw;            //4 bytes	//Cummulative KhW since emonTx Power On
  unsigned char numV;      //1 Byte	//Number of Voltage Sensors so the other nodes can use this payload data
  unsigned char numI;      //1 Byte	//Number of Current Sensors so the other nodes can use this payload data
  float V1;                //4 bytes	//Voltage1
  float Irms1;             //4 bytes	//Current1
  float RP1;               //4 bytes	//real Power1
  float PF1;               //4 bytes	//Power Factor1
  float V2;                //4 bytes	//Voltage2      
  float Irms2;             //4 bytes	//Current2      
  float RP2;               //4 bytes	//real Power2   
  float PF2;               //4 bytes	//Power Factor2 
  float V3;                //4 bytes	//Voltage3      
  float Irms3;             //4 bytes	//Current3      
  float RP3;               //4 bytes	//real Power3   
  float PF3;               //4 bytes	//Power Factor3 
} PayloadTX;