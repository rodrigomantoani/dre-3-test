### Missing Airflow User/Role in Postgres
**Issue Identified**: Through Docker Compose logs, it was observed that the role "airflow" did not exist in Postgres.  
**Resolution**: Adjusted the `compose.yaml` to correctly configure the PostgreSQL user. Also, created the "airflow" user in Postgres and assigned the necessary privileges.

- **Before**:
  ```yaml
  environment:
    POSTGRES_USER: admin
  ```

- **After**:
  ```yaml
  environment:
    POSTGRES_USER: airflow
  ```

- **Manual Fix** (alternative option):
  ```sql
  CREATE USER airflow WITH PASSWORD 'airflow';
  GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO airflow;
  GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO airflow;
  ```

---

### Incorrect DAG Volume Binding
**Issue Identified**: Detected through a pre-check on the `compose.yaml` that the DAG volume was incorrectly mounted, preventing the proper execution of DAGs.  
**Resolution**: Corrected the DAG volume binding to match the correct path in the repository.

- **Before**:
  ```yaml
  volumes:
    - ./dag:/opt/airflow/dags
  ```

- **After**:
  ```yaml
  volumes:
    - ./dags:/opt/airflow/dags
  ```

---

### Misconfigured Airflow Webserver Health Check
**Issue Identified**: The health check for the Airflow webserver was referencing an incorrect port, causing failures.  
**Resolution**: Aligned the health check URL with the correct service port.

- **Before**:
  ```yaml
  healthcheck:
    test: ["CMD", "curl", "--fail", "http://localhost:xxxx/health"]
  ```

- **After**:
  ```yaml
  healthcheck:
    test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
  ```

---

### Incorrect UID for Airflow User
**Issue Identified**: During the Airflow initialization step, insufficient path permissions were preventing successful DAG execution due to improper user ownership settings.  
**Resolution**: Updated the UID to ensure the correct user ownership for file permissions.

- **Before**:
  ```yaml
  user: "5000"
  ```

- **After**:
  ```yaml
  user: "1000"
  ```

---

### Syntax Error in DAG Code
**Issue Identified**: A syntax error in the DAG code was detected via the Airflow UI, causing import failures.  
**Resolution**: Corrected the missing colon in the function definition.

- **Before**:
  ```python
  def smooth() 
  ```

- **After**:
  ```python
  def smooth():
  ```

### Appendix: Steps Used to Validate and Troubleshoot

Below is a detailed breakdown of the steps I followed to validate and troubleshoot the Docker Compose setup for Airflow:

---

#### Step 1: Start the Containers
I initiated the Docker Compose setup by running the following command to bring up all the services as defined in the `compose.yaml` file.

```bash
docker-compose -f compose.yaml up -d
```

Output:
- The Docker Compose setup successfully pulled all necessary images and started the services.
- However, the `airflow-init` container failed to complete successfully with exit code 1.

---

#### Step 2: Investigate the `airflow-init` Failure
To diagnose the issue with the `airflow-init` container, I checked its logs:

```bash
docker-compose -f compose.yaml logs airflow-init
```

Key findings:
- The error message indicated a **password authentication failure** for the "airflow" user when attempting to connect to the Postgres database.
- The logs suggested that the `airflow-init` process could not complete due to this authentication issue.

---

#### Step 3: Connect to the Postgres Container
To address the issue, I accessed the running Postgres container and manually created the missing "airflow" user.

```bash
docker exec -it internal-postgres-1 psql -U admin -d airflow
```

Commands executed inside the Postgres container:

```sql
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
```

Validation:
- After executing these commands, I listed the roles to confirm that the "airflow" user was successfully created and had the appropriate privileges:

```sql
\du
```

---

#### Step 4: Restart the Docker Compose Setup
After resolving the database user issue, I restarted the `docker-compose` setup to verify that the services were running smoothly.

```bash
docker-compose -f compose.yaml up -d
```

Outcome:
- The services started successfully, and the `airflow-init` container completed its initialization without errors.
