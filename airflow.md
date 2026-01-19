
**Date:** DD MMM YYYY  
**Context:** Brief explanation of what you were trying to do and what you expected to happen  
**Version:** Tool/framework version (e.g., Python 3.11, FastAPI 0.104.1, Docker 24.0.6)

## Error
```[language]
Paste the full error message or stack trace here.
If it's a custom exception, note the parent class.
```

## Code (Before)
```[language]
The problematic code that caused the error.
Include enough context to understand what's happening.
```

## Solution
```[language]
The corrected code or configuration.
Highlight what changed if it's not obvious.
```

## Explanation

A detailed breakdown of:
- **What caused the error** - Root cause analysis
- **Why the solution works** - The underlying reason
- **Key takeaways** - What to remember for next time
- **Watch out for** - Related gotchas or edge cases (optional)


---



**Date:** 23 Oct 2025
**Version:** Airflow 3.0.6

**Context:** Postgres COPY: Server-Side vs Client-Side. An Airflow worker generates a CSV file locally inside its container:

```python
@task
def create_file():
    df.to_csv('/opt/airflow/data/file.csv', index=False)
    return '/opt/airflow/data/file.csv'

```
A subsequent task attempts to load this file into Postgres using the COPY command:

```sql
sql = "COPY page_views FROM '/opt/airflow/data/file.csv' WITH (FORMAT csv);"
```
## Error
```
ERROR: could not open file "/opt/airflow/data/file.csv": No such file or directory
```
OR 

```
ERROR: could not open file "/opt/airflow/data/file.csv": permission denied
```


## Solution

Use COPY FROM STDIN to stream the file from the client (Airflow) to the server (Postgres).
This works regardless of filesystem differences.
```
sql = """
 COPY page_views
 FROM STDIN
 WITH (FORMAT csv, HEADER true, DELIMITER ',', QUOTE '"', ESCAPE '"');
 """

 # Open file on Airflow worker and stream to Postgres
 with open(file_path, 'r') as f:
 hook.copy_expert(sql, f)

 return f"Loaded data from {file_path}"
```

This works because the file is read from Airflow's filesystem and streamed to Postgres over the network connection.

## Explanation

**COPY FROM in Postgres executes server-side, meaning the Postgres backend itself tries to open the file path you specify.
That file must exist on the Postgres container’s filesystem, not Airflow’s.

When running Airflow and Postgres in separate Docker containers, their filesystems are isolated unless explicitly shared via Docker volumes. 
A file created by an Airflow worker at /opt/airflow/data/file.csv won’t exist in the Postgres container at the same path.  This means a file that exists in the Airflow worker won't be accessible to Postgres

**Key takeaway**: COPY FROM in Postgres executes on the server side. 
In Docker/containerized environments, use COPY FROM STDIN with copy_expert() to stream files from Airflow(client) to Postgres(server).

------



**Date:** 22 Oct 2025

**Context:** Templating File Paths in Postgres COPY Commands with Airflow

**Version:** Airflow 3.0.6, PostgreSQL v15.1 

## Error
```
ERROR: syntax error at or near "/"
```

## Code (Before)
```sql
sql = '''
COPY page_views
FROM {{ti.xcom_pull(task_ids='process_page_views_count')}}
WITH (FORMAT csv, HEADER true, DELIMITER ',', QUOTE '"', ESCAPE '"');
'''
```

## Solution
```sql
sql = '''
COPY page_views
FROM '{{ti.xcom_pull(task_ids='process_page_views_count')}}'
WITH (FORMAT csv, HEADER true, DELIMITER ',', QUOTE '"', ESCAPE '"');
'''
```

## Explanation

**Postgres sees the path as separate tokens, not a string**
Wrapping the Jinja template in single quotes ensure that after Airflow expands the template,
the resulting path is treated as a SQL string literal rather than bare tokens.

**Key takeaway**: When templating file paths or any string values in SQL, always wrap the Jinja
template expression in single quotes so the rendered result is a valid SQL string literal.

