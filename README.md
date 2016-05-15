#### laudmariaformysqlusr
#####1 Dynamic columns
######creating dynamic columns
```
create database dynamic_cols charset utf8;
show variables like '%char%';
show variables like 'default_storage_engine';
create table items(item_id int unsigned not null auto_increment primary key,type varchar(64) not null,
title varchar(255) not null,price decimal(5,2) not null,manufacturer varchar(64) not null,extra blob);
```
insert
```
insert into items set title='my',type='cd',price='1',manufacturer='cd';
```
create dynamic columns
```
insert into items set title='you',type='cd',price='2',manufacturer='ef',extra=column_create('color','red','size','l');
```

######querying dynamic columns
column_get
```
select column_get(extra, 'color' as char) from items where title='you';   //must use `char`, cannot be any another chars
```
column_exists
```
select title,type,column_exists(extra,'color') from items where 1=1;
```
combine
```
select type,title,column_get(extra,'color' as char) from items where column_exists(extra,'color');
select distinct(column_get(extra,'color' as char)) from items where column_exists(extra,'color');
```
column_list
```
select title,type,column_list(extra) from items where 1=1;
```
column_json
```
```
