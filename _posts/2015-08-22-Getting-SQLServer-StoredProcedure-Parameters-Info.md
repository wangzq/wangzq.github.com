---
layout: post
category: notes
tags: [sqlserver]
---

## Problem
I am trying to write a t4 template or PowerShell script to generate wrappers for calling SQL Server Stored Procedures, to do that I am going to know detailed information of the stored procedure's parameters, such as:

 - name
 - data type
 - input or output
 - default value

## First try of using plain ado.net with SqlCommandBuilder.DeriveParameters
This approach is tried first because it doesn't require extra assemblies to do, just use default System.Data namespace:

	public static IEnumerable<SqlParameter> GetSprocParameters(SqlConnection conn, string name)
	{
		var cmd = new SqlCommand(name, conn) { CommandType = CommandType.StoredProcedure };
		SqlCommandBuilder.DeriveParameters(cmd);
		return cmd.Parameters.Cast<SqlParameter>();
	}

However, the SqlParameters returned from this, doesn't have the last two information I needed, it only has the name and data type.

## Second try of using sys objects in sql server
This is to use `sys.parameters` to query:

	select * from sys.parameters where object_id = object_id('dbo.MySproc')

Unfortunately this table seems also doesn't have the parameter direction and default value information. I might be missing something here, since I am sure SQL Server should be storing such information in a table, it is just I haven't found it and I didn't spend too much information in this, since I haven't tried third approach yet.

## Third approach of using SMO
Before trying out this, I am pretty sure it should support since SMO is feature rich and I have seen similar code has used it before. It turns out it does meet my requirement; I wrote a simple PowerShell module to help me load the SMO assemblies and connect to local sql server, then find the database, and sproc, then parameters:

	imp smohelpers
	Load-SmoAssemblies
	$srv = Connect-SmoServer
	$db = $srv.Databases | ? { $_.Name -eq 'mydb' }
	$sprocs = $db.StoredProcedures | ? { $_.Schema -eq 'dbo' }
	$sp = $sprocs | ? { $_.Name -eq 'mysp' }
	$sp.Parameters | ft

The parameters shown do have the default value and input/output stuff.


