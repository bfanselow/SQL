# Useful SQL Queries
## Complex useful queries for common database admin tasks

### Get Duplicate values
Suppose we have a table of usernames and we want to find out which rows have duplicate **username** values
```
SELECT username, COUNT(*) c FROM user_accounts GROUP BY username HAVING c > 1;
```
