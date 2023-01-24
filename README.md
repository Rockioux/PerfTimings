Rudimentary performance timer allowing to get min, max, cumul and average processing time for a block of code. Meant to be used by plugin developers.

Either create a PerfTimer by initializing one as a function member to get the timings of the local scope. Like this:
```
protected override void LoadConfig()
{
    PerfTimer timer = PerfTimings.CreatePerfTimer(Name, nameof(LoadConfig)))
    
    base.LoadConfig();
    config = Config.ReadObject<Configuration>();
    SaveConfig();
}
```

Or explicitely specify the scope you wish to monitor with the using directive: 
```
protected override void LoadConfig()
{
    using (PerfTimings.CreatePerfTimer(Name, nameof(LoadConfig)))
    {
        base.LoadConfig();
        config = Config.ReadObject<Configuration>();
        SaveConfig();
    }
}
```
