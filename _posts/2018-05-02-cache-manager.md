---
layout: post
title: Benchmark ValueTask&lt;T&gt; vs Task&lt;T&gt;
---

`ValueTask<T>` 伴隨著 C# 7.0 推出一陣時間了，但因為抽換掉 `Task<T>` 不一定有幫助，有時甚至會[降低效能](https://stackoverflow.com/a/43003779)。因此大多數人的建議都是： Benchmark 比較再說。

>the default choice for any asynchronous method should be to return a Task or Task<TResult>. Only if performance analysis proves it worthwhile should a ValueTask<TResult> be used instead of Task<TResult>.


照著[這篇文章](https://blogs.msdn.microsoft.com/seteplia/2018/01/25/the-performance-characteristics-of-async-methods/)的思路，藉此驗證一下實際案例中替換成 ValueTask<T> 能否帶來成效。

----

這是一個快取的例子

```csharp
public sealed class CacheManager
{
	// one day into the future
	private static TimeSpan DefaultTimeSpan = new TimeSpan((long)24 * 60 * 60 * TimeSpan.TicksPerSecond);

	private static readonly ConcurrentDictionary<string, Tuple<DateTime, object>> _objectByString =
		new ConcurrentDictionary<string, Tuple<DateTime, object>>(StringComparer.OrdinalIgnoreCase);

	public void Clear()
	{
		_objectByString.Clear();
	}

	public Task<T> GetOrCreateObjectAsync<T>(object cacheKey, TimeSpan? expiresIn, Func<Task<T>> getFunc)
	{
		var key = cacheKey.ToString();

		if (_objectByString.TryGetValue(key, out var tuple) && tuple.Item1 >= DateTime.UtcNow)
		{
      // cache hit
			return Task.FromResult((T)tuple.Item2);
		}

    // cache miss
		return CreateAndCacheAsync(key, expiresIn, getFunc);

		async Task<T> CreateAndCacheAsync(string stringKey, TimeSpan? expires, Func<Task<T>> getter)
		{

			var val = await getter();
			var expirationTime = expires == TimeSpan.MaxValue
				? DateTime.MaxValue
				: DateTime.UtcNow.Add(expires ?? DefaultTimeSpan);
			var tuple2 = Tuple.Create(expirationTime, (object)val);
			_objectByString.AddOrUpdate(stringKey, tuple2, (k, v) => tuple2);
			return val;
		}
	}
}
```

用戶端使用方式

```csharp
int cacheKey = 0;
var cachedValue = _cacheManager.GetOrCreateObjectAsync(cacheKey, TimeSpan.MaxValue, 
  () => FetchDataAsync(cacheKey));
```

從用戶端代碼我們可以看到幾個問題:

1. ```stringKey``` 參數型別是 object，如果用戶端想要用單純的 value type(例如: int)當作 key，就會產生額外的裝箱(boxing)操作。
2. 匿名委派要使用的區域變數 ```cacheKey``` 會被當作捕獲變量(captured variable)而產生匿名物件。

為了消除這些 heap allocate 我們改成

```csharp
// WithoutCapturedVariable
public Task<T> GetOrCreateObjectV2Async<TKey, T>(TKey cacheKey, TimeSpan? expiresIn, Func<TKey, Task<T>> getFunc)
{
	var key = cacheKey.ToString();

	//...

	async Task<T> CreateAndCacheAsync(string stringKey, TKey rawkey, TimeSpan? expires, Func<TKey, Task<T>> getter)
	{
		var val = await getter(rawkey);
		//...
		return val;
	}
}
```

Benchmark結果如下:

|                                           Method |     Mean |     Error |    StdDev |  Gen 0 | Allocated |
|:----------------------------------------------- |---------:|----------:|----------:|-------:|----------:|
|                          GetOrCreateObjectAsync | 175.0 ns | 0.8471 ns | 0.7073 ns | 0.0710 |     224 B |
|                         WithoutCapturedVariable | **163.4** ns | 0.3946 ns | 0.3691 ns | 0.0558 |     **176** B |

------

用戶端可以採用更積極的方式快取`getFunc`委派參數，以減少記憶體配置

|                                          Method |     Mean |     Error |    StdDev |  Gen 0 | Allocated |
|:----------------------------------------------- |---------:|----------:|----------:|-------:|----------:|
|                         WithoutCapturedVariable | 163.4 ns | 0.3946 ns | 0.3691 ns | 0.0558 |     176 B |
|           WithoutCapturedVariable_CacheDelegate | **156.0** ns | 0.2541 ns | 0.1984 ns | 0.0355 |     **112** B |

------

接下來我們來試看看用 ValueTask<T> 取代 Task<T>，完整的benchmark結果如下：

Cache Miss

|                                            Method |     Mean |     Error |    StdDev |  Gen 0 | Allocated |
|:----------------------------------------------- |---------:|----------:|----------:|-------:|----------:|
|                          GetOrCreateObjectAsync | 173.4 ns | 0.4277 ns | 0.3791 ns | 0.0710 |     **224** B |
|                         WithoutCapturedVariable | 164.8 ns | 0.3074 ns | 0.2725 ns | 0.0558 |     176 B |
|           WithoutCapturedVariable_CacheDelegate | **154.2** ns | 0.5140 ns | 0.4292 ns | 0.0355 |     112 B |
|                ValueTask_GetOrCreateObjectAsync | 170.5 ns | 0.4437 ns | 0.4151 ns | 0.0381 |     120 B |
|               ValueTask_WithoutCapturedVariable | 168.0 ns | 0.4448 ns | 0.4161 ns | 0.0305 |      96 B |
| ValueTask_WithoutCapturedVariable_CacheDelegate | **158.7** ns | 0.3101 ns | 0.2749 ns | 0.0100 |      **32** B |

Cache Hit

|                                          Method |     Mean |     Error |    StdDev |  Gen 0 | Allocated |
|:----------------------------------------------- |---------:|----------:|----------:|-------:|----------:|
|                          GetOrCreateObjectAsync | 173.4 ns | 0.3568 ns | 0.3338 ns | 0.0710 |     224 B |
|                         WithoutCapturedVariable | 162.4 ns | 0.4580 ns | 0.4060 ns | 0.0558 |     176 B |
|           WithoutCapturedVariable_CacheDelegate | **154.5** ns | 0.2841 ns | 0.2658 ns | 0.0355 |     112 B |
|                ValueTask_GetOrCreateObjectAsync | 169.8 ns | 0.3438 ns | 0.3048 ns | 0.0379 |     120 B |
|               ValueTask_WithoutCapturedVariable | 166.6 ns | 0.2539 ns | 0.2251 ns | 0.0303 |      96 B |
| ValueTask_WithoutCapturedVariable_CacheDelegate | **156.8** ns | 0.2152 ns | 0.1797 ns | 0.0100 |      32 B |

結果顯示：雖然 ValueTask 沒有比 Task 快，但差距幾乎可以忽略，就看 Memory 與 GC 成本的優化值不值得了。

------

[完整的程式碼](https://github.com/mkjeff/Benchmark/blob/master/CacheManager.linq)
