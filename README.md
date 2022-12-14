# PspCIDTable-pool
Handle Table NT-Kernel
# PspCidTable Analysis

## PspCidTable shares the same data structure with handle table

let's see it:

            kd> dt _HANDLE_TABLE poi(nt!PspCidTable)
            ntdll!_HANDLE_TABLE
               +0x000 TableCode        : 0xfffff8a0`018f6001
               +0x008 QuotaProcess     : (null) 
               +0x010 UniqueProcessId  : (null) 
               +0x018 HandleLock       : _EX_PUSH_LOCK
               +0x020 HandleTableList  : _LIST_ENTRY [ 0xfffff8a0`000048a0 - 0xfffff8a0`000048a0 ]
               +0x030 HandleContentionEvent : _EX_PUSH_LOCK
               +0x038 DebugInfo        : (null) 
               +0x040 ExtraInfoPages   : 0n0
               +0x044 Flags            : 1
               +0x044 StrictFIFO       : 0y1
               +0x048 FirstFreeHandle  : 0x314
               +0x050 LastFreeHandleEntry : 0xfffff8a0`018f2a80 _HANDLE_TABLE_ENTRY
               +0x058 HandleCount      : 0x27d
               +0x05c NextHandleNeedingPool : 0xc00
               +0x060 HandleCountHighWatermark : 0x29f

Note nt!PspCidTable are pointing to an address of the PspCidTable, but not holding this structrue.

the lowest byte of TableCode indicates this handle table is 2 level table:

            kd> dq (0xfffff8a0`018f6001 & 0xFFFFFFFFFFFFFF00)
            fffff8a0`018f6000  fffff8a0`00005000 fffff8a0`018f2000
            fffff8a0`018f6010  fffff8a0`01f83000 00000000`00000000
            fffff8a0`018f6020  00000000`00000000 00000000`00000000

so we got 3 sub tables, each occupy 1 page.

the first entry in the first sub table is :

            kd> dt _HANDLE_TABLE_ENTRY fffff8a0`00005000
            ntdll!_HANDLE_TABLE_ENTRY
               +0x000 Object           : (null) 
               +0x000 ObAttributes     : 0
               +0x000 InfoTable        : (null) 
               +0x000 Value            : 0
               +0x008 GrantedAccess    : 0xfffffffe
               +0x008 GrantedAccessIndex : 0xfffe
               +0x00a CreatorBackTraceIndex : 0xffff
               +0x008 NextFreeTableEntry : 0xfffffffe

a dummy one, let's see the second entry:

            kd> dt _HANDLE_TABLE_ENTRY fffff8a0`00005000+0x10
            ntdll!_HANDLE_TABLE_ENTRY
               +0x000 Object           : 0xfffffa80`18dde741 Void
               +0x000 ObAttributes     : 0x18dde741
               +0x000 InfoTable        : 0xfffffa80`18dde741 _HANDLE_TABLE_ENTRY_INFO
               +0x000 Value            : 0xfffffa80`18dde741
               +0x008 GrantedAccess    : 0
               +0x008 GrantedAccessIndex : 0
               +0x00a CreatorBackTraceIndex : 0
               +0x008 NextFreeTableEntry : 0

PspCidTable's entries are direct \_KPROCESS or \_KTHREAD object, rather than normal \_OBJECT\_HEADER data struct.
But how to determine if it is a \_KPROCESS or a \_KTHREAD?

If we assume it is \_KPROCESS: 

            kd> dt _KPROCESS 0xfffffa80`18dde741
            ntdll!_KPROCESS
               +0x000 Header           : _DISPATCHER_HEADER
               +0x018 ProfileListHead  : _LIST_ENTRY [ 0x58fffffa`8018dde7 - 0x00fffffa`8018dde7 ]
               +0x028 DirectoryTableBase : 0x58000000`00001870
               +0x030 ThreadListHead   : _LIST_ENTRY [ 0x58fffffa`8018dde5 - 0x00fffffa`801a8afe ]
               +0x040 ProcessLock      : 0x01000000`00000000
               +0x048 Affinity         : _KAFFINITY_EX
               +0x070 ReadyListHead    : _LIST_ENTRY [ 0xb0fffffa`8018dde7 - 0x00fffffa`8018dde7 ]
               +0x080 SwapListEntry    : _SINGLE_LIST_ENTRY
               +0x088 ActiveProcessors : _KAFFINITY_EX
               +0x0b0 AutoAlignment    : 0y0
               +0x0b0 DisableBoost     : 0y0
               +0x0b0 DisableQuantum   : 0y0
               +0x0b0 ActiveGroupsMask : 0y0000
               +0x0b0 ReservedFlags    : 0y0000100000000000000000000 (0x100000)
               +0x0b0 ProcessFlags     : 0n134217728
               +0x0b4 BasePriority     : 6 ''
               +0x0b5 QuantumReset     : 0 ''
               +0x0b6 Visited          : 0 ''
               +0x0b7 Unused3          : 0 ''
               +0x0b8 ThreadSeed       : [4] 0
               +0x0c8 IdealNode        : [4] 0
               +0x0d0 IdealGlobalNode  : 0
               +0x0d2 Flags            : _KEXECUTE_OPTIONS
               +0x0d3 Unused1          : 0 ''
               +0x0d4 Unused2          : 0
               +0x0d8 Unused4          : 0xf8000000
               +0x0dc StackCount       : _KSTACK_COUNT
               +0x0e0 ProcessListEntry : _LIST_ENTRY [ 0xf0fffffa`8019323c - 0x3afffff8`0002c79a ]
               +0x0f0 CycleTime        : 0x40000000`00af2f1d
               +0x0f8 KernelTime       : 0
               +0x0fc UserTime         : 0
               +0x100 InstrumentationCallback : (null) 
               +0x108 LdtSystemDescriptor : _KGDTENTRY64
               +0x118 LdtBaseAddress   : 0x01000000`00000000 Void
               +0x120 LdtProcessLock   : _KGUARDED_MUTEX
               +0x158 LdtFreeSelectorHint : 0
               +0x15a LdtTableLength   : 0

And what about \_KTHREAD:

            kd> dt _KTHREAD 0xfffffa80`18dde741
            ntdll!_KTHREAD
               +0x000 Header           : _DISPATCHER_HEADER
               +0x018 CycleTime        : 0x58fffffa`8018dde7
               +0x020 QuantumTarget    : 0x00fffffa`8018dde7
               +0x028 InitialStack     : 0x58000000`00001870 Void
               +0x030 StackLimit       : 0x58fffffa`8018dde5 Void
               +0x038 KernelStack      : 0x00fffffa`801a8afe Void
               +0x040 ThreadLock       : 0x01000000`00000000
               +0x048 WaitRegister     : _KWAIT_STATUS_REGISTER
               +0x049 Running          : 0x4 ''
               +0x04a Alerted          : [2]  ""
               +0x04c KernelStackResident : 0y0
               +0x04c ReadyTransition  : 0y0
               +0x04c ProcessReadyQueue : 0y0
               +0x04c WaitNext         : 0y0
               +0x04c SystemAffinityActive : 0y0
               +0x04c Alertable        : 0y0
               +0x04c GdiFlushActive   : 0y0
               +0x04c UserStackWalkActive : 0y0
               +0x04c ApcInterruptRequest : 0y0
               +0x04c ForceDeferSchedule : 0y0
               +0x04c QuantumEndMigrate : 0y0
               +0x04c UmsDirectedSwitchEnable : 0y0
               +0x04c TimerActive      : 0y0
               +0x04c SystemThread     : 0y0
               +0x04c Reserved         : 0y000000010000000000 (0x400)
               +0x04c MiscFlags        : 0n16777216
               +0x050 ApcState         : _KAPC_STATE
               +0x050 ApcStateFill     : [43]  ""
               +0x07b Priority         : -128 ''
               +0x07c NextProcessor    : 0xfffffa
               +0x080 DeferredProcessor : 0
               +0x088 ApcQueueLock     : 0x400
               +0x090 WaitStatus       : 0n0
               +0x098 WaitBlockList    : (null) 
               +0x0a0 WaitListEntry    : _LIST_ENTRY [ 0x00000000`00000000 - 0x09000000`00000000 ]
               +0x0a0 SwapListEntry    : _SINGLE_LIST_ENTRY
               +0x0b0 Queue            : 0x00000006`08000000 _KQUEUE
               +0x0b8 Teb              : (null) 
               +0x0c0 Timer            : _KTIMER
               +0x100 AutoAlignment    : 0y0
               +0x100 DisableBoost     : 0y0
               +0x100 EtwStackTraceApc1Inserted : 0y0
               +0x100 EtwStackTraceApc2Inserted : 0y0
               +0x100 CalloutActive    : 0y0
               +0x100 ApcQueueable     : 0y0
               +0x100 EnableStackSwap  : 0y0
               +0x100 GuiThread        : 0y0
               +0x100 UmsPerformingSyscall : 0y0
               +0x100 VdmSafe          : 0y0
               +0x100 UmsDispatched    : 0y0
               +0x100 ReservedFlags    : 0y000000000000000000000 (0)
               +0x100 ThreadFlags      : 0n0
               +0x104 Spare0           : 0
               +0x108 WaitBlock        : [4] _KWAIT_BLOCK
               +0x108 WaitBlockFill4   : [44]  ""
               +0x134 ContextSwitches  : 0x7000000
               +0x108 WaitBlockFill5   : [92]  ""
               +0x164 State            : 0 ''
               +0x165 NpxState         : 0 ''
               +0x166 WaitIrql         : 0 ''
               +0x167 WaitMode         : -37 ''
               +0x108 WaitBlockFill6   : [140]  ""
               +0x194 WaitTime         : 0xfffff8
               +0x108 WaitBlockFill7   : [168]  ""
               +0x1b0 TebMappedLowVa   : 0x21000000`00000000 Void
               +0x1b8 Ucb              : (null) 
               +0x108 WaitBlockFill8   : [188]  ""
               +0x1c4 KernelApcDisable : 0n-8
               +0x1c6 SpecialApcDisable : 0n255
               +0x1c4 CombinedApcDisable : 0xfffff8
               +0x1c8 QueueListEntry   : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`000097b0 ]
               +0x1d8 TrapFrame        : 0x00000000`000041d0 _KTRAP_FRAME
               +0x1e0 FirstArgument    : (null) 
               +0x1e8 CallbackStack    : (null) 
               +0x1e8 CallbackDepth    : 0
               +0x1f0 ApcStateIndex    : 0 ''
               +0x1f1 BasePriority     : 0 ''
               +0x1f2 PriorityDecrement : 0 ''
               +0x1f2 ForegroundBoost  : 0y0000
               +0x1f2 UnusualBoost     : 0y0000
               +0x1f3 Preempted        : 0 ''
               +0x1f4 AdjustReason     : 0 ''
               +0x1f5 AdjustIncrement  : 0 ''
               +0x1f6 PreviousMode     : 0 ''
               +0x1f7 Saturation       : 0 ''
               +0x1f8 SystemCallNumber : 0
               +0x1fc FreezeCount      : 0x80000000
               +0x200 UserAffinity     : _GROUP_AFFINITY
               +0x210 Process          : (null) 
               +0x218 Affinity         : _GROUP_AFFINITY
               +0x228 IdealProcessor   : 0
               +0x22c UserIdealProcessor : 0
               +0x230 ApcStatePointer  : [2] 0xd0000000`00000000 _KAPC_STATE
               +0x240 SavedApcState    : _KAPC_STATE
               +0x240 SavedApcStateFill : [43]  ""
               +0x26b WaitReason       : 0 ''
               +0x26c SuspendCount     : 0 ''
               +0x26d Spare1           : 0 ''
               +0x26e CodePatchInProgress : 0 ''
               +0x270 Win32Thread      : (null) 
               +0x278 StackBase        : (null) 
               +0x280 SuspendApc       : _KAPC
               +0x280 SuspendApcFill0  : [1]  ""
               +0x281 ResourceIndex    : 0 ''
               +0x280 SuspendApcFill1  : [3]  ""
               +0x283 QuantumReset     : 0 ''
               +0x280 SuspendApcFill2  : [4]  ""
               +0x284 KernelTime       : 0
               +0x280 SuspendApcFill3  : [64]  ""
               +0x2c0 WaitPrcb         : 0x00000007`fffffe00 _KPRCB
               +0x280 SuspendApcFill4  : [72]  ""
               +0x2c8 LegoData         : 0x00000000`0077cd90 Void
               +0x280 SuspendApcFill5  : [83]  ""
               +0x2d3 LargeStack       : 0 ''
               +0x2d4 UserTime         : 0
               +0x2d8 SuspendSemaphore : _KSEMAPHORE
               +0x2d8 SuspendSemaphorefill : [28]  ""
               +0x2f4 SListFaultCount  : 0
               +0x2f8 ThreadListEntry  : _LIST_ENTRY [ 0x00000000`00000000 - 0x80000000`00000000 ]
               +0x308 MutantListHead   : _LIST_ENTRY [ 0x80fffffa`8018dde6 - 0x00fffffa`801a8aff ]
               +0x318 SListFaultAddress : (null) 
               +0x320 ReadOperationCount : 0n7493989779944505344
               +0x328 WriteOperationCount : 0n360287970189639680
               +0x330 OtherOperationCount : 0n0
               +0x338 ReadTransferCount : 0n0
               +0x340 WriteTransferCount : 0n1080863910568919040
               +0x348 OtherTransferCount : 0n3314649325744685056
               +0x350 ThreadCounters   : 0x02000000`00000000 _KTHREAD_COUNTERS
               +0x358 XStateSave       : 0x80000000`00000002 _XSTATE_SAVE

According to 0x28 offset, 0x5800000000001870 is reasonable as DirectoryTableBase than InitialStack,
because page directory is a physical address, while InitialStack is a virtual address.

But don't you feel a little awkward with the alignment, let just ignore the least significant 4 bits:

            kd> dt _KPROCESS 0xfffffa80`18dde740
            ntdll!_KPROCESS
               +0x000 Header           : _DISPATCHER_HEADER
               +0x018 ProfileListHead  : _LIST_ENTRY [ 0xfffffa80`18dde758 - 0xfffffa80`18dde758 ]
               +0x028 DirectoryTableBase : 0x187000
               +0x030 ThreadListHead   : _LIST_ENTRY [ 0xfffffa80`18dde558 - 0xfffffa80`1a8afe58 ]
               +0x040 ProcessLock      : 0
               +0x048 Affinity         : _KAFFINITY_EX
               +0x070 ReadyListHead    : _LIST_ENTRY [ 0xfffffa80`18dde7b0 - 0xfffffa80`18dde7b0 ]
               +0x080 SwapListEntry    : _SINGLE_LIST_ENTRY
               +0x088 ActiveProcessors : _KAFFINITY_EX
               +0x0b0 AutoAlignment    : 0y1
               +0x0b0 DisableBoost     : 0y0
               +0x0b0 DisableQuantum   : 0y0
               +0x0b0 ActiveGroupsMask : 0y0001
               +0x0b0 ReservedFlags    : 0y0000000000000000000000000 (0)
               +0x0b0 ProcessFlags     : 0n9
               +0x0b4 BasePriority     : 8 ''
               +0x0b5 QuantumReset     : 6 ''
               +0x0b6 Visited          : 0 ''
               +0x0b7 Unused3          : 0 ''
               +0x0b8 ThreadSeed       : [4] 0
               +0x0c8 IdealNode        : [4] 0
               +0x0d0 IdealGlobalNode  : 0
               +0x0d2 Flags            : _KEXECUTE_OPTIONS
               +0x0d3 Unused1          : 0 ''
               +0x0d4 Unused2          : 0
               +0x0d8 Unused4          : 0
               +0x0dc StackCount       : _KSTACK_COUNT
               +0x0e0 ProcessListEntry : _LIST_ENTRY [ 0xfffffa80`19323c10 - 0xfffff800`02c79af0 ]
               +0x0f0 CycleTime        : 0xaf2f1d3a
               +0x0f8 KernelTime       : 0x40
               +0x0fc UserTime         : 0
               +0x100 InstrumentationCallback : (null) 
               +0x108 LdtSystemDescriptor : _KGDTENTRY64
               +0x118 LdtBaseAddress   : (null) 
               +0x120 LdtProcessLock   : _KGUARDED_MUTEX
               +0x158 LdtFreeSelectorHint : 0
               +0x15a LdtTableLength   : 0

now it seems more reasonable.

And the \_KPROCESS is also an object, it should have an object header, so step back to its header:

            kd> dt _OBJECT_HEADER
            nt!_OBJECT_HEADER
               +0x000 PointerCount     : Int8B
               +0x008 HandleCount      : Int8B
               +0x008 NextToFree       : Ptr64 Void
               +0x010 Lock             : _EX_PUSH_LOCK
               +0x018 TypeIndex        : UChar
               +0x019 TraceFlags       : UChar
               +0x01a InfoMask         : UChar
               +0x01b Flags            : UChar
               +0x020 ObjectCreateInfo : Ptr64 _OBJECT_CREATE_INFORMATION
               +0x020 QuotaBlockCharged : Ptr64 Void
               +0x028 SecurityDescriptor : Ptr64 Void
               +0x030 Body             : _QUAD

we see that object header is located before 0x30 bytes:

            kd> dt _OBJECT_HEADER 0xfffffa80`18dde740-0x30
            nt!_OBJECT_HEADER
               +0x000 PointerCount     : 0n146
               +0x008 HandleCount      : 0n3
               +0x008 NextToFree       : 0x00000000`00000003 Void
               +0x010 Lock             : _EX_PUSH_LOCK
               +0x018 TypeIndex        : 0x7 ''
               +0x019 TraceFlags       : 0 ''
               +0x01a InfoMask         : 0 ''
               +0x01b Flags            : 0x2 ''
               +0x020 ObjectCreateInfo : 0xfffff800`02c06c00 _OBJECT_CREATE_INFORMATION
               +0x020 QuotaBlockCharged : 0xfffff800`02c06c00 Void
               +0x028 SecurityDescriptor : 0xfffff8a0`000045a5 Void
               +0x030 Body             : _QUAD

what does TypeIndex means, the following enum definitions are digested from volatility:

For Windows 7:

            type_map = { 2: 'Type',
                        3: 'Directory',
                        4: 'SymbolicLink',
                        5: 'Token',
                        6: 'Job',
                        7: 'Process',
                        8: 'Thread',
                        9: 'UserApcReserve',
                        10: 'IoCompletionReserve',
                        11: 'DebugObject',
                        12: 'Event',
                        13: 'EventPair',
                        14: 'Mutant',
                        15: 'Callback',
                        16: 'Semaphore',
                        17: 'Timer',
                        18: 'Profile',
                        19: 'KeyedEvent',
                        20: 'WindowStation',
                        21: 'Desktop',
                        22: 'TpWorkerFactory',
                        23: 'Adapter',
                        24: 'Controller',
                        25: 'Device',
                        26: 'Driver',
                        27: 'IoCompletion',
                        28: 'File',
                        29: 'TmTm',
                        30: 'TmTx',
                        31: 'TmRm',
                        32: 'TmEn',
                        33: 'Section',
                        34: 'Session',
                        35: 'Key',
                        36: 'ALPC Port',
                        37: 'PowerRequest',
                        38: 'WmiGuid',
                        39: 'EtwRegistration',
                        40: 'EtwConsumer',
                        41: 'FilterConnectionPort',
                        42: 'FilterCommunicationPort',
                        43: 'PcwObject',
                    }

For Windows 8:

                type_map = { 2: 'Type',
                            3: 'Directory',
                            4: 'SymbolicLink',
                            5: 'Token',
                            6: 'Job',
                            7: 'Process',
                            8: 'Thread',
                            9: 'UserApcReserve',
                            10: 'IoCompletionReserve',
                            11: 'DebugObject',
                            12: 'Event',
                            13: 'EventPair',
                            14: 'Mutant',
                            15: 'Callback',
                            16: 'Semaphore',
                            17: 'Timer',
                            18: 'Profile',
                            19: 'KeyedEvent',
                            20: 'WindowStation',
                            21: 'Desktop',
                            22: 'TpWorkerFactory',
                            23: 'Adapter',
                            24: 'Controller',
                            25: 'Device',
                            26: 'Driver',
                            27: 'IoCompletion',
                            28: 'File',
                            29: 'TmTm',
                            30: 'TmTx',
                            31: 'TmRm',
                            32: 'TmEn',
                            33: 'Section',
                            34: 'Session',
                            35: 'Key',
                            36: 'ALPC Port',
                            37: 'PowerRequest',
                            38: 'WmiGuid',
                            39: 'EtwRegistration',
                            40: 'EtwConsumer',
                            41: 'FilterConnectionPort',
                            42: 'FilterCommunicationPort',
                            43: 'PcwObject',
                        }

So now we are sure that the second entry in PspCidTable is a process object, its Cid is 0x10 / 4 = 0x04:

                kd> dt _EPROCESS fffffa80`18dde740
                ntdll!_EPROCESS
                   +0x000 Pcb              : _KPROCESS
                   +0x160 ProcessLock      : _EX_PUSH_LOCK
                   +0x168 CreateTime       : _LARGE_INTEGER 0x01d0c825`4a7fe0db
                   +0x170 ExitTime         : _LARGE_INTEGER 0x0
                   +0x178 RundownProtect   : _EX_RUNDOWN_REF
                   +0x180 UniqueProcessId  : 0x00000000`00000004 Void
                   +0x188 ActiveProcessLinks : _LIST_ENTRY [ 0xfffffa80`19323cb8 - 0xfffff800`02c28b90 ]
                   +0x198 ProcessQuotaUsage : [2] 0
                   +0x1a8 ProcessQuotaPeak : [2] 0
                   +0x1b8 CommitCharge     : 0x21
                   +0x1c0 QuotaBlock       : 0xfffff800`02c06c00 _EPROCESS_QUOTA_BLOCK
                   +0x1c8 CpuQuotaBlock    : (null) 
                   +0x1d0 PeakVirtualSize  : 0x97b000
                   +0x1d8 VirtualSize      : 0x41d000
                   +0x1e0 SessionProcessLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
                   +0x1f0 DebugPort        : (null) 
                   +0x1f8 ExceptionPortData : (null) 
                   +0x1f8 ExceptionPortValue : 0
                   +0x1f8 ExceptionPortState : 0y000
                   +0x200 ObjectTable      : 0xfffff8a0`00001780 _HANDLE_TABLE
                   +0x208 Token            : _EX_FAST_REF
                   +0x210 WorkingSetPage   : 0
                   +0x218 AddressCreationLock : _EX_PUSH_LOCK
                   +0x220 RotateInProgress : (null) 
                   +0x228 ForkInProgress   : (null) 
                   +0x230 HardwareTrigger  : 0
                   +0x238 PhysicalVadRoot  : 0xfffffa80`18e3f7d0 _MM_AVL_TABLE
                   +0x240 CloneRoot        : (null) 
                   +0x248 NumberOfPrivatePages : 0xc
                   +0x250 NumberOfLockedPages : 0x40
                   +0x258 Win32Process     : (null) 
                   +0x260 Job              : (null) 
                   +0x268 SectionObject    : (null) 
                   +0x270 SectionBaseAddress : (null) 
                   +0x278 Cookie           : 0
                   +0x27c Spare8           : 0
                   +0x280 WorkingSetWatch  : (null) 
                   +0x288 Win32WindowStation : (null) 
                   +0x290 InheritedFromUniqueProcessId : (null) 
                   +0x298 LdtInformation   : (null) 
                   +0x2a0 Spare            : (null) 
                   +0x2a8 ConsoleHostProcess : 0
                   +0x2b0 DeviceMap        : 0xfffff8a0`00008bc0 Void
                   +0x2b8 EtwDataSource    : (null) 
                   +0x2c0 FreeTebHint      : 0x000007ff`fffe0000 Void
                   +0x2c8 PageDirectoryPte : _HARDWARE_PTE
                   +0x2c8 Filler           : 0x77cd9000
                   +0x2d0 Session          : (null) 
                   +0x2d8 ImageFileName    : [15]  ""
                   +0x2e7 PriorityClass    : 0 ''
                   +0x2e8 JobLinks         : _LIST_ENTRY [ 0x02000000`00000000 - 0x00000000`00000000 ]
                   +0x2f8 LockedPagesList  : (null) 
                   +0x300 ThreadListHead   : _LIST_ENTRY [ 0x00000000`00000000 - 0xfffffa80`18dde680 ]
                   +0x310 SecurityPort     : 0xfffffa80`1a8aff80 Void
                   +0x318 Wow64Process     : (null) 
                   +0x320 ActiveThreads    : 0
                   +0x324 ImagePathHash    : 0
                   +0x328 DefaultHardErrorProcessing : 0x68
                   +0x32c LastThreadExitStatus : 0n0
                   +0x330 Peb              : 0x00000000`00000005 _PEB
                   +0x338 PrefetchTrace    : _EX_FAST_REF
                   +0x340 ReadOperationCount : _LARGE_INTEGER 0x0
                   +0x348 WriteOperationCount : _LARGE_INTEGER 0xf
                   +0x350 OtherOperationCount : _LARGE_INTEGER 0x2e
                   +0x358 ReadTransferCount : _LARGE_INTEGER 0x202
                   +0x360 WriteTransferCount : _LARGE_INTEGER 0x24c5880
                   +0x368 OtherTransferCount : _LARGE_INTEGER 0x5a0200
                   +0x370 CommitChargeLimit : 0x1407
                   +0x378 CommitChargePeak : 0
                   +0x380 AweInfo          : 0x00000000`00000048 Void
                   +0x388 SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
                   +0x390 Vm               : _MMSUPPORT
                   +0x418 MmProcessLinks   : _LIST_ENTRY [ 0x00000000`00000000 - 0xfffffa80`19323f50 ]
                   +0x428 HighestUserAddress : 0xfffff800`02c025e0 Void
                   +0x430 ModifiedPageCount : 0
                   +0x434 Flags2           : 0
                   +0x434 JobNotReallyActive : 0y0
                   +0x434 AccountingFolded : 0y0
                   +0x434 NewProcessReported : 0y0
                   +0x434 ExitProcessReported : 0y0
                   +0x434 ReportCommitChanges : 0y0
                   +0x434 LastReportMemory : 0y0
                   +0x434 ReportPhysicalPageChanges : 0y0
                   +0x434 HandleTableRundown : 0y0
                   +0x434 NeedsHandleRundown : 0y0
                   +0x434 RefTraceEnabled  : 0y0
                   +0x434 NumaAware        : 0y0
                   +0x434 ProtectedProcess : 0y0
                   +0x434 DefaultPagePriority : 0y000
                   +0x434 PrimaryTokenFrozen : 0y0
                   +0x434 ProcessVerifierTarget : 0y0
                   +0x434 StackRandomizationDisabled : 0y0
                   +0x434 AffinityPermanent : 0y0
                   +0x434 AffinityUpdateEnable : 0y0
                   +0x434 PropagateNode    : 0y0
                   +0x434 ExplicitAffinity : 0y0
                   +0x438 Flags            : 0x23fb
                   +0x438 CreateReported   : 0y1
                   +0x438 NoDebugInherit   : 0y1
                   +0x438 ProcessExiting   : 0y0
                   +0x438 ProcessDelete    : 0y1
                   +0x438 Wow64SplitPages  : 0y1
                   +0x438 VmDeleted        : 0y1
                   +0x438 OutswapEnabled   : 0y1
                   +0x438 Outswapped       : 0y1
                   +0x438 ForkFailed       : 0y1
                   +0x438 Wow64VaSpace4Gb  : 0y1
                   +0x438 AddressSpaceInitialized : 0y00
                   +0x438 SetTimerResolution : 0y0
                   +0x438 BreakOnTermination : 0y1
                   +0x438 DeprioritizeViews : 0y0
                   +0x438 WriteWatch       : 0y0
                   +0x438 ProcessInSession : 0y0
                   +0x438 OverrideAddressSpace : 0y0
                   +0x438 HasAddressSpace  : 0y0
                   +0x438 LaunchPrefetched : 0y0
                   +0x438 InjectInpageErrors : 0y0
                   +0x438 VmTopDown        : 0y0
                   +0x438 ImageNotifyDone  : 0y0
                   +0x438 PdeUpdateNeeded  : 0y0
                   +0x438 VdmAllowed       : 0y0
                   +0x438 CrossSessionCreate : 0y0
                   +0x438 ProcessInserted  : 0y0
                   +0x438 DefaultIoPriority : 0y000
                   +0x438 ProcessSelfDelete : 0y0
                   +0x438 SetTimerResolutionLink : 0y0
                   +0x43c ExitStatus       : 0n186368
                   +0x440 VadRoot          : _MM_AVL_TABLE
                   +0x480 AlpcContext      : _ALPC_PROCESS_CONTEXT
                   +0x4a0 TimerResolutionLink : _LIST_ENTRY [ 0x00000000`00000360 - 0x00000000`00000000 ]
                   +0x4b0 RequestedTimerResolution : 0
                   +0x4b4 ActiveThreadsHighWatermark : 0
                   +0x4b8 SmallestTimerResolution : 0
                   +0x4c0 TimerResolutionStackRecord : (null) 

Its image file name is :

            kd> db  fffffa80`18dde740+0x2d8
            fffffa80`18ddea18  00 00 00 00 00 00 00 00-53 79 73 74 65 6d 00 00  ........System..

So the Cid is the offset in the table divided by 4 :

            Offset / 4 = Cid

            0x10 / 4 = 0x04

## Several ways to locate the PspCidTable

PspCidTable is not an exported symbol, so we cannot use it directly in WDK.

1. Find the hardcoded offset in PsLookupProcessByProcessId routine:

            kd> u nt!PsLookupProcessByProcessId L10
            nt!PsLookupProcessByProcessId:
            fffff800`02d541fc 48895c2408      mov     qword ptr [rsp+8],rbx
            fffff800`02d54201 48896c2410      mov     qword ptr [rsp+10h],rbp
            fffff800`02d54206 4889742418      mov     qword ptr [rsp+18h],rsi
            fffff800`02d5420b 57              push    rdi
            fffff800`02d5420c 4154            push    r12
            fffff800`02d5420e 4155            push    r13
            fffff800`02d54210 4883ec20        sub     rsp,20h
            fffff800`02d54214 65488b3c2588010000 mov   rdi,qword ptr gs:[188h]
            fffff800`02d5421d 4533e4          xor     r12d,r12d
            fffff800`02d54220 488bea          mov     rbp,rdx
            fffff800`02d54223 66ff8fc4010000  dec     word ptr [rdi+1C4h]
            fffff800`02d5422a 498bdc          mov     rbx,r12
            fffff800`02d5422d 488bd1          mov     rdx,rcx
            fffff800`02d54230 488b0d9149edff  mov     rcx,qword ptr [nt!PspCidTable (fffff800`02c28bc8)]
            fffff800`02d54237 e834480200      call    nt!ExMapHandleToPointer (fffff800`02d78a70)
            fffff800`02d5423c 458d6c2401      lea     r13d,[r12+1]

we can get the second operand from the instruct before call to ExMapHandleToPointer.

2. Through \_KPCR

            kd> dt _KPCR fffff800`02bf3d00
            ntdll!_KPCR
               +0x000 NtTib            : _NT_TIB
               +0x000 GdtBase          : 0xfffff800`00b95000 _KGDTENTRY64
               +0x008 TssBase          : 0xfffff800`00b96080 _KTSS64
               +0x010 UserRsp          : 0x166f7b8
               +0x018 Self             : 0xfffff800`02bf3d00 _KPCR
               +0x020 CurrentPrcb      : 0xfffff800`02bf3e80 _KPRCB
               +0x028 LockArray        : 0xfffff800`02bf44f0 _KSPIN_LOCK_QUEUE
               +0x030 Used_Self        : 0x000007ff`fffd7000 Void
               +0x038 IdtBase          : 0xfffff800`00b95080 _KIDTENTRY64
               +0x040 Unused           : [2] 0
               +0x050 Irql             : 0 ''
               +0x051 SecondLevelCacheAssociativity : 0xc ''
               +0x052 ObsoleteNumber   : 0 ''
               +0x053 Fill0            : 0 ''
               +0x054 Unused0          : [3] 0
               +0x060 MajorVersion     : 1
               +0x062 MinorVersion     : 1
               +0x064 StallScaleFactor : 0xc90
               +0x068 Unused1          : [3] (null) 
               +0x080 KernelReserved   : [15] 0
               +0x0bc SecondLevelCacheSize : 0x600000
               +0x0c0 HalReserved      : [16] 0xbfbc2ae0
               +0x100 Unused2          : 0
               +0x108 KdVersionBlock   : (null) 
               +0x110 Unused3          : (null) 
               +0x118 PcrAlign1        : [24] 0
               +0x180 Prcb             : _KPRCB

but the KdVersionblock is null, a dead end.













