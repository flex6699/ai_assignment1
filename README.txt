REASONING AGENT - README
========================

What it does
------------
Takes a knowledge base in English (context) and a query (statement) and predicts
True (entailed) or False (not entailed):

  English -> parser -> facts + rules / FOL clauses -> classical reasoning -> True/False

No generative model ever decides an answer. The parser is pattern-based; for
free-form English there are two translation methods (Method 1: rule-based,
Method 2: our trained translator, which only converts sentences into formulas).
Every answer is produced by one of three classical engines:
  * RuleTaker-style rule bases : indexed forward chaining with unification and
                                 stratified negation-as-failure
  * CounterBench counterfactuals: Boolean structural equations, abduction,
                                 do()-intervention, forward propagation
  * FOLIO-style first-order text: CNF + Skolemisation + resolution refutation
                                 (best-first, set-of-support, unit preference)
A router picks the engine from the shape of the input text.

HOW TO RUN: METHOD 1 (without training) OR METHOD 2 (with the trained model)
---------------------------------------------------------------------------
Free-form English problems (FOLIO-style, without formulas) can be translated
into logic in two ways. Choose with ONE setting in section 0 of the notebook:

  Method 1 - rule-based translator (DEFAULT, fastest; no trained model needed)
    1. In section 0 keep:  ENGLISH_METHOD = "rule"
    2. Run all cells.
    Needs only Python; under 1 ms per problem; fully offline. FOLIO English-only: 64.71%.

  Method 2 - our trained translator (how it was trained: notebook section 17)
    1. In section 0 set:   ENGLISH_METHOD = "trained"
       (model: TRAINED_MODEL = "flexnotop/folio-fol-t5", downloaded once; or
        TRAINED_MODEL = "folio-fol-t5" for a local folder, works offline)
    2. pip install torch transformers   (already installed on Google Colab)
    3. Run all cells.
    FOLIO English-only: 71.57%. If torch/transformers or the model are not
    available, it falls back to Method 1 automatically (within ~2 s).
    Speed: the model is loaded ONLY if the test file contains FOLIO-style
    problems written in English without formulas; for every other problem
    both methods are equally fast. Method 2 costs about 0.8 s (GPU) or
    1.3 s (CPU) per English-only problem; Method 1 takes under 1 ms.

Where the results appear: section 13 reports Method 1 on English-only FOLIO,
and section 13b always evaluates Method 2 and prints a METHOD COMPARISON table
(skipped in prediction-only mode). ENGLISH_METHOD only decides which method
makes the predictions in submission.csv.

Why Method 1 is the default: the score counts all questions together, and
speed is graded too. English-only FOLIO problems are a tiny share of the
questions, so Method 2's gain barely changes overall accuracy, while it can
make the run many times slower. Choose "trained" if accuracy on English-only
FOLIO matters more than speed. In both methods the translator only
produces formulas; the resolution prover decides every True/False answer.
RuleTaker, CounterBench and FOLIO problems that come with formulas never use
a translator, so their results are the same with either method.

Requirements
------------
  Python 3.9+
  pip install datasets jupyter
Only the Python standard library is used by the reasoning code itself;
`datasets` / `huggingface_hub` are needed only to download the public data.
No GPU is needed; everything runs on a CPU.

FOLIO is gated on Hugging Face; if it is not accessible, the notebook
automatically downloads the public release (v0.0) from the FOLIO GitHub
repository. If any dataset cannot be loaded, the notebook prints a message
and continues with the others.

How to run
----------
  1. Open reasoning_agent.ipynb and run all cells (Kernel > Restart & Run All).
  2. Section 13 prints the final accuracy and total inference time.
  3. Section 16 writes submission.csv with columns: id, label (True / False).

Running on a new / private test file
------------------------------------
Put the test file next to the notebook and run all cells. Any csv, tsv, json,
jsonl or parquet file with "test" in its name is found automatically (or set
TEST_FILE in section 16, or the TEST_FILE environment variable). Columns: an
id column, a context column, and a query column (statement, question, query,
conclusion or hypothesis). When a test file is found, the notebook runs in
PREDICTION-ONLY mode: no downloads and no experiments, only prediction (the
settings the experiments chose are the defaults). Predictions are written to
submission.csv with columns id,label (True / False). Prediction runs offline.

Method 2 details: our trained translator (section 13b; ENGLISH_METHOD = "trained")
-----------------------------------------------------------------------------
FOLIO-style problems that arrive WITHOUT formulas are translated from English
into first-order logic by a flan-t5-base model we fine-tuned ourselves
(training code in the notebook, section 17; FOLIO training stories only). The model ONLY translates;
the resolution prover decides every True/False answer. RuleTaker and
CounterBench questions never use it.
  Model:        https://huggingface.co/flexnotop/folio-fol-t5 (downloaded once)
  Needs:        torch + transformers (preinstalled on Google Colab)
  Device:       GPU if available, else CPU (about 1.3 s per problem on 16 threads)
  Offline:      the model is loaded from a local folder or an earlier download if
                present; otherwise a 5-second check finds no network and the
                notebook falls back to the rule-based translator within ~2 s and
                prints a NOTE. Everything else (RuleTaker, CounterBench, FOLIO
                with formulas, rule-based translation) never needs internet.
  Local copy:   to use the trained translator offline, place the model folder
                next to the notebook and set TRAINED_MODEL = "folio-fol-t5" in section 0.
  Turn on:      ENGLISH_METHOD = "trained" in section 0 (default is "rule").

Quick run
---------
Set EVAL_LIMIT (section 10) to e.g. 20000 to evaluate on a subset.

Configuration (CONFIG in section 1)
-----------------------------------
  RULETAKER_MODE   "CWA" (negation as failure) or "OWA" (explicit negation);
                   chosen automatically on the dev split in section 12
  NONSTRATIFIED    "wfs" (well-founded semantics) for negation cycles; section 12b
  RT_ENGINE        "fc" or "bc"; chosen end to end on the dev split in section 12c
  CB_POLICY        CounterBench conventions per question type; section 12d
  CB_UNKNOWN_ROOT  value for a causal root variable that is never mentioned
  RES_MAX_STEPS    step limit for resolution (keeps inference time bounded)
  RES_STRATEGY     "best_first" or "bfs"
  DEFAULT_LABEL    answer when nothing could be parsed (majority FOLIO label)

Notebook layout
---------------
  1  config and helpers          9  offline smoke test
  2  RuleTaker parser            10 dataset loading
  3  forward / backward chaining 11 parser coverage diagnostics
  4  CounterBench SCM engine     12 calibration on dev data
  5  FOL + resolution            13 final accuracy and timing
  6  English -> FOL translator   12b-d negation cycles, engine choice, CounterBench conventions
                                 14 experiments (FC vs BC, oracle FOL, BFS vs best-first)
  7  router, loading, evaluation 15 reasoning traces
  8  trace printers              16 submission.csv
