<!-- DO NOT EDIT THIS FILE DIRECTLY - it is generated from source file PLEX.pks -->

PL/SQL Export Utilities
=======================

- [Package PLEX](#plex)
- [Function backapp](#backapp)
- [Procedure add_query](#add_query)
- [Function queries_to_csv](#queries_to_csv)
- [Function to_zip](#to_zip)
- [Function view_error_log](#view_error_log)
- [Function view_runtime_log](#view_runtime_log)


<h2><a id="plex"></a>Package PLEX</h2>
<!----------------------------------->

PLEX was created to be able to quickstart version control for existing (APEX) apps and has currently two main functions called **BackApp** and **Queries_to_CSV**. Queries_to_CSV is used by BackApp as a helper function, but its functionality is also useful standalone.

See also this resources for more information:

- [Blog post on how to getting started](https://ogobrecht.github.io/posts/2018-08-26-plex-plsql-export-utilities)
- [PLEX project page on GitHub](https://github.com/ogobrecht/plex)
- [Give feedback on GitHub](https://github.com/ogobrecht/plex/issues/new).


DEPENDENCIES

The package itself is independend, but functionality varies on the following conditions:

- For APEX app export: APEX >= 5.1.4 installed
- For ORDS modules export: ORDS >= FIXME installed


INSTALLATION

- Download the [latest version](https://github.com/ogobrecht/plex/releases/latest)
- Unzip it, open a shell and go into the root directory
- Start SQL*Plus (or another tool which can run SQL scripts)
- To install PLEX run the provided install script `plex_install.sql` (script provides compiler flags)
- To uninstall PLEX run the provided script `plex_uninstall.sql` or drop the package manually


CHANGELOG

- 2.1.0 (2019-xx-xx)
    - New parameter to include ORDS modules
- 2.0.2 (2019-08-16)
    - Fixed: Function BackApp throws error on large APEX UI install files (ORA-06502: PL/SQL: numeric or value error: character string buffer too small)
- 2.0.1 (2019-07-09)
    - Fixed: Compile error when DB version is lower then 18.1 (PLS-00306: wrong number or types of arguments in call to 'REC_EXPORT_FILE')
- 2.0.0 (2019-06-20)
    - Package is now independend from APEX to be able to export schema object DDL and table data without an APEX installation
        - ATTENTION: The return type of functions BackApp and Queries_to_CSV has changed from `apex_t_export_files` to `plex.tab_export_files`
    - New parameters to filter for object types
    - New parameters to change base paths for backend, frontend and data
- 1.2.1 (2019-03-13)
    - Fix script templates: Change old parameters in plex.backapp call
    - Add install and uninstall scripts for PLEX itself
- 1.2.0 (2018-10-31)
    - New: All like/not like parameters are now translated internally with the escape character set to backslash like so `... like 'YourExpression' escape '\'`
    - Fixed: Binary data type columns (raw, long_raw, blob, bfile) should no longer break the export data to CSV functionality
- 1.1.0 (2018-09-23)
    - Change filter parameter from regular expression to list of like expressions for easier handling
- 1.0.0 (2018-08-26)
    - First public release

SIGNATURE

```sql
PACKAGE PLEX AUTHID current_user IS
c_plex_name        CONSTANT VARCHAR2(30 CHAR) := 'PLEX - PL/SQL Export Utilities';
c_plex_version     CONSTANT VARCHAR2(10 CHAR) := '2.0.2';
c_plex_url         CONSTANT VARCHAR2(40 CHAR) := 'https://github.com/ogobrecht/plex';
c_plex_license     CONSTANT VARCHAR2(10 CHAR) := 'MIT';
c_plex_license_url CONSTANT VARCHAR2(60 CHAR) := 'https://github.com/ogobrecht/plex/blob/master/LICENSE.txt';
c_plex_author      CONSTANT VARCHAR2(20 CHAR) := 'Ottmar Gobrecht';
```


<h2><a id="backapp"></a>Function backapp</h2>
<!------------------------------------------>

Get a file collection of an APEX application (or the current user/schema only) including:

- The app export SQL files splitted ready to use for version control and deployment
- Optional the DDL scripts for all objects and grants
- Optional the data in CSV files (this option was implemented to track catalog tables, can be used as logical backup, has the typical CSV limitations...)
- Everything in a (hopefully) nice directory structure

EXAMPLE BASIC USAGE

```sql
DECLARE
  l_file_collection plex.tab_export_files;
BEGIN
  l_file_collection := plex.backapp(
    p_app_id               => 100,   -- parameter only available when APEX is installed
    p_include_ords_modules => false, -- parameter only available when ORDS is installed
    p_include_object_ddl   => false,
    p_include_data         => false);

  -- do something with the file collection
  FOR i IN 1..l_file_collection.count LOOP
    dbms_output.put_line(i || ' | '
      || lpad(round(length(l_file_collection(i).contents) / 1024), 3) || ' kB' || ' | '
      || l_file_collection(i).name);
  END LOOP;
END;
/
```

EXAMPLE ZIP FILE PL/SQL

```sql
DECLARE
  l_zip_file BLOB;
BEGIN
  l_zip_file := plex.to_zip(plex.backapp(
    p_app_id               => 100,  -- parameter only available when APEX is installed
    p_include_ords_modules => true, -- parameter only available when ORDS is installed
    p_include_object_ddl   => true,
    p_include_data         => false));
  -- do something with the zip file
  -- Your code here...
END;
/
```

EXAMPLE ZIP FILE SQL

```sql
-- Inline function because of boolean parameters (needs Oracle 12c or higher).
-- Alternative create a helper function and call that in a SQL context.
WITH
  FUNCTION backapp RETURN BLOB IS
  BEGIN
    RETURN plex.to_zip(plex.backapp(
      -- All parameters are optional and shown with their defaults
      -- APEX App (only available, when APEX is installed):
      p_app_id                    => NULL,
      p_app_date                  => true,
      p_app_public_reports        => true,
      p_app_private_reports       => false,
      p_app_notifications         => false,
      p_app_translations          => true,
      p_app_pkg_app_mapping       => false,
      p_app_original_ids          => false,
      p_app_subscriptions         => true,
      p_app_comments              => true,
      p_app_supporting_objects    => NULL,
      p_app_include_single_file   => false,
      p_app_build_status_run_only => false,
      -- ORDS Modules (only available, when ORDS is installed):
      p_include_ords_modules      => false,
      -- Schema Objects:
      p_include_object_ddl        => false,
      p_object_type_like          => NULL,
      p_object_type_not_like      => NULL,
      p_object_name_like          => NULL,
      p_object_name_not_like      => NULL,
      -- Table Data:
      p_include_data              => false,
      p_data_as_of_minutes_ago    => 0,
      p_data_max_rows             => 1000,
      p_data_table_name_like      => NULL,
      p_data_table_name_not_like  => NULL,
      -- General options:
      p_include_templates         => true,
      p_include_runtime_log       => true,
      p_include_error_log         => true,
      p_base_path_backend         => 'app_backend',
      p_base_path_frontend        => 'app_frontend',
      p_base_path_web_services    => 'app_web_services',
      p_base_path_data            => 'app_data'));
  END backapp;
SELECT backapp FROM dual;
```

SIGNATURE

```sql
FUNCTION backapp (
  $if $$apex_installed $then
  -- APEX App:
  p_app_id                    IN NUMBER   DEFAULT null,  -- If null, we simply skip the APEX app export.
  p_app_date                  IN BOOLEAN  DEFAULT true,  -- If true, include export date and time in the result.
  p_app_public_reports        IN BOOLEAN  DEFAULT true,  -- If true, include public reports that a user saved.
  p_app_private_reports       IN BOOLEAN  DEFAULT false, -- If true, include private reports that a user saved.
  p_app_notifications         IN BOOLEAN  DEFAULT false, -- If true, include report notifications.
  p_app_translations          IN BOOLEAN  DEFAULT true,  -- If true, include application translation mappings and all text from the translation repository.
  p_app_pkg_app_mapping       IN BOOLEAN  DEFAULT false, -- If true, export installed packaged applications with references to the packaged application definition. If FALSE, export them as normal applications.
  p_app_original_ids          IN BOOLEAN  DEFAULT false, -- If true, export with the IDs as they were when the application was imported.
  p_app_subscriptions         IN BOOLEAN  DEFAULT true,  -- If true, components contain subscription references.
  p_app_comments              IN BOOLEAN  DEFAULT true,  -- If true, include developer comments.
  p_app_supporting_objects    IN VARCHAR2 DEFAULT null,  -- If 'Y', export supporting objects. If 'I', automatically install on import. If 'N', do not export supporting objects. If null, the application's include in export deployment value is used.
  p_app_include_single_file   IN BOOLEAN  DEFAULT false, -- If true, the single sql install file is also included beside the splitted files.
  p_app_build_status_run_only IN BOOLEAN  DEFAULT false, -- If true, the build status of the app will be overwritten to RUN_ONLY.
  $end
  $if $$ords_installed $then
  -- ORDS Modules:
  p_include_ords_modules      IN BOOLEAN  DEFAULT false, -- If true, include ORDS modules of current user/schema.
  $end
  -- Schema Objects:
  p_include_object_ddl        IN BOOLEAN  DEFAULT false, -- If true, include DDL of current user/schema and all its objects.
  p_object_type_like          IN VARCHAR2 DEFAULT null,  -- A comma separated list of like expressions to filter the objects - example: '%BODY,JAVA%' will be translated to: ... from user_objects where ... and (object_type like '%BODY' escape '\' or object_type like 'JAVA%' escape '\').
  p_object_type_not_like      IN VARCHAR2 DEFAULT null,  -- A comma separated list of not like expressions to filter the objects - example: '%BODY,JAVA%' will be translated to: ... from user_objects where ... and (object_type not like '%BODY' escape '\' and object_type not like 'JAVA%' escape '\').
  p_object_name_like          IN VARCHAR2 DEFAULT null,  -- A comma separated list of like expressions to filter the objects - example: 'EMP%,DEPT%' will be translated to: ... from user_objects where ... and (object_name like 'EMP%' escape '\' or object_name like 'DEPT%' escape '\').
  p_object_name_not_like      IN VARCHAR2 DEFAULT null,  -- A comma separated list of not like expressions to filter the objects - example: 'EMP%,DEPT%' will be translated to: ... from user_objects where ... and (object_name not like 'EMP%' escape '\' and object_name not like 'DEPT%' escape '\').
  -- Table Data:
  p_include_data              IN BOOLEAN  DEFAULT false, -- If true, include CSV data of each table.
  p_data_as_of_minutes_ago    IN NUMBER   DEFAULT 0,     -- Read consistent data with the resulting timestamp(SCN).
  p_data_max_rows             IN NUMBER   DEFAULT 1000,  -- Maximum number of rows per table.
  p_data_table_name_like      IN VARCHAR2 DEFAULT null,  -- A comma separated list of like expressions to filter the tables - example: 'EMP%,DEPT%' will be translated to: where ... and (table_name like 'EMP%' escape '\' or table_name like 'DEPT%' escape '\').
  p_data_table_name_not_like  IN VARCHAR2 DEFAULT null,  -- A comma separated list of not like expressions to filter the tables - example: 'EMP%,DEPT%' will be translated to: where ... and (table_name not like 'EMP%' escape '\' and table_name not like 'DEPT%' escape '\').
  -- General Options:
  p_include_templates         IN BOOLEAN  DEFAULT true,  -- If true, include templates for README.md, export and install scripts.
  p_include_runtime_log       IN BOOLEAN  DEFAULT true,  -- If true, generate file plex_runtime_log.md with detailed runtime infos.
  p_include_error_log         IN BOOLEAN  DEFAULT true,  -- If true, generate file plex_error_log.md with detailed error messages.
  p_base_path_backend         IN VARCHAR2 DEFAULT 'app_backend',      -- The base path in the project root for the Schema objects.
  p_base_path_frontend        IN VARCHAR2 DEFAULT 'app_frontend',     -- The base path in the project root for the APEX app.
  p_base_path_web_services    IN VARCHAR2 DEFAULT 'app_web_services', -- The base path in the project root for the ORDS modules.
  p_base_path_data            IN VARCHAR2 DEFAULT 'app_data')         -- The base path in the project root for the table data.
RETURN tab_export_files;
```


<h2><a id="add_query"></a>Procedure add_query</h2>
<!----------------------------------------------->

Add a query to be processed by the method queries_to_csv. You can add as many queries as you like.

EXAMPLE

```sql
BEGIN
  plex.add_query(
    p_query     => 'select * from user_tables',
    p_file_name => 'user_tables');
END;
/
```

SIGNATURE

```sql
PROCEDURE add_query (
  p_query     IN VARCHAR2,                -- The query itself
  p_file_name IN VARCHAR2,                -- File name like 'Path/to/your/file-without-extension'.
  p_max_rows  IN NUMBER    DEFAULT 1000); -- The maximum number of rows to be included in your file.
```


<h2><a id="queries_to_csv"></a>Function queries_to_csv</h2>
<!-------------------------------------------------------->

Export one or more queries as CSV data within a file collection.

EXAMPLE BASIC USAGE

```sql
DECLARE
  l_file_collection plex.tab_export_files;
BEGIN
  --fill the queries array
  plex.add_query(
    p_query     => 'select * from user_tables',
    p_file_name => 'user_tables');
  plex.add_query(
    p_query     => 'select * from user_tab_columns',
    p_file_name => 'user_tab_columns',
    p_max_rows  => 10000);
  -- process the queries
  l_file_collection := plex.queries_to_csv;
  -- do something with the file collection
  FOR i IN 1..l_file_collection.count LOOP
    dbms_output.put_line(i || ' | '
      || lpad(round(length(l_file_collection(i).contents) / 1024), 3) || ' kB' || ' | '
      || l_file_collection(i).name);
  END LOOP;
END;
/
```

EXPORT EXPORT ZIP FILE PL/SQL

```sql
DECLARE
  l_zip_file BLOB;
BEGIN
  --fill the queries array
  plex.add_query(
    p_query     => 'select * from user_tables',
    p_file_name => 'user_tables');
  plex.add_query(
    p_query     => 'select * from user_tab_columns',
    p_file_name => 'user_tab_columns',
    p_max_rows  => 10000);
  -- process the queries
  l_zip_file := plex.to_zip(plex.queries_to_csv);
  -- do something with the zip file
  -- Your code here...
END;
/
```

EXAMPLE EXPORT ZIP FILE SQL

```sql
WITH
  FUNCTION queries_to_csv_zip RETURN BLOB IS
    v_return BLOB;
  BEGIN
    plex.add_query(
      p_query     => 'select * from user_tables',
      p_file_name => 'user_tables');
    plex.add_query(
      p_query     => 'select * from user_tab_columns',
      p_file_name => 'user_tab_columns',
      p_max_rows  => 10000);
    v_return := plex.to_zip(plex.queries_to_csv);
    RETURN v_return;
  END queries_to_csv_zip;
SELECT queries_to_csv_zip FROM dual;
```

SIGNATURE

```sql
FUNCTION queries_to_csv (
  p_delimiter                 IN VARCHAR2 DEFAULT ',',   -- The column delimiter.
  p_quote_mark                IN VARCHAR2 DEFAULT '"',   -- Used when the data contains the delimiter character.
  p_header_prefix             IN VARCHAR2 DEFAULT NULL,  -- Prefix the header line with this text.
  p_include_runtime_log       IN BOOLEAN  DEFAULT true,  -- If true, generate file plex_runtime_log.md with runtime statistics.
  p_include_error_log         IN BOOLEAN  DEFAULT true)   -- If true, generate file plex_error_log.md with detailed error messages.
RETURN tab_export_files;
```


<h2><a id="to_zip"></a>Function to_zip</h2>
<!---------------------------------------->

Convert a file collection to a zip file.

EXAMPLE

```sql
DECLARE
  l_zip BLOB;
BEGIN
  l_zip := plex.to_zip(plex.backapp(
    p_app_id             => 100,
    p_include_object_ddl => true));
  -- do something with the zip file...
END;
```

SIGNATURE

```sql
FUNCTION to_zip (
  p_file_collection IN tab_export_files) -- The file collection to zip.
RETURN BLOB;
```


<h2><a id="view_error_log"></a>Function view_error_log</h2>
<!-------------------------------------------------------->

View the error log from the last plex run. The internal array for the error log is cleared on each call of BackApp or Queries_to_CSV.

EXAMPLE

```sql
SELECT * FROM TABLE(plex.view_error_log);
```

SIGNATURE

```sql
FUNCTION view_error_log RETURN tab_error_log PIPELINED;
```


<h2><a id="view_runtime_log"></a>Function view_runtime_log</h2>
<!------------------------------------------------------------>

View the runtime log from the last plex run. The internal array for the runtime log is cleared on each call of BackApp or Queries_to_CSV.

EXAMPLE

```sql
SELECT * FROM TABLE(plex.view_runtime_log);
```

SIGNATURE

```sql
FUNCTION view_runtime_log RETURN tab_runtime_log PIPELINED;
```


