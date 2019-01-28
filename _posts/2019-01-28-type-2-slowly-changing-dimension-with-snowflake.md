---
layout: post
title: "Working with SCD-Type-II in Snowflake"
subtitle: "Merge and Update"
image: mandlebrot.jpg
date: 2019-01-28
comments: true
---
# Introduction
In this post, we'll see how to deal with a Type 1 and Type 2 Slowly Changing Dimension attributes (SCD) using a Snowflake database.

## What's a SCD

Values in a master record can change on an infrequent basis, for example, in an employee database table then someone's surname may rarely change but their line manager could change more frequently.

In the context of a data mart, we may refer to an employee table as a dimension - and the fact that the data changes infrequently allows us to refer to it as a [Slowly Changing Dimension](https://en.wikipedia.org/wiki/Slowly_changing_dimension).

Sometimes, when a value changes we don't care what the old value was and we just overwrite the old value with the new value - this is known as a Type 1 SCD attribute.

Other times, when a value changes we may want to retain what the original value was and record the date of the change - this is known as a Type 2 SCD attribute.

There are other types - but for this example we'll restrict ourselves to Type 1 and Type 2 SCD's.

# Example
## Employee Database Table
To illustrate how to work with SCD's we'll use a simple scenario of an employee database table where the originating system has the following columns:

- Employee ID
- Employee Surname
- Working Hours
- Line Manager

Let's say that we want to track changes for Surname and Line Manager - but we don't care about changes to Working Hours.

This means that:

- Working Hours is a Type 1 SCD; and
- Surname and Line Manager are Type 2 SCD's

So let's say our initial database table looks like this:

| EMPLOYEE ID | LAST NAME (Type 2) | WORKING HOURS (Type 1) | LINE MANAGER ID (Type 2) |
| --- | --- | --- | --- |
| 100 | Adams | 37.5	| 102 |
| 101 | Burns | 37.5	| 102 |
| 102 | Croft | 37.5 | NULL |

&nbsp;

# Data Mart ETL Process
Now let's say that our Data Warehouse has a LANDING schema where every day our ETL process extracts the employee records from the original system and then loads them into an equivalent table in our LANDING schema.   Let's call that table LANDING.EMPLOYEES.

Our design pattern for the ETL process is to have additional meta-data columns in the LANDING table where we record the extract date, extract time, and extract source.

The ETL process runs overnight -  around 3am to extract the status of the employee table as at the end of the previous day.  So a typical table - representing the employees as at 1st Jan 2019 - might look like this in LANDING.EMPLOYEES

## LANDING.EMPLOYEES table

| EMPLOYEE ID | &nbsp;LAST NAME | &nbsp;WORKING HOURS |  &nbsp;LINE MANAGER ID |  &nbsp;META EXTRACT DATE | &nbsp;META EXTRACT TIME | &nbsp;META SOURCE |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 100 | Adams | 37.5 | 102 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| 101 | Burns | 37.5 | 102 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| 102 | Croft | 37.5 | NULL | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |

Remember that the extract date for 1st Jan 2019 will be recorded as 2nd Jan - since the process is running overnight.

Our Data Warehouse also has a PERSIST schema - which is where we keep our permanent record of our source tables.   It is in the PERSIST schema where we will deal with the Type 2 SCD's and track changes.

To track the changes we'll introduce some new columns to the PERSIST table structure:

1. A hash value - we'll use this as a method to determine if a row has changed.
2. A current record indicator - we'll use this as a flag to denote the rows which represents the latest version of the data.
3. A valid from date - this will denote the date on which the row became valid.
4. A valid to date - this is the expiry date of the record.

Here's what our PERSIST.EMPLOYEES might look like before being updated on 2nd Jan 2019.

## PERSIST.EMPLOYEES table

| TRACKING HASH | EMPLOYEE ID | LAST NAME | WORKING HOURS | LINE MANAGER ID | CURRENT RECORD YN | VALID_FROM | VALID_TO | META EXTRACT DATE | META EXTRACT TIME | META SOURCE |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 34A21DA5... | 100 | Adams | 37.5 | 102 | Y | 1970-01-01 | 9999-12-31 | 2019-01-01 | 03:00:00 | ADP.EMPLOYEES
| 31CBB37C... | 101 | Burns | 37.5 | 102 | Y | 1970-01-01 | 9999-12-31 | 2019-01-01 | 03:00:00 | ADP.EMPLOYEES
| 809EC3D0... | 102 | Croft | 37.5 | NULL | Y | 1970-01-01 | 9999-12-31 | 2019-01-01 | 03:00:00 | ADP.EMPLOYEES

Points to Note:

- The tracking hash uses a function which calculates a 'pretty close to unique' value for a given combination of fields.
- For the valid_from date we're using the start of time as 1st Jan 1970 ... which is the start of time in the world of Unix.
- For the valid_to date we could leave this NULL for a valid record - but if we use a future date then we don't need to test for NULL when we use it in queries.  We'll use 31st Dec 9999 as a date we're unlikely to ever worry about!
- The meta-extract date is "2019-01-01" so this means the recordset is as-at 31st Dec 2018.

# Interlude: The Tracking Hash
Before we move on to the Snowflake update routine, we need to understand the Tracking Hash function.

In Snowflake we're going to use the SHA1 function to generate hashes for the fields we're going to monitor - have a look at [Hash Functions](https://en.wikipedia.org/wiki/Hash_function) to understand what they are.

Using a hash function allows us to compare the hash value of an existing record with the hash value of a new record to determine if there are any value changes - we need only compare the combined hash for all the Type 2 fields - rather than comparing each field value in turn.

We'll create a couple of helper functions which will allow us to create hashes based on a variable number of fields.   Here's the Snowflake function definitions - note that we create our functions in the STAGING schema:
```sql
create or replace function 
    staging.concatenate_for_hash(myArray array)
    returns varchar
  language javascript
  as
  $$
    function return_string(myArray) {
      var my_string = '';
      var arrayLength = myArray.length;
      for (var i = 0; i < arrayLength; i++) {
        if (i>0) my_string += ';' 
        my_string += typeof(myArray[i]) == 'undefined' ? '' : myArray[i] ;
      }
      return my_string.toUpperCase().trim();
    }
  return return_string(MYARRAY)
  $$
;
create or replace function 
    staging.hash_diff(key1 array)  
    returns varchar(40)
    comment='Returns SHA1 Hashkey: usage> hash_diff(array_construct(field1, field2, ...))'
  as 
  $$
    UPPER( SHA1(staging.concatenate_for_hash(key1)) )
  $$
;
```

# Back On Track

Ok, so now we have a hash function defined - we can use this to monitor changes in our SCD Type 2 fields.

We have to run the update in two sweeps. 

1. For the first sweep we update any SCD Type 1 Fields and insert new rows for SCD Type 2 Fields.
2. For the second sweep we expire any old SCD Type 2 rows.

```sql
-- Sweep #1: Update rows where the hash is unchanged and insert rows where the hash doesn't match
MERGE INTO persist.employees T
  USING (
    select staging.hash_diff(
            array_construct(
                employee_id,
                last_name, 
                line_manager_id
            )
        ) as tracking_hash, * 
    from landing.employees
  ) S
  ON T.tracking_hash = S.tracking_hash
  AND T.employee_id = S.employee_id
  AND T.valid_to = to_date('9999-12-31')
  WHEN MATCHED THEN UPDATE SET
    t.last_name=s.last_name, 
    t.working_hours=s.working_hours, 
    t.line_manager_id=s.line_manager_id,
    t.meta_extract_date=s.meta_extract_date, 
    t.meta_extract_time=s.meta_extract_time, 
    t.meta_source=s.meta_source   
  WHEN NOT MATCHED THEN INSERT(
    tracking_hash, employee_id, last_name, working_hours, 
    line_manager_id, current_record_yn, 
    valid_from,
    valid_to,
    meta_extract_date, meta_extract_time, meta_source
  ) 
  VALUES(
    s.tracking_hash, s.employee_id, s.last_name, s.working_hours, 
    s.line_manager_id, 'Y',
    dateadd(day,-1,to_date(s.meta_extract_date)),
    to_date('9999-12-31'),
    s.meta_extract_date, s.meta_extract_time, s.meta_source
  );

-- Sweep #2: Update (expire) rows where we've inserted a replacement row
update persist.employees T
  set 
    T.valid_to = dateadd(day,-2,to_date(S.META_EXTRACT_DATE)), 
    t.current_record_yn = 'N'
  from (
    select staging.hash_diff(
            array_construct(
                employee_id,
                last_name, 
                line_manager_id
            )
        ) as tracking_hash, employee_id, meta_extract_date 
    from landing.employees
  ) S
  where t.employee_id = s.employee_id 
    and t.tracking_hash <> s.tracking_hash 
    and t.valid_to = to_date('9999-12-31')
;
```

So after this merge statement, the only thing changed in the PERSIST.EMPLOYEES table is the extract_date - since we didn't change any of the dimension field values:

## PERSIST.EMPLOYEES table

| TRACKING HASH | EMPLOYEE ID | LAST NAME | WORKING HOURS | LINE MANAGER ID | CURRENT RECORD YN | VALID_FROM | VALID_TO | META EXTRACT DATE | META EXTRACT TIME | META SOURCE |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 34A21DA... | 100 | Adams | 37.5 | 102 | Y | 1970-01-01 | 9999-12-31 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| 31CBB37C... | 101 | Burns | 37.5 | 102 | Y | 1970-01-01 | 9999-12-31 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| 809EC3D0... | 102 | Croft | 37.5 | NULL | Y | 1970-01-01 | 9999-12-31 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |

&nbsp;

## Tracking Changes
Now lets say that our source database has the following changes:

- Employee_id 100 changes their line_manager_id to 101
- Employee_id 101 changes their last_name to Brown
- Employee_id 102 changes their working hours to 30.0

The source table will therefore look like this:

| EMPLOYEE_ID | LAST_NAME | WORKING_HOURS |	LINE_MANAGER_ID |
| --- | --- | --- | --- |
| 100 | Adams | 37.5 | 101 |
| 101 | Brown | 37.5 | 102 |
| 102 | Croft | 30.0 | NULL |

So after our overnight ETL process has run, the LANDING.EMPLOYEES table will look like this:

## LANDING.EMPLOYEES table

| EMPLOYEE ID | &nbsp;LAST NAME | &nbsp;WORKING HOURS |  &nbsp;LINE MANAGER ID |  &nbsp;META EXTRACT DATE | &nbsp;META EXTRACT TIME | &nbsp;META SOURCE |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 100 | Adams | 37.5 | 101 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |
| 101 | Brown | 37.5 | 102 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |
| 102 | Croft | 30.0 | NULL | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |


After running the Merge and Update sequence here's what the PERSIST.EMPLOYEES table looks like

## PERSIST.EMPLOYEES table

| TRACKING HASH | EMPLOYEE ID | LAST NAME | WORKING HOURS | LINE MANAGER ID | CURRENT RECORD YN | VALID_FROM | VALID_TO | META EXTRACT DATE | META EXTRACT TIME | META SOURCE |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 34A21DA... | 100 | Adams | 37.5 | 102 | N | 1970-01-01 | 2019-01-01 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| D73D614... | 100 | Adams | 37.5 | 101 | Y | 2019-01-02 | 9999-12-31 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |
| 31CBB37C... | 101 | Burns | 37.5 | 102 | N | 1970-01-01 | 2019-01-01 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| 664DE022... | 101 | Brown | 37.5 | 102 | Y | 2019-01-02 | 9999-12-31 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |
| 809EC3D0... | 102 | Croft | 30 | NULL | Y | 1970-01-01 | 9999-12-31 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |

So:
- Employee record 100 has changed their line manager from 102 to 101.
- Employee record 101 has changed their name from Burns to Brown.
- And both the above records have a change history saved for them.
- Employee record 102 has changed their working hours - with the existing row being updated.

## One Last Change
Now lets simulate one last change:

- Employee 100 changes their name from Adams to Akers and their working hours to 32.0
- Employee 101 change their name back to Burns.
- Employee 102 is unchanged.

After the table has been loaded to LANDING and then our merge and update has run - here's what the PERSIST table looks like:

## PERSIST.EMPLOYEES table

| TRACKING HASH | EMPLOYEE ID | LAST NAME | WORKING HOURS | LINE MANAGER ID | CURRENT RECORD YN | VALID_FROM | VALID_TO | META EXTRACT DATE | META EXTRACT TIME | META SOURCE |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 34A21DA... | 100 | Adams | 37.5 | 102 | N | 1970-01-01 | 2019-01-01 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| D73D614... | 100 | Adams | 37.5 | 101 | N | 2019-01-02 | 2019-01-02 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |
| E5870EC... | 100 | Akers | 32 | 101 | Y | 2019-01-03 | 9999-12-31 | 2019-01-04 | 03:00:00 | ADP.EMPLOYEES |
| 31CBB37C... | 101 | Burns | 37.5 | 102 | N | 1970-01-01 | 2019-01-01 | 2019-01-02 | 03:00:00 | ADP.EMPLOYEES |
| 664DE022... | 101 | Brown | 37.5 | 102 | N | 2019-01-02 | 2019-01-02 | 2019-01-03 | 03:00:00 | ADP.EMPLOYEES |
| 31CBB37C... | 101 | Burns | 37.5 | 102 | Y | 2019-01-03 | 9999-12-31 | 2019-01-04 | 03:00:00 | ADP.EMPLOYEES |
| 809EC3D0... | 102 | Croft | 30 | NULL | Y | 1970-01-01 | 9999-12-31 | 2019-01-04 | 03:00:00 | ADP.EMPLOYEES |

&nbsp;

## Searching For Changes
Now if we're interested in identifying changes in the last_name field we can use Snowflake's LAG statement to report changed fields:
```sql
select
    *, 
    case 
        when
            last_name <> lag(last_name) 
                            over (partition by employee_id
                                  order by valid_from) 
            then 'Changed' 
        else 'Not changed' 
    end as change_flag
  from persist.employees
  order by employee_id, valid_from;
```

And here's what the resulting query looks like

| EMPLOYEE_ID | LAST_NAME | CURRENT_RECORD_YN | VALID_FROM | VALID_TO | CHANGE_FLAG
|:---:|:---:|:---:|:---:|:---:|:---:|
| 100 | Adams | N | 1970-01-01 | 2019-01-01 | Not changed |
| 100 | Adams | N | 2019-01-02 | 2019-01-02 | Not changed |
| 100 | Akers | Y | 2019-01-03 | 9999-12-31 | Changed |
| 101 | Burns | N | 1970-01-01 | 2019-01-01 | Not changed |
| 101 | Brown | N | 2019-01-02 | 2019-01-02 | Changed |
| 101 | Burns | Y | 2019-01-03 | 9999-12-31 | Changed |
| 102 | Croft | Y | 1970-01-01 | 9999-12-31 | Not changed |

&nbsp;
# Summary

So where we have a Type 2 SCD we now know how to:

- add additional columns with the PERSIST schema to track changes
- determine if a row changes using a hash function
- use a 2 pass merge and update statement to update the PERSIST table
- run a query to see when a field of interest has changed
