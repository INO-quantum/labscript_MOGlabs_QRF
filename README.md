# labscript_MOGlabs_QRF
labscript-suite internal pseudoclock device for MOGlabs QRF

This is a labscript user device class which implements the MOGlabs Quad-RF (QRF) DDS module. It depends on the [MOGlabs mogdevice](https://pypi.org/project/mogdevice/). Please install this. Alternatively, you can copy the file `mogdevice.py` into the `labscript_MOGlabs_QRF` folder and import it in `blacs_workers.py`.

The first version is based on the [MOGlabs XRF source code](https://github.com/specialforcea/labscript_suite/blob/a4ad5255207cced671990fff94647b1625aa0049/labscript_devices/MOGLabs_XRF021.py). The actual implementation builds on the [internal pseudoclock device](https://github.com/INO-quantum/labscript_iPCdev) as base class. 

Please copy the entire `iPCdev` and `labscript_MOGlabs_QRF` folders into your `user_devices` folder. [Here](https://github.com/INO-quantum/labscript_MOGlabs_QRF/tree/main/example_experiment) you find the `connection_table` and an example experiment script. 

> [!Note]
> The first simple version was tested with hardware borrowed from a neighboring laboratory. The present version has been changed a lot but could not yet been tested with hardware! But the new device should arrive soon such that this can be tested. 

The implementation as an internal pseudoclock device (iPCdev) creates a separate device tab for each QRF device and offers more flexibility than an implementation as IntermediateDevice. The iPCdev part implements everything needed for labscript while the labscript_MOGlabs_QRF part takes care of the hardware communication in the worker and adds in the blacs tab the checkboxes to enable the RF signals and amplifiers. Each RF channel can be configured in different modes in the [connection_table.py](https://github.com/INO-quantum/labscript_MOGlabs_QRF/tree/main/example_experiment). The possible modes are:

| mode                               | description                                                                          |
|------------------------------------|--------------------------------------------------------------------------------------|
| 'NSB' or 'basic'                   | static programming of freq/amp/phase or from front panel values if nothing in script. RF on/off via digital gate. do not specify trigger_delay and trigger_duration in experiment script but use enable/disable             |
|------------------------------------|--------------------------------------------------------------------------------------|
| 'NSA' or 'advanced'                | direct programming of DDS registers. not imemented!                                  |
|------------------------------------|--------------------------------------------------------------------------------------|
| 'TSB' or 'table timed'             | table mode, timed by microcontroller of QRF. resolution 5us + microcontroller clock drift. started by external trigger attached to channel 1 trigger input given to QRF.__init__ as DO_trg with trigger_connection = channel.  |
|------------------------------------|--------------------------------------------------------------------------------------|
| 'TSB sw' or 'table timed software' | same as 'TSB' or 'table timed' but started by software trigger. in this mode several QRF's are not synchronized! |
|------------------------------------|--------------------------------------------------------------------------------------|
| 'TSB_trg' or 'table triggered'     | table mode, timed by external trigger given as digital_gate to QRF_DDS. resolution 5us with relative big jitter. |
|------------------------------------|--------------------------------------------------------------------------------------|

The example connection table is a bit complex since it is used also to test several configurations of type of other devices in the system: either FPGA_device or other iPCdev boards or stand-alone QRF. In a real experiment it would be simpler since only one of these configuration is used.

When you need to switch on/off the RF fast configure the channel into 'basic' mode and attach a TTL signal to the trigger input and give the signal channel as the `connection` entry for the `digital_gate` dictionary given to QRF_DDS.__init__. The `device` entry depends on the device class used to generate the digital channel. If it is `FPGA_device` with `DigitalChannels` you have to give the DigitalChannels (`DO_trg`), otherwise give the iPCdev parent device of the channel (`prim`).

When you need to change the frequency, amplitude or phase of the channels during runtime you have several `table mode` options with different advantages and disadvantages. In this mode frequency, amplitude and phase and optionally also time are uploaded as a `table` into the memory of the QRF module and this `table` can then be executed in different modes.

In the `table timed` mode the time is uploaded onto the QRF module with 5us resolution and the execution is timed by the microcontroller of the QRF. It needs a `start trigger` which is generated by an external TTL signal attached on channel 1 of the QRF and specified with trigger_connection on the QRF.__init__ function. Give as the parent device of the QRF an Intermediate device (DO_trg) and as trigger_connection the channel connection used to identify the trigger channel (format depends on the type of device used). The trigger channel must be always given but the iPCdev class allows to use a virtual device (see hardware_trigger = False option). In this mode the timing except of the start is given by the internal QRF frequency stability. Although the QRF can be locked to an external RF source, as far as I have understood this only locks the RF output frequencies but not the microcontroller frequency. Therefore, this mode is not recommended for long (say > 1s) sequences.

The `table timed software` is the same as `table timed` but does not need a `start trigger` since it will be started by software as soon as all data is uploaded. So this mode might even be less useful, maybe for standalone systems or for simple tests it might be ok.

The last mode is `table triggered` where no time is uploaded on the QRF but for each channel in this mode a digital_gate TTL signal must be connected (and specified as for the `basic` mode). For each pulse of the gate the table is advanced by one step. The resolution is 5us but there is quite some jitter (order of 5us as far as I remember) due to the low sampling rate of the microcontroller. This is the most flexible mode but is only useful when no precise timing is needed. However, one can sychronize the microcontroller with an additional start trigger - but which was not yet implemented in software or tested. Maybe this can help to improve the situation.

The 'advanced' mode should allow to directly program the DDS which could in principle generate also linear ramps. This has not been tested but might be an interesting option for the future. However, if this is useful one would need to trigger the DDS directly with an external signal.


