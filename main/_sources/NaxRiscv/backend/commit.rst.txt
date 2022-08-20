.. role:: raw-html-m2r(raw)
   :format: html

Commit
============================

The commit logic is mostly composed :

- **Reschedule status** Which specifies the older ROB ID on which a jump / trap is pending.
- **Commit Logic** Which waits for the older instruction completion before committing it, and eventually applying some reschedule.


.. image:: /asset/image/commit.png
