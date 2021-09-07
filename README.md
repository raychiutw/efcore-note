# efcore-note

1. 不要提早輸出到記憶體,盡量保持 IQuaryable 延遲查詢 DB 的時間點, 以下情境會輸出到記憶體, (Count, Any, ToList, First, Single, ForEach...)
2. 查看 EF Core 組出的 T-SQL 並觀察執行計畫,優化語法

步驟1 appsetting.json - 增加顯示 efcore sql `"Microsoft.EntityFrameworkCore.Database.Command": "Information"`

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft": "Warning",
    "Microsoft.Hosting.Lifetime": "Information",
    "Microsoft.EntityFrameworkCore.Database.Command": "Information"
  }
},
```

步驟2 增加顯示變數值(注意機敏資訊問題) - ` options.EnableSensitiveDataLogging();`

```csharp
services.AddDbContext<EpkDbContext>(options =>
{
    options.UseSqlServer(
        configuration.GetConnectionString("XXXX"),
        sqlServerOption => sqlServerOption.CommandTimeout(180));
    options.EnableSensitiveDataLogging();
});
```

3. 當 EFCore 組出沒有效率的語法時,可以調整 linq 順序與寫法, 可能會有不同 sql
4. 最後可提早輸出 (Count, Any, ToList, First, Single, ForEach...), 善用後端資料量小運算快的優勢

其他參考說明

https://docs.microsoft.com/zh-tw/ef/core/performance/efficient-querying

https://docs.microsoft.com/zh-tw/ef/core/performance/efficient-updating

https://docs.microsoft.com/zh-tw/ef/core/querying/null-comparisons

## 額外加場 API 分頁優化寫法

old

```csharp
var result = this._mapper.Map<List<EPKAccount>>(_process.GetAccount(request.Account));

var page = PageConverter.GetPaged(result.OrderBy(x => x.Account).ToList(), request.CurrentPage, request.PageSize);

var returnData = new AccountSearchResponse()
{
    ReturnCode = "00",
    Accounts = page
};
```

new

```csharp
var accounts = this._process.GetAccount(request.Account, take, skip);
var result = this._mapper.Map<List<EPKAccount>>(accounts);

var total = this._process.GetTotalAccount(request.Account);
var returnData = new CarCoinSearchResponse()
{
    ReturnCode = "00",
    CarCoinReqs = new PagedResult<CarCoinReqDto>()
    {
        Results = result,
        CurrentPage = request.CurrentPage,
        PageSize = request.PageSize,
        PageCount = (int)Math.Ceiling((double)total / request.PageSize),
        RowCount = total
    }
};
```

修改 GetAccount, 增加 Take 與 Skip, 與增加 Total 相關程式

> Porcess

```csharp
public int GetAccount(string account, int take, int skip)
{
    var count = _repository.Find(account, take, skip);
    return count;
}

public int GetTotalAccount(string account)
{
    var count = _repository.Count(account);
    return count;
}
```

> Repository 修改

```csharp
public int GetAccount(string account, int take, int skip)
{
    var accounts = _context.EpkAccAccount
      .AsNoTracking()
      .Skip(skip)
      .Take(take)
      .Select(account)
      .ToList()
    
    return accounts;
}

public int Count(string account)
{
    var count = _context.EpkAccAccount
      .AsNoTracking()
      .Select(account)
      .Count();
    return count;
}
```
