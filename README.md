Redo Sim Package
================

The redo_sim package is a PL/SQL package for generating a desired redo throughout for an Oracle database.

This can be useful in tuning, troubleshooting, and planning as redo throughput is often a bottleneck for write bound systems, and for standbys that are applying redo.

If used on a RAC database, it will distribute the requested redo generation across all RAC instances.

Prerequisites
-------------

The package assumes:

  - It is being installed by an administrative user called `admin`. 
  - The database is configured to run jobs via DBMS_JOBs and there are sufficient job queue processes available for use.  
    - The sim will use up to J job queue processes, where J = max( megabytes_per_second, commits_per_second/100 ) + 2
  - The user installing the package can run `dbms_job.submit`
  - The user installing the package can see stuff in the `admin` schema via the `all_` views and can read `gv$instance`, `v$sesstat`, `v$statname` views *via a direct grant* not via a role
    - You may have to run `grant select on sys.v_$sesstat to admin;` and similarly for v$statname.
    - Note to AWS RDS users, you may need to run:
      - exec rdsadmin.rdsadmin_util.grant_sys_object( 'V_$SESSTAT', 'ADMIN' );
      - exec rdsadmin.rdsadmin_util.grant_sys_object( 'V_$STATNAME', 'ADMIN' );

Side Effects
------------

This package will create the following permanent database objects:

  - admin.redo_sim package
  - admin.redo_sim_output table
  - admin.redo_sim_storage table
  - admni.pk_redo_sim_storage index
  - admin.redo_sim_sequence sequence

While the simulation is running these additional objects will be created:

 - A number of jobs run by DMBS_JOBs which are removed on completion.

Public Methods
--------------

### redo_sim.configure

Parameter|Type|Required|Default|Info
--- | --- | --- | --- | ---
tablespace | IN | No | scratch | The tablespace to create temporary objects in
bytes | IN | No | 134217728 | The size of the underlying data table used to generate redo with, defaults to 128 MB.  Actual storage used will be higher due to block overhead and indexes.  

For tests that are focusing on redo apply on a standby it is good to make the bytes size comparable with the data that will change in production, as redo apply performance depends in part on the ability of the standby to write to the data files where the redo is generated.

### redo_sim.sim

Parameter|Type|Required|Default|Info
--- | --- | --- | --- | ---
seconds | IN | Yes | - | Seconds the run the simulation for
bytes_ps | IN | Yes | - | Goal redo bytes per second to produce
commits_ps | IN | Yes | - | Goal commits per second to perform
verbose | IN | No | 1 | If set to 1 prints progress to screen and logs results to `admin.redo_sim_output`.  If set to 0 stores results in `admin.redo_sim_output` only

Notes:

  - `sim` tries to generate at least as much redo as requested.  Due to Oracle overhead some additional redo will be created - for high throughput simulations this is around 20%.  For low throughput simulations (e.g. 1 byte per second) the overhead will be much greater than the generated redo.
  - If the database is not capable of driving the requested throughput the simulation will run longer than the requested `seconds`.  The `sim` ensures `seconds*byte_ps` are written, and `seconds*commits_ps` commits are performed - if they can not be performed within `seconds` then the simulation will run longer.

Usage Examples
--------------

### One Time Setup

Install package to the `test` tablespace with 384 MB of storage for the table on which redo is generated:

```
sqlplus>@redo_sim.pkg

PL/SQL procedure successfully completed.
Package created.
Package body created.

sqlplus>--This step can take a while as it populates the redo_sim_storage table.
sqlplus>exec redo_sim.configure( 'USERS', 384*1024*1024 );
Executing: DROP TABLE admin.redo_sim_storage
Executing: CREATE TABLE admin.redo_sim_storage (pk NUMBER NOT NULL, data VARCHAR2(512), CONSTRAINT pk_redo_sim_storage
PRIMARY KEY (pk) USING INDEX (CREATE UNIQUE INDEX admin.pk_redo_sim_storage ON admin.redo_sim_storage (pk) TABLESPACE
USERS)) TABLESPACE
USERS INITRANS 16 STORAGE (INITIAL 471859 K)
Inserting rows into admin.redo_sim_storage.
Configuration complete

PL/SQL procedure successfully completed.
```



### Redo simulations

Generate 3 MB/s with 50 commits per second for 10 seconds.

```
sqlplus>exec redo_sim.simulate( seconds => 10, bytes_ps => 3*1024*1024, commits_ps => 50 );
Starting 3 workers to produce 3145728 bytes per second of redo with 50 commits per second.
Starting worker 1 with job id 21 on instance 1.
Starting worker 2 with job id 22 on instance 1.
Starting worker 3 with job id 23 on instance 1.

PL/SQL procedure successfully completed.

sqlplus>select * from admin.redo_sim_output order by sim_number, event_time;

SIM_NUMBER	  SID ROLE	     EVENT_TIM MESSAGE
---------- ---------- -------------- --------- --------------------------------------------------------------------------------
	-1	   44 Configuration  03-DEC-16 Inserting rows into admin.redo_sim_storage.
	-1	   44 Configuration  03-DEC-16 Executing: DROP TABLE admin.redo_sim_storage
	-1	   44 Configuration  03-DEC-16 Executing: CREATE TABLE admin.redo_sim_storage (pk NUMBER NOT NULL, data VARCHAR
					       2(512), CONSTRAINT pk_redo_sim_storage PRIMARY KEY (pk) USING INDEX (CREATE UNIQ
					       UE INDEX admin.pk_redo_sim_storage ON admin.redo_sim_storage (pk) TABLESPACE USE
					       RS)) TABLESPACE USERS INITRANS 16 STORAGE (INITIAL 471859 K)

	-1	   44 Configuration  03-DEC-16 Configuration complete
	 1	   44 Sim Master     03-DEC-16 Starting 3 workers to produce 3145728 bytes per second of redo with 50 commits p
					       er second.

	 1	   44 Sim Master     03-DEC-16 Starting worker 3 with job id 23 on instance 1.
	 1	   44 Sim Master     03-DEC-16 Starting worker 2 with job id 22 on instance 1.
	 1	   44 Sim Master     03-DEC-16 Starting worker 1 with job id 21 on instance 1.
	 1	   46 Sim Worker 1   03-DEC-16 Starting redo generation.
	 1	   32 Sim Worker 3   03-DEC-16 Starting redo generation.
	 1	   55 Sim Worker 2   03-DEC-16 Starting redo generation.
	 1	   46 Sim Worker1    03-DEC-16 Completed simulation, generated 13093140 bytes of redo and committed 160 times.
					       Averaged 1272.26 Kb of redo per second, with 15.92 commits per second.

	 1	   55 Sim Worker2    03-DEC-16 Completed simulation, generated 13091260 bytes of redo and committed 160 times.
					       Averaged 1272.08 Kb of redo per second, with 15.92 commits per second.

	 1	   32 Sim Worker3    03-DEC-16 Completed simulation, generated 13123552 bytes of redo and committed 160 times.
					       Averaged 1276.49 Kb of redo per second, with 15.93 commits per second.


14 rows selected.
```



License
-------

This package is free to use or modify under the MIT License.



