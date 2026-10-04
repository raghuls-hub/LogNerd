## Project Status: **10%**

We are currently at **Dataset Collection & Planning**.  
Since data quality is the foundation, I recommend we **finish the real dataset collection and quality analysis before generating synthetic data**.

---

# 1. Do we need a GPU?

### For our current LogNerd work: **No.**

Dataset:

- downloading
- cleaning
- regex processing
- Drain template parsing
- TF-IDF
- feature engineering
- correlation
- Mutual Information
- L1 selection

can all be done on your **VS Code + Jupyter notebooks + CPU**.

### Later models

| Task | GPU needed? |
|---|---|
| Data preprocessing | ❌ No |
| Drain/Spell parsing | ❌ No |
| TF-IDF | ❌ No |
| XGBoost | ❌ No |
| Random Forest | ❌ No |
| Isolation Forest | ❌ No |
| Decision Tree | ❌ No |
| L1/Lasso | ❌ No |
| Classical ML experiments | ❌ No |
| Deep Learning / Transformer | ✅ Usually |
| LLM-based log analysis | ✅ Recommended |

**Decision: Stay in VS Code + notebooks for now.**

Don't move to Colab/Kaggle yet.

---

# 2. Datasets we should use

### Main dataset

## **GHALogs**

[GHALogs GitHub repository](https://github.com/D2KLab/gha-dataset?utm_source=chatgpt.com)

- 116K+ workflows
- 513K workflow runs
- 2.3M steps
- 25K+ repositories
- 20 programming languages
- Complete execution logs

This is currently the **best foundation** for our project. [GitHub](https://github.com/D2KLab/gha-dataset?utm_source=chatgpt.com)

### Why?

It represents **modern GitHub Actions CI/CD**, which is much closer to the system we want to build than older Travis-only datasets.

---

# 3. Supporting datasets

| Dataset | Why use it? | Main problem |
|---|---|---|
| **GHALogs** | Large + modern + raw logs | ~142 GB raw logs |
| **TravisTorrent** | Huge + Python/JS/Java | 2013–2017, relatively old |
| **LogChunks** | Manually labeled failure portions | Only 797 logs |
| **CI Compiler Errors** | Massive Java compiler coverage | Java-focused |
| **CIBench** | Large raw Travis logs + metadata | Old Travis ecosystem |
| **LogDx-CI** | Modern labeled failure examples | Only 35 cases |
| **Local Java Builds** | Large Java build logs | Not actually CI/CD |

### TravisTorrent

TravisTorrent contains **2.64M builds** and specifically includes Java, Python and JavaScript projects. [PLOS](https://journals.plos.org/plosone/article?id=10.1371%2Fjournal.pone.0340167\&utm_source=chatgpt.com)

**Use:** supplementary historical data.

**Don't make it our main dataset** because it represents Travis CI from 2013–2017.

---

### LogChunks

797 Travis CI logs from 80 repositories, with manually labeled sections explaining build failures. [Zenodo](https://zenodo.org/records/3632351?utm_source=chatgpt.com)

**Use:** high-quality **failure-label validation**.

**Don't use as main training data** because it is too small.

---

### CI Compiler Errors

Very large Java CI compilation dataset.

**Use:** improve coverage of Java compilation errors.

**Don't use as the main dataset** because it is heavily specialized toward Java compiler failures.

---

### CIBench

Contains TravisTorrent data plus **38.3 GB of raw logs**. [Zenodo](https://zenodo.org/records/4682056?utm_source=chatgpt.com)

**Use:** supplementary historical CI logs.

**Don't prioritize it over GHALogs** because of the older Travis ecosystem and overlap with TravisTorrent.

---

### LogDx-CI

35 real GitHub Actions failures covering:

- Python/pytest
- Java/Maven/Gradle
- Node/pnpm/Jest
- TypeScript
- Docker
- mypy
- etc. [LogDx-CI](https://logdx-bench.github.io/index.html?utm_source=chatgpt.com)

**Use:** validation/test dataset.

**Don't train on it** — 35 cases is far too small.

---

### Local Java Builds

Contains Java projects built using Maven, Gradle and Ant, including console output and exit status. [DOI](https://doi.org/10.3390/data5030086?utm_source=chatgpt.com)

**Use:** optional Java supplementary dataset.

**Don't mix directly with CI logs**, because local builds have different environmental failure characteristics.

---

# 4. Final dataset architecture

I recommend this:

```text
                    LogNerd Dataset
                          │
              ┌───────────┴───────────┐
              │                       │
         MAIN DATA                SUPPORTING
              │                       │
          GHALogs              ┌──────┼──────┐
              │                 │      │      │
       ┌──────┼──────┐      Travis  LogChunks  Java
       │      │      │      Torrent           Errors
     Python   JS    Java
              │
              ↓
       RAW CI/CD LOGS
              │
              ↓
        PREPROCESSING
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
    Clean   Parse   Mask
              │
              ↓
        Drain Templates
              │
              ↓
        Time Windows
              │
              ↓
       Feature Matrix X
              │
              ↓
       Labels + Metadata
              │
              ↓
       Quality Filtering
              │
              ↓
       Final LogNerd Dataset
```

---

# 5. Why this architecture?

### GHALogs = **core**

Because it gives us:

**modern + large + complete logs + multiple languages**

So Python, JavaScript and Java can share the same preprocessing pipeline.

### Other datasets = **specialized supplements**

They solve specific weaknesses:

- **TravisTorrent →** historical diversity
- **LogChunks →** human-labeled failure evidence
- **CI Compiler Errors →** Java compiler coverage
- **LogDx-CI →** modern labeled validation

We **do not blindly merge everything**.

That would introduce:

- different CI environments
- different logging formats
- different time periods
- duplicate projects/builds
- different label definitions
- potential data leakage

Instead:

```text
GHALogs
   ↓
PRIMARY DATASET

Other datasets
   ↓
Validation / enrichment / rare-class supplementation
```

---

# 6. What we reject

We should **not use these as primary data**:

### ❌ Pure synthetic logs

Not initially.

Synthetic data can make our model learn the generator's patterns rather than real CI behavior.

### ❌ Local Java builds as primary

Not CI/CD.

### ❌ LogDx-CI as training

Only 35 cases.

### ❌ LogChunks as main training

Only 797 logs, although its labels are valuable.

### ❌ TravisTorrent as primary

Very useful, but old.

### ❌ Combining every dataset immediately

This is the biggest danger.

**More data ≠ better data.**

---

# 7. Our final collection strategy

### Phase 1 — Main

**GHALogs**

↓

Extract only:

```text
Python
JavaScript / TypeScript / Node
Java
```

### Phase 2 — Supplement

Add useful failure examples from:

```text
TravisTorrent
LogChunks
CI Compiler Errors
```

### Phase 3 — Validation

Keep completely separate:

```text
LogDx-CI
```

### Phase 4 — Synthetic

Only after we calculate:

```text
Class distribution
        ↓
Rare categories
        ↓
Missing categories
        ↓
Generate synthetic examples
```

This gives us **real → clean → balanced → synthetic supplementation**, rather than synthetic → model.

---

## Current decision

**Workspace:** VS Code + Jupyter ✅  
**GPU:** Not required now ✅  
**Primary dataset:** GHALogs ✅  
**Supplementary:** TravisTorrent + LogChunks + CI Compiler Errors  
**Validation:** LogDx-CI  
**Synthetic generation:** Later  
**Primary languages:** Python + JavaScript/TypeScript + Java

**Project completion: 10% → next milestone is 20% after we successfully acquire and inspect the selected datasets.**

## Dataset final target categories
| # | Main category | Examples |
|---|---|---|
| 1 | **Compilation / Syntax** | SyntaxError, javac error, TypeScript compile error |
| 2 | **Dependency** | Missing package, package not found, install failure |
| 3 | **Dependency Conflict** | Version conflict, peer dependency conflict |
| 4 | **Configuration** | Invalid config, missing environment variable |
| 5 | **Test Failure** | Assertion failure, test crash, test timeout |
| 6 | **Code Quality** | ESLint, Pylint, Checkstyle, formatting |
| 7 | **Type Error** | TypeScript, mypy, Java type mismatch |
| 8 | **Runtime Error** | Exception, application crash, startup failure |
| 9 | **Network** | DNS, connection timeout, registry failure |
| 10 | **Authentication / Permission** | Invalid token, access denied |
| 11 | **Resource** | OOM, disk full, CPU limit, timeout |
| 12 | **Environment / Toolchain** | Wrong Python/Node/JDK, missing tool |
| 13 | **CI Workflow** | YAML, wrong command, failed workflow step |
| 14 | **Git / Version Control** | Clone, checkout, merge, submodule |
| 15 | **Build / Package / Deployment** | Docker, JAR, artifact, deployment |