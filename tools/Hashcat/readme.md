# Hashcat

```mermaid
mindmap
  root((HASHCAT))
    (INPUT LAYER)
      [Hash Targets]
        MD5 SHA1 SHA256 SHA512
        NTLM / NetNTLMv2
        bcrypt / scrypt / Argon2
        WPA2 / WPA3
        Kerberos TGS-REP
        Crypto Wallets
        400+ Hash Types
      [Candidate Sources]
        Wordlists rockyou.txt
        Masks ?u?l?l?d?d
        Rules best64.rule
        Combinator two lists
        Stdin Pipe
      [Config Options]
        Attack Mode -a 0 to 9
        Device Select -d 1,2,3
        Workload -w 1 to 4
        Optimized Mode -O
        Session and Restore

    (CORE ENGINE)
      [dispatcher.c]
        Work Distribution
        Multi GPU Balancing
        Keyspace Chunking
        Progress Tracking
      [autotune.c]
        kernel_accel
        kernel_loops
        kernel_threads
        Workload Profiles w1 to w4
      [rp.c Rule Engine]
        Lowercase Uppercase Toggle
        Append Prepend Replace
        Reverse Duplicate
        GPU Side Execution
        64 to 30000 Rules
      [wordlist.c]
        Memory Mapped IO
        Chunk Reading
        Stdin Support
      [hashes.c]
        Sort Digests
        Binary Search
        Bloom Filter Build
        Duplicate Remove
      [potfile.c]
        Skip Cracked Hashes
        Persistent Cache
        Auto Load on Start
      [monitor.c]
        Temperature Watch
        Fan Speed Query
        Power Usage
        Auto Throttle
      [selftest.c]
        Known Pair Verify
        Per Module Test
        Kernel Correctness

    (HARDWARE LAYER)
      [NVIDIA GPU]
        CUDA Cores 16384 on 4090
        RTX 4090 164 GHs NTLM
        RTX 3090 78 GHs NTLM
        CUDA and OpenCL Runtime
      [AMD GPU]
        Stream Processors
        RX 7900 XTX 68 GHs
        ROCm HIP Runtime
      [Intel GPU]
        Arc A770 8 GHs
        oneAPI Runtime
      [CPU Fallback]
        POCL Runtime
        Last Resort Only
      [Multi GPU]
        Near Linear Scaling
        Up to 128 Devices
        Independent Chunks
        No GPU Communication

    (KERNEL PIPELINE)
      [Stage 1 BASE]
        Load Base Words
        Prepare Hash State
        Store Intermediate
      [Stage 2 AMPLIFIER]
        Apply Rules amp_a0.cl
        Apply Masks amp_a3.cl
        Combinator amp_a1.cl
        Multiply Keyspace
      [Stage 3 COMPARE]
        Hash Final Candidate
        Bloom Filter Reject
        Binary Search Match
        Write Result Buffer

    (ATTACK MODES)
      [a0 Dictionary]
        Wordlist plus Rules
        straight.c
      [a1 Combinator]
        Word1 concat Word2
        combinator.c
      [a3 Brute Force]
        Mask plus Markov
        bruteforce.c
      [a6 Hybrid WL Mask]
        Word then Mask
      [a7 Hybrid Mask WL]
        Mask then Word
      [a9 Association]
        Targeted Per Hash

    (OUTPUT LAYER)
      [Potfile]
        hash colon plain pairs
        Auto loaded next run
      [Outfile -o]
        Custom path
        15 Format options
      [Screen Display]
        Speed H per second
        Progress Percent
        ETA Remaining
        Temperature
```


```mermaid
sequenceDiagram
    actor User
    participant CLI as main.c / CLI
    participant Core as hashcat_core
    participant Module as Hash Module
    participant Backend as backend.c
    participant GPU as GPU Device
    participant Potfile as  Potfile

    User->>CLI: hashcat -m 1000 -a 0 hash.txt wordlist.txt -r best64.rule

    Note over CLI,Core: INITIALIZATION PHASE

    CLI->>Core: hashcat_init()
    Core->>Core: Parse arguments & load hctune DB
    Core-->>CLI: Config ready 

    CLI->>Module: Load module_01000.c (NTLM)
    Module->>Module: module_hash_decode()
    Module->>Module: module_kern_type()
    Module->>Module: module_opti_type()
    Module-->>CLI: Module loaded 

    CLI->>Potfile: Check already cracked hashes
    Potfile-->>CLI: Skip N already cracked

    CLI->>Backend: backend_init()
    Backend->>GPU: Enumerate OpenCL / CUDA devices
    GPU-->>Backend: Device info (VRAM, CUs, clock)
    Backend-->>CLI: Devices ready 

    CLI->>GPU: Compile OpenCL kernels
    Note right of GPU: m01000_a0-optimized.cl\namp_a0.cl\ninc_hash_md5.cl
    GPU->>GPU: JIT compile kernels
    GPU-->>CLI: Kernels cached 

    Note over Core,GPU: AUTO-TUNE PHASE

    CLI->>GPU: autotune() — run test kernel KA=1 KL=1
    loop Auto Tune Loop
        GPU->>GPU: Run kernel, measure ms
        GPU-->>Core: Execution time result
        Core->>Core: Time < target? Double KA or KL
    end
    Core-->>CLI: Optimal KA=128 KL=256 found 

    CLI->>GPU: selftest() — known hash/plain pair
    GPU-->>CLI: Self test passed 

    Note over User,GPU: MAIN CRACKING LOOP

    loop Every Batch Until Keyspace Exhausted
        Core->>Core: Read next wordlist chunk
        Core->>Core: Apply rules (rp.c) — best64.rule
        Note right of Core: 64 rules × 10K words\n= 640K candidates/batch

        Core->>GPU: Transfer candidates\nHost → Device buffer (d_pws_buf)

        GPU->>GPU: Stage 1 — BASE KERNEL\nLoad base words

        GPU->>GPU: Stage 2 — AMPLIFIER KERNEL amp_a0.cl\nApply rules to each base word

        GPU->>GPU: Stage 3 — HASH KERNEL m01000_a0.cl\nNTLM hash each candidate

        Note over GPU: BLOOM FILTER CHECK
        GPU->>GPU: Check bitmap stage 1
        GPU->>GPU: Check bitmap stage 2
        GPU->>GPU: Binary search in d_digests_buf

        GPU-->>Core: Return result buffer

        alt Match Found 
            Core->>Potfile: Write hash:plain to potfile
            Core-->>User: CRACKED — Password1!
        else No Match
            Core->>Core: Update progress / speed / ETA
        end

        Note over Core: HARDWARE MONITOR
        loop Temperature Watch
            Backend->>GPU: Query temperature
            GPU-->>Backend: Temp °C
            alt Overheating > 90°C
                Backend->>GPU: Throttle kernel launch
                Note right of Backend: Reduce speed\nprevent damage
            else Normal Temp
                Backend->>GPU: Continue full speed
            end
        end
    end

    Note over User,Potfile: FINISH

    Core->>Core: hashcat_session_destroy()
    Core->>GPU: Free VRAM buffers
    Core->>Backend: Release OpenCL contexts
    Core-->>User: Session complete\nResults in potfile
  ```

```mermaid
sequenceDiagram
    participant WL as Wordlist
    participant RE as Rule Engine rp.c
    participant AMP as Amplifier amp_a0.cl
    participant HASH as  Hash Kernel
    participant BF as  Bloom Filter
    participant BS as  Binary Search
    participant OUT as  Output

    Note over WL,OUT:  SINGLE BATCH — Dictionary + Rules Attack

    WL->>RE: Base word: "password"
    Note right of WL: Read chunk from\nwordlist.txt

    loop For Each Rule in best64.rule
        RE->>AMP: Rule 1 — as-is → "password"
        RE->>AMP: Rule 2 — c → "Password"
        RE->>AMP: Rule 3 — c $1 → "Password1"
        RE->>AMP: Rule 4 — c $1 $! → "Password1!"
        RE->>AMP: Rule 5 — sa@ → "p@ssword"
        RE->>AMP: Rule 6 — r → "drowssap"
        RE->>AMP: Rule N... → ...
    end

    Note over AMP,HASH: All candidates sent to GPU in parallel

    AMP->>HASH: Candidate batch → GPU threads
    Note right of AMP: Each GPU thread\nhandles 1 candidate

    loop GPU Threads — SIMT Parallel
        HASH->>HASH: NTLM hash candidate
        HASH->>BF: Send digest for comparison
    end

    loop Bloom Filter Stages
        BF->>BF: Stage 1 — bitmap_s1 check
        alt Stage 1 Miss
            BF-->>HASH: Reject — not a match
        else Stage 1 Hit
            BF->>BF: Stage 2 — bitmap_s2 check
            alt Stage 2 Miss
                BF-->>HASH: Reject — not a match
            else Stage 2 Hit — Possible Match
                BF->>BS: Send for full comparison
            end
        end
    end

    BS->>BS: Binary search O(log n)\nin sorted digest list
    alt Match Found
        BS->>OUT: hash:Password1!
        OUT->>OUT: Write to potfile
        OUT->>OUT: Write to outfile
    else No Match
        BS-->>HASH: Continue next candidate
    end
```
```mermaid
sequenceDiagram
    participant User
    participant HC as Hashcat Core
    participant AT as autotune.c
    participant GPU as GPU

    Note over User,GPU: AUTO-TUNE SEQUENCE

    User->>HC: Launch hashcat -w 2
    HC->>AT: autotune_init()
    AT->>GPU: Set KA=1, KL=1 (minimum)

    loop Auto Tune — Find Optimal KA
        AT->>GPU: Launch test kernel with current KA
        GPU->>GPU: Execute kernel
        GPU-->>AT: Execution time = Xms
        AT->>AT: Time < 16ms target?
        alt Too Fast — Need More Work
            AT->>AT: Double KA value
            Note right of AT: KA: 1→2→4→8→16→32...
        else Time ≥ Target or Mem Limit
            AT->>AT: Use previous KA value
        end
    end

    loop Auto Tune — Find Optimal KL
        AT->>GPU: Launch test kernel with current KL
        GPU->>GPU: Execute kernel
        GPU-->>AT: Execution time = Xms
        AT->>AT: Time < 16ms target?
        alt Too Fast — Need More Work
            AT->>AT: Double KL value
            Note right of AT: KL: 1→2→4→8...256
        else Time ≥ Target or Mem Limit
            AT->>AT: Use previous KL value
        end
    end

    AT-->>HC: Optimal params found\nKA=128 KL=256 KT=64
    Note right of AT: -w 1 → 8ms target\n-w 2 → 16ms target\n-w 3 → 64ms target\n-w 4 → 256ms target

    HC-->>User: Auto tune complete \nBegin cracking at full speed
```
```mermaid
sequenceDiagram
    participant HC as Hashcat
    participant BE as backend.c
    participant OCL as OpenCL ICD
    participant NV as NVIDIA GPU
    participant AMD as AMD GPU
    participant INTEL as Intel GPU
    participant CPU as CPU Fallback

    Note over HC,CPU:  BACKEND DEVICE DISCOVERY

    HC->>BE: backend_init()
    BE->>OCL: clGetPlatformIDs()
    OCL-->>BE: Platforms found [NVIDIA, AMD, Intel, CPU]

    BE->>NV: clGetDeviceInfo(CL_DEVICE_NAME)
    NV-->>BE: RTX 4090\n24GB VRAM\n16384 Cores\n2520 MHz
    BE->>NV: clCreateContext()
    NV-->>BE: Context 

    BE->>AMD: clGetDeviceInfo(CL_DEVICE_NAME)
    AMD-->>BE: RX 7900 XTX\n24GB VRAM\n6144 SPs\n2500 MHz
    BE->>AMD: clCreateContext()
    AMD-->>BE: Context 

    BE->>INTEL: clGetDeviceInfo(CL_DEVICE_NAME)
    INTEL-->>BE: Arc A770\n16GB VRAM\n512 EUs
    BE->>INTEL: clCreateContext()
    INTEL-->>BE: Context 

    BE->>CPU: clGetDeviceInfo(CL_DEVICE_NAME)
    CPU-->>BE: Intel i9-13900K\n24 Cores\nFallback Mode
    BE->>CPU: clCreateContext()
    CPU-->>BE: Context 

    Note over BE,CPU: KERNEL COMPILATION PER DEVICE

    loop For Each Device
        BE->>BE: Select kernel source\nm01000_a0-optimized.cl
        BE->>NV: clBuildProgram(source, -D flags)
        NV->>NV: JIT Compile kernel
        NV-->>BE: Binary ready 
        BE->>BE: Cache to ~/.hashcat/kernels/
    end

    Note over BE,CPU: MEMORY ALLOCATION PER DEVICE

    BE->>NV: clCreateBuffer(d_pws_buf)
    BE->>NV: clCreateBuffer(d_digests_buf)
    BE->>NV: clCreateBuffer(d_result)
    BE->>NV: clCreateBuffer(d_bitmap)
    NV-->>BE: Buffers allocated 

    BE-->>HC: All devices ready\nNVIDIA + AMD + Intel + CPU
```
```mermaid
sequenceDiagram
    participant HC as Hashcat
    participant MOD as module_01000.c
    participant HASH as Hash Storage hashes.c
    participant POT as Potfile potfile.c
    participant GPU as GPU

    Note over HC,GPU: MODULE LOAD & HASH PARSE SEQUENCE

    HC->>MOD: Load module for -m 1000 (NTLM)
    MOD-->>HC: module_kern_type = KERN_TYPE_NTLM
    MOD-->>HC: module_opti_type = OPTI_TYPE_OPTIMIZED
    MOD-->>HC: module_salt_type = SALT_TYPE_NONE
    MOD-->>HC: module_dgst_size = DGST_SIZE_4_4
    MOD-->>HC: module_pw_max = PW_MAX_31
    Note right of MOD: Optimized kernel:\nmax 31 char passwords\n2-3x faster than pure

    HC->>MOD: module_hash_decode(hash_line)
    loop Parse Each Hash From File
        MOD->>MOD: Validate hash format
        MOD->>MOD: Extract digest bytes
        MOD->>HASH: Store in hashes_buf[]
        HASH->>HASH: Sort for binary search
    end

    HASH->>POT: Check potfile for each hash
    loop Potfile Lookup
        POT->>POT: Search potfile cache
        alt Already Cracked
            POT-->>HASH: Skip this hash 
        else Not Cracked
            HASH->>HASH: Keep in active list
        end
    end

    HASH->>HASH: Build bitmap / bloom filter
    Note right of HASH: bitmap_s1_a/b/c/d\nbitmap_s2_a/b/c/d\nMulti-stage false\npositive reduction

    HASH->>GPU: Upload digests → d_digests_buf
    HASH->>GPU: Upload bitmaps → d_bitmap_s1/s2
    GPU-->>HC: Ready to crack 

    HC->>MOD: module_st_hash() — self test
    MOD-->>GPU: Known plain → hash pair
    GPU->>GPU: Hash the known plain
    GPU-->>HC: Result matches? Y or N
    Note right of HC: Verifies kernel\nworks correctly\nbefore real attack
```
```mermaid
sequenceDiagram
    participant User
    participant HC as Hashcat
    participant MASK as Mask Engine
    participant MARKOV as Markov Chain
    participant GPU as  GPU
    participant OUT as Output

    Note over User,OUT: BRUTE FORCE MASK ATTACK -a 3 ?u?l?l?l?d?d

    User->>HC: hashcat -m 1000 -a 3 hash.txt ?u?l?l?l?d?d

    HC->>MASK: Parse mask pattern
    MASK-->>HC: Charset map\n?u = ABCDEFGHIJKLMNOPQRSTUVWXYZ\n?l = abcdefghijklmnopqrstuvwxyz\n?d = 0123456789
    HC->>MASK: Calculate total keyspace
    MASK-->>HC: 26×26×26×26×10×10\n= 45,697,600 candidates

    HC->>MARKOV: Load hashcat.hcstat2
    Note right of MARKOV: Markov chains rank\ncharacter sequences\nby real-world frequency
    MARKOV->>MARKOV: Build transition table
    MARKOV-->>HC: Probability-ordered\ncandidate sequence 
    Note right of HC: "Pass12" tested before\n"Zzzz99" — smarter order

    loop Keyspace Batch Loop
        HC->>MASK: Generate next batch
        MASK->>MARKOV: Order by probability
        MARKOV-->>GPU: Batch → d_markov_css_buf

        GPU->>GPU: Stage 1 — BASE KERNEL\nGenerate candidates from mask

        GPU->>GPU: Stage 2 — AMPLIFIER amp_a3.cl\nExpand with charset positions

        GPU->>GPU: Stage 3 — HASH + COMPARE\nNTLM hash each candidate

        GPU-->>HC: Results buffer

        alt Match Found
            HC->>OUT: CRACKED
            OUT-->>User: Hash → "Pass12"
        else No Match
            HC->>HC: Next batch
            Note right of HC: Progress: X/45,697,600\nSpeed: 164 GH/s\nETA: Xs remaining
        end
    end
```
```mermaid
sequenceDiagram
    participant HC as Hashcat
    participant MON as monitor.c
    participant GPU as GPU
    participant THR as Thread Manager
    participant STATUS as status.c
    participant User

    Note over HC,User: HARDWARE MONITOR + STATUS LOOP

    HC->>THR: Spawn monitor thread
    HC->>THR: Spawn status thread
    HC->>THR: Spawn cracking threads

    Note over MON,GPU: Runs in parallel with cracking

    loop Monitor Loop — Every 1 Second
        MON->>GPU: Query temperature
        GPU-->>MON: Temp = 78°C

        alt Temp > 90°C CRITICAL
            MON->>HC: EMERGENCY throttle
            HC->>GPU: Reduce kernel_accel
            MON-->>User: Warning: High temp throttling
        else Temp > 80°C WARNING
            MON->>HC: Soft throttle
            HC->>GPU: Slight reduction
        else Temp OK 
            MON->>HC: Continue full speed
        end

        MON->>GPU: Query fan speed RPM
        MON->>GPU: Query GPU utilization %
        MON->>GPU: Query power usage W
        GPU-->>MON: Fan=85% Util=99% Power=450W
    end

    loop Status Display — Every 10 Seconds
        STATUS->>HC: Request progress data
        HC-->>STATUS: candidates_done\nkeyspace_total\ntime_started
        STATUS->>STATUS: Calculate speed H/s
        STATUS->>STATUS: Calculate ETA
        STATUS->>STATUS: Calculate progress %

        STATUS-->>User: Speed: 164.5 GH/s\nProgress: 12.4%\nETA: 00:04:32\nTemp: 78°C\nFan: 85%
    end

    loop User Interaction
        User->>HC: Press S — show status
        HC-->>User: Full status report

        User->>HC: Press P — pause
        HC->>GPU: Halt kernel launches
        HC->>HC: Save checkpoint restore.file
        HC-->>User: Session paused 

        User->>HC: Press R — resume
        HC->>HC: Load restore.file
        HC->>GPU: Resume from checkpoint
        HC-->>User: Session resumed 

        User->>HC: Press Q — quit
        HC->>HC: hashcat_session_destroy()
        HC-->>User: Goodbye 
    end
```
