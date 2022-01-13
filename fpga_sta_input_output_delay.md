##  FPGA: STA: set_input_delay and set_output_delay

##  Input/Output Delay

`set_input_delay` defines the allowed range of delays of the data toggle after a clock.
-   `set_input_delay -clock clk_i -max Ti_max`: The maximal clock-to-output of the sending chip.
-   `set_input_delay -clock clk_i -min Ti_min`: The minimal clock-to-output of the sending chip.

E.g. **Data is Invalid** from *Ti_min* to *Ti_max* after the edge of *clk_i*.
In following direction, **Ti_min > 0** and **Ti_max > 0**.

```
            +------------------+                  +--------------
            |                  |                  |
------------+                  +------------------+

            |--- Ti_min -->|
--------------------------x            x-------------------------
                           x----------x
--------------------------x            x-------------------------
            |--- Ti_max            -->|
```

`set_output_delay` defines the range of delays of the clock after a data toggle.
-   `set_output_delay -clock clk_o -max To_max`: The setup time of the receiving chip.
-   `set_output_delay -clock clk_o -min To_min`: **Minus** The hold time of the receiving chip.

E.g. **Data is Valid** from *To_max* to *To_min* before the edge of *clk_o*.
In following direction, **To_max > 0** and **-To_min > 0**.

```
   +------------------+                  +------------------+
   |                  |                  |                  |
---+                  +------------------+                  +----

                          |<-- To_max ---|--- -To_min -->|
                           x----------------------------x
--------------------------x                              x-------
                           x----------------------------x
```

##  STA for Input/Output Delay

STA uses *un-propagated clock* (where it is defined) to decide launch/capture edge. The clock used is shown as below.

| Delay  |       Launch Clock       |       Capture Clock       |
|:------:|:------------------------:|:-------------------------:|
| Input  | clock of set_input_delay |   decided by capture FF   |
| Output |   decided by launch FF   | clock of set_output_delay |

PLL/MMCM input/output clocks are different. They may have same timing property, but they are defined at different point.

| Delay  | Capture Clock Path               | Clock Defined At   | Clock Type (Typical) |
|--------|----------------------------------|--------------------|----------------------|
| Input  | Chip Pin -> FF                   | Chip Pin           | Primary              |
| ^      | Chip Pin -> PLL -> FF            | Output Pin of PLL  | Generated            |
| Output | External Clock -> FF             | Clock Source       | Virtual/Primary      |
| ^      | Internal Clock -> Chip Pin -> FF | Output Pin of Chip | Generated            |

Capture edge is the clock edge following the Launch edge of the *un-propagated clock*.
And Input/Output Delay is defined to the Launch/Capture edge.

Setup time is checked from Launch edge to Capture edge.
Hold time is checked from Launch edge to the edge before the Capture edge.

```
SDR Mode, DDR is similar.

Edge Aligned:

   v-- Launch Edge
   +------------------+                  +------------------+
   |                  |                  |                  |
---+                  +------------------+                  +----

                                         v-- Capture Edge
   +------------------+                  +------------------+
   |                  |                  |                  |
---+                  +------------------+                  +----
   ^-- Hold Check                        ^-- Setup Check


Center Aligned:

                                         v-- Launch Edge
   +------------------+                  +------------------+
   |                  |                  |                  |
---+                  +------------------+                  +----

                                             v-- Capture Edge
       +------------------+                  +------------------+
       |                  |                  |                  |
-------+                  +------------------+                  +----
       ^-- Hold Check                        ^-- Setup Check
```

##  Implementation Notes

2 clocks are preferred for Input/Output Delay.
-   Input delay: 1 virtual clock for launching FF; 1 primary clock at chip clock pin.
-   Output delay: 1 primary clock for Chip; 1 generated clock at output pin for capture FF.

For Edge-Aligned Source Synchronous input/output, there may be simple way for constraining:
-   Input delay, primary clock, direct to capture FF:

        # defined clock is the capture clock.
        # clock delay change "Edge Aligned" to "Center Aligned".
        create_clock -period 10.00 -waveform {0.001 5.001} [get_ports -prop_thru_buffers XX]

-   Output Delay, internal clock:

        # output delay, internal clock as source.
        # define clock at output pin and set as "Center Aligned".
        create_generated_clock -source [get_ports XX] -edges {1 2 3} -edge_shift {0.001 0.001 0.001} [get_ports YY]

The Complete Process shall be:
1.  Define 2 clocks for Input/Output Delay.

1.  Analyse which path is correct: Launch R/F edge to Capture R/F edge.

1.  Adjust capture edge for setup time, using set_multicycle_path and set_false_path, if required.
    E.g. for DDR, it may looks like:

        set_multicycle_path 0 -from [get_clocks clk_virt] -to [get_clocks clk_chip]

        set_false_path -setup -rise_from [get_clocks clk_virt] -fall_to [get_clock clk_chip]
        set_false_path -setup -fall_from [get_clocks clk_virt] -rise_to [get_clock clk_chip]

1.  Adjust capture edge for hold time, using set_multicycle_path and set_false_path, if required.
    E.g. for DDR, it may looks like:

        set_multicycle_path -1 -hold -from [get_clocks clk_virt] -to [get_clocks clk_chip]

        set_false_path -hold -rise_from [get_clocks clk_virt] -rise_to [get_clock clk_chip]
        set_false_path -hold -fall_from [get_clocks clk_virt] -fall_to [get_clock clk_chip]

##  Reference Page
-   https://zhuanlan.zhihu.com/p/31585375

    It comes from Xilinx forum, but original post seems lost.

-   http://billauer.co.il/blog/2017/04/io-timing-constraints-meaning

    Description for set_input_delay and set_output_delay definitions.

-   https://support.xilinx.com/s/article/59893?language=en_US

    Description for set_input_delay, clock through PLL cases.
