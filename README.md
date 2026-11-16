# DADA3: A bug-fixing team

**Deployed app:** https://chiragsuthar-dada3.onrender.com

## Write-up

### Setup

- Model: Haiku 5.5
- Sandbox: Daytona
- Runs: 2
- *Changes* - None. (I did add ChatNvidia capability in config, but then removed it.)

### Results

**Results from 1st Run**

```
team          resolved 10/10, triage 10/10, bugs fixed 7/7, false claims 0, 64 calls, 672,524 tokens, 149 s
baseline      resolved 10/10, triage 10/10, bugs fixed 7/7, false claims 0, 8 calls, 105,762 tokens, 54 s
```

**Results from 2nd Run**

```
team (2)      resolved 10/10, triage 10/10, bugs fixed 7/7, false claims 0, 58 calls, 314,746 tokens, 95 s
baseline (2)  resolved 10/10, triage 10/10, bugs fixed 7/7, false claims 0, 10 calls, 142,802 tokens, 57 s
```

(The numbers printed in section 2 of notebook 03, for every run.)

### Did the team do better?
For the current experiment and the baseline seemed to perform comparable to the team. For a much bigger experiment with more isolation desired team could do better. Team took longer than baseline as with usage of ~6.35 times the token amount usage, the teams run was slower.

### The handoff

team 6 verified
  evidence: Wrote tests/test_issue.py in both folders (10 tests: docstring examples shipping_cost(0.4)==500 and shipping_cost(5.2)==1100, plus boundary tests at 1, 1.0, 0.999, 1.001, 1.5, 5, 6, 6.01).
original/: `python -m unittest discover -s tests -v` -> 35 tests, FAILED (failures=2): test_exactly_1kg_is_500 



### What did it cost?
The cost for using the team was much higher than baseline. Team costs about 672,524 tokens while baseline only costed 105,762 tokens, making team cost approx 6.35 times without any performance upgrade offered.