# Closing the Loop: A Real Code Fix for the Bug I Diagnosed

**Status: 🟢 [PR #71407](https://github.com/ceph/ceph/pull/71407) open, a code fix with tests, awaiting `ci-approved` and maintainer review**

## Context

This is Part 10 of a hands-on Ceph series:

- **[Part 1: ceph-handson](https://joshjan20.github.io/ceph-handson/)**, building a 3-node Ceph cluster from scratch
- **[Part 2: ceph-self-healing](https://joshjan20.github.io/ceph-self-healing/)**, simulating and recovering from a node failure
- **[Part 3: ceph-grafana](https://joshjan20.github.io/ceph-grafana/)**, diagnosing and fixing a real cephadm bug locally
- **[Part 4: ceph-alerting](https://joshjan20.github.io/ceph-alerting/)**, building and triggering a real automated alert
- **[Part 5: ceph-tracker-report](https://joshjan20.github.io/ceph-tracker-report/)**, reporting the Part 3 bug to Ceph's tracker
- **[Part 6: ceph-maintenance-tool](https://joshjan20.github.io/ceph-maintenance-tool/)**, building a real health-check script
- **[Part 7: ceph-doc-contribution](https://joshjan20.github.io/ceph-doc-contribution/)**, contributing documentation to Ceph
- **[Part 8: ceph-benchmarking](https://joshjan20.github.io/ceph-benchmarking/)**, performance benchmarking and a dashboard
- **[Part 9: ceph-ansible](https://joshjan20.github.io/ceph-ansible/)**, automating tooling deployment with Ansible

Part 3 diagnosed a real bug, precisely, down to the exact line of Python causing it, but stopped at a local workaround. Part 5 reported it to Ceph's official tracker. This part goes one step further: writing and submitting the actual fix, with tests, as a real pull request against `ceph/ceph`.

---

## From Diagnosis to Fix

Part 3 established, with certainty, exactly what was wrong: cephadm's certificate generation code uses a host's FQDN directly as a certificate's Common Name (CN), and X.509 limits CN to 64 characters (RFC 5280). Google Cloud's auto-assigned FQDNs, which embed the full project ID, routinely exceed that.

What Part 3 didn't yet have was the actual source line. Since cephadm is pure Python, not C++, the fix was genuinely within reach. Finding it meant locating `src/pybind/mgr/cephadm/ssl_cert_utils.py` in Ceph's own repository and reading through `SSLCerts.generate_cert()`, the method responsible for building per-host certificates.

**The exact line:**
```python
builder = builder.subject_name(x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, addrs[0]), ]))
```

`addrs[0]` is used directly as the CN, with no length check at all. This is the precise root cause, reproduced at the Python level.

---

## Proving the Bug First, Locally, Before Touching the Fix

Before writing a fix, I wrote a small standalone script to import `SSLCerts` directly (no cephadm module, no full Ceph environment needed, since this class has no such dependency) and reproduce the failure with the exact FQDN from Part 3:

```python
long_fqdn = "ceph-node1.europe-west3-a.c.project-12850071-31c6-4077-a2f.internal"
certs = SSLCerts(fsid="test-fsid-1234")
certs.generate_root_cert(addr="10.156.0.5")
cert_pem, key_pem = certs.generate_cert(_hosts=[long_fqdn], _addrs=[long_fqdn])
```

Result, against the original, unmodified code:
```
FAILED AS EXPECTED: ValueError: Attribute's length must be >= 1 and <= 64, but it was 67
```

This is the actual Python exception underlying the opaque `asn1 encoding routines: ... string too long` error seen in production, confirmed at the source, not inferred.

---

## The Fix

A small, targeted change: truncate the CN to 64 characters only when it exceeds the limit, and rely on the certificate's SAN (Subject Alternative Name) list, which already contains the full, untruncated hostname, for actual TLS hostname verification.

```python
COMMON_NAME_MAX_LENGTH = 64  # RFC 5280 hard limit on X.509 Common Name

...

cn_value = addrs[0]
if len(cn_value) > COMMON_NAME_MAX_LENGTH:
    # Certificate verification relies on the SAN list (added below), not
    # CN, so truncating here is safe: the full hostname is still present
    # in the SAN and will be used for actual TLS hostname verification.
    cn_value = cn_value[:COMMON_NAME_MAX_LENGTH]
builder = builder.subject_name(x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, cn_value), ]))
```

**Why this is safe, not just convenient:** the CN field has been effectively deprecated for TLS hostname verification since RFC 2818; modern clients check the SAN. `generate_cert()` already adds the full hostname to the SAN a few lines below this change, unmodified, regardless of CN length. Truncating the CN therefore has no effect on the certificate's actual validity or the hostnames it correctly covers.

Re-running the same reproduction script against the patched code:
```
SUCCESS: certificate generated without error
FQDNs in SAN: ['ceph-node1.europe-west3-a.c.project-12850071-31c6-4077-a2f.internal']
CONFIRMED: full FQDN present in SAN (hostname verification unaffected)
```

---

## Adding Regression Tests

A bug fix without a test can silently regress. Ceph's cephadm module already has an extensive test suite (`src/pybind/mgr/cephadm/tests/`), including `test_certmgr.py` for certificate-related code, so a new `test_ssl_cert_utils.py` was added, matching their existing `unittest.TestCase` conventions, covering:

1. A CN over 64 characters no longer raises during certificate generation
2. The full FQDN remains present in the SAN after CN truncation (proving hostname verification is unaffected)
3. A short hostname is used for the CN completely unmodified, no truncation applied where none is needed
4. When truncation does happen, the result is exactly 64 characters, not accidentally off-by-one

All 4 pass against the fix, verified in a standalone environment:
```
test_cn_is_truncated_to_exactly_max_length PASSED
test_long_fqdn_does_not_raise PASSED
test_long_fqdn_is_preserved_in_san PASSED
test_short_hostname_is_not_truncated PASSED
============================== 4 passed in 4.04s ===============================
```

And, to confirm these are genuine regression tests and not trivially-passing ones, running them against the *original*, unpatched code fails immediately, since `COMMON_NAME_MAX_LENGTH` doesn't exist there at all:
```
ImportError: cannot import name 'COMMON_NAME_MAX_LENGTH' from 'cephadm.ssl_cert_utils'
```

---

## A Real Limitation, Stated Honestly

Ceph's full test suite (`tox -e py3`) couldn't be run locally in this environment: `conftest.py` pulls in the entire cephadm module, which depends on Ceph's own custom Python bindings (`ceph.deployment.service_spec` and others) that only exist after building Ceph from source, not something achievable on a personal Mac without significant additional setup. The new test file and the fix were verified thoroughly in a standalone environment instead, isolating exactly the class under test. The PR notes this explicitly, and Ceph's own CI (once a maintainer labels it `ci-approved`) will run the test inside their actual build environment, which is the authoritative place for this to be confirmed, not a workaround, the correct division of responsibility between local verification and upstream CI.

---

## The Complete Arc

```
Part 1: Built the cluster
   │
   ▼
Part 3: Grafana fails, diagnosed the exact root cause
   │        (socket.getfqdn() → CN → 64-char X.509 limit)
   ▼
Part 3: Worked around it locally (/etc/hosts + mgr failover)
   │
   ▼
Part 5: Reported it to Ceph's official issue tracker
   │
   ▼
Part 10 (this part): Found the exact source line, wrote the
   fix, added regression tests, opened PR #71407
```

This is the difference between finding a problem and actually helping fix it, the kind of full-cycle engagement described directly in the job posting: not just "enthusiastic about Ceph," but diagnosing, reporting, and contributing a working fix back to the project.

---

*Series: [Part 1](https://joshjan20.github.io/ceph-handson/) | [Part 2](https://joshjan20.github.io/ceph-self-healing/) | [Part 3](https://joshjan20.github.io/ceph-grafana/) | [Part 4](https://joshjan20.github.io/ceph-alerting/) | [Part 5](https://joshjan20.github.io/ceph-tracker-report/) | [Part 6](https://joshjan20.github.io/ceph-maintenance-tool/) | [Part 7](https://joshjan20.github.io/ceph-doc-contribution/) | [Part 8](https://joshjan20.github.io/ceph-benchmarking/) | [Part 9](https://joshjan20.github.io/ceph-ansible/)*
