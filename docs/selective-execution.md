# Selective Notebook Execution Guide

## :material-target: Overview

The selective execution feature enables directory-specific CI/CD for notebook repositories with organized subdirectory structures. When a `requirements.txt` file changes in a specific notebook directory, only notebooks in that directory are processed, dramatically reducing execution time and compute costs.

## :material-folder: Repository Structure Requirements

To use selective execution, organize your repository like this:

```
your-repo/
├── notebooks/
│   ├── data_analysis/
│   │   ├── requirements.txt      # Directory-specific dependencies
│   │   ├── analysis1.ipynb
│   │   ├── analysis2.ipynb
│   │   └── data_prep.py
│   ├── visualization/
│   │   ├── requirements.txt      # Different dependencies for viz
│   │   ├── plot_generator.ipynb
│   │   └── dashboard.ipynb
│   ├── modeling/
│   │   ├── requirements.txt      # ML-specific dependencies
│   │   ├── train_model.ipynb
│   │   └── evaluate.ipynb
│   └── shared_utils/
│       ├── utils.py              # Shared utilities
│       └── config.py
├── requirements.txt              # Root-level dependencies (affects all)
├── _config.yml                   # Jupyter Book configuration
├── _toc.yml                      # Table of contents
└── .github/
    └── workflows/
        ├── notebook-ci-pr-selective.yml
        └── notebook-ci-main-selective.yml
```

## :material-magnify: How Selective Detection Works

### File Change Analysis

The selective workflows analyze changed files and categorize them:

1. **Directory-specific requirements**: `notebooks/*/requirements.txt`
   - Only affects notebooks in that specific directory
   - Enables parallel processing of multiple directories

2. **Root requirements**: `requirements.txt`, `pyproject.toml`, `setup.py`
   - Affects all notebooks (safety first)
   - Triggers full repository CI

3. **Notebook files**: `notebooks/**/*.ipynb`
   - Only affects the directory containing the changed notebook

4. **Documentation files**: `*.md`, `_config.yml`, `_toc.yml`
   - Triggers documentation-only rebuild

### Execution Strategies

| Change Type | Strategy | Affected Scope | Performance |
|-------------|----------|----------------|-------------|
| `notebooks/data_analysis/requirements.txt` | `selective-directories` | Only `data_analysis/` | 5-10 minutes |
| `notebooks/viz/plot1.ipynb` | `selective-directories` | Only `viz/` | 3-8 minutes |
| `requirements.txt` (root) | `full-repository` | All notebooks | 15-25 minutes |
| `_config.yml` | `docs-only` | Documentation | 2-5 minutes |

## :material-rocket-launch: Benefits

### Performance Improvements

- **Targeted Execution**: Only process notebooks in affected directories
- **Parallel Processing**: Multiple directories can run simultaneously
- **Faster Feedback**: Developers get results for their specific changes quickly
- **Cost Optimization**: Significant reduction in GitHub Actions minutes

### Example Scenarios

#### Scenario 1: Update Data Analysis Dependencies
```bash
# Change made:
echo "pandas==2.1.0" >> notebooks/data_analysis/requirements.txt

# Result:
# - Only notebooks in data_analysis/ directory are executed
# - visualization/ and modeling/ directories are skipped
# - Execution time: ~5 minutes instead of 20 minutes
# - Cost savings: ~75%
```

#### Scenario 2: Fix Visualization Notebook
```bash
# Change made:
# Edit notebooks/visualization/dashboard.ipynb

# Result:
# - Only notebooks in visualization/ directory are validated/executed
# - Other directories remain untouched
# - Parallel execution if multiple directories affected
```

#### Scenario 3: Update Root Dependencies
```bash
# Change made:
echo "numpy==1.24.0" >> requirements.txt

# Result:
# - All notebooks are executed (safety first)
# - Full repository validation
# - Same behavior as traditional workflow
```

## :material-clipboard-text: Implementation Guide

### Step 1: Choose Selective Workflows

Replace your existing workflows with selective versions:

```bash
# Remove old workflows
rm .github/workflows/notebook-ci-pr.yml
rm .github/workflows/notebook-ci-main.yml

# Add selective workflows
cp examples/workflows/notebook-ci-pr-selective.yml .github/workflows/
cp examples/workflows/notebook-ci-main-selective.yml .github/workflows/
```

### Step 2: Organize Directory Structure

Ensure each notebook subdirectory has appropriate dependencies:

```bash
# Create directory-specific requirements
echo "pandas==2.1.0
matplotlib==3.7.0
seaborn==0.12.0" > notebooks/data_analysis/requirements.txt

echo "plotly==5.15.0
dash==2.11.0
bokeh==3.2.0" > notebooks/visualization/requirements.txt

echo "scikit-learn==1.3.0
xgboost==1.7.0
tensorflow==2.13.0" > notebooks/modeling/requirements.txt
```

### Step 3: Validate Structure

Use the validation script to check your setup:

```bash
# Run structure validation
bash scripts/validate-repository.sh your-repo-name

# Check for proper directory organization
find notebooks -name "requirements.txt" -type f
```

### Step 4: Test Selective Execution

Create test PRs to verify the selective behavior:

```bash
# Test 1: Directory-specific change
git checkout -b test-selective-data
echo "# Test change" >> notebooks/data_analysis/analysis1.ipynb
git add . && git commit -m "test: data analysis change"
git push origin test-selective-data
# Should only run data_analysis validation

# Test 2: Root requirements change
git checkout -b test-full-repo
echo "requests==2.31.0" >> requirements.txt
git add . && git commit -m "test: root requirements change"
git push origin test-full-repo
# Should run full repository validation
```

## :material-wrench: Advanced Configuration

### Custom Directory Patterns

Modify the workflow to support custom directory structures:

```yaml
# In detect-changes job, add custom patterns:
case "$file" in
  notebooks/experiments/*/requirements.txt)
    echo "  → Experiment-specific requirements changed: $file"
    dir=$(dirname "$file")
    AFFECTED_DIRECTORIES+=("$dir")
    ;;
```

### Dependency Inheritance

Create a hierarchy where subdirectories inherit from parent requirements:

```bash
# Root requirements (base dependencies)
echo "jupyter
nbval
pytest" > requirements.txt

# Directory adds specific dependencies
echo "-r ../../requirements.txt
pandas==2.1.0
matplotlib==3.7.0" > notebooks/data_analysis/requirements.txt
```

### Parallel Execution Limits

Control the number of parallel directory executions:

```yaml
strategy:
  matrix:
    directory: ${{ fromJson(needs.detect-changes.outputs.affected-directories) }}
  max-parallel: 3  # Limit concurrent jobs
  fail-fast: false  # Continue even if one directory fails
```

## :material-alert: Troubleshooting

### Common Issues

#### 1. Directory Not Detected
**Problem**: Changes in subdirectory don't trigger selective execution
**Solution**: Ensure the directory path matches the pattern `notebooks/*/`

#### 2. All Directories Running
**Problem**: Expected selective execution but all notebooks run
**Solution**: Check if root `requirements.txt` was modified (triggers full execution)

#### 3. Missing Dependencies
**Problem**: Directory-specific notebooks fail due to missing packages
**Solution**: Ensure each directory has complete `requirements.txt` or inherits from root

#### 4. Matrix Strategy Errors
**Problem**: Workflow fails with empty matrix
**Solution**: Check that `affected-directories` output contains valid directory paths

### Debug Mode

Enable detailed logging in workflows:

```yaml
- name: Debug selective execution
  run: |
    echo "Execution strategy: ${{ needs.detect-changes.outputs.execution-strategy }}"
    echo "Affected directories: ${{ needs.detect-changes.outputs.affected-directories }}"
    echo "Changed notebooks: ${{ needs.detect-changes.outputs.changed-notebooks }}"
```

## :material-chart-box: Performance Comparison

### Traditional vs Selective Execution

| Repository Size | Traditional Time | Selective Time | Savings |
|----------------|------------------|----------------|---------|
| Small (5 directories, 20 notebooks) | 15 minutes | 3-5 minutes | 67-80% |
| Medium (10 directories, 50 notebooks) | 25 minutes | 5-8 minutes | 68-80% |
| Large (20 directories, 100 notebooks) | 45 minutes | 8-12 minutes | 73-82% |

### Cost Analysis

Assuming GitHub Actions pricing of $0.008/minute:

| Change Type | Traditional Cost | Selective Cost | Monthly Savings* |
|-------------|------------------|----------------|------------------|
| Single directory change | $0.20 | $0.04 | $48/month |
| Documentation only | $0.20 | $0.024 | $52.8/month |
| Root requirements | $0.20 | $0.20 | $0/month |

*Based on 30 changes per month

## :material-road: Migration Path

### Phase 1: Assessment
1. Analyze current repository structure
2. Identify logical notebook groupings
3. Document existing dependencies

### Phase 2: Preparation
1. Organize notebooks into directories
2. Create directory-specific requirements files
3. Test local execution with new structure

### Phase 3: Implementation
1. Deploy selective workflows to staging
2. Test with various change scenarios
3. Monitor performance improvements

### Phase 4: Optimization
1. Fine-tune directory organization
2. Optimize parallel execution settings
3. Document best practices for team

## :material-note-text: Best Practices

1. **Logical Grouping**: Organize directories by functionality, not arbitrary splits
2. **Dependency Isolation**: Each directory should have self-contained requirements
3. **Shared Resources**: Use a common directory for shared utilities and configurations
4. **Testing Strategy**: Always test both selective and full execution paths
5. **Documentation**: Keep README files in each directory explaining the purpose and dependencies
6. **Monitoring**: Track performance improvements and adjust as needed

## :material-link-variant: Related Documentation

- [Unified Workflow Guide](https://github.com/spacetelescope/notebook-ci-actions/blob/main/README.md)
- [Configuration Reference](configuration-reference.md)
- [Example Workflows](https://github.com/spacetelescope/notebook-ci-actions/blob/main/examples/README.md)

---

**Implementation Date**: July 2025  
**Status**: :material-check-circle: Ready for Production  
**Compatibility**: Repositories with organized directory structures

---

## Smart Workflow System - Conditional CI Documentation

### :material-brain: Overview

The Smart Workflow System intelligently detects file changes in pull requests and main branch pushes to determine the optimal CI/CD path. This system dramatically reduces execution time and compute costs for documentation-only changes while maintaining full validation for notebook and code changes.

### :material-target: Key Benefits

#### :material-lightning-bolt: Performance Optimization
- **Documentation-only PRs**: ~2-5 minutes instead of 15-30 minutes
- **Config-only changes**: Skip notebook execution entirely
- **Faster feedback**: Immediate validation for non-notebook changes

#### :material-cash: Cost Reduction
- **Reduced GitHub Actions minutes**: Up to 80% savings on documentation PRs
- **Efficient resource usage**: Only run expensive operations when needed
- **Smart caching**: Optimized workflow execution paths

#### :material-lock: Quality Assurance
- **Full validation when needed**: Complete CI for notebook/code changes
- **Selective testing**: Targeted validation based on change type
- **Security scanning**: Maintained where appropriate

### :material-folder: Available Smart Workflows

#### 1. `notebook-ci-pr-smart.yml` - Smart Pull Request CI

**Purpose**: Intelligent PR validation with conditional execution paths

**Features**:
- **File Change Detection**: Analyzes PR changes to determine workflow path
- **Conditional Execution**: Full CI for notebooks, docs-only for config changes
- **Smart Feedback**: Clear summary of why specific CI path was chosen
- **Security Scanning**: Maintained for code changes

**Workflow Paths**:
- **Notebooks/Requirements Changed** → Full notebook validation + security scan + docs preview
- **Documentation/Config Changed** → Documentation rebuild only + preview
- **Mixed Changes** → Full CI path (safer approach)

#### 2. `notebook-ci-main-smart.yml` - Smart Main Branch CI  

**Purpose**: Optimized main branch CI with deployment intelligence

**Features**:
- **Production-Ready Execution**: Full notebook execution for main branch when needed
- **Documentation-Only Deployment**: Fast deployment for config-only changes
- **Performance Metrics**: Built-in timing and cost optimization reporting
- **Deployment Summary**: Clear feedback on what was deployed and why

**Workflow Paths**:
- **Notebooks/Requirements Changed** → Full CI + execution + deployment
- **Documentation/Config Changed** → Documentation rebuild + deployment
- **No Significant Changes** → Skip CI entirely

### :material-magnify: File Change Detection Logic

#### File Categories

The smart workflows categorize changed files into these groups:

##### :material-notebook: Notebook Files (Trigger Full CI)
```
notebooks/*.ipynb     # Jupyter notebooks
notebooks/*.py        # Python scripts  
notebooks/*.R         # R scripts
```

##### :material-wrench: Requirements Files (Trigger Full CI)
```
requirements.txt      # Python dependencies
pyproject.toml       # Python project config
setup.py            # Python package setup
```

##### :material-bookshelf: Documentation Files (Docs-Only Rebuild)
```
_config.yml          # JupyterBook config
_toc.yml            # Table of contents
*.md                # Markdown files
*.rst               # ReStructuredText files
scripts/*           # Helper scripts
*.html              # Static HTML
*.css               # Styling
*.js                # JavaScript
```

##### :material-cog: Workflow Files (Special Handling)
```
.github/workflows/*  # GitHub Actions workflows
*.yml               # Other YAML configs
*.yaml              # Other YAML configs
```

#### Decision Matrix

| Files Changed | PR Workflow | Main Branch Workflow | Reason |
|---------------|-------------|---------------------|--------|
| Only `.ipynb` | Full CI (validation-only) | Full CI + Execution | Notebooks need validation/execution |
| Only requirements | Full CI (validation-only) | Full CI + Execution | Dependencies affect notebook execution |
| Only `_config.yml` | Docs rebuild only | Docs rebuild + deploy | Documentation configuration |
| Only `.md` files | Docs rebuild only | Docs rebuild + deploy | Documentation content |
| Mixed changes | Full CI | Full CI + Execution | Safety: validate everything |
| Workflow files only | Minimal/Skip | Skip | No content changes |

### :material-tools: Implementation Guide

#### Step 1: Choose Your Smart Workflow

Replace your existing PR and main branch workflows with the smart versions:

```bash
# Remove old workflows
rm .github/workflows/notebook-ci-pr.yml
rm .github/workflows/notebook-ci-main.yml

# Add smart workflows
cp notebook-ci-pr-smart.yml .github/workflows/
cp notebook-ci-main-smart.yml .github/workflows/
```

#### Step 2: Repository-Specific Customization

##### For `jwst-pipeline-notebooks`:
```yaml
# In both smart workflows, ensure post-processing is configured:
post-run-script: "scripts/jdaviz_image_replacement.sh"
```

##### For `hst_notebooks`:
```yaml
# The smart workflows automatically detect hst_notebooks and use micromamba
# No additional configuration needed
```

##### For `hello_universe`:
```yaml
# Consider disabling security scanning for educational repository:
security-scan: false  # Add this to full-notebook-ci job
```

#### Step 3: Testing Smart Workflows

##### Test Documentation-Only Path:
```bash
# Create a PR that only changes documentation
git checkout -b test-docs-only
echo "# New section" >> README.md
git add README.md
git commit -m "docs: add new section"
git push origin test-docs-only
# Create PR - should trigger docs-only rebuild
```

##### Test Notebook Change Path:
```bash
# Create a PR that changes a notebook
git checkout -b test-notebook-change
# Edit a notebook file
git add notebooks/example.ipynb
git commit -m "feat: update example notebook"
git push origin test-notebook-change
# Create PR - should trigger full CI
```

#### Step 4: Monitor and Optimize

##### Workflow Run Analysis:
- Monitor GitHub Actions usage in repository Insights
- Track time savings in workflow summaries
- Review smart workflow decision accuracy

##### Fine-Tuning:
- Adjust file pattern matching if needed
- Customize execution modes per repository
- Update documentation rebuild triggers

### :material-chart-box: Performance Comparison

#### Traditional Workflow
```
Every PR: Full CI → 15-30 minutes
Every Push: Full CI → 15-30 minutes
Monthly Actions Minutes: ~500-1000 minutes
```

#### Smart Workflow
```
Documentation PR: Docs-only → 2-5 minutes
Notebook PR: Full CI → 15-30 minutes  
Documentation Push: Docs-only → 2-5 minutes
Notebook Push: Full CI → 15-30 minutes
Monthly Actions Minutes: ~200-400 minutes (60% savings)
```

#### Example Scenarios

##### Scenario 1: Documentation Update PR
- **Traditional**: 20 minutes (full notebook execution)
- **Smart**: 3 minutes (documentation rebuild only)
- **Time Saved**: 17 minutes (85% faster)

##### Scenario 2: Configuration File Update
- **Traditional**: 25 minutes (full CI + deployment)
- **Smart**: 4 minutes (docs rebuild + deployment)
- **Time Saved**: 21 minutes (84% faster)

##### Scenario 3: Notebook Bug Fix
- **Traditional**: 25 minutes (full CI + deployment)
- **Smart**: 25 minutes (full CI + deployment)
- **Time Saved**: 0 minutes (appropriate full validation)

### :material-wrench: Advanced Configuration

#### Custom File Patterns

To add custom file types to the documentation category:

```yaml
# In detect-changes job, add to the case statement:
*.custom)
  echo "  → Custom documentation file changed"
  DOCS_CONFIG_CHANGED=true
  ;;
```

#### Repository-Specific Logic

Add repository-specific detection logic:

```yaml
# Example: Special handling for specific repositories
- name: Repository-specific logic
  run: |
    if [ "${{ github.repository }}" = "spacetelescope/special_repo" ]; then
      # Custom logic for special_repo
      echo "special-handling=true" >> $GITHUB_OUTPUT
    fi
```

#### Environment-Specific Execution

Configure different execution modes based on file changes:

```yaml
# Example: More conservative approach for critical files
execution-mode: ${{ needs.detect-changes.outputs.notebooks-changed == 'true' && 'full' || 'validation-only' }}
```

### :material-alert: Troubleshooting

#### Common Issues

##### Smart Detection Not Working
**Symptoms**: All PRs trigger full CI even for documentation changes
**Solution**: 
1. Check file pattern matching in detect-changes job
2. Verify git diff is detecting changes correctly
3. Review changed files list in workflow logs

##### Docs-Only Builds Failing  
**Symptoms**: Documentation rebuilds fail for config-only changes
**Solution**:
1. Ensure all required files are present for documentation build
2. Check if notebooks are needed for documentation generation
3. Verify JupyterBook configuration is complete

##### False Positives in Detection
**Symptoms**: Documentation changes trigger full CI
**Solution**:
1. Review file categorization logic
2. Add specific file extensions to documentation category
3. Check for mixed change scenarios

#### Debug Mode

Enable detailed logging in smart workflows:

```yaml
# Add to detect-changes job for debugging
- name: Debug file detection
  run: |
    echo "Debug: Changed files analysis"
    git diff --name-status ${{ github.event.pull_request.base.sha }}..${{ github.event.pull_request.head.sha }}
    echo "Debug: Environment variables"
    env | grep GITHUB_ | head -10
```

### :material-note-text: Best Practices

#### 1. Gradual Rollout
- Test smart workflows in a development environment first
- Monitor performance improvements and accuracy
- Gradually roll out to all repositories

#### 2. Clear Communication
- Document the smart workflow behavior for contributors
- Add workflow summaries to PR descriptions
- Train team members on expected behavior

#### 3. Regular Monitoring
- Review workflow execution patterns monthly
- Analyze cost savings and performance improvements
- Adjust file detection logic based on usage patterns

#### 4. Fallback Strategy
- Keep traditional workflows as backup
- Implement easy rollback procedures
- Monitor for edge cases and unexpected behaviors

### :material-link-variant: Related Documentation

- [Core Workflows Documentation](https://github.com/spacetelescope/notebook-ci-actions/blob/main/README.md)
- [Example Workflows Guide](https://github.com/spacetelescope/notebook-ci-actions/blob/main/README.md)

### :material-lifebuoy: Support

For issues with smart workflows:
1. Check workflow logs for file detection details
2. Review this documentation for common solutions
3. Create an issue in the `notebook-ci-actions` repository
4. STScI staff: [Submit a ticket to SPB](https://innerspace.stsci.edu/pages/viewpage.action?pageId=637400835&spaceKey=DDP&title=DMD%2BDev%2BPortal%2BHome)

---

**Implementation Date**: July 2025  
**Status**: :material-check-circle: Ready for Production  
**Compatibility**: All STScI notebook repositories
