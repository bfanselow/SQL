# Useful SQL Queries
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
