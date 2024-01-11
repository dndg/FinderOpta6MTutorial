---
title: "Introduzione a Finder Opta e agli analizzatori di rete Finder serie 6M"
description: "Impara a leggere i registri del 6M utilizzando il protocollo Modbus su Finder Opta."
author: 'Fabrizio Trovato'
libraries:
  - name: 'ArduinoRS485'
    url: https://www.arduino.cc/reference/en/libraries/arduinors485
  - name: 'ArduinoModbus'
    url: https://www.arduino.cc/reference/en/libraries/arduinomodbus
difficulty: intermediate
tags:
  - Getting-started
  - ModbusRTU
  - RS-485
  - Finder 6M
software:
  - ide-v1
  - ide-v2
  - arduino-cli
  - web-editor
hardware:
  - hardware/07.opta/opta-family/opta
---

## Panoramica

Tra i protocolli supportati dal Finder Opta, troviamo Modbus RTU. In questo
tutorial impareremo a implementare la comunicazione Modbus RTU tramite RS-485
tra Finder Opta e un analizzatore di rete Finder serie 6M. In particolare,
impareremo come utilizzare il Finder Opta per configurare un Finder serie 6M e
leggerne i registri.

## Obiettivi

* Imparare a stabilire la connettività dell'interfaccia RS-485 tra Finder Opta
  e un dispositivo Finder serie 6M.
* Imparare a utilizzare il protocollo di comunicazione Modbus RTU per
  configurare e leggere i registri di un dispositivo Finder serie 6M.

## Requisiti hardware e software

### Requisiti hardware

* PLC Finder Opta con supporto RS-485 (x1).
* Analizzatore di rete Finder serie 6M (x1).
* Alimentatore DIN rail 12VDC/500mA (x1).
* Cavo USB-C® (x1).
* Cavo per la connettività RS-485 con una delle seguenti specifiche (x2):
  * STP/UTP 24-18AWG (non terminato) con impedenza di 100-130Ω.
  * STP/UTP 22-16AWG (terminato) con impedenza di 100-130Ω.

### Requisiti software

* [Arduino IDE 1.8.10+](https://www.arduino.cc/en/software), [Arduino IDE
  2.0+](https://www.arduino.cc/en/software) o [Arduino Web
  Editor](https://create.arduino.cc/editor).
* Se si utilizza Arduino IDE offline, è necessario installare le librerie
  `ArduinoRS485` e `ArduinoModbus` utilizzando il Library Manager di Arduino
  IDE.
* [Codice di esempio](assets/Opta6MExample.zip).

## Finder serie 6M e il protocollo Modbus RTU

Gli analizzatori di rete Finder serie 6M forniscono accesso a una serie di
*holding registers* tramite il protocollo Modbus RTU su connessione seriale
RS-485.

Come documentato nel documento [Modbus communication
protocol](https://cdn.findernet.com/app/uploads/Modbus_RS485_6MTx.pdf), le
misure sui dispositivi Finder serie 6M sono disponibili su Modbus tramite una
serie di letture a 16 bit: ad esempio, la misura dell'energia è disponibile
come un valore a 32 bit ottenuto combinando la lettura dei due registri
adiacenti a 16 bit, situati agli indirizzi Modbus `40089` e `40090`. Si noti
che, per i dispositivi Finder serie 6M, tutti gli offset sono *register
offset*, non *byte offset*. Inoltre, sui dispositivi Finder serie 6M,
l'indirizzamento Modbus parte da `0`: questo significa che, ad esempio, bisogna
acceddere all'indirizzo Modbus `40006` come *holding register* numero `5`.

Per ulteriori informazioni sul protocollo di comunicazione Modbus, dai
un'occhiata a questo [articolo su
Modbus](https://docs.arduino.cc/learn/communication/modbus): tutte le
funzionalità fornite dalla libreria `ArduinoModbus` sono supportate da Finder
Opta.

## Istruzioni

### Configurazione dell'Arduino IDE

Per seguire questo tutorial, sarà necessaria [l'ultima versione dell'Arduino
IDE](https://www.arduino.cc/en/software). Se è la prima volta che configuri il
Finder Opta, dai un'occhiata al tutorial [Getting Started with
Opta](/tutorials/opta/getting-started).

Assicurati di installare la versione più recente delle librerie
[ArduinoModbus](https://www.arduino.cc/reference/en/libraries/arduinomodbus/) e
[ArduinoRS485](https://www.arduino.cc/reference/en/libraries/arduinors485/),
poiché verranno utilizzate per implementare il protocollo di comunicazione
Modbus RTU.

### Connessione tra Finder Opta e Finder serie 6M

Per osservare misurazioni effettive, sarà necessario collegare l'analizzatore
di rete Finder serie 6M alla rete elettrica e fornire un carico adeguato. Sarà
anche necessario alimentare il Finder Opta con un alimentatore da 12VDC/500mA e
configurare correttamente la connessione seriale RS-485. Il diagramma
sottostante mostra la configurazione corretta dei collegamenti tra i due
dispositivi.

![Connecting Opta and Finder 6M](assets/connection.svg)

### Configurazione dei parametri Modbus su Finder serie 6M

Per configurare i parametri Modbus iniziali del Finder serie 6M, dobbiamo
consultare pagina 6 del [manuale
utente](https://cdn.findernet.com/app/uploads/6M.Tx-User-Guide.pdf). Infatti,
la posizione degli switch DIP del Finder serie 6M determina la configurazione
Modbus del dispositivo.

Di solito, lo switch DIP 1 si troverà in posizione `UP` e il 2 si troverà in
posizione `DOWN`, il che determina una configurazione Modbus con indirizzo `1`
e baudrate `9600`. Tuttavia, in questo esempio i parametri iniziali di
comunicazione del Finder serie 6M sono:

* Indirizzo Modbus: `1`.
* Baudrate: `38400`.

Possiamo impostare questi valori **posizionando entrambi gli switch DIP del
Finder serie 6M alla posizione `UP`**.

Successivamente, quando richiesto dallo sketch, posizioneremo entrambi gli
switch DIP in posizione `DOWN`, in modo che il Finder serie 6M utilizzi la
configurazione Modbus custom assegnatali dallo sketch stesso.

### Panoramica del codice

Lo scopo del seguente esempio è configurare l'analizzatore di rete Finder serie
6M utilizzando il Finder Opta, e successivamente leggere alcune misurazioni dai
registri del 6M e stamparle su console seriale.

Il codice completo dell'esempio è disponibile [qui](assets/Opta6MExample.zip):
dopo aver estratto i file, è possibile compilare e caricare lo sketch sul
Finder Opta.

#### Configurazione del Finder serie 6M

Nel metodo `setup()` procediamo a:

* Configurare i parametri Modbus secondo la guida [Modbus over serial
  line](https://modbus.org/docs/Modbus_over_serial_line_V1_02.pdf).
* Impostare l'indirizzo Modbus e il Baudrate del nostro 6M.

Ricordiamo nuovamente che in questo esempio la posizioni iniziale di entrambi
switch DIP del Finder serie 6M è `UP`.

```cpp
#include <ArduinoModbus.h>
#include <ArduinoRS485.h>
#include "finder-6m.h"

constexpr uint8_t MODBUS_6M_DEFAULT_ADDRESS = 1;
constexpr uint8_t MODBUS_6M_ADDRESS = 3;

void setup()
{
    Serial.begin(BAUDRATE);

    RS485.setDelays(PREDELAY, POSTDELAY);
    ModbusRTUClient.setTimeout(TIMEOUT);
    if (ModbusRTUClient.begin(BAUDRATE, SERIAL_8N1) != 1)
    {
        while (1)
            ;
    }

    // Change Modbus address
    modbus6MWrite16(MODBUS_6M_DEFAULT_ADDRESS, FINDER_6M_REG_MODBUS_ADDRESS, MODBUS_6M_ADDRESS);
    // Baudrate 38400 has code 5
    modbus6MWrite16(MODBUS_6M_DEFAULT_ADDRESS, FINDER_6M_REG_BAUDRATE, FINDER_6M_BAUDRATE_CODE_38400);
    // Save above settings
    if (modbus6MWrite16(MODBUS_6M_DEFAULT_ADDRESS, FINDER_6M_REG_COMMAND, FINDER_6M_COMMAND_SAVE))
    {
        // We have 20 seconds to lower the DIP switches
        Serial.println("Waiting 20s while you:");
        Serial.println("1. Power OFF the 6M.");
        Serial.println("2. Set both DIP switches DOWN.");
        Serial.println("3. Power back ON the 6M.");
        delay(20000);
    }
    else
    {
        while (1)
            ;
    }
}
```

Il file `finder-6m.h` contiene tutte le definizioni necessarie, inclusi i
parametri Modbus e gli offset dei registri; si noti che l'esempio utilizza la
configurazione seriale `8-N-1`. Dopo aver salvato le impostazioni, lo sketch ci
lascia 20 secondi per configurare gli switch DIP del 6M in posizione `DOWN`, in
modo che il dispositivo utilizzi i parametri da noi configurati: le
istruzioni rilevanti verranno visualizzate su monitor seriale.

Tutti i valori di configurazione che scriviamo sul Finder serie 6M vengono
memorizzati su registri a 16 bit, quindi utilizzeremo la seguente funzione per
scriverli:

```cpp
boolean modbus6MWrite16(uint8_t address, uint16_t reg, uint16_t toWrite)
{
    uint8_t attempts = 3;
    while (attempts > 0)
    {
        if (ModbusRTUClient.holdingRegisterWrite(address, reg, toWrite) == 1)
        {
            return true;
        }
        else
        {
            attempts -= 1;
            delay(10);
        }
    }
    return false;
}
```

#### Lettura dal Finder serie 6M

Nella funzione `loop()` leggeremo le seguenti misurazioni dal Finder serie 6M
appena configurato e le stamperemo sulla console seriale:

* Frequenza (Hz/100).
* Potenza attiva (W/100).
* Potenza apparente (VA/100).
* Energia (kWh/100).

```cpp
void loop()
{
    int32_t frequency = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_FREQUENCY_100);
    int32_t activePower = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_ACTIVE_POWER_100);
    int32_t apparentPower = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_APPARENT_POWER_100);
    int32_t energy = modbus6MRead32(MODBUS_6M_ADDRESS, FINDER_6M_REG_ENERGY_100);

    Serial.println("   frequency = " + (frequency != INVALID_DATA ? String(frequency) : String("read error!")));
    Serial.println("   active power = " + (activePower != INVALID_DATA ? String(activePower) : String("read error!")));
    Serial.println("   apparent power = " + (apparentPower != INVALID_DATA ? String(apparentPower) : String("read error!")));
    Serial.println("   energy = " + (energy != INVALID_DATA ? String(energy) : String("read error!")));
}
```

Tutte le misurazioni in questo esempio sono lunghe 32 bit e vengono memorizzate
in registri di 16 bit utilizzando la notazione LSW-first, come indicato nella
documentazione del Finder serie 6M. Ciò significa che dobbiamo leggere due
registri di 16 bit adiacenti a partire dall'offset specificato e comporre la
misurazione, cosa che facciamo con la seguente funzione:

```cpp
uint32_t modbus6MRead32(uint8_t address, uint16_t reg)
{
    uint8_t attempts = 3;
    while (attempts > 0)
    {
        ModbusRTUClient.requestFrom(address, HOLDING_REGISTERS, reg, 2);
        uint32_t data1 = ModbusRTUClient.read();
        uint32_t data2 = ModbusRTUClient.read();
        if (data1 != INVALID_DATA && data2 != INVALID_DATA)
        {
            return data2 << 16 | data1;
        }
        else
        {
            attempts -= 1;
            delay(10);
        }
    }
    return INVALID_DATA;
}
```

## Utilizzo della libreria Finder6M

Per semplificare tutte le operazioni eseguite in questo tutorial, è possibile
utilizzare la libreria `Finder6M`. In questo caso, il codice di `setup()`
diventa molto più semplice, poiché la libreria fornisce funzioni integrate per
configurare i parametri RS-485 e il Finder serie 6M:

```cpp
#include <Finder6M.h>

Finder6M f6m;
constexpr uint8_t MODBUS_6M_DEFAULT_ADDRESS = 1;
constexpr uint8_t MODBUS_6M_ADDRESS = 3;

void setup()
{
    Serial.begin(38400);

    if (!f6m.init())
    {
        while (1)
            ;
    }

    // Change Modbus address
    f6m.setModbusAddress(MODBUS_6M_ADDRESS, MODBUS_6M_DEFAULT_ADDRESS);
    // Set baudrate to 38400
    f6m.setBaudrate(MODBUS_6M_DEFAULT_ADDRESS, 38400);
    // Save above settings
    if (f6m.saveSettings(MODBUS_6M_DEFAULT_ADDRESS))
    {
        Serial.println("Waiting 20s while you:");
        Serial.println("1. Power OFF the 6M.");
        Serial.println("2. Set both DIP switches DOWN.");
        Serial.println("3. Power back ON the 6M.");
        delay(20000);
    }
    else
    {
        while (1)
            ;
    }
}
```

Anceh il codice nel `loop()` diventa più semplice, e non è più necessario
scrivere funzioni per interagire con i registri:

```cpp
void loop()
{
    int32_t frequency = f6m.getFrequency100(MODBUS_6M_ADDRESS);
    int32_t activePower = f6m.getActivePower100(MODBUS_6M_ADDRESS);
    int32_t apparentPower = f6m.getApparentPower100(MODBUS_6M_ADDRESS);
    int32_t energy = f6m.getEnergy100(MODBUS_6M_ADDRESS);

    Serial.println("   frequency = " + (frequency != INVALID_DATA ? String(frequency) : String("read error!")));
    Serial.println("   active power = " + (activePower != INVALID_DATA ? String(activePower) : String("read error!")));
    Serial.println("   apparent power = " + (apparentPower != INVALID_DATA ? String(apparentPower) : String("read error!")));
    Serial.println("   energy = " + (energy != INVALID_DATA ? String(energy) : String("read error!")));
}
```

Per saperne di più sulla libreria, visita la [repository
ufficiale](https://github.com/dndg/Finder6M).

## Conclusioni

Questo tutorial mostra come utilizzare le librerie `ArduinoRS485` e
`ArduinoModbus` per implementare il protocollo Modbus RTU tra il Finder Opta e
un analizzatore di rete Finder serie 6M. Inoltre, mostra come sia possibile
utilizzare la libreria `Finder6M` per leggere facilmente le misurazioni da un
6M.
