title: "利用反射映射数据库对象(二)"
date: 2015-09-13 13:27:42
tags:
- .NET
- C#
categories:
- .NET
---

在上一篇文章里，我们里用反射，将数据库的查询结果集映射到对象中，以减少代码。现在我们就利用相同的原理来完成更新记录的操作。

更新记录时，我们需要知道表名，主键名，主键值，以及需要更新的字段信息。最初阶段，这些信息我们只要通过参数传递到方法中。
```csharp
public int Update(string tableName, string primaryKeyName, object entity)
{

}
```

这时我们需要做的就是拼接sql语句，从entity对象中反射出属性名称更新到数据库中，当然，entity中的属性必须与数据库字段名称对应，并且必须包含主键。代码还是比较简单的:
```
public int Update(string tableName, string primaryKeyName, object entity)
{
    using (var conn = new SqlConnection(SqlHelper.ConnectionString))
    {
        conn.Open();
        using (var cmd = conn.CreateCommand())
        {
            var sb = new StringBuilder();
            var index = 0;
            var props = entity.GetType().GetProperties().Where(prop => prop.CanRead);
            object primaryKeyValue = null;

            foreach (var p in props)
            {
                // Don't update the primary key, but grab the value if we don't have it
                if (String.Compare(p.Name, primaryKeyName, StringComparison.Ordinal) == 0)
                {
                    primaryKeyValue = p.GetValue(entity);
                    continue;
                }

                // Build the sql
                if (index > 0)
                {
                    sb.Append(", ");
                }
                sb.AppendFormat("[{0}] = @{1}", p.Name, index++);

                // Store the parameter in the command
                AddParam(cmd, p.GetValue(entity));
            }

            cmd.CommandText = string.Format("UPDATE [{0}] SET {1} WHERE [{2}] = @{3}",
                tableName, sb, primaryKeyName, index++);

            // Throw exception if primary key not mapped.
            // It'll be dangerous.
            if (primaryKeyValue == null)
            {
                throw new ArgumentException("primary key not mapped.");
            }
            AddParam(cmd, primaryKeyValue);

            var retv = cmd.ExecuteNonQuery();
            return retv;
        }
    }
}

private static void AddParam(IDbCommand cmd, object item)
{
    // Support passed in parameters
    var idbParam = item as IDbDataParameter;
    if (idbParam != null)
    {
        idbParam.ParameterName = cmd.Parameters.Count.ToString();
        cmd.Parameters.Add(idbParam);
        return;
    }

    var p = cmd.CreateParameter();
    p.ParameterName = cmd.Parameters.Count.ToString();
    if (item == null)
    {
        p.Value = DBNull.Value;
    }
    else
    {
        var t = item.GetType();
        if (t.IsEnum)
        {
            p.Value = (int)item;
        }
        else if (t == typeof(Guid))
        {
            p.Value = item.ToString();
            p.DbType = DbType.String;
            p.Size = 40;
        }
        else if (t == typeof(string))
        {
            p.Size = Math.Max((item as string).Length + 1, 4000);
            p.Value = item;
        }
        else if (t == typeof(bool))
        {
            p.Value = ((bool)item) ? 1 : 0;
        }
        else
        {
            p.Value = item;
        }
    }
    cmd.Parameters.Add(p);
}
```

这样，我们在写更新记录时，只需要一行代码就行了:
```csharp
Update("User", "Id", user);
```

大家应该能注意到，以上代码中entity参数类型是object，而不是泛型T等具体的类型，这样做可以使之支持匿名类型，当我们只需要更新数据库的某一列时就尤为方便:
```csharp
Update("User", "id", new { Id=1, Name="Kitty" })
```

到这里还没有结束，因为我们每次更新时都需要传入表名，主键名，最理想的情况是我们只需要传入user对象就可以了。那么我们通过什么来告诉方法表名与主键名称呢？同样是反射。首先我们得找个地方定义User类对应的表名、主键名，这里我们参考其他ORM框架利用.NET的自定义Attribute，见代码:
```csharp
/// <summary>
/// Specify the table name of a data class
/// </summary>
[AttributeUsage(AttributeTargets.Class)]
public class TableNameAttribute : Attribute
{
    public TableNameAttribute(string tableName)
    {
        Value = tableName;
    }
    public string Value { get; private set; }
}

/// <summary>
/// Specific the primary key of a data class 
/// </summary>
[AttributeUsage(AttributeTargets.Class)]
public class PrimaryKeyAttribute : Attribute
{
    public PrimaryKeyAttribute(string primaryKey)
    {
        Value = primaryKey;
    }

    public string Value { get; private set; }
}

[TableName("User")]
[PrimaryKey("Id")]
public class User
{
    public Int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public string Gender { get; set; }
    public string Address { get; set; }
}

public int Update<T>(T entity)
{
    var t = typeof(T);
    // Get the table name
    var a = t.GetCustomAttributes(typeof(TableNameAttribute), true);
    var tableName = a.Length == 0 ? t.Name : (a[0] as TableNameAttribute).Value;

    // Get the primary key
    a = t.GetCustomAttributes(typeof(PrimaryKeyAttribute), true);
    var primaryKeyName = a.Length == 0 ? "ID" : (a[0] as PrimaryKeyAttribute).Value;
    return Update(tableName, primaryKeyName, entity);
}
```

最终，我们更新数据的代码就变成了 Update(user), 而不需要每张表都去大段地拼接字符串了。

另外，以上代码还是可以优化的。重载代码时，我们对T entity类型反射了两次，假如Update方法是写在Dal类中，我们可以为之添加两个readonly字段，并在类型构造器中将其赋值，这样每个Dal类在第一次调用时，就将表名，主键名永久保存了。


```csharp
public class DalBase<T> where T : class, new()
{
    private static readonly string TableName;
    private static readonly string PrimaryKeyName;
    static DalBase()
    {
        var t = typeof(T);
        // Get the table name
        var a = t.GetCustomAttributes(typeof(TableNameAttribute), true);
        TableName = a.Length == 0 ? t.Name : (a[0] as TableNameAttribute).Value;

        // Get the primary key
        a = t.GetCustomAttributes(typeof(PrimaryKeyAttribute), true);
        PrimaryKeyName = a.Length == 0 ? "ID" : (a[0] as PrimaryKeyAttribute).Value;
    }
}
```

最后，相信大家已经可以自己写Insert与Delete方法了。
