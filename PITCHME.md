@title[Enumeration in .NET]

# Mastering Enumeration in .NET 
## IEnumerable and LINQ internals

**Antão Almada**<br>
*Principal Engineer @ Farfetch*<br>

@fa[creative-commons] @fa[creative-commons-by] 

---

```
public interface IEnumerable
{
	IEnumerator GetEnumerator();
}
```

---

Enumerable is a factory of enumerators!

---

```
public interface IEnumerator
{
    object Current { get; }
    
    bool MoveNext();
    void Reset();
}
```

---

Enumerator abstracts a stream from where data can be pulled!

---

- Read-only 
- Sequential
- **Nothing more!...** 

---

Producing an enumerable!

---

```
public static class Enumerable
{
    public static IEnumerable<T> Empty<T>() => 
        new EmptyEnumerable<T>();
    
    readonly struct EmptyEnumerable<T> : IEnumerable<T> 
    {
        public IEnumerator<T> GetEnumerator() => 
            new Enumerator();
        
        IEnumerator IEnumerable.GetEnumerator() => 
            GetEnumerator();
            
        readonly struct Enumerator: IEnumerator<T>
        {
            public T Current => default;
            
            object IEnumerator.Current => Current;
            
            public bool MoveNext() => false;
                        
            public void Reset() => throw new NotSupportedException();
            
            public void Dispose() { /* nothing to do */ }
        }
    }
}
```

@[3-4]
@[6,14]
@[8-12]
@[16-24]

---

Consuming an enumerable!

---

```
public int Sum(IEnumerable<int> enumerable) 
{
	var sum = 0;
    foreach (var item in enumerable)
    	sum += item;
    return sum;
}
```

---

```
public int Sum(IEnumerable<int> enumerable) 
{
    using(var enumerator = enumerable.GetEnumerator())
    {
		var sum = 0;
        while(enumerator.MoveNext())
     		sum += enumerator.Current;
    	return sum;
    }
}
```

---

Transforming an enumerable!

---

```
public IEnumerable<int> Even(IEnumerable<int> enumerable) 
{
	var list = new List<int>();
	foreach (var item in enumerable)
		if ((item & 0x01) == 0)
			list.Add(list);
    return list;
}
```

---

```
public IEnumerable<int> Even(IEnumerable<int> enumerable) 
{
	foreach (var item in enumerable)
		if ((item & 0x01) == 0)
			yield return item;
}
```

---

```
public IEnumerable<T> Where<T>(IEnumerable<T> enumerable, Func<T, bool> predicate) 
{
	foreach (var item in enumerable)
		if (predicate(item))
			yield return item;
}
```

---

### References
- TODO

---

https://gitpitch.com/aalmada/EnumerationDotNet/master