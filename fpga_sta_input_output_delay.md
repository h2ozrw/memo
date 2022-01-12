##  FPGA: STA: set_input_delay and set_output_delay

##  Input/Output Delay

`set_input_delay` defines the allowed range of delays of the data toggle after a clock.
-   `set_input_delay -clock clk_i -max Ti_max`: The maximal clock-to-output of the sending chip.
-   `set_input_delay -clock clk_i -min Ti_min`: The minimal clock-to-output of the sending chip.

E.g. Data is **Invalid** from *Ti_min* to *Ti_max* after the edge of *clk_i*.

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

E.g. Data is **Valid** from *To_max* to *To_min* before the edge of *clk_o*.

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
-   STA looks at the **un-propagated clock** (the create_clock defined one), for launch/capture edge.
-   Input/Output delay is defined to the edge of the *un-propagated clock*.
-   Capture edge is the clock edge following the launch edge of the *un-propagated clock*.
-   Setup time is checked from Launch edge to Capture edge.
-   Hold time is checked from Launch edge to the edge before the Capture edge.

```
SDR Mode, DDR is the same.

   v-- Launch Edge
   +------------------+                  +------------------+
   |                  |                  |                  |
---+                  +------------------+                  +----

                                         v-- Capture Edge
   +------------------+                  +------------------+
   |                  |                  |                  |
---+                  +------------------+                  +----
   ^-- Hold Check                        ^-- Setup Check



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
-   2 clocks are preferred.
    -   Input delay: 1 virtual clock for launching FF, 1 primary clock at FPGA clock pin.
    -   Output delay: 1 primary clock from FPGA, 1 generated clock at FPGA clock output pin for capture FF.

-   Add small input/output delay to capture clock is the simplest way, for Source Sync case:

        # input delay, FPGA clock
        create_clock -period 8.00 -waveform {0.002 4.002} [get_ports -prop_thru_buffers XX]

        # output delay, CHIP clock
        create_generated_clock -source [get_ports XX] -edges {1 2 3} -edge_shift {0.002 0.002 0.002} [get_ports YY]

-   The complex way for Source Sync case:
    1.  Analyse which edge path is correct: launch R/F edge to capture R/F edge.

    2.  Adjust capture edge for setup time, e.g. for DDR, may looks like:

            set_multicycle_path 0 -from [get_clocks clk_virt] -to [get_clocks clk_fpga]

            set_false_path -setup -rise_from [get_clocks clk_virt] -fall_to [get_clock clk_fpga]
            set_false_path -setup -fall_from [get_clocks clk_virt] -rise_to [get_clock clk_fpga]

    3.  Adjust edge for hold time, e.g.

            set_multicycle_path -1 -hold -from [get_clocks clk_virt] -to [get_clocks clk_fpga]

            set_false_path -hold -rise_from [get_clocks clk_virt] -rise_to [get_clock clk_fpga]
            set_false_path -hold -fall_from [get_clocks clk_virt] -fall_to [get_clock clk_fpga]

##  Reference Page
-   https://zhuanlan.zhihu.com/p/31585375

    It comes from Xilinx forum, but original post seems lost.

-   http://billauer.co.il/blog/2017/04/io-timing-constraints-meaning
