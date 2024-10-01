# Object Column Mapping with Dapper

When working with Dapper, mapping result entities to columns can sometimes be tricky, especially when the property names don’t align perfectly with database column names. 
I faced the challenge of mismatched property names in the result entity. While Dapper offers options like dynamic objects or mapping results to custom types, I wanted a more structured approach without relying on attributes for mapping. I decided to take a different path by implementing a custom object-column mapper, drawing inspiration from tools like FluentValidation and AutoMapper.

All starts with the `ObjectColumnMapper` base class that handles all the heavy lifting.

```csharp
using static Dapper.SqlMapper;

public abstract class ObjectColumnMapper<T>
{
    private readonly IDictionary<string, string> _columnPropertyMap = new Dictionary<string, string>();

    private readonly IDictionary<string, PropertyInfo> _properties;

    protected ObjectColumnMapper()
    {
        var type = GetType();
        while (type is not null && type.IsGenericType == false)
        {
            type = type.BaseType;
        }

        if (type is null)
        {
            throw new ArgumentNullException();
        }

        var generic = type.GenericTypeArguments[0];
        _properties = generic.GetProperties().ToDictionary(k => k.Name, v => v);
    }

    protected void SetMapping<TProp>(string column, Expression<Func<T, TProp>> expr)
    {
        if (expr.Body is MemberExpression memberExpression)
        {
            var name = memberExpression.Member.Name;
            _columnPropertyMap.Add(column, name);
        }
    }

    private PropertyInfo Map(Type type, string column)
    {
        if (!_columnPropertyMap.TryGetValue(column, out var key)) return null;

        return _properties[key];
    }

    public void RegisterTypeMap(Action<Type, ITypeMap> register)
    {
        register(typeof(T), new CustomPropertyTypeMap(typeof(T), Map));
    }
}
```

Then you implement your mapper where you define mappings.

```csharp
public class UserEntityMapper : ObjectColumnMapper<UserEntity>
{
    public UserEntityMapper()
    {
        SetMapping("firstName", model => model.Name);
        SetMapping("lastName", model => model.Surname);
        SetMapping("dateOfBirth", model => model.Birthday);
    }
}
```

Inside your application you have to register type mapping by calling the `RegisterTypeMap()` from the base object-mapper. The thing is, the registration should happen only once. I personally found my composition root a good candidate, especially for class library, but `Program.cs` would be also fine.

```csharp
public static class CompositionRoot
{
    public static IServiceCollection AddPersistence(this IServiceCollection services, IConfiguration configuration)
    {
        new UserEntityMapper().RegisterTypeMap(SqlMapper.SetTypeMap);
    }
}
```

Finally, just use the Dapper ORM and that's it.

```csharp
public async Task<UserEntity> ReadAsync(Guid id)
{
    var parameters = new DynamicParameters();
    parameters.Add("@id", id);

    return await _connection.QueryFirstOrDefaultAsync<UserEntity>(SQL, parameters);
}
```

Keep codes going and see you later.

---

September, 2024 <br />
[Sebastian](mailto:ja@sebastianbusek.cz) <br />