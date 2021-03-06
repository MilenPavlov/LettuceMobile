For simple conversions to and from JSON strings and .NET objects, ServiceStack.Text provides handy extension methods to Deserialize and Serialize Objects.

```csharp
using ServiceStack.Text;
...

public class Person
{
    public string Name { get; set; }
    public string LastName { get; set; }
}

void PersonToJsonToPersonExample ()
{
    // Object to Json
	var person = new Person { Name = "John", LastName = "Doe" };
	var json = person.ToJson ();
	Console.WriteLine ("JSON representation of person: {0}", json);

	//Json to Object
	var john = json.FromJson<Person> ();
	Console.WriteLine ("Name: {0} LastName: {1}", john.Name, john.LastName);
}
```

## T.Dump() Extension method
Another useful library to have in your .NET toolbox is the [T.Dump() Extension Method](http://www.servicestack.net/mythz_blog/?p=202). Under the hood it uses a *Pretty Print* Output of the JSV Format to recursively dump the contents of any .NET object. Example usage and output: 

	var model = new TestModel();
	Console.WriteLine(model.Dump());

	//Example Output
	{
		Int: 1,
		String: One,
		DateTime: 2010-04-11,
		Guid: c050437f6fcd46be9b2d0806a0860b3e,
		EmptyIntList: [],
		IntList:
		[
			1,
			2,
			3
		],
		StringList:
		[
			one,
			two,
			three
		],
		StringIntMap:
		{
			a: 1,
			b: 2,
			c: 3
		}
	}

# ServiceStack's JsonSerializer

ServiceStack's JsonSerializer is optimized for serializing C# POCO types in and out of JSON as fast, compact and cleanly as possible. In most cases C# objects serializes as you would expect them to without added json extensions or serializer-specific artefacts.

JsonSerializer provides a simple API that allows you to serialize any .NET generic or runtime type into a string, TextWriter/TextReader or Stream.

### Serialization API

	string SerializeToString<T>(T)
	void SerializeToWriter<T>(T, TextWriter)
	void SerializeToStream<T>(T, Stream)
	string SerializeToString(object, Type)
	void SerializeToWriter(object, Type, TextWriter)
	void SerializeToStream(object, Type, Stream)

### Deserialization API

	T DeserializeFromString<T>(string)
	T DeserializeFromReader<T>(TextReader)
	object DeserializeFromString(string, Type)
	object DeserializeFromReader(reader, Type)
	object DeserializeFromStream(Type, Stream)
	T DeserializeFromStream<T>(Stream)

### Extension methods

	string ToJson<T>(this T)
	T FromJson<T>(this string)

Convenient **ToJson/FromJson** extension methods are also included reducing the amount of code required, e.g:

	new []{ 1, 2, 3 }.ToJson()   //= [1,2,3]
	"[1,2,3]".FromJson<int[]>()  //= int []{ 1, 2, 3 }

## JSON Format 

JSON is a lightweight text serialization format with a spec that's so simple that it fits on one page: [http://www.json.org](json.org).

The only valid values in JSON are:

  * string
  * number
  * object
  * array
  * true
  * false
  * null

Where most allowed values are scalar and the only complex types available are objects and arrays. Although limited, the above set of types make a good fit and can express most programming data structures.

### number, true, false types

All C# boolean and numeric data types are stored as-is without quotes.

### null type

For the most compact output null values are omitted from the serialized by default. If you want to include null values set the global configuration:

	JsConfig.IncludeNullValues = true;

### string type

All other scalar values are stored as strings that are surrounded with double quotes.

### C# Structs and Value Types

Because a C# struct is a value type whose public properties are normally just convenience properties around a single scalar value, they are ignored instead the **TStruct.ToString()** method is used to serialize and either the **static TStruct.ParseJson()**/**static TStruct.ParseJsv()** methods or **new TStruct(string)** constructor will be used to deserialize the value type if it exists.

### array type

Any List, Queue, Stack, Array, Collection, Enumerables including custom enumerable types are stored in exactly the same way as a JavaScript array literal, i.e:

	[1,2,3,4,5]

All elements in an array must be of the same type. If a custom type is both an IEnumerable and has properties it will be treated as an array and the extra properties will be ignored.

### object type

The JSON object type is the most flexible and is how most complex .NET types are serialized. The JSON object type is a key-value pair JavaScript object literal where the key is always a double-quoted string.

Any IDictionary is serialized into a standard JSON object, i.e:

	{"A":1,"B":2,"C":3,"D":4}

Which happens to be the same as C# POCO types (inc. Interfaces) with the values:

`new MyClass { A=1, B=2, C=3, D=4 }`

	{"A":1,"B":2,"C":3,"D":4}

Only public properties on reference types are serialized with the C# Property Name used for object key and the Property Value as the value. At the moment it is not possible to customize the Property Name.

JsonSerializer also supports serialization of anonymous types in much the same way:

`new { A=1, B=2, C=3, D=4 }`

	{"A":1,"B":2,"C":3,"D":4}


## Custom Serialization

Although JsonSerializer is optimized for serializing .NET POCO types, it still provides some options to change the convention-based serialization routine.

### Using Structs to Customize JSON

This makes it possible to customize the serialization routine and provide an even more compact wire format. 

E.g. Instead of using a JSON object to represent a point 

	{ Width=20, Height=10 }
	
You could use a struct and reduce it to just: 

	"20x10" 

By overriding **ToString()** and providing a static **Size ParseJson()** method:

	public struct Size
	{
		public double Width { get; set; }
		public double Height { get; set; }

		public override string ToString()
		{
			return Width + "x" + Height;
		}

		public static Size ParseJson(string json)
		{
			var size = json.Split('x');
			return new Size { 
				Width = double.Parse(size[0]), 
				Height = double.Parse(size[1]) 
			};
		}
	}

Which would change it to the more compact JSON output:

	new Size { Width = 20, Height = 10 }.ToJson() // = "20x10"

That allows you to deserialize it back in the same way:

	var size = "20x10".FromJson<Size>(); 

### Using Custom IEnumerable class to serialize a JSON array

In addition to using a Struct you can optionally use a custom C# IEnumerable type to provide a strong-typed wrapper around a JSON array:

	public class Point : IEnumerable
	{
		double[] points = new double[2];
	
		public double X 
		{
			get { return points[0]; }
			set { points[0] = value; }
		}
	
		public double Y
		{
			get { return points[1]; }
			set { points[1] = value; }
		}
	
		public IEnumerator GetEnumerator()
		{
			foreach (var point in points) 
				yield return point;
		}
	}

Which serializes the Point into a compact JSON array:

	new Point { X = 1, Y = 2 }.ToJson() // = [1,2]

### Custom Serialization Routines

If you can't change the definition of a ValueType (e.g. because its in the BCL), you can assign a custom serialization /
deserialization routine to use instead. E.g. here's how you can add support for `System.Drawing.Color`:

    JsConfig<System.Drawing.Color>.SerializeFn = c => c.ToString().Replace("Color ","").Replace("[","").Replace("]","");
    JsConfig<System.Drawing.Color>.DeSerializeFn = System.Drawing.Color.FromName;

## Custom Deserialization

Because the same wire format shared between Dictionaries, POCOs and anonymous types, in most cases what you serialize with one type can be deserialized with another, i.e. an Anonymous type can be deserialized back into a Dictionary<string,string> which can be deserialized into a strong-typed POCO and vice-versa.

Although the JSON Serializer is best optimized for serializing and deserializing .NET types, it's flexible enough to consume 3rd party JSON apis although this generally requires custom de-serialization to convert it into an idiomatic .NET type.

[GitHubRestTests.cs](https://github.com/ServiceStack/ServiceStack.Text/blob/master/tests/ServiceStack.Text.Tests/UseCases/GitHubRestTests.cs)

  1. Using [JsonObject](https://github.com/ServiceStack/ServiceStack.Text/blob/master/src/ServiceStack.Text/JsonObject.cs)
  2. Using Generic .NET Collection classes
  3. Using Customized DTO's in the shape of the 3rd party JSON response

[CentroidTests](https://github.com/ServiceStack/ServiceStack.Text/blob/master/tests/ServiceStack.Text.Tests/UseCases/CentroidTests.cs) is another example that uses the JsonObject to parse a complex custom JSON response. 


#TypeSerializer Details (JSV Format)

Out of the box .NET provides a fairly quick but verbose Xml DataContractSerializer or a slightly more compact but slower JsonDataContractSerializer. 
Both of these options are fragile and likely to break with any significant schema changes. 
TypeSerializer addresses these shortcomings by being both smaller and significantly faster than the most popular options. 
It's also more resilient, e.g. a strongly-typed POCO object can be deserialized back into a loosely-typed string Dictionary and vice-versa.

With that in mind, TypeSerializer's main features are:

 - Fastest and most compact text-serializer for .NET
 - Human readable and writeable, self-describing text format
 - Non-invasive and configuration-free
 - Resilient to schema changes
 - Serializes / De-serializes any .NET data type (by convention)
   + Supports custom, compact serialization of structs by overriding `ToString()` and `static T Parse(string)` methods
   + Can serialize inherited, interface or 'late-bound objects' data types
   + Respects opt-in DataMember custom serialization for DataContract dto types.

These characteristics make it ideal for use anywhere you need to store or transport .NET data-types, e.g. for text blobs in a ORM, data in and out of a key-value store or as the text-protocol in .NET to .NET web services.  
 
As such, it's utilized within ServiceStack's other components:
 - OrmLite - to store complex types on table models as text blobs in a database field and
 - [ServiceStack.Redis](https://github.com/ServiceStack/ServiceStack.Redis) - to store rich POCO data types into the very fast [redis](http://code.google.com/p/redis) instances.

You may also be interested in the very useful [T.Dump() extension method](http://www.servicestack.net/mythz_blog/?p=202) for recursively viewing the contents of any C# POCO Type.

---

# JSV Text Format (JSON + CSV)

Type Serializer uses a hybrid CSV-style escaping + JavaScript-like text-based format that is optimized for both size and speed. I'm naming this JSV-format (i.e. JSON + CSV) 

In many ways it is similar to JavaScript, e.g. any List, Array, Collection of ints, longs, etc are stored in exactly the same way, i.e:
	[1,2,3,4,5]

Any IDictionary is serialized like JavaScript, i.e:
	{A:1,B:2,C:3,D:4}

Which also happens to be the same as C# POCO class with the values 

`new MyClass { A=1, B=2, C=3, D=4 }`

	{A:1,B:2,C:3,D:4}

JSV is *white-space significant*, which means normal string values can be serialized without quotes, e.g: 

`new MyClass { Foo="Bar", Greet="Hello World!"}` is serialized as:

	{Foo:Bar,Greet:Hello World!}


### CSV escaping

Any string with any of the following characters: `[]{},"`
is escaped using CSV-style escaping where the value is wrapped in double quotes, e.g:

`new MyClass { Name = "Me, Junior" }` is serialized as:
	
	{Name:"Me, Junior"}

A value with a double-quote is escaped with another double quote e.g:

`new MyClass { Size = "2\" x 1\"" }` is serialized as:

	{Size:"2"" x 1"""}


## Rich support for resilience and schema versioning
To better illustrate the resilience of `TypeSerializer` and the JSV Format check out a real world example of it when it's used to [Painlessly migrate between old and new types in Redis](https://github.com/ServiceStack/ServiceStack.Redis/wiki/MigrationsUsingSchemalessNoSql). 

Support for dynamic payloads and late-bound objects is explained in the post [Versatility of JSV Late-bound objects](http://www.servicestack.net/mythz_blog/?p=314).


# Community Resources

  - [ServiceStack.Text has nice extension method called Dump and has a few friends](http://blogs.lessthandot.com/index.php/DesktopDev/MSTech/servicestack-text-has-a-nice) by [@chrissie1](https://twitter.com/chrissie1)
  - [JSON.NET vs ServiceStack](http://daniel.wertheim.se/2011/02/07/json-net-vs-servicestack/)
