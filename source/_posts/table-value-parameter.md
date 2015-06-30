title: "表值参数"
date: 2015-05-24 22:13:27
tags: 
- 表值参数
categories:
- DataBase
---

表值参数（Table-value parameter）可以将.NET中的DataTable类与SQL Server Table类型进行映射，可以把多行数据作为参数传递到存储过程，进行批量操作。

简单举个例子，假如我们有这样一个表：
```sql
CREATE TABLE [dbo].[People] (
    Id  INT IDENTITY PRIMARY KEY,
    Name    NVARCHAR(255),
    Description NVARCHAR(255)
)

```

以及一个插入数据的存储过程：
```sql
CREATE PROCEDURE [dbo].[usp_Insert_People]
    @Name  NVARCHAR(255),
    @Description  NVARCHAR(255)
AS
    -- something logic

    INSERT INTO People(Name, Description)
    VALUES(@Name, @Description)

    -- something logic
```

这时如果我们需要通过.NET往People表里插入1000条数据，就需要调用1000次[usp_Insert_People]这个存储过程。这里我们就可以通过表值参数只调用一次存储过程来完成需求。

首先定自定义一个表类型
```sql
CREATE TYPE [dbo].[udt_PeopleTable] AS TABLE(
    Name    NVARCHAR(255),
    Description NVARCHAR(255)
)
```
同时修改下存储过程
```sql
CREATE PROCEDURE [dbo].[usp_Insert_People]
   @Peoples udt_PeopleTable READONLY
AS
    -- something logic

    INSERT INTO People(Name, Description)
        SELECT Name, Description FROM @Peoples

    -- something logic
```

这样当我们需要插入1000条数据时，就只需要将table作为参数传入存储过程，调用一次就ok了，.NET简单示例如下：
```csharp
var peopleList = new List<People> {
    new People{ Name="Bill" },
    new People{ Name="胡一刀" }
};
using (SqlCommand command = new SqlCommand("dbo.usp_Insert_People", conn))
{
    command.CommandType = CommandType.StoredProcedure;
    // ToDataTable方法将list中的People对象转换成DataTable,
    // column为"Name","Description"。
    SqlParameter parameter = command.Parameters.AddWithValue("@Peoples", peopleList.ToDataTable());
    parameter.SqlDbType = SqlDbType.Structured;
    parameter.TypeName = "udt_PeopleTable";
    command.ExecuteNonQuery();
}
```

另一些注意点：
> * EF目前不支持表值参数
> * 表值参数是Readonly的
> * C# DataTable与表值参数的结构必须一致

