# PostgreSQL-Monitoring-Specification
Specification for monitoring implementations for PostgreSQL Instances.

<html>
<head>

<style>
  body {
    color:#000;font: arial, sans-serif;
  }
  h2 ~ *:not(h1):not(h2):not(pre) {
    margin-left: 35px;
  }
  p, ul {
    color:#000;font: 14px;
  }
</style>

<script type="text/javascript">
window.onload = function () {
    var toc = "";
    var level = 0;

    document.getElementById("contents").innerHTML =
        document.getElementById("contents").innerHTML.replace(
            /<h([\d])>([^<]+)<\/h([\d])>/gi,
            function (str, openLevel, titleText, closeLevel) {
                if (openLevel != closeLevel) {
                    return str;
                }

                if (openLevel > level) {
                    toc += (new Array(openLevel - level + 1)).join("<ul>");
                } else if (openLevel < level) {
                    toc += (new Array(level - openLevel + 1)).join("</ul>");
                }

                level = parseInt(openLevel);

                var anchor = titleText.replace(/ /g, "_");
                toc += "<li><a href=\"#" + anchor + "\">" + titleText
                    + "</a></li>";

                return "<h" + openLevel + "><a name=\"" + anchor + "\">"
                    + titleText + "</a></h" + closeLevel + ">";
            }
        );

    if (level) {
        toc += (new Array(level + 1)).join("</ul>");
    }

    document.getElementById("toc").innerHTML += toc;
};
</script>

<!--ENTER TITLE-->
	<title>PostgreSQL Monitoring Specification - v9.4
</title>
<!--END TITLE-->
  <h2>
</head>
<body>

<!--ENTER MAIN HEADER-->
	<h1>PostgreSQL Monitoring Specification - v9.4
</h1>
<!--END MAIN HEADER-->

<!--ENTER COPYRIGHT NOTICE-->
  <h3>&copy; William Dunn (dunnwjr@gmail.com) 2018
</h3>
<!--ENTER COPYRIGHT NOTICE-->

<div id="toc">
	<h2>Table of Contents</h2>
</div>

<div id="contents">
<body>
<!--ENTER CONTENT BELOW-->



<h1>PostgreSQL Monitoring Specification v9.4</h1>

  <h2>Overview</h2>
    <p>This document provides a specification for the monitoring that should be implemented for a PostgreSQL 9.4 instance. The preferred monitoring tool of the PostgreSQL community is Bucardo's Check Postgres, but this specification provides a basis for implementation in another framework. It includes SQL queries and bash commands that should be implemented, as well as detailed explanations of how the parameters should be set and what actions should be taken when the parameter passes the threshold.</p>

  <h2>Layout of this Document</h2>
    <p>Important note: the results listed in "SQL Example Results" were not taken at the same time so they will not match up. This is the case even between the 'base' and 'Dashboard
' versions of the same monitor</p>

    <h3>Overview</h3>
      <ul><li>Commentary provided describing the monitor.</li></ul>

    <h3>Default Frequency/Thresholds</h3>
      <ul><li>Recommended thresholds for warning and critical statuses.</li></ul>

    <h3>Alert Response</h3>
      <ul><li>Suggested action that the DBA should take when that monitor triggers a warning.</li></ul>

    <h3>bash Command</h3>
      <ul><li>If the monitor uses a bash command - the command used is described here.</li></ul>

    <h3>SQL Scope</h3>
      <ul><li>If the monitor uses SQL - whether the query provides the details for all the databases in the cluster, or must be run for each database.</li></ul>

    <h3>Base Query</h3>
      <ul>
        <li>If the monitor uses SQL - the SQL query that the monitor is based on. See the 'Dashboard
 Query' section for details on the differences.</li>
        <li>I kept this copy of the base query separate from the query used by Dashboard
 so it can later extended to specific monitoring tools which may have fewer limitations listed in the Dashboard Query section.</li>
      </ul>

      <h4>SQL</h4>
        <ul>
          <li>If the monitor uses SQL - the SQL query used by the Base Query.</li>
          <li>Note this may be cluster wide OR may need be executed on each database, as specified in the 'SQL Scope'.</li>
        </ul>

      <h4>SQL Example Results</h4>
        <ul>
          <li>If the monitor uses SQL - example results of the SQL query used by the monitor.</li>
          <li>Note that the query results were not collected at the same time nor under the same conditions so they may not be consistent.</li>
          <li>There may be multiple 'SQL Example Results' sections for a query to show what results will look like </li>
        </ul>

      <h4>SQL Example Results Pretty</h4>
        <ul><li>For queries with many rows we executed the same query as 'SQL Example Results' except the results are the 'expanded table formatting mode' to make it easier to read (psql '\x' command).</li></ul>

    <h3>Dashboard Query</h3>
      <ul><li>This is based on the Base Query but for a typical Dashboard application. The base query has been modified to support the limited fields monitoring applications typically have:</li></ul>
        <ul>
          <li>Combine multiple fields into a single 'Comments' field to be used as the body of a ticket or email alert which monitoring applications typically make.</li>
          <li>The fields used by percentages in the base query were multiplied by 100 so they are a fraction of 100 rather than a fraction of 1.</li>
            <ul><li>For example, 90.2% will be displayed as .902 in the base query, 90.02 in the Dashboard
 query.</li></ul>
        </ul>

      <h4>SQL</h4>
        <ul><li>The Base Query with the modifications made for Dashboard
 described above.</li></ul>

      <h4>SQL Example Results</h4>
        <ul><li>Same as Base Query except Dashboard
 Query was executed.</li></ul>

      <h4>SQL Example Results Pretty</h4>
        <ul><li>Same as Base Query except Dashboard
 Query was executed.</li></ul>



<h1>Autovacuum</h1>

  <h2>perc_over_autovacuum_freeze_max_age/perc_until_autovacuum_freeze_max_age/perc_until_xid_wraparound</h2>

    <h3>Overview</h3>

      <p>PostgreSQL's MVCC transaction semantics depend on being able to compare transaction ID (XID) numbers: a row version with an insertion XID greater than the current transaction's XID is "in the future" and should not be visible to the current transaction. Normal XIDs are compared using modulo-2^32 arithmetic, which means that ~2^32/2-1 transactions appear in the future and ~2^32/2-1 transactions appear in the past. To prevent XID wraparound the PostgreSQL Autovacuum worker will add a marker to the XIDs of old transactions which mark them as "frozen" and cause them to always be evaluated as older than any running transaction.</p>

      <p>This behavior of autovacuum is primarily dependent on the settings autovacuum_freeze_table_age and autovacuum_freeze_max_age, which are set as database defaults but can also be specified on a per table basis (as storage parameters in CREATE TABLE or ALTER TABLE)</p>
      <ul>
          <li>When a table's oldest transaction reaches autovacuum_freeze_table_age, the next autovacuum that is performed on that table will be a vacuum freeze</li>
            <ul><li>PostgreSQL implicitly caps autovacuum_freeze_table_age at 0.95*autovacuum_freeze_max_age.</li></ul>
          <li>When a table reaches autovacuum_freeze_max_age PostgreSQL will force an autovacuum freeze on that table, even if the table would not otherwise be autovacuumed or autovacuum is disabled.</li>
            <ul><li>PostgreSQL implicitly caps autovacuum_freeze_max_age at 2 billion (2000000000)</li></ul>
      </ul>

      <p>The actual age that a wraparound occurs is ((2^32)/2)-1. When a PostgreSQL database comes within 1 million of this age (2^32/2-1-1000000) the database will go into the safety shutdown mode" and no longer accept commands, including the vacuum commands, and your only recovery option is to stop the server and use a single-user backend (where shutdown mode is not enforced) to execute VACUUM. This should, obviously, be avoided at all costs.</p>


      <p>References:</p>
        <ul>
          <li>http://www.PostgreSQL.org/docs/current/static/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND</li>
          <li>http://www.PostgreSQL.org/docs/current/static/runtime-config-client.html#GUC-VACUUM-FREEZE-TABLE-AGE</li>
          <li>http://www.PostgreSQL.org/docs/devel/static/runtime-config-autovacuum.html#GUC-AUTOVACUUM-FREEZE-MAX-AGE</li>
        </ul>

    <h3>perc_over_autovacuum_freeze_max_age</h3>
      <h4>Overview</h4>
        <p>As described above, when a table reaches autovacuum_freeze_max_age PostgreSQL will force an autovacuum freeze on that table, even if the table would not otherwise be autovacuumed or autovacuum is disabled. For each table we divide the age of the oldest transaction / the autovacuum_freeze_max_age. This number should never exceed 1 (100%) (or exceed it by much) because when 1 is reached (age of the oldest transaction = the autovacuum_freeze_max_age) an autovacuum freeze operation should be forced on this table. If this value far exceeds 1 something is wrong with autovacuum or there is something preventing the vacuum freeze operation from completing. Monitoring and addressing this is critical to ensure that transaction wraparound is being prevented by the "transaction id freeze' operation.</p>
        <p>Note that in our measurement of autovacuum_freeze_max_age we cap it at 2 billion (because PostgreSQL implicitly caps it at that value) so the value reported by the monitor may be different (2 billion) than that listed in the config of the config lists a value greater than 2 billion.</p>

      <h4>Default Frequency/Thresholds</h4>
        <ul>
          <li><b>Frequency: </b>twice a day to start with. If a customer database has a high volume of transactions maybe go more frequent</li>
          <li><b>Warning: </b>none</li>
          <li><b>Critical: </b>1.01 (101%)</li>
        </ul>

      <h4>Alert Response</h4>
        <p><b>Warning</b></p>
          <ul>
            <li>None</li>
          </ul>
        <p><b>Critical</b></p>
          <ol>
            <li>Run vacuum freeze at next slow period</li>
              <ul>
                <li>On the problem table(s)</li>
                <li>If other tables are also past autovacuum_freeze_table_age perform vacuum freeze on the full database</li>
                <li>Syntax for vaccum freeze:</li>
                  <ul>
                    <li><code>VACUUM (FREEZE, VERBOSE)</code> <b>schema.table_name</b> <code>;</code></li>
                    <li><b>Reference: </b>http://www.postgresql.org/docs/current/static/sql-vacuum.html</li>
                  </ul>
              </ul>
            <li>Check the logs for any error in the vacuum freeze operation (both the one you just ran and previously)</li>
            <li>Review vacuum settings and make sure it is running and make it more aggressive on the cluster, at least for the freeze settings</li>
              <ul>
                <li>Note: freeze settings can be set per database, but that is more difficult to train first level support on</li>
                <li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li>
                <li>To modify individual tables:</li>
                  <ul>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>SET (</code><b>autovacuum_freeze_max_age</b><code> = </code><b>value</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp SET (autovacuum_freeze_max_age=180000000);</code></li></ul>
                    </ul>
                  <li>After altering the table execute the query to check the specific settings again to confirm they changed to the desired values</li>
                  <li>If any table needs to be changed back to the server default setting for a storage_parameter use the ALTER TABLE RESET command:</li>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>RESET (</code><b>storage_parameter</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp RESET (autovacuum_freeze_max_age);</code></li></ul>
                    </ul>
                  <li><b>Alter Table Reference: </b>http://www.postgresql.org/docs/current/static/sql-altertable.html</li>
                  <li><b>Autovacuum Settings Reference: </b>http://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html</li>
                </ul>
              </ul>
          </ol>

    <h3>perc_until_autovacuum_freeze_max_age_10kRows</h3>
      <h4>Overview</h4>
        <p>As described above, when a table reaches autovacuum_freeze_max_age PostgreSQL will force an autovacuum freeze on that table, even if the table would not otherwise be autovacuumed or autovacuum is disabled. This operation acts on the table's entire visibility map so for very large tables (tables with more than 10,000 rows) this can use a lot of system resources and may impact performance, especially during peak load. For each large table (more than 10,000 rows) we divide the age of the oldest transaction / the autovacuum_freeze_max_age, and for tables smaller than 10,000 rows we return 0. Note that the number of rows is important, not the amount of data, because the visibility map size contains only high level row information such as ID and XID. Monitoring and addressing this is allows the DBA to manually perform a vacuum freeze operation on the table(s) during a slow period to prevent it from occurring during peak load.</p>
        <p>Note that in our measurement of autovacuum_freeze_max_age we cap it at 2 billion (because PostgreSQL implicitly caps it at that value) so the value reported by the monitor may be different (2 billion) than that listed in the config of the config lists a value greater than 2 billion.</p>

      <h4>Default Frequency/Thresholds</h4>
        <ul>
          <li><b>Frequency: </b>twice a day to start with. If a customer database has a high volume of transactions maybe go more frequent</li>
          <li><b>Warning: </b>none</li>
          <li><b>Critical: </b>.95 (95%)</li>
        </ul>

      <h4>Alert Response</h4>
        <p><b>Warning</b></p>
          <ul>
            <li>None</li>
          </ul>
        <p><b>Critical</b></p>
          <ul>
            <li>Run vacuum freeze at next slow period</li>
              <ul>
                <li>On the problem table(s)</li>
                <li>If other tables are also past autovacuum_freeze_table_age perform vacuum freeze on those tables or the full database</li>
                <li>Syntax for vaccum freeze:</li>
                  <ul>
                    <li><code>VACUUM (FREEZE, VERBOSE)</code> <b>schema.table_name</b> <code>;</code></li>
                    <li><b>Reference: </b>http://www.postgresql.org/docs/current/static/sql-vacuum.html</li>
                  </ul>
              </ul>
          </ul>
        <p><b>Trending</b></p>
          <ul>
            <li>If this is occurring frequently review vacuum settings and tune it to be more aggressive. It can be tuned for the entire cluster and for individual tables</li>
              <ul>
                <li>Note: freeze settings can be set per database, but that is more difficult to train first level support on</li>
                <li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li>
                <li>To modify individual tables:</li>
                  <ul>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>SET (</code><b>autovacuum_freeze_max_age</b><code> = </code><b>value</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp SET (autovacuum_freeze_max_age=180000000);</code></li></ul>
                    </ul>
                  <li>After altering the table execute the query to check the specific settings again to confirm they changed to the desired values</li>
                  <li>If any table needs to be changed back to the server default setting for a storage_parameter use the ALTER TABLE RESET command:</li>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>RESET (</code><b>storage_parameter</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp RESET (autovacuum_freeze_max_age);</code></li></ul>
                    </ul>
                  <li><b>Alter Table Reference: </b>http://www.postgresql.org/docs/current/static/sql-altertable.html</li>
                  <li><b>Autovacuum Settings Reference: </b>http://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html</li>
                </ul>
              </ul>
          </ul>


    <h3>perc_until_wraparound_server_freeze</h3>
      <h4>Overview</h4>
        <p>As described above, the actual age that a wraparound occurs is ((2^32)/2)-1. When a PostgreSQL database comes within 1 million of this age (2^32/2-1-1000000) the database will go into the safety shutdown mode" and no longer accept commands, including the vacuum commands, and your only recovery option is to stop the server and use a single-user backend (where shutdown mode is not enforced) to execute VACUUM. For each table we divide the age of the oldest transaction / (2^32/2-1-1000000). This should never occur because once autovacuum_freeze_max_age is reached a vacuum freeze should be triggered automatically (even if the table would not otherwise be autovacuumed or autovacuum is disabled) but we monitor this anyway because preventing a wraparound freeze is so critical.</p>

      <h4>Default Frequency/Thresholds</h4>
        <ul>
          <li><b>Frequency: </b>twice a day to start with. If a customer database has a high volume of transactions maybe go more frequent</li>
          <li><b>Warning: </b>.6 (60%)</li>
            <ul><li>Must be set higher if autovacuum_freeze_max_age exceeds 1200000000 because autovacuum freeze should not be forced until that value is reached</li></ul>
          <li><b>Critical: </b>.65 (65%)</li>
            <ul><li>Must be set higher if autovacuum_freeze_max_age exceeds 1200000000 because autovacuum freeze should not be forced until that value is reached</li></ul>
        </ul>

      <h4>Alert Response</h4>
        <p><b>Warning</b></p>
          <ol>
            <li>Manually run a database wide vacuum freeze at next slow period</li>
            <li>Syntax for database vaccum freeze:</li>
              <ul>
                <li><code>VACUUM FREEZE VERBOSE;</code></li>
                <li><b>Reference: </b>http://www.postgresql.org/docs/current/static/sql-vacuum.html</li>
              </ul>
            <li>Check the logs for any error in the vacuum freeze operation (both the one you just ran and previously)</li>
            <li>Review vacuum settings and make sure it is running and make it more aggressive on the cluster, at least for the freeze settings</li>
              <ul>
                <li>Note: freeze settings can be set per database, but that is more difficult to train first level support on</li>
                <li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li>
              </ul>
          </ol>
        <p><b>Critical</b></p>
          <ol>
            <li>Force a database wide vacuum as soon as can get scheduled</li>
            <li>Syntax for database vaccum freeze:</li>
              <ul>
                <li><code>VACUUM FREEZE VERBOSE;</code></li>
                <li><b>Reference: </b>http://www.postgresql.org/docs/current/static/sql-vacuum.html</li>
              </ul>
            <li>Check the logs for any error in the vacuum freeze operation (both the one you just ran and previously)</li>
            <li>Check the autovacuum_freeze_max_age settings (both in the config and for the individual tables) and decrease if it is too high</li>
              <ul>
                <li>Check database/server settings:</li>
                  <ul>
                    <li><b>autovacuum_freeze_max_age: </b><code>SELECT current_setting('autovacuum_freeze_max_age');</code></li>
                    <li><b>vacuum_freeze_table_age: </b><code>SELECT current_setting('vacuum_freeze_table_age');</code></li>
                  </ul>
                <li>To modify the server settings in the postgresql.conf and reload the config</li>
                <li>Check individual table settings:</li>
<pre>
SELECT current_catalog AS database,
       storage_settings.nspname AS schema,
       storage_settings.relname AS table,
       least(autovacuum_freeze_table_age,(0.95*(least(autovacuum_freeze_max_age, 2000000000)))) AS autovacuum_freeze_table_age,
       uses_default_autovacuum_freeze_table_age AS uses_default,
       CASE 
           WHEN autovacuum_freeze_table_age>(0.95*(least(autovacuum_freeze_max_age, 2000000000))) THEN TRUE
           ELSE FALSE
       END AS exceeds_max,
       least(autovacuum_freeze_max_age, 2000000000) AS autovacuum_freeze_max_age,
       uses_default_autovacuum_freeze_max_age AS uses_default,
       CASE
           WHEN autovacuum_freeze_max_age>2000000000 THEN TRUE
           ELSE FALSE
       END AS exceeds_max
FROM
  (SELECT oid,
          nspname,
          relname,
          CASE
              WHEN relopts LIKE '%autovacuum_freeze_table_age%' THEN CAST (regexp_replace(relopts, '.*autovacuum_freeze_table_age=([0-9.]+).*', E'\\1') AS real)
              ELSE CAST (current_setting('vacuum_freeze_table_age') AS real)
          END AS autovacuum_freeze_table_age,
          CASE
              WHEN relopts LIKE '%autovacuum_freeze_table_age%' THEN FALSE
              ELSE TRUE
          END AS uses_default_autovacuum_freeze_table_age,
          CASE
              WHEN relopts LIKE '%autovacuum_freeze_max_age%' THEN CAST (regexp_replace(relopts, '.*autovacuum_freeze_max_age=([0-9.]+).*', E'\\1') AS real)
              ELSE CAST (current_setting('autovacuum_freeze_max_age') AS real)
          END AS autovacuum_freeze_max_age,
          CASE
              WHEN relopts LIKE '%autovacuum_freeze_max_age%' THEN FALSE
              ELSE TRUE
          END AS uses_default_autovacuum_freeze_max_age
   FROM
     (SELECT pg_class.oid,
             relname,
             nspname,
             array_to_string(reloptions, '') AS relopts
      FROM pg_class
      INNER JOIN pg_namespace ns ON relnamespace = ns.oid) table_opts) storage_settings;
</pre>
                <li>To modify individual tables:</li>
                  <ul>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>SET (</code><b>autovacuum_freeze_max_age</b><code> = </code><b>value</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp SET (autovacuum_freeze_max_age=180000000);</code></li></ul>
                    </ul>
                  <li>After altering the table execute the query to check the specific settings again to confirm they changed to the desired values</li>
                  <li>If any table needs to be changed back to the server default setting for a storage_parameter use the ALTER TABLE RESET command:</li>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>RESET (</code><b>storage_parameter</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp RESET (autovacuum_freeze_max_age);</code></li></ul>
                    </ul>
                  <li><b>Alter Table Reference: </b>http://www.postgresql.org/docs/current/static/sql-altertable.html</li>
                  <li><b>Autovacuum Settings Reference: </b>http://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html</li>
                </ul>
              </ul>
            <li>Review vacuum settings and make sure it is running and make it more aggressive on the cluster, at least for the freeze settings</li>
              <ul>
                <li>Note: freeze settings can be set per database, but that is more difficult to train first level support on</li>
                <li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li>
              </ul>
          </ol>


    <h3>SQL Scope</h3>
      <p>Must be run for each database.</p>


    <h3>Base Query</h3>
        <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       storage_settings.nspname AS schema,
       storage_settings.relname AS table,
       (pg_relation_size(pg_class.oid)) AS table_bytes,
       pg_class.relpages AS pages_on_disk,
       n_live_tup+n_dead_tup AS disk_row_count,
       age(relfrozenxid) AS relfrozenxid_age,
       least(autovacuum_freeze_table_age,(0.95*(least(autovacuum_freeze_max_age, 2000000000)))) AS autovacuum_freeze_table_age,
       CAST (age(relfrozenxid) AS real) / CAST (autovacuum_freeze_table_age AS real) AS perc_until_vacuum_freeze_table_age ,
       CASE
           WHEN age(relfrozenxid) > autovacuum_freeze_table_age THEN TRUE
           ELSE FALSE
       END AS next_autovacuum_will_be_a_freeze,
       (least(autovacuum_freeze_max_age, 2000000000)) AS autovacuum_freeze_max_age,
       CAST (age(relfrozenxid) AS real) / CAST ((least(autovacuum_freeze_max_age, 2000000000)) AS real) AS perc_until_freeze_max_age,
       CASE
           WHEN n_live_tup+n_dead_tup > 10000 THEN CAST(age(relfrozenxid) AS real) / CAST((least(autovacuum_freeze_max_age, 2000000000)) AS real)
           ELSE 0
       END AS perc_until_freeze_max_age_10kRows,
       trunc(((2^32)/2)-1-1000000) AS wraparound_server_freeze_age,
       CAST (age(relfrozenxid) AS real) / CAST(trunc(((2^32)/2)-1-1000000) AS real) AS perc_until_wraparound_server_freeze
FROM pg_stat_user_tables
INNER JOIN pg_class ON pg_stat_user_tables.relid = pg_class.oid
INNER JOIN
  (SELECT oid,
          relname,
          nspname,
          CASE
              WHEN relopts LIKE '%autovacuum_freeze_table_age%' THEN CAST (regexp_replace(relopts, '.*autovacuum_freeze_table_age=([0-9.]+).*', E'\\1') AS real)
              ELSE CAST (current_setting('vacuum_freeze_table_age') AS real)
          END AS autovacuum_freeze_table_age,
          CASE
              WHEN relopts LIKE '%autovacuum_freeze_max_age%' THEN CAST (regexp_replace(relopts, '.*autovacuum_freeze_max_age=([0-9.]+).*', E'\\1') AS real)
              ELSE CAST (current_setting('autovacuum_freeze_max_age') AS real)
          END AS autovacuum_freeze_max_age
   FROM
     (SELECT pg_class.oid,
             relname,
             nspname,
             array_to_string(reloptions, '') AS relopts
      FROM pg_class
      INNER JOIN pg_namespace ns ON relnamespace = ns.oid) table_opts) storage_settings ON (pg_class.oid = storage_settings.oid)
ORDER BY storage_settings.relname;
</pre>


        <h4>SQL Example Result</h4>
<pre>
 database |  schema  |   table    | table_bytes | pages_on_disk | disk_row_count | relfrozenxid_age | autovacuum_freeze_table_age | perc_until_vacuum_freeze_table_age | next_autovacuum_will_be_a_freeze | autovacuum_freeze_max_age | perc_until_freeze_max_age | perc_until_freeze_max_age_10krows | wraparound_server_freeze_age | perc_until_wraparound_server_freeze 
----------+----------+------------+-------------+---------------+----------------+------------------+-----------------------------+------------------------------------+----------------------------------+---------------------------+---------------------------+-----------------------------------+------------------------------+-------------------------------------
 pgp      | edbstore | categories |        8192 |             1 |             16 |              712 |                   150000000 |                        4.74667e-06 | f                                |                     2e+08 |                  3.56e-06 |                                 0 |                   2146483647 |                         3.31705e-07
 pgp      | edbstore | cust_hist  |     2678784 |           327 |          60350 |              707 |                   150000000 |                        4.71333e-06 | f                                |                     2e+08 |                 3.535e-06 |                         3.535e-06 |                   2146483647 |                         3.29376e-07
 pgp      | edbstore | customers  |     3997696 |           488 |          20000 |              705 |                   150000000 |                            4.7e-06 | f                                |                     2e+08 |                 3.525e-06 |                         3.525e-06 |                   2146483647 |                         3.28444e-07
 pgp      | edbstore | dept       |        8192 |             1 |              4 |              700 |                   150000000 |                        4.66667e-06 | f                                |                     2e+08 |                   3.5e-06 |                                 0 |                   2146483647 |                         3.26115e-07
 pgp      | edbstore | emp        |        8192 |             1 |             14 |              698 |                   150000000 |                        4.65333e-06 | f                                |                     2e+08 |                  3.49e-06 |                                 0 |                   2146483647 |                         3.25183e-07
 pgp      | edbstore | inventory  |      450560 |            55 |          10000 |              696 |                   150000000 |                           4.64e-06 | f                                |                     2e+08 |                  3.48e-06 |                                 0 |                   2146483647 |                         3.24251e-07
 pgp      | edbstore | job_grd    |        8192 |             1 |              4 |              694 |                   150000000 |                        4.62667e-06 | f                                |                     2e+08 |                  3.47e-06 |                                 0 |                   2146483647 |                          3.2332e-07
 pgp      | edbstore | jobhist    |        8192 |             1 |             17 |              692 |                   150000000 |                        4.61333e-06 | f                                |                     2e+08 |                  3.46e-06 |                                 0 |                   2146483647 |                         3.22388e-07
 pgp      | edbstore | locations  |           0 |             0 |              0 |              690 |                   150000000 |                            4.6e-06 | f                                |                     2e+08 |                  3.45e-06 |                                 0 |                   2146483647 |                         3.21456e-07
 pgp      | edbstore | orderlines |     3153920 |           385 |          60350 |              686 |                   150000000 |                        4.57333e-06 | f                                |                     2e+08 |                  3.43e-06 |                          3.43e-06 |                   2146483647 |                         3.19592e-07
 pgp      | edbstore | orders     |      819200 |           100 |          12000 |              684 |                   150000000 |                           4.56e-06 | f                                |                     2e+08 |                  3.42e-06 |                          3.42e-06 |                   2146483647 |                         3.18661e-07
 pgp      | edbstore | products   |      827392 |           101 |          10000 |              679 |                   150000000 |                        4.52667e-06 | f                                |                     2e+08 |                 3.395e-06 |                                 0 |                   2146483647 |                         3.16331e-07
 pgp      | edbstore | reorder    |           0 |             0 |              0 |              674 |                   150000000 |                        4.49333e-06 | f                                |                     2e+08 |                  3.37e-06 |                                 0 |                   2146483647 |                         3.14002e-07
(13 rows)
</pre>


    <h3>Dashboard
 Query</h3>
        <h4>SQL</h4>
<pre>
SELECT base.database,
       base.schema,
       base.table,
       base.perc_until_freeze_max_age_10krows,
       'autovacuum_freeze_max_age: ' || base.autovacuum_freeze_max_age || '\n' || 'relfrozenxid_age: ' || base.relfrozenxid_age AS perc_until_freeze_max_age_10krows_details,
       base.perc_until_freeze_max_age,
       'autovacuum_freeze_max_age: ' || base.autovacuum_freeze_max_age || '\n' || 'relfrozenxid_age: ' || base.relfrozenxid_age AS perc_until_freeze_max_age_details,
       base.perc_until_wraparound_server_freeze,
       'wraparound_server_freeze_age: ' || base.wraparound_server_freeze_age || '\n' || 'relfrozenxid_age: ' || base.relfrozenxid_age AS perc_until_wraparound_server_freeze_details
FROM
(SELECT current_catalog AS database,
        storage_settings.nspname AS schema,
        storage_settings.relname AS table,
        (pg_relation_size(pg_class.oid)) AS table_bytes,
        pg_class.relpages AS pages_on_disk,
        n_live_tup+n_dead_tup AS disk_row_count,
        age(relfrozenxid) AS relfrozenxid_age,
        least(autovacuum_freeze_table_age,(0.95*(least(autovacuum_freeze_max_age, 2000000000)))) AS autovacuum_freeze_table_age,
        100.00 * (CAST (age(relfrozenxid) AS real) / CAST (autovacuum_freeze_table_age AS real)) AS perc_until_vacuum_freeze_table_age ,
        CASE
            WHEN age(relfrozenxid) > autovacuum_freeze_table_age THEN TRUE
            ELSE FALSE
        END AS next_autovacuum_will_be_a_freeze,
        least(autovacuum_freeze_max_age, 2000000000) AS autovacuum_freeze_max_age,
        100.00 * (CAST (age(relfrozenxid) AS real) / CAST ((least(autovacuum_freeze_max_age, 2000000000)) AS real)) AS perc_until_freeze_max_age,
        CASE
            WHEN n_live_tup+n_dead_tup > 10000 THEN 100.00 * (CAST(age(relfrozenxid) AS real) / CAST((least(autovacuum_freeze_max_age, 2000000000)) AS real))
            ELSE 0
        END AS perc_until_freeze_max_age_10kRows,
        trunc(((2^32)/2)-1-1000000) AS wraparound_server_freeze_age,
        100.00 * (CAST (age(relfrozenxid) AS real) / CAST(trunc(((2^32)/2)-1-1000000) AS real)) AS perc_until_wraparound_server_freeze
 FROM pg_stat_user_tables
 INNER JOIN pg_class ON pg_stat_user_tables.relid = pg_class.oid
 INNER JOIN
   (SELECT oid,
           relname,
           nspname,
           CASE
               WHEN relopts LIKE '%autovacuum_freeze_table_age%' THEN CAST (regexp_replace(relopts, '.*autovacuum_freeze_table_age=([0-9.]+).*', E'\\1') AS real)
               ELSE CAST (current_setting('vacuum_freeze_table_age') AS real)
           END AS autovacuum_freeze_table_age,
           CASE
               WHEN relopts LIKE '%autovacuum_freeze_max_age%' THEN CAST (regexp_replace(relopts, '.*autovacuum_freeze_max_age=([0-9.]+).*', E'\\1') AS real)
               ELSE CAST (current_setting('autovacuum_freeze_max_age') AS real)
           END AS autovacuum_freeze_max_age
    FROM
      (SELECT pg_class.oid,
              relname,
              nspname,
              array_to_string(reloptions, '') AS relopts
       FROM pg_class
       INNER JOIN pg_namespace ns ON relnamespace = ns.oid) table_opts) storage_settings ON (pg_class.oid = storage_settings.oid)
 ORDER BY storage_settings.relname) base;
</pre>

        <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    | perc_until_freeze_max_age_10krows |        perc_until_freeze_max_age_10krows_details        | perc_until_freeze_max_age |            perc_until_freeze_max_age_details            | perc_until_wraparound_server_freeze |           perc_until_wraparound_server_freeze_details           
----------+----------+------------+-----------------------------------+---------------------------------------------------------+---------------------------+---------------------------------------------------------+-------------------------------------+-----------------------------------------------------------------
 pgp      | edbstore | categories |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 712 |                  3.56e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 712 |                         3.31705e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 712
 pgp      | edbstore | cust_hist  |                         3.535e-06 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 707 |                 3.535e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 707 |                         3.29376e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 707
 pgp      | edbstore | customers  |                         3.525e-06 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 705 |                 3.525e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 705 |                         3.28444e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 705
 pgp      | edbstore | dept       |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 700 |                   3.5e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 700 |                         3.26115e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 700
 pgp      | edbstore | emp        |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 698 |                  3.49e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 698 |                         3.25183e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 698
 pgp      | edbstore | inventory  |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 696 |                  3.48e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 696 |                         3.24251e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 696
 pgp      | edbstore | job_grd    |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 694 |                  3.47e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 694 |                          3.2332e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 694
 pgp      | edbstore | jobhist    |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 692 |                  3.46e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 692 |                         3.22388e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 692
 pgp      | edbstore | locations  |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 690 |                  3.45e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 690 |                         3.21456e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 690
 pgp      | edbstore | orderlines |                          3.43e-06 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 686 |                  3.43e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 686 |                         3.19592e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 686
 pgp      | edbstore | orders     |                          3.42e-06 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 684 |                  3.42e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 684 |                         3.18661e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 684
 pgp      | edbstore | products   |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 679 |                 3.395e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 679 |                         3.16331e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 679
 pgp      | edbstore | reorder    |                                 0 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 674 |                  3.37e-04 | autovacuum_freeze_max_age: 2e+08\nrelfrozenxid_age: 674 |                         3.14002e-05 | wraparound_server_freeze_age: 2146483647\nrelfrozenxid_age: 674
(13 rows)
</pre>


  <h2>user_table_bloat</h2>

    <h3>Overview</h3>
      <p>The PostgreSQL autovacuum demon performs several critical functions</p>
        <ul>
          <li>It removes "dead" updated/deleted rows which are no longer needed for MVCC (and thus useless). Rows that are no longer needed for MVCC but remain in the table are referred to as "bloat"</li>
            <ul>
              <li>All the dead rows must be scanned during queries even though they are no longer needed, so removing them decreases the processing impact of having them remain there</li>
              <li>All dead rows remain on disk, so removing them allows disk space to be recovered or reused</li>
            </ul>
          <li>It updates data statistics used by the PostgreSQL query planner</li>
          <li>It updates the visibility map, which speeds up index-only scans</li>
          <li>It freezes old transaction ids to avoid transaction ID wraparound or multixact ID wraparound (as described and monitored by perc_over_autovacuum_freeze_max_age/perc_until_autovacuum_freeze_max_age/perc_until_xid_wraparound)</li>
        </ul>
      <p>To measure the performance of autovacuum to ensure that it is running and tuned to be aggressive enough we calculate the "bloat" for each table. If autovacuum is maintaining healthy levels of bloat it is safe to conclude that it is also maintaining statistics and the visibility map as well. To estimate table bloat for each table we divide the (live rows + dead rows)/(live rows)-1. Autovacuum will not vacuum a table until there are at least 'autovacuum_vacuum_threshold' dead rows as specified in postgresql.conf (the default is 50), so we put a threshold that the table must be at least autovacuum_vacuum_threshold*10 in size which will make us have no false positives above 0.1 (10%) bloat.</p>

      <p>References:</p>
        <ul>
          <li>http://www.postgresql.org/docs/devel/static/routine-vacuuming.html</li>
          <li>http://www.PostgreSQL.org/docs/devel/static/runtime-config-autovacuum.html</li>
        </ul>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>once a day, at the end of slow period (~7 PM)</li>
        <li><b>Warning: </b>.15 (15%)</li>
        <li><b>Critical: </b>.2 (20%)</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ol>
          <li>Run vacuum against the table(s) at next slow period</li>
            <ul>      
              <li>Syntax for vaccum:</li>
                <ul>
                  <li><code>VACUUM (VERBOSE)</code> <b>schema.table_name</b> <code>;</code></li>
                  <li><b>Reference: </b>http://www.postgresql.org/docs/current/static/sql-vacuum.html</li>
                </ul>
            </ul>
          <li>If there are other tables nearing warning level run a vacuum against them as well</li>
        </ol>
      <p><b>Critical</b></p>
        <ol>
          <li>Run vacuum against the table(s) immediately</li>
            <ul>      
              <li>Syntax for vaccum:</li>
                <ul>
                  <li><code>VACUUM (VERBOSE)</code> <b>schema.table_name</b> <code>;</code></li>
                  <li><b>Reference: </b>http://www.postgresql.org/docs/current/static/sql-vacuum.html</li>
                </ul>
            </ul>
          <li>If there are many tables reaching warning/critical tune the server autovacuum settings to make autovacuum more aggressive. </li>
            <ul><li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li></ul>
          <li>If there is one or two tables reaching warning/critical set specific autovacuum settings on the table(s) to make it more aggressive</li>
                <ul>
                  <li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li>
                  <li>The following query will show you table specific settings (if there are any):</li>
<pre>
SELECT pg_class.oid,
       nspname AS schema,
       relname AS table,
       array_to_string(reloptions, '') AS custom_settings
FROM pg_class
INNER JOIN pg_namespace ON (pg_namespace.oid=pg_class.relnamespace)
WHERE nspname='[SCHEMA NAME]'
  AND relname='[TABLE NAME]';
</pre>
                    <ul>
                      <li>Replace <code>[SCHEMA NAME]</code> with the name of the schema the table is in</li>
                      <li>Replace <code>[TABLE NAME]</code> with the name of the table</li>
                    </ul>
                  <li>Syntax for changing settings for an individual table</li>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>SET (</code><b>storage_parameter</b><code> = </code><b>value</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp SET (autovacuum_vacuum_scale_factor=.5);</code></li></ul>
                    </ul>
                  <li>After altering the table execute the query to check the specific settings again to confirm they changed to the desired values</li>
                  <li>If any table needs to be changed back to the server default setting for a storage_parameter use the ALTER TABLE RESET command:</li>
                    <ul>
                      <li><code>ALTER TABLE</code> <b>schema.table</b> <code>RESET (</code><b>storage_parameter</b><code>);</code></li>
                        <ul><li>For example: <code>ALTER TABLE edbstore.emp RESET (autovacuum_vacuum_scale_factor);</code></li></ul>
                    </ul>
                  <li><b>Alter Table Reference: </b>http://www.postgresql.org/docs/current/static/sql-altertable.html</li>
                  <li><b>Autovacuum Settings Reference: </b>http://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html</li>
                </ul>
        </ol> 
      <p><b>Trending</b></p>
        <ol>
          <li>If warning is triggered frequently for many tables tune the server-wide autovacuum settings in the config to make autovacuum more aggressive and ensure no table has individual storage parameters less aggressive than those values</li>
            <ul><li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li></ul>
          <li>If warning triggered frequently on on or a few tables set specific autovacuum settings on the table(s) (ALTER TABLE SET ( storage_parameter = value [, ... ] )) to make it more aggressive</li>
            <ul>
              <li>See the 'autovacuum' section of guide "PostgreSQL Server Tuning" for instructions</li>
              <li>The following query will show you table specific settings (if there are any):</li>
<pre>
SELECT pg_class.oid,
   nspname AS schema,
   relname AS table,
   array_to_string(reloptions, '') AS custom_settings
FROM pg_class
INNER JOIN pg_namespace ON (pg_namespace.oid=pg_class.relnamespace)
WHERE nspname='[SCHEMA NAME]'
AND relname='[TABLE NAME]';
</pre>
                <ul>
                  <li>Replace <code>[SCHEMA NAME]</code> with the name of the schema the table is in</li>
                  <li>Replace <code>[TABLE NAME]</code> with the name of the table</li>
                </ul>
              <li>Syntax for changing settings for an individual table</li>
                <ul>
                  <li><code>ALTER TABLE</code> <b>schema.table</b> <code>SET (</code><b>storage_parameter</b><code> = </code><b>value</b><code>);</code></li>
                    <ul><li>For example: <code>ALTER TABLE edbstore.emp SET (autovacuum_vacuum_scale_factor=.5);</code></li></ul>
                </ul>
              <li>After altering the table execute the query to check the specific settings again to confirm they changed to the desired values</li>
              <li>If any table needs to be changed back to the server default setting for a storage_parameter use the ALTER TABLE RESET command:</li>
                <ul>
                  <li><code>ALTER TABLE</code> <b>schema.table</b> <code>RESET (</code><b>storage_parameter</b><code>);</code></li>
                    <ul><li>For example: <code>ALTER TABLE edbstore.emp RESET (autovacuum_vacuum_scale_factor);</code></li></ul>
                </ul>
              <li><b>Alter Table Reference: </b>http://www.postgresql.org/docs/current/static/sql-altertable.html</li>
              <li><b>Autovacuum Settings Reference: </b>http://www.postgresql.org/docs/current/static/runtime-config-autovacuum.html</li>
            </ul>
        </ol>
      <p><b>Health Check: </b></p>
        <ul>
          <li>If we are regularly monitoring the server and the above Warning/Critical alert responses are followed the level of bloat will never become overwhelming. However, a server that is not monitored or administrated by us may have an extremely high level of bloat or autovacuum might be off altogether. In these cases you may need to take an outage or wait for the next scheduled outage and run a vacuum of all tables (full database vacuum)</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>Must be run once per database.</p>

    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
/*
if you remove the WHERE clause you will need to add logic to the bloat
calculation to avoid division by 0 error that would occur when n_live_tup=0.
For example: 
    CASE WHEN n_live_tup>0 THEN ((n_live_tup::float+n_dead_tup)/n_live_tup)-1
    ELSE 0 END AS bloat 
*/
SELECT current_catalog AS database,
       schemaname AS schema,
       relname AS table,
       (pg_relation_size(relid)) AS table_bytes,
       n_live_tup,
       n_dead_tup,
       ((CAST(n_live_tup AS real) + n_dead_tup)/n_live_tup)-1 AS bloat
FROM pg_stat_user_tables
WHERE n_live_tup>(CAST(current_setting('autovacuum_vacuum_threshold') AS bigint)*10+1);
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    | table_bytes | n_live_tup | n_dead_tup |      bloat       
----------+----------+------------+-------------+------------+------------+------------------
 pgp      | edbstore | orders     |      819200 |      12000 |          0 |                0
 pgp      | edbstore | products   |      827392 |      10000 |          0 |                0
 pgp      | edbstore | customers  |     3997696 |      20000 |          0 |                0
 pgp      | edbstore | inventory  |      540672 |      10000 |       2102 |           0.2102
 pgp      | edbstore | cust_hist  |     2678784 |      60350 |          0 |                0
 pgp      | edbstore | orderlines |     3153920 |      60350 |          0 |                0
 pgp      |edbstore  | emp        |      450560 |      14000 |       5000 | 0.35714285714286
(7 rows)
</pre>


      <h4>SQL Example After Vacuum</h4>
<pre>
 database |  schema  |   table    | table_bytes | n_live_tup | n_dead_tup | bloat 
----------+----------+------------+-------------+------------+------------+-------
 pgp      | edbstore | orders     |      819200 |      12000 |          0 |     0
 pgp      | edbstore | products   |      827392 |      10000 |          0 |     0
 pgp      | edbstore | customers  |     3997696 |      20000 |          0 |     0
 pgp      | edbstore | inventory  |      540672 |      10000 |          0 |     0
 pgp      | edbstore | cust_hist  |     2678784 |      60350 |          0 |     0
 pgp      | edbstore | orderlines |     3153920 |      60350 |          0 |     0
 pgp      |edbstore  | emp        |      450560 |      14000 |          0 |     0
(7 rows)
</pre>

    <h3>Dashboard
 Query</h3>
        <h4>SQL</h4>
<pre>
SELECT base.database,
       base.schema,
       base.table,
       base.bloat,
       'table_bytes: ' || base.table_bytes || '\n' || 'n_live_tup: ' || base.n_live_tup || '\n' || 'n_dead_tup: ' || base.n_dead_tup AS details
FROM
  (SELECT current_catalog AS database,
          schemaname AS schema,
          relname AS table,
          (pg_relation_size(relid)) AS table_bytes,
          n_live_tup,
          n_dead_tup,
          100.00 * (((CAST(n_live_tup AS real) + n_dead_tup)/n_live_tup)-1) AS bloat
   FROM pg_stat_user_tables
   WHERE n_live_tup>(CAST(current_setting('autovacuum_vacuum_threshold') AS bigint)*10+1)) base;
</pre>


        <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    |      bloat       |                          details                          
----------+----------+------------+------------------+-----------------------------------------------------------
 pgp      | edbstore | orders     |                0 | table_bytes: 819200\nn_live_tup: 12000\nn_dead_tup: 0
 pgp      | edbstore | products   |                0 | table_bytes: 827392\nn_live_tup: 10000\nn_dead_tup: 0
 pgp      | edbstore | customers  |                0 | table_bytes: 3997696\nn_live_tup: 20000\nn_dead_tup: 0
 pgp      | edbstore | inventory  |                0 | table_bytes: 450560\nn_live_tup: 10000\nn_dead_tup: 0
 pgp      | edbstore | cust_hist  | 6.30206957287538 | table_bytes: 2678784\nn_live_tup: 56775\nn_dead_tup: 3578
 pgp      | edbstore | orderlines |                0 | table_bytes: 3153920\nn_live_tup: 60350\nn_dead_tup: 0
(6 rows)
</pre>


        <h4>SQL Example After Vacuum</h4>
<pre>
 database |  schema  |   table    | bloat |                        details                         
----------+----------+------------+-------+--------------------------------------------------------
 pgp      | edbstore | orders     |     0 | table_bytes: 819200\nn_live_tup: 12000\nn_dead_tup: 0
 pgp      | edbstore | products   |     0 | table_bytes: 827392\nn_live_tup: 10000\nn_dead_tup: 0
 pgp      | edbstore | customers  |     0 | table_bytes: 3997696\nn_live_tup: 20000\nn_dead_tup: 0
 pgp      | edbstore | inventory  |     0 | table_bytes: 450560\nn_live_tup: 10000\nn_dead_tup: 0
 pgp      | edbstore | cust_hist  |     0 | table_bytes: 2678784\nn_live_tup: 56789\nn_dead_tup: 0
 pgp      | edbstore | orderlines |     0 | table_bytes: 3153920\nn_live_tup: 60350\nn_dead_tup: 0
(6 rows)
</pre>



<h1>General</h1>

  <h2>blocked_transactions</h2>

    <h3>Overview</h3>
      <p>This monitor reports the number of sessions that are waiting on locks held by other sessions. It gives you a general sense of how much lockup the database is experiencing but is not directly actionable. This would normally be catagorized as "Information Only" but some customers like to receive an email notification when a certain number of blocked sessions is reached.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>as needed to support warning/critical logic</li>
        <li><b>Warning: </b>as specified by customer at on-boarding</li>
        <li><b>Critical: </b>none</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>As specified by customer at on-boarding either do nothing or send an email notifications to customer</li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>None</li>
        </ul>

    <h3>SQL Scope</h3>
      <p>Run once per cluster.</p>

    <h3>Base Query</h3>

        <h4>SQL Compatibility</h4>
          <ul>
            <li>In 9.1 pg_stat_activity.pid must be renamed to pg_stat_activity.procpid</li>
            <li>In 9.1 pg_stat_activity.state must be removed</li>
          </ul>

        <h4>SQL</h4>
<pre>
SELECT pg_stat_activity.datname AS database,
       pg_stat_activity.pid AS waiting_pid,
       pg_stat_activity.usename AS waiting_username,
       pg_stat_activity.backend_start AS waiting_login,
       left(pg_stat_activity.application_name, 200) AS waiting_application_name,
       pg_stat_activity.client_addr AS waiting_client_addr,
       left(pg_stat_activity.client_hostname, 200) AS waiting_client_hostname,
       pg_stat_activity.client_port AS waiting_client_port,
       pg_stat_activity.query_start AS waiting_query_start,
       pg_stat_activity.state AS waiting_transaction_state,
       now()-pg_stat_activity.query_start AS waiting_query_runtime,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_activity.query_start))) AS waiting_query_runtime_seconds,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_activity.query_start)))/60 AS waiting_query_runtime_minutes
FROM pg_stat_activity
WHERE pg_stat_activity.waiting = TRUE;
</pre>

        <h4>SQL Example Results</h4>
<pre>
 database | waiting_pid | waiting_username |         waiting_login         | waiting_application_name | waiting_client_addr | waiting_client_hostname | waiting_client_port |      waiting_query_start      | waiting_transaction_state | waiting_query_runtime | waiting_query_runtime_seconds | waiting_query_runtime_minutes 
----------+-------------+------------------+-------------------------------+--------------------------+---------------------+-------------------------+---------------------+-------------------------------+---------------------------+-----------------------+-------------------------------+-------------------------------
 pgp      |        8731 | pgp              | 2015-06-05 10:03:07.528725-07 | psql                     | ::1                 |                         |               41154 | 2015-06-09 15:49:12.167473-07 | active                    | 00:01:58.758353       |                           119 |              1.98333333333333
 pgp      |        9031 | pgp              | 2015-06-05 11:06:15.305611-07 | psql                     | ::1                 |                         |               41158 | 2015-06-09 15:50:41.503815-07 | active                    | 00:00:29.422011       |                            29 |             0.483333333333333
(2 rows)
</pre>


        <h4>SQL Example Results Pretty</h4>
<pre>
-[ RECORD 1 ]-----------------+------------------------------
database                      | pgp
waiting_pid                   | 8731
waiting_username              | pgp
waiting_login                 | 2015-06-05 10:03:07.528725-07
waiting_application_name      | psql
waiting_client_addr           | ::1
waiting_client_hostname       | 
waiting_client_port           | 41154
waiting_query_start           | 2015-06-09 15:49:12.167473-07
waiting_transaction_state     | active
waiting_query_runtime         | 00:02:27.438982
waiting_query_runtime_seconds | 147
waiting_query_runtime_minutes | 2.45
-[ RECORD 2 ]-----------------+------------------------------
database                      | pgp
waiting_pid                   | 9031
waiting_username              | pgp
waiting_login                 | 2015-06-05 11:06:15.305611-07
waiting_application_name      | psql
waiting_client_addr           | ::1
waiting_client_hostname       | 
waiting_client_port           | 41158
waiting_query_start           | 2015-06-09 15:50:41.503815-07
waiting_transaction_state     | active
waiting_query_runtime         | 00:00:58.10264
waiting_query_runtime_seconds | 58
waiting_query_runtime_minutes | 0.966666666666667
</pre>


    <h3>Dashboard
 Query</h3>

        <h4>SQL Compatibility</h4>
          <ul>
            <li>In 9.1 pg_stat_activity.pid must be renamed to pg_stat_activity.procpid</li>
            <li>In 9.1 pg_stat_activity.state must be removed</li>
              <ul><li>pg_stat_activity.state is not used anyway</li></ul>
          </ul>

        <h4>SQL</h4>
<pre>
SELECT database,
       count_blocked,
       average_runtime_minutes,
       max_runtime_minutes,
       average_runtime_seconds,
       max_runtime_seconds,
       'count_blocked: ' || count_blocked || '\n' || 'average_runtime_minutes: ' || average_runtime_minutes || '\n' || 
       'max_runtime_minutes: ' || max_runtime_minutes || '\n' || 'average_runtime_seconds: ' || average_runtime_seconds || '\n' || 
       'max_runtime_seconds: ' || max_runtime_seconds AS details
FROM
  (SELECT database,
          count(base.waiting_pid) AS count_blocked,
          avg(base.waiting_query_runtime_minutes) AS average_runtime_minutes,
          max(base.waiting_query_runtime_minutes) AS max_runtime_minutes,
          avg(base.waiting_query_runtime_seconds) AS average_runtime_seconds,
          max(base.waiting_query_runtime_seconds) AS max_runtime_seconds
   FROM
     (SELECT pg_stat_activity.datname AS database,
             pg_stat_activity.pid AS waiting_pid,
             pg_stat_activity.usename AS waiting_username,
             pg_stat_activity.backend_start AS waiting_login,
             left(pg_stat_activity.application_name, 200) AS waiting_application_name,
             pg_stat_activity.client_addr AS waiting_client_addr,
             left(pg_stat_activity.client_hostname, 200) AS waiting_client_hostname,
             pg_stat_activity.client_port AS waiting_client_port,
             pg_stat_activity.query_start AS waiting_query_start,
             pg_stat_activity.state AS waiting_transaction_state,
             now()-pg_stat_activity.query_start AS waiting_query_runtime,
             ROUND(EXTRACT(EPOCH
                           FROM (now())) - EXTRACT(EPOCH
                                                   FROM (pg_stat_activity.query_start))) AS waiting_query_runtime_seconds,
             ROUND(EXTRACT(EPOCH
                           FROM (now())) - EXTRACT(EPOCH
                                                   FROM (pg_stat_activity.query_start)))/60 AS waiting_query_runtime_minutes
      FROM pg_stat_activity
      WHERE pg_stat_activity.waiting = TRUE) base
   GROUP BY database) top
UNION ALL
SELECT 'PLACEHOLDER' AS database,
       0 AS count_blocked,
       0 AS average_runtime_minutes,
       0 AS max_runtime_minutes,
       0 AS average_runtime_seconds,
       0 AS max_runtime_seconds,
       NULL AS description;
</pre>

        <h4>SQL Example Results</h4>
<pre>
  database   | count_blocked | average_runtime_minutes | max_runtime_minutes | average_runtime_seconds | max_runtime_seconds |                                                                           details                                                                            
-------------+---------------+-------------------------+---------------------+-------------------------+---------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------
 postgres    |             2 |       0.258333333333333 |   0.433333333333333 |                    15.5 |                  26 | count_blocked: 2\naverage_runtime_minutes: 0.258333333333333\nmax_runtime_minutes: 0.433333333333333\naverage_runtime_seconds: 15.5\nmax_runtime_seconds: 26
 PLACEHOLDER |             0 |                       0 |                   0 |                       0 |                   0 | 
(2 rows)
</pre>


  <h2>blocking_idle_in_transaction/idle_in_transaction</h2>

    <h3>Overview</h3>
      <p>As of now sessions in PostgreSQL can be in one of the following five states (note that in PostgreSQL a "session"/"connection" is frequently referred to as a "backend"): </p>
        <ul>
          <li><b>active: </b>the backend is executing a query</li>
          <li><b>idle: </b>the backend is waiting for a new client command</li>
          <li><b>idle in transaction: </b>the backend is in a transaction, but is not currently executing a query</li>
          <li><b>idle in transaction (aborted): </b>this state is similar to idle in transaction, except one of the statements in the transaction caused an error</li>
          <li><b>fastpath function call: </b>the backend is executing a fast-path function</li>
          <li><b>disabled: </b>this state is reported if track_activities is disabled in this backend</li>
            <ul><li>Note: this is not an actual state of a transaction. The transaction is actually in one of the other states but they are not being tracked by the view we are querying</li></ul>
        </ul>
      <p>Aside from using system resources sessions in most of these states do not cause any issue for PostgreSQL, but"idle in transaction" can cause potentially severe problems:</p>
        <ul>
          <li>The session that is idle in transaction can be holding a lock on a resource needed by another transaction, blocking the other transaction from running to completion</li>
            <ul><li>This acts very similar to a deadlock, but the system does not automatically resolve the issue (deadlocks automatically rollback and restart)</li></ul>
          <li>In MVCC, rows are not considered "dead" and eligiable to be vacuumed until there is no transaction older than it. So a transaction that has started but left idle prevents all deleted or old versions of updated rows to be retained until that transaction ends. This can cause a considerable amount of bloat in the database which leads to potentially serious performance problems</li>
        </ul>
      <p>The only way to resolve these issues is for the transaction that is idle to be resumed by the user, committed/rolledback by the user, or forcefully rolled back by a superuser</p>


    <h3>blocking_idle_in_transaction</h3>

      <h4>Overview</h4>
        <p>As described above, transactions that are in state 'idle in transaction' can hold a lock on a resource needed by another transaction, blocking the other transaction from running to completion. This acts very similar to a deadlock, but the system does not automatically resolve the issue (deadlocks automatically rollback and restart). The only way to resolve these issues is for the transaction that is idle to be resumed by the user, committed/rolledback by the user, or forcefully rolled back by a superuser. In this monitor we identify all the sessions that are in state "idle in transaction", measure how long each has been in state "idle in transaction", and how many other transactions each is blocking (0 if they are not blocking other transactions), and the maximum amount of time that it has been blocking another transaction. This gives us the ability to identify when this problem arises either ask the user to resume, commit, or rollback their transaction OR forcefull roll it back. Note that it is also possible for a automated cron job to be run periodically which identifies such transactions (via the below query) and forcefully rolls them back.</p>

      <h4>Default Frequency/Thresholds</h4>
        <ul>
          <li><b>Frequency: </b>as needed to support warning/critical logic</li>
          <li><b>Warning: </b>as specified by customer at onboarding, recommend blocking any transaction for longer than 30 minutes</li>
            <li>Note: Dashboard
 can only support using one parameter as a threshold, so use either the amount of time it has been blocking another transaction (which will be 0 if no transaction is blocked) rather than some combination of time idle in transaction and number of transactions blocked. We could theoretically use a combination of the number of transactions that are being blocked and modify the the frequency of the check, but usually we would want to make sure no transactions are being blocked so being warned if any transaction is blocked</li>
          <li><b>Critical: </b>as specified by customer at onboarding</li>
        </ul>

      <h4>Alert Response</h4>
        <p><b>Warning</b></p>
          <ul>
            <li>As specified by customer at onboarding either notify customer, kill the transaction, or notify customer asking if transaction should be killed</li>
            <ul><li>You can force a rollback by executing 'SELECT pg_cancel_backend(holding_pid);'. pg_terminate_backend() will disconnect the user.</li></ul>
          </ul>
        <p><b>Critical</b></p>
          <ul>
            <li>As specified by customer at onboarding either notify customer, kill the transaction, or notify customer asking if transaction should be killed</li>
            <ul><li>You can force a rollback by executing 'SELECT pg_cancel_backend(holding_pid);'. pg_terminate_backend() will disconnect the user.</li></ul>
          </ul> 
        <p><b>Trending</b></p>
          <ul>
            <li>If this happens frequently you should notify the client and ask their application developers to revise their code to avoid leaving transactions idle</li>
            <li>A cron job can be created to, on a schedule, automatically kill transactions that have been 'idle in transaction' for a certain amount of time and blocking a certain number of transactions</li>
          </ul>


    <h3>idle_in_transaction</h3>
      
      <h4>Overview</h4>
        <p>As described above, transactions that are in state "idle in transaction" cause problems for MVCC - rows are not considered "dead" and eligiable to be vacuumed until there is no transaction older than it, so a transaction that has started but left idle prevents all deleted or old versions of updated rows to be retained until that transaction ends. This can cause a considerable amount of bloat in the database which leads to potentially serious performance problems. In this monitor we identify all the sessions that are in state "idle in transaction" and report on how long each has been idle.</p>

      <h4>Default Frequency/Thresholds</h4>
        <ul>
          <li><b>Frequency: </b>as needed to support warning/critical logic, but not higher than 10 minutes. EnterpriseDB's PEM (PostgreSQL Enterprise Monitor) checks this every minute</li>
          <li><b>Warning: </b>as specified by customer at onboarding, recommend 30 minutes</li>
          <li><b>Critical: </b>as specified by customer at onboarding</li>
        </ul>

      <h4>Alert Response</h4>
        <p><b>Warning</b></p>
          <ul>
            <li>As specified by customer at onboarding either notify customer, kill the transaction, or notify customer asking if transaction should be killed</li>
              <ul><li>You can force a rollback by executing 'SELECT pg_cancel_backend(holding_pid);'. pg_terminate_backend() will disconnect the user.</li></ul>
          </ul>
        <p><b>Critical</b></p>
          <ul>
            <li>As specified by customer at onboarding either notify customer, kill the transaction, or notify customer asking if transaction should be killed</li>
            <ul><li>You can force a rollback by executing 'SELECT pg_cancel_backend(holding_pid);'. pg_terminate_backend() will disconnect the user.</li></ul>
          </ul> 
        <p><b>Trending</b></p>
          <ul>
            <li>If this happens frequently you should notify the client and ask their application developers to revise their code to avoid leaving transactions idle</li>
            <li>A cron job can be created to, on a schedule, automatically kill transactions that have been 'idle in transaction' for a certain amount of time</li>
          </ul>


    <h3>Alert Response SQL</h3>
    <p>TODO: Revise and add SQL to be run by DBA</p>

      <h4>SQL to Kill Blocking Idle Transactions Idle Longer than 2 Hours (7200 seconds)</h4>
<pre>
-- Kill blocking idle transactions running longer than 7200 seconds (2 hours)
SELECT pg_cancel_backend(holding.pid)
FROM pg_catalog.pg_locks AS waiting
JOIN pg_catalog.pg_stat_activity AS waiting_stm ON (waiting_stm.pid = waiting.pid)
JOIN pg_catalog.pg_locks AS holding ON ((waiting."database" = holding."database"
                                         AND waiting.relation = holding.relation)
                                        OR waiting.transactionid = holding.transactionid)
JOIN pg_catalog.pg_stat_activity AS holding_stm ON (holding_stm.pid = holding.pid)
WHERE NOT waiting.granted
  AND holding.granted
  AND waiting.pid <> holding.pid
  AND holding_stm.state = 'idle in transaction'
  AND (ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (holding_stm.state_change))))>7200;
</pre>

      <h4>SQL to Kill Idle Transactions Idle Longer than 2 Hours (7200 seconds)</h4>
<pre>

</pre>

    <h3>SQL Scope</h3>
    <p>Run once per cluster.</p>

    <h3>SQL Compatibility</h3>
      <ul>
        <li></li>
      </ul>

    <h3>Base Query</h3>
        <h4>SQL</h4>
<pre>
SELECT pg_stat_activity.datname AS database,
       -- transaction info from pg_stat_activity:
       pg_stat_activity.pid,
       pg_stat_activity.usename,
       pg_stat_activity.backend_start AS login,
       left(pg_stat_activity.application_name, 200) AS application_name,
       pg_stat_activity.client_addr,
       left(pg_stat_activity.client_hostname, 200) AS client_hostname,
       pg_stat_activity.client_port,
       pg_stat_activity.xact_start AS transaction_start,
       pg_stat_activity.state AS transaction_state,
       1 AS idle_in_transaction,
       pg_stat_activity.state_change AS transaction_state_since,
       now()-pg_stat_activity.state_change AS time_in_state,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_activity.state_change))) AS seconds_in_state,
       (ROUND(EXTRACT(EPOCH
                      FROM (now())) - EXTRACT(EPOCH
                                              FROM (pg_stat_activity.state_change))))/60 AS minutes_in_state,
       -- if the transaction is idle_in_transaction and blocking other transactions from completing
       count(blocking.waiting_pid) OVER (PARTITION BY pg_stat_activity.pid) AS blocking_count,
       COALESCE((max(blocking.waiting_query_runtime_seconds) OVER (PARTITION BY pg_stat_activity.pid)),0) AS max_blocking_time_seconds,
       COALESCE((max(blocking.waiting_query_runtime_minutes) OVER (PARTITION BY pg_stat_activity.pid)),0) AS max_blocking_time_minutes,
       blocking.locktype,
       blocking.holding_xid,
       blocking.waiting_pid,
       blocking.waiting_usename,
       blocking.waiting_login,
       blocking.waiting_application_name,
       blocking.waiting_client_addr,
       blocking.waiting_client_hostname,
       blocking.waiting_client_port,
       blocking.waiting_query_start,
       blocking.waiting_transaction_state,
       blocking.waiting_query_runtime,
       blocking.waiting_query_runtime_seconds,
       blocking.waiting_query_runtime_minutes,
       blocking.waiting_locktype,
       blocking.waiting_xid
FROM pg_stat_activity
LEFT OUTER JOIN
  (SELECT holding.pid AS pid,
          -- holding transaction info from pg_lock:
          holding.locktype AS locktype,
          holding.transactionid AS holding_xid,
          -- waiting transaction info from pg_stat_activity:
          waiting.pid AS waiting_pid,
          waiting_stm.usename AS waiting_usename,
          waiting_stm.backend_start AS waiting_login,
          left(waiting_stm.application_name, 200) AS waiting_application_name,
          waiting_stm.client_addr AS waiting_client_addr,
          left(waiting_stm.client_hostname, 200) AS waiting_client_hostname,
          waiting_stm.client_port AS waiting_client_port,
          waiting_stm.query_start AS waiting_query_start,
          waiting_stm.state AS waiting_transaction_state,
          now()-waiting_stm.query_start AS waiting_query_runtime,
          ROUND(EXTRACT(EPOCH
                        FROM (now())) - EXTRACT(EPOCH
                                                FROM (waiting_stm.query_start))) AS waiting_query_runtime_seconds,
          ROUND(EXTRACT(EPOCH
                        FROM (now())) - EXTRACT(EPOCH
                                                FROM (waiting_stm.query_start)))/60 AS waiting_query_runtime_minutes,
          -- waiting transaction info from pg_lock:
          waiting.locktype AS waiting_locktype,
          waiting.transactionid AS waiting_xid
   FROM pg_catalog.pg_locks AS waiting
   JOIN pg_catalog.pg_stat_activity AS waiting_stm ON (waiting_stm.pid = waiting.pid)
   JOIN pg_catalog.pg_locks AS holding ON ((waiting."database" = holding."database"
                                            AND waiting.relation = holding.relation)
                                           OR waiting.transactionid = holding.transactionid)
   JOIN pg_catalog.pg_stat_activity AS holding_stm ON (holding_stm.pid = holding.pid)
   WHERE NOT waiting.granted
     AND holding.granted
     AND waiting.pid <> holding.pid
     AND holding_stm.state = 'idle in transaction') blocking ON (blocking.pid = pg_stat_activity.pid)
WHERE pg_stat_activity.state = 'idle in transaction';
</pre>

        <h4>SQL Example Results</h4>
<pre>
 database |  pid  | usename |             login             | application_name | client_addr | client_hostname | client_port |       transaction_start       |  transaction_state  | idle_in_transaction |    transaction_state_since    |  time_in_state  | seconds_in_state | minutes_in_state | blocking_count | max_blocking_time_seconds | max_blocking_time_minutes |   locktype    | holding_xid | waiting_pid | waiting_usename |         waiting_login         | waiting_application_name | waiting_client_addr | waiting_client_hostname | waiting_client_port |      waiting_query_start      | waiting_transaction_state | waiting_query_runtime | waiting_query_runtime_seconds | waiting_query_runtime_minutes | waiting_locktype | waiting_xid 
----------+-------+---------+-------------------------------+------------------+-------------+-----------------+-------------+-------------------------------+---------------------+---------------------+-------------------------------+-----------------+------------------+------------------+----------------+---------------------------+---------------------------+---------------+-------------+-------------+-----------------+-------------------------------+--------------------------+---------------------+-------------------------+---------------------+-------------------------------+---------------------------+-----------------------+-------------------------------+-------------------------------+------------------+-------------
 pgp      |  7317 | pgp     | 2015-06-04 14:38:41.040442-07 | psql             | ::1         |                 |       41144 | 2015-06-09 15:49:02.195267-07 | idle in transaction |                   1 | 2015-06-09 15:49:04.755746-07 | 00:06:24.459614 |              384 |              6.4 |              2 |                       377 |          6.28333333333333 | relation      |             |        9031 | pgp             | 2015-06-05 11:06:15.305611-07 | psql                     | ::1                 |                         |               41158 | 2015-06-09 15:50:41.503815-07 | active                    | 00:04:47.711545       |                           288 |                           4.8 | tuple            |            
 pgp      |  7317 | pgp     | 2015-06-04 14:38:41.040442-07 | psql             | ::1         |                 |       41144 | 2015-06-09 15:49:02.195267-07 | idle in transaction |                   1 | 2015-06-09 15:49:04.755746-07 | 00:06:24.459614 |              384 |              6.4 |              2 |                       377 |          6.28333333333333 | transactionid |        2720 |        8731 | pgp             | 2015-06-05 10:03:07.528725-07 | psql                     | ::1                 |                         |               41154 | 2015-06-09 15:49:12.167473-07 | active                    | 00:06:17.047887       |                           377 |              6.28333333333333 | transactionid    |        2720
 pgp      | 11353 | pgp     | 2015-06-08 11:29:44.599319-07 | psql             | ::1         |                 |       41181 | 2015-06-09 15:54:34.959313-07 | idle in transaction |                   1 | 2015-06-09 15:54:34.959731-07 | 00:00:54.255629 |               54 |              0.9 |              0 |                           |                           |               |             |             |                 |                               |                          |                     |                         |                     |                               |                           |                       |                               |                               |                  |            
(3 ro
</pre>

        <h4>SQL Example Results Pretty</h4>
<pre>
-[ RECORD 1 ]-----------------+------------------------------
database                      | pgp
pid                           | 7317
usename                       | pgp
login                         | 2015-06-04 14:38:41.040442-07
application_name              | psql
client_addr                   | ::1
client_hostname               | 
client_port                   | 41144
transaction_start             | 2015-06-09 15:49:02.195267-07
transaction_state             | idle in transaction
idle_in_transaction           | 1
transaction_state_since       | 2015-06-09 15:49:04.755746-07
time_in_state                 | 00:05:33.935822
seconds_in_state              | 334
minutes_in_state              | 5.56666666666667
blocking_count                | 2
max_blocking_time_seconds     | 327
max_blocking_time_minutes     | 5.45
locktype                      | relation
holding_xid                   | 
waiting_pid                   | 9031
waiting_usename               | pgp
waiting_login                 | 2015-06-05 11:06:15.305611-07
waiting_application_name      | psql
waiting_client_addr           | ::1
waiting_client_hostname       | 
waiting_client_port           | 41158
waiting_query_start           | 2015-06-09 15:50:41.503815-07
waiting_transaction_state     | active
waiting_query_runtime         | 00:03:57.187753
waiting_query_runtime_seconds | 237
waiting_query_runtime_minutes | 3.95
waiting_locktype              | tuple
waiting_xid                   | 
-[ RECORD 2 ]-----------------+------------------------------
database                      | pgp
pid                           | 7317
usename                       | pgp
login                         | 2015-06-04 14:38:41.040442-07
application_name              | psql
client_addr                   | ::1
client_hostname               | 
client_port                   | 41144
transaction_start             | 2015-06-09 15:49:02.195267-07
transaction_state             | idle in transaction
idle_in_transaction           | 1
transaction_state_since       | 2015-06-09 15:49:04.755746-07
time_in_state                 | 00:05:33.935822
seconds_in_state              | 334
minutes_in_state              | 5.56666666666667
blocking_count                | 2
max_blocking_time_seconds     | 327
max_blocking_time_minutes     | 5.45
locktype                      | transactionid
holding_xid                   | 2720
waiting_pid                   | 8731
waiting_usename               | pgp
waiting_login                 | 2015-06-05 10:03:07.528725-07
waiting_application_name      | psql
waiting_client_addr           | ::1
waiting_client_hostname       | 
waiting_client_port           | 41154
waiting_query_start           | 2015-06-09 15:49:12.167473-07
waiting_transaction_state     | active
waiting_query_runtime         | 00:05:26.524095
waiting_query_runtime_seconds | 327
waiting_query_runtime_minutes | 5.45
waiting_locktype              | transactionid
waiting_xid                   | 2720
-[ RECORD 3 ]-----------------+------------------------------
database                      | pgp
pid                           | 11353
usename                       | pgp
login                         | 2015-06-08 11:29:44.599319-07
application_name              | psql
client_addr                   | ::1
client_hostname               | 
client_port                   | 41181
transaction_start             | 2015-06-09 15:54:34.959313-07
transaction_state             | idle in transaction
idle_in_transaction           | 1
transaction_state_since       | 2015-06-09 15:54:34.959731-07
time_in_state                 | 00:00:03.731837
seconds_in_state              | 4
minutes_in_state              | 0.0666666666666667
blocking_count                | 0
max_blocking_time_seconds     | 0
max_blocking_time_minutes     | 0
locktype                      | 
holding_xid                   | 
waiting_pid                   | 
waiting_usename               | 
waiting_login                 | 
waiting_application_name      | 
waiting_client_addr           | 
waiting_client_hostname       | 
waiting_client_port           | 
waiting_query_start           | 
waiting_transaction_state     | 
waiting_query_runtime         | 
waiting_query_runtime_seconds | 
waiting_query_runtime_minutes | 
waiting_locktype              | 
waiting_xid                   | 
</pre>

    <h3>Dashboard
 Query</h3>
        <h4>SQL</h4>
<pre>
-- If you are pasting into psql you may need to paste 1/3rd at a time because it has too many characters to paste properly. This was fun to write...
SELECT DISTINCT ON (pid) base.database,
                         pid,
                         usename,
                         idle_in_transaction,
                         minutes_in_state,
                         blocking_count,
                         max_blocking_time_minutes,
                         ('database: ' || base.database || '\n' 
                         || 'pid: ' || base.pid || '\n' 
                         || 'username: ' || base.usename || '\n' 
                         || 'application_name: ' || COALESCE(base.application_name, '') || '\n' 
                         || 'client_addr: ' || base.client_addr || '\n' 
                         || 'client_hostname: ' || COALESCE(base.client_hostname, '') || '\n' 
                         || 'client_port: ' || base.client_port || '\n' 
                         || 'idle_since: ' || COALESCE( CAST(base.transaction_state_since AS text), '') || '\n' 
                         || 'minutes_idle: ' || CAST(round(CAST(base.minutes_in_state AS numeric), 2) AS text) || '\n' 
                         || 'xid: ' || COALESCE( CAST(base.holding_xid as text), '') || '\n' 
                         || 'blocking_count: ' || base.blocking_count || '\n' 
                         || 'max_blocking_time_minutes: ' || CAST(round(CAST(base.max_blocking_time_minutes AS numeric), 2) AS text)) AS idle_in_transaction_details,
                         left(('db: ' || base.database || '\n' 
                         || 'pid: ' || base.pid || '\n' 
                         || 'user: ' || base.usename || '\n' 
                         || 'app: ' || COALESCE(base.application_name, '') || '\n' 
                         || 'client_addr: ' || base.client_addr || '\n' 
                         || 'host: ' || COALESCE(base.client_hostname, '') || '\n' 
                         || 'port: ' || base.client_port || '\n' 
                         || 'idle_since: ' || COALESCE( CAST(base.transaction_state_since AS text), '') || '\n' 
                         || 'min_idle: ' || CAST(round(CAST(base.minutes_in_state AS numeric), 2) AS text) || '\n' 
                         || 'xid: ' || COALESCE( CAST(base.holding_xid as text), '') || '\n' 
                         || 'blocked: ' || base.blocking_count) || '\n' 
                         || 'max_block_mins: ' || CAST(round(CAST(base.max_blocking_time_minutes AS numeric), 2) AS text), 200) AS idle_in_transaction_details_200char
FROM
  (SELECT current_catalog AS database,
  -- this blank record is required for Dashboard
; the alert will never shut off unless there is a result with value '0'
          NULL AS pid,
          'PLACEHOLDER' AS usename,
          NULL AS login,
          NULL AS application_name,
          NULL AS client_addr,
          NULL AS client_hostname,
          NULL AS client_port,
          NULL AS transaction_start,
          NULL AS transaction_state,
          0 AS idle_in_transaction,
          NULL AS transaction_state_since,
          NULL AS time_in_state,
          0 AS seconds_in_state,
          0 AS minutes_in_state,
          0 AS blocking_count,
          0 AS max_blocking_time_seconds,
          0 AS max_blocking_time_minutes,
          NULL AS locktype,
          NULL AS holding_xid,
          NULL AS waiting_pid,
          NULL AS waiting_usename,
          NULL AS waiting_login,
          NULL AS waiting_application_name,
          NULL AS waiting_client_addr,
          NULL AS waiting_client_hostname,
          NULL AS waiting_client_port,
          NULL AS waiting_query_start,
          NULL AS waiting_transaction_state,
          NULL AS waiting_query_runtime,
          NULL AS waiting_query_runtime_seconds,
          NULL AS waiting_query_runtime_minutes,
          NULL AS waiting_locktype,
          NULL AS waiting_xid
UNION ALL
SELECT pg_stat_activity.datname AS database,
       -- transaction info from pg_stat_activity:
       pg_stat_activity.pid,
       pg_stat_activity.usename,
       pg_stat_activity.backend_start AS login,
       left(pg_stat_activity.application_name, 200) AS application_name,
       pg_stat_activity.client_addr,
       left(pg_stat_activity.client_hostname, 200) AS client_hostname,
       pg_stat_activity.client_port,
       pg_stat_activity.xact_start AS transaction_start,
       pg_stat_activity.state AS transaction_state,
       1 AS idle_in_transaction,
       pg_stat_activity.state_change AS transaction_state_since,
       now()-pg_stat_activity.state_change AS time_in_state,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_activity.state_change))) AS seconds_in_state,
       (ROUND(EXTRACT(EPOCH
                      FROM (now())) - EXTRACT(EPOCH
                                              FROM (pg_stat_activity.state_change))))/60 AS minutes_in_state,
       -- if the transaction is idle_in_transaction and blocking other transactions from completing
       count(blocking.waiting_pid) OVER (PARTITION BY pg_stat_activity.pid) AS blocking_count,
       COALESCE((max(blocking.waiting_query_runtime_seconds) OVER (PARTITION BY pg_stat_activity.pid)),0) AS max_blocking_time_seconds,
       COALESCE((max(blocking.waiting_query_runtime_minutes) OVER (PARTITION BY pg_stat_activity.pid)),0) AS max_blocking_time_minutes,
       blocking.locktype,
       blocking.holding_xid,
       blocking.waiting_pid,
       blocking.waiting_usename,
       blocking.waiting_login,
       blocking.waiting_application_name,
       blocking.waiting_client_addr,
       blocking.waiting_client_hostname,
       blocking.waiting_client_port,
       blocking.waiting_query_start,
       blocking.waiting_transaction_state,
       blocking.waiting_query_runtime,
       blocking.waiting_query_runtime_seconds,
       blocking.waiting_query_runtime_minutes,
       blocking.waiting_locktype,
       blocking.waiting_xid
FROM pg_stat_activity
LEFT OUTER JOIN
  (SELECT holding.pid AS pid,
          -- holding transaction info from pg_lock:
          holding.locktype AS locktype,
          holding.transactionid AS holding_xid,
          -- waiting transaction info from pg_stat_activity:
          waiting.pid AS waiting_pid,
          waiting_stm.usename AS waiting_usename,
          waiting_stm.backend_start AS waiting_login,
          left(waiting_stm.application_name, 200) AS waiting_application_name,
          waiting_stm.client_addr AS waiting_client_addr,
          left(waiting_stm.client_hostname, 200) AS waiting_client_hostname,
          waiting_stm.client_port AS waiting_client_port,
          waiting_stm.query_start AS waiting_query_start,
          waiting_stm.state AS waiting_transaction_state,
          now()-waiting_stm.query_start AS waiting_query_runtime,
          ROUND(EXTRACT(EPOCH
                        FROM (now())) - EXTRACT(EPOCH
                                                FROM (waiting_stm.query_start))) AS waiting_query_runtime_seconds,
          ROUND(EXTRACT(EPOCH
                        FROM (now())) - EXTRACT(EPOCH
                                                FROM (waiting_stm.query_start)))/60 AS waiting_query_runtime_minutes,
          -- waiting transaction info from pg_lock:
          waiting.locktype AS waiting_locktype,
          waiting.transactionid AS waiting_xid
   FROM pg_catalog.pg_locks AS waiting
   JOIN pg_catalog.pg_stat_activity AS waiting_stm ON (waiting_stm.pid = waiting.pid)
   JOIN pg_catalog.pg_locks AS holding ON ((waiting."database" = holding."database"
                                            AND waiting.relation = holding.relation)
                                           OR waiting.transactionid = holding.transactionid)
   JOIN pg_catalog.pg_stat_activity AS holding_stm ON (holding_stm.pid = holding.pid)
   WHERE NOT waiting.granted
     AND holding.granted
     AND waiting.pid <> holding.pid
     AND holding_stm.state = 'idle in transaction') blocking ON (blocking.pid = pg_stat_activity.pid)
WHERE pg_stat_activity.state = 'idle in transaction') base;
</pre>

        <h4>SQL Example Results</h4>
<pre>
 database |  pid  |   usename   | idle_in_transaction | minutes_in_state | blocking_count | max_blocking_time_minutes |                                                                                              idle_in_transaction_details                                                                                               |                                                                idle_in_transaction_details_200char                                                                 
----------+-------+-------------+---------------------+------------------+----------------+---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------
 pgp      |  7317 | pgp         |                   1 | 7.13333333333333 |              2 |                       7.4 | database: pgp\npid: 7317\nusername: pgp\napplication_name: psql\nclient_addr: ::1/128\nclient_hostname: \nclient_port: 41144\nidle_since: 2015-06-09 15:49:04.755746-07\nminutes_idle: 7.13\nxid: \nblocking_count: 2  | db: pgp\npid: 7317\nuser: pgp\napp: psql\nclient_addr: ::1/128\nhost: \nport: 41144\nidle_since: 2015-06-09 15:49:04.755746-07\nmin_idle: 7.13\nxid: \nblocked: 2
 pgp      | 11353 | pgp         |                   1 | 1.63333333333333 |              0 |                         0 | database: pgp\npid: 11353\nusername: pgp\napplication_name: psql\nclient_addr: ::1/128\nclient_hostname: \nclient_port: 41181\nidle_since: 2015-06-09 15:54:34.959731-07\nminutes_idle: 1.63\nxid: \nblocking_count: 0 | db: pgp\npid: 11353\nuser: pgp\napp: psql\nclient_addr: ::1/128\nhost: \nport: 41181\nidle_since: 2015-06-09 15:54:34.959731-07\nmin_idle: 1.63\nxid: \nblocked: 0
 pgp      |       | PLACEHOLDER |                   0 |                0 |              0 |                         0 |                                                                                                                                                                                                                        | 
(3 rows)
</pre>

        <h4>SQL Example Results Pretty</h4>
<pre>
-[ RECORD 1 ]-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
database                            | pgp
pid                                 | 7317
usename                             | pgp
idle_in_transaction                 | 1
minutes_in_state                    | 7.51666666666667
blocking_count                      | 2
max_blocking_time_minutes           | 7.4
idle_in_transaction_details         | database: pgp\npid: 7317\nusername: pgp\napplication_name: psql\nclient_addr: ::1/128\nclient_hostname: \nclient_port: 41144\nidle_since: 2015-06-09 15:49:04.755746-07\nminutes_idle: 7.52\nxid: \nblocking_count: 2
idle_in_transaction_details_200char | db: pgp\npid: 7317\nuser: pgp\napp: psql\nclient_addr: ::1/128\nhost: \nport: 41144\nidle_since: 2015-06-09 15:49:04.755746-07\nmin_idle: 7.52\nxid: \nblocked: 2
-[ RECORD 2 ]-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
database                            | pgp
pid                                 | 11353
usename                             | pgp
idle_in_transaction                 | 1
minutes_in_state                    | 2.01666666666667
blocking_count                      | 0
max_blocking_time_minutes           | 0
idle_in_transaction_details         | database: pgp\npid: 11353\nusername: pgp\napplication_name: psql\nclient_addr: ::1/128\nclient_hostname: \nclient_port: 41181\nidle_since: 2015-06-09 15:54:34.959731-07\nminutes_idle: 2.02\nxid: \nblocking_count: 0
idle_in_transaction_details_200char | db: pgp\npid: 11353\nuser: pgp\napp: psql\nclient_addr: ::1/128\nhost: \nport: 41181\nidle_since: 2015-06-09 15:54:34.959731-07\nmin_idle: 2.02\nxid: \nblocked: 0
-[ RECORD 3 ]-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
database                            | pgp
pid                                 | 
usename                             | PLACEHOLDER
idle_in_transaction                 | 0
minutes_in_state                    | 0
blocking_count                      | 0
max_blocking_time_minutes           | 0
idle_in_transaction_details         | 
idle_in_transaction_details_200char | 
</pre>



  <h2>connections_databases</h2>

    <h3>Overview</h3>
      <p>Each PostgreSQL database has a connection limit that can be specified. The default, and most common value, is unlimited (value = -1). However, a limit can be set on each database using the ALTER DATABASE command. Once the number of connections to a database has reached its limit PostgreSQL will throw an error to any new connection attempts to that database. We divide the current number of connections to each database / that databases' connection limit. When the connection limit is reached this value will be 1 (100%). When the limit is being reached a superuser can perform an "ALTER DATABASE name WITH CONNECTION LIMIT connlimit" to increase the connection limit, and can make connections unlimited by setting it to -1. A superuser can also use pg_terminate_backend() to force idle users to disconnect.</p>

    <p>References:</p>
      <ul><li>http://www.postgresql.org/docs/devel/static/sql-alterdatabase.html</li></ul>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>.85 (85%)</li>
        <li><b>Critical: </b>.90 (90%)</li>
      </ul>
    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ol>
          <li>Confirm that database level connection limits are necessary and perform an "ALTER DATABASE name WITH CONNECTION LIMIT connlimit" to increase it to an appropriately value or set it to unlimited(setting -1 removes the limit). See http://www.postgresql.org/docs/devel/static/sql-alterdatabase.html</li>
          <li>Notify customer and ask if this is expected</li>
            <ul>
              <li>If not, ask if the app can be restarted and see if some clear up</li>
              <li>If so, re-evaluate why there is a database level connection limit and increase</li>
                  <ul><li>If they are approaching 350 'active' connections on the <i>server</i> consider implementing a connection pooler (such as PgBouncer)</li></ul>
            </ul>
        </ol>
      <p><b>Critical</b></p>
        <ol>
          <li>Perform an "ALTER DATABASE name WITH CONNECTION LIMIT connlimit" to increase the connection limit or set it to unlimited(setting -1 removes the limit). See http://www.postgresql.org/docs/devel/static/sql-alterdatabase.html</li>
          <li>Confirm that the database level connection limits are necessary and perform an "ALTER DATABASE name WITH CONNECTION LIMIT connlimit" to increase it to an appropriately value or set it to unlimited(setting -1 removes the limit)</li>
          <li>Notify the customer and ask if this is expected</li>
              <li>If not, ask if the app can be restarted and see if some clear up</li>
              <li>If so, re-evaluate why there is a database level connection limit and increase</li>
                  <ul><li>If they are approaching 350 'active' connections on the <i>server</i> consider implementing a connection pooler (such as PgBouncer)</li></ul>
            </ul>
        </ol> 
      <p><b>Trending</b></p>
        <ul>
          <li>If they are approaching 350 'active' connections to the <i>server</i> consider implementing a connection pooler (such as PgBouncer)</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>Run once per cluster.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT pg_database.datname AS database,
       pg_database.datconnlimit AS database_max_connections,
       pg_stat_database.numbackends AS database_connection_count,
       CASE pg_database.datconnlimit
           WHEN -1 THEN 0
           ELSE CAST(pg_stat_database.numbackends AS real)/pg_database.datconnlimit
       END AS perc_database_connections_used,
       temp_files,
       temp_bytes
FROM pg_database
LEFT JOIN pg_stat_database ON (pg_stat_database.datid = pg_database.oid)
ORDER BY database;
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database  | database_max_connections | database_connection_count | perc_database_connections_used | temp_files | temp_bytes
-----------+--------------------------+---------------------------+--------------------------------+------------+------------
 pgp       |                       10 |                         1 |                            0.1 |          0 |          0
 postgres  |                       -1 |                         1 |                              0 |          0 |          0
 template0 |                       -1 |                         0 |                              0 |          0 |          0
 template1 |                       -1 |                         0 |                              0 |          0 |          0
(4 rows)
</pre>


    <h3>Dashboard
 Query</h3>
      <h4>SQL</h4>
<pre>
SELECT pg_database.datname AS database,
       pg_database.datconnlimit AS database_max_connections,
       pg_stat_database.numbackends AS database_connection_count,
       CASE pg_database.datconnlimit
           WHEN -1 THEN 0
           ELSE 100.00 * (CAST(pg_stat_database.numbackends AS real)/pg_database.datconnlimit)
       END AS perc_database_connections_used,
       ('database_max_connections: ' || CASE pg_database.datconnlimit
                                            WHEN -1 THEN 'unlimited'
                                            ELSE CAST(pg_database.datconnlimit AS text)
                                        END || '\n' || 'database_connection_count: ' || pg_stat_database.numbackends) AS perc_database_connections_used_details,
       temp_files,
       temp_bytes
FROM pg_database
LEFT JOIN pg_stat_database ON (pg_stat_database.datid = pg_database.oid)
ORDER BY database;
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database  | database_max_connections | database_connection_count | perc_database_connections_used |              perc_database_connections_used_details               | temp_files | temp_bytes
-----------+--------------------------+---------------------------+--------------------------------+-------------------------------------------------------------------+------------+------------
 pgp       |                       10 |                         1 |                              1 | database_max_connections: 10\ndatabase_connection_count: 1        |          0 |          0
 postgres  |                       -1 |                         1 |                              0 | database_max_connections: unlimited\ndatabase_connection_count: 1 |          0 |          0
 template0 |                       -1 |                         0 |                              0 | database_max_connections: unlimited\ndatabase_connection_count: 0 |          0 |          0
 template1 |                       -1 |                         0 |                              0 | database_max_connections: unlimited\ndatabase_connection_count: 0 |          0 |          0
(4 rows)
</pre>



  <h2>connections_server</h2>
    <p>TODO: provide SQL to force backends in state 'idle' to disconnect</p>

    <h3>Overview</h3>
      <p>Each PostgreSQL server has a connection limit set in the config file ("max_connections" parameter). Once the number of connections to a server has reached this limit PostgreSQL will throw an error to any new connection attempts. We divide the current number of connections to the server / the server's connection limit (max_connections). When the connection limit is reached this value will be 1 (100%). In this case a superuser can use pg_terminate_backend(pid) to force connections that are in state "idle" to disconnect (note "idle in transaction" connections should not be terminated), may modify the max_connections parameter (requires restart), or suggest a connection pooler. Note that there is also a per-database limit that is tracked separately (see connections_databases).</p>

    <p>References:</p>
      <ul><li>http://www.PostgreSQL.org/docs/devel/static/runtime-config-connection.html#GUC-MAX-CONNECTIONS</li></ul>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>.85 (85%)</li>
        <li><b>Critical: </b>.90 (90%)</li>
      </ul>
    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Notify customer and ask if this is expected</li>
            <ul>
              <li>If not, ask if the app can be restarted and see if some clear up</li>
              <li>If so, increase max_connections in the config and restart the server</li>
                <ul>
                  <li>If you modify max_connections you may also want to modify the work_mem setting which we typically set = (1/4 total_RAM / max_connections) per the tuning guide</li>
                  <li>If they are approaching 350 'active' connections consider implementing a connection pooler (such as PgBouncer)
              </li>
                </ul>
            </ul>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>Forcefully kill any 'idle' sessions (note: 'idle in transaction' connections should not be terminated)</li>
          <li>Notify the customer and ask if this is expected</li>
            <ul>
              <li>If not, ask if the app can be restarted and see if some clear up</li>
              <li>If so, increase max_connections in the config and restart the server</li>
                <ul>
                  <li>If you modify max_connections you may also want to modify the work_mem setting which we typically set = (1/4 total_RAM / max_connections) per the tuning guide</li>
                  <li>If they are approaching 350 'active' connections consider implementing a connection pooler (such as PgBouncer)
</li>
                </ul>
            </ul>
        </ul> 
      <p><b>Trending</b></p>
        <ul>
          <li>If they are approaching 350 'active' connections consider implementing a connection pooler (such as PgBouncer)
</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>Run once per cluster.</p>


    <h3>Base Query</h3>
        <h4>SQL</h4>
<pre>
SELECT server_max_connections,
       server_connection_count,
       CAST(server_connection_count AS real)/server_max_connections AS perc_server_connections_used
FROM
  (SELECT
     (SELECT CAST(setting AS numeric) AS server_max_connections
      FROM pg_settings
      WHERE name = 'max_connections') AS server_max_connections,
          SUM(pg_stat_database.numbackends) AS server_connection_count
   FROM pg_stat_database
   GROUP BY 1) base;
</pre>


        <h4>SQL Example Results</h4>
<pre>
 server_max_connections | server_connection_count | perc_server_connections_used 
------------------------+-------------------------+------------------------------
                    100 |                       1 |                         0.01
(1 row)
</pre>


    <h3>Dashboard
 Query</h3>
        <h4>SQL</h4>
<pre>
SELECT server_max_connections,
       server_connection_count,
       100.00 * (CAST(server_connection_count AS real)/server_max_connections) AS perc_server_connections_used,
       ('server_max_connections: ' || server_max_connections || '\n' || 'server_connection_count: ' || server_connection_count) AS perc_server_connections_used_details
FROM
  (SELECT
     (SELECT CAST(setting AS numeric) AS server_max_connections
      FROM pg_settings
      WHERE name = 'max_connections') AS server_max_connections,
          SUM(pg_stat_database.numbackends) AS server_connection_count
   FROM pg_stat_database
   GROUP BY 1) base;
</pre>


        <h4>SQL Example Results</h4>
<pre>
 server_max_connections | server_connection_count | perc_server_connections_used |          perc_server_connections_used_details           
------------------------+-------------------------+------------------------------+---------------------------------------------------------
                    100 |                       1 |                            1 | server_max_connections: 100\nserver_connection_count: 1
(1 row)
</pre>



  <h2>os_process_autovacuum_launcher</h2>

    <h3>Overview</h3>
      <p>The 'postgres: autovacuum launcher process' operating system process should be started by the postgres/postmaster process when PostgreSQL starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: autovacuum launcher process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: autovacuum launcher process' -c)-1))"
</pre>



  <h2>os_process_checkpointer</h2>

    <h3>Overview</h3>
      <p>The 'postgres: checkpointer process' operating system process should be started by the postgres/postmaster process when PostgreSQL starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: checkpointer process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: checkpointer process' -c)-1))"
</pre>



  <h2>os_process_logger</h2>

    <h3>Overview</h3>
      <p>The 'postgres: logger process' operating system process should be started by the postgres/postmaster process when PostgreSQL starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: logger process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: logger process' -c)-1))"
</pre>



  <h2>os_process_postgres_postmaster</h2>

    <h3>Overview</h3>
      <p>The the postgres/postmaster process is the process that starts (and is therefor the parent) of all other PostgreSQL processes, and manages connections to the server. It must always be running. If the process is terminated all of the other PostgreSQL processes (for which it is the parent) will also be terminated and you will have a complete outage and you must follow recovery procedures. Note that the operating system process may be either 'postgres' or 'postmaster' ('postmaster' is a deprecated alias for 'postgres'). We use linux bash to get the count of running processes with text './postgres' or text './postmaster'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Check the operating system and ensure the process is actually down</li>
            <ul>
              <li>Command: ps -ef | grep ./postgres</li>
              <li>Command: ps -ef | grep ./postmaster</li>
            </ul>
          <li>Follow server outage procedures</li>
          <li>Confirm that the outage was not caused by an operating system user killing the process, and request access be restricted as necessary</li>
        </ol>

    <h3>bash Command</h3>
<pre>
# Note we subtract 2 because our 2 greps will be returned as well
echo "$((($(ps -ef | grep ./postgres  -c) + $(ps -ef | grep ./postmaster  -c))-2))"
</pre>



  <h2>os_process_stats_collector</h2>

    <h3>Overview</h3>
      <p>The 'postgres: stats collector process' operating system process should be started by the postgres/postmaster process when PostgreSQL starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: stats collector process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: stats collector process' -c)-1))"
</pre>



  <h2>os_process_wal_writer</h2>

    <h3>Overview</h3>
      <p>The 'postgres: wal writer process' operating system process should be started by the postgres/postmaster process when PostgreSQL starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: wal writer process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: wal writer process'  -c)-1))"
</pre>



  <h2>os_process_writer</h2>

    <h3>Overview</h3>
      <p>The 'postgres: writer process' operating system process should be started by the postgres/postmaster process when PostgreSQL starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: writer process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: writer process' -c)-1))"
</pre>



  <h2>outage</h2>

    <h3>Overview</h3>
      <p>To ensure the database is up and users are able to connect we connect and execute a simple 'SELECT 1;' or 'SELECT now();' query. EBD consultant Douglas Hunley recommends 'SELECT now();' because you will not receive a cached value if you are using a fancy pooler or sharding application that caches values. However, 'SELECT now();' has more overhead than 'SELECT 1;' so if we know we will connect directly or through PgBouncer 'SELECT 1;' may be preferred</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>command fails</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>Check 'os_process_postgres_postmaster' monitor. Follow standard outage procedure</li>
        </ul> 
      <p><b>Trending</b></p>
        <ul>
          <li>If outage occur frequently check the tickets and evaluate why they occur so often; resolve if possible.</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>In most situations it is sufficient to run once for the entire cluster (outages are typically cluster wide) but you can run once per database for coverage on database specific issues such as the database connection max being reached (as described and covered by connections_databases monitor).</p>


    <h3>Base Query</h3>
      <p>See notes in 'Overview' to evaluate if 'SELECT 1;' or 'SELECT now();' should be used.</p>

      <h4>SQL</h4>
      
      <p>Most efficient and sufficient for most cases:</p>
        <code>SELECT 1;</code>

      <p>Less efficient but provides coverage for situations when you are connecting through a pooling or sharding application which may return a cached value when you execute 'SELECT 1;':</p>
      <code>SELECT now();</code>

    <h3>Dashboard
 Query</h3>
      <p>Same as Base Query.</p>



  <h2>pg_basebackup_log</h2>

    <h3>Overview</h3>
      <p>The pg_basebackup utility is run from cron to backup the database. Errors are directed to a log file which the monitoring tool reads and parses for errors.</p>
      <p><b>Note: </b>this requires coordination between the team configuring the monitoring and the team setting up the pg_basebackup cron jobs. The cron job must be set up to direct pg_basebackup log output to a file, with a policy to delete old data from the file. It must be consistent and any changes need to be coordinated with the monitoring team.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>TBD</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>'ERROR'</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li></li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>Review the pg_basebackup log and cron job to investigate and mitigate the issue</li>
        </ul> 

    <h3>bash Command</h3>
      <p>Use the monitoring tool's built in file scraping utilities.</p>



  <h2>xlog_readyfiles_age</h2>

    <h3>Overview</h3>
      <p>When WAL (write ahead log) archiving is enabled, PostgreSQL uses the bash command listed next to "archive_command" in the config to copy each finished xlog to the archive destination. PostgreSQL tracks this process as follows:</p>
        <ol>
          <li>When a WAL log is finished (full) PostgreSQL creates a file in the directory '$PGDATA/pg_xlog/archive_status/' with the name of the WAL file and file extension .ready</li>
            <ul><li>For example, if WAL log 'abc123' filled up PostgreSQL would create a file named '$PGDATA/pg_xlog/archive_status/abc123.ready'</li></ul>
          <li>PostgreSQL executes the archive_command on that finished WAL log to copy it to the archive destination</li>
          <li>If the archive_command succeeded PostgreSQL will rename the .ready file it created in '$PGDATA/pg_xlog/archive_status/' with the name of the WAL file and the extension.done</li>
            <ul><li>For example, if WAL log 'abc123' filled up PostgreSQL would have created a file named '$PGDATA/pg_xlog/archive_status/abc123.ready'. Once the archive_command completes (and the file is succesfully copied to the archive destination) the file '$PGDATA/pg_xlog/archive_status/abc123.ready' is renamed to '$PGDATA/pg_xlog/archive_status/abc123.done'</li></ul>
          <li>The .done file that was created will eventually be deleted</li>
        </ol>
        <ul><li>However: if the archive_command fails the file will not be archived and the .ready file created in '$PGDATA/pg_xlog/archive_status/' will remain and will not be renamed or deleted until the archive command succeeds (and the finished WAL is copied to the archive destination)</li></ul>
       <p>The successful completion of the archive_command is critical for recovery. If the archive_command is failing the completed WAL logs may not be available should the database's filesystem fail. Luckily the above process makes it very easy to monitor failure of archive command and ensure WAL logs are being logged within your Recovery Point Objective. We check the age in seconds of the oldest .ready file in $PGDATA/pg_xlog/archive_status/ and make sure there is no .ready file older than our recovery point objective.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>as needed to ensure your recovery time objective for this client is always met. Suggest (.90 * (Recovery Time Objective - response time))</li>
        <li><b>Critical: </b>as needed to ensure your recovery time objective for this client is always met. Suggest (.95 * (Recovery Time Objective - response time))</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Check logs for details on why archive command is failing and correct it</li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>Manually copy WAL logs to a recovery destination</li>
          <li>Check logs for details on why archive command is failing and correct it</li>
        </ul>


    <h3>bash Command</h3>
<pre>
The following bash command will return the number of seconds old the oldest .ready file is. If there is no .ready file it will return 0. We cannot divide the seconds by 60 to get minutes in bash because bash cannot do floating point arithmetic
if [ "$(ls -A $PGDATA/pg_xlog/archive_status/*.ready)" ]; then echo "$(($(date +'%s') - $(date -d "$(ls $PGDATA/pg_xlog/archive_status/*.ready -clt --time-style=full-iso | tail -1 | awk '{print $6, $7}')" +'%s')))"; else echo 0; fi 2>/dev/null
</pre>

    <h3>SQL Scope - PostgreSQL 9.4 and above</h3>
      <p>This SQL is compatible with PostgreSQL versions 9.4 or higher. Run once per cluster.</p>

    <h3>Base Query - PostgreSQL 9.4 and above</h3>
      <h4>SQL</h4>
        <p><b>Note: </b>requires PostgreSQL version 9.4 or higher. The pg_stat_archiver view was added in PostgreSQL 9.4</p>
        <p><b>Note: </b>will always return exactly 1 row</p>
<pre>
SELECT archived_count,
       last_archived_wal,
       last_archived_time,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_archiver.last_archived_time))) AS time_since_last_archive_seconds,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_archiver.last_archived_time)))/60 AS time_since_last_archive_minutes,
       failed_count,
       last_failed_wal,
       last_failed_time,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_archiver.last_failed_time))) AS time_since_last_failed_seconds,
       ROUND(EXTRACT(EPOCH
                     FROM (now())) - EXTRACT(EPOCH
                                             FROM (pg_stat_archiver.last_failed_time)))/60 AS time_since_last_failed_minutes,
       stats_reset
FROM pg_stat_archiver;
</pre>

      <h4>SQL Example Results</h4>
<pre>
 archived_count | last_archived_wal | last_archived_time | time_since_last_archive_seconds | time_since_last_archive_minutes | failed_count | last_failed_wal | last_failed_time | time_since_last_failed_seconds | time_since_last_failed_minutes |          stats_reset          
----------------+-------------------+--------------------+---------------------------------+---------------------------------+--------------+-----------------+------------------+--------------------------------+--------------------------------+-------------------------------
              0 |                   |                    |                                 |                                 |            0 |                 |                  |                                |                                | 2015-06-03 14:55:21.658737-07
(1 row)
</pre>

      <h4>SQL Example Results Pretty</h4>
<pre>
-[ RECORD 1 ]-------------------+------------------------------
archived_count                  | 0
last_archived_wal               | 
last_archived_time              | 
time_since_last_archive_seconds | 
time_since_last_archive_minutes | 
failed_count                    | 0
last_failed_wal                 | 
last_failed_time                | 
time_since_last_failed_seconds  | 
time_since_last_failed_minutes  | 
stats_reset                     | 2015-06-03 14:55:21.658737-07
</pre>

    <h3>Dashboard
 Query</h3>
      <p>Same as base query.</p>



<h1>Hot Standby</h1>


  <h2>hot_standby_outage</h2>

    <h3>Overview</h3>
    <p>The pg_stat_replication view displays an entry for each connected hot standby. If a hot standby goes down or becomes disconnected there will not be any entry for that standby. The pg_stat_replication view will also have an entry for each pg_basebackup that is running, but those will have a value in the 'state' field = 'backup' so we can exclude them easily. For this monitor we get the count of values returned by "SELECT from pg_stat_replication WHERE state NOT IN ('backup','startup')" which should return the count of hot standbys that this server should be connected to. If one standby server goes down or becomes disconnected the value returned will decrease by one.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 5 minutes</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of standby servers the server is sending WAL logs to - 1. For example, if you have a master sending WAL logs to 2 standby servers you would set critical warning to 1 (which is 2-1)</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>Investigate status of hot standby servers and recover them as necessary</li>
        </ul> 


    <h3>SQL Scope</h3>
    <p>Run once per cluster.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       connected_standby_count
FROM
  (SELECT COUNT(pid) connected_standby_count
   FROM pg_stat_replication
   WHERE pg_stat_replication.state NOT IN ('backup',
                                           'startup')) base;
</pre>

        <h4>SQL Example Result</h4>
<pre>
 database | connected_standby_count 
----------+-------------------------
 pgp      |                       0
(1 row)
</pre>

    <h3>Dashboard
 Query</h3>
    <p>Same as base query</p>



  <h2>os_process_wal_receiver</h2>

    <h3>Overview</h3>
      <p>If you have streaming replication implemented each standby server should have a WAL receiver operating system process. The 'postgres: wal receiver process' operating system process should be started by the postgres/postmaster process when the standby server starts. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process fails to restart it the server needs to be rebooted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: wal receiver process'. Note that this is checked on the machine where the <i>standby</i> is located, <i>not the master</i>.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of servers you have running on the machine - 1. For example, if you have 2 servers running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PostgreSQL server is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>If it was not restarted by the postgres/postmaster process restart the server</li>
          <li>Check the logs for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
        </ol> 

    <h3>bash Command</h3>
    <p><b>Note: </b>run on the <i>standby</i> server.</p>
<pre>
# Note we subtract 1 because our grep will be returned as well
echo "$(($(ps -ef | grep 'postgres: wal receiver process' -c)-1))"
</pre>



  <h2>os_process_wal_sender</h2>

    <h3>Overview</h3>
      <p>If you have streaming replication implemented the operating system of the master should have one WAL sender operating system process running for each standby it is sending logs to. The 'postgres: wal sender process' process should be started by the postgres/postmaster process. If it stops the postgres/postmaster process is supposed to restart it. It it stops and the postgres/postmaster process does not try to restart it you must investigate the connectivity with and the health of the standby. If the postgres/postmaster process tries to restart it but fails the server needs to be restarted. Even if postgres/postmaster managed to restart it you should check the logs for the errors. We use linux bash to get the count of running processes with text 'postgres: wal sender process'.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of standby servers the server is sending WAL logs to - 1. For example, if you have a master sending WAL logs to 2 standby servers you would set critical warning to 1 (which is 2-1)</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ol>
          <li>Recheck the status and confirm that the process was restarted by the postgres/postmaster process</li>
          <li>Check the logs and for details on the error and take action as necessary</li>
            <ul><li>You may need to confirm no operating system users have access to kill the process and request access be restricted as necessary</li></ul>
          <li>Confirm connectivity and health of the standby server</li>
          <li>If the standby(s) is/are fine and the process was not restarted by the postgres/postmaster process restart the server</li>
        </ol> 

    <h3>bash Command</h3>
    <p><b>Note: </b>run on the <i>master</i> server.</p>
<pre>
# Note we subtract 1 because our greps will be returned as well
echo "$(($(ps -ef | grep 'postgres: wal sender process'  -c)-1))"
</pre>



<h1>Hot Standby Information Only</h1>

  <h2>hot_standby_send_delay/hot_standby_replay_delay</h2>

    <h3>Overview</h3>
    <p>Execute this SQL against the master to get a status of the replicas. If the replicas are not connected they will not have an entry. This is not the same as the hot_standby_delay check in check_Postgres.pl. check_Postgres.pl requires connecting to both the master and the slave.</p>

    <p>The function pg_xlog_location_diff is used to compare the pg_lsn of 'pg_current_xlog_location' of master to the pg_lsn of the standby's write_location to determine the 'send_delay' of the standby. pg_xlog_location_diff is also used to compare the pg_lsn of the standby's write_location to the pg_lsn of the standby's replay_location to determine the 'replay_delay' of the standby. The returned value is the difference in bytes.<br>
    Requires pg_stat_replication view and therefore requires 9.2 and above. The documentation for 9.1 shows that it has this view as well but does not provide any details about the view so we do not know what it contains.</p>

    <p>The state of the standby is more complicated to determine - if the standby is not connected the query will not return any result for that standby.</p>

    <h3>Differences from check_Postgres.pl Version</h3>
    <p>The Bucardo tool check_Postgres.pl has a similar check with the same name, but rather than using the pg_stat_replication view it requires you connect to the slave as well, collect it's pg_current_xlog_location(), and compute the difference from the master.</p>

    <p><b>References:</b></p>
    <p>Slave pg_lsn Information - pg_stat_replication View</p>
        <ul><li>http://www.PostgreSQL.org/docs/devel/static/monitoring-stats.html#PG-STAT-REPLICATION-VIEW</li></ul>
    <p>Master pg_lsn Information</p>
    <ul>
        <li>http://www.PostgreSQL.org/docs/devel/static/functions-admin.html#FUNCTIONS-ADMIN-BACKUP-TABLE</li>
        <li>"SELECT pg_current_xlog_location();"</li>
        <li>"pg_current_xlog_location displays the current transaction log write location in the same format used by the above functions. Similarly, pg_current_xlog_insert_location displays the current transaction log insertion point. The insertion point is the "logical" end of the transaction log at any instant, while the write location is the end of what has actually been written out from the server's internal buffers. The write location is the end of what can be examined from outside the server, and is usually what you want if you are interested in archiving partially-complete transaction log files. The insertion point is made available primarily for server debugging purposes. These are both read-only operations and do not require superuser permissions."</li>
    </ul>
    <p>Calculate pg_lsn Difference</p>
    <ul>
    <li>pg_xlog_location_diff function</li>
    <li>http://www.PostgreSQL.org/docs/devel/static/functions-admin.html#FUNCTIONS-ADMIN-BACKUP-TABLE</li>
    <li>SELECT pg_xlog_location_diff(Slave_location, master_location);</li>
    </ul>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 5 minutes</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Health Check: </b></p>
        <ul>
          <li><b>hot_standby_replay_delay: </b>tune the standby server as needed</li>
          <li><b>hot_standby_send_delay: </b>evaluate trends. May indicate network connectivity issues</li>
        </ul>


    <h3>SQL Scope</h3>
    <p>Run once per cluster.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       pid AS standby_pid,
       usesysid AS standby_usesysid,
       usename AS standby_usename,
       client_addr AS standby_client_addr,
       client_port AS standby_client_port,
       state AS standby_state,
       write_location AS standby_write_location,
       replay_location AS standby_replay_location,
  (SELECT pg_current_xlog_location()) AS master_location,
       (pg_xlog_location_diff(pg_current_xlog_location(), write_location)) AS send_delay,
       (pg_xlog_location_diff(write_location, replay_location)) AS replay_delay
FROM pg_stat_replication
WHERE pg_stat_replication.state != 'backup';
</pre>

        <h4>SQL Example Result</h4>
<pre>
 database | standby_pid | standby_usesysid | standby_usename | standby_client_addr | standby_client_port | standby_state | standby_write_location | standby_replay_location | master_location | send_delay | replay_delay 
----------+-------------+------------------+-----------------+---------------------+---------------------+---------------+------------------------+-------------------------+-----------------+------------+--------------
 pgp      |       18242 |            16515 | replicator      | ::1                 |               46870 | streaming     | 0/13000060             | 0/13000000              | 0/13000060      |          0 |           96
</pre>

    <h3>Dashboard
 Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       pid AS standby_pid,
       usesysid AS standby_usesysid,
       usename AS standby_usename,
       client_addr AS standby_client_addr,
       client_port AS standby_client_port,
       state AS standby_state,
  (SELECT pg_current_xlog_location()) AS master_location,
       (pg_xlog_location_diff(pg_current_xlog_location(), write_location)) AS send_delay,
       (pg_xlog_location_diff(write_location, replay_location)) AS replay_delay
FROM pg_stat_replication
WHERE pg_stat_replication.state != 'backup';
</pre>

        <h4>SQL Example Result</h4>
<pre>
 database | standby_pid | standby_usesysid | standby_usename | standby_client_addr | standby_client_port | standby_state | send_delay | replay_delay 
----------+-------------+------------------+-----------------+---------------------+---------------------+---------------+------------+--------------
 pgp      |       18242 |            16515 | replicator      | ::1                 |               46870 | streaming     |          0 |           96
</pre>



<h1>Information</h1>


  <s><h2>cache_use</h2></s>
    <p><b>Not implemented. Instead when we need this data for troubleshooting we should set up a cron job to execute SQL in the database to collect it.</b></p>
      <ul>
        <li>Only desired when you are troubleshooting a problem, otherwise it adds too much overhead</li>
          <ul>
            <li>Needs to be run fairly frequently to be useful</li>
              <ul>
                <li>Would require a lot of data to be tracked in Dashboard
</li>
                <li>Would cause a lot of unnecessary load on the server</li>
              </ul>
          </ul>
        <li>Requires the contrib module pg_buffercache be installed and added as an extension to each database</li>
      </ul>
<s>
    <h3>Overview</h3>
      <p>When data from a table, index, or other object is read into the buffer cache data from other tables/objects must be evicted from the cache to make space. Poorly written SQL which is less selective than it should be and queries that have to use sequential scans because they do not have indexes that are needed can cause a large amount of uneccesary data to be read into the buffer cache, evicting more useful data and decreasing performance. We use the pg_buffercache view provided by the pg_buffercache contrib module to track how much of shared buffers each table is occupying. If a table has periodic spikes where more buffer cache is being allocated to it the queries against that table may be candidate for performance tuning.</p>
      <p>When the pg_buffercache view is accessed, internal buffer manager locks are taken for long enough to copy all the buffer state data that the view will display. This ensures that the view produces a consistent set of results, while not blocking normal buffer activity longer than necessary. Nonetheless there could be some impact on database performance if this view is read often.</p>
      <p>We wrote two versions of this query:</p>
        <ul>
          <li><b>Version 1: </b>only objects that currently have data in the buffercache are listed. This is more efficient for obvious reasons</li>
          <li><b>Version 2: </b>all objects in the database are listed regardless of whether they have data in the buffercache. This is less efficient, but it results in there being an entry for every table with 0 buffers listed rather than no entry at all. Some monitors require this when creating graphs, and Dashboard
 requires this if you are alerting (the object would need an entry with a value below the threshold for the alert to be turned off).</li>
        </ul>
      <p><b>Note: </b>this monitor requires the pg_buffercache contrib module be installed and added as an EXTENSION in each database. Use the command: CREATE EXTENSION IF NOT EXISTS pg_buffercache;</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>TBD. There is some impact on database performance if this view is read often.</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>None</li>
        </ul> 
      <p><b>Health Check</b></p>
        <ul>
          <li>If you see a table frequently filling up the cache see if there are queries for that table using sequential scans</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>Must be run once per database.</p>


    <h3>Base Query - VERSION 1</h3>
      <p>There are two versions of this query. This version (version 1) lists only those tables that have data in the buffercache, and is more efficient. The other version (version 2) lists all objects in the database regardless of if they have data in the buffercache and display a buffer count of 0 for tables with no data in cache.</p>

      <h4>SQL</h4>
<pre>
-- Each object that is using shared_buffers and how many buffers they are occupying
-- Requires EXTENSION pg_buffercache (execute "CREATE EXTENSION IF NOT EXISTS pg_buffercache;")
-- This is less computationally expensive than the version that lists all tables, but does not display objects that do not have any data in shared_buffers
SELECT current_database() AS database,
       pg_namespace.nspname AS schema,
       pg_class.relname AS relation,
       CASE pg_class.relkind
           WHEN 'r' THEN CAST('ordinary table' AS varchar(14))
           WHEN 'i' THEN CAST('index' AS varchar(14))
           WHEN 'S' THEN CAST('sequence' AS varchar(14))
           WHEN 'v' THEN CAST('view' AS varchar(14))
           WHEN 'c' THEN CAST('composite type' AS varchar(14))
           WHEN 't' THEN CAST('TOAST table' AS varchar(14))
           WHEN 'f' THEN CAST('foreign table' AS varchar(14))
           ELSE CAST(pg_class.relkind AS varchar(14))
       END as relation_kind,
       count(*) AS buffers,
       CAST(
              (SELECT setting
               FROM pg_settings
               WHERE name='shared_buffers') AS integer) AS total_buffers,
       ((count(*))/(CAST (
                            (SELECT setting
                             FROM pg_settings
                             WHERE name='shared_buffers') AS real))) AS perc_buffers
FROM pg_buffercache
INNER JOIN pg_class ON (pg_class.relfilenode = pg_buffercache.relfilenode)
AND pg_buffercache.reldatabase IN
  (SELECT oid
   FROM pg_database
   WHERE datname = current_database())
INNER JOIN pg_namespace ON (pg_namespace.oid = pg_class.relnamespace)
WHERE pg_namespace.nspname NOT IN ('pg_catalog',
                                   'information_schema')
GROUP BY 1,2,3,4
ORDER BY 1,2,3;
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |         relation         | relation_kind  | buffers | total_buffers |  perc_buffers  
----------+----------+--------------------------+----------------+---------+---------------+----------------
 postgres | edbstore | categories               | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | categories_category_seq  | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | cust_hist                | ordinary table |     331 |          4096 | 0.080810546875
 postgres | edbstore | customers                | ordinary table |     492 |          4096 |   0.1201171875
 postgres | edbstore | customers_customerid_seq | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | dept                     | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | emp                      | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | emp_pk                   | index          |       2 |          4096 |  0.00048828125
 postgres | edbstore | inventory                | ordinary table |      59 |          4096 | 0.014404296875
 postgres | edbstore | ix_cust_hist_customerid  | index          |      32 |          4096 |      0.0078125
 postgres | edbstore | job_grd                  | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | jobhist                  | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | next_empno               | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | orderlines               | ordinary table |     389 |          4096 | 0.094970703125
 postgres | edbstore | orders                   | ordinary table |     104 |          4096 |    0.025390625
 postgres | edbstore | orders_orderid_seq       | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | products                 | ordinary table |     105 |          4096 | 0.025634765625
 postgres | edbstore | products_prod_id_seq     | sequence       |       1 |          4096 | 0.000244140625
 postgres | pg_toast | pg_toast_2618            | TOAST table    |       7 |          4096 | 0.001708984375
 postgres | pg_toast | pg_toast_2618_index      | index          |       2 |          4096 |  0.00048828125
 postgres | pg_toast | pg_toast_2619            | TOAST table    |      11 |          4096 | 0.002685546875
 postgres | pg_toast | pg_toast_2619_index      | index          |       2 |          4096 |  0.00048828125
(22 rows)
</pre>


    <h3>Dashboard
 Query - Version 1</h3>
      <p>See notes about the versions in section 'Base Query - VERSION 1'</p>
      <p>This is the same as the base query except perc_buffers is multiplied by 100 to show percentage as a fraction of 100 rather than a fraction of 1 (for example .204 changed to 20.4)</p>

      <h4>SQL</h4>
<pre>
SELECT current_database() AS database,
       pg_namespace.nspname AS schema,
       pg_class.relname AS relation,
       CASE pg_class.relkind
           WHEN 'r' THEN CAST('ordinary table' AS varchar(14))
           WHEN 'i' THEN CAST('index' AS varchar(14))
           WHEN 'S' THEN CAST('sequence' AS varchar(14))
           WHEN 'v' THEN CAST('view' AS varchar(14))
           WHEN 'c' THEN CAST('composite type' AS varchar(14))
           WHEN 't' THEN CAST('TOAST table' AS varchar(14))
           WHEN 'f' THEN CAST('foreign table' AS varchar(14))
           ELSE CAST(pg_class.relkind AS varchar(14))
       END as relation_kind,
       count(*) AS buffers,
       CAST(
              (SELECT setting
               FROM pg_settings
               WHERE name='shared_buffers') AS integer) AS total_buffers,
       100.00 * ((count(*))/(CAST (
                                     (SELECT setting
                                      FROM pg_settings
                                      WHERE name='shared_buffers') AS real))) AS perc_buffers
FROM pg_buffercache
INNER JOIN pg_class ON (pg_class.relfilenode = pg_buffercache.relfilenode)
AND pg_buffercache.reldatabase IN
  (SELECT oid
   FROM pg_database
   WHERE datname = current_database())
INNER JOIN pg_namespace ON (pg_namespace.oid = pg_class.relnamespace)
WHERE pg_namespace.nspname NOT IN ('pg_catalog',
                                   'information_schema')
GROUP BY 1,2,3,4
ORDER BY 1,2,3;
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |         relation         | relation_kind  | buffers | total_buffers | perc_buffers 
----------+----------+--------------------------+----------------+---------+---------------+--------------
 postgres | edbstore | categories               | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | categories_category_seq  | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | cust_hist                | ordinary table |     331 |          4096 | 8.0810546875
 postgres | edbstore | customers                | ordinary table |     492 |          4096 |  12.01171875
 postgres | edbstore | customers_customerid_seq | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | dept                     | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | emp                      | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | emp_pk                   | index          |       2 |          4096 |  0.048828125
 postgres | edbstore | inventory                | ordinary table |      59 |          4096 | 1.4404296875
 postgres | edbstore | ix_cust_hist_customerid  | index          |      32 |          4096 |      0.78125
 postgres | edbstore | job_grd                  | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | jobhist                  | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | next_empno               | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | orderlines               | ordinary table |     389 |          4096 | 9.4970703125
 postgres | edbstore | orders                   | ordinary table |     104 |          4096 |    2.5390625
 postgres | edbstore | orders_orderid_seq       | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | products                 | ordinary table |     105 |          4096 | 2.5634765625
 postgres | edbstore | products_prod_id_seq     | sequence       |       1 |          4096 | 0.0244140625
 postgres | pg_toast | pg_toast_2618            | TOAST table    |       7 |          4096 | 0.1708984375
 postgres | pg_toast | pg_toast_2618_index      | index          |       2 |          4096 |  0.048828125
 postgres | pg_toast | pg_toast_2619            | TOAST table    |      11 |          4096 | 0.2685546875
 postgres | pg_toast | pg_toast_2619_index      | index          |       2 |          4096 |  0.048828125
(22 rows)
</pre>


    <h3>Base Query - VERSION 2</h3>
      <p>There are two versions of this query. This version (version 2) lists lists all objects in the database regardless of if they have data in the buffercache and display a buffer count of 0 for tables with no data in cache. The other version (version 2) only those tables that have data in the buffercache, and is more efficient.</p>

      <h4>SQL</h4>
<pre>
-- Each object and how many buffers they are occupying
-- Requires EXTENSION pg_buffercache (execute 'CREATE EXTENSION IF NOT EXISTS pg_buffercache;')
SELECT current_database() AS database,
       pg_namespace.nspname AS schema,
       pg_class.relname AS relation,
       CASE pg_class.relkind
           WHEN 'r' THEN CAST('ordinary table' AS varchar(14))
           WHEN 'i' THEN CAST('index' AS varchar(14))
           WHEN 'S' THEN CAST('sequence' AS varchar(14))
           WHEN 'v' THEN CAST('view' AS varchar(14))
           WHEN 'c' THEN CAST('composite type' AS varchar(14))
           WHEN 't' THEN CAST('TOAST table' AS varchar(14))
           WHEN 'f' THEN CAST('foreign table' AS varchar(14))
           ELSE CAST(pg_class.relkind AS varchar(14))
       END as relation_kind,
       count(pg_buffercache.bufferid) AS buffers,
       CAST(
              (SELECT setting
               FROM pg_settings
               WHERE name='shared_buffers') AS integer) AS total_buffers,
       ((count(pg_buffercache.bufferid))/(CAST (
                                                  (SELECT setting
                                                   FROM pg_settings
                                                   WHERE name='shared_buffers') AS real))) AS perc_buffers
FROM pg_class
INNER JOIN pg_namespace ON (pg_namespace.oid = pg_class.relnamespace)
LEFT OUTER JOIN pg_buffercache ON (pg_buffercache.relfilenode = pg_class.relfilenode)
AND pg_buffercache.reldatabase IN
  (SELECT oid
   FROM pg_database
   WHERE datname = current_database())
WHERE pg_namespace.nspname NOT IN ('pg_catalog',
                                   'information_schema')
GROUP BY 1,2,3,4
ORDER BY 1,2,3;
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |         relation         | relation_kind  | buffers | total_buffers |  perc_buffers  
----------+----------+--------------------------+----------------+---------+---------------+----------------
 postgres | edbstore | categories               | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | categories_category_seq  | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | categories_pkey          | index          |       0 |          4096 |              0
 postgres | edbstore | cust_hist                | ordinary table |     331 |          4096 | 0.080810546875
 postgres | edbstore | customers                | ordinary table |     492 |          4096 |   0.1201171875
 postgres | edbstore | customers_customerid_seq | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | customers_pkey           | index          |       0 |          4096 |              0
 postgres | edbstore | dept                     | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | dept_dname_uq            | index          |       0 |          4096 |              0
 postgres | edbstore | dept_pk                  | index          |       0 |          4096 |              0
 postgres | edbstore | emp                      | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | emp_pk                   | index          |       2 |          4096 |  0.00048828125
 postgres | edbstore | inventory                | ordinary table |      59 |          4096 | 0.014404296875
 postgres | edbstore | inventory_pkey           | index          |       0 |          4096 |              0
 postgres | edbstore | ix_cust_hist_customerid  | index          |      32 |          4096 |      0.0078125
 postgres | edbstore | ix_cust_username         | index          |       0 |          4096 |              0
 postgres | edbstore | ix_order_custid          | index          |       0 |          4096 |              0
 postgres | edbstore | ix_orderlines_orderid    | index          |       0 |          4096 |              0
 postgres | edbstore | ix_prod_category         | index          |       0 |          4096 |              0
 postgres | edbstore | ix_prod_special          | index          |       0 |          4096 |              0
 postgres | edbstore | job_grd                  | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | jobhist                  | ordinary table |       5 |          4096 | 0.001220703125
 postgres | edbstore | jobhist_pk               | index          |       0 |          4096 |              0
 postgres | edbstore | locations                | ordinary table |       0 |          4096 |              0
 postgres | edbstore | next_empno               | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | orderlines               | ordinary table |     389 |          4096 | 0.094970703125
 postgres | edbstore | orders                   | ordinary table |     104 |          4096 |    0.025390625
 postgres | edbstore | orders_orderid_seq       | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | orders_pkey              | index          |       0 |          4096 |              0
 postgres | edbstore | products                 | ordinary table |     105 |          4096 | 0.025634765625
 postgres | edbstore | products_pkey            | index          |       0 |          4096 |              0
 postgres | edbstore | products_prod_id_seq     | sequence       |       1 |          4096 | 0.000244140625
 postgres | edbstore | reorder                  | ordinary table |       0 |          4096 |              0
 postgres | edbstore | salesemp                 | view           |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12450           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12450_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12455           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12455_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12460           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12460_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12465           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12465_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12470           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12470_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12475           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12475_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12480           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_12480_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_1255            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_1255_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_16523           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_16523_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_16541           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_16541_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_16551           | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_16551_index     | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2396            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2396_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2604            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2604_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2606            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2606_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2609            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2609_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2618            | TOAST table    |       7 |          4096 | 0.001708984375
 postgres | pg_toast | pg_toast_2618_index      | index          |       2 |          4096 |  0.00048828125
 postgres | pg_toast | pg_toast_2619            | TOAST table    |      11 |          4096 | 0.002685546875
 postgres | pg_toast | pg_toast_2619_index      | index          |       2 |          4096 |  0.00048828125
 postgres | pg_toast | pg_toast_2620            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2620_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2964            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_2964_index      | index          |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_3596            | TOAST table    |       0 |          4096 |              0
 postgres | pg_toast | pg_toast_3596_index      | index          |       0 |          4096 |              0
 postgres | public   | pg_buffercache           | view           |       0 |          4096 |              0
(75 rows)
</pre>


    <h3>Dashboard
 Query - Version 2</h3>
      <p>See notes about the versions in section 'Base Query - VERSION 2'</p>
      <p>This is the same as the base query except perc_buffers is multiplied by 100 to show percentage as a fraction of 100 rather than a fraction of 1 (for example .204 changed to 20.4)</p>

      <h4>SQL</h4>
<pre>
-- Each object and how many buffers they are occupying
-- Requires EXTENSION pg_buffercache (execute 'CREATE EXTENSION IF NOT EXISTS pg_buffercache;')
SELECT current_database() AS database,
       pg_namespace.nspname AS schema,
       pg_class.relname AS relation,
       CASE pg_class.relkind
           WHEN 'r' THEN CAST('ordinary table' AS varchar(14))
           WHEN 'i' THEN CAST('index' AS varchar(14))
           WHEN 'S' THEN CAST('sequence' AS varchar(14))
           WHEN 'v' THEN CAST('view' AS varchar(14))
           WHEN 'c' THEN CAST('composite type' AS varchar(14))
           WHEN 't' THEN CAST('TOAST table' AS varchar(14))
           WHEN 'f' THEN CAST('foreign table' AS varchar(14))
           ELSE CAST(pg_class.relkind AS varchar(14))
       END as relation_kind,
       count(pg_buffercache.bufferid) AS buffers,
       CAST(
              (SELECT setting
               FROM pg_settings
               WHERE name='shared_buffers') AS integer) AS total_buffers,
       100.00 * ((count(pg_buffercache.bufferid))/(CAST (
                                                           (SELECT setting
                                                            FROM pg_settings
                                                            WHERE name='shared_buffers') AS real))) AS perc_buffers
FROM pg_class
INNER JOIN pg_namespace ON (pg_namespace.oid = pg_class.relnamespace)
LEFT OUTER JOIN pg_buffercache ON (pg_buffercache.relfilenode = pg_class.relfilenode)
AND pg_buffercache.reldatabase IN
  (SELECT oid
   FROM pg_database
   WHERE datname = current_database())
WHERE pg_namespace.nspname NOT IN ('pg_catalog',
                                   'information_schema')
GROUP BY 1,2,3,4
ORDER BY 1,2,3;
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |         relation         | relation_kind  | buffers | total_buffers | perc_buffers 
----------+----------+--------------------------+----------------+---------+---------------+--------------
 postgres | edbstore | categories               | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | categories_category_seq  | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | categories_pkey          | index          |       0 |          4096 |            0
 postgres | edbstore | cust_hist                | ordinary table |     331 |          4096 | 8.0810546875
 postgres | edbstore | customers                | ordinary table |     492 |          4096 |  12.01171875
 postgres | edbstore | customers_customerid_seq | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | customers_pkey           | index          |       0 |          4096 |            0
 postgres | edbstore | dept                     | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | dept_dname_uq            | index          |       0 |          4096 |            0
 postgres | edbstore | dept_pk                  | index          |       0 |          4096 |            0
 postgres | edbstore | emp                      | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | emp_pk                   | index          |       2 |          4096 |  0.048828125
 postgres | edbstore | inventory                | ordinary table |      59 |          4096 | 1.4404296875
 postgres | edbstore | inventory_pkey           | index          |       0 |          4096 |            0
 postgres | edbstore | ix_cust_hist_customerid  | index          |      32 |          4096 |      0.78125
 postgres | edbstore | ix_cust_username         | index          |       0 |          4096 |            0
 postgres | edbstore | ix_order_custid          | index          |       0 |          4096 |            0
 postgres | edbstore | ix_orderlines_orderid    | index          |       0 |          4096 |            0
 postgres | edbstore | ix_prod_category         | index          |       0 |          4096 |            0
 postgres | edbstore | ix_prod_special          | index          |       0 |          4096 |            0
 postgres | edbstore | job_grd                  | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | jobhist                  | ordinary table |       5 |          4096 | 0.1220703125
 postgres | edbstore | jobhist_pk               | index          |       0 |          4096 |            0
 postgres | edbstore | locations                | ordinary table |       0 |          4096 |            0
 postgres | edbstore | next_empno               | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | orderlines               | ordinary table |     389 |          4096 | 9.4970703125
 postgres | edbstore | orders                   | ordinary table |     104 |          4096 |    2.5390625
 postgres | edbstore | orders_orderid_seq       | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | orders_pkey              | index          |       0 |          4096 |            0
 postgres | edbstore | products                 | ordinary table |     105 |          4096 | 2.5634765625
 postgres | edbstore | products_pkey            | index          |       0 |          4096 |            0
 postgres | edbstore | products_prod_id_seq     | sequence       |       1 |          4096 | 0.0244140625
 postgres | edbstore | reorder                  | ordinary table |       0 |          4096 |            0
 postgres | edbstore | salesemp                 | view           |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12450           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12450_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12455           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12455_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12460           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12460_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12465           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12465_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12470           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12470_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12475           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12475_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12480           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_12480_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_1255            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_1255_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_16523           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_16523_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_16541           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_16541_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_16551           | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_16551_index     | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2396            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2396_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2604            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2604_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2606            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2606_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2609            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2609_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2618            | TOAST table    |       7 |          4096 | 0.1708984375
 postgres | pg_toast | pg_toast_2618_index      | index          |       2 |          4096 |  0.048828125
 postgres | pg_toast | pg_toast_2619            | TOAST table    |      11 |          4096 | 0.2685546875
 postgres | pg_toast | pg_toast_2619_index      | index          |       2 |          4096 |  0.048828125
 postgres | pg_toast | pg_toast_2620            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2620_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2964            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_2964_index      | index          |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_3596            | TOAST table    |       0 |          4096 |            0
 postgres | pg_toast | pg_toast_3596_index      | index          |       0 |          4096 |            0
 postgres | public   | pg_buffercache           | view           |       0 |          4096 |            0
(75 rows)
</pre>
</s>


  <h2>connections_count_by_state</h2>

    <h3>Overview</h3>
      <p>This monitor provides the number of connections that each database has in each state (active/idle/idle in transaction/etc.). This is mostly useful because it helps us to determine if a connection pooler (such as PgBouncer) should be implemented and helps us determine what a reasonable value of max_connections should be.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 30 minutes. 1 hour may be appropriate. 15 minutes may be appropriate</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Health Check</b></p>
        <ul>
          <li>If 'active' connections count during peak hours is above 350 consider implementing a connection pooler (such as PgBouncer)
</li>
          <li>If there is a large number of 'idle' connections relative to active connections consider implementing a connection pooler (such as PgBouncer) so you can tune max_connections to a lower value</li>
          <li>If there are frequently connections in state 'idle in transaction' complain to the application developers per the concerns listed in monitor 'idle_in_transaction'</li>
          <li>View trends and discuss with the client to tune max_connections to a reasonable value given the trend in connections and expectations for the future</li>
        </ul>


    <h3>SQL Scope</h3>
    <p>Run once per cluster.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT datname AS database,
       state,
       COUNT(pid) AS count_of_connections
FROM pg_stat_activity
GROUP BY 1,2
ORDER BY database;
</pre>
      <h4>SQL Example Result</h4>
<pre>
 database |        state        | count_of_connections 
----------+---------------------+----------------------
 pgp      | active              |                    3
 pgp      | idle                |                    1
 pgp      | idle in transaction |                    1
(3 rows)
</pre>

    <h3>Dashboard
 Query</h3>
    <p>Same as base query</p>



  <h2>cpu_load</h2>

    <h3>Overview</h3>
      <p>This is a standard monitor that is included by default for every monitoring tool. We just use whatever is standard for for the monitoring tool.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p>None</p>


    <h3>bash Command</h3>
      <p>Whatever is standard for the monitoring tool. It is usually better to use an OS API than a bash command because 'top -bn1' is innacurate and 'top -bn2' takes a long time to return. OS APIs are usually accurate and faster than 'top -bn2'.</p>



  <h2>database_stats (includes database_size/cache_hit_ratio/commit_ratio)</h2>

    <h3>Overview</h3>
      <p>From pg_stat_database for each database we list the database size, the settings that determine what statistics are tracked (settings: track_activities, track_counts, and track_io_timing), the number of rollbacks, commits, divide commits by (commits + rollbacks) to get the commit ratio, divide the blocks hit in buffer cache by (blocks hit + blocks read from disk) to get the cache hit ratio, add the count of rows returned + count inserted + count updated + count deleted to get "row activity", list of rows returned by SELECT queries, ratio of rows returned by select queries to "row activity", count inserted rows, ratio of rows inserted to "row activity", count updated rows, ratio of rows updated to "row activity", count deleted rows, ratio of rows deleted to "row activity", add the time spend reading blocks to the time writing blocks to get "blk read write time", time spend reading blocks, ratio of time spend reading blocks to "blk read write time", time spent writing blocks, ratio of time spent writing blocks to "blk read write time", number of temp files, total size of temp files, number of deadlocks there have been, and when these stats were last reset</p>
      <p><b>Issue:</b> the information in this pg_stat_database view is not preserved in major version upgrades. You can find when these were last reset via the 'stats_reset' field.</p>
      <p><b>Important value: </b>database_size</p>
      <p><b>Important value: </b>cache_hit_ratio</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>once a day</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>

    <h3>Alert Response</h3>

    <h3>cache_hit_ratio</h3>

      <h4>Overview</h4>
        <p>The performance of an OLTP database is highly dependent on the working data set being contained in the buffer cache (RAM). To measure the percentage of the working data set that is (usually) contained in the buffer cache we divide the number of blocks read from buffer cache/(blocks read from disk + blocks read from buffers). For a healthy database we try to tune this value as close to .9 (90%) as possible. To increase the amount of working set that is contained in the buffer cache the DBA can increase the size of shared_buffers, can add indexes to prevent queries from doing full page scans (which evict other data from buffers to make space for the table scan), and the application developers can tune their queries to be more selective (for the same reason).</p>
        <p>Note that this is for OLTP only: OLAP databases typically have more random access patterns which are unlikely to be in the buffer cache. For these we already increase shared_buffers to be extremely high because we know in advance that the read/write ratio will be highly skewed to read. It would be helpful to make sure we have indexes set up where needed but we are monitoring that separately via index_hit_ratio</p>


      <h4>Alert Response</h4>
        <p><b>Health Check</b></p>
          <ul>
            <li>Tune as close to .9 (90%) as possible</li>
            <li>Ensure that shared_buffers is set right and increase as needed</li>
              <ul>
                <li>We normally cap shared_buffers at 8gb, but that "rule" is only true for high write workloads. We limit this to avoid potential write storms at checkpoint time so</li>
                <ul>
                  <li>The more your read/write ratio skews to read, the more you can go over 8gb</li>
                  <li>If your I/O is extremely fast you can go over it too</li>
                </ul>
              </ul>
            <li>Inspect the buffers and see what it contains and confirm you are not frequently evicting data from the buffers because a query does a full table scan</li>
              <ul>
                <li>Add indexes or improve/tune the queries as needed</li>
                  <ul><li>The index_hit_ratio may help</li></ul>
              </ul>
          </ul>


    <h3>database_size</h3>

      <h4>Overview</h4>
        <p>We track database size for reports. To some extent covered by 'pgdata_directory_size' and 'tablespace_directory_size'. For each database we collect the database size in bytes (pg_database_size). This will allow us to project future database growth and ensure there is enough hardware resources to support it.</p>


      <h4>Alert Response</h4>
        <p><b>Health Check</b></p>
          <ul>
            <li>Project future database growth and ensure there is enough hardware resources to support it</li>
          </ul>


    <h3>SQL Scope</h3>
      <p>Run once per cluster.</p>

    <h3>SQL Compatibility</h3>
      <p>In PostgreSQL 9.1 the setting 'track_io_timing' did not exist so the following change must be made:</p>
        <ul>
          <li><b>FROM: </b>"(SELECT current_setting('track_io_timing')) AS track_io_timing,"</li>
          <li><b>TO: </b>"(SELECT setting FROM pg_settings WHERE name = 'track_io_timing') AS track_io_timing,"</li>
          <li><b>NOTE: </b>we implemented "(SELECT current_setting('track_io_timing')) AS track_io_timing," because it is faster.</li>
        </ul>

    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT datname AS database,
       pg_database_size(datid) AS database_size,
       pg_size_pretty(pg_database_size(datid)) AS database_size_pretty,
       (SELECT current_setting('track_activities')) AS track_activities,
       (SELECT current_setting('track_counts')) AS track_counts,
       (SELECT current_setting('track_io_timing')) AS track_io_timing,
       numbackends AS database_connection_count,
       xact_commit,
       xact_rollback,
       CASE xact_commit+xact_rollback
           WHEN 0 THEN 1
           ELSE CAST(xact_commit AS real)/(xact_commit+xact_rollback)
       END AS commit_ratio,
       CASE pg_stat_database.blks_read+pg_stat_database.blks_hit
           WHEN 0 THEN 1
           ELSE CAST(pg_stat_database.blks_hit AS real)/(pg_stat_database.blks_read+pg_stat_database.blks_hit)
       END AS cache_hit_ratio,
       (tup_returned+tup_inserted+tup_updated+tup_deleted) AS row_activity,
       tup_returned,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE CAST(tup_returned AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted)
       END AS read_ratio,
       tup_inserted,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE CAST(tup_inserted AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted)
       END AS insert_ratio,
       tup_updated,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE CAST(tup_updated AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted)
       END AS update_ratio,
       tup_deleted,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE CAST(tup_deleted AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted)
       END AS delete_ratio,
       blk_read_time + blk_write_time AS blk_read_write_time,
       blk_read_time,
       CASE blk_read_time+blk_write_time
           WHEN 0 THEN 0
           ELSE CAST(blk_read_time AS real)/(blk_read_time+blk_write_time)
       END AS blk_read_time_ratio,
       blk_write_time,
       CASE blk_read_time+blk_write_time
           WHEN 0 THEN 0
           ELSE CAST(blk_write_time AS real)/(blk_read_time+blk_write_time)
       END AS blk_write_time_ratio,
       temp_files,
       CASE numbackends
           WHEN 0 THEN 0
           ELSE CAST(temp_files AS real)/numbackends
       END AS temp_files_connections_ratio,
       temp_bytes,
       CASE numbackends
           WHEN 0 THEN 0
           ELSE CAST(temp_bytes AS real)/numbackends
       END AS temp_bytes_connections_ratio,
       deadlocks,
       stats_reset
FROM pg_stat_database
WHERE datname NOT IN ('template0',
                      'template1');
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database | database_size | database_size_pretty | track_activities | track_counts | track_io_timing | database_connection_count | xact_commit | xact_rollback |   commit_ratio   |  cache_hit_ratio  | row_activity | tup_returned |    read_ratio     | tup_inserted |    insert_ratio     | tup_updated |    update_ratio     | tup_deleted |    delete_ratio     | blk_read_write_time | blk_read_time | blk_read_time_ratio | blk_write_time | blk_write_time_ratio | temp_files | temp_bytes | temp_bytes_connections_ratio | deadlocks |          stats_reset          
----------+---------------+----------------------+------------------+--------------+-----------------+---------------------------+-------------+---------------+------------------+-------------------+--------------+--------------+-------------------+--------------+---------------------+-------------+---------------------+-------------+---------------------+---------------------+---------------+---------------------+----------------+----------------------+------------+------------+------------------------------+-----------+-------------------------------
 postgres |       7037444 | 6873 kB              | on               | on           | on              |                         0 |           0 |             0 |                0 |                 0 |            0 |            0 |                 0 |            0 |                   0 |           0 |                   0 |           0 |                   0 |                   0 |             0 |                   0 |              0 |                    0 |          0 |          0 |                            0 |         0 | 
 pgp      |      25784084 | 25 MB                | on               | on           | on              |                         1 |        2388 |            28 | 0.98841059602649 | 0.931982772371989 |       694216 |       686315 | 0.988618816045726 |         1073 | 0.00154562844993489 |        1530 | 0.00220392500316904 |        5298 | 0.00763163050116967 |             244.604 |       244.604 |    1.00000001596969 |              0 |                    0 |          0 |          0 |                            0 |         0 | 2015-06-17 15:23:13.649375-07
(2 rows)
</pre>

      <h4>SQL Example Results Pretty</h4>
<pre>
-[ RECORD 1 ]----------------+------------------------------
database                     | postgres
database_size                | 7037444
database_size_pretty         | 6873 kB
track_activities             | on
track_counts                 | on
track_io_timing              | on
database_connection_count    | 0
xact_commit                  | 0
xact_rollback                | 0
commit_ratio                 | 1
cache_hit_ratio              | 1
row_activity                 | 0
tup_returned                 | 0
read_ratio                   | 0
tup_inserted                 | 0
insert_ratio                 | 0
tup_updated                  | 0
update_ratio                 | 0
tup_deleted                  | 0
delete_ratio                 | 0
blk_read_write_time          | 0
blk_read_time                | 0
blk_read_time_ratio          | 0
blk_write_time               | 0
blk_write_time_ratio         | 0
temp_files                   | 0
temp_bytes                   | 0
temp_bytes_connections_ratio | 0
deadlocks                    | 0
stats_reset                  | 
-[ RECORD 2 ]----------------+------------------------------
database                     | pgp
database_size                | 25784084
database_size_pretty         | 25 MB
track_activities             | on
track_counts                 | on
track_io_timing              | on
database_connection_count    | 1
xact_commit                  | 2393
xact_rollback                | 28
commit_ratio                 | 0.988434531185461
cache_hit_ratio              | 0.932156646442361
row_activity                 | 695276
tup_returned                 | 687375
read_ratio                   | 0.98863616750758
tup_inserted                 | 1073
insert_ratio                 | 0.00154327202434717
tup_updated                  | 1530
update_ratio                 | 0.00220056495549969
tup_deleted                  | 5298
delete_ratio                 | 0.00761999551257342
blk_read_write_time          | 244.604
blk_read_time                | 244.604
blk_read_time_ratio          | 1.00000001596969
blk_write_time               | 0
blk_write_time_ratio         | 0
temp_files                   | 0
temp_bytes                   | 0
temp_bytes_connections_ratio | 0
deadlocks                    | 0
stats_reset                  | 2015-06-17 15:23:13.649375-07
</pre>


    <h3>Dashboard
 Query</h3>

    <h3>SQL Compatibility</h3>
      <p>In PostgreSQL 9.1 the setting 'track_io_timing' did not exist so the following change must be made:</p>
        <ul>
          <li><b>FROM: </b>"(SELECT current_setting('track_io_timing')) AS track_io_timing,"</li>
          <li><b>TO: </b>"(SELECT setting FROM pg_settings WHERE name = 'track_io_timing') AS track_io_timing,"</li>
          <li><b>NOTE: </b>we implemented "(SELECT current_setting('track_io_timing')) AS track_io_timing," because it is faster.</li>
        </ul>

      <h4>SQL</h4>
<pre>
SELECT datname AS database,
       pg_database_size(datid) AS database_size,
       pg_size_pretty(pg_database_size(datid)) AS database_size_pretty,
       (SELECT current_setting('track_activities')) AS track_activities,
       (SELECT current_setting('track_counts')) AS track_counts,
       (SELECT current_setting('track_io_timing')) AS track_io_timing,
       numbackends AS database_connection_count,
       xact_commit+xact_rollback AS commit_rollback_count,
       CASE xact_commit+xact_rollback
           WHEN 0 THEN 100
           ELSE 100.00 * (CAST(xact_commit AS real)/(xact_commit+xact_rollback))
       END AS commit_ratio,
       CASE pg_stat_database.blks_read+pg_stat_database.blks_hit
           WHEN 0 THEN 100
           ELSE 100.00 * (CAST(pg_stat_database.blks_hit AS real)/(pg_stat_database.blks_read+pg_stat_database.blks_hit))
       END AS cache_hit_ratio,
       (tup_returned+tup_inserted+tup_updated+tup_deleted) AS row_activity,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(tup_returned AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted))
       END AS read_ratio,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(tup_inserted AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted))
       END AS insert_ratio,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(tup_updated AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted))
       END AS update_ratio,
       CASE (tup_returned+tup_inserted+tup_updated+tup_deleted)
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(tup_deleted AS real)/(tup_returned+tup_inserted+tup_updated+tup_deleted))
       END AS delete_ratio,
       blk_read_time + blk_write_time AS blk_read_write_time,
       CASE blk_read_time+blk_write_time
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(blk_read_time AS real)/(blk_read_time+blk_write_time))
       END AS blk_read_time_ratio,
       CASE blk_read_time+blk_write_time
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(blk_write_time AS real)/(blk_read_time+blk_write_time))
       END AS blk_write_time_ratio,
       temp_files,
       CASE numbackends
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(temp_files AS real)/numbackends)
       END AS temp_files_connections_ratio,
       temp_bytes,
       CASE numbackends
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(temp_bytes AS real)/numbackends)
       END AS temp_bytes_connections_ratio,
       deadlocks,
       stats_reset
FROM pg_stat_database
WHERE datname NOT IN ('template0',
                      'template1');
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database | database_size | database_size_pretty | track_activities | track_counts | track_io_timing | database_connection_count | commit_rollback_count |   commit_ratio   | cache_hit_ratio  | row_activity |    read_ratio    |   insert_ratio    |   update_ratio    |   delete_ratio    | blk_read_write_time | blk_read_time_ratio | blk_write_time_ratio | temp_files | temp_bytes | temp_bytes_connections_ratio | deadlocks |          stats_reset          
----------+---------------+----------------------+------------------+--------------+-----------------+---------------------------+-----------------------+------------------+------------------+--------------+------------------+-------------------+-------------------+-------------------+---------------------+---------------------+----------------------+------------+------------+------------------------------+-----------+-------------------------------
 postgres |       7037444 | 6873 kB              | on               | on           | on              |                         0 |                     0 |                0 |                0 |            0 |                0 |                 0 |                 0 |                 0 |                   0 |                   0 |                    0 |          0 |          0 |                            0 |         0 | 
 pgp      |      25784084 | 25 MB                | on               | on           | on              |                         1 |                  2431 | 98.8482106129165 | 93.2463391716551 |       699516 | 98.8705047489979 | 0.153391773740701 | 0.218722659667542 | 0.757380817593879 |             244.669 |    100.000002594385 |                    0 |          0 |          0 |                            0 |         0 | 2015-06-17 15:23:13.649375-07
(2 rows)
</pre>

      <h4>SQL Example Results Pretty</h4>
<pre>
-[ RECORD 1 ]----------------+------------------------------
database                     | postgres
database_size                | 7037444
database_size_pretty         | 6873 kB
track_activities             | on
track_counts                 | on
track_io_timing              | on
database_connection_count    | 0
commit_rollback_count        | 0
commit_ratio                 | 0
cache_hit_ratio              | 0
row_activity                 | 0
read_ratio                   | 0
insert_ratio                 | 0
update_ratio                 | 0
delete_ratio                 | 0
blk_read_write_time          | 0
blk_read_time_ratio          | 0
blk_write_time_ratio         | 0
temp_files                   | 0
temp_bytes                   | 0
temp_bytes_connections_ratio | 0
deadlocks                    | 0
stats_reset                  | 
-[ RECORD 2 ]----------------+------------------------------
database                     | pgp
database_size                | 25784084
database_size_pretty         | 25 MB
track_activities             | on
track_counts                 | on
track_io_timing              | on
database_connection_count    | 1
commit_rollback_count        | 2428
commit_ratio                 | 98.8467874794069
cache_hit_ratio              | 93.2373994227048
row_activity                 | 698432
read_ratio                   | 98.8687517181343
insert_ratio                 | 0.153629845138825
update_ratio                 | 0.219062127737561
delete_ratio                 | 0.758556308989279
blk_read_write_time          | 244.604
blk_read_time_ratio          | 100.000001596969
blk_write_time_ratio         | 0
temp_files                   | 0
temp_bytes                   | 0
temp_bytes_connections_ratio | 0
deadlocks                    | 0
stats_reset                  | 2015-06-17 15:23:13.649375-07
</pre>



  <h2>disk_io</h2>

    <h3>Overview</h3>
      <p>Monitor the disk-write load. This is a standard monitor that is included by default for every monitoring tool. We just use whatever is standard for for the monitoring tool. We need this value so we can tune the size of shared_buffers and checkpoints to avoid stressing I/O during checkpoints.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Health Check</b></p>
        <ul>
          <li>Tune the buffer cache size and checkpointing settings to get the best possible I/O utilization and avoid stressing it</li>
        </ul>


    <h3>bash Command</h3>
      <p>Whatever is standard for the monitoring tool.</p>



  <h2>index_hit_ratio</h2>

    <h3>Overview</h3>
      <p>We track the percentage of queries that are scanning indexes vs performing sequential scans so we can ensure indexes are set up where needed. Index use by queries is critical for the following reasons:</p>
      <ul>
        <li>The sorting of index values allows very CPU efficient lookups</li>
        <li>Using indexes allows the database to limit the amount of data that must be read from disk</li>
          <ul>
            <li>This greatly improves query performance because retreiving data from disk is very slow</li>
            <li>This reduces the load on disk resources which allows those resources to be used by other queries</li>
            <li>This reduces the amount of data that must be evicted from the buffer cache to make room for data read from disk</li>
          </ul>
      </ul>
      <p>We calculate this value by dividing the count of index scans/(the count of sequential scans + the count of index scans) of tables larger 5.12kb. Tables smaller than 5.12kb are excluded because for very small tables the optimizer may prefer sequential scan even if there is an index. To increase the number of queries using indexes the DBA can  rebuild indexes and update stats to increase the likelyhood that the optimizer will use it, and investigate query changes with customer and see if other indexes may be needed</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>once a day</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Health Check</b></p>
        <ul>
          <li>For OLTP databases tune as close to .95 (95%) as possible. For OLAP databases tune as high as possible but expect a lower value.</li>
            <ul>
              <li>OLTP databases typically have consistent queries which can be highly optimized</li>
              <li>OLAP databases tend to be more random access and index needs inconsistent</li>
            </ul>
          <li>If usage is trending downward, rebuild indexes and update stats to increase the likelyhood that the optimizer will use it, and investigate query changes with customer and see if other indexes may be needed</li>
          <li>Execute the following SQL to identify the problem tables. It will list tables with the percent of queries using an index, ordered by lowest percentage first. You can then grep the logs for queries that use that table and tune them to be selective and add indexes where needed:</li>
        </ul>
<pre>
-- List tables with the percent of queries using an index, ordered by lowest percentage first
SELECT current_catalog AS database,
       schemaname,
       relname,
       idx_scan,
       seq_scan,
       (100 * idx_scan::float / (seq_scan + idx_scan)) AS perc_idx_used
FROM pg_stat_user_tables
WHERE pg_relation_size(relid)>(5*8192)
  AND NOT ((idx_scan=0
            OR idx_scan=NULL)
           AND seq_scan=0)
ORDER BY perc_idx_used;
</pre>


    <h3>SQL Scope</h3>
      <p>Must be run once per database.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       CAST(SUM(idx_scan) AS real)/ (SUM(idx_scan)+SUM(seq_scan)) AS index_hit_rate
FROM pg_stat_user_tables
WHERE pg_relation_size(relid)>(5*8192)
  AND NOT ((idx_scan=0
            OR idx_scan=NULL)
           AND seq_scan=0);
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |     index_hit_rate     
----------+----------------------
 pgp      | 0.730434782608695652
(1 row)
</pre>


    <h3>Dashboard
 Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       (100 * SUM(idx_scan)/ (SUM(idx_scan)+SUM(seq_scan))) AS index_hit_rate
FROM pg_stat_user_tables
WHERE pg_relation_size(relid)>(5*8192)
  AND NOT ((idx_scan=0
            OR idx_scan=NULL)
           AND seq_scan=0);
</pre>


      <h4>SQL Example Results</h4>
<pre>
 database |     index_hit_rate     
----------+---------------------
 pgp      | 73.0434782608695652
(1 row)
</pre>



  <h2>index_pages_row_ratio</h2>

    <h3>Overview</h3>
      <p>B-tree index pages that have become completely empty are reclaimed by vacuum for re-use. However, if all but a few index keys on a page have been deleted, the page remains allocated. Therefore, a usage pattern in which most, but not all, keys in each range are eventually deleted will see poor use of space. For such usage patterns, periodic reindexing is recommended. In this monitor we track, for each index, the number of pages the index is using divided by the number of rows the index has (the number of rows in the </p>

      <p><b>References:</b></p>
      <ul>
        <li>http://www.postgresql.org/docs/current/static/routine-reindex.html</li>
      </ul>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>Once a day</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Trending</b></p>
        <ul>
          <li>If you see this value increasing a lot over time and Autovacuum is tuned properly you may have this kind of bloat which cannot be resolved by normal vacuum. Rebuild or REINDEX the index as described in the document "Routine Reindexing"</li>
        </ul>
      <p><b>Health Check</b></p>
        <ul>
          <li>Review trends and make recommendations/ take action as necessary</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>Must be run once per database.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_database() AS database,
       pg_namespace.nspname AS schema,
       pg_class.relname AS index,
       pg_class.relpages AS idxpages,
       pg_class.reltuples,
       CASE pg_class.reltuples
           WHEN 0 THEN 0
           ELSE (CAST(pg_class.relpages AS real) / pg_class.reltuples)
       END AS index_pages_row_ratio
FROM pg_index
INNER JOIN pg_class ON (pg_class.oid = pg_index.indexrelid)
INNER JOIN pg_namespace ON (pg_namespace.oid = pg_class.relnamespace)
WHERE pg_namespace.nspname NOT IN ('pg_catalog',
                                   'information_schema',
                                   'pg_toast');
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |          index          | idxpages | reltuples | index_pages_row_ratio 
----------+----------+-------------------------+----------+-----------+-----------------------
 pgp      | edbstore | emp_pk                  |        2 |        14 |              0.142857
 pgp      | edbstore | jobhist_pk              |        2 |        17 |              0.117647
 pgp      | edbstore | categories_pkey         |        2 |        16 |                 0.125
 pgp      | edbstore | customers_pkey          |       57 |     19301 |            0.00295321
 pgp      | edbstore | ix_cust_username        |       78 |     19301 |            0.00404124
 pgp      | edbstore | ix_orderlines_orderid   |      167 |     60350 |            0.00276719
 pgp      | edbstore | inventory_pkey          |       30 |     10000 |                 0.003
 pgp      | edbstore | products_pkey           |       30 |     10000 |                 0.003
 pgp      | edbstore | ix_prod_category        |       30 |     10000 |                 0.003
 pgp      | edbstore | ix_prod_special         |       30 |     10000 |                 0.003
 pgp      | edbstore | orders_pkey             |       35 |     12000 |            0.00291667
 pgp      | edbstore | ix_order_custid         |       35 |     12000 |            0.00291667
 pgp      | edbstore | dept_dname_uq           |        2 |         4 |                   0.5
 pgp      | edbstore | dept_pk                 |        2 |         4 |                   0.5
 pgp      | edbstore | ix_cust_hist_customerid |      167 |     56772 |            0.00294159
(15 rows)
</pre>


    <h3>Dashboard
 Query</h3>
      <p>Same as base except index_pages_row_ratio is as a fraction of 100 rather than of 1</p>

      <h4>SQL</h4>
<pre>
SELECT current_database() AS database,
       pg_namespace.nspname AS schema,
       pg_class.relname AS index,
       pg_class.relpages AS idxpages,
       pg_class.reltuples,
       CASE pg_class.reltuples
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(pg_class.relpages AS real) / pg_class.reltuples)
       END AS index_pages_row_ratio
FROM pg_index
INNER JOIN pg_class ON (pg_class.oid = pg_index.indexrelid)
INNER JOIN pg_namespace ON (pg_namespace.oid = pg_class.relnamespace)
WHERE pg_namespace.nspname NOT IN ('pg_catalog',
                                   'information_schema',
                                   'pg_toast');
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |          index          | idxpages | reltuples | index_pages_row_ratio 
----------+----------+-------------------------+----------+-----------+-----------------------
 pgp      | edbstore | emp_pk                  |        2 |        14 |      14.2857149243355
 pgp      | edbstore | jobhist_pk              |        2 |        17 |      11.7647059261799
 pgp      | edbstore | categories_pkey         |        2 |        16 |                  12.5
 pgp      | edbstore | customers_pkey          |       57 |     19301 |     0.295321480371058
 pgp      | edbstore | ix_cust_username        |       78 |     19301 |     0.404124148190022
 pgp      | edbstore | ix_orderlines_orderid   |      167 |     60350 |     0.276719126850367
 pgp      | edbstore | inventory_pkey          |       30 |     10000 |     0.300000002607703
 pgp      | edbstore | products_pkey           |       30 |     10000 |     0.300000002607703
 pgp      | edbstore | ix_prod_category        |       30 |     10000 |     0.300000002607703
 pgp      | edbstore | ix_prod_special         |       30 |     10000 |     0.300000002607703
 pgp      | edbstore | orders_pkey             |       35 |     12000 |      0.29166666790843
 pgp      | edbstore | ix_order_custid         |       35 |     12000 |      0.29166666790843
 pgp      | edbstore | dept_dname_uq           |        2 |         4 |                    50
 pgp      | edbstore | dept_pk                 |        2 |         4 |                    50
 pgp      | edbstore | ix_cust_hist_customerid |      167 |     56772 |      0.29415909666568
(15 rows)
</pre>



  <h2>rarely_used_indexes</h2>

    <h3>Overview</h3>
      <p>List each index and the percent of queries against its table which use it. The results are sorted by the percentage of queries against that table that use the index from lowest to highest. Note that index use is expected to be less for OLAP databases which tend to be more random-access and may have indexes that exist to support infrequent queries.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>once a day</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Health Check</b></p>
        <ul>
          <li>For each rarely used index evaluate if it should be removed</li>
        </ul>


    <h3>SQL Scope</h3>
      <p>Must be run once per database.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       pg_stat_user_indexes.schemaname AS schema,
       pg_stat_user_indexes.relname AS table,
       pg_stat_user_indexes.indexrelid,
       pg_stat_user_indexes.indexrelname,
       pg_stat_user_indexes.idx_scan,
       pg_stat_user_tables.seq_scan AS table_seq_scan,
       pg_stat_user_tables.idx_scan AS table_idx_scan,
       (pg_stat_user_tables.seq_scan + pg_stat_user_tables.idx_scan) AS table_scans,
       (CAST(pg_stat_user_indexes.idx_scan AS real)/(pg_stat_user_tables.seq_scan + pg_stat_user_tables.idx_scan)) AS perc_index_used
FROM pg_stat_user_indexes
INNER JOIN pg_stat_user_tables ON pg_stat_user_indexes.relid = pg_stat_user_tables.relid
WHERE pg_relation_size(pg_stat_user_indexes.relid)>(5*8192)
  AND NOT ((pg_stat_user_indexes.idx_scan=0
            OR pg_stat_user_indexes.idx_scan=NULL)
           AND pg_stat_user_tables.seq_scan=0)
ORDER BY perc_index_used;
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    | indexrelid |      indexrelname       | idx_scan | table_seq_scan | table_idx_scan | table_scans | perc_index_used 
----------+----------+------------+------------+-------------------------+----------+----------------+----------------+-------------+-----------------
 pgp      | edbstore | cust_hist  |      16473 | ix_cust_hist_customerid |        0 |              9 |              0 |           9 |               0
 pgp      | edbstore | customers  |      16474 | ix_cust_username        |        0 |              5 |              2 |           7 |               0
 pgp      | edbstore | orderlines |      16476 | ix_orderlines_orderid   |        0 |              2 |              0 |           2 |               0
 pgp      | edbstore | inventory  |      16465 | inventory_pkey          |        0 |              2 |              0 |           2 |               0
 pgp      | edbstore | products   |      16471 | products_pkey           |        0 |              4 |              0 |           4 |               0
 pgp      | edbstore | products   |      16477 | ix_prod_category        |        0 |              4 |              0 |           4 |               0
 pgp      | edbstore | products   |      16478 | ix_prod_special         |        0 |              4 |              0 |           4 |               0
 pgp      | edbstore | orders     |      16475 | ix_order_custid         |        0 |              3 |             29 |          32 |               0
 pgp      | edbstore | customers  |      16457 | customers_pkey          |        2 |              5 |              2 |           7 | 0.2857142857142
 pgp      | edbstore | orders     |      16469 | orders_pkey             |       29 |              3 |             29 |          32 |         0.90625
(10 rows)
</pre>



    <h3>Dashboard
 Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       pg_stat_user_indexes.schemaname AS schema,
       pg_stat_user_indexes.relname AS table,
       pg_stat_user_indexes.indexrelid,
       pg_stat_user_indexes.indexrelname,
       pg_stat_user_indexes.idx_scan,
       pg_stat_user_tables.idx_scan AS table_idx_scan,
       (pg_stat_user_tables.seq_scan + pg_stat_user_tables.idx_scan) AS table_scans,
       100.00 * ((CAST(pg_stat_user_indexes.idx_scan AS real)/(pg_stat_user_tables.seq_scan + pg_stat_user_tables.idx_scan))) AS perc_index_used
FROM pg_stat_user_indexes
INNER JOIN pg_stat_user_tables ON pg_stat_user_indexes.relid = pg_stat_user_tables.relid
WHERE pg_relation_size(pg_stat_user_indexes.relid)>(5*8192)
  AND NOT ((pg_stat_user_indexes.idx_scan=0
            OR pg_stat_user_indexes.idx_scan=NULL)
           AND pg_stat_user_tables.seq_scan=0)
ORDER BY perc_index_used;
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    | indexrelid |      indexrelname       | idx_scan | table_idx_scan | table_scans | perc_index_used 
----------+----------+------------+------------+-------------------------+----------+----------------+-------------+-----------------
 pgp      | edbstore | cust_hist  |      16473 | ix_cust_hist_customerid |        0 |              0 |           9 |               0
 pgp      | edbstore | customers  |      16474 | ix_cust_username        |        0 |              0 |           3 |               0
 pgp      | edbstore | orderlines |      16476 | ix_orderlines_orderid   |        0 |              0 |           2 |               0
 pgp      | edbstore | inventory  |      16465 | inventory_pkey          |        0 |              0 |           2 |               0
 pgp      | edbstore | products   |      16471 | products_pkey           |        0 |              0 |           4 |               0
 pgp      | edbstore | products   |      16477 | ix_prod_category        |        0 |              0 |           4 |               0
 pgp      | edbstore | products   |      16478 | ix_prod_special         |        0 |              0 |           4 |               0
 pgp      | edbstore | customers  |      16457 | customers_pkey          |        0 |              0 |           3 |               0
 pgp      | edbstore | orders     |      16475 | ix_order_custid         |        0 |             29 |          32 |               0
 pgp      | edbstore | orders     |      16469 | orders_pkey             |       29 |             29 |          32 |          90.625
(10 rows)
</pre>



  <h2>table_stats (includes table_pages_row_ratio)</h2>

    <h3>Overview</h3>
      <p>Track the size of each table and activity performed on each table for reports. We take various values from pg_stat_user_tables of n_tup_ins (the number of rows inserted), n_tup_upd (the number of rows updated), n_tup_del  (the number of rows deleted), autoanalyze_count (the number of times the table was autoanalyzed to update optimizer statistics). We also use the pg_relation_size and pg_size_pretty system administration functions to get the size of each table.</p>
      <p><b>Issue:</b> the information in this pg_stat_user_tables view is not preserved in major version upgrades. You can find when these were last reset via the 'stats_reset' field of the 'database_stats' monitor.</p>
      <p>Another useful statistic tracked is 'table_pages_row_ratio' which is the number of pages the table is using divided by the number of (live) rows. It tracks the kind of bloat that normal vacuum cannot handle: normal vacuum can shrink the table only if the trailing page(s) are empty – it just truncates the table file by how many empty pages are at the very end. If you get this kind of bloat you must either do vacuum full (which locks the index) or pg_repack (which requires double the size of the table in available space) or manually move the tuples and vacuum as described in Depesz blog post (http://www.depesz.com/2013/06/21/bloat-removal-by-tuples-moving/)</p>
      <p><b>References:</b></p>
      <ul>
        <li>http://www.postgresql.org/docs/devel/static/monitoring-stats.html#PG-STAT-ALL-TABLES-VIEW</li>
        <li>http://www.postgresql.org/docs/devel/static/functions-admin.html#FUNCTIONS-ADMIN-DBSIZE</li>
          <ul><li>See 'pg_relation_size' and 'pg_size_pretty'</li></ul>
      </ul>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>once a day</li>
        <li><b>Warning: </b></li>
        <li><b>Critical: </b></li>
      </ul>


    <h3>Alert Response</h3>

    <h3>table_pages_row_ratio</h3>

        <h4>Overview</h4>
          <p>'table_pages_row_ratio' which is the number of pages the table is using divided by the number of (live) rows. It tracks the kind of bloat that normal vacuum cannot handle: normal vacuum can shrink the table only if the trailing page(s) are empty – it just truncates the table file by how many empty pages are at the very end. If you get this kind of bloat you must either do vacuum full (which locks the index) or pg_repack (which requires double the size of the table in available space) or manually move the tuples and vacuum as described in Depesz blog post (http://www.depesz.com/2013/06/21/bloat-removal-by-tuples-moving/)</p>

        <h3>Alert Response</h3>
          <p><b>Trending/ Health Check</b></p>
            <ul>
              <li>If you get this kind of bloat you must either do vacuum full (which locks the index) or pg_repack (which requires double the size of the table in available space) or manually move the tuples and vacuum as described in Depesz blog post (http://www.depesz.com/2013/06/21/bloat-removal-by-tuples-moving/)</li>
            </ul>


    <h3>SQL Scope</h3>
      <p>Must be run once per database.</p>


    <h3>Base Query</h3>
      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       pg_stat_user_tables.schemaname AS schema,
       pg_stat_user_tables.relname AS table,
       pg_relation_size(pg_stat_user_tables.relid) AS table_size,
       pg_size_pretty(pg_relation_size(pg_stat_user_tables.relid)) AS table_size_pretty,
       seq_scan,
       seq_tup_read,
       idx_scan,
       idx_tup_fetch,
       (seq_scan + idx_scan) AS table_scan,
       (seq_tup_read + idx_tup_fetch) AS tup_fetched,
       CASE idx_scan+seq_scan
           WHEN 0 THEN 0
           ELSE CAST(idx_scan AS real)/(idx_scan+seq_scan) 
       END AS index_hit_rate,
       n_tup_ins,
       n_tup_upd,
       n_tup_del,
       n_live_tup,
       n_dead_tup,
       n_tup_hot_upd,
       pg_class.relpages,
       CASE n_live_tup
           WHEN 0 THEN 0
           ELSE CAST(pg_class.relpages AS float) / n_live_tup
       END AS table_pages_row_ratio,
       autoanalyze_count,
       CASE heap_blks_hit+heap_blks_read
           WHEN 0 THEN 1
           ELSE CAST(heap_blks_hit AS real)/(heap_blks_hit+heap_blks_read)
       END AS heap_cache_hit_ratio,
       CASE
           WHEN idx_blks_read IS NULL THEN 1
           WHEN idx_blks_hit+idx_blks_read=0 THEN 1
           ELSE CAST(idx_blks_hit AS real)/(idx_blks_hit+idx_blks_read)
       END AS idx_cache_hit_ratio
FROM pg_stat_user_tables
INNER JOIN pg_class ON (pg_class.oid = pg_stat_user_tables.relid)
RIGHT JOIN pg_statio_user_tables ON (pg_stat_user_tables.relid=pg_statio_user_tables.relid)
ORDER BY pg_stat_user_tables.schemaname,
         pg_stat_user_tables.relname;
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    | table_size | table_size_pretty | seq_scan | seq_tup_read | idx_scan | idx_tup_fetch | table_scan | tup_fetched | n_tup_ins | n_tup_upd | n_tup_del | n_tup_hot_upd | n_live_tup | n_dead_tup | relpages | table_pages_row_ratio | autoanalyze_count | heap_cache_hit_ratio | idx_cache_hit_ratio 
----------+----------+------------+------------+-------------------+----------+--------------+----------+---------------+------------+-------------+-----------+-----------+-----------+---------------+------------+------------+----------+-----------------------+-------------------+----------------------+---------------------
 pgp      | edbstore | categories |       8192 | 8192 bytes        |        2 |           32 |        0 |             0 |          2 |          32 |         0 |         0 |         0 |             0 |         16 |          0 |        1 |                0.0625 |                 0 |    0.722222222222222 |                   0
 pgp      | edbstore | cust_hist  |    2678784 | 2616 kB           |        9 |       541986 |        0 |             0 |          9 |      541986 |         0 |         0 |      3578 |             0 |      56789 |          0 |      327 |   0.00575815738963532 |                 0 |    0.889805013927577 |  0.0824742268041237
 pgp      | edbstore | customers  |    3997696 | 3904 kB           |        3 |        60000 |        0 |             0 |          3 |       60000 |         0 |         0 |         0 |             0 |      20000 |          0 |      488 |                0.0244 |                 0 |    0.253180661577608 |                   0
 pgp      | edbstore | dept       |       8192 | 8192 bytes        |        5 |           20 |        0 |             0 |          5 |          20 |         0 |         0 |         0 |             0 |          4 |          0 |        1 |                  0.25 |                 0 |    0.846153846153846 |                 0.6
 pgp      | edbstore | emp        |       8192 | 8192 bytes        |       23 |          322 |        0 |             0 |         23 |         322 |         0 |         0 |         0 |             0 |         14 |          0 |        1 |    0.0714285714285714 |                 0 |    0.923076923076923 |   0.777777777777778
 pgp      | edbstore | inventory  |     450560 | 440 kB            |        2 |        20000 |        0 |             0 |          2 |       20000 |         0 |         0 |         0 |             0 |      10000 |          0 |       55 |                0.0055 |                 0 |    0.180048661800487 |                   0
 pgp      | edbstore | job_grd    |       8192 | 8192 bytes        |        1 |            4 |          |               |            |             |         0 |         0 |         0 |             0 |          4 |          0 |        0 |                     0 |                 0 |    0.742857142857143 |                   0
 pgp      | edbstore | jobhist    |       8192 | 8192 bytes        |        2 |           34 |        0 |             0 |          2 |          34 |         0 |         0 |         0 |             0 |         17 |          0 |        1 |    0.0588235294117647 |                 0 |    0.722222222222222 |                   0
 pgp      | edbstore | locations  |          0 | 0 bytes           |        1 |            0 |          |               |            |             |         0 |         0 |         0 |             0 |          0 |          0 |        0 |                     0 |                 0 |                    0 |                   0
 pgp      | edbstore | orderlines |    3153920 | 3080 kB           |        2 |       120700 |        0 |             0 |          2 |      120700 |         0 |         0 |         0 |             0 |      60350 |          0 |      385 |   0.00637945318972659 |                 0 |    0.148474825431827 |                   0
 pgp      | edbstore | orders     |     819200 | 800 kB            |        3 |        36000 |       29 |            22 |         32 |       36022 |         0 |         0 |         0 |             0 |      12000 |          0 |      100 |   0.00833333333333333 |                 0 |    0.509367681498829 |   0.210843373493976
 pgp      | edbstore | products   |     827392 | 808 kB            |        4 |        40000 |        0 |             0 |          4 |       40000 |         0 |         0 |         0 |             0 |      10000 |          0 |      101 |                0.0101 |                 0 |    0.344385026737968 |                   0
 pgp      | edbstore | reorder    |          0 | 0 bytes           |        1 |            0 |          |               |            |             |         0 |         0 |         0 |             0 |          0 |          0 |        0 |                     0 |                 0 |                    0 |                   0
(13 rows)
</pre>

    <h3>Dashboard
 Query</h3>
      <p>Same as base query except ratios are a fraction of 100 rather than of 1</p>

      <h4>SQL</h4>
<pre>
SELECT current_catalog AS database,
       pg_stat_user_tables.schemaname AS schema,
       pg_stat_user_tables.relname AS table,
       pg_relation_size(pg_stat_user_tables.relid) AS table_size,
       pg_size_pretty(pg_relation_size(pg_stat_user_tables.relid)) AS table_size_pretty,
       seq_scan,
       seq_tup_read,
       idx_scan,
       idx_tup_fetch,
       (seq_scan + idx_scan) AS table_scan,
       (seq_tup_read + idx_tup_fetch) AS tup_fetched,
       CASE idx_scan+seq_scan
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(idx_scan AS real)/(idx_scan+seq_scan))
       END AS index_hit_rate,
       n_tup_ins,
       n_tup_upd,
       n_tup_del,
       n_live_tup,
       n_dead_tup,
       n_tup_hot_upd,
       pg_class.relpages,
       CASE n_live_tup
           WHEN 0 THEN 0
           ELSE 100.00 * (CAST(pg_class.relpages AS float) / n_live_tup)
       END AS table_pages_row_ratio,
       autoanalyze_count,
       CASE heap_blks_hit+heap_blks_read
           WHEN 0 THEN 100
           ELSE 100.00 * (CAST(heap_blks_hit AS real)/(heap_blks_hit+heap_blks_read))
       END AS heap_cache_hit_ratio,
       CASE
           WHEN idx_blks_read IS NULL THEN 100
           WHEN (idx_blks_hit+idx_blks_read)=0 THEN 100
           ELSE 100.00 * (CAST(idx_blks_hit AS real)/(idx_blks_hit+idx_blks_read))
       END AS idx_cache_hit_ratio
FROM pg_stat_user_tables
INNER JOIN pg_class ON (pg_class.oid = pg_stat_user_tables.relid)
RIGHT JOIN pg_statio_user_tables ON (pg_stat_user_tables.relid=pg_statio_user_tables.relid)
ORDER BY pg_stat_user_tables.schemaname,
         pg_stat_user_tables.relname;
</pre>

      <h4>SQL Example Results</h4>
<pre>
 database |  schema  |   table    | table_size | table_size_pretty | seq_scan | seq_tup_read | idx_scan | idx_tup_fetch | table_scan | tup_fetched | n_tup_ins | n_tup_upd | n_tup_del | n_tup_hot_upd | n_live_tup | n_dead_tup | relpages | table_pages_row_ratio | autoanalyze_count | heap_cache_hit_ratio | idx_cache_hit_ratio 
----------+----------+------------+------------+-------------------+----------+--------------+----------+---------------+------------+-------------+-----------+-----------+-----------+---------------+------------+------------+----------+-----------------------+-------------------+----------------------+---------------------
 pgp      | edbstore | categories |       8192 | 8192 bytes        |        2 |           32 |        0 |             0 |          2 |          32 |         0 |         0 |         0 |             0 |         16 |          0 |        1 |                  6.25 |                 0 |     72.2222222222222 |                   0
 pgp      | edbstore | cust_hist  |    2678784 | 2616 kB           |        9 |       541986 |        0 |             0 |          9 |      541986 |         0 |         0 |      3578 |             0 |      56789 |          0 |      327 |     0.575815738963532 |                 0 |     88.9805013927577 |    8.24742268041237
 pgp      | edbstore | customers  |    3997696 | 3904 kB           |        3 |        60000 |        0 |             0 |          3 |       60000 |         0 |         0 |         0 |             0 |      20000 |          0 |      488 |                  2.44 |                 0 |     25.3180661577608 |                   0
 pgp      | edbstore | dept       |       8192 | 8192 bytes        |        5 |           20 |        0 |             0 |          5 |          20 |         0 |         0 |         0 |             0 |          4 |          0 |        1 |                    25 |                 0 |     84.6153846153846 |                  60
 pgp      | edbstore | emp        |       8192 | 8192 bytes        |       23 |          322 |        0 |             0 |         23 |         322 |         0 |         0 |         0 |             0 |         14 |          0 |        1 |      7.14285714285714 |                 0 |     92.3076923076923 |    77.7777777777778
 pgp      | edbstore | inventory  |     450560 | 440 kB            |        2 |        20000 |        0 |             0 |          2 |       20000 |         0 |         0 |         0 |             0 |      10000 |          0 |       55 |                  0.55 |                 0 |     18.0048661800487 |                   0
 pgp      | edbstore | job_grd    |       8192 | 8192 bytes        |        1 |            4 |          |               |            |             |         0 |         0 |         0 |             0 |          4 |          0 |        0 |                     0 |                 0 |     74.2857142857143 |                   0
 pgp      | edbstore | jobhist    |       8192 | 8192 bytes        |        2 |           34 |        0 |             0 |          2 |          34 |         0 |         0 |         0 |             0 |         17 |          0 |        1 |      5.88235294117647 |                 0 |     72.2222222222222 |                   0
 pgp      | edbstore | locations  |          0 | 0 bytes           |        1 |            0 |          |               |            |             |         0 |         0 |         0 |             0 |          0 |          0 |        0 |                     0 |                 0 |                    0 |                   0
 pgp      | edbstore | orderlines |    3153920 | 3080 kB           |        2 |       120700 |        0 |             0 |          2 |      120700 |         0 |         0 |         0 |             0 |      60350 |          0 |      385 |     0.637945318972659 |                 0 |     14.8474825431827 |                   0
 pgp      | edbstore | orders     |     819200 | 800 kB            |        3 |        36000 |       29 |            22 |         32 |       36022 |         0 |         0 |         0 |             0 |      12000 |          0 |      100 |     0.833333333333333 |                 0 |     50.9367681498829 |    21.0843373493976
 pgp      | edbstore | products   |     827392 | 808 kB            |        4 |        40000 |        0 |             0 |          4 |       40000 |         0 |         0 |         0 |             0 |      10000 |          0 |      101 |                  1.01 |                 0 |     34.4385026737968 |                   0
 pgp      | edbstore | reorder    |          0 | 0 bytes           |        1 |            0 |          |               |            |             |         0 |         0 |         0 |             0 |          0 |          0 |        0 |                     0 |                 0 |                    0 |                   0
(13 rows)
</pre>



<h1>PgBouncer</h1>


  <s><h2>connections_pgbouncer</h2></s>
      <p>Not implemented</p>
        <ul><li>Incomplete, requires programming</li></ul>
<s>
    <p>Getting information from PgBouncer on its connections and pools are available by connecting to PgBouncer via psql (psql -p PGBOUNCERPORT -d pgbouncer) and querying PgBouncer system views. However, there is not any actual SQL query engine so ordinary SQL commands cannot be used. So, you can have PgBouncer display the entire contents of its view tables but you cannot manipulate them within the application. External scripting is required (perl, etc.)</p>

    <h3>SHOW CONFIG view</h3>
<pre>
pgbouncer=# SHOW CONFIG;
            key            |              value               | changeable 
---------------------------+----------------------------------+------------
 job_name                  | pgbouncer                        | no
 conffile                  | /vdx/pgp/data/pgbouncer.ini      | yes
 logfile                   | /vdx/pgp/data/pgbouncer.log      | yes
 pidfile                   | /var/run/pgbouncer/pgbouncer.pid | no
 listen_addr               | 127.0.0.1                        | no
 listen_port               | 6432                             | no
 listen_backlog            | 128                              | no
 unix_socket_dir           | /tmp                             | no
 unix_socket_mode          | 511                              | no
 unix_socket_group         |                                  | no
 auth_type                 | md5                              | yes
 auth_file                 | /vdx/pgp/data/userlist.txt       | yes
 pool_mode                 | transaction                      | yes
 max_client_conn           | 100                              | yes
 default_pool_size         | 20                               | yes
 min_pool_size             | 0                                | yes
 reserve_pool_size         | 0                                | yes
 reserve_pool_timeout      | 5                                | yes
 syslog                    | 0                                | yes
 syslog_facility           | daemon                           | yes
 syslog_ident              | pgbouncer                        | yes
 user                      |                                  | no
 autodb_idle_timeout       | 3600                             | yes
 server_reset_query        | DISCARD ALL                      | yes
 server_check_query        | select 1                         | yes
 server_check_delay        | 30                               | yes
 query_timeout             | 0                                | yes
 query_wait_timeout        | 0                                | yes
 client_idle_timeout       | 0                                | yes
 client_login_timeout      | 60                               | yes
 idle_transaction_timeout  | 0                                | yes
 server_lifetime           | 3600                             | yes
 server_idle_timeout       | 600                              | yes
 server_connect_timeout    | 15                               | yes
 server_login_retry        | 15                               | yes
 server_round_robin        | 0                                | yes
 suspend_timeout           | 10                               | yes
 ignore_startup_parameters |                                  | yes
 disable_pqexec            | 0                                | no
 dns_max_ttl               | 15                               | yes
 dns_zone_check_period     | 0                                | yes
 max_packet_size           | 2147483647                       | yes
 pkt_buf                   | 2048                             | no
 sbuf_loopcnt              | 5                                | yes
 tcp_defer_accept          | 1                                | yes
 tcp_socket_buffer         | 0                                | yes
 tcp_keepalive             | 1                                | yes
 tcp_keepcnt               | 0                                | yes
 tcp_keepidle              | 0                                | yes
 tcp_keepintvl             | 0                                | yes
 verbose                   | 0                                | yes
 admin_users               | pgp                              | yes
 stats_users               | stats, pgp                       | yes
 stats_period              | 60                               | yes
 log_connections           | 1                                | yes
 log_disconnections        | 1                                | yes
 log_pooler_errors         | 1                                | yes
(57 rows)
</pre>

    <h3>SHOW POOLS View</h3>
<pre>
pgbouncer=# SHOW POOLS;
 database  |   user    | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait 
-----------+-----------+-----------+------------+-----------+---------+---------+-----------+----------+---------
 pgbouncer | pgbouncer |         1 |          0 |         0 |       0 |       0 |         0 |        0 |       0
 pgp       | pgp       |         0 |          0 |         0 |       0 |       0 |         0 |        0 |       0
(2 rows)
</pre>


    <h3>check_postgres script for check_pgbouncer_backends</h3>
<pre>
sub check_pgbouncer_backends {

    ## Check the number of connections to pgbouncer compared to
    ## max_client_conn
    ## Supports: Nagios, MRTG
    ## It makes no sense to run this more than once on the same cluster
    ## Need to be superuser, else only your queries will be visible
    ## Warning and criticals can take three forms:
    ## critical = 12 -- complain if there are 12 or more connections
    ## critical = 95% -- complain if >= 95% of available connections are used
    ## critical = -5 -- complain if there are only 5 or fewer connection slots left
    ## The former two options only work with simple numbers - no percentage or negative
    ## Can also ignore databases with exclude, and limit with include

    my $warning  = $opt{warning}  || '90%';
    my $critical = $opt{critical} || '95%';
    my $noidle   = $opt{noidle}   || 0;

    ## If only critical was used, remove the default warning
    if ($opt{critical} and !$opt{warning}) {
        $warning = $critical;
    }

    my $validre = qr{^(\-?)(\d+)(\%?)$};
    if ($critical !~ $validre) {
        ndie msg('pgb-backends-users', 'Critical');
    }
    my ($e1,$e2,$e3) = ($1,$2,$3);
    if ($warning !~ $validre) {
        ndie msg('pgb-backends-users', 'Warning');
    }
    my ($w1,$w2,$w3) = ($1,$2,$3);

    ## If number is greater, all else is same, and not minus
    if ($w2 > $e2 and $w1 eq $e1 and $w3 eq $e3 and $w1 eq '') {
        ndie msg('range-warnbig');
    }
    ## If number is less, all else is same, and minus
    if ($w2 < $e2 and $w1 eq $e1 and $w3 eq $e3 and $w1 eq '-') {
        ndie msg('range-warnsmall');
    }
    if (($w1 and $w3) or ($e1 and $e3)) {
        ndie msg('range-neg-percent');
    }

    ## Grab information from the config
    $SQL = 'SHOW CONFIG';

    my $info = run_command($SQL, { regex => qr{\d+}, emptyok => 1 } );

    ## Default values for information gathered
    my $limit = 0;

    ## Determine max_client_conn
    for my $r (@{$info->{db}[0]{slurp}}) {
        if ($r->{key} eq 'max_client_conn') {
            $limit = $r->{value};
            last;
        }
    }

    ## Grab information from pools
    $SQL = 'SHOW POOLS';

    $info = run_command($SQL, { regex => qr{\d+}, emptyok => 1 } );

    $db = $info->{db}[0];

    my $total = 0;
    my $grandtotal = @{$db->{slurp}};

    for my $r (@{$db->{slurp}}) {

        ## Always want perf to show all
        my $nwarn=$w2;
        my $ncrit=$e2;
        if ($e1) {
            $ncrit = $limit-$e2;
        }
        elsif ($e3) {
            $ncrit = (int $e2*$limit/100);
        }
        if ($w1) {
            $nwarn = $limit-$w2;
        }
        elsif ($w3) {
            $nwarn = (int $w2*$limit/100)
        }

        if (! skip_item($r->{database})) {
            my $current = $r->{cl_active} + $r->{cl_waiting};
            $db->{perf} .= " '$r->{database}'=$current;$nwarn;$ncrit;0;$limit";
            $total += $current;
        }
    }

    if ($MRTG) {
        $stats{$db->{dbname}} = $total;
        $statsmsg{$db->{dbname}} = msg('pgb-backends-mrtg', $db->{dbname}, $limit);
        return;
    }

    if (!$total) {
        if ($grandtotal) {
            ## We assume that exclude/include rules are correct, and we simply had no entries
            ## at all in the specific databases we wanted
            add_ok msg('pgb-backends-none');
        }
        else {
            add_unknown msg('no-match-db');
        }
        return;
    }

    my $percent = (int $total / $limit*100) || 1;
    my $msg = msg('pgb-backends-msg', $total, $limit, $percent);
    my $ok = 1;

    if ($e1) { ## minus
        $ok = 0 if $limit-$total <= $e2;
    }
    elsif ($e3) { ## percent
        my $nowpercent = $total/$limit*100;
        $ok = 0 if $nowpercent >= $e2;
    }
    else { ## raw number
        $ok = 0 if $total >= $e2;
    }
    if (!$ok) {
        add_critical $msg;
        return;
    }

    if ($w1) {
        $ok = 0 if $limit-$total <= $w2;
    }
    elsif ($w3) {
        my $nowpercent = $total/$limit*100;
        $ok = 0 if $nowpercent >= $w2;
    }
    else {
        $ok = 0 if $total >= $w2;
    }
    if (!$ok) {
        add_warning $msg;
        return;
    }

    add_ok $msg;

    return;

} ## end of check_pgbouncer_backends
</pre>

    <h3>check_postgres script random pool info</h3>
<pre>
sub check_pgb_pool {

    # Check various bits of the pgbouncer SHOW POOLS ouptut
    my $stat = shift;
    my ($warning, $critical) = validate_range({type => 'positive integer'});

    $SQL = 'SHOW POOLS';
    my $info = run_command($SQL, { regex => qr[$stat] });

    $db = $info->{db}[0];
    my $output = $db->{slurp};
    my $gotone = 0;
    for my $i (@$output) {
        next if skip_item($i->{database});
        my $msg = "$i->{database}=$i->{$stat}";

        if ($MRTG) {
            $stats{$i->{database}} = $i->{$stat};
            $statsmsg{$i->{database}} = msg('pgbouncer-pool', $i->{database}, $stat, $i->{$stat});
            next;
        }

        if ($critical and $i->{$stat} >= $critical) {
            add_critical $msg;
        }
        elsif ($warning and $i->{$stat} >= $warning) {
            add_warning $msg;
        }
        else {
            add_ok $msg;
        }
    }

    return;

} ## end of check_pgb_pool
</pre>
</s>


  <h2>outage_pgbouncer</h2>

    <h3>Overview</h3>
      <p>To ensure PgBouncer is up and users are able to connect we connect via PgBouncer's port and execute a simple 'SELECT 1;'. There is not much we can do with PgBouncer: we can restart it and modify the max_client_conn setting but not much else.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>command fails</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ul>
          <li>Restart PgBouncer and review the PgBouncer logs</li>
          <li>Check the PgBouncer connection limits and adjust as necessary.</li>
        </ul> 


    <h3>SQL Scope</h3>
      <p>Run once per cluster</p>


    <h3>Base Query</h3>

      <h4>SQL</h4>
        <code>SELECT 1;</code>


    <h3>Dashboard
 Query</h3>
      <p>Same as Base Query.</p>



  <h2>pgbouncer_os_process_status</h2>

    <h3>Overview</h3>
      <p>The './pgbouncer' operating system process is the process that runs PgBouncer. We use linux bash to get the count of running processes with text './pgbouncer' to ensure PgBouncer is running.</p>

    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every minute</li>
        <li><b>Warning: </b>none</li>
        <li><b>Critical: </b>the number of PgBouncer processes you have running on the machine - 1. For example, if you have 2 instances of PgBouncer running on the machine you would set critical warning to 1 (which is 2-1). In most cases only 1 PgBouncer instance is running on a machine so this value should usually be 0.</li>
      </ul>

    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>None</li>
        </ul>
      <p><b>Critical</b></p>
        <ul><li>Restart PgBouncer and review the PgBouncer logs</li></ul>

    <h3>bash Command</h3>
<pre>
# Note we subtract 1 because our greps will be returned as well
echo "$(($(ps -ef | grep ./pgbouncer  -c)-1))"
</pre>



<h1>Storage</h1>
  <p>Monitoring tool only (no SQL or bash)</p>


  <h2>filesystem_inodes</h2>

    <h3>Overview</h3>
      <p>PostgreSQL tables are split into 2gb chunks, each requires an inode. Each PgBouncer connection also requires an inode. Monitor the filesystem inodes to ensure there are enough inodes available to support PostgreSQL and PgBouncer.</p>
      <p>We use the monitoring tool's built in measure.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>based on database growth. 10 minutes is probably too often but is a good place to start</li>
        <li><b>Warning: </b>how quickly you can increase the disk space</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Standard storage expansion procedure</li>
        </ul>
      <p><b>Critical</b></p>
        <ul><li>None</li></ul> 


    <h3>bash Command</h3>
      <p>Use the monitoring tool's built in measure.</p>



  <h2>pgdata_directory_size</h2>

    <h3>Overview</h3>
      <p>Monitor the capacity of the filesystem that the database is mounted on (pgdata) to ensure it is not running out of space.</p>
      <p>We use the monitoring tool's built in measure.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>how quickly you can increase the disk space</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Standard storage expansion procedure</li>
        </ul>
      <p><b>Critical</b></p>
        <ul><li>None</li></ul> 


    <h3>bash Command</h3>
      <p>Use the monitoring tool's built in measure. storage bytes/ available bytes</p>



  <h2>tablespace_directory_size</h2>

    <h3>Overview</h3>
      <p>Monitor the capacity of any filesystem that database tablespaces are mounted on (mounts pointed to in the symbolic links located in 'PGDATA/pg_tblspc') to ensure they are not running out of space.</p>
      <p>We use the monitoring tool's built in measure.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>how quickly you can increase the disk space</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Standard storage expansion procedure</li>
        </ul>
      <p><b>Critical</b></p>
        <ul><li>None</li></ul> 


    <h3>bash Command</h3>
      <p>Use the monitoring tool's built in measure. storage bytes/ available bytes</p>



  <h2>xlog_archive_directory_size</h2>

    <h3>Overview</h3>
      <p>Monitor the capacity of the filesystem that the xlogs are being archived to and ensure it is not running out of space. This is very critical because if it fills up the archive command will fail, resulting in an increased 'xlog_directory_size' and a compromised recovery time objective.</p>
      <p>We use the monitoring tool's built in measure.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>how quickly you can increase the disk space</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Standard storage expansion procedure</li>
        </ul>
      <p><b>Critical</b></p>
        <ul><li>None</li></ul> 

    <h3>bash Command</h3>
      <p>Use the monitoring tool's built in measure. storage bytes/ available bytes</p>



  <h2>xlog_directory_size</h2>

    <h3>Overview</h3>
      <p>Monitor the size and capacity of the xlog directory (PGDATA/pg_xlog) to ensure it has available capacity and track changes in size. The size of this directory should not fluctuate unless the xlog archive_command fails or a hot standby server becomes disconnected or falls behind which will cause more xlogs to be retained in the directory.</p>
      <p>We use the monitoring tool's built in measure.</p>


    <h3>Default Frequency/Thresholds</h3>
      <ul>
        <li><b>Frequency: </b>every 10 minutes</li>
        <li><b>Warning: </b>how quickly you can increase the disk space</li>
        <li><b>Critical: </b>none</li>
      </ul>


    <h3>Alert Response</h3>
      <p><b>Warning</b></p>
        <ul>
          <li>Standard storage expansion procedure</li>
          <li>Check 'xlog_readyfiles_age' and see if it is trending upwards. If so check the logs and see if there are errors in the archive_command and remediate</li>
          <li>Check 'hot_standby_send_delay' and see if it is trending upwards. If so investigate why sending xlogs to the hot standby is taking a long time and remediate if possible</li>
        </ul>
      <p><b>Critical</b></p>
        <ul><li>None</li></ul> 


    <h3>bash Command</h3>
      <p>Use the monitoring tool's built in measure. storage bytes/ available bytes</p>



<!--END CONTENT-->
</div>
</body>
</html>
