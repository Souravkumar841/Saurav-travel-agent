# Travel Reimbursement Approval Agent

A lightweight, runnable AI agent that reviews employee travel reimbursement claims against a mock company policy and returns a structured decision: `APPROVE`, `PARTIAL_APPROVE`, `REJECT` or `MANUAL_REVIEW`.

Built as a single Jupyter notebook for the GenAI / Agentic AI Developer assignment. It uses only mock data. No real employee or company data is included.

![Dashboard](UI%20SS_1.png)

---

## What it does

- **Claim intake:** accepts claims as JSON (the 5 sample claims from Appendix B are loaded in the notebook).
- **Policy grounding:** retrieves relevant `POL-*` rules before deciding, and cites them in the output.
- **Tool use:** 7 small tools (policy lookup, eligibility, per-diem limits, receipts, timeliness, approval threshold, output validator).
- **Agent workflow:** an LLM chooses which tools to call and writes the explanation; deterministic code does all arithmetic and sets the policy verdict.
- **Reliability:** schema validation, fallbacks, and routing of anything uncertain to `MANUAL_REVIEW`.
- **Dashboard and audit trail:** decision breakdown, approved vs deducted amounts, and a per-claim log of every tool call.

## Results on the sample claims

| Claim | Decision | Approved | Deducted | Why |
|---|---|---|---|---|
| CLM-001 | APPROVE | $1,110 | $0 | Eligible, receipts present, within caps, manager tier |
| CLM-002 | REJECT | $0 | $380 | Spa and minibar are ineligible (POL-CAT-02) |
| CLM-003 | PARTIAL_APPROVE | $840 | $100 | Lodging $250/night exceeds the $200 cap (POL-PD-02) |
| CLM-004 | MANUAL_REVIEW | pending | pending | Business-class fare, missing lodging receipt, total over $2,000 |
| CLM-005 | MANUAL_REVIEW | pending | pending | Missing receipt; group meal makes the per-diem ambiguous |

For manual-review claims the agent reports `approved_amount = 0` and `deducted_amount = 0`. The pending amount is stated in the explanation.

## Repository contents

| File | Description |
|---|---|
| `<yourname>.ipynb` | The full solution: README, policy, tools, agent, dashboard, design notes, final JSON |
| `UI SS_1.png` | Saved dashboard screenshot |
| `README.md` | This file |

## Setup and run

**Requirements:** Python 3.9+

```bash
git clone https://github.com/<your-username>/travel-reimbursement-agent.git
cd travel-reimbursement-agent

python3 -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate

pip install jupyterlab pandas matplotlib ipywidgets
jupyter lab
```

Open the notebook and choose **Kernel > Restart Kernel and Run All Cells**. No manual steps are needed. The last cell prints the JSON results for all five claims.

On WSL, use `jupyter lab --no-browser` and open the printed URL in your Windows browser.

### Environment variables (optional)

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Enables LLM tool-calling mode. Requires `pip install anthropic`. |
| `AGENT_MODEL` | Model name for LLM mode (default `claude-sonnet-4-6`). |

Set the key in the same terminal **before** launching Jupyter:

```bash
export ANTHROPIC_API_KEY="your-key"
```

Without a key the notebook runs in **offline mode**: a fixed planner calls the same tools in the same order and produces the same decisions, so the demo works anywhere and is reproducible.

## How the agent works

```
Claim (JSON)
   |
   v
Agent  --calls-->  policy_lookup, check_eligibility, check_limits,
(LLM or offline    check_receipts, check_timeliness, check_threshold
 planner)                      |
   |                           v
   |                   Policy engine (code) computes amounts and verdict
   v                           |
LLM verdict + explanation ---->|  validator reconciles
                               v
                  validate_output  ->  final result object
```

If the LLM and the policy engine disagree:

| LLM says | Policy engine says | Final |
|---|---|---|
| same | same | accepted |
| `MANUAL_REVIEW` | anything | accepted (the LLM may always escalate) |
| anything | `MANUAL_REVIEW` | policy engine wins |
| other mismatch | other | `MANUAL_REVIEW` |
| invalid output or API error | any | falls back to offline planner |

## Output format

One object per claim, with exactly these fields:

```json
{
  "claim_id": "CLM-003",
  "decision": "PARTIAL_APPROVE",
  "approved_amount": 840.0,
  "deducted_amount": 100.0,
  "missing_docs": [],
  "policy_refs": ["POL-APR-02", "POL-CAT-01", "POL-PD-02", "POL-RCT-01"],
  "confidence": 0.93,
  "explanation": "Claim is valid but exceeds limits. lodging: $500.00 claimed vs cap $400.00 ...",
  "tools_used": ["policy_lookup", "check_eligibility", "check_limits", "check_receipts",
                 "check_timeliness", "check_threshold", "validate_output"]
}
```

## Key design choices

- **Code owns the money.** The LLM never computes amounts or relaxes a flagged claim. It can only escalate to manual review.
- **Manual review over guessing.** Missing receipts, policy exceptions, ambiguity, late claims and totals above $2,000 all go to a human.
- **Simple retrieval.** Twelve policy rules fit in context, so a lexical lookup is used instead of a vector database.
- **No agent framework.** A short tool loop on the raw API is easier to review than LangChain or CrewAI for six tools.

## Assumptions and limitations

- The expense date for the 30-day window is the trip end date (no per-item dates are given).
- Nights and days are read from each line's description and stored as `qty`. Caps are `qty x limit` on the line total.
- Confidence is a heuristic, not a calibrated probability.
- LLM mode was not run in the saved outputs. They were produced in offline mode.

## Next steps

- Evaluate the LLM mode over repeated runs and measure agreement with the rule engine.
- Per-item dates and true per-day aggregation of meals and ground transport.
- Duplicate-claim detection, receipt OCR, currency conversion, pre-approval lookup.
- Clarify the per-person meal rule with Finance and encode it.
- Unit tests for each tool and a golden-set evaluation script.

## License

For assessment purposes only. All data is fictional.
