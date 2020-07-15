# Useful SQL Queries
---

## Complex useful queries for common database admin tasks

### Get Duplicate values
Suppose we have a table of usernames and we want to find out which rows have duplicate **username** values.
```
SELECT username, COUNT(*) c FROM user_accounts GROUP BY username HAVING c > 1;
```

### Get id gaps
Suppose we have a table with a row that stores an **id** value which is typically sequential, but may have gaps due to deletes. We want to see what those id gaps are.
```
SELECT (t1.id + 1) as gap_starts_at, (SELECT MIN(t3.id) -1 FROM reports_list t3 WHERE t3.id > t1.id) as gap_ends_at FROM reports_list t1 WHERE NOT EXISTS (SELECT t2.id FROM reports_list t2 WHERE t2.id = t1.id + 1) HAVING gap_ends_at IS NOT NULL;
```

---
---

### Suppose we have the following 3 table schemas for managing workflows:
  
#### workflow_requests
```
+----------------------+--------------+------+-----+---------+----------------+
| Field                | Type         | Null | Key | Default | Extra          |
+----------------------+--------------+------+-----+---------+----------------+
| id                   | int(11)      | NO   | PRI | NULL    | auto_increment |
| request_id           | char(32)     | NO   | UNI | NULL    |                |
.... other fields
```

#### workflow_tasks_list  
 * joins to workflow_requests via FK_workflow_request_id
```
+----------------------------+--------------+------+-----+---------+----------------+
| Field                      | Type         | Null | Key | Default | Extra          |
+----------------------------+--------------+------+-----+---------+----------------+
| id                         | int(11)      | NO   | PRI | NULL    | auto_increment |
| FK_workflow_request_id     | int(11)      | NO   | MUL | NULL    |                |
... other fields
```

#### workflow_tasks_logs  
 * joins to workflow_tasks_list via FK_workflow_tasks_list_id
 * joins to workflow_states via FK_workflow_state_id
```
+---------------------------+--------------+------+-----+---------+----------------+
| Field                     | Type         | Null | Key | Default | Extra          |
+---------------------------+--------------+------+-----+---------+----------------+
| id                        | int(11)      | NO   | PRI | NULL    | auto_increment |
| FK_workflow_tasks_list_id | int(11)      | NO   | UNI | NULL    |                |
| FK_workflow_state_id      | int(11)      | NO   | MUL | 1       |                |
... other fields
```

##### where **workflow_states** data is as follows:
```
+----+-------------+---------------------------------------------------------------------------------------------------+
| id | state       | description                                                                                       |
+----+-------------+---------------------------------------------------------------------------------------------------+
|  1 | queued      | Task waiting to be executed. NA for workflow.                                                     |
|  2 | scheduled   | Start of workflow scheduled for triggering on future date. NA for task.                           |
|  3 | inprogress  | Workflow/task currently being executed                                                            |
|  4 | failed      | Task (or one or more tasks in workflow) failed                                                    |
|  5 | completed   | Task (or all tasks in workflow) successfully completed                                            |
|  6 | paused      | Workflow deliberately paused at current task                                                      |
|  7 | cancelled   | Task cancelled - workflow continues; or workflow cancelled - no tasks proceed                     |
|  8 | expired     | Task TTL has exipired. Workflow will not proceed without user interaction                         |
|  9 | waiting     | Interactive task has completed and is waiting for user ack. Workflow is also waiting for user ack |
+----+-------------+---------------------------------------------------------------------------------------------------+
```

#### Get all tasks with cancelled task status for a specific workflow-request id
```
select * from workflow_tasks_log wtlog JOIN workflow_tasks_list wtls on (wtlog.FK_workflow_tasks_list_id = wtls.id) JOIN workflow_requests wr on (wtls.FK_workflow_request_id = wr.id) where wtlog.FK_workflow_state_id = 7 and wr.id = 14693;
```

#### Same as above, but just get the task-log id's
```
select wtlog.id from workflow_tasks_log wtlog JOIN workflow_tasks_list wtls on (wtlog.FK_workflow_tasks_list_id = wtls.id) JOIN workflow_requests wr on (wtls.FK_workflow_request_id = wr.id) where wtlog.FK_workflow_state_id = 7 and wr.id = 14693;
```

#### Now try to update those same rows using this sub-query. You might expect this to work...
```
mysql> update workflow_tasks_log set workflow_tasks_log.FK_workflow_state_id = 1 where workflow_tasks_log.id IN (select wtlog.id from workflow_tasks_log wtlog JOIN workflow_tasks_list wtls on (wtlog.FK_workflow_tasks_list_id = wtls.id) JOIN workflow_requests wr on (wtls.FK_workflow_request_id = wr.id) where wtlog.FK_workflow_state_id = 7 and wr.id = 14693);
ERROR 1093 (HY000): You can't specify target table 'workflow_tasks_log' for update in FROM clause
```

#### Why doesn't that work? In MySQL, you can't modify the same table which you use in the SELECT part.

#### Solution: Wrap the sub-query in an additional sub-select
```
mysql> update workflow_tasks_log set workflow_tasks_log.FK_workflow_state_id = 1 where workflow_tasks_log.id IN ( select id from (select wtlog.id from workflow_tasks_log wtlog JOIN workflow_tasks_list wtls on (wtlog.FK_workflow_tasks_list_id = wtls.id) JOIN workflow_requests wr on (wtls.FK_workflow_request_id = wr.id) where wtlo.FK_workflow_state_id = 7 and wr.id = 14693) as sq );
Query OK, 5 rows affected (3.98 sec)
Rows matched: 5  Changed: 5  Warnings: 0
```

#### Why does the second one work? The query optimizer does a derived merge optimization for the first query (which causes it to fail with the error), but the second query doesn't qualify for the derived merge optimization. Hence the optimizer is forced to execute the subquery first.
