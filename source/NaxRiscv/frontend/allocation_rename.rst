.. role:: raw-html-m2r(raw)
   :format: html

Physical register allocation
========================================================

This is done by implementing a circular buffer containing the indexes of unallocated physical register. This fit well in FPGA with distributed ram.

Architectural to physical
========================================================

The translation from architectural register file to physical is done by implementing three tables : 

- **Speculative mapping** : Translate from architectural to physical, updated after the instruction decoding, implemented in distributed ram
- **Commited mapping** : Translate from architectural to physical, updated after the instruction commit, implemented in distributed ram
- **Location** : Translate from architectural to which mapping should be used (speculative or commited), implemented as register (need to be cleared on branch missprediction)

This allows to revert the state of the translation instantly when the pipeline did a wrong branch prediction.

.. image:: /asset/image/rf_translation.png

Physical to ROB ID
========================================================

Once the physical register file of the depedencies is calculated, they are translated into the ROB ID on which it depends. This is done by two things : 

- **ROB Mapping** : A distributed ram which translate from physical to ROB ID
- **Busy** : Which specify if the given ROB ID is still executing. It is set when a instruction is dispatched, cleared when the a instruction complete.