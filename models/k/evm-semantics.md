---
title: K Model of EVM Execution Environment
geometry: margin=2.5cm
...


```k
requires "evm-syntax.k"
```


Configuration
=============

We need to define the configuration before defining the semantics of any rules
which span multiple cells.

```k
module EVM-CONFIGURATION
    imports EVM-SYNTAX

    configuration <T>
                    // local execution
                    <k> $PGM:EVMSimulation </k>
                    <accountID> .AcctID </accountID>
                    <programCounter> 0 </programCounter>
                    <wordStack> .WordStack </wordStack>
                    <localMem> .Map </localMem>
                    // suspended processes
                    <callStack> .CallStack </callStack>
                    // account information
                    <accounts>
                        <account multiplicity="*">
                            <acctID> .AcctID </acctID>
                            <program> .Map </program>
                            <storage> .Map </storage>
                            <balance> 0:Word </balance>
                        </account>
                    </accounts>
                  </T>
endmodule
```

Machine Initialization
----------------------

This module just takes the parsed world definition (list of accounts and account
to start with) and loads it all into the correct cells of the configuration.

### Utilities

We'll start with some helpers which do "standard" things to our configuration.
These will make defining `EVM-INITIALIZATION` easier, and may be useful later.

```k
module EVM-INITIALIZATION-UTIL
    imports EVM-CONFIGURATION
```

We start out with setters for the `Program` and `Storage` associated to an
account. Because they are stored as maps, we have to keep a little bit of extra
state (what the next map key is) when actually running these.

```k
    syntax KItem ::= "#setAcctProgram" AcctID Int Program
                   | "#setAcctStorage" AcctID Int Storage
                   | "#setAcctProgram" AcctID Program
                   | "#setAcctStorage" AcctID Storage
                   | "#increaseAcctBalance" AcctID Word      [strict(2)]
                   | "#decreaseAcctBalance" AcctID Word      [strict(2)]

    rule #setAcctProgram ACCT PROGRAM => #setAcctProgram ACCT 0 PROGRAM [macro]
    rule #setAcctStorage ACCT STORAGE => #setAcctStorage ACCT 0 STORAGE [macro]

    rule #setAcctProgram ACCT N .Program => .
    rule #setAcctStorage ACCT N .Storage => .

    rule <k> #setAcctProgram ACCT (N => N +Int 1) (OpCode:OpCode ; Program => Program) ... </k>
         <account>
            <acctID> ACCT </acctID>
            <program> STORAGE:Map (. => N |-> OpCode) </program>
            ...
         </account>

    rule <k> #setAcctStorage ACCT (N => N +Int 1) (W0:Word , WS => WS) ... </k>
         <account>
            <acctID> ACCT </acctID>
            <storage> STORAGE:Map (. => N |-> W0) </storage>
            ...
         </account>

    rule <k> #increaseAcctBalance ACCT Balance => . ... </k>
         <account>
            <acctID> ACCT </acctID>
            <balance> X => X +Word Balance </balance>
            ...
         </account>

    rule <k> #decreaseAcctBalance ACCT Balance => . ... </k>
         <account>
            <acctID> ACCT </acctID>
            <balance> X => X -Word Balance </balance>
            ...
         </account>
```

The next setters allow setting information about the current running process,
and are useful at initialization as well as when `CALL` or `RETURN` is invoked.

```k

    syntax KItem ::= "#setAccountID" AcctID
                   | "#setProgramCounter" Word
                   | "#setWordStack" WordStack
                   | "#setLocalMem" MergeMemOp
                   | "#addToLocalMem" LocalMem
                   | "#setProcess" Process
                   | "#pushCallStack"
                   | "#popCallStack"

    syntax KResult ::= LocalMem

    rule <k> #setAccountID ACCTID => . ... </k>
         <accountID> _ => ACCTID </accountID>

    rule <k> #setProgramCounter PC => . ... </k>
         <programCounter> _ => PC </programCounter>

    rule <k> #setWordStack WS => . ... </k>
         <wordStack> _ => WS </wordStack>

    rule <k> #setLocalMem LM => . ... </k>
         <localMem> _ => LM </localMem>

    rule #setProcess { ACCT | PC | WS | LM }
         =>    #setAccountID ACCT
            ~> #setProgramCounter PC
            ~> #setWordStack WS
            ~> #setLocalMem LM

    rule <k> #pushCallStack => . ... </k>
         <accountID> ACCT </accountID>
         <programCounter> PC </programCounter>
         <wordStack> WS </wordStack>
         <localMem> LM </localMem>
         <callStack> CS => { ACCT | PC | WS | LM } CS </callStack>

    rule <k> #popCallStack => #setProcess P ... </k>
         <callStack> P:Process CS => CS </callStack>

    rule <k> #addToLocalMem LM => #setLocalMem #mergeMem {INITIAL| INITIAL | SIZE | LM} ... </k>
         <wordStack> (INITIAL : SIZE : WS) => WS </wordStack>

    rule <k> #mergeMem {INITIAL | CURRENT |  SIZE | LM} => #mergeMem{INITIAL | CURRENT +Word 1 | SIZE |  LM} ...</k> 
         <localMem> L[Current <- LM[Current -Word Initial]] </localMem>
         requires ((CURRENT -Int INITIAL) <=Int SIZE) andBool ((CURRENT -Int INITIAL) in keys(LM))

    rule <k> #mergeMem {INITIAL | CURRENT | SIZE | LM} => L ...</k> 
         <localMem> L </localMem>
         requires ((CURRENT -Int INITIAL) >Int SIZE) orBool (notBool ((CURRENT -Int INITIAL) in keys(LM)))
endmodule
```

### Initialization

Actual initialization is easy now, we just have to desugar the top-level syntax
into setting up the accounts (using our helpers), then setting the correct
process (also using the helpers).

```k
module EVM-INITIALIZATION
    imports EVM-INITIALIZATION-UTIL

    syntax KItem ::= EVMSimulation

    rule ACCTS:Accounts START ACCT => ACCTS ~> #setProcess { ACCT | 0 | .WordStack | .LocalMem }

    rule .Accounts => .
    rule ACCT:Account ACCTS:Accounts => ACCT ~> ACCTS

    rule <k> account:
             -   id: ACCT
             -   balance: BALANCE
             -   program: PROGRAM
             -   storage: STORAGE
         =>    #setAcctStorage ACCT STORAGE
            ~> #setAcctProgram ACCT PROGRAM
             ...
         </k>
         <accounts>
            ...
            (.Bag => <account>
                    <acctID> ACCT </acctID>
                    <program> .Map </program>
                    <storage> .Map </storage>
                    <balance> BALANCE </balance>
                  </account>)
         </accounts>

endmodule
```

Here we define how to get operands out of the `WordStack` and into the arguments
of various operators so that the already defined operations can act on them.

```k
module EVM-STACK
    imports EVM-CONFIGURATION
    imports EVM-INITIALIZATION-UTIL

    syntax KResult ::= LocalMem | Word

    rule <k> . => OP </k>
         <accountID> ACCT </accountID>
         <programCounter> PC => PC +Int 1 </programCounter>
         <account>
            <acctID> ACCT </acctID>
            <program> ... PC |-> OP ... </program>
            ...
         </account>

    rule <k> UOP:UnStackOp => UOP W0 ... </k>
         <wordStack> W0 : WS => WS </wordStack>

    rule <k> BOP:BinStackOp => BOP W0 W1 ... </k>
         <wordStack> W0 : W1 : WS => WS </wordStack>

    rule <k> TOP:TernStackOp => TOP W0 W1 W2 ... </k>
         <wordStack> W0 : W1 : W2 : WS => WS </wordStack>

    rule <k> W0:Int ~> #push => . ... </k>
         <wordStack> WS => W0 : WS </wordStack>

    rule <k> #checkStackSize => #stackSize WS ~> #checkStackSize ... </k>
         <wordStack> WS </wordStack>

endmodule
```


```k
module EVM-PROCESS-CALL
    imports EVM-STACK

    syntax GatherArgs ::= "#gatherArgs" "{" Word "|" Word "|" Word "|" LocalMem "}"
    syntax KItem ::= "#processCall" "{" AcctID "|" Word "}"
    syntax KItem ::= "#processReturn" "{" GatherArgs"}"       [strict]
                     | "#processReturn" "{" LocalMem "}"

    syntax KResult ::= LocalMem

    rule <k> #gatherArgs {N | S | N' | LM}
          => #gatherArgs {N +Word 1 | S | N' +Word 1 | LM[N' <- W]}
         ... </k>
         <localMem>... N |-> W ...</localMem>

    rule #gatherArgs {N | SIZE | N' | LM} => LM
         requires N' >=Int SIZE

    rule <k> CALL
          =>    #gatherArgs { BEGIN | SIZE | 0 | .LocalMem }
             ~> #processCall { ACCT | ETHER }
         ... </k>
         <wordStack> (ACCT : ETHER : BEGIN : SIZE : WS) => WS </wordStack>
    
    rule <k> RETURN => #processReturn { #gatherArgs { BEGIN | SIZE | 0 | .LocalMem } } ... </k>
            <wordStack> (BEGIN : SIZE : WS) => WS </wordStack>

    rule #processReturn {LM} => #popCallStack ~> #addToLocalMem LM

    rule JUMP W0 => #setProgramCounter W0
    
    rule JUMP1 W0 W1 => #setProgramCounter W0 requires notBool W1 ==Int 0
    rule JUMP1 W0 W1 => . requires W1 ==Int 0

    // TODO: How are we handling refunding unused gas?
    rule <k> LM:LocalMem ~> #processCall {ACCT | ETHER}
          =>    #decreaseAcctBalance CURRACCT ETHER
             ~> #increaseAcctBalance ACCT ETHER
             ~> #pushCallStack
             ~> #setProcess {ACCT | 0 | .WordStack | LM}
         ... </k>
         <accountID> CURRACCT </accountID>

endmodule
```

```k
module EVM
    imports EVM-INITIALIZATION
    imports EVM-PROCESS-CALL

endmodule
```

```
    // load/store elements in memory
    rule <k> MLOAD => . ... </k>
         <wordStack> W0 : WS => M : WS </wordStack>
         <localMem> ... W0 |-> M ... </localMem>

    rule <k> MLOAD => . ... </k>
         <wordStack> W0 : WS => 0 : WS </wordStack>
         <localMem> Rho:Map </localMem>
         requires notBool (W0 in keys(Rho))

    rule <k> MSTORE => . ... </k>
         <wordStack> W0 : W1 : WS => WS </wordStack>
         <localMem> ... W0 |-> (_ => W1) ... </localMem>

    rule <k> MSTORE => . ... </k>
         <wordStack> W0 : W1 : WS => WS </wordStack>
         <localMem> Rho:Map (.Map => W0 |-> W1) </localMem>
         requires notBool (W0 in keys(Rho))



    // call another process
    rule <k> CALL => #gatherArgs ARG0 ARGN 0 ~> #mkCall DEST .Map ~> #recieveReturn RET0 RETN ... </k>
         <wordStack> DEST : ETHER : ARG0 : ARGN : RET0 : RETN : WS => WS </wordStack>

    rule <k> #gatherArgs (ARG0 => ARG0 +Word 1) ARGN (N => N +Word 1) ~> #mkCall DEST (RHO:Map (.Map => N |-> M)) ...  </k>
         <localMem> ... ARG0 |-> M ... </localMem>
         requires ARG0 <Int ARGN

    rule <k> #gatherArgs (ARG0 => ARG0 +Word 1) ARGN (N => N +Word 1) ~> #mkCall DEST (RHO:Map (.Map => N |-> 0)) ...  </k>
         <localMem> LM </localMem>
         requires (ARG0 <Int ARGN) andBool (notBool (N in keys(LM)))

    rule #gatherArgs ARG0 ARGN N => .
         requires ARG0 >=Int ARGN
endmodule
```