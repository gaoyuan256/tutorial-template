Oracle-数据表空间
=================

[TOC]

表空间数据文件路径及使用情况查询
--------------------------------

.. code:: plsql

   set linesize 400 pagesize 500
   col file_name for a80
   select tablespace_name,file_name from dba_data_files;
   --select tablespace_name,file_name from dba_data_files where tablespace_name = '&tablespace_name';

   TABLESPACE_NAME 	       FILE_NAME
   ------------------------------ ---------------------------------------------------
   USERS			       /oradata/ORCL/datafile/o1_mf_users_hb8582d8_.dbf
   UNDOTBS1		       /oradata/ORCL/datafile/o1_mf_undotbs1_hb8582ct_.dbf
   SYSAUX			       /oradata/ORCL/datafile/o1_mf_sysaux_hb8582cr_.dbf
   SYSTEM			       /oradata/ORCL/datafile/o1_mf_system_hb8582bq_.dbf

表空间情况
----------

.. code:: plsql

   set linesize 400 pagesize 500
   col TABLESPACE_NAME for a20
   col CONTENTS for a10
   select status,
          t1.tablespace_name,
          contents,
          extent_management "Extent Management",
          segment_space_management "Segment Space Management",
          allocation_type "Allocation Type",
          logging,
          force_logging,
          round(t1.maxsize / 1024 / 1024 / 1024, 2) "MaxSize(GB)",
          round(t1.bytes / 1024 / 1024 / 1024, 3) "File Size(GB)",
          round((t1.bytes - t2.bytes) / 1024 / 1024 / 1024, 3) "Used(GB)",
          round((t1.bytes - t2.bytes) / t1.bytes * 100, 2) "Used%",
          round((t1.maxsize - t1.bytes + t2.bytes) / 1024 / 1024 / 1024, 2) "AVL Size(GB)",
          round((t1.maxsize - t1.bytes + t2.bytes) / (t1.bytes - t2.bytes), 1) x,
          t2.fsfi,
          t2.frags
     from (select tablespace_name, sum(bytes) bytes, sum(maxsize) maxsize
             from (select tablespace_name, bytes, maxbytes maxsize
                     from dba_data_files
                    where autoextensible = 'YES'
                   union all
                   select tablespace_name, bytes, bytes maxsize
                     from dba_data_files
                    where autoextensible = 'NO')
            group by tablespace_name) t1,
          (select tablespace_name,
                  sum(bytes) bytes,
                  round(sqrt(max(blocks) / sum(blocks)) /
                        sqrt(sqrt(count(blocks))) * 100,
                        0) fsfi,
                  count(blocks) frags
             from dba_free_space
            group by tablespace_name) t2,
          dba_tablespaces
    where dba_tablespaces.tablespace_name = t1.tablespace_name
      and dba_tablespaces.tablespace_name = t2.tablespace_name
    order by "Used%" desc;

数据文件具体情况
----------------

.. code:: plsql

   set linesize 400 pagesize 500
   col name for a70
   col tablespace for a20
   select online_status "Status",
          tablespace_name "Tablespace",
          t1.file_id,
          file_name "Name",
          round(t1.bytes / 1024 / 1024, 2) "Size(MB)",
          round((t1.bytes - nvl(t2.bytes, 0)) / 1024 / 1024, 2) "Used(MB)",
          round((t1.bytes - nvl(t2.bytes, 0)) / t1.bytes * 100, 2) "Used%",
          autoextensible "Autoextensible",
          t1.increment_by "Inc (bs)",
          t1.online_status "Status"
     from dba_data_files t1
     left join (select file_id, sum(bytes) bytes
                  from dba_free_space
                 group by file_id) t2
       on t1.file_id = t2.file_id
    order by t1.tablespace_name, t1.file_id;

数据文件相关操作
----------------

创建表空间数据文件
~~~~~~~~~~~~~~~~~~

::

   create tablespace ACCOUNT datafile '+DATADG' size 200M autoextend on;

添加表空间数据文件
~~~~~~~~~~~~~~~~~~

::

   alter tablespace ACCOUNT  add datafile  '+DATADG'  size 200m autoextend on;

删除表空间和数据文件
~~~~~~~~~~~~~~~~~~~~

::

   drop tablespace ACCOUNT including contents and datafiles;

删除空的表空间，但是不包含物理文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   drop tablespace tablespace_name;

删除非空表空间，但是不包含物理文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   drop tablespace tablespace_name including contents;

删除空表空间，包含物理文件
~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   drop tablespace tablespace_name including datafiles;

删除非空表空间，包含物理文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   drop tablespace tablespace_name including contents and datafiles;

如果其他表空间中的表有外键等约束关联到了本表空间中的表的字段，就要加上CASCADE CONSTRAINTS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   drop tablespace tablespace_name including contents and datafiles CASCADE CONSTRAINTS;

查询和修改用户默认表空间
------------------------

::

   select USERNAME,DEFAULT_TABLESPACE from dba_users where username in ('YANGLAO','DSG','NEWGZDB');

   alter user yanglao default tablespace USERS;

查询用户对象大小
----------------

::

   select sum(bytes)/1024/1024/1024 from dba_segments where OWNER='YANGLAO';
