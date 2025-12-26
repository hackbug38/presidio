# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Presidio is a Python-based data protection and de-identification SDK developed by Microsoft. It detects and anonymizes PII (Personally Identifiable Information) in text, images, and structured data using NLP, regex patterns, and LLMs.

**Key Technologies:**
- Python 3.10-3.13
- Package Manager: Poetry
- Linter/Formatter: Ruff (with pre-commit hooks)
- Testing: pytest with coverage requirements (90% for new code)
- NLP Engines: spaCy (default), Stanza, Transformers, GLiNER
- REST APIs: Flask

## Repository Architecture

The repository is a monorepo with 5 independent Python packages:

### 1. presidio-analyzer
**Core PII detection engine.** Identifies entities using:
- 40+ predefined recognizers (country-specific IDs, credit cards, emails, etc.)
- Regex-based pattern matching (`pattern_recognizer.py`)
- NLP-based recognition via spaCy/Stanza/Transformers (`nlp_engine/`)
- LLM-based detection via LangExtract (`lm_recognizer.py`, `llm_utils/`)
- Context-aware enhancement (`context_aware_enhancers/`)

**Key modules:**
- `analyzer_engine.py` - Main orchestration
- `predefined_recognizers/` - 40+ entity recognizers organized by country and type
- `nlp_engine/` - Pluggable NLP backends
- `conf/default_recognizers.yaml` - Recognizer configuration (country-specific ones are `enabled: false` by default)

### 2. presidio-anonymizer
**PII anonymization/deanonymization engine.** Supports operators:
- `redact`, `replace`, `mask`, `hash` (SHA256/SHA512)
- `encrypt`/`decrypt` (AES), `keep` (no-op)
- Custom lambda functions via `custom` operator
- Azure Health Data Services surrogates (`ahds_surrogate`)

**Key modules:**
- `anonymizer_engine.py`, `deanonymize_engine.py`, `batch_anonymizer_engine.py`
- `operators/` - All anonymization strategies
- Handles overlapping entity conflicts

### 3. presidio-image-redactor
**Image and DICOM PII redaction.** Uses OCR (Tesseract or Azure Form Recognizer) to extract text, then analyzer to detect PII, then redacts bounding boxes.

**Key features:**
- Standard image formats (PNG, JPG, etc.) and DICOM medical images
- OCR via `tesseract_ocr.py` or `document_intelligence_ocr.py`
- Verification engines to validate redaction quality

### 4. presidio-structured
**Structured/semi-structured data handler.** De-identifies Pandas DataFrames and JSON by analyzing column schemas and applying analyzer + anonymizer.

### 5. presidio-cli
**Command-line interface** for file/directory PII scanning with YAML configuration support (`.presidiocli`).

## Common Development Commands

### Setup (First Time)
```bash
# Install Poetry and tooling
pip install poetry ruff

# Install pre-commit hooks (auto-formats on commit)
pip install pre-commit
pre-commit install

# Component setup (example: analyzer)
cd presidio-analyzer
poetry install --all-extras  # Takes ~5 min, NEVER CANCEL
poetry run python -m spacy download en_core_web_lg  # Required for NLP
```

### Running Tests
```bash
# Single component
cd presidio-analyzer  # or any component
poetry run pytest -vv  # All tests

# With coverage
poetry run pytest --cov=presidio_analyzer --cov-report=html tests/

# Run specific test file
poetry run pytest tests/test_analyzer_engine.py -v

# Run single test
poetry run pytest tests/test_analyzer_engine.py::test_simple -v
```

### Linting and Formatting
```bash
# From repository root
ruff check .  # Check for issues
ruff check . --fix  # Auto-fix issues

# Pre-commit hooks run ruff automatically on commit
git commit  # Will auto-format and re-commit if changes needed
```

### Docker Testing
```bash
# Pull and run pre-built images (recommended)
docker pull mcr.microsoft.com/presidio-analyzer:latest
docker pull mcr.microsoft.com/presidio-anonymizer:latest
docker run -d -p 5002:3000 mcr.microsoft.com/presidio-analyzer:latest
docker run -d -p 5001:3000 mcr.microsoft.com/presidio-anonymizer:latest
sleep 20  # Wait for startup
curl http://localhost:5002/health
curl http://localhost:5001/health

# Test analyze endpoint
curl -X POST http://localhost:5002/analyze \
  -H "Content-Type: application/json" \
  -d '{"text": "My email is test@example.com", "language": "en"}'

# Build from source (may fail in restricted networks)
docker compose up --build -d  # Takes 10-15 min
```

### E2E Integration Tests
```bash
# Ensure services are running first
docker compose up -d

# Run E2E tests
cd e2e-tests
python -m venv presidio-e2e
source presidio-e2e/bin/activate
pip install -r requirements.txt
pytest -v  # Takes ~2 min, 90%+ tests should pass
```

## Code Architecture Patterns

### Plugin Architecture
- **Recognizers**: Inherit from `EntityRecognizer` (base) or `PatternRecognizer` (regex-based)
- **Operators**: Inherit from `Operator` base class
- **NLP Engines**: Implement `NlpEngine` interface

### Configuration-Driven Design
- Recognizers configured in `conf/default_recognizers.yaml`
- Country-specific recognizers disabled by default (`enabled: false`)
- NLP engine selection via `NlpEngineProvider`

### Factory Patterns
- `AnalyzerEngineProvider` - Creates configured analyzer instances
- `OperatorsFactory` - Creates anonymization operators by name
- `NlpEngineProvider` - Instantiates NLP backends

### PII Detection Flow (Analyzer Architecture)

Understanding the multi-layered detection approach:

1. **Text Input** → `AnalyzerEngine.analyze()`
2. **NLP Processing** → NLP engine extracts tokens, lemmas, POS tags, NER entities
3. **Recognizer Execution** (parallel):
   - **Pattern-based**: Regex patterns with context enhancement
   - **NLP-based**: Uses NER results from spaCy/Stanza/Transformers
   - **Rule-based**: Custom logic (checksums, validators)
   - **LLM-based**: LangExtract for prompt-based detection
4. **Context Enhancement** → `ContextAwareEnhancers` boost confidence scores
5. **Result Aggregation** → Merge overlapping entities, resolve conflicts
6. **Output** → List of `RecognizerResult` with entity type, location, confidence

**Key insight**: Multiple recognizers can detect the same text span. The engine handles deduplication and picks the highest-confidence result.

### Anonymization Flow (Anonymizer Architecture)

1. **Anonymization Request** → Text + analyzer results + operators config
2. **Conflict Resolution** → Handle overlapping entities (merge or remove)
3. **Operator Application** → Apply operators (redact, replace, mask, hash, encrypt, etc.)
4. **Text Reconstruction** → Build anonymized text maintaining positions
5. **Deanonymization Mapping** (optional) → Store reversible transformations
6. **Output** → Anonymized text + operator metadata

**Key insight**: Operators are applied in reverse order (end to start) to maintain text positions.

### Testing Conventions
- Test naming: `test_when_[condition]_then_[expected_behavior]`
- Unit tests in `tests/` directory per package
- Integration tests for external dependencies (Azure, Transformers)
- E2E tests in `/e2e-tests/` for REST API flows
- AHDS tests skip when `AHDS_ENDPOINT` not set (expected)
- Transformers tests may fail in restricted networks (expected)

## Adding New Recognizers

1. **Choose correct location** in `presidio-analyzer/presidio_analyzer/predefined_recognizers/`:
   - `country_specific/<country>/` - Region-specific (e.g., US SSN, UK NHS)
   - `generic/` - Globally applicable (e.g., EMAIL_ADDRESS)
   - `nlp_engine_recognizers/` - NLP-based
   - `third_party/` - External service integrations

2. **Implementation requirements**:
   - Inherit from `PatternRecognizer` for regex or `EntityRecognizer` for custom logic
   - Make regex patterns specific to minimize false positives
   - Document pattern sources with comments (link to standards/references)
   - Add comprehensive tests including edge cases

3. **Configuration**:
   - Add to `conf/default_recognizers.yaml`
   - Set `enabled: false` for country-specific recognizers
   - Update imports in `predefined_recognizers/__init__.py`

4. **Documentation**:
   - Update `docs/supported_entities.md` for new entity types
   - Add usage examples in docstrings

## Functional Validation

**CRITICAL**: Always test actual functionality after making changes. Build success and passing unit tests alone are insufficient.

### Quick Functionality Tests

After making changes, validate with these quick scenarios:

```bash
# 1. CLI Functionality
cd presidio-cli
echo "My name is John Doe and my phone number is 555-123-4567" > /tmp/test.txt
poetry run presidio /tmp/test.txt
# Expected: Detects PERSON and PHONE_NUMBER entities

# 2. REST API Analysis (requires containers running)
curl -X POST http://localhost:5002/analyze \
  -H "Content-Type: application/json" \
  -d '{"text": "My name is John Doe and my email is john@example.com", "language": "en"}'
# Expected: Returns JSON with detected PERSON and EMAIL_ADDRESS entities

# 3. REST API Anonymization
curl -X POST http://localhost:5001/anonymize \
  -H "Content-Type: application/json" \
  -d '{"text": "My name is John Doe", "analyzer_results": [{"entity_type": "PERSON", "start": 11, "end": 19, "score": 0.85}]}'
# Expected: Returns anonymized text with "<PERSON>" replacement

# 4. End-to-End Pipeline
# Use analyzer API to detect → Use anonymizer API with results → Verify quality
```

## CI/CD and Coverage Requirements

### Coverage Thresholds
- **90% minimum** for new code (enforced via diff-cover)
- **85%** for green status, **70%** for warning
- Coverage reports uploaded to `coverage-data-*` branches

### CI Pipeline
- **GitHub Actions** - CI workflow for linting, testing, Docker builds, E2E tests
- **Azure Pipelines** - Additional build and release automation
- Uses multi-Python version matrix (3.10, 3.11, 3.12, 3.13)
- Includes security analysis (CodeQL, Defender for DevOps)

**CI Steps:**
1. **Lint** - Ruff check on all code
2. **Dependency Review** - Check for vulnerabilities and license compliance
3. **Tests** - Multi-Python × Multi-component matrix (unit + integration)
4. **Build** - Docker images with multi-platform support (amd64, arm64)
5. **E2E Tests** - Full service integration testing

### Expected Build Times and Timeouts

**CRITICAL**: Always set appropriate timeouts and NEVER CANCEL long-running operations.

| Operation | Expected Time | Minimum Timeout | Notes |
|-----------|---------------|-----------------|-------|
| `poetry install --all-extras` (analyzer) | 5 minutes | 10 minutes | Downloads many dependencies |
| spaCy model downloads (`en_core_web_lg`) | 1 minute | 3 minutes | ~500MB download |
| Analyzer tests (`pytest -vv`) | 2 minutes | 5 minutes | 900+ tests |
| Anonymizer tests | 5 seconds | 1 minute | 266 tests, very fast |
| CLI tests | 40 seconds | 2 minutes | 23 tests |
| Docker image pulls | 3 minutes | 10 minutes | Pre-built images |
| Docker builds from source | 15 minutes | 30 minutes | May fail in restricted networks |
| Docker service startup | 20 seconds | 60 seconds | Wait before testing |
| E2E test suite | 2 minutes | 5 minutes | Requires running containers |

## Pull Request Requirements

1. **Open GitHub issue first** before creating PR
2. **Two maintainer approvals** required
3. **All tests must pass** - Unit, integration, E2E, and linting
4. **90% diff-cover** for new code
5. **Update CHANGELOG.md** under "Unreleased" section
6. **Small PRs** - Solve one issue at a time
7. **CLA required** - Microsoft Contributor License Agreement

## Common Issues

### Network/SSL Issues
- Docker builds may fail in sandboxed environments
- Use pre-built images from `mcr.microsoft.com/presidio-*`
- Transformers/HuggingFace tests skip when network restricted (expected)

### Test Failures
- **Expected skips**: AHDS tests without `AHDS_ENDPOINT` env var
- **Expected failures**: Transformers tests in restricted networks
- **Success criteria**: 900+ analyzer tests, 266 anonymizer tests, 23 CLI tests passing

### Build Failures
- Ensure Python 3.10-3.13 available
- Use `poetry run` prefix for all commands in Poetry environments
- spaCy models required: `python -m spacy download en_core_web_lg`

## REST API Endpoints

**Analyzer** (port 5002 in compose):
- `POST /analyze` - Detect PII entities
- `GET /health` - Health check

**Anonymizer** (port 5001 in compose):
- `POST /anonymize` - Anonymize detected entities
- `POST /deanonymize` - Reverse anonymization
- `GET /anonymizers` - List operators
- `GET /health` - Health check

**Image Redactor** (port 5003 in compose):
- `POST /redact` - Redact PII in images
- `GET /health` - Health check

## Important Files

- `/pyproject.toml` - Root Ruff configuration (line length 88, NumPy docstring style)
- `<component>/pyproject.toml` - Package metadata and dependencies
- `.pre-commit-config.yaml` - Auto-formatting on commit (ruff check + format)
- `conf/default_recognizers.yaml` - Recognizer configuration per component
- `CONTRIBUTING.md` - Full contribution guidelines
- `docs/development.md` - Detailed development setup

## External Resources

- **Documentation**: https://microsoft.github.io/presidio
- **Demo**: https://aka.ms/presidio-demo
- **presidio-research repo**: Advanced samples and research datasets
- **docs/samples/**: Reference examples (not production code, use for inspiration)
