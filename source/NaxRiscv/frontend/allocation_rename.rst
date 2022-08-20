.. role:: raw-html-m2r(raw)
   :format: html

Physical register allocation
========================================================

This is done by implementing a circular buffer containing the indexes of unallocated physical registers. This fits well into an FPGA with distributed ram.

Architectural to physical
========================================================

The translation from architectural register file to physical is done by implementing three tables :

- **Speculative mapping** : Translate from architectural to physical, updated after the instruction decoding, implemented in distributed ram
- **Committed mapping** : Translate from architectural to physical, updated after the instruction commit, implemented in distributed ram
- **Location** : Translate from architectural to which mapping should be used (speculative or committed), implemented as register (need to be cleared on branch misprediction)

This allows to revert the state of the translation instantly when the pipeline predicted a branch wrong.

.. image:: /asset/image/rf_translation.png

Physical to ROB ID
========================================================

Once the physical register file of the dependencies is calculated, they are translated into the ROB ID on which it depends. This is done by two things :

- **ROB Mapping** : A distributed ram which translates from physical to ROB ID
- **Busy** : Which specify if the given ROB ID is still executing. It is set when a instruction is dispatched, cleared when the an instruction completes.