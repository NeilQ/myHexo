title: "利用反射映射数据库对象"
date: 2015-07-10 15:56:17
tags:
- .NET
- C#
categories:
- .NET
---

```csharp
public IEnumerable<User> GetUsers()
{
    var conn = "";
    using (var sqlConnection = new SqlConnection(conn))
    {
        using (var command = sqlConnection.CreateCommand())
        {
            command.CommandText = "select * from users";
            sqlConnection.Open();
            using (var reader = command.ExecuteReader())
            {
                if (!reader.HasRows) yield break;

                while (reader.Read())
                {
                    yield return new User
                    {
                        Id = reader.GetInt32(reader.GetOrdinal("Id")),
                        Name = reader.GetString(reader.GetOrdinal("Name")),
                        Phone = reader.GetString(reader.GetOrdinal("Phone")),
                        GroupId = reader.GetInt32(reader.GetOrdinal("GroupId")),
                        Nick = reader.GetString(reader.GetOrdinal("Nick"))
                    };
                }
            }
        }
    }
}
```

以上这段代码大家一定不陌生，当我们用Ado.net写数据层时，都会写上无数遍，无趣、麻烦、枯燥。当然，我们可以用一些代码生成工具来完成这个任务。但我们也可以优雅地把它做好。
这里介绍一种方法，利用反射机制遍历实体属性，并将数据库字段映射到实体中。Show you the code.

```csharp
public static T MapEntity<T>(SqlDataReader reader) where T : class, new()
{
    var props = typeof(T).GetProperties();
    var entity = new T();
    foreach (var prop in props.Where(prop => prop.CanWrite))
    {
        try
        {
            var index = reader.GetOrdinal(prop.Name);
            var data = reader.GetValue(index);
            if (data != DBNull.Value)
            {
                prop.SetValue(entity, Convert.ChangeType(data, prop.PropertyType), null);
            }
        }
        catch (IndexOutOfRangeException)
        {
            continue;
        }
    }
    return entity;
}
```

如此这般，我们只要一行代码就能完成实体映射：
```csharp
yield return MapEntity<User>(reader);

//yield return new User
//{
//    Id = reader.GetInt32(reader.GetOrdinal("Id")),
//    Name = reader.GetString(reader.GetOrdinal("Name")),
//    Phone = reader.GetString(reader.GetOrdinal("Phone")),
//    GroupId = reader.GetInt32(reader.GetOrdinal("GroupId")),
//    Nick = reader.GetString(reader.GetOrdinal("Nick"))
//};
```

引申一下，既然查询可以通过反射映射实体，那插入、更新、删除同理也是可以，这样开发中我们就可以这样写代码：
```csharp
var user = SqlHelper.Query<User>("select top 1 * from users");
SqlHelper.Insert<User>(user);
SqlHelper.Update<User>(user);
SqlHelper.Delete<User>(user);
```
这是什么？Yes, ORM! 其实这就是.NET ORM的核心原理。

这里推荐两个轻量级ORM: dapper, PetaPoco。
