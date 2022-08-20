
.. role:: raw-html-m2r(raw)
   :format: html

.. _abstraction_hdl:


========================
Abstractions / HDL
========================

The NaxRiscv implementation takes advantage of quite a few paradigms available when using the SpinalHDL (a Scala hardware description library).


Framework
===================

The toplevel of NaxRiscv is mostly an empty Component with a framework which can schedule a
list of plugins. The framework itself does not create any hardware.

Here is the NaxRiscv toplevel

.. code:: scala

	class NaxRiscv(xlen : Int,
				   plugins : Seq[Plugin]) extends Component{
	  NaxScope.create(xlen = xlen) //Will come back on that line later
	  val framework = new Framework(plugins)
	}

To give an overview of how much the design is split between plugins,
here is the list of them for one functional CPU :

.. code:: scala

    val plugins = ArrayBuffer[Plugin]()
    plugins += new DocPlugin()
    plugins += new MmuPlugin(
      spec    = MmuSpec.sv32,
      ioRange = ioRange,
      fetchRange = fetchRange
    )

    //FETCH
    plugins += new FetchPlugin()
    plugins += new PcPlugin(resetVector)
    plugins += new FetchCachePlugin(
      cacheSize = 4096*4,
      wayCount = 4,
      injectionAt = 2,
      fetchDataWidth = 64,
      memDataWidth = 64,
      reducedBankWidth = false,
      hitsWithTranslationWays = true,
      translationStorageParameter = MmuStorageParameter(
        levels   = List(
          MmuStorageLevel(
            id    = 0,
            ways  = 4,
            depth = 32
          ),
          MmuStorageLevel(
            id    = 1,
            ways  = 2,
            depth = 32
          )
        ),
        priority = 0
      ),
      translationPortParameter  = MmuPortParameter(
        readAt = 1,
        hitsAt = 1,
        ctrlAt = 1,
        rspAt  = 1
      )
    )
    plugins += new AlignerPlugin(
      decodeCount = 2,
      inputAt = 2
    )

    //FRONTEND
    plugins += new FrontendPlugin()
    plugins += new DecompressorPlugin()
    plugins += new DecoderPlugin()
    plugins += new RfTranslationPlugin()
    plugins += new RfDependencyPlugin()
    plugins += new RfAllocationPlugin(riscv.IntRegFile)
    plugins += new DispatchPlugin(
      slotCount = 32
    )

    //BRANCH PREDICTION
    plugins += new BranchContextPlugin(
      branchCount = 16
    )
    plugins += new HistoryPlugin(
      historyFetchBypass = true
    )
    plugins += new DecoderPredictionPlugin(
      flushOnBranch = false //TODO remove me (DEBUG)
    )
    plugins += new BtbPlugin(
      entries = 512,
      readAt = 0,
      hitAt = 1,
      jumpAt = 1
    )
    plugins += new GSharePlugin(
      memBytes = 4 KiB,
      historyWidth = 24,
      readAt = 0
    )

    //LOAD / STORE
    plugins += new LsuPlugin(
      lqSize = 16,
      sqSize = 16,
      loadToCacheBypass = true,
      lqToCachePipelined = true,
      hitPedictionEntries = 1024,
      translationStorageParameter = MmuStorageParameter(
        levels   = List(
          MmuStorageLevel(
            id    = 0,
            ways  = 4,
            depth = 32
          ),
          MmuStorageLevel(
            id    = 1,
            ways  = 2,
            depth = 32
          )
        ),
        priority = 1
      ),
      loadTranslationParameter  = MmuPortParameter(
        readAt = 0,
        hitsAt = 1,
        ctrlAt = 1,
        rspAt  = 1
      ),
      storeTranslationParameter = MmuPortParameter(
        readAt = 1,
        hitsAt = 1,
        ctrlAt = 1,
        rspAt  = 1
      )
    )
    plugins += new DataCachePlugin(
      memDataWidth = 64,
      cacheSize    = 4096*4,
      wayCount     = 4,
      refillCount = 2,
      writebackCount = 2,
      tagsReadAsync = true,
      reducedBankWidth = false,
      loadRefillCheckEarly = false
    )

    //MISC
    plugins += new RobPlugin(
      robSize = 64,
      completionWithReg = false
    )
    plugins += new CommitPlugin(
      commitCount = 2,
      ptrCommitRetimed = true
    )
    plugins += new RegFilePlugin(
      spec = riscv.IntRegFile,
      physicalDepth = 64,
      bankCount = 1
    )
    plugins += new CommitDebugFilterPlugin(List(4, 8, 12))
    plugins += new CsrRamPlugin()
    plugins += new PrivilegedPlugin(PrivilegedConfig.full.copy(withRdTime = withRdTime))
    plugins += new PerformanceCounterPlugin(
      additionalCounterCount = 4,
      bufferWidth            = 6
    )

    //EXECUTION UNITES
    plugins += new ExecutionUnitBase("EU0")
    plugins += new SrcPlugin("EU0", earlySrc = true)
    plugins += new IntAluPlugin("EU0", aluStage = 0)
    plugins += new ShiftPlugin("EU0" , aluStage = 0)
    plugins += new BranchPlugin("EU0")

    plugins += new ExecutionUnitBase("EU1")
    plugins += new SrcPlugin("EU1")
    plugins += new IntAluPlugin("EU1")
    plugins += new ShiftPlugin("EU1")
    plugins += new BranchPlugin("EU1")

    plugins += new ExecutionUnitBase("EU2", writebackCountMax = 1)
    plugins += new SrcPlugin("EU2", earlySrc = true)
    plugins += new MulPlugin("EU2", writebackAt = 2, staticLatency = false)
    plugins += new DivPlugin("EU2", writebackAt = 2)
    plugins += new LoadPlugin("EU2")
    plugins += new StorePlugin("EU2")
    plugins += new EnvCallPlugin("EU2")(rescheduleAt = 2)
    plugins += new CsrAccessPlugin("EU2")(
      decodeAt = 0,
      readAt = 1,
      writeAt = 2,
      writebackAt = 2,
      staticLatency = false
    )


Each of those plugins may :

-  Implement services used by other plugins (ex : provide jump
   interfaces, provide rescheduling interface, provide a pipeline
   skeleton)
-  Use other plugins functionalities
-  Create hardware
-  Create early tasks (used to setup things between plugins)
-  Create late tasks (used in general to create the required hardware)


Plugin tasks
------------

Here is can instance of dummy plugin creating two tasks (setup / logic):

.. code:: scala

   class DummyPlugin extends Plugin {
     val setup = create early new Area {
       //Here you can setup things with other plugins
       //This code will always run before any late tasks
     }

     val logic = create late new Area {
       //Here you can (for instance) generate hardware
       //This code will always start after any early task
     }
   }

Note that create early and create late will execute their code in a
new threads, which are scheduled by the Framework class.


Service definition
------------------

For instance, the JumpService, providing a hardware jump interface to other plugins. Such a service can be defined as :

.. code:: scala

   //Software interface (elaboration time)
   trait JumpService extends Service{
     def createJumpInterface(priority : Int) : Flow[JumpCmd]
   }

   //Hardware payload of the interface
   case class JumpCmd(pcWidth : Int) extends Bundle{
     val pc = UInt(pcWidth bits)
   }

Service implementation
----------------------

Taking the previously shown JumpService, the PcPlugin could implement it the following way :

.. code:: scala

   case class JumpSpec(interface :  Flow[JumpCmd], priority : Int)
   class PcPlugin() extends Plugin with JumpService{
     val jumpsSpec = ArrayBuffer[JumpSpec]()

     override def createJumpInterface(priority : Int): Flow[JumpCmd] = {
       val spec = JumpSpec(Flow(JumpCmd(32)), priority)
       jumpsSpec += spec
       return spec.interface
     }

     val logic = create late new Area{
       //Here, implement the PC logic and manage the jumpsSpec interfaces
	   val pc = Reg(UInt(32 bits))
	   val sortedJumps = jumpsSpec.sortBy(_.priority) //Lower priority first
	   for(jump <- sortedJumps){
		 when(jump.interface.valid){
		   pc := jump.interface.pc
		 }
	   }
	   ...
     }
   }

Service usage
-------------

Another plugin could then retrieve and use this service by :

.. code:: scala

   class AnotherPlugin() extends Plugin {
     val setup = create early new Area {
       val jump = getService[JumpService].createJumpInterface(42)
     }

     val logic = create late new Area {
       setup.jump.valid := ???
       setup.jump.pc := ???
     }
   }

Service Pipeline definition
---------------------------

Some plugins may even create a pipeline skeleton which can then be
populated by other plugins. For instance :

.. code:: scala

   class FetchPlugin() extends Plugin with LockedImpl {
     val pipeline = create early new Pipeline{
       val stagesCount = 2
       val stages = Array.fill(stagesCount)(newStage())

       import spinal.lib.pipeline.Connection._
       //Connect every stage together
       for((m, s) <- (stages.dropRight(1), stages.tail).zipped){
         connect(m, s)(M2S())
       }
     }

     val logic = create late new Area{
       lock.await() //Allow other plugins to make this blocking until they specified everything they wanted in the pipeline stages.
       pipeline.build()
     }
   }

Service Pipeline usage
----------------------

For instance, the PcPlugin will want to introduce the PC value into the
fetch pipeline :

.. code:: scala

   object PcPlugin extends AreaObject{
     val FETCH_PC = Stageable(UInt(32 bits))  //Define the concept of a FETCH_PC signal being usable through a pipeline
   }

   class PcPlugin() extends Plugin with ...{

     val setup = create early new Area{
       getService[FetchPlugin].retain() //We need to hold the FetchPlugin logic task until we create all the associated accesses
     }

     val logic = create late new Area{
       val fetch = getService[FetchPlugin]
       val firstStage = fetch.pipeline.stages(0)

       firstStage(PcPlugin.FETCH_PC) := ???   //Assign the FETCH_PC value in firstStage of the pipeline. Other plugins may access it down stream.
       fetch.release()
     }
   }

Execution units
---------------------

Implementation of the execution units is another practical use of this concept. You can spawn an execution unit by creating a new ExecutionUnitBase with
a unique execution unit identifier :

.. code:: scala

       plugins += new ExecutionUnitBase("EU0")

Then you can populate that execution unit by adding new
ExecutionUnitElementSimple with the same identifier :

.. code:: scala

       plugins += new SrcPlugin("EU0")
       plugins += new IntAluPlugin("EU0")
       plugins += new ShiftPlugin("EU0")

Here is the example of an execution unit handling :

-  mul/div
-  jump/branches
-  load/store
-  CSR accesses
-  ebreak/ecall/mret/wfi

.. code:: scala

       plugins += new ExecutionUnitBase("EU1", writebackCountMax = 1)
       plugins += new SrcPlugin("EU1")
       plugins += new MulPlugin("EU1", writebackAt = 2, staticLatency = false)
       plugins += new DivPlugin("EU1", writebackAt = 2)
       plugins += new BranchPlugin("EU1", writebackAt = 2, staticLatency = false)
       plugins += new LoadPlugin("EU1")
       plugins += new StorePlugin("EU1")
       plugins += new CsrAccessPlugin("EU1")(
         decodeAt = 0,
         readAt = 1,
         writeAt = 2,
         writebackAt = 2,
         staticLatency = false
       )
       plugins += new EnvCallPlugin("EU1")(rescheduleAt = 2)

ShiftPlugin
-----------

Here is the ShiftPlugin as an example of ExecutionUnitElementSimple
plugin:

.. code:: scala

   object ShiftPlugin extends AreaObject {
     val SIGNED = Stageable(Bool())
     val LEFT = Stageable(Bool())
   }

   class ShiftPlugin(euId : String, staticLatency : Boolean = true, aluStage : Int = 0) extends ExecutionUnitElementSimple(euId, staticLatency) {
     import ShiftPlugin._

     override def euWritebackAt = aluStage

     override val setup = create early new Setup{
       import SrcKeys._

       add(Rvi.SLL , List(SRC1.RF, SRC2.RF), DecodeList(LEFT -> True,  SIGNED -> False))
       add(Rvi.SRL , List(SRC1.RF, SRC2.RF), DecodeList(LEFT -> False, SIGNED -> False))
       add(Rvi.SRA , List(SRC1.RF, SRC2.RF), DecodeList(LEFT -> False, SIGNED -> True))
       add(Rvi.SLLI, List(SRC1.RF, SRC2.I ), DecodeList(LEFT -> True , SIGNED -> False))
       add(Rvi.SRLI, List(SRC1.RF, SRC2.I ), DecodeList(LEFT -> False, SIGNED -> False))
       add(Rvi.SRAI, List(SRC1.RF, SRC2.I ), DecodeList(LEFT -> False, SIGNED -> True))
     }

     override val logic = create late new Logic{
       val process = new ExecuteArea(aluStage) {
         import stage._
         val ss = SrcStageables

         assert(Global.XLEN.get == 32)
         val amplitude  = ss.SRC2(4 downto 0).asUInt
         val reversed   = Mux[SInt](LEFT, ss.SRC1.reversed, ss.SRC1)
         val shifted = (S((SIGNED & ss.SRC1.msb) ## reversed) >> amplitude).resize(Global.XLEN bits)
         val patched = LEFT ? shifted.reversed | shifted

         wb.payload := B(patched)
       }
     }
   }

Pipeline
===================

To allow the definition of extendable/flexible pipelines, the Pipeline abstraction was put into place. This abstraction allows to define stages, arbitrations, connections, and values connected through them.

Here is a simple example :

.. code:: scala

	 new Pipeline{
		val fetch = newStage()
		val decoded = newStage()
		val execute = newStage()
		val memory = newStage()
		val writeback = newStage()

		connect(fetch, decoded)(M2S())
		connect(decoded, execute)(M2S())
		connect(execute, memory)(M2S())
		connect(memory, writeback)(M2S())

		val PC = Stageable(UInt(32 bits)) //This isn't a hardware signal, but it is a "key" used to identify the concept of PC (program counter) in the whole pipeline.
		fetch(PC) := xxx      //Assign xxx to the fetch(PC) value
		yyy := writeback(PC)  //Assign the writeback(PC) value to yyy

		execute.haltWhen(execute(PC) === zzz) //Halt the pipeline at the execute stage when the eecute(PC) match zzz

		build()  //Generate all the required hardware. This will for instance pipeline the PC from the fetch stage to where it is needed (execute/writeback stage)
	  }

Based on that API, multiple plugins in the NaxRiscv CPU can compose / extend existing pipelines.
Also notes that some plugins maybe put into place as a skeleton pipeline for other plugin to work with. This was done for the FetchPlugin, FrontendPlugin and the ExecutionUnitBase.

This API, combined with the concurrent hardware elaboration of the plugins, also allows to adjust the number of stages in the pipeline depending on the plugins' needs, as it is done for the ExecutionUnitBase plugin.

To be more concrete, execution unit plugins using an ExecutionUnitBase as a pipeline skeleton only have to refer to a given stage number for it to be dynamically created (if it wasn't already created before).

Another function of the pipeline API is that it allows identifying pipeline elements with a secondary key. For instance, in the following example some logic which could be used to calculate the next PC of a branch in a pipelined way.

.. code:: scala

	val PC = Stageable(UInt(32 bits))
	val COND = Stageable(UInt(32 bits))
	stageA(COND) := xxx
	stageA(PC, "WITHOUT_BRANCH") := stageA(PC) + 4 //Using the string "WITHOUT_BRANCH" as a secondary key
	stageA(PC, "WITH_BRANCH") := stageA(PC) + offset
	when(stageB(COND)){
	  stageB(PC, "NEXT") := stageB(PC, "WITH_BRANCH") ohterwise stageB(PC, "WITHOUT_BRANCH")
	} otherwise {
	  stageB(PC, "NEXT") := stageB(PC, "WITHOUT_BRANCH")
	}

But more generally, this secondary key is used to access a "dimension". For instance, the pipeline being used to decode two instruction at the time :

.. code:: scala

	val OPCODE = Stageable(Bits(32 bits))
	for(decodeIndex <- 0 until decodeCount){
	   decodeStage(OPCODE, decodeIndex) := xxx   //Using the index of the decoding unit we are working on as a secondary key
	}

State machine API
===================

Not something ground breaking, but in a few places, the SpinalHDL state machine API is used. Here is a short example of the API :

.. code:: scala

 val fsm = new StateMachine{
    val stateA = new State with EntryPoint
    val stateB = new State
    val stateC = new State
    val counter = Reg(UInt(8 bits)) init (0)

    stateA.whenIsActive (goto(stateB))

    stateB.onEntry(counter := 0)
    stateB.whenIsActive {
	counter := counter + 1
	  when(counter === 4){
	    goto(stateC)
	  }
    }
    stateB.onExit(io.result := True)

    stateC.whenIsActive (goto(stateA))
  }

Automated multiport memory transformation
=========================================================

In quite a few places in the design, there are memories which need multiple write ports. Such memories are not directly inferable by the FPGA synthesis tools most of the time.

Still the NaxRiscv Scala code defines them all over the place, and to fill this gap, a custom SpinalHDL transformation phase was added to simplify them into groups of simple dual port ram with xor based glue.

Here is how this transformation phase is added into the flow :

.. code:: scala

    val spinalConfig = SpinalConfig()
    spinalConfig.addTransformationPhase(new MultiPortWritesSymplifier)
    spinalConfig.generateVerilog(new NaxRiscv(xlen = 32, plugins))


And here is how such a transformation phase is defined :

.. code:: scala

	class MultiPortWritesSymplifier extends PhaseMemBlackboxing{
	  override def doBlackboxing(pc: PhaseContext, typo: MemTopology) = {
		if(typo.writes.size > 1 && typo.readsSync.size == 0){
		  typo.writes.foreach(w => assert(w.mask == null))
		  typo.writes.foreach(w => assert(w.clockDomain == typo.writes.head.clockDomain))
		  val cd = typo.writes.head.clockDomain

		  import typo._

		  val ctx = List(mem.parentScope.push(), cd.push())

		  // RamAsyncMwXor implement multiple write ports using simple dual port rams and xors
		  val c = RamAsyncMwXor(
			payloadType = Bits(mem.width bits),
			depth       = mem.wordCount,
			writePorts  = writes.size,
			readPorts   = readsAsync.size
		  ).setCompositeName(mem)

		  //Connect the write ports to the RamAsyncMwXor
		  for((dst, src) <- (c.io.writes, writes).zipped){
			dst.valid.assignFrom(src.writeEnable)
			dst.address.assignFrom(src.address)
			dst.data.assignFrom(src.data)
		  }

		  //Connect the read ports to the RamAsyncMwXor
		  for((reworked, old) <- (c.io.read, readsAsync).zipped){
			reworked.cmd.payload.assignFrom(old.address)
			wrapConsumers(typo, old, reworked.rsp)
		  }

          //Cleanup the old memory from the netlist
		  mem.removeStatement()
		  mem.foreachStatements(s => s.removeStatement())

		  ctx.foreach(_.restore())
		}
	  }
	}
