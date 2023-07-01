
# GPT code editing benchmarks

![benchmark results](../assets/benchmarks.svg)

Aider is an open source command line chat tool that lets you ask GPT to edit
code in your local git repos.
You can use aider to ask GPT to add features, write tests or make other changes and
improvements to your code.

Having a reliable way for GPT to read/modify/write
local source code files is a critical component of this functionality.
Making code editing more reliable often
involves changing and experimenting with 
the "edit format" that aider uses.
The edit format is a key part of the system prompt,
specifying how GPT should format code edits in its replies.
Different edit formats can range in
complexity from something simple like "return an updated copy of the whole file" to
a much more sophisticaled format 
that uses the
[function calling API](https://openai.com/blog/function-calling-and-other-api-updates)
to specify a series of specific diffs

To measure the impact of changes to the edit format,
I created a benchmark based on the
[Exercism python](https://github.com/exercism/python)
coding exercises.
This benchmark measures how well aider & GPT can turn
a natural language coding request into
actual runnable code saved into files that pass unit tests.
This is an end-to-end assessment
of not just how well GPT can write code, but also how well it
can *edit existing code* and
*package up those code changes*
so that aider can save the edits to the
local source files.

I ran this code editing benchmark
on almost all the ChatGPT models, using a variety of edit formats.
This produced some interesting results:

  - Asking GPT to just return an updated copy of the whole file in a normal fenced code block is by far the most reliable edit format. This is true across all GPT-3.5 and GPT-4 models.
  - Using the new function calling API is worse than the above whole file method, for all models. GPT writes worse code and frequently mangles this output format, even though OpenAI introduced the function calling API to make structured output formatting more reliable. This was a big surprise.
  - The new June (`0613`) versions of `gpt-3.5-turbo` are worse at code editing than the older February (`0301`) version. This was unexpected.
  - The GPT-4 models are much better at code editing than the GPT-3.5 models, as expected.

The quantitative benchmark results agree with an intuition that I've been
developing about how to prompt GPT for complex tasks like coding.
You want to minimize the "cognitive overhead" of formatting the response, so that
GPT can focus on the task at hand.
As an analogy, you wouldn't expect a good result if you asked a junior developer to
implement a new feature by hand typing the required code
changes as `diff -c` formatted updates.

Using more complex output formats seem to cause two problems:

  - It makes GPT write worse code. Keeping the output format simple seems to leave GPT with more attention to devote to the actual coding task.
  - It makes GPT less likely to adhere to the output format.

I had hoped that the new function calling API would enable more reliable use of
structured output formats, but it does not appear to be a panacea
when working with source code.

More details on the benchmark, edit formats and results are discussed below.


## The benchmark

The benchmark uses 
[133 practice exercises from the Exercism python repository](https://github.com/exercism/python/tree/main/exercises/practice).
These exercises were designed for people to learn python and practice
their coding skills.

Each exercise has:

  - Some instructions for the exercise, in markdown files.
  - Stub code for the implementation in a python file, specifying the functions/classes that need to be implemented.
  - Unit tests in a seperate python file.

The goal is for GPT to read the instructions, implement the provided functions/class skeletons
and pass all the unit tests. The benchmark measures what percentage of
the 133 exercises are completed successfully, causing all the associated unit tests to pass.

To complete an exercise, aider sends GPT
the initial contents of the implementation file,
the Exercism instructions
and a final instruction:

```
Use the above instructions to modify the supplied files: <implementation file>
Keep and implement the existing function or class stubs, they will be called from unit tests.
Only use standard python libraries, don't suggest installing any packages.
```

Aider updates the implementation file based on GPT's reply and runs the unit tests.
If they all pass, we are done. If some tests fail, aider sends
GPT a second message with the test error output.
It only sends the first 50 lines of test errors, to avoid exhausting the context
window of the smaller models.
Aider also includes this final instruction:

```
See the testing errors above.
The tests are correct.
Fix the code in <implementation file> to resolve the errors.
```

Editing the implementation in response to test failures is
another way that this benchmark measures how well GPT can perform code editing.
This second chance is also important because
many of the unit tests check for specifics that are not
called out in the instructions.
For example, many tests want to see
[specific phrases in ValueErrors](https://github.com/exercism/python/blob/f6caa44faa8fb7d0de9a54ddb5c6183e027429c6/exercises/practice/queen-attack/queen_attack_test.py#L31)
raised by
the implementation.
There's no way for a human or an AI
to pass these unit tests
without seeing their error output.

It's worth noting that GPT never gets to see the source code of the unit tests.
Just the error output from failed tests.

## Edit formats

I benchmarked 4 different edit formats, described below.
Each description includes a sample response that GPT might provide in response to a user who
requests:
"Change the print from hello to goodbye."

### whole

The
[whole](https://github.com/paul-gauthier/aider/blob/main/aider/coders/wholefile_prompts.py)
format asks GPT to return an updated copy of the entire file, including any changes.
The file should be
formatted with normal markdown triple-backtick fences, inlined with the rest of its response text.

This format is very similar to how ChatGPT returns code snippets during normal chats, except with the addition of a filename right before the opening triple-backticks.

````
Here is the updated copy of your file demo.py:

demo.py
```python
def main():
    print("goodbye")
```
````

### diff

The [diff](https://github.com/paul-gauthier/aider/blob/main/aider/coders/editblock_prompts.py)
format also asks GPT to return edits as part of the normal response text,
in a simple diff format.
Each edit is a fenced code block that
specifies the filename and a chunk of ORIGINAL and UPDATED code.
GPT provides some original lines from the file and then a new updated set of lines.

````
Here are the changes you requested to demo.py:

```python
demo.py
<<<<<<< ORIGINAL
    print("hello")
=======
    print("goodbye")
>>>>>>> UPDATED
```
````

### whole-func

The [whole-func](https://github.com/paul-gauthier/aider/blob/main/aider/coders/wholefile_func_coder.py) 
format requests updated copies of whole files to be returned using the function call API.


```
{
    "explanation": "Changed hello to goodbye.",
    "files": [
        {
            "path": "demo.py",
            "content": "def main():\n    print(\"goodbye\")\n"
        }
}
```

### diff-func

The
[diff-func](https://github.com/paul-gauthier/aider/blob/main/aider/coders/editblock_func_coder.py)
format requests a list of 
original/updated style edits to be returned using the function call API.

```
{
    "explanation": "Changed hello to goodbye.",
    "edits": [
        {
            "path": "demo.py",
            "original_lines": [
                "    print(\"hello\")"
            ],
            "updated_lines": [
                "    print(\"goodbye\")"
            ],
        }
    ]
}       
```

## GPT-3.5 hallucinates function calls

GPT-3.5 is prone to ignoring the JSON Schema that specifies valid functions,
and often returns a completely novel and semantically
invalid `function_call` fragment with `"name": "python"`.

```
        "function_call": {
          "name": "python",
          "arguments": "def main():\n    print(\"hello\")\n"
        },
```

The `arguments` attribute is supposed to be a set of key/value pairs
with the arguments to the function specified in the `name` field.
Instead, GPT-3.5 frequently just stuffs an entire python
file into that field.

It feels like it might be getting confused by fine tuning that was done for ChatGPT plugins?

## Randomness

The benchmark goes to some trouble to be deterministic, always sending identical
requests for each exercise on repeated runs.
As part of this effort,
when sending test error output to GPT
it removes the wall-clock timing information that
is normally included by the `unittest` module.

The benchmarking harness also logs sha hashes of the
API requests and replies.
This makes it easy to identify sources of randomness or nondeterminism
in the bechmarking process.

It turns out that the OpenAI chat APIs are not deterministic, even at `temperature=0`.
The same identical request will produce multiple distinct responses,
usually on the order of 3-6 different variations. This feels
like they are load balancing across a number of slightly different
instances of the model.

For some exercises, some of these variable responses pass the unit tests while
other variants do not. Results for exercises like this which are
"on the bubble" 
are therefore a bit random, depending on which variant OpenAI returns.

Given that, it would be ideal to run all 133 exercises many times for each
model/edit-format combination and report an average performance.
This would average away the effect of the API variance.
It would also significantly increase the cost of this sort of benchmarking.
So I didn't do that.

Benchmarking against 133 exercises provides some robustness all by itself, since
we are measuring the performance across many exercises.

But to get a sense of how much the API variance impacts the benchmark outcomes,
I ran all 133 exercises 10 times each
against `gpt-3.5-turbo-0613` with the `whole` edit format.
You'll see one set of error bars in the graph, which show
the range of results from those 10 runs.

The OpenAI API randomness doesn't seem to
cause a large variance in the benchmark results.

## Conclusions

Based on these benchmarking results, aider will continue to use
the `whole` edit format for GPT-3.5, and `diff` for GPT-4.
While GPT-4 gets somewhat better results with the `whole` edit format,
it significantly increases costs and latency compared to `diff`.

The latency of streaming back the entire updated copy of each edited file
is the real challenge. The GPT-3.5 models are quite responsive, and can
stream back entire files at an acceptable speed.
Aider displays a progress bar and
live diffs of the files as they stream in,
which helps pass the time.

The GPT-4 models are much slower, and waiting for even small files
to be completely "retyped" on each request is probably unacceptable.

