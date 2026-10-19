TMDL SOURCE TABLES
==================

Find every database table and view a Power BI semantic model reads from,
straight from its TMDL files.

Paste TMDL, get a list of source objects. No install, no server, no build.
One HTML file that runs entirely in your browser.


WHY
---
"Which tables does this report actually read from?" usually means opening
every query one by one. This tool answers it in seconds:

  - Impact analysis     what breaks if this source table changes?
  - Access requests     which tables does the service account need SELECT on?
  - Migration / docs    source-to-model lineage for documentation


QUICK START
-----------
1. Download the HTML file from this repository
   (Code > Download ZIP, or open the file and click "Download raw file").
2. Double-click it. It opens in your default browser.
   Note: opening the file on github.com only shows its source code. Download
   it, or use the GitHub Pages link if the repository owner enabled one.
3. Get your TMDL:
   - Power BI Desktop > TMDL view > copy the script, or
   - in a PBIP project, open <name>.SemanticModel/definition/ and copy the
     contents of tables/*.tmdl and expressions.tmdl
     (you can paste several files one after another into the same box).
4. Paste into the left panel. Results appear about 0.3 s after you stop
   typing. Ctrl+Enter (or Cmd+Enter) or the Extract button runs it at once.


WHAT YOU GET
------------
Summary chips      model tables, partitions/expressions, how many contain SQL,
                   unique source objects, CTEs skipped, text parameters
                   resolved, and a warning chip if some source was not found.

Source objects     the right-hand panel, two views:
                   - By object:       each source table/view once, with the
                                      model tables using it and a ref count
                   - By model table:  each model table with its connection and
                                      the objects it reads
                   A filter box narrows either view.

Per partition      one collapsible row per partition/expression with the
                   connection, detected objects, skipped CTEs and the decoded
                   SQL with detected tables highlighted. Use it to verify the
                   result.

Buttons
  Extract                  run the analysis again
  Copy object list         one object name per line
  Copy as table (Excel)    tab-separated, paste straight into Excel
                           (columns: Object, Schema, Name, Via,
                           Used by model table, Refs)
  Clear                    empty the input and results

Options (checkboxes)
  Uppercase names          treat SCHEMA.Table and schema.table as one object
  Hide DUAL                hide Oracle's DUAL pseudo-table
  Include M navigation     also report objects reached by M steps such as
                           Source{[Schema="dbo", Item="DimDate"]}[Data]
  Resolve text parameters  replace text parameters used inside SQL strings
                           with their value (see "How it works")


HOW IT WORKS
------------
1. The TMDL is read line by line. Each table's partitions and each top-level
   expression (query/parameter) is collected. Tables may be nested under
   "model Model" (Power BI's TMDL view export) or stand alone (PBIP files).
2. The M code of each source is scanned for string literals. SQL written as
   one string with #(lf) / #(tab) escapes, or built with "..." & Param & "..."
   concatenation, is decoded into plain SQL. Text parameters
   (expression X = "value" meta [IsParameterQuery=true, ...]) are substituted
   when the option is on; other values appear as {Name}.
3. The SQL is tokenised (comments and string literals are ignored) and table
   names are collected after FROM, JOIN, comma lists in FROM,
   INSERT INTO / MERGE INTO, and USING.
4. CTE names, table functions and the FROM inside EXTRACT / TRIM / SUBSTRING
   are skipped.
5. Separately, M navigation steps are read:
   - {[Schema="..", Item=".."]}           (SQL Server style)
   - {[Name="..", Kind="Database"/"Schema"/"Table"/"View"]}  (Snowflake style)
6. If a pasted text contains no table sources at all, the whole text is
   scanned as M/SQL, so you can also paste a bare query.

Recognised connectors (shown as the "connection" of a source):
Oracle.Database, Sql.Database / Sql.Databases, Snowflake.Databases,
Odbc.Query / Odbc.DataSource, OleDb.DataSource, PostgreSQL.Database,
MySQL.Database, GoogleBigQuery.Database, Databricks.Catalogs,
AmazonRedshift.Database, Teradata.Database, DB2.Database,
Lakehouse.Contents, Fabric.Warehouse.


LIMITATIONS
-----------
- Detection is lexical, not a full SQL parser. Unusual syntax can be missed
  or over-reported. Spot-check important models in the "Per partition" view.
- It was developed and tested mainly against Oracle- and SQL-Server-style
  queries. Other dialects may need small changes (see "Customize").
- Views are listed as objects, but the tables behind them are not followed;
  the tool only sees the TMDL, not the database.
- Objects used only inside a stored procedure or function called from M are
  not visible.
- Non-SQL sources (Excel, SharePoint, web, folders, etc.) are not reported as
  database objects.
- Sources that only reference another query ("Source = SomeOtherQuery") show
  "none found"; the objects appear under the query they reference.
- Dynamic SQL built from a non-text value shows as {Name} in the object name.
- Calculated tables and calculation groups have no external source and are
  listed as such.


PRIVACY
-------
All processing happens in your browser. Your TMDL is never uploaded or
stored. Do keep in mind that the page requests two fonts from Google Fonts;
if you need zero external requests, delete the <link rel="stylesheet" ...
fonts.googleapis.com ...> line in the <head>. The tool then uses your system
fonts and everything else works the same.


CUSTOMIZE
---------
The tool is plain HTML, CSS and JavaScript with no dependencies or build
step. Open the file in any text editor. Everything is in the <script> block:

  parseTMDL        splits TMDL into tables, partitions and expressions
  stringChains /   decode M strings, #(lf)/#(tab) escapes and "&" chains
   decodeEscapes   into SQL text
  CONN_RE          list of recognised connector names; add yours here
  navObjects       M navigation patterns (Schema/Item, Name/Kind)
  sqlTokens        SQL tokenizer (comments, quoted names, bind variables)
  STOP, SKIP_FN    keywords that end a table list; functions whose FROM is
                   ignored (EXTRACT, TRIM, SUBSTRING, ...)
  extractTables    the table detection itself (FROM / JOIN / INTO / USING,
                   CTE and table-function handling)
  run              reads the options and builds the result model
  render,          output tables, per-partition view and the Excel copy
   renderParts     (change columns or labels here)

Typical tweaks
  - Add a connector:        extend the regular expression in CONN_RE
  - Treat another keyword as the end of a table list:  add it to STOP
  - Ignore more pseudo-tables (like DUAL):  extend the check in rec()
  - Change the Excel columns:  edit the "copyTsv" handler at the bottom
  - Light/dark theme follows the operating system; colours are CSS variables
    at the top of the file.


TROUBLESHOOTING
---------------
Nothing appears
  Check that the pasted text contains "table ..." blocks with "partition"
  and "source =", or an "expression ..." block. Indentation can be tabs or
  spaces. Press F12 and look for a red error in the Console; include it if
  you open an issue.

A source shows "none found"
  It uses a non-SQL connector, or references another query. Open its row in
  "Per partition" to see what the tool read.

An unexpected name appears (or one is missing)
  Open "Per partition" and read the decoded SQL with the highlighted tables.
  If the SQL is valid and the result is wrong, please open an issue with a
  small anonymised example.

Page opens as code on github.com
  GitHub shows HTML files as source. Download the file and open it locally.


CONTRIBUTING / ISSUES
---------------------
Bug reports and pull requests are welcome. For a detection problem, please
include a minimal, anonymised TMDL or SQL snippet that reproduces it
(replace real schema, table and server names). Never post real company data.


LICENSE
-------
MIT License (add a LICENSE file to the repository), provided "as is",
without warranty of any kind.
