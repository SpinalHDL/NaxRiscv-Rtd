.. role:: raw-html-m2r(raw)
   :format: html

Dispatch / Issue
============================

Here are a few specific points about the current implementation :

- **Unified design** : Mostly to save area / having the most usage of each entry
- **2D queue** : The entries arranged in C=decodeCount columns L=slotCount/decodeCount rows
- **Row push** : When something is pushed into the queue, a whole row is "consumed", even if the row isn't fully used
- **No compression** : There is no compression for empty rows. The while queue is shifted by one row on each push. It allows for better inference of the matrix FF and a smaller/faster ROB ID wake logic.
- **Matrix based** : The storage of which instruction depends on what is done as a half matrix
- **Older first** : If multiple instructions can be dispatched at once on a given execution unit, the older one is selected
- **Wake by ROB ID** : For dynamic wakes, the ROB ID is used as the identifier (not the physical register file ID)

So, overall, a 32 slot queue seems to be a limit to not go beyond to preserve the timings. Also, with the current design, the area occupancy of the queue doesn't seem to to be a big deal compared to the CPU as a whole.

Here are a few illustrations :

.. image:: /asset/image/iq_push_pop.png

.. image:: /asset/image/iq_wake_logic_nc.png

.. image:: /asset/image/iq_wake_logic_storage.png

.. image:: /asset/image/iq_issue_logic.png

