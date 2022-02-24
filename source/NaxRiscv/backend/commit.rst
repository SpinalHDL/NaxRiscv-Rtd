.. role:: raw-html-m2r(raw)
   :format: html

Commit
============================

The commit logic is mostly composed : 

- **Reschedule status** Which specify the older ROB ID on which a jump / trap is pending.
- **Commit Logic** Which wait for the older instruction completion before commiting it, and eventualy applying some reschedule.


.. image:: /asset/image/commit.png
