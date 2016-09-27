## laudmariaformysqlusr
###1 Dynamic columns
####1 creating dynamic columns
```
create database dynamic_cols charset utf8;
show variables like '%char%';
show variables like 'default_storage_engine';
create table items(item_id int unsigned not null auto_increment primary key,type varchar(64) not null,
title varchar(255) not null,price decimal(5,2) not null,manufacturer varchar(64) not null,extra blob);
use dynamic_cols;
```
insert
```
insert into items set title='my',type='cd',price='1',manufacturer='cd';
```
[insert into vs set](http://stackoverflow.com/questions/861722/mysql-insert-into-table-values-vs-insert-into-table-set?noredirect=1&lq=1)
create dynamic columns (column_create)
```
insert into items set title='you',type='cd',price='2',manufacturer='ef',extra=column_create('color','red','size','l');
```

####2 querying dynamic columns
######column_get
```
select column_get(extra, 'color' as char) from items where title='you';   //must use `char`, cannot be any another chars
```
######column_exists
```
select title,type,column_exists(extra,'color') from items where 1=1;
```
return 1: this item has the column.  
return 0:  this item has dynamic column,but no 'color' column in dynamic column part.  
return NULL: this item has no dynamic column  

######combine these two
```
select type,title,column_get(extra,'color' as char) from items where column_exists(extra,'color');
select distinct(column_get(extra,'color' as char)) from items where column_exists(extra,'color');
```
######column_list
```
select title,type,column_list(extra) from items where 1=1;
```
######column_json(pretty printing)
```
select title,type,column_json(extra) from items where 1=1;
```
####update dynamic columns
```
update items set extra=column_add(extra, 'color','blue','size','2') where type='cd';
update items set extra=column_add(extra, 'color','blue') where type='cd';  -- this will only update color, but keep size
select title,type,column_json(extra) from items where type='cd';
```
to remove a attr, use column_delete
```
update items set extra=column_delete(extra, 'color') where type='cd'; 
```
####adding a dynamic column part to all records
######add a attr
```
update items set extra=column_add(extra, 'in_stock',10) where type='cd'; //error.because cd has no dynamic columns
```

So how would we handle, if we wanted to add a dynamic column part to more than one record. Where some records have an empty BLOB and others already have some values?  

######use column check:
```
select title,type,column_check(extra) from items where 1=1;
```
If returns 1, then dynamic columns exist, else return NULL
######update
```
update items set extra='test' where column_get(extra,'color' as char)='blue';
select title,type,column_check(extra) from items where 1=1;  -- return 0
update items set extra=column_create('color','blue','size','2') where extra='test';
select title,type,column_check(extra) from items where 1=1;  --return 1
```

add a attr when extra is null
```
update items set extra=column_create('in stock',20) where column_check(extra) is null;
update items set extra=column_add(extra, 'in stock',20) where column_check(extra)=1;
```


###2. The power of CONNECT
####2 Installing the CONNECT storage engine
check install status
```
select plugin_status from information_schama.plugins where plugin_name='connect';
```
show plugin_dir
```
show variables like 'plugin_dir';
```

execute linux command in mysql :
```
\! ls -l /usr/lib/mysql/plugin
```
install connect plugin:
```
install plugin connect soname 'ha_connect';
```
show engines
```
select * from information_schema.engines where engine='connect' limit 5;
```


####3 Connecting to a remote MySQL database
```
show variables like 'hostname';
```

create a table at local machine, copy from remote machine
```
create table tbname engine=connect table_type=mysql connection='mysql://user:pass@remotehost.com:3306/remotetable';
```
test
```
select * from tbname;
```
####4 Executing batch commands on a remote database
Execute code  
execsrc=1  
flag=0: default; command to be run  
flag=1: DML(affected rows)  
flag=2: mwssage returned  
flag=3: warnings returned  

create table tbname_remote:
```
create table tbname_remote (
command varchar(255) not null, 
message varchar(255) flag=2, 
cols_rows int not null flag=1, 
warnings not null flag=3)
engine=connect table_type=mysql 
connection='mysql://user:pass@remotehost.com:3306/remotetable option_list='execsrc=1';
```


execute commands:
```
select command,cols_rows, message, warnings from tbname_remote where command in (
"insert into tbname(auth,mytest)values('','')",
"alter table tbname add column tx timestamp default current_timestamp on update current_timestamp",
"update tbname set auth='' where 1=1")
```







###4. The Cassandra Storage Engine
####1 Cassandra storage engine overview
```
mysql -u root -p root -e "SHOW variables like 'plugin_dir';"
```
list 1 file per line
```
ls -1 ...
```
install this plugin
```
install plugin cassandra soname 'ha_cassandra';
```
cassandra avail in mdb:
```
cassandra_failure_retries //default=3
cassandra_timeout_exceptions
cassandra_unavailable_exceptions
```
####2 Creating a table
data type"
cassandra->maria:
```
boolean->BOOL
float->FLOAT
double->DOUBLE
int->INT
blob->BLOB,VARBINARY
timestamp->TIMESTAMP,BIGINT(timestamp use micorsec,BINGINT gets millsec since epoch)
bigint->TINYINT,BIGINT,MEDIUMINT,INT
varint->VARBINARY
decinal->VARBINARY
ascii->BLOB,VARCHAR(latin1)
text->BLOB,VARCHAR(utf8)
uuid->CHAR(36)
counter->BININT(readonly)
```

######Cass's Dynamic columns
blob data type, any field name  
set
```
DYNAMIC_COLUMN_STORAGE=yes
```

connect mariadb to cassandra, or use SET cassandra_default_thrift_host=...
```
cassandra_default_thrift_host
```

sample table:
```
SET cassandra_default_thrift_host='';
create table cass_tbl{
inv_id primary key,
type varchar(20) charset latin1,
brand varchar(20) charset utf8,
price varbinary(10),
title blob,
other_attr blob dynamic_column_storage=yes
) engine=cassandra
column_family='my_family'
keyspace='my_space';
```

####3 Querying data
[cassandra status vars](https://mariadb.com/kb/en/mariadb/cassandra-status-variables/)
[cassandra system vars](https://mariadb.com/kb/en/mariadb/cassandra-system-variables)
```
cassandra_rnd_batch_size //default=10000
```

batched key access
```
optimizer_switch='join_cache_hashed=on'
```

set
```
join_cache_level=7/8
```
######Batched key access
```
cassandra_multiget_batch_size //set batch size
```

```
cassandra_multiget_keys_scanned
cassandra_multiget_reads
cassandra_multiget_rows_read
```

