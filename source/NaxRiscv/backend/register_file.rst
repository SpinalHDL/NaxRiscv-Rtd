.. role:: raw-html-m2r(raw)
   :format: html

Register file
============================

Currently, the register file is infered into simple dual ports distributed ram : 

- Each write port will create its own bank
- Each read port will read each bank, and mux the correct one using a distributed-ram-xor-based LVT (live value table)



