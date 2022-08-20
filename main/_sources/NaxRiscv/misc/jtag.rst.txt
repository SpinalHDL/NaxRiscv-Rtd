.. role:: raw-html-m2r(raw)
   :format: html

Jtag / OpenOCD / GDB
========================

NaxRiscv implements the RISCV External Debug Support v. 0.13.2 specification via JTAG. This enables upstream openocd support,
which in itself allows to use GDB to debug software running on a target.

The JTAG layer supports 2 modes:

- Native : Where a full JTAG TAP is implemented
- Tunneled : Where an external JTAG TAP should provide a single JTAG instruction port to take control (ex BSCANE2 from xilinx)

The implementation of the debug spec is done as follows :

- Abstract access register to x0-x31 (implemented by running a hidden abstract program buffer)
- Abstract program buffer to do everything else (memory/CSR access)
- The HART can read abstract command arguments via a CSR read, and provide a return value via a CSR write
- Triggers can only be used as hardware breakpoints
- PrivilegedPlugin implement things related to the HART, while the EmbeddedJtagPlugin integrate the different JTAG alternatives and DM in the core.
- The tunneled jtag mode use the BSCAN_TUNNEL_DATA_REGISTER mode, not the default one.

The reason why BSCAN_TUNNEL_DATA_REGISTER was picked instead of the BSCAN_TUNNEL_NESTED_TAP is to properly support JTAG chains with multiple taps.
During a JTAG scan with openocd, the only reference you can have is relative to the end of a DR-SHIFT, you can't use the beginning of the DR-SHIFT as a reference,
because openocd may add a few cycles there to propagate the data through the chain to the selected tap, which make BSCAN_TUNNEL_NESTED_TAP unpractical as the bits which identify the nature of its DR-SHIFT are sent first.
