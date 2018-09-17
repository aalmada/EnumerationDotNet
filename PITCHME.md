@title[Mastering Enumeration in .NET]

### Mastering Enumeration in .NET 




**Antão Almada**<br>
Principal Engineer<br>
Farfetch - Store of the Future<br>

---

### `IEnumerable`/`IEnumerator`

+++

## `IEnumerable` 
```
public interface IEnumerable
{
	IEnumerator GetEnumerator();
}
```

@[3] (Returns an instance of `IEnumerator`)

+++

## `IEnumerable` is a factory of enumerators!

+++

## `IEnumerator`
```
public interface IEnumerator
{
    object Current { get; }
    
    bool MoveNext();
    void Reset();
}
```

@[5] (Returns true if moved to next position. **Must be called first!**)
@[3] (Returns value at current position.)
@[6] (Resets to first position. **Implementation is optional!**)

+++

## `IEnumerator` abstracts a stream where data can be **pulled** from!

+++

## `IEnumerator`
- Pull
- Read-only 
- Sequential
- **Nothing more!...** 

---

# Producing an enumerable!

Provide an implementation of `IEnumerable`

+++

### `Enumerable.Empty<T>()`
```
public static class MyEnumerable
{
    public static IEnumerable<T> Empty<T>() => 
    	new EmptyEnumerable<T>();

    class EmptyEnumerable<T> : IEnumerable<T> 
    {
        public IEnumerator<T> GetEnumerator() => 
            new Enumerator();
        
        IEnumerator IEnumerable.GetEnumerator() => 
            new Enumerator();
            
        class Enumerator: IEnumerator<T>
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

@[3-4] (Returns instance of `EmptyEnumerable` as `IEnumerable`.)
@[6,14] (`IEnumerable` and `IEnumerator` implemented by private classes.)
@[7-12] (`IEnumerable` implementation.)
@[16-24] (`IEnumerator` implementation.)

+++

### `IEnumerable` implementations should:
- Store all the required data to perform the enumeration.
- Pass this data to all `IEnumerator` instances created.
- Be immutable!

+++

### `IEnumerator` implementations should:
- Manage state so that enumeration can be performed.
- Not share state with other instances.
- Throw `InvalidOperationException` when the collection was modified after the enumerator was created.

---

# Consuming an enumerable!

Pull values from an `IEnumerable`

+++

### Sum() 
```
public static int MySum(this IEnumerable<int> enumerable) 
{
	var sum = 0;
    foreach (var number in enumerable)
    	sum += number;
    return sum;
}
```

@snap[south-west]
[Live code](https://sharplab.io/#v2:D4AQDABCCMDcCwAocVoBYHMjAdAGQEsA7AR0yRAGZUA2KAJggFkBPAUSIFcBbAUwCcAhgCMANryQBvJBFlRqMOsQAuzFgGUeACmUALAgGdUlADwqAfBF5c+QsbwCUEGXOmI5HiADdB/CAZ4IAF4IMExPOQAzAHt+XkEAY10ILR8/G2EBCGIrGwERcQcXCI8A7ggAahCMgXCSkAB2fx462QBfJA7ECgVoOhBGGCapYvlaKDRUAA4tJ1G3Eu9fZvKQjh58+xwAJUEiAHNeLTAAGghoMCL3RbkcVk1uWdbPUY8YAE4tModnrragA===)
@snapend

+++

### `foreach` expands to...
```
public static int MySum(this IEnumerable<int> enumerable) 
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
@[3, 4, 9] (A call to `GetEnumerator()` method, later disposing the returned `IEnumerator`.)
@[6] (Loop while `MoveNext()` returns true. )
@[7] (Use `Current` property to get the value.)

+++

### `foreach` does not check for `null`!!!

- Methods that return `IEnumerable` must never return `null`. 
- Return `Enumerator.Empty<T>()` instead!!!

+++

### `foreach` uses 'duck typing'

- Implementation of `IEnumerable` and `IEnumerator` is not mandatory.
- Looks for a `GetEnumerator()` and then for `Current` and `MoveNext()` on the returned type.
- If it's a value-type:
	- It's not boxed!
	- Methods calls are direct (no virtual calls)!

+++

#### `System.Collections.Generic.List<T>`
```
public class List<T> : 
	ICollection<T>, IEnumerable<T>, IEnumerable, IList<T>,
    IReadOnlyCollection<T>, IReadOnlyList<T>, ICollection, IList
{
    public Enumerator GetEnumerator() => 
    	new Enumerator(this);
    IEnumerator<T> IEnumerable<T>.GetEnumerator() => 
    	new Enumerator(this);
    IEnumerator IEnumerable.GetEnumerator() => 
    	new Enumerator(this);

	public struct Enumerator : 
		IEnumerator<T>, IEnumerator, IDisposable
	{
        private List<T> list;
        private int index;
        private int version;
        private T current;

        Enumerator(List<T> list) { ... }           
        public T Current => current;
        public bool MoveNext() { ... }
        public void Reset() { ... }
        public void Dispose() {}
    }
}
```

@[5-6, 12-13] (Public `GetEnumerator()` returns the `Enumerator`, not the `IEnumerable`! )
@[7-10] (`GetEnumerator()` that return `IEnumerable` are implemented explicitly.)
@[12-13] (`Enumerator` is a value-type so method calls are more efficient!)
@[12-13] (**BEWARE:** Value-types are passed-by-copy when used as parameters!)

---

# Projecting/filtering an enumerable!

An implementation of `IEnumerable` that pulls values from another `IEnumerable`.

+++

### `Even()` 
#### 'Improper' implementation
```
public static IEnumerable<int> MyEven(
	this IEnumerable<int> enumerable) 
{
	var list = new List<int>();
	foreach (var item in enumerable)
		if ((item & 0x01) == 0)
			list.Add(item);
    return list;
}
```

@[1-2] (Method returns `IEnumerable`.)
@[4-8] (Stores all values into a collection and returns it.)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsWuyaAdADICWUAjsTnAMz4BsKATEgLICeAolAgBbAKZgAhgCMANqJwBvHEhUoWadmiYAeGsAB8PAQDdRUABTAAFlQDO+HXsNmR46XICUSZaqXZVAUjGEmBIMnbASAC8SFCiAO5I1LbAulAG5h7EgaoAZgD2YKISAMZWSObBoVTAosJINEguYpKyoh4+OQFUuRXmNXVIAGRICAAeCKheUTEIHf5di+EpZACCACbr/bXCWZ2BcADsYRHZqgC+OJfYjOqomlxox4r7zGwoBPgAHJneCyp+RZVJqmKAAOREUnE9higlcrTkZAAShIoABzUTmBAAGiQqDm+0WZD4/FBmTOOUJeUKxTKFWBQmEUOqUBBZghTOh80WAEg0ABOcyM5l7f5Ia7nIA=)
@snapend

+++

### `Even()`
#### 'Proper' implementation
```
public static IEnumerable<int> MyEven(
	this IEnumerable<int> enumerable) 
{
	foreach (var number in enumerable)
		if ((number & 0x01) == 0)
			yield return number;
}
```

@[6] (Returns one element on each iteration.)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsWuyaAdADICWUAjsTnAMz4BsKATEgLICeAolAgBbAKZgAhgCMANqJwBvHEhUoWadmiYAeGsAB8PAQDdRUABTAAFlQDO+HXsNmR46XICUSZaqXYAkABmAPZgohIAxlZI5sYSYEhCwlLiSDRILmKSsqIeOP7+VIEx5kkpCQBkSAgAHgioXgC8jdV5AQVoqCgA7Iki5cSqSAC+OKPYjOqomlxovYo+KsxsKAT4ABzmXotIfkNDcQmiplAAcv3i9i2CrtlyZABKElAA5qLmCAA0SKgIbfsApBkPj8E5bQaAnZDEJhSLRWLxPrJVLpY5mc7IsC2f6A/xoACcpQuYA8EJU42GQA)
@snapend

---

# `yield` keyword

+++

### `yield`

- Generates a state machine where:
    - `yield return <value>;`
        - Sets a new value to Current when `MoveNext()` is called.
        - Continues to the next statement.
    - `yield break;`
        - Exits the enumeration.

+++

### `yield`
```
public static IEnumerable<int> OneToFive()
{
	yield return 1;
	yield return 2;
	yield return 3;
	yield return 4;
	yield return 5;
	yield break;
	yield return 6;
}
```

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBM+A7EjgN45LsoVrVpkA8ASyjAAfEgDyUAKYAVAPYAxAQDcpACgCUbDq2wd9+VCkYZtBw8aQ1i59miNxGZG7fuWiZg28dIArC/M3ACMwKQBDAGsAr1QHRkpogF9PT3IqFAIkAFkwoU0mPR1PfQAzOVCwgGMACzVlMLAkASaoCWl5JVVNLULzAEg0AE41AQ0knESgA)
@snapend

+++

### `Range()`
```
public static IEnumerable<int> Range(int start, int count)
{
    var end = start + count;
    for (var value = start; value < end; value++)
    	yield return value;
}
```

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwBLKMAB8SAEo8oAc04AKZcCQBnYDzDAANEiNIAxgHtowAJTjJY7JJ9IAbhZInFAAJkgAvKbmlkgA1A7OKsS+kgBmjmBI+gGZAQIQnBFRFsDo/jz5hfJBoWV5BbGx7t4pvmioKADs5ZXJkgC+OIMkuNKosnRo3aIeEuRUKAT4ABz6rkizSF6t5Zls7HzcJkVMrBzc/EIAdFq6Bgg2qAiufSmbPulgnDz2ABZZOSQ+0OmWUQI4IJMzR2AEg0ABOfTA7gvTbDfpAA=)
@snapend

+++

### `Empty<T>()` II
```
public static IEnumerable<T> Empty<T>()
{
	yield break;
}
```

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwAVAHxJm7AA7BGS5QAoAlOMljsASDNpUSPmE48A1sUlIAvjnclc01LLpoAdiRRYwlyKhQCfAAOQ2DsF1MXFwAzAHs7HgBjAAskPQA3HjAkNnY+biQASygGFjLufiEAOnUtHRrgfQMjBOSJS1QATj0yirADZ0lPVyA==)
@snapend

+++

### `Where()`
```
public static IEnumerable<T> MyWhere<T>(
	this IEnumerable<T> enumerable, 
	Func<T, bool> predicate) 
{
	foreach (var item in enumerable)
		if (predicate(item))
			yield return item;
}
```

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwBLKMAB8SAEo8oAc04AKZcCQBnYDzDAANEiNIAxgHtowAJTjJY7JJ9IAbhZInFAAJkgAvKbmlkgA1A7OKsS+kgBmjmBI+gGZAQIQnBFRFsDo/jz5hfJBoWV5BbGx7t4pvmioKADs5ZXJkgC+HhJDUlT4CgAq6kwA6gAW3JzyU/ojPsBziibjy+rBHNz8QjZrknAArMs2fI6OAuoADmCcIYr2PMCcrkgjXq1I6WePHscyyOVsn3YtigNQOvEEX1OKUUqSyTxebw+BkUkNczX+/3aXQhnHYfQkg2wlNI0lQsjoaG6ohG5DGcAI+AAHPpvr8keVMmx2HxuNtIkxWHCjpwAHRaXQGBA2VAIfEEiQy2YLZ76IUizLhdS6jj6pAAMiQCAAHghUN9wpFVeTfPzAZxgaDsoE9dxoUgfWATGqUgBINAATmNwu4rmdlP6QA==)
@snapend

+++

### `Select()`
```
public static IEnumerable<U> MySelect<T, U>(
	this IEnumerable<T> enumerable, 
	Func<T, U> selector) 
{
	foreach (var item in enumerable)
        yield return selector(item);
}
```

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwBLKMAB8SAEo8oAc04AKZcCQBnYDzDAANEiNIAxgHtowAJTjJY7JJ9IAbhZInFAAJkgAvKbmlkgA1A7OKsS+kgBmjmBI+gGZAQIQnBFRFsDo/jz5hfJBoWV5BbGx7t4pvmioKADs5ZXJkgC+HhJDUlT4CgCq6kwAypxC9sDyACo2U/ojPsAAFoom4yvqwRzc/EI2m5JwAKwra+om85yLGa5II16tSOlgnDz22yyOVswE47FsUBqJ14gk4zS+PnaXVMTxeYEMoPYrj6EkG2DxpGkqFkdDQ3VEI3IYzgBHwAA59G8PpcJMC2Ow+Nx9pEmKxoWdOAA6LS6AwIGyoBDwhESQWzVHAfTszmZcLqZXcJAAKho2JZ7xaKR+fwBQMCGsyyiQFpM0pSAEg0ABOJUcFV6w1IPH9IA)
@snapend

+++

# Argument validation

+++

### `Where()` II
```
public static IEnumerable<T> MyWhere<T>(
	this IEnumerable<T> enumerable, 
	Func<T, bool> predicate) 
{
	if(enumerable == null)
		throw new ArgumentNullException(nameof(enumerable));
	if(predicate == null)
		throw new ArgumentNullException(nameof(predicate));
        
	foreach (var item in enumerable)
		if (predicate(item))
			yield return item;
}
```

@[5-8] (Argument validation)
@[10-12] (Operator implementation)
@[12] (`yield` makes all body lazily executed. **Including validation!**)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsWuyaAdADICWUAjsTnAMz4BsKATEgLICeAolAgBbAKZgAhgCMANqJwBvHEhUoWadmiYAeACoA+HrwDqAC3Gi9+gBTLV94KaoBnfDoNJRQsZNmiANEh29ipwAKx6gVIA9tEyhgAOYKIAJlQAxhLAogCUQdj2SgUhKlQAZtZeIuLSckgAvPVIQjIyOcElqo5g0QDuzaL9AIJgAObVUMAAchCt/AAe6aIJwFTRUNZQEmLRFVU+tbk5xJ1I5dZJqRlZog1NLW0dp919A8NjE9OzMgtLK2sbLY7CqXNKZbI5Y5PVTQlRlaLJCTpUxIawANwkYDO2WEZygnm8NT87WKp1KZVRoOu2WsVBxkNhZLQqBQAHZsaJhCdVABfHB87CMdSoTRcNDsxQdZhsFAEfAADmseQ6RVOGKx3ik4lcTUE1V8cjIACUJFBRqJrAhAqgECSyaoyHwzBZNt8oaSQoy0ABOawAIgAVEG/e7Toz4YjkejMRzcTRmiItWBnHbOgBIH20+nclQCnlAA)
@snapend

+++

### `Where()` III
```
public static IEnumerable<T> MyWhere<T>(
	this IEnumerable<T> enumerable, 
	Func<T, bool> predicate) 
{
	if(enumerable == null)
		throw new ArgumentNullException(nameof(enumerable));
	if(predicate == null)
		throw new ArgumentNullException(nameof(predicate));
        
	return MyWhere();

    IEnumerable<T> MyWhere()
    {
        foreach (var item in enumerable)
	        if (predicate(item))
    		    yield return item;
    }
}
```

@[12-17] (Move operator implementation into a local function!)
@[5-10] (Immediately executed!)
@[12-17] (Lazily executed!)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsWuyaAdADICWUAjsTnAMz4BsKATEgLICeAolAgBbAKZgAhgCMANqJwBvHEhUoWadmiYAeACoA+HrwDqAC3Gi9+gBTLV94KaoBnfDoNJRQsZNmiANEh29ipwAKx6gVIA9tEyhgAOYKIAJlQAxhLAogCUQdj2SgUhKlQAZtZeIuLSckgAvPVIQjIyOcElqo5g0QDuzaL9AIJgAObVUMAAchCt/AAe6aIJwFTRUNZQEmLRFVU+tbk5xJ1I5dZJqRlZog1NLW0dp919A8NjE9OzMgtLK2sbLY7CqXNKZbI5Y5PVTQ0IAdiMZgs1ihxRKsLcVkR5mSKIxRVO9jK0WSEnSpiQ1gAbhIwGdssIzlBPN4an52mjCfZypTQddstYqAzIRiuaFUKgUAihaJhCdOgBfDpK7AqxjqVCaLhoBGKDrMNgoAj4AAcKPyhQxNLp3ik4lcTUE1V8cjIACUJFBRqJrAhAqgEByxUgyHwkbiHqjThi0ABOawAIgAVCmE1HOhjiaTydTafTZUzmiI7WBnEHOgBIOOC4XylQqhVAA=)
@snapend

---

# Composition

+++

### Composition
```
var numbers = MyEnumerable.Range(0, 100)
	.MyWhere (number => (number & 0x01) == 0)
	.MySelect (number => number * 2);
	
foreach (var number in numbers)
	Console.WriteLine(number);
```

@[1-3] (Composition of statements)
@[5-6] (Enumeration of the composition)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwBLKMAB8SAEo8oAc04AKZcCQBnYDzDAANEiNIAxgHtowAJTjJY7JJ9IAbhZInFAAJkgAvKbmlkgA1A7OKsS+kgBmjmBI+gGZAQIQnBFRFsDo/jz5hfJBoWV5BbGx7t4pvmioKADs5ZXJkgC+HhJDUlT4CgAq6kwA6gAW3JzyU/ojPsBziibjy+rBHNz8QjZrknAArMs2fI6OAuoADmCcIYr2PMCcrkgjXq1I6WePHscyyOVsn3YtigNQOvEEX1OKUUqSyTxebw+BkUkNczX+/3aXQhnHYfQkgxawypoxkOwAqtNGABlThCezAK5IRmrGnrTbbOS7WFceHHH58s6XCY2RmmNmcDkZb6/JEAjKcYGg7KBHGk6Eiw4I/EEiREuDdEwKpVgQy48lISmU0jSVCyOhobqiEbkMZwAj4AAc+hVNL+rXBbHYfG420iTFYcKOnAAdFpdAYEDZUAgTaaU7MFs99FGY5lwuoSxwy0gAGRIBAADwQqG+4UiubVPgLLOtwCr0e4EXUpaHACokDRXA6fGrAZqQWDAqPMsokCuTHnfABINAATgHZenIydQA===)
@snapend

+++

### Composition
#### Result of improper implementation
```
var evenNumbers = new List<int>();
foreach (var number in MyEnumerable.Range(0, 100))
	if ((number & 0x01) == 0)
		evenNumbers.Add(number);
		
var doubleNumbers = new List<int>();		
foreach (var number in evenNumbers)
		doubleNumbers.Add(number * 2);

foreach (var number in doubleNumbers)
	Console.WriteLine(number);
```

@[1-4] (Improper `MyWhere()` would store even numbers in a list.)
@[6-8] (Improper `MySelect()` would store doubled numbers in another list)
@[] (**I'm sure this is not the behavior you want!!!**)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwBLKMAB8SAEo8oAc04AKZcCQBnYDzDAANEiNIAxgHtowAJTjJY7JJ9IAbhZInFAAJkgAvKbmlkgA1A7OKsS+kgBmjmBI+gGZAQIQnBFRFsDo/jz5hfJBoWV5BbGx7t4pvmioKADs5ZXJkgC+OIMkuNKosnRo3aIeEuRUKAT4ABz6rkizSF6t5ZmcfsEAchx83CZFUJwA7kgAMopmSiqqa30p6WCcPPYAFlk5SDY7FOmWUDBYQO4/CEADotLoDAgbKgEK5mjsfIpUll9ECQUgAGRIBAADwQqHW4UiqM2GMk+yOJzOMIAgiEQrimWBXMRaZIASFnIJOMdgWcLtc7g9gE81K8AJDyvkSD5fX7/QJ47i2KBBA5QUUgkzoum+QUQYWG5lsjlazIAKiQNB5OGVSFV3z+2U1XJ1SHNlq5xrdPjQAE5OWLuW8kMN+kA)
@snapend

+++

### Composition
#### Order is important
```
var numbers = Enumerable.Range(0, 100)
	.Select (number => number * 2)
	.Where (number => (number & 0x01) == 0);
	
foreach (var number in numbers)
	Console.WriteLine(number);
```

@[2] (Projection requires computing and memory.)
@[3] (Filtering reduces the number of projected elements, wasting resources.)

+++

### Composition
#### Order is important
```
var numbers = Enumerable.Range(0, 100)
	.Where (number => (number & 0x01) == 0)
	.Select (number => number * 2);
	
foreach (var number in numbers)
	Console.WriteLine(number);
```

@[2] (Filtering reduces the number of elements in sequence.)
@[3] (Projects only the filtered results.)

---

# Is the sequence empty?

+++

### `Average()`
```
public static double? Average(this IEnumerable<int> enumerable) 
{
    if(enumerable.Count() == 0)
        return null;

    var sum = 0;
    foreach (var value in enumerable)
        sum += value;

    return sum / (double)enumerable.Count();
}
```

@[3-4] (Check if number of elements in sequence is zero.)
@[6-8] (Enumerate sequence to get sum of elements.)
@[10] (Divide sum by the number of elements in sequence.)
@[3-4] (Avoids unnecessary sum calculation and division by zero!)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsWuyaAdADICWUAjsTnAMz4BsKATEgLICeAolAgBbAKZgAhgCMANqJwBvHEhUoWadgBMA9hFmiA/EgCCAN3ESA5qIAUwABZUAzviYAeGsAB8SUULGS+gCUSMqqStiqUUhUAGY2fiIW+mQAwrpQwDYhALw5SAhBYdHRcADsSEIyMgyRJUimEmBITiJI+QjE9Uix2mCiEgDG9kg2jc2NMhCiMVC+/slyRXXdKq3CSADU+ZPTtd3lLW0A9KM6ekuJAdJyaRlZQV2qAL44r9iM6qjscFxoFYpimo2CgCPgABzZUIrJARbrjSoiKTiFz5KCiADus2AAG0ALqwpCoAA0SA4pJYzyeJSBUTQAE4bP5kWAnGQzBZrNlHkD3s8gA===)
@snapend

+++

### Count()
```
public static int MyCount<T>(this IEnumerable<T> enumerable) 
{
    var counter = 0;
    foreach (var item in enumerable)
	    counter++;
    return counter;
}
```

@[3-6] (Enumerates all the sequence.)
@[3-6] (**O(n) complexity!**)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTU0ZADwBLKMAB8SAEo8oAc04AKZcCQBnYDzDAANEiNIAxgHtowAJTjJY7JJ9IAbhZInFAAJkgAvKbmlkgA1A7OKsS+kgBmjmBI+gGZAQIQnBFRFsDo/jz5hfJBoWV5BbGx7t4pvmioKADs5ZXJkgC+HhJDUlS2KgyMAMKJwPIAKqr6wAAWiib4Cos1HNz8Qq5II16t5ZlOLtxFCH0p6WCcPPYrWTm2wJzs4ztcvIKczVOPguKm4jVubW6II+YAhg2w8NI0lQsjoaG6ohG5DGcAI+AAHPpDscRj43mx2HxuBtIkxWLs/kIAHRaXQGBA2VAIVwQnykyRoACc+gpVLAJiZTBmLiJPJG8P6QA)
@snapend

+++

### `Any()`
```
public static bool MyAny<T>(this IEnumerable<T> enumerable) 
{
    using (var enumerator = enumerable.GetEnumerator())
    {
    	return enumerator.MoveNext();
    }
}
```

@[3-6] (Calls MoveNext() only once!)
@[3-6] (**O(1) complexity!**)

+++

### `Average()` II
```
public static double? Average(this IEnumerable<int> enumerable) 
{
    if(!enumerable.MyAny())
	    return null;

    var counter = 0;
    var sum = 0;
    foreach (var value in enumerable)
    {
        counter++;
        sum += value;
    }

    return sum / counter;
}
```

@[3-4] (Better performance!)
@[6-14] (One enumerations calculates number and sum of elements.)
@[3, 8] (Two allocations of `IEnumerator` instances.)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsWuyaAdADICWUAjsTnAMz4BsKATEgLICeAolAgBbAKZgAhgCMANqJwBvHEhUoWadlID2WmT14BBKLwA8AFQB8ACmAALKgGd8TcxaSihYybNEBKJMqqStiqoSjIVgBuEmDunuISwFqxALxxIgk+ZADiosCCGZJJYFa+voFhSMGVlXAA7Oleiclk3FqRogByogAewKXENUgAvhUjY2PMbEgAJloQPgD8SAYdkgDmojb2TmguNMBuHoXScv5j1TVUAGZWAITHTVl8Rryl5SFDqvVIQjIyDE+lWisQAxvMoMBxEg0ghBjUQUgHCIYUg4WNQtdkqIJKDbEgojEkNEZBBREgaI1MmcMUFaZVwdAoWAANQs+FfJEollpElkjlhUbYekoBrI4RIAD0SEZkPEAqFQsY6lQ7DgXDQDUUkxVaoI+AAHKUAkCqiLEZ4pOInGkoKIAO4UyEAbQAulUkKgADRIDg+ljDAWhEVoACcVkt1rIqwSm3eCpwwyAA===)
@snapend

+++

### `Average()` III
```
public static double? Average(this IEnumerable<int> enumerable) 
{
    var counter = 0;
    var sum = 0;
    foreach (var value in enumerable)
    {
        counter++;
        sum += value;
    }

    if (counter == 0)
	    return null;

    return sum / counter;
}
```

@[5] (Only enters `foreach` if sequence not empty. Same as `Any()`.)
@[3-14] (**One single allocation! One single enumeration!**)

@snap[south-west]
[Live code](https://sharplab.io/#v2:C4LgTgrgdgPgAgBgARwIwG4CwAoRLUAsW2OcAzPgGwoBMSAsgJ4CiUEAtgKZgCGARgBtOOAN44kElBTTUAJgHsIgzgH4kAQQBu3HgHNOACmAALAJYBnfGQA8pqMAB8STmy69lASiTjJY7JICkTR4wJABjRXtuJABeJARiQMlg0PMOWPjEpKQAM3kwTh4w4yQDFKCeAQhOJDtnVx1PHyS/bMCI6GBuAGpurLaJNPYkbrjgqs5+wIBfZsC5gNMc0o6o0Ji4hA8F7LgAdiQ2AQFiHYl9pCGkAHpwyK6wKdnsZ9JpVGo4OjQD0TnyKgoAj4AAcBi8c1a2XKrj43EscSgnAA7rV7ABtAC6SBESBoABokGRCcDplMAmd8ABOAyw+EAOi0On04I8Txw0yAA)
@snapend

---

### Collection interfaces
- Always expose the highest admissible level interface of the returned collection, and consume the lowest admissible level interface. 
- `IEnumerable`, `IReadOnlyCollection`, `IReadOnlyList` and `IReadOnlyDictionary` maintain immutability.

+++

### `IReadOnlyCollection` 
```
    public interface IReadOnlyCollection<out T> : 
    	IEnumerable<T>, IEnumerable
    {
        int Count { get; }
    }

```

@[2] (Inherites `IEnumerable` behavior.)
@[4] (Returns the number of elements.)

+++

### `IReadOnlyList` 
```
    public interface IReadOnlyList<out T> : 
    	IReadOnlyCollection<T>, 
    	IEnumerable<T>, IEnumerable
    {
        T this[int index] { get; }
    }
```

@[2] (Inherites `IReadOnlyCollection` and `IEnumerable` behaviors.)
@[6] (Allows random access.)

+++

```
public static IReadOnlyList GetProperties()
{
    return new Property[] {
        new Property("Property 1"),
        new Property("Property 2"),
        new Property("Property 3"),
        new Property("Property 4"),
        new Property("Property 5"),
    };
}
```

@[1] (Exposes all the read-only features of the array.)

+++

```
public static double? Average(this IReadOnlyCollection<int> enumerable) 
{
    if (enumerable.Count == 0)
        return null;
    
    var sum = 0;
    foreach (var value in enumerable)
        sum += value;
    
    return sum / (double)enumerable.Count;
}
```

@[1,3] (Allows the use of `Count`.)

---

# `First()` and `Single()`

+++

### `First()` and `Single()`
- Both return first element of a sequence.
- Both throw `InvalidOperationException` is sequence is empty.
- `Single()` throws `InvalidOperationException` if sequence contains more than one element.

+++

### `First()`
```
public static T MyFirst<T>(this IEnumerable<T> enumerable)
{
    using(var enumerator = enumerable.GetEnumerator())
    {
        if(!enumerator.MoveNext())
          throw new InvalidOperationException(
          	"Sequence contains no elements");

        return enumerator.Current;
    }
}
```

@[5] (One call to `MoveNext()`.)
@[6-7] (Throws is sequence is empty.)
@[9] (Otherwise, returns first element.)

+++

### `Single()`
```
public static T MySingle<T>(this IEnumerable<T> enumerable)
{
    using(var enumerator = enumerable.GetEnumerator())
    {
        if(!enumerator.MoveNext())
          throw new InvalidOperationException(
          	"Sequence contains no elements");

        var first = enumerator.Current;

        if(enumerator.MoveNext())
          throw new InvalidOperationException(
          	"Sequence contains more than one element");

        return first;
    }
}
```

@[5-9, 15] (Just like `First()`.)
@[11-13] (One more call to `MoveNext()` to check if any more elements.)

+++

### Empty sequence may be a valid result.
## Throwing is expensive!

+++

### FirstOrDefault()
```
public static T MyFirstOrDefault<T>(this IEnumerable<T> enumerable)
{
    using(var enumerator = enumerable.GetEnumerator())
    {
        if(!enumerator.MoveNext())
          return default;

        return enumerator.Current;
    }
}
```

@[6] (Returns `default` is sequence is empty.)

+++

### SingleOrDefault()
```
public static T MySingleOrDefault<T>(this IEnumerable<T> enumerable)
{
    using(var enumerator = enumerable.GetEnumerator())
    {
        if(!enumerator.MoveNext())
          return default;

        var first = enumerator.Current;

        if(enumerator.MoveNext())
          throw new InvalidOperationException(
          	"Sequence contains more than one element");

        return first;
    }
}
```

@[6] (Returns `default` is sequence is empty.)
@[11-12] (Still throws if more than one element.)

---

# O(1) Complexity! 

+++

## `Any()`, `First()`, `Single()`, `FirstOrDefault()` and `SingleOrDefault()`, all have O(1) complexity!

+++

# Sorry but, not exactly!!!

+++

These methods are usually preceded by a filter...

+++

```
var customer = Customers()
	.Where(customer => customer.Name = "John Doe")
	.FirstOrDefault();
```

+++

There's usually an equivalent overload that takes the filter as a parameter...

+++

```
var customer = Customers()
	.FirstOrDefault(customer => customer.Name = "John Doe");
```

+++

Let's look inside...

+++

### `FirstOrDefault()` II
```
public static T FirstOrDefault<T>(
	this IEnumerable<T> enumerable, 
	Func<T, bool> predicate)
{
	foreach(var item in enumerable)
		if(predicate(item))
			return item;

	return default;
}
```

@[5-7] (Returns first element that makes `predicate` return `true`.)
@[9] (Returns `default` if all return false.)

+++

When combined with a filter (which is usual)<br/>
`First()` and `FirstOrDefault()` have O(n) complexity!<br/>
*n* is the position of the **first element** that makes `predicate()` return `true`.

+++

### `SingleOrDefault()` II
```
public static T SingleOrDefault<T>(
	this IEnumerable<T> enumerable, 
	Func<T, bool> predicate)
{
    using(var enumerator = enumerable.GetEnumerator())
    {
        while(enumerator.MoveNext())
        {
            var current = enumerator.Current;
            if (predicate(current))
            {
                // found first
                // keep going until find second or reach end
                while (enumerator.MoveNext())
                {
                    if (predicate(enumerator.Current))
                        throw new InvalidOperationException(
                        	"Sequence contains more than one element");
                }
                return current;
            }
        }
        return default;
    }
}
```

@[12-20] (Continues until a second element makes `predicate()` return `true`.)

+++

When combined with a filter (which is usual)<br/>
Single() and SingleOrDefault() have O(n) complexity!<br/>
*n* is the position of the **second element** the makes `predicate()` return `true`.

+++

### `SingleOrDefault()` 
```
var customer = Customers()
	.SingleOrDefault(customer => customer.Id = 42);
```

@[1-2] (`Id` is an unique identifier.)
@[1-2] (No second element will make `predicate` return `true`.)
@[1-2] (** *Single()* will enumerate all sequence!!! **)

+++

Only use `Single()` or `SingleOrDefault()` when singularity checking is mandatory!!!

+++

**If you want to search, use a data structure that supports it!**

Most commonly used is `Dictionary<T>` but there are others...

---

## LINQ optimizations

+++

### LINQ optimizations
- Examples shown here are simplified versions.
- LINQ contains optimizations that result in better complexities, but...
- Small changes in project may unexpectedly invalidate these... 

+++

### LINQ optimizations

```
public static bool MyMethod(this IEnumerable<int> enumerable) 
{
    if(enumerable.Count() == 0)
    	return false;

    return true;
}

public static void Main() 
{
    var collection = Enumerable.Range(0, 10)
    	.ToList();

    Console.WriteLine(collection.MyMethod());
}

```

@[11-12] (Implements `ICollection`.)
@[3,14] (Complexity O(1).)
+++

### LINQ optimizations

```
public static bool MyMethod(this IEnumerable<int> enumerable) 
{
    if(enumerable.Count() == 0)
    	return false;

    return true;
}

public static void Main() 
{
    var collection = Enumerable.Range(0, 10)
    	.ToList();

    Console.WriteLine(collection.MyEven().MyMethod());
}

```

@[3,14] (No longer implements `ICollection`. Complexity is O(n)!)

---

## IQueryable

+++

### IQueryable
```
public interface IQueryable : 
	IEnumerable 
{
	Expression Expression { get; }
	Type ElementType { get; }
	IQueryProvider Provider { get; }
}

public interface IQueryable<out T> : 
	IEnumerable<T>, IQueryable 
{
}
```

@[1-2, 9-10] (Implements `IEnumerable` so LINQ can be used.)

+++

`IQueryable` is an enumeration interface that converts the LINQ expression tree into something equivalent that a database can process.

+++

Entity Framework, LinqToExcel, LinqToTwitter, LinqToCsv, LINQ-to-BigQuery, LINQ-to-GameObject-for-Unity, ElasticLINQ, GpuLinq 

+++

## Entity Framework Core

- object-relational mapping (ORM)
- “LINQ-to-SQL” - supports LINQ queries on a SQL database engine.

+++

```
using (var db = new BloggingContext())
{
    var blogs = db.Blogs
        .Where(blog => blog.Rating > 3)
        .OrderByDescending(blog => blog.Rating)
        .Take(5)
        .Select(blog => blog.Url);

    Console.WriteLine(blogs.ToSql());
    Console.WriteLine();

    foreach (var blog in blogs)
    {
        Console.WriteLine(blog);
    }
}
```

@[3] (`DbSet<T>` implements the `IQueryable` interface.)
@[3-7] (LINQ is used to define the query.)

+++

### LINQ expression converted to SQL
```
SELECT TOP(5) [blog].[Url]
FROM [Blogs] AS [blog]
WHERE [blog].[Rating] > 3
ORDER BY [blog].[Rating] DESC
```

+++

### 'Improper' query projection
```
using (var db = new BloggingContext())
{
    var blogs = db.Blogs
        .Where(blog => blog.Rating > 3)
        .OrderByDescending(blog => blog.Rating)
        .Take(5)
        .Select(blog => blog.Url)
        .ToList();

    Console.WriteLine(blogs.ToSql());
    Console.WriteLine();

    var urls = blogs
        .Select(url => $"URL: {url}");

    foreach (var url in urls)
    {
        Console.WriteLine(url);
    }
}
```

@[3-8] (Store the result of the query in a `List<T>`.)
@[13-14] (Projects the `List<T>` elements.)

+++

### 'Proper' query projection
```
using (var db = new BloggingContext())
{
    var blogs = db.Blogs
        .Where(blog => blog.Rating > 3)
        .OrderByDescending(blog => blog.Rating)
        .Take(5)
        .Select(blog => blog.Url);

    Console.WriteLine(blogs.ToSql());
    Console.WriteLine();

    var urls = blogs
    	.AsEnumerable()
        .Select(url => $"URL: {url}");

    foreach (var url in urls)
    {
        Console.WriteLine(url);
    }
}
```

@[12-14] (Projects the query elements directly.)
@[12-14] (Everything before `AsEnumerable()` is executed by the database.)
@[12-14] (Everything after `AsEnumerable()` is executed in memory.)

+++

### DataStax C# Driver for Apache Cassandra

- Uses CQL for querying.
- Not all LINQ operations are supported!

+++

```
var clusterBuilder = Cluster.Builder()
    .AddContactPoint("localhost")
    .WithPort(9042);

using (var cluster = clusterBuilder.Build())
using (var session = cluster.Connect("blogging"))
{
    var blogsTable = new Table<BlogByRating>(session);

    var blogs = blogsTable
        .Where(blog => blog.Rating == 3)
        .Select(blog => blog.Url);

    Console.WriteLine(blogs);
    Console.WriteLine();

    foreach (var blog in await blogs.ExecuteAsync())
    {
	    Console.WriteLine(blog);
    }
}

```

@[8] (`CqlQuery<T>` implements the `IQueryable` interface.)
@[10-12] (LINQ is used to define the query.)

+++

### LINQ expression converted to CQL
```
SELECT "url" FROM "blogs_by_rating" WHERE "rating" = ?
```

---

### `IEnumerable` bullying!

```
var viewModel = new ViewModel();
viewModel.Answer1 = model.Replies
    .OrderBy(c => c.ReplyId).Skip(0).Take(1)
    .FirstOrDefault().AnswerId;
viewModel.Answer2 = model.Replies
    .OrderBy(c => c.ReplyId).Skip(1).Take(1)
    .FirstOrDefault().AnswerId;
viewModel.Answer3 = model.Replies
    .OrderBy(c => c.ReplyId).Skip(2).Take(1)
    .FirstOrDefault().AnswerId;
viewModel.Answer4 = model.Replies
    .OrderBy(c => c.ReplyId).Skip(3).Take(1)
    .FirstOrDefault().AnswerId;
viewModel.Answer5 = model.Replies
    .OrderBy(c => c.ReplyId).Skip(4).Take(1)
    .FirstOrDefault().AnswerId;    
```

**Source:** http://www.kspace.pt/posts/ienumerable-bullying

+++

### `IEnumerable` awareness!

```
var viewModel = new ViewModel();
var orderedReplies = model.Replies.OrderBy(c => c.ReplyId);
using(var enumerator = orderedReplies.GetEnumerator())
{
    enumerator.MoveNext();
    viewModel.Answer1 = enumerator.Current.AnswerId;
    enumerator.MoveNext();
    viewModel.Answer2 = enumerator.Current.AnswerId;
    enumerator.MoveNext();
    viewModel.Answer3 = enumerator.Current.AnswerId;
    enumerator.MoveNext();
    viewModel.Answer4 = enumerator.Current.AnswerId;
    enumerator.MoveNext();
    viewModel.Answer5 = enumerator.Current.AnswerId;
}  
```

---

<table>
	<tr>
		<th>@fa[twitter]</th>
		<th>[@AntaoAlmada](https://twitter.com/AntaoAlmada) </th>
	</tr>
	<tr>
		<th>@fa[medium]</th>
		<th>[@antao.almada](https://medium.com/@antao.almada)</th>
	</tr>
	<tr>
		<th>@fa[github]</th>
		<th>[aalmada](https://github.com/aalmada) </th>
	</tr>
</table>


