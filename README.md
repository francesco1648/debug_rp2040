# Guida: Connettere un Raspberry Pi Pico come PicoProbe Debug ad un altro Raspberry Pi Pico  
*Tutorial per Windows*
English version below
---

## Prerequisiti

- **Visual Studio Code**  
  Installare VS Code e le seguenti estensioni:  
  - CMake  
  - C/C++  
  - Cortex Debug

- **Pico SDK**  
  - Scaricare il Pico SDK dal repository Git:  
    [https://github.com/raspberrypi/pico-sdk](https://github.com/raspberrypi/pico-sdk)  
  - Copiare la cartella in, ad esempio, `C:\Program Files\Raspberry_Pi\pico-sdk`
  - Aggiungere la variabile di sistema:
    - **Nome:** `PICO_SDK_PATH`
    - **Valore:** `C:\Program Files\Raspberry_Pi\pico-sdk`

- **GNU Arm Embedded Toolchain**  
  - Scaricare da:  
    [https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain)  
  - Se non configurato automaticamente, aggiungere il percorso `arm-none-eabi\14.2 rel1\bin` alla variabile di sistema `Path`.

- **Zadig**  
  Scaricare Zadig da:  
  [https://zadig.akeo.ie/](https://zadig.akeo.ie/)

- **OpenOCD**  
  - Scaricare OpenOCD (preferibilmente una versione precompilata).  
  - Nella cartella di riferimento è già presente una versione testata di OpenOCD, se si desidera usarla.

---

## Preparazione del PicoProbe

1. **Modalità BOOTSEL e caricamento firmware**  
   - Mettere il Pico (che fungerà da probe) in modalità BOOTSEL.  
   - Caricare il file `debugprobe_on_pico.uf2`.  
   - Se si desidera utilizzare una versione più recente, controllare le release su:  
     [https://github.com/raspberrypi/debugprobe/releases](https://github.com/raspberrypi/debugprobe/releases)

2. **Collegamento del Pico Target al PicoProbe**

   Collega i pin come segue:

   | **PicoProbe** | **Pico Target**       |
   |---------------|-----------------------|
   | GND           | GND                   |
   | GND           | GND (DEBUG)           |
   | GP2           | SWCLK (DEBUG)         |
   | GP3           | SWDIO (DEBUG)         |
   | GP4           | GP1                   |
   | GP5           | GP0                   |
   | VBUS          | VBUS **oppure** VSYS → VSYS |

3. **Configurazione di Zadig**  
   - Collega il PicoProbe al PC tramite cavo USB.  
   - Apri Zadig e seleziona **Options > List All Devices**.
   - Associa:
     - **cmsis-dap v2** → `libusb-win32`
     - **cdc-acm uart interface** → `libwdi`
   - In **Gestione dispositivi**:  
     Vai su *Porte (COM & LPT) > cdc-acm uart interface > Proprietà > Impostazioni della porta* e imposta il **bit per secondo** a `1280000`.

---

## Configurazione e Debug

### Avviare OpenOCD

1. **Apri due terminali**.

2. **Terminale 1 (cmm1):**  
   - Naviga nella cartella `open_ocd/bin`.
   - Avvia OpenOCD con il comando:
     ```bash
     openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg -c "adapter speed 2000"
     ```
   - Dovresti vedere un output simile a:
     ```
     xPack Open On-Chip Debugger 0.12.0+dev-... (data e ora)
     Licensed under GNU GPL v2
     ...
     Info : adapter speed: 2000 kHz
     ...
     Info : [rp2040.core0] starting gdb server on 3333
     ...
     ```
   - **Nota:** Non chiudere questo terminale.

3. **Terminale 2 (cmm2):**  
   - Posizionarsi nella directory contenente lo script da testare (ad esempio, un semplice blink).
   - Avvia `gdb` con il comando:
     ```bash
     arm-none-eabi-gdb blink.ino.elf
     ```
   - All'interno di GDB, esegui i seguenti comandi:
     ```gdb
     target remote localhost:3333
     load
     monitor reset init
     continue
     ```
   - Se tutto è configurato correttamente, il LED del Pico Target dovrebbe iniziare a lampeggiare e non verranno mostrati errori.

---

## Configurazione di Visual Studio Code

1. **Impostazioni Globali**  
   - In basso a sinistra in VS Code, vai su **Settings**.
   - Clicca sull'icona della pagina per aprire il file `settings.json`.
   - Aggiungi (ciò che si trova () risulta essere un esempio) le seguenti configurazioni:
     ```json
     {
       "cortex-debug.openocdPath": "...PATH TO OPENOCD.EXE... (es. open_ocd/bin/openocd.exe)",  // Modifica con il percorso corretto
       "cortex-debug.gdbPath": "arm-none-eabi-gdb"
     }
     ```

2. **Configurazione del Progetto**  
   - All'interno della cartella del progetto, crea (se non esiste) una cartella denominata `.vscode`.
   - All'interno di `.vscode`, crea un file chiamato `launch.json` con il seguente contenuto:
     ```json
     {
         "version": "0.2.0",
         "configurations": [
             {
                 "name": "Cortex Debug",
                 "cwd": "${workspaceRoot}",
                 "executable": "${workspaceRoot}/build/blink.ino.elf",  // Modifica con il percorso corretto
                 "request": "launch",
                 "type": "cortex-debug",
                 "servertype": "openocd",
                 "device": "RP2040",
                 "runToMain": true,
                 "configFiles": ["interface/cmsis-dap.cfg", "target/rp2040.cfg"],
                 "searchDir": ["...PATH TO SCRIPTS... (es. open_ocd/openocd/scripts)"],                      // Modifica con il percorso corretto
                 "svdFile": "C:/Program Files/Raspberry_Pi/pico-sdk/src/rp2040/hardware_regs/rp2040.svd",    // Modifica con il percorso corretto
                 "setupCommands": [
                     { "text": "adapter speed 2000" },
                     { "text": "transport select swd" },
                     { "text": "init" },
                     { "text": "reset init" },
                     { "text": "arm semihosting enable" },
                     { "text": "monitor reset init" },
                     { "text": "monitor reset halt" },
                     { "text": "monitor halt" },
                     { "text": "monitor reset halt" },
                     { "text": "monitor halt" }
                 ]
             }
         ]
     }
     ```

3. **Avvio del Debug**  
   - Salva tutte le modifiche.
   - In VS Code, vai alla sezione **Debug** (icona del bug/play) e seleziona il profilo **Cortex Debug**.
   - Avvia il debug cliccando sul triangolo verde.

> **Nota:** Se compare un errore del tipo "failed to launch gdb...", prova a chiudere l'errore, scollega e ricollega la connessione USB, quindi riavvia il debug.

---

## Conclusioni

Seguendo attentamente tutti i passaggi sopra descritti, dovresti essere in grado di configurare correttamente un Raspberry Pi Pico come PicoProbe per il debug di un altro Pico.  
Buon debug!


//-------------------------------------------------------------------------------------------------

# Guide: Connecting a Raspberry Pi Pico as PicoProbe Debug to Another Raspberry Pi Pico  
*Tutorial for Windows*

---

## Prerequisites

- **Visual Studio Code**  
  Install VS Code along with these extensions:  
  - CMake  
  - C/C++  
  - Cortex Debug

- **Pico SDK**  
  - Download the Pico SDK from GitHub:  
    [https://github.com/raspberrypi/pico-sdk](https://github.com/raspberrypi/pico-sdk)  
  - Place the folder in a directory, for example:  
    `C:\Program Files\Raspberry_Pi\pico-sdk`
  - Set up an environment variable:
    - **Variable Name:** `PICO_SDK_PATH`
    - **Variable Value:** `C:\Program Files\Raspberry_Pi\pico-sdk`

- **GNU Arm Embedded Toolchain**  
  - Download it from:  
    [https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain)  
  - If not configured automatically, add the path (e.g., `arm-none-eabi\14.2 rel1\bin`) to your system's `Path` variable.

- **Zadig**  
  Download Zadig from:  
  [https://zadig.akeo.ie/](https://zadig.akeo.ie/)

- **OpenOCD**  
  - Download OpenOCD (a precompiled version is recommended).  
  - Alternatively, you may use the tested version provided in the specified folder.

---

## Setting Up the PicoProbe

1. **BOOTSEL Mode and Firmware Flashing**  
   - Put the Pico (which will act as the probe) into BOOTSEL mode.
   - Flash the file `debugprobe_on_pico.uf2` onto it.  
   - To use a more recent version, check the releases at:  
     [https://github.com/raspberrypi/debugprobe/releases](https://github.com/raspberrypi/debugprobe/releases)

2. **Connecting the Pico Target to the PicoProbe**

   Connect the pins as follows:

   | **PicoProbe** | **Pico Target**            |
   |---------------|----------------------------|
   | GND           | GND                        |
   | GND           | GND (DEBUG)                |
   | GP2           | SWCLK (DEBUG)              |
   | GP3           | SWDIO (DEBUG)              |
   | GP4           | GP1                        |
   | GP5           | GP0                        |
   | VBUS          | VBUS **or** VSYS → VSYS    |

3. **Configuring Zadig**  
   - Connect the PicoProbe to the PC via USB.
   - Open Zadig and select **Options > List All Devices**.
   - Assign the drivers as follows:
     - **cmsis-dap v2** → `libusb-win32`
     - **cdc-acm uart interface** → `libwdi`
   - In **Device Manager**:  
     Navigate to *Ports (COM & LPT) > cdc-acm uart interface > Properties > Port Settings* and set the **bits per second** to `1280000`.

---

## Debugging Setup

### Starting OpenOCD

1. **Open Two Terminals**

2. **Terminal 1 (OpenOCD)**  
   - Navigate to the `open_ocd/bin` directory.
   - Launch OpenOCD with the command:
     ```bash
     openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg -c "adapter speed 2000"
     ```
   - You should see output similar to:
     ```
     xPack Open On-Chip Debugger 0.12.0+dev-... (timestamp)
     Licensed under GNU GPL v2
     ...
     Info : adapter speed: 2000 kHz
     ...
     Info : [rp2040.core0] starting gdb server on 3333
     ...
     ```
   - **Important:** Do not close this terminal.

3. **Terminal 2 (GDB Session)**  
   - Navigate to the folder containing your test script (for example, a simple blink project).
   - Start GDB with:
     ```bash
     arm-none-eabi-gdb blink.ino.elf
     ```
   - Within GDB, run the following commands:
     ```gdb
     target remote localhost:3333
     load
     monitor reset init
     continue
     ```
   - If everything is set up correctly, the LED on the Pico Target should start blinking without errors.

---

## Configuring Visual Studio Code

1. **Global Settings**  
   - In VS Code, go to **Settings** (bottom left) and open the `settings.json` file by clicking the icon resembling a document.
   - Add or modify the following configurations:
     ```json
     {
       "cortex-debug.openocdPath": "...PATH TO OPENOCD.EXE... (e.g., open_ocd/bin/openocd.exe)",
       "cortex-debug.gdbPath": "arm-none-eabi-gdb"
     }
     ```

2. **Project-Specific Configuration**  
   - Inside your project folder, create a folder named `.vscode` (if it doesn’t already exist).
   - Within `.vscode`, create a file named `launch.json` with the following content:
     ```json
     {
         "version": "0.2.0",
         "configurations": [
             {
                 "name": "Cortex Debug",
                 "cwd": "${workspaceRoot}",
                 "executable": "${workspaceRoot}/build/blink.ino.elf",  // Adjust the path accordingly
                 "request": "launch",
                 "type": "cortex-debug",
                 "servertype": "openocd",
                 "device": "RP2040",
                 "runToMain": true,
                 "configFiles": ["interface/cmsis-dap.cfg", "target/rp2040.cfg"],
                 "searchDir": ["...PATH TO SCRIPTS... (e.g., open_ocd/openocd/scripts)"],
                 "svdFile": "C:/Program Files/Raspberry_Pi/pico-sdk/src/rp2040/hardware_regs/rp2040.svd",
                 "setupCommands": [
                     { "text": "adapter speed 2000" },
                     { "text": "transport select swd" },
                     { "text": "init" },
                     { "text": "reset init" },
                     { "text": "arm semihosting enable" },
                     { "text": "monitor reset init" },
                     { "text": "monitor reset halt" },
                     { "text": "monitor halt" },
                     { "text": "monitor reset halt" },
                     { "text": "monitor halt" }
                 ]
             }
         ]
     }
     ```

3. **Start Debugging**  
   - Save all changes.
   - In VS Code, open the **Debug** panel and select the **Cortex Debug** configuration.
   - Click the green triangle to start the debug session.

> **Note:** If you encounter an error such as "failed to launch gdb...", try closing the error message, unplug the USB cable, plug it back in, and restart the debug process.

---

## Conclusion

By following the steps above, you should be able to set up a Raspberry Pi Pico as a PicoProbe for debugging another Pico successfully. Happy debugging!
