.. role:: raw-html-m2r(raw)
   :format: html

Security
========================

Side channel attack
---------------------

You can find an example of side channel attack here (need NaxRiscv hardware protection to be disabled): 
https://github.com/SpinalHDL/NaxSoftware/tree/main/baremetal/side_channel/src
https://github.com/SpinalHDL/NaxSoftware/blob/main/baremetal/side_channel/src/side_channel.S

It allows in user mode to read the data from the supervisor. Bypassing page fault.

It is based on a few things 

- Speculative execution
- Load page fault still writting in the register file the readed value
- Load dependencies being waked up by the load cache hit predictor
- Using that rogue value to produce a cache line refill
- Mesuring which cache line is loaded to figure out the secret value

The image bellow show the execution of the attack (in grayish color is the speculative execution which wasn't commited) : 

.. image:: /asset/image/side_channel_timings.png

1) Call the attack function
2) Execute some slow code (store -> load value forwarding) to delay the commit / flush of the pipeline
3) Having a misspredicted branch to avoid commiting the faulty code
4) Attacking the supervisor memory with a load
5) Producing a cache load at an address which depend on the readed supervisor memory (mem[(data >> bitId) << 6]
6) Back on track, the cpu rolled back all the speculative execution, but the step 5 got a specific cache line loaded

Then after that execution, mesuring which cache line is loaded will tell use the value of the secret bit from supervisor
