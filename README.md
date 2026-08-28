# Closing the Loop: Fixing the cephadm Certificate Bug (PR #71407)

Part 10 of a hands-on Ceph learning series. Closes the loop opened in Part 3: not just diagnosing and reporting a bug, but writing and submitting the actual fix, with regression tests, as a real PR against `ceph/ceph`.

📖 **Read the write-up:** [index.md](index.md)

**Status: 🟢 [PR #71407](https://github.com/ceph/ceph/pull/71407) open**, a code fix with tests, awaiting `ci-approved` and maintainer review.

## What's inside

- Finding the exact source line responsible for the bug diagnosed in Part 3
- Reproducing the bug at the Python level, standalone, before writing any fix
- The actual fix: truncating the X.509 Common Name to 64 characters, safely, since TLS hostname verification relies on the SAN, not the CN
- 4 regression tests, verified passing against the fix and meaningfully failing against the original code
- An honest note on a real local testing limitation, and why that's fine

## Series

- [Part 1: ceph-handson](https://joshjan20.github.io/ceph-handson/), full cluster setup and initial troubleshooting
- [Part 2: ceph-self-healing](https://joshjan20.github.io/ceph-self-healing/), simulating and recovering from a node failure
- [Part 3: ceph-grafana](https://joshjan20.github.io/ceph-grafana/), diagnosing the bug this part fixes
- [Part 4: ceph-alerting](https://joshjan20.github.io/ceph-alerting/), building and triggering a real automated alert
- [Part 5: ceph-tracker-report](https://joshjan20.github.io/ceph-tracker-report/), reporting the bug upstream
- [Part 6: ceph-maintenance-tool](https://joshjan20.github.io/ceph-maintenance-tool/), a maintenance/debugging tool
- [Part 7: ceph-doc-contribution](https://joshjan20.github.io/ceph-doc-contribution/), contributing documentation to Ceph
- [Part 8: ceph-benchmarking](https://joshjan20.github.io/ceph-benchmarking/), performance benchmarking
- [Part 9: ceph-ansible](https://joshjan20.github.io/ceph-ansible/), automating tooling deployment with Ansible
- Part 10 (this repo): the actual code fix
