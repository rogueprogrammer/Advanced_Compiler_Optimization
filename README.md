Parser → Optimizer → Code Generation

OPTIMIZER:
1. Compute info about a program
2. Use that info to perform program transformations with the goal of optimizing the run-time execution.




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
D = {n} ∪  T
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

These data flow analysis RDEFs are used to tell us if a variable defined a certain way is still within scope. Can help us in register allocation →  we get rid of the variable stored in register?

PARTIALLY ORDERED SETS (poset)
elements, ordering, relations
{a, b, c, …} <= 
Partial means – order is not defined on every pair
⊆   ← means subset
a ⊆ b may be true, but we may not be sure if b ⊆ a. This is what partially ordered means.




Flow function (FF): functions on the elements on our domain, are monotonic (ordered preserving)
example of monotonicity: if  in1 <= in2             =>    f(in1) <= f(in2)
Note that we are not talking about monotonically increasing (x <= f(x)).

Each FF has a fixed gen set (definitions created) and kill set (definitions removed). 
. .
∪ 


LIVE VARIABLE ANALYSIS
find paths in the program and redefinitions of variables
A variable v is alive at program point p if there exists a path from p to the end of the program such that v is used before being redefined. 
Confluence: 
Come up with data flow equations:
in(si) = gen(si) ∪  [out(si) – kill(si)]
EG: if f: x = y op z
gen = {y, z}
kill = {x}


Problem: Very busy expressions
1. Domain : set of all expressions in our program
2.  An expression “x op y” is very busy at point p if “x op y” is evaluated in al paths from p to the end, and neither x nor y is redefined between p and the expressions
3. Backward
4. Confluence is intersection
5. f: x = y op z
Gen = {y, z}
kill = {x}
in(f) = gen(f) ∪  [out(f) – kill(f)]


EARLY OPTIMIZATIONS:
Constant Folding (Evaluation)
x = 3 + 2 => x = 5
watch out for overflow and error conditions (eg 32 bit vs 64 bit, division by 0)
Scalar Replacement of Aggregates
Eg: 
typedef struct {
	   	int x; int y;
	       } coord;
	instead of:
	coord z; z.x, z.y, use int z_x, z_y
Algebraic simplifications:
I * 0 => 0 or I + 0 => I
Boolean short curcuiting:
b || 1 => 1
b && 1 => b
     

Copy Propogation:
eg: x = y.
Just replace all instances of x with y, remove above statement.
DU-Chains: At a def, give a list of all uses that are reached by the definitions
UD-Cchains: At a use, give a list of all defs that reach the use
For each copy statement S: “x = y” do:
If every use of x which is replaced by this def by x (DU chain) is also reached by this in the reaching copies sense,
replace each use of x by every use of y
delete the copy statement. 
Dead code elimination
eliminate a statement where no variables in that statement is used
EG: 
x = y op z
if x is never used in the future → dead code, then we dont need it
if we remove it, we remove every use of y,z
	Testifdead(S)
		if the DU-CHAIN of the LHS of S is empty then:
			remove S (mark as dead)
			For each var v in RHS of S
				for each def in the UD chain of V
					remove use of v from the definition (modify its DU chain)
					Testifdead(def)

REDUNDANCY ELIMINATION (RE)
Common Subexpression elimination (CSE)
Local form: within a BB
EG: 
x = a + b; //stmt 1
y = c + d; //stmt 2
z = a + b; //stmt 3
c = d + 1; //stmt 4
u = a + b; //stmt 5
			use 5 tuples < pos, opnd1, op, opnd2, temp>
			Basic process: 
look at the statement → expression, see if there's a match in our list of 5 tuples
if not → create one
if yes → if the temp value is null
create a new temp ti → change the null to ti
insert ti = opnd1 op opnd2 prior to pos
replace stmt at pos with LHS = ti
replace current stmt with LHS = ti
                                                  → if the temp value is not null by tj
						replace the current stmt by LHS = tj

			EG: From the above code, the local form:
1. < 1, a, +, b, null>
2. < 1, a, +, b, null>, <2, c, +, d, null>
3. statement 1 is now: t1 = a + b; x = t1; statement 3 is now: z = t1;  < 1, a, +, b, t1>. <2, c, +, d, null>
4.  < 1, a, +, b, t1>, < 4, d, + 1, null>
5. statement 5 becomes u = t1; 
Forward Substitution
opposite of CSE

LOOP INVARIANT CODE MOTION
if instructions inside a loop capture the same thing in every iteration, move outside loop


LOOP OPTIMIZATIONS:
look for nice loops
for (exp1 ; exp2; exp3)
body
exp1 – asign a value to an integer loop index I
exp2 – compares I to a loop constant
exp3 – increments or decrements I by a loop constant
body – contains no definitions of I
no arbitrary loop exits
→ this is essentially a very simple loop , easy to understand loop index

look for induction variables – variables that change in an arithmetic fashion (ie increments, decrements), forming some kind of arithmetic sequence

ALIASING & Points-To
Aliasing: 2 variables refer to the same memory location
eg: x = &y (x and y are aliases)
not only refers to pointers, but also objects
Points to: 
Hierarchy of Points-to problems:
1) Aliases due to call-by-reference
eg: 
int a[], b[];     
		main(){
		m1:	foo(a, b);
		m2:	foo(a,a);
		}


		void foo(int x[], int y[]){
		s1:	x[i] = …
		s2:	… = y[i]
		s3:	… = a[i]
		}
	in foo() for m1, x and a are aliased, and y and b are aliased.

we can get aliases between parameters, between parameters and globals
to avoid computing the same results twice as in foo, just do an if statement:
if(x == y){ … }

	f(){
		int x, y, z, *p, *q;
		p = &x;
		if(...) q = & y; //p → x, q-> y
		else q = &z; //p → x, q->z
		*p = *q; //may aliasing : q → y or z
		p = q;
	}
	for may-aliasing, we put a ? mark at the end of the statement, for must aliasing (aliasing we know for sure is true), we put ! At the end of the statement. 


Flow sensitive points-to Analysis
Basically doing analysis of aliasing
expensive


2) Use of indication operators (&, *)
We assign types to each pointer, and do layers of direction and indirection, and compare types at each step, to see if the types match when we access and assign pointers. 
3) Dynamic allocation
Steensguard almost linear pointer-to algorithm
tries to build storage shape graph\
INTRAPROCEDURAL ANALYSIS
not done in practice, too expensive, and when would it be done (given all the DLLs, SLLs, etc) → Link time? Not worth it.

REGISTER ALLOCATION (RA)
- in the IR 's (Intermediate Representations) , we used as many vars as we like
where are variables stored → on the stack
But registers are much much faster to access
so, let's try to use as many registers as possible for storing variables
Different levels of allocation
1. Expression level: 
create expresion tree. Do a post order traversal on the tree, allocating registers to variable nodes in the tree
 	- 2. Basic Block Register Allocation:


