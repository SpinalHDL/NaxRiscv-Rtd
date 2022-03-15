.. role:: raw-html-m2r(raw)
   :format: html

NaxRiscv
==========


NaxRiscv is a core currently characterised by : 

- Out of order execution with register renaming
- Superscalar (ex : 2 decode, 3 execution units, 2 retire)
- (RV32 RV64)IMASU (Linux / Buildroot works on harwdare)
- Portable HDL, but target FPGA with distributed ram (Xilinx series 7 is the reference used so far)
- Target a (relatively) low area usage and high fmax (not the best IPC)
- Decentralized hardware elaboration (Empty toplevel parametrized with plugins)
- Frontend implemented around a pipelining framework to ease customisation
- Non-blocking Data cache with multiple refill and writeback slots
- BTB + GSHARE + RAS branch predictors
- Hardware refilled MMU (SV32, SV39)
- Load to use latency of 3 cycles via the speculative cache hit predictor 
- Pipeline visualisation via verilator simulation and Konata (gem5 file format)

Project developpement and status
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- This project is free and open source
- It can run upstream buildroot/linux on hardware (ArtyA7-35T / Litex)
- It started in October 2021 without any funding
- Currently looking for funding to implement more features (FPU, SMP, ...)
- Unfortunately the project started with a single crew (not by wish), contribution are welcome.

Why a OoO core targeting FPGA
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There is a few reasons

- Improving single threaded performance. 
  During the tests made with VexRiscv running linux, it was clear that even if the can multi core help, "most" applications aren't made to take advantage of it. 
- Hiding the memory latency (There isn't much memory to have a big L2 cache on FPGA)
- To experiment with more advanced hardware description paradigms (scala / SpinalHDL)
- By personnal interest

Also there wasn't many OoO opensource softcore out there in the wild (Marocchino, RSD, OPA, ..). 
The bet was that it was possible to do better in some metrics, and hopfully being good enough to justify in some project
the replacement of single issue / in order core softcore by providing better performances (at the cost of area).


