.. role:: raw-html-m2r(raw)
   :format: html

Load store unite
============================

The LSU implementation is characterised by : 

- **LQ / SQ** : Usualy, 16 of each
- **Store Address / Data** : For store, only the address is provided through the issue queue / AGU, while there is some logic from the SQ to fetch the store value directly from the register file.
- **Load Hit speculation** : In order to reduce the load to use latency (to 3 cycles instead of 6), there is a cache hit predictor, speculatively waking up depending instructions
- **Load from AGU** : To reduce the load latency, if the LQ has nothing for the load pipeline, then the AGU can directly provide its fresh calculation without passing by the LQ registers
- **Store to load bypass** : If a given load depend on a single store of the same size, then the load pipeline may bypass the store value instead of waiting for the store writeback
- **Parallel memory translation** : For loads, to reduce the latency, the memory translation run in parallel of the cache read.

Here is a few illustrations of the load and store pipeline :

.. image:: /asset/image/load_pipeline.png

.. image:: /asset/image/store_pipeline.png
