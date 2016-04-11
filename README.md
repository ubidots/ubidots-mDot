# Multitech mDot - LoRa

<img class="imgHeader" src ="../images/devices/mDot.png" />

LoRa is a modulation technique with a significantly long range. The modulation is based on spread-spectrum techniques and a variation of chirp spread spectrum (CSS) with integrated forward error correction (FEC). Mdot is a node to connect an IoT project with a LoRa network.

<aside class="notice">
Before to di this tutorial, you need to do the MultiConnect Conduit tutorial, and set the Network name, password and sub-band channel. This tutorial is <p href="../devices/conduit.html">here</p>
</aside>

## Requiremets

* [MultiTech mDot](http://www.multitech.com/models/94557148LF)
* [An mbed developer account](https://developer.mbed.org/compiler)
* [A MultiConnect® mDot™ Micro Developer Kit](http://www.multitech.com/brands/micro-mdot-devkit)

## Setup

1. [Create an account in mbed Developer site](https://developer.mbed.org/accounts/login/?next=%2Fcompiler%2F).
2. Login to your mbed account and go to **Platforms**.
3. Select **MultiTech** under Platform vendor.
4. Click on **MultiTech mDot** and then click on **Add to your mbed compiler**.
    <img class="imgBody" src ="../images/devices/add_platform_mdot.gif" />
5. Click on the **Compiler** button in the upper right corner.
6. Create a new project, select **MultiTech mDot** as your platform, then select the template **Connecting to gateway and sending package with a MultiTech mDot** and click OK.
7. Go to main.cpp, erase its content and paste the corresponding sample code which you'll find in the next steps.

<img class="imgBody" src ="../images/devices/select_platform_mdot.gif" />


## Send one value to Ubidots

To send a value from Analog Pin copy the next code and paste in into **main.cpp** don't forget to change in the code the **Lora network name**, **Frequency Sub-Band** and **Passphrase**.

```c
#include "mbed.h"
#include "mDot.h"
#include "MTSLog.h"
#include <AnalogIn.h>
#include <string>
#include <vector>
#include <algorithm>
AnalogIn value(PB_1);
// these options must match the settings on your Conduit
// uncomment the following lines and edit their values to match your configuration
static std::string config_network_name = "Network_Name_of_Conduit";
static std::string config_network_pass = "Network_Pass_of_Conduit";
static uint8_t config_frequency_sub_band = 1;
int main() {
    int32_t ret;
    mDot* dot;
    std::vector<uint8_t> data;
    int inValue;
    char* str = new char[3];
    float sensorValue;    
    sensorValue = value.read();
    inValue = int(sensorValue*1000);
    sprintf(str, "%d", inValue );
    
    // get a mDot handle
    dot = mDot::getInstance();    
    // print library version information
    logInfo("version: %s", dot->getId().c_str());
    //*******************************************
    // configuration
    //*******************************************
    // reset to default config so we know what state we're in
    dot->resetConfig();    
    dot->setLogLevel(mts::MTSLog::INFO_LEVEL);
    // set up the mDot with our network information: frequency sub band, network name, and network password
    // these can all be saved in NVM so they don't need to be set every time - see mDot::saveConfig()    
    // frequency sub band is only applicable in the 915 (US) frequency band
    // if using a MultiTech Conduit gateway, use the same sub band as your Conduit (1-8) - the mDot will use the 8 channels in that sub band
    // if using a gateway that supports all 64 channels, use sub band 0 - the mDot will use all 64 channels    
    if ((ret = dot->setPublicNetwork(true)) != mDot::MDOT_OK) {
        logError("failed to set public network %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }
    logInfo("setting frequency sub band");    
    if ((ret = dot->setFrequencySubBand(config_frequency_sub_band)) != mDot::MDOT_OK) {
        logError("failed to set frequency sub band %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }    
    logInfo("setting network name");
    if ((ret = dot->setNetworkName(config_network_name)) != mDot::MDOT_OK) {
        logError("failed to set network name %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }    
    logInfo("setting network password");
    if ((ret = dot->setNetworkPassphrase(config_network_pass)) != mDot::MDOT_OK) {
        logError("failed to set network password %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }    
    // a higher spreading factor allows for longer range but lower throughput
    // in the 915 (US) frequency band, spreading factors 7 - 10 are available
    // in the 868 (EU) frequency band, spreading factors 7 - 12 are available
    logInfo("setting TX spreading factor");
    if ((ret = dot->setTxPower(20)) != mDot::MDOT_OK) {
        logError("failed to set TX datarate %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }
    logInfo("setting TX power");
    if ((ret = dot->setTxDataRate(mDot::SF_9)) != mDot::MDOT_OK) {
        logError("failed to set TX datarate %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }        
    // request receive confirmation of packets from the gateway
    logInfo("enabling ACKs");
    if ((ret = dot->setAck(1)) != mDot::MDOT_OK) {
        logError("failed to enable ACKs %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }   
    // save this configuration to the mDot's NVM
    logInfo("saving config");
    if (! dot->saveConfig()) {
        logError("failed to save configuration");
    }
    //*******************************************
    // end of configuration
    //*******************************************
    // attempt to join the network
    logInfo("joining network");
    while ((ret = dot->joinNetwork()) != mDot::MDOT_OK) {
        logError("failed to join network %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
        // in the 868 (EU) frequency band, we need to wait until another channel is available before transmitting again
        osDelay(std::max((uint32_t)1000, (uint32_t)dot->getNextTxMs()));
    }
    // format data for sending to the gateway
    for (int i = 0; i < 3; i++)
        data.push_back((uint8_t) str[i]);
    while (true) {
        // send the data to the gateway
        if ((ret = dot->send(data)) != mDot::MDOT_OK) {
            logError("failed to send", ret, mDot::getReturnCodeString(ret).c_str());
        } else {
            logInfo("successfully sent data to gateway");
        }
        // in the 868 (EU) frequency band, we need to wait until another channel is available before transmitting again
        osDelay(std::max((uint32_t)5000, (uint32_t)dot->getNextTxMs()));
    }
    return 0;
}
```

## Send multiple values to Ubidots 

To send three values from Analog Pins (A0, A1, A2) copy the next code and paste in into **main.cpp** don't forget to change in the code the **Lora network name**, **Frequency Sub-Band** and **Passphrase**. Now pres compile and copi the .bin file to your 

```c
#include "mbed.h"
#include "mDot.h"
#include "MTSLog.h"
#include <AnalogIn.h>
#include <string>
#include <vector>
#include <algorithm>
AnalogIn value(PB_1);
AnalogIn value2(PB_2);
AnalogIn value3(PB_3);
// these options must match the settings on your Conduit
// uncomment the following lines and edit their values to match your configuration
static std::string config_network_name = "UbidotsLora";
static std::string config_network_pass = "clave123456789";
static uint8_t config_frequency_sub_band = 1;
int main() {
    int32_t ret;
    mDot* dot;
    std::vector<uint8_t> data;
    std::vector<uint8_t> data2;
    std::vector<uint8_t> data3;
    int inValue, inValue2, inValue3;
    char* str = new char[3];
    char* str2 = new char[3];
    char* str3 = new char[3];
    float sensorValue, sensorValue2, sensorValue3;    
    sensorValue = value.read();
    sensorValue = value2.read();
    sensorValue = value3.read();
    inValue = int(sensorValue*1000);
    inValue2 = int(sensorValue*1000);
    inValue3 = int(sensorValue*1000);
    sprintf(str, "%d", inValue );
    sprintf(str2, "%d", inValue2 );
    sprintf(str3, "%d", inValue3 );
    
    // get a mDot handle
    dot = mDot::getInstance();    
    // print library version information
    logInfo("version: %s", dot->getId().c_str());
    //*******************************************
    // configuration
    //*******************************************
    // reset to default config so we know what state we're in
    dot->resetConfig();    
    dot->setLogLevel(mts::MTSLog::INFO_LEVEL);
    // set up the mDot with our network information: frequency sub band, network name, and network password
    // these can all be saved in NVM so they don't need to be set every time - see mDot::saveConfig()    
    // frequency sub band is only applicable in the 915 (US) frequency band
    // if using a MultiTech Conduit gateway, use the same sub band as your Conduit (1-8) - the mDot will use the 8 channels in that sub band
    // if using a gateway that supports all 64 channels, use sub band 0 - the mDot will use all 64 channels    
    if ((ret = dot->setPublicNetwork(true)) != mDot::MDOT_OK) {
        logError("failed to set public network %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }
    logInfo("setting frequency sub band");    
    if ((ret = dot->setFrequencySubBand(config_frequency_sub_band)) != mDot::MDOT_OK) {
        logError("failed to set frequency sub band %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }    
    logInfo("setting network name");
    if ((ret = dot->setNetworkName(config_network_name)) != mDot::MDOT_OK) {
        logError("failed to set network name %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }    
    logInfo("setting network password");
    if ((ret = dot->setNetworkPassphrase(config_network_pass)) != mDot::MDOT_OK) {
        logError("failed to set network password %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }    
    // a higher spreading factor allows for longer range but lower throughput
    // in the 915 (US) frequency band, spreading factors 7 - 10 are available
    // in the 868 (EU) frequency band, spreading factors 7 - 12 are available
    logInfo("setting TX spreading factor");
    if ((ret = dot->setTxPower(20)) != mDot::MDOT_OK) {
        logError("failed to set TX datarate %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }
    logInfo("setting TX power");
    if ((ret = dot->setTxDataRate(mDot::SF_9)) != mDot::MDOT_OK) {
        logError("failed to set TX datarate %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }        
    // request receive confirmation of packets from the gateway
    logInfo("enabling ACKs");
    if ((ret = dot->setAck(1)) != mDot::MDOT_OK) {
        logError("failed to enable ACKs %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
    }   
    // save this configuration to the mDot's NVM
    logInfo("saving config");
    if (! dot->saveConfig()) {
        logError("failed to save configuration");
    }
    //*******************************************
    // end of configuration
    //*******************************************
    // attempt to join the network
    logInfo("joining network");
    while ((ret = dot->joinNetwork()) != mDot::MDOT_OK) {
        logError("failed to join network %d:%s", ret, mDot::getReturnCodeString(ret).c_str());
        // in the 868 (EU) frequency band, we need to wait until another channel is available before transmitting again
        osDelay(std::max((uint32_t)1000, (uint32_t)dot->getNextTxMs()));
    }
    // format data for sending to the gateway
    // data one
    for (int i = 0; i < 3; i++)
        data.push_back((uint8_t) str[i]);
    // data two
    for (int i = 0; i < 3; i++)
        data2.push_back((uint8_t) str2[i]);
    // data three
    for (int i = 0; i < 3; i++)
        data3.push_back((uint8_t) str3[i]);
    while (true) {
        // send the data to the gateway
        if ((ret = dot->send(data)) != mDot::MDOT_OK) {
            logError("failed to send", ret, mDot::getReturnCodeString(ret).c_str());
        } else {
            logInfo("successfully sent data to gateway");
        }
        if ((ret = dot->send(data2)) != mDot::MDOT_OK) {
            logError("failed to send", ret, mDot::getReturnCodeString(ret).c_str());
        } else {
            logInfo("successfully sent data to gateway");
        }
        if ((ret = dot->send(data3)) != mDot::MDOT_OK) {
            logError("failed to send", ret, mDot::getReturnCodeString(ret).c_str());
        } else {
            logInfo("successfully sent data to gateway");
        }
        // in the 868 (EU) frequency band, we need to wait until another channel is available before transmitting again
        osDelay(std::max((uint32_t)5000, (uint32_t)dot->getNextTxMs()));
    }
    return 0;
}
```