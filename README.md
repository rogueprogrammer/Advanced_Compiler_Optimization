# Advanced_Compiler_Optimization

Parser → Optimizer → Code Generation

OPTIMIZER:
1. Compute info about a program
2. Use that info to perform program transformations.




ALIASING:

2 variables are aliased if they refer to the same memory location.

	Void * x;
	….
	Int a = 1;
	int b = 2;
	if(exp) {
		x = &a; //x and a are alias
	}


INTERMEDIATE REPRESENTATIONS:
FE: Lexical analysis (Tokenizer) → Syntax Analysis (Create an internal representation, abstract syntax tree (AST)) → IR Generation
(IR)
Good representation of the program
High Level code → IR (good for optimization) → low level machine code
Original Motivation: Have a universal IR to translate various high level languages into different architectures (x86, Power PC, 390, etc)
I.R.s : Byte code, wcode, LLVM-IR, RTL
IR for us is meant to make optimization easier (needs to have enough info to optimize)
Front-end → IR → Backend ( optimization and code generation ) 
Backend: IR → Analysis → Transformation → Resource Alocation (Registers, Instructions) → Code Generation

DESIGNING IR 's:

no fixed rules to design and structure your code
not all “levels” of IR good for all purposes
there are High-level IR (HIR), Medium level IR (MIR), Low level IR (LIR)
HIR: relatively close to the code
the classic HIR is the AST
MIR:
“usual” level for optimizations
control flow via explicit branch and test
still have variables, procedure calls, returns, array indexing
LIR:
close to machine code
register offsets
expand array indexing (offset calculations, dereferencing, etc)
register allocation

HL → HIR → ( do analysis, opts on HIR) → MIR ( do analysis, opts on MIR) → LIR ( do analysis, opts on LIR) → code

	“Traditional MIR” (focus of this course)
3 address code
every statement involves <= 3 addresses (variables)

		z = (x * 3) + 2 + a;
		if(z[3] == z[7]){ …. }
		
	3 address code:
	t1 = x * 3
	t2 = t1 + 2
	t3 = t2 + a
	z = t3
	t4 = 3 * 4 //z[3]
	t5 = z + t4 //z[3]
	t6 = *t5 //z[3]

4 tuples (another form of repping 3 address code):
(*, t1, x, 3)    //t1 = x * 3
(+ , t2, t1, 2) 
…

Index list:
(1) (* , x, 3)
(2) (+ , (1), 2)
(3) (+ , (2), *)
(4) …
this form of 3 address code (3 tuples) is more expensive than using 4 tuples

So, essentially MIR is a list of 3 address-code
a structured MIR is a Control Flow Graph (CFG)
replaces GOTO and labels with arrows
it would be nice if every use had exactly one definition
SSA form (Static single assignment)
CFG with exactly one definition
how to do one definition (rename each definition)




CONTROL FLOW ANALYSIS: (Take an IR, and build a CFG)
recover structure in an IR
1. Generate Basic Blocks
we start off with a stream of 3 address-code
we want to divide it into basic blocks (BB 's)
enter at the top , and leave at the bottom. Execute code in order from top to bottom of box
look for leaders (the statement which is start of a basic block)
Leader Properties:
1. First statement is our leader
2. Any statement that is a target of a conditional or unconditional GOTO (Labels) indicate a new BB, and is thus a leader
3. Any statement that follows a conditional or unconditional GOTO is a Leader
		- A BB is a leader starting from but not upto the next leader

2. Generate Extended Basic Blocks (EEB 's)
larger chunks of straight line code
maximal sequence of instructions beginning with a leader and containing the”join-node” (2 arrows incoming) 
Reverse EBB (constructed EBB bottom up)
3. Detecting Loops / Recursive cycles
- So we have a CFG already now
Dominance – a node d dominates node n (d dom n or dom(d, n)) if every path from the initial start node to n must pass through d.
Trivial condition: every node dominates itself
Then initial node dominates everything.
We can draw dominance trees (connected acyclic graphs)
Computing Dominance:
a dom b iff a is the unique predecessor of b or for all predecessors of b, a dom b.
Algo to Compute Dom(a) → set of nodes:
Dom(entry) = {entry}
Dom (for all n, n != entry) = N (N = set of all nodes)
DO while(change)
change = false;
For all n, n != entry
T = N //temp T is equal to the set of all nodes
For each predecessor p of n
T (Intersect= ) Dom(p)
D = {n} (Union) T
If D != Dom(n)
change = true
Dom(n) = D
Post dominates (PDOM) – a pdom b if all 
Find the loops
Back edge: x (tail)  → y (head). An edge (x, y) is a back edge if its head dominates tail (y dom x)
Given a back edge b: (x → y), the natural loop of b is the node y & the set of nodes that can reach x without passing through y. y is our loop header

4. Reducibility / Well formedness (reducing graphs)
All loops are well formed. => There are no jumps into / out of loops
A graph G = (N, E) is reducible iff all loops are natural loops
meaning: E can be partitioned into 2 sets:
1. Forward edges Ef
2. Backend edges Eb
st (N, Ef) forms a DAG, headed by the entry node with all nodes reachable and all Eb's are backedges. 

DATA FLOW ANALYSIS:
	- Levels of Analysis:
Local Analysis – analyze within a Basic block (BB)
Global Analysis – analyze CFG flowing info across Bbs
(aka Intraprocedural Analysis) ← Intraprocedural means within procedure
Interprocedural Analysis ← Analyze entire program at once
 	Using the CFG to do dataflow
Reaching definition : which definition reaches which uses
may reach problem – all posible definitions that could reach
vs must reach problems – all possible def's that will reach
In general , we want this information at every program point (not necessarily at specific use points)
Process:
1. Form BB and the CFG
2. State the problem precisely
3. Formulate data flow equations
4. Solve the equations using an iterative solver

Rdefs (Reaching definitions)
2) A definition d (x = y (op) z) of x reaches a point p in the program if there exists a path from d to p that does not pass through any other definition of x. → May reach














	
