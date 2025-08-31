# Google Summer of Code 2025 - Final Submission Report

<img width="790" height="197" alt="image" src="https://github.com/user-attachments/assets/e508bd8b-ad9c-47bd-a5db-abfae7cd6a16" />

# Project Details

https://summerofcode.withgoogle.com/programs/2025/projects/uzgYlzc5

- Organization: [MariaDB](https://mariadb.org/)
- Mentors: [Brandon Nesternko](https://github.com/bnestere), [Andrei Elkin](https://github.com/andrelkin)
- Contributor: [Mohanad Khaled](https://github.com/MohanadKh03)
- Project: [(MDEV-9345) - Replication to enable filtering on master ](https://jira.mariadb.org/browse/MDEV-9345)



# Project Overview
[Replication](https://mariadb.com/docs/server/ha-and-performance/standard-replication/replication-overview) is a feature allowing the contents of one or more servers (called primaries) to be mirrored on one or more servers (called replicas) by writing the queries in a specific format inside a log binary file called the **binlog** and the queries are written as **events** in three formats:
1) Statement
2) Row
3) Mixed 

Replication filters in MariaDB are a set of features that control which events are replicated and which are ignored. Currently, these filters work in three main ways:

1. **Replica-side filtering**  
   All events are written to the binary log and sent over the network to the replica. Filtering rules are then applied on the replica side.  
   [Docs → Replica-side filtering](https://mariadb.com/docs/server/ha-and-performance/standard-replication/replication-filters#replication-filters-for-replicas)

2. **Binlog-exclusion filtering (Primary-side omission)**  
   Events are not written to the binary log at all on the primary. As a result, they never reach the replica.  
   [Docs → Primary-side omission](https://mariadb.com/docs/server/ha-and-performance/standard-replication/replication-filters#binary-log-filters-for-replication-primaries)

3. **Global skip**  
   All events are skipped entirely without fine-grained control over databases or tables.  
   [Docs → Skip events](https://mariadb.com/docs/server/ha-and-performance/standard-replication/selectively-skipping-replication-of-binlog-events)


This project introduces **database- and table-level filtering at the primary side**.  
Unlike the existing *binlog-exclusion* approach (where events are never written to the binary log), this enhancement writes events to the binary log but filters them before transmission.


# Work Done

Over the past few months, I worked on adding primary-side replication filtering, which is one of the main goals of the project.  
This feature lets users apply replication filters directly on the primary while still keeping Point-in-Time Recovery (PITR) safe,  
since all needed events are still written to the binary log.

This is an important step, but there are still more tasks and improvements left.  
Reaching this stage gives a good base for making replication filtering safer and more useful in the future.


**Technical Details Notes**: The project started with an initial design that was supposed to be a basic per-event filter although this caused lots of design flaws such as: 
- Possibility of the replica receiving two GTID events in case of a per-event filter which would conflict with the possibility of network failure as it has been reported in a previous issue [MDEV-27697](https://jira.mariadb.org/browse/MDEV-27697) 
- In order to filter tables in a per-event basis we had to parse the query again after the query has already been parsed and the binlog has already been written since there is no table support inside the current ROW-format's TABLE_MAP_EVENT or *_ROWS_EVENT and this would be very inefficient let alone would conflict with the parser/lexer states and the need to initialize it again which would complicate matters a lot

**The solution and redesign** we took halfway through the project is to not filter on a per-event basis but on a event-group basis ... An event group is basically a GTID_EVENT then a number of queries (ROW or STATEMENT or MIXED events) and then possibly and end event such as a XA_EVENT or ROLLBACk or COMMIT that is represented either in a XID_EVENT or a QUERY_EVENT with a COMMIT string

# Pull Request

Still under review: [Replication to enable filtering on master](https://github.com/MariaDB/server/pull/4086)

The Pull Request is still under review because more features need to be completed before it can be fully merged.

# Work Left after GSoC

For the project to be fully complete, there are two main things left:
1. Add more replication tests  
2. Add partial filtering for queries that affect multiple tables or databases

**Technical details notes**: 
- As mentioned before the event-group filter design is what has been implemented but it currently lacks support for queries that affect multiple tables or databases
- If a query affects a single table/database then the current milestone works by applying the replication filter rules over the primary side successfully ... But if a query affects multiple tables or databases then either all of them will get filtered or none of them get filtered , This is still a problem to be continued beyond GSoC as it is a common case to have a query that affects multiple tables/databases but not all of them should get filtered (partial filtering still not supported)

e.g: Filtering rule says that a table with the name db1.table1 should get filtered ... lets say a query affects db1.table1 and db1.table2 so even though db1.table2 should not be filtered , currently it gets filtered since there is a table that gets filtered in the same query/event logged in the binlog  

# Challenges Faced
- **Large and complex codebase**: The project involved working with a codebase of around 4–5 million lines of code. Many components were highly interdependent, and several lacked documentation, making navigation and understanding difficult.  

- **Evolving design requirements**: The initial design seemed straightforward, but after the first patch, several design flaws became apparent. Due to the complexity of integrating new features into replication, we had to redesign the project to ensure long-term maintainability and extensibility.  
  - *Note*: This redesign was the main reason the pull request is still ongoing, as it took place about halfway through the project.  
  - Initial design composed of per-event filtering but after changing the project's technical design due to the reasons mentioned in the previous technical details notes it had to be redesigned and more time than what was supposed to take this is why some work is still left after GSoC  
- **Timezone differences**: My mentor and I had a 9-hour timezone gap, which made real-time collaboration challenging and required careful planning and asynchronous communication.  

# Lessons Learnt

- A project that looks easy at first may reveal significant challenges once development begins highlighting the importance of **software architecture and design considerations**.  
- In-depth understanding of **MariaDB replication internals** and the structure of **binary log events**.  
- The value of **Test-Driven Development (TDD)** and why it plays a critical role in building reliable features.  
- How projects often **evolve and change** as they progress, and how **design limitations in long-standing, large projects** introduce unique complexities that require careful navigation and trade-offs.


# Acknowledgments  

I would like to thank my mentors **Brandon** and **Andrei**, as well as [**Kristian**](https://github.com/knielsen), who, even though not an official GSoC mentor, helped a lot in the redesign process and shared many valuable advices on how to approach the new design.
