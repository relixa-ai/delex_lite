## `delex_lite`: Rule-Based Blocking for Entity Matching

`delex_lite` is a tool for the blocking step of entity matching. For more details on entity matching, see [this introduction](https://anhaidgroup.github.io/magellan/about).

* `delex_lite` runs locally on a Mac with Apple silicon and installs in a single folder.
* It supports tables of up to 3,000,000 rows each.
* It lets you combine rules based on similar text, equal values, and numbers within a tolerance to select or exclude possible matches.

`delex_lite` is a simplified version of [Delex](https://github.com/anhaidgroup/delex). For the matching step, see [Iris](https://github.com/relixa-ai/iris).

### Why Combine Blocking Rules?

Different ways of finding candidates can complement one another. For example, the same product might have different names in two catalogs but similar descriptions. Combining rules for names and descriptions lets you keep pairs found by either comparison.

The same approach can help with customer records: combine a rule for similar names with another for matching phone numbers. A customer listed as "Alex Morgan" in one table and "Alexander Morgan" in the other might be found through their phone number.

Additional conditions let you narrow the results. When comparing products from two stores, you might find pairs with similar names, require similar descriptions, and exclude pairs with different model numbers. Two headphone listings, for example, may have similar names but describe different models. These choices depend on your data. A condition that is too strict can remove true matches.

### How `delex_lite` Works

You express these choices as a *blocking program*: a set of rules that compare values in your input tables. Rules that select possible matches are called *keep rules*. Optional *drop rules* exclude selected pairs that meet conditions you specify. The remaining pairs form the *candidate set*, ready for a matcher to examine.

A summary shows how many pairs remain after each step, helping you assess the effect of each rule.

You can run a program from a JSON file using the `delex-lite` command, or build and run it from Python. A table can also be compared with itself to find possible duplicates. See [How delex_lite works](doc/how-it-works.md) for the concepts and the [tutorial](doc/tutorial.md) for examples.

### Installation

`delex_lite` requires a Mac with Apple silicon (M1 or later), macOS 11 or later, and Python 3.9 or later. Follow the [installation guide](doc/installation.md) to get started.

### How to Use

Start with the [tutorial](doc/tutorial.md) for command-line and Python examples. See the [technical guide](doc/technical-guide.md) for the full reference and [troubleshooting](doc/troubleshooting.md) for help with errors.

#### For questions / comments, contact [our team](mailto:hello@relixa.ai).
