# Some tests of unrolling the 0.14.0 XOD generated code

See the XOD project in xod-optimization-test/. This README, and the test code, and the unrolling code, are at <https://github.com/awgrover/XOD-Unroll-Test>.

The test XOD project is a fairly simple one that just adds some numbers, does an ifelse and a digitalWrite. It would only run once, since the top of all the branches are constants.

# Configuration

XOD 0.14.0
Arduino ID 1.8.1
Compile for UNO

# Measuring

We'll compare the standard "generated code" ("Deploy: Show Code for Arduino") with some "hand" attempts at unrolling. I'll compile for the UNO as a reference, using the Arduino IDE.

I'll make several arduino projects, see all the subdirectories in the git archive:

* The original xod generate code
* Unrolled

And a doctored version of each with timing code, and dirtying the graph each time (so it re-runs the graph):

    void loop() {
        static int loop_count = 0;
        static unsigned long start_time = millis();
        
        // the original XOD body:
        xod::idle();
        xod::runTransaction();
        
        // doctoring: say how long 100 executions takes
        memset(g_dirtyFlags, 255, sizeof(g_dirtyFlags));
        loop_count++;
        if (loop_count >= 100) {
            Serial.println( millis() - start_time );
            start_time = millis();
            loop_count = 0;
            }
        }
             
# Unrolling

I'm more concerned with execution speed than RAM usage, but expect to see some memory improvements.

I expect that unrolling, and especially inlining the "evaluate" call, should make more static analysis possible for the compiler, so code size should decrease (unrolling would increase it though!). And, inlining allows removing some stored values. Also, the unrolling and inlining removes many fetches (most of the fetches are from PROGMEM, so are particularly slow).

For the first test, unroll the loop, and inline the "evaluate" call.

## The Loop

The generated code has done a topological-sort of the graph, and then a breadth-first traverse to produce the node-list(s). It runs the "program" like this:

    for (NodeId nid = 0; nid < NODE_COUNT; ++nid) {
        if (isNodeDirty(nid)) {
            evaluateNode(nid);
    ...

## Data structures

Basically, the data is several lists, holding the dirty flags, evaluate function pointer, parent-nodes, dependant nodes, etc. All the data is put in PROGMEM (except the dirty-flags).

## isNodeDirty(nid)

Is just

    g_dirtyFlags[nid] & 0x1;

and 

    DirtyFlags g_dirtyFlags[NODE_COUNT] = {
        ... for each node
    
So, I'll just use that code.

## evaluateNode

Gets the function pointer for the node-class's "evaluate" and calls it with the node-index:

    EvalFuncPtr eval = getWiringValue<EvalFuncPtr>(nid, 0);
    eval(nid);

and 

    g_wiring[NODE_COUNT] PROGMEM = {
        pointer to a Wiring struct
        ... for each node

'Wiring' is a structure, possibly different for each type of node:

    // xod__core__add's:
    struct Wiring {
        EvalFuncPtr eval;
        UpstreamPinRef input_X;
        UpstreamPinRef input_Y;
        const NodeId* output_VAL; // a list of node-indexes
    };

And an "instance" of Wiring looks like:

    const xod__core__add::Wiring wiring_6 PROGMEM = {
        &xod__core__add::evaluate,
        ....
 
Some gyrations have to be gone through to return the function pointer from PROGMEM storage.

I'll just put the function call inline.

# A program to edit the XOD code

Of course I'll use Perl to edit the generated code and create the unroll code. The XOD code is very stereotyped, so it should be reliable to just do text processing on it.

# First Unrolling Performance Test

I only unrolled the loop, inlining the "evaluation" call. I even left the unused EvalFuncPtr in the Wiring structures.

    ./unroll arduino_optimization_test/arduino_optimization_test.ino

    Project    Flash/RAM    code-growth%    ms/100-loops    speedup%    
    (UNO board)    32256/2048            
    XOD Generated Code    2756/105            
    XOD w/timing    3990/294    45%    23ms    
    Unroll1    3214/105    17%        
    Unroll1 w/timing    4464/294    12%    18ms    22%

Adding the timing code causes a large increase in code and RAM: I think that's all due to Serial. 

As I guessed, unrolling speeds things up quite a bit (22% faster). I'm still guessing that is because of skipping a lot of fetches from progmem. 

Code size goes up, not surprisingly, since I unroll the loop. The percentages don't take into account the overhead for the basic arduino code (444/9 for null void() and setup()), etc. A better percent would take into account the XOD library code, etc.

Without the timing code, RAM usage doesn't change. That's not surprising, since the RAM is almost all the dirty flags and the node's output values.

## Conclusion

Significant speed improvement by unrolling, but it's not obvious that there was much code-optimization.

# Remove EvalFuncPtr Performance Test

I deleted the unused EvalFuncPtr in the Wiring structures.

	code/ram code-growth%
	3070/105	14%

It's not surprising to see an improvement, but we just saved 14.4 bytes per node: 144 bytes, 10 nodes. That seems like a funny number.

