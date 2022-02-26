.. role:: raw-html-m2r(raw)
   :format: html

MMU
============================

The MMU implementation is characterised by : 

- **2D organisation** : For each level of the pages table a parameterable number of direct mapped ways of translation cache can be specified.
- **Hardware refilled** : Because that's cheap
- **Caches direct hit** : Allows the instruction cache to check his way tags directly against the MMU TLB storage in order to improve timings (at the cost of area)

For RV32, the default configuration is to have :

- 4 ways * 32 entries of level 0 (4KB pages) TLB
- 2 ways * 32 entries of level 1 (4MB pages) TLB

The area of the TLB cache is keept low by inferring each way into a distributed ram.

Here is a few illustrations of the MMU design

.. image:: /asset/image/mmu_general.png

.. image:: /asset/image/mmu_translation.png
