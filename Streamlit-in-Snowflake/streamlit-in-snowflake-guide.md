# Streamlit in Snowflake Development Guide for Cursor.ai

## Overview
This guide provides comprehensive rules and guidelines for developing Streamlit applications within Snowflake (SiS). Follow these instructions carefully to avoid common pitfalls and ensure successful deployment.

## 🚨 Critical Limitations

### Security & Network Restrictions
- **Content Security Policy (CSP)**: Snowflake blocks loading code from external domains
  - ❌ NO external scripts, styles, fonts, or iframe embedding
  - ✅ Loading images/media from external domains is allowed
  - ❌ Frontend calls like `eval()` are blocked
  - ❌ External component calls that require external services

### Size & Performance Limits
- **32 MB message limit** between backend and frontend
  - Design apps to retrieve data in increments < 32 MB
  - Use pagination for large datasets
- **200 MB limit** for `st.file_uploader`
- **250 MB limit** per file when uploading to stage
- **WebSocket timeout**: Connection expires ~15 minutes after last use

### Feature Restrictions
- ❌ **External stages** not supported
- ❌ **Replication** not supported
- ❌ **.so files** not supported
- ❌ **Temporary tables/stages** not supported
- ⚠️ **Caching limitations**: 
  - `st.cache_data` and `st.cache_resource` only work within single session
  - No cache sharing between users/sessions
- ⚠️ **Query parameters**: `streamlit-` prefix added to all URL parameters
- ❌ **Server-side encryption** for stages in editor not supported

### Package Management
- **ONLY packages from Snowflake Anaconda Channel** are supported
- ❌ NO external Anaconda channels
- ❌ NO pip install from PyPI
- Default packages: `python`, `streamlit`, `snowflake-snowpark-python`

## 📋 Prerequisites & Permissions

### Required Privileges for Development
```sql
-- For creating/editing Streamlit apps
GRANT USAGE ON DATABASE <database> TO ROLE <role>;
GRANT USAGE ON SCHEMA <schema> TO ROLE <role>;
GRANT CREATE STREAMLIT ON SCHEMA <schema> TO ROLE <role>;
GRANT CREATE STAGE ON SCHEMA <schema> TO ROLE <role>;
GRANT USAGE ON WAREHOUSE <warehouse> TO ROLE <role>;
```

### Required for Viewers
```sql
-- For viewing Streamlit apps
GRANT USAGE ON DATABASE <database> TO ROLE <viewer_role>;
GRANT USAGE ON SCHEMA <schema> TO ROLE <viewer_role>;
GRANT USAGE ON STREAMLIT <app_name> TO ROLE <viewer_role>;
```

### Network Requirements
- **Allowlist**: `*.snowflake.app` and `*.snowflake.com`
- **WebSockets**: Must not be blocked
- **Firewall**: Add domains to allowlist for app communication

## 🏗️ Project Structure

### Recommended Directory Layout
```
project/
├── .github/
│   └── workflows/
│       └── deploy.yml           # CI/CD workflow
├── src/
│   ├── streamlit_app.py        # Main app file
│   └── pages/                  # Multi-page apps
│       ├── page1.py
│       └── page2.py
├── config/
│   ├── config.toml             # SnowCLI config
│   └── connection.toml         # Local connection config
├── environment.yml              # Snowflake environment
├── requirements.txt            # Local development packages
├── snowflake.yml               # Deployment manifest
└── README.md
```

### Environment Configuration (environment.yml)
```yaml
name: sf_env
channels:
  - snowflake  # REQUIRED - only channel allowed
dependencies:
  - streamlit=1.35.0  # Pin version to prevent auto-upgrade
  - pandas
  - numpy
  - scikit-learn
  # Only packages from Snowflake Anaconda Channel
```

### Deployment Manifest (snowflake.yml)
```yaml
definition_version: 2
streamlit:
  - name: my_streamlit_app
    stage: streamlit_stage
    query_warehouse: STREAMLIT_WH
    main_file: src/streamlit_app.py
    title: "My Streamlit App"
    pages_dir: src/pages/  # Optional for multi-page apps
    environment_file: environment.yml
```

## 💻 Development Workflow

### 1. Local Development Setup
```python
# Connection handling for local vs Snowflake environment
import streamlit as st
import os

# Check if running in Snowflake
if 'SNOWFLAKE_ACCOUNT' in os.environ:
    # Running in Snowflake
    from snowflake.snowpark.context import get_active_session
    session = get_active_session()
else:
    # Local development
    session = st.connection('snowflake').session()
```

### 2. Data Access Pattern
```python
# ALWAYS use owner's rights pattern
# App runs with privileges of the owner, not the viewer

@st.cache_data
def load_data():
    # Use session from above
    query = """
    SELECT * FROM database.schema.table
    WHERE condition = true
    LIMIT 10000  -- Keep under 32MB limit
    """
    return session.sql(query).to_pandas()
```

### 3. Warehouse Management
```python
# Use different warehouses for different operations
def run_heavy_computation():
    # Switch to larger warehouse for complex queries
    session.sql("USE WAREHOUSE LARGE_WH").collect()
    result = session.sql(complex_query).collect()
    
    # Switch back to smaller warehouse
    session.sql("USE WAREHOUSE STREAMLIT_WH").collect()
    return result
```

### 4. External Access Integration
```python
# For external API calls (requires setup)
import _snowflake

# Access secrets securely
api_key = _snowflake.get_generic_secret_string('my_api_key')

# Use in external API calls
import requests
response = requests.get(
    "https://api.example.com/data",
    headers={"Authorization": f"Bearer {api_key}"}
)
```

## 🧪 Testing Strategy

### Local Testing
1. **Use SnowCLI for local development**:
```bash
# Initialize project
snow streamlit init my_app

# Deploy to test environment
snow streamlit deploy --replace
```

2. **Test with sample data locally**:
```python
# Create test fixtures
if st.secrets.get("environment") == "test":
    df = pd.read_csv("test_data.csv")
else:
    df = load_from_snowflake()
```

### Integration Testing
```python
# Test Snowflake connectivity
def test_connection():
    try:
        session.sql("SELECT CURRENT_ROLE()").collect()
        return True
    except Exception as e:
        st.error(f"Connection failed: {e}")
        return False
```

## 🚀 Deployment Process

### CI/CD with GitHub Actions
```yaml
# .github/workflows/deploy.yml
name: Deploy Streamlit to Snowflake
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install SnowCLI
        run: |
          pip install snowflake-cli-labs
      
      - name: Configure SnowCLI
        env:
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: |
          mkdir -p ~/.snowflake
          cat > ~/.snowflake/config.toml << EOF
          [connections.default]
          account = "${{ secrets.SNOWFLAKE_ACCOUNT }}"
          user = "${{ secrets.SNOWFLAKE_USER }}"
          role = "${{ secrets.SNOWFLAKE_ROLE }}"
          warehouse = "STREAMLIT_WH"
          database = "STREAMLIT_DB"
          schema = "STREAMLIT_SCHEMA"
          password = "$SNOWFLAKE_PASSWORD"
          EOF
      
      - name: Deploy Application
        run: |
          snow streamlit deploy --replace
```

### Manual Deployment via SQL
```sql
-- Create or replace Streamlit app
CREATE OR REPLACE STREAMLIT app_name
  ROOT_LOCATION = '@stage_name/app_folder'
  MAIN_FILE = 'streamlit_app.py'
  QUERY_WAREHOUSE = 'STREAMLIT_WH'
  TITLE = 'My App'
  COMMENT = 'Production version';

-- With external access (if needed)
ALTER STREAMLIT app_name 
  SET EXTERNAL_ACCESS_INTEGRATIONS = (api_integration)
  SECRETS = ('api_key' = db.schema.secret_name);
```

### Git Repository Integration
```sql
-- Create git repository
CREATE GIT REPOSITORY my_repo
  API_INTEGRATION = git_api_integration
  ORIGIN = 'https://github.com/org/repo.git';

-- Create Streamlit from git
CREATE STREAMLIT app_from_git
  ROOT_LOCATION = '@my_repo/branches/main/src'
  MAIN_FILE = 'app.py'
  QUERY_WAREHOUSE = 'STREAMLIT_WH';

-- Auto-refresh from git
CREATE TASK refresh_git_task
  WAREHOUSE = 'TASK_WH'
  SCHEDULE = 'USING CRON 0 * * * * UTC'
AS
  ALTER GIT REPOSITORY my_repo FETCH;
```

## ⚠️ Common Pitfalls & Solutions

### Issue: App doesn't load
**Solution**: Check firewall allows `*.snowflake.app`, WebSockets enabled

### Issue: Query returns error about 32MB limit
**Solution**: 
```python
# Paginate large queries
def get_paginated_data(offset=0, limit=10000):
    query = f"""
    SELECT * FROM large_table
    LIMIT {limit} OFFSET {offset}
    """
    return session.sql(query).to_pandas()
```

### Issue: Package not found
**Solution**: Only use packages from [Snowflake Anaconda Channel](https://repo.anaconda.com/pkgs/snowflake/)

### Issue: Cache not working between users
**Solution**: This is by design. Use Snowflake tables for shared state:
```python
# Store shared state in Snowflake
def save_state(key, value):
    session.sql(f"""
        MERGE INTO app_state USING (
            SELECT '{key}' as key, '{value}' as value
        ) s ON app_state.key = s.key
        WHEN MATCHED THEN UPDATE SET value = s.value
        WHEN NOT MATCHED THEN INSERT (key, value) VALUES (s.key, s.value)
    """).collect()
```

### Issue: External API calls failing
**Solution**: Set up External Access Integration properly:
```sql
-- 1. Create network rule
CREATE NETWORK RULE api_rule
  TYPE = HOST_PORT
  MODE = EGRESS
  VALUE_LIST = ('api.example.com:443');

-- 2. Create integration
CREATE EXTERNAL ACCESS INTEGRATION api_integration
  ALLOWED_NETWORK_RULES = (api_rule)
  ENABLED = TRUE;

-- 3. Grant to Streamlit
ALTER STREAMLIT my_app
  SET EXTERNAL_ACCESS_INTEGRATIONS = (api_integration);
```

## 📊 Performance Best Practices

1. **Warehouse Optimization**
   - Use dedicated warehouse for Streamlit apps
   - Set auto-suspend to minimum 30 seconds
   - Use `USE WAREHOUSE` for query-specific sizing

2. **Query Optimization**
   - Always include LIMIT clauses
   - Use result caching where possible
   - Implement pagination for large datasets

3. **Session Management**
   ```python
   # Reuse session throughout app
   @st.cache_resource
   def get_session():
       return get_active_session()
   ```

4. **Data Loading**
   ```python
   # Cache data appropriately
   @st.cache_data(ttl=3600)  # 1 hour cache
   def load_reference_data():
       return session.table("reference_table").to_pandas()
   ```

## 🔒 Security Considerations

1. **Owner's Rights Model**
   - Apps run with owner's privileges
   - Viewers see data through owner's access
   - Be cautious with sensitive data exposure

2. **Secret Management**
   ```python
   # Never hardcode credentials
   # Use Snowflake secrets
   import _snowflake
   secret = _snowflake.get_generic_secret_string('secret_name')
   ```

3. **Input Validation**
   ```python
   # Always validate user inputs
   user_input = st.text_input("Enter ID")
   if user_input:
       # Sanitize input to prevent injection
       safe_input = session.sql(
           "SELECT :1 as clean_input", 
           params=[user_input]
       ).collect()[0]['CLEAN_INPUT']
   ```

## 📚 Additional Resources

## 📚 Additional Resources

### Essential Documentation
- [Snow CLI Documentation](https://docs.snowflake.com/en/developer-guide/snowflake-cli/index)
- [Snow CLI Streamlit Commands](https://docs.snowflake.com/en/developer-guide/snowflake-cli/streamlit-apps/overview)
- [Snowflake Streamlit Documentation](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit)
- [Snowflake Anaconda Channel](https://repo.anaconda.com/pkgs/snowflake/)

### Example Repositories
- [Official Streamlit Demos](https://github.com/Snowflake-Labs/snowflake-demo-streamlit)
- [Snow CLI Examples](https://github.com/Snowflake-Labs/snowcli-examples)

### Community Resources
- [Snowflake Community](https://community.snowflake.com/)
- [Streamlit Forum](https://discuss.streamlit.io/)

## Version History
- Last Updated: 2025-01-06
- Snow CLI Version: 2.x (snowflake-cli-labs)
- Based on Snowflake Documentation as of January 2025
- Streamlit version: 1.35.0 (latest stable in Snowflake)