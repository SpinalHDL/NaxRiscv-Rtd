===============
Execution units
===============

The main characteristic of an execution unit is the way in which it will wake up the instruction which depend on it.

- **Static wake** : For EU with fixed latency (no stall), a static wake can be asked to the issue queue. This can reduce the completion-to-use delay by two cycles
- **Dynamic wake** : For EU with variable latency (which can stall), the EU is responsible to send a wakes commands to the issue queue.

Here is an illustration with two EU (one with static wake, one with dynamic wake) interacting with the issue queue to wake up dependent instructions

.. image:: /asset/image/eu_wakes.png

.. include:: custom.rst


.. role:: raw-html-m2r(raw)
   :format: html



