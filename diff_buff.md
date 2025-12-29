TSMC 12nm 0.8v core voltage 9M. 
TSMC 12nm I/O Voltage: 1.2 V / 1.8 V.
1.2v/1.8v HV devices are used in ESD/IO but the entire diff_buff design's power domain is vddp 0.8v
Max. allowed voltage is 0.9v. Do not overstress during testing.
Can share vddp 0.8v with vp 0.8v of PCIe PHY block on substrate.
Note: IO is bi-directional thus diff_buff instance should always set to TX_mode as PCIe Host Root Complex.
