.. role:: raw-html-m2r(raw)
   :format: html

Load store unite
============================

The LSU implementation is characterised by :

- **LQ / SQ** : Usually, 16 of each
- **Load from AGU** : To reduce the load latency, if the LQ has nothing for the load pipeline, then the AGU can directly provide its fresh calculation without passing by the LQ registers
- **Load Hit speculation** : In order to reduce the load to use latency (to 3 cycles instead of 6), there is a cache hit predictor, speculatively waking up depending instructions
- **Hazard prediction** : For store, both address and data are provided through the issue queue. So, a late data will also create a late address, potential creating store to load hazard. To reduce that occurence, a hazard predictor was added to the loads.
- **Store to load bypass** : If a given load depend on a single store of the same size, then the load pipeline may bypass the store value instead of waiting for the store writeback
- **Parallel memory translation** : For loads, to reduce the latency, the memory translation run in parallel of the cache read.
- **Shared address pipeline** : Load and store use the same pipeline to translate the virtual address and check for hazards.

Here is a few illustrations of the shared and the writeback pipeline :

.. image:: /asset/image/lsu2.png

