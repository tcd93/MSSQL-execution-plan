# Basics of MSSQL Plan Optimizer
An article about Microsoft SQL Server's plan optimizer

## Execution Plan & Optimizer
### What is an Execution Plan?
**IMG**

An execution plan is a set of physical operations (operators) that can be performed to produce the required result

The data flow order is from right to left, the thickness of the arrow indicate the amount of data compared to the entire plan

**IMG**

### Retrieving the estimated plan
From MSSM (Microsoft SQL Server Management tool), select an SQL block:
- Press `CTRL + L`
- or: Right click → Display estimated execution plan
- or Query → Display estimated execution plan
