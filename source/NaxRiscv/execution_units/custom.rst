

Custom instruction
==============================

There is multiple ways you can add custom instruction into NaxRiscv, the following chapter will provide some demo.

SIMD add
^^^^^^^^^^

Let's define a plugin which will implement a SIMD add (4x8bits adder), working on the integer register file.

The plugin will be based on the ExecutionUnitElementSimple which make implementing ALU plugins simpler. Such plugin can then be used to compose a given execution unit (hosted by a ExecutionUnitBase).

For instance the Plugin configuration could be : 

.. code:: scala

    plugins += new ExecutionUnitBase("ALU0")
    plugins += new IntFormatPlugin("ALU0")
    plugins += new SrcPlugin("ALU0", earlySrc = true)
    plugins += new IntAluPlugin("ALU0", aluStage = 0)
    plugins += new ShiftPlugin("ALU0" , aluStage = 0)
    plugins += new ShiftPlugin("ALU0" , aluStage = 0)
    plugins += new SimdAddPlugin("ALU0") // <- We will implement this plugin

Here is a example how this plugin could be implemented (https://github.com/SpinalHDL/NaxRiscv/blob/d44ac3a3a3a4328cf2c654f9a46171511a798fae/src/main/scala/naxriscv/execute/SimdAddPlugin.scala#L36): 

.. code:: scala

    package naxriscv.execute

    import spinal.core._
    import spinal.lib._
    import naxriscv._
    import naxriscv.riscv._
    import naxriscv.riscv.IntRegFile
    import naxriscv.interfaces.{RS1, RS2}
    import naxriscv.utilities.Plugin

    //This plugin example will add a new instruction named SIMD_ADD which do the following :
    //
    //RD : Regfile Destination, RS : Regfile Source
    //RD( 7 downto  0) = RS1( 7 downto  0) + RS2( 7 downto  0)
    //RD(16 downto  8) = RS1(16 downto  8) + RS2(16 downto  8)
    //RD(23 downto 16) = RS1(23 downto 16) + RS2(23 downto 16)
    //RD(31 downto 24) = RS1(31 downto 24) + RS2(31 downto 24)
    //
    //Instruction encoding :
    //0000000----------000-----0001011   <- Custom0 func3=0 func7=0
    //       |RS2||RS1|   |RD |
    //
    //Note :  RS1, RS2, RD positions follow the RISC-V spec and are common for all instruction of the ISA


    object SimdAddPlugin{
      //Define the instruction type and encoding that we wll use
      val ADD4 = IntRegFile.TypeR(M"0000000----------000-----0001011")
    }

    //ExecutionUnitElementSimple Is a base class which will be coupled to the pipeline provided by a ExecutionUnitBase with
    //the same euId. It provide quite a few utilities to ease the implementation of custom instruction.
    //Here we will implement a plugin which provide SIMD add on the register file.
    //staticLatency=true specify that our plugin will never halt the pipeling, allowing the issue queue to statically
    //wake up instruction which depend on its result.
    class SimdAddPlugin(val euId : String) extends ExecutionUnitElementSimple(euId, staticLatency = true) {
      //We will assume our plugin is fully combinatorial
      override def euWritebackAt = 0

      //The setup code is by plugins to specify things to each others before it is too late
      //create early blockOfCode will
      override val setup = create early new Setup{
        //Let's assume we only support RV32 for now
        assert(Global.XLEN.get == 32)

        //Specify to the ExecutionUnitBase that the current plugin will implement the ADD4 instruction
        add(SimdAddPlugin.ADD4)
      }

      override val logic = create late new Logic{
        val process = new ExecuteArea(stageId = 0) {
          //Get the RISC-V RS1/RS2 values from the register file
          val rs1 = stage(eu(IntRegFile, RS1)).asUInt
          val rs2 = stage(eu(IntRegFile, RS2)).asUInt

          //Do some computation
          val rd = UInt(32 bits)
          rd( 7 downto  0) := rs1( 7 downto  0) + rs2( 7 downto  0)
          rd(16 downto  8) := rs1(16 downto  8) + rs2(16 downto  8)
          rd(23 downto 16) := rs1(23 downto 16) + rs2(23 downto 16)
          rd(31 downto 24) := rs1(31 downto 24) + rs2(31 downto 24)

          //Provide the computation value for the writeback
          wb.payload := rd.asBits
        }
      }
    }
    
    
Then, to generate a NaxRiscv with this new plugin, we could run the following App (https://github.com/SpinalHDL/NaxRiscv/blob/d44ac3a3a3a4328cf2c654f9a46171511a798fae/src/main/scala/naxriscv/execute/SimdAddPlugin.scala#L71): 

.. code:: scala

    object SimdAddNaxGen extends App{
      import naxriscv.compatibility._
      import naxriscv.utilities._

      def plugins = {
        //Get a default list of plugins
        val l = Config.plugins(
          withRdTime = false,
          aluCount    = 2,
          decodeCount = 2
        )
        //Add our plugin to the two ALUs
        l += new SimdAddPlugin("ALU0")
        l += new SimdAddPlugin("ALU1")
        l
      }

      //Create a SpinalHDL configuration that will be used to generate the hardware
      val spinalConfig = SpinalConfig(inlineRom = true)
      spinalConfig.addTransformationPhase(new MemReadDuringWriteHazardPhase)
      spinalConfig.addTransformationPhase(new MultiPortWritesSymplifier)

      //Generate the NaxRiscv verilog file
      val report = spinalConfig.generateVerilog(new NaxRiscv(xlen = 32, plugins))
      
      //Generate some C header files used by the verilator testbench to connect to the DUT
      report.toplevel.framework.getService[DocPlugin].genC()
    }    


To run this App, you can go to the NaxRiscv directory and run : 

.. code:: shell

    sbt "runMain naxriscv.execute.SimdAddNaxGen"
    
    
Then let's write some assembly test code (https://github.com/SpinalHDL/NaxSoftware/tree/849679c70b238ceee021bdfd18eb2e9809e7bdd0/baremetal/simdAdd): 

.. code:: shell

    .globl _start
    _start:

    #include "../../driver/riscv_asm.h"
    #include "../../driver/sim_asm.h"
    #include "../../driver/custom_asm.h"

        //Test 1
        li x1, 0x01234567
        li x2, 0x01FF01FF
        opcode_R(CUSTOM0, 0x0, 0x00, x3, x1, x2) //x3 = ADD4(x1, x2)

        //Print result value
        li x4, PUT_HEX
        sw x3, 0(x4)

        //Check result
        li x5, 0x02224666
        bne x3, x5, fail

        j pass

    pass:
        j pass
    fail:
        j fail

Compile it with 

.. code:: shell

    make clean rv32im
    
And the run a simulation in src/test/cpp/naxriscv (You will have to setup things as described in its readme first)

.. code:: shell

    make clean compile
    ./obj_dir/VNaxRiscv --load-elf ../../../../ext/NaxSoftware/baremetal/simdAdd/build/rv32im/simdAdd.elf --spike-disable --pass-symbol pass --fail-symbol fail --trace
    
Which will output us the value 2224666 in the shell :D

So overall this example didn't introduced how to specify some additional decoding, nor how to define multi-cycle ALU. (TODO). 
But you can take a look in the IntAluPlugin, ShiftPlugin, DivPlugin, MulPlugin and BranchPlugin which are doing those things using the same ExecutionUnitElementSimple base class.

Also, you don't have to use the ExecutionUnitElementSimple base class, you can have more fondamental accesses, as the LoadPlugin, StorePlugin, EnvCallPlugin.
    
    

