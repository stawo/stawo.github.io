---
title: Complex TSQL stored procedures and PyODBC
description: How to properly write complex TSQL stored procedures that play nice with PyODBC (via SQLAlchemy).
tags:
  - sql
  - pyodbc
  - sqlalchemy
  - python
  - microsoft sql server
---

## Introduction

Assume you have a database that you want to manage via a Python program, but for certain operations over your tables (e.g., data updates), you need to execute a complex stored procedure.  
With complex I mean a stored procedure that might contain multiple statements like `INSERT`, `UPDATE`, and `DELETE`, possibly using variables and temporary tables too.

What I want:
- be able to call the stored procedure from my Python script;
- catch if any error happens during the execution of the stored procedure and details of it;
- capture some sort of log from the execution of the stored procedure (imagine if we could have something like `logger.info()` in the SQL code).

I must say that I banged my head more than I would have liked to get all of that using PyODBC and stored procedure over a MS SQL Server.  
PyODBC seems pretty picky of what can happen in the stored procedure when calling it.

Briefly, to make it work, the stored procedure:
- cannot contain `PRINT` and `RAISEERROR` statements;
- must declare `SET NOCOUNT ON` at the beginning of the procedure;
- can contain only 1 `SELECT` statement (if you use the `TRY...CATCH` statements, you can put 1 `SELECT` in each block).

And here is the SQL template that I created:
```sql
CREATE PROCEDURE [stored procedure name]
	-- INPUT PARAMETERS
	@input1 type,
    ...
	@inputN type,
	-- OUTPUT PARAMETERS - DO NOT MODIFY
	@sp_exit_value int OUTPUT,
	@sp_log nvarchar(max) OUTPUT,
	@sp_error nvarchar(max) OUTPUT
AS
BEGIN TRY

    -- MUST SET NOCOUNT ON
    -- Without it, the call from python will fail
    SET NOCOUNT ON;

    declare @message nvarchar(max);

    -- NCHAR(13)+NCHAR(10) represents a new line
    -- https://stackoverflow.com/questions/53115490/how-to-correctly-insert-newline-in-nvarchar
    set @message = concat(@message,NCHAR(13)+NCHAR(10),'Start');

    ----------------------------------------------------------------

    -- YOUR CODE HERE

    ----------------------------------------------------------------

    -- Output

    set @message = concat(@message,NCHAR(13)+NCHAR(10),'End');

    SELECT
        @sp_exit_value = 0,
        @sp_log = @message,
        @sp_error = Null
        ;

END TRY
BEGIN CATCH  
	
    SELECT
		@sp_exit_value = -1,
		@sp_log = @message,
		@sp_error = concat(
			'ERROR_NUMBER: ', ERROR_NUMBER(),
			NCHAR(13)+NCHAR(10),'ERROR_SEVERITY: ', ERROR_SEVERITY(),
			NCHAR(13)+NCHAR(10),'ERROR_STATE: ', ERROR_STATE(),
			NCHAR(13)+NCHAR(10),'ERROR_PROCEDURE: ', ERROR_PROCEDURE(),
			NCHAR(13)+NCHAR(10),'ERROR_LINE: ', ERROR_LINE(),
			NCHAR(13)+NCHAR(10),'ERROR_MESSAGE: ', ERROR_MESSAGE()
		)
		;
END CATCH;
```

You can modify the template above to fit your needs.  
In particulat, note the command:
```sql
set @message = concat(@message,NCHAR(13)+NCHAR(10),'YOUR TEXT');
```

That forms the logging functionality of the stored procedure.

In your Python code, we will call the stored procedure using the following template:
```python
import sqlalchemy

sp_name = "stored procedure name"
prefix = "dbo"

parameters = [
    input1 value,
    ...,
    inputN value,
]

# This part is needed as parameters needs to be escaped
# based on their type to be properly passed in the SQL
# command EXEC
escaped_parameters = []
    for parameter in parameters:
        if isinstance(parameter, (int, float)):
            escaped_parameters.append(str(parameter))
        elif isinstance(parameter, bool):
            escaped_parameters.append("1" if parameter else "0")
        else:
            escaped_parameters.append(f"'{str(parameter)}'")
    
sql_query = f"""\
DECLARE @exit_value int;
DECLARE @log nvarchar(max);
DECLARE @error nvarchar(max);
		EXEC [{prefix}].[{sp_name}] {', '.join(escaped_parameters)}{',' if len(escaped_parameters)>0 else ''} @sp_exit_value=@exit_value OUTPUT, @sp_log=@log OUTPUT, @sp_error=@error OUTPUT;
SELECT @exit_value AS exit_value, @log AS log, @error AS error;
"""

engine = sqlalchemy.create_engine("mssql+pyodbc:/// CONNECTION STRING")

with engine.begin() as connection:
    sql_results = connection.execute(sql_query)

    if sql_results.returns_rows:
        # We retrieve the first row of the results (which should be the only row)
        for row in sql_results:
            results = {
                "exit_value": row[0],
                "log": row[1],
                "error": row[2],
            }
            break
    else:
        ValueError("The stored procedure did not return any row, but it should!")

if results['exit_value'] < 0:
    message = f"Failed to execute stored procedure [{sp_name}].\nError:\n{results['error']}\n--------\nLog:\n{results['log']}\n--------"
    raise ExecError(message)
```

Just in case you were thinking of it, do not use the Pandas function `read_sql()` to execute the stored procedure.
This function, as the name implies (duh!), is designed to only read data, and does not commit any changes the stored procedure might perform.

```python
# DON'T DO THIS
pandas.read_sql(sql_query, engine)
```

Ok, that's all, have fun!

## Resources

- <https://www.sqlservertutorial.net/sql-server-stored-procedures/stored-procedure-output-parameters/>
- <https://docs.microsoft.com/en-us/sql/t-sql/language-elements/try-catch-transact-sql?view=sql-server-ver15>
