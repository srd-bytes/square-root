# Generic Verification Template

This template provides a reusable verification framework for
**combinational Verilog designs**. The philosophy is simple:

-   **Python** acts as the **golden reference model**.
-   **Python generates** input vectors and expected outputs.
-   **Verilog** reads those vectors from text files.
-   The **DUT (Design Under Test)** is stimulated with the inputs.
-   The testbench compares the DUT output against the Python-generated
    expected output.
-   Only **failing test cases** are printed, followed by a summary.

------------------------------------------------------------------------

# Verification Flow

``` text
            Python Golden Model
                    │
                    │
          Generate Test Vectors
                    │
      ┌─────────────┴──────────────┐
      │                            │
 inputs.txt                 expected.txt
      │                            │
      └─────────────┬──────────────┘
                    │
             Verilog Testbench
                    │
              Device Under Test
                    │
                 Compare
                    │
        PASS (silent) / FAIL (log)
                    │
          Final Summary + Waveform
```

------------------------------------------------------------------------

# Structure

``` text
project/
│
├── python/
│      model.py
│      generate_vectors.py
│
├── vectors/
│      input.txt
│      expected.txt

```

------------------------------------------------------------------------

# Generating Test Vectors using the python model

------------------------------------------------------------------------

## Implementation

``` python
class VectorWriter:

    def __init__(self,
                 input_filename="input.txt",
                 output_filename="expected.txt"):

        self.fin = open(input_filename, "w")
        self.fexp = open(output_filename, "w")

    def write(self, inputs, outputs):

        self.fin.write(
            " ".join(map(str, inputs)) + "\n"
        )

        self.fexp.write(
            " ".join(map(str, outputs)) + "\n"
        )

    def close(self):

        self.fin.close()
        self.fexp.close()
```

------------------------------------------------------------------------

# Writing Test Vectors

The `write()` method accepts two lists.

``` python
writer.write(inputs, outputs)
```

-   `inputs` : List of DUT input signals
-   `outputs`: List of expected DUT output signals

Each call writes **one complete test vector**.

Example:

``` python
writer.write(
    ["00000010", "00000011"], # inputs
    ["00000101"] # outputs
)
```

Generates

**input.txt**

    00000010 00000011

**expected.txt**

    00000101

------------------------------------------------------------------------

# Typical Usage

``` python
writer = VectorWriter()

for A in range(256):
    for B in range(256):

        expected = my_python_model(A, B)

        writer.write(
            [
                format(A, "08b"),
                format(B, "08b")
            ],
            [
                expected
            ]
        )

writer.close()
```

------------------------------------------------------------------------

# Generated Files

After several calls to `write()`:

**input.txt**

    00000000 00000000
    00000000 00000001
    00000000 00000010
    ...

**expected.txt**

    00000000
    00000001
    00000010
    ...

Each line in both files corresponds to the same test case.

------------------------------------------------------------------------

# Verification Flow

    Python Model
            │
            ▼
    Compute Expected Output
            │
            ▼
    VectorWriter
            │
            ├──────────► input.txt
            │
            └──────────► expected.txt
                         │
                         ▼
                Verilog Testbench
                         │
                         ▼
                   Compare with DUT
------------------------------------------------------------------------

# Generic Verilog Testbench

``` verilog
`timescale 1ns/1ps

module tb;

parameter WIDTH = 8;

reg  [WIDTH-1:0] A;
reg  [WIDTH-1:0] B;
wire [WIDTH-1:0] Y;

//-------------------------------
// DUT
//-------------------------------
my_design dut
(
    .A(A),
    .B(B),
    .Y(Y)
);

//-------------------------------
// File Handles
//-------------------------------
integer fin;
integer fexp;
integer status;

reg [WIDTH-1:0] expected;

// Statistics
integer total;
integer passed;
integer failed;

//-------------------------------
// Waveform Dump
//-------------------------------
initial begin
    $dumpfile("wave.vcd");
    $dumpvars(0,tb);
end




//-------------------------------
// Main Test Process
//-------------------------------

initial begin

    //--------------------------------------------Opening files------------------------------
    fin  = $fopen("../vectors/input.txt","r");
    fexp = $fopen("../vectors/expected.txt","r");

    if(fin==0) begin
        $display("Cannot open input.txt");
        $finish;
    end

    if(fexp==0) begin
        $display("Cannot open expected.txt");
        $finish;
    end
    //--------------------------------------------------------------------------------------

    total  = 0;
    passed = 0;
    failed = 0;

    while(!$feof(fin)) begin

        status = $fscanf(fin,"%b %b\n",A,B);
        status = $fscanf(fexp,"%b\n",expected);

        #1;

        if(Y===expected)
            passed++;
        else begin
            failed++;

            log_fail();
        end

        total++;

    end

    log_summary();

    $finish;

end

endmodule

//------------------------------------------------Reusable blocks----------------------------------

task automatic log_fail;
begin

    $display("\n====================================");
    $display("[%0t] TEST %0d FAILED",$time,total);
    $display("------------------------------------");

    $display("Inputs");
    $display("  A        = %b",A);
    $display("  B        = %b",B);

    $display("");

    $display("Expected");
    $display("  Y        = %b",expected);

    $display("");

    $display("Observed");
    $display("  Y        = %b",Y);

    $display("");

    $display("Difference");
    $display("  XOR      = %b",expected ^ Y);

    $display("====================================");

end
endtask

task automatic log_summary;
begin
    $display("\n====================================");
    $display("Simulation Summary");
    $display("------------------------------------");
    $display("Total Tests : %0d",total);
    $display("Passed      : %0d",passed);
    $display("Failed      : %0d",failed);

    if(failed==0)
        $display("RESULT      : ALL TESTS PASSED");
    else
        $display("RESULT      : SOME TESTS FAILED");

    $display("====================================");
end
endtask

//-----------------------------------------------------------------------------------------
```

------------------------------------------------------------------------
# Some facts about tasks

Without automatic, tasks are static—their local variables are shared between calls. With automatic, each call gets its own storage, similar to local variables in a C or Python function. It's good practice to declare utility tasks as automatic

# Extending This Template

Only a few parts need changing for a new design:

-   DUT instantiation
-   Input/output signal declarations
-   `$fscanf()` format string

Everything else (logging, comparison, statistics, waveform generation,
and summary) can remain unchanged.

//-----------------------------------------------------------------------------------------------------------------------

