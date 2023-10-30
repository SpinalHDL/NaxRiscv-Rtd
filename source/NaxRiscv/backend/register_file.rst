.. role:: raw-html-m2r(raw)
   :format: html

Register file
============================

The default register file is the dual-port based one

Dual-port based
------------------

This configuration fit well on FPGA.

- Each write port will create its own bank
- Each read port will read each bank, and mux the correct one using a distributed-ram-xor-based LVT (live value table)

Latch based
------------------

This configuration fit well on ASIC. It's implementation was inspired from ibex, with the addition of multi write ports support.


.. image:: /asset/image/rf_latch.png
