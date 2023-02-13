Rudimentary performance timer allowing to get min, max, cumul and average processing time for a block of code. Meant to be **used by plugin developers**.

I apologize in advance to the experienced programmers here since I'll be going over some basic stuff to get the plugin set up, so this is accessible to new programmers and server owners.

# Adding Perf Timings to Plugins

## Referencing PerfTimings

To be able to use PerfTimings, you'll need to first reference it in your plugin:

```
// Requires: PerfTimings

namespace Oxide.Plugins
{
    ...
```

Then, to avoid having to query the Plugin Manager for our PerfTimings plugin, we can create an accessor that I'll be using to add PerfTimers in the next section:

```
...
namespace Oxide.Plugins
{
    [Info("Test Plugin", "author", "1.0.0")]
    [Description("Test Plugin")]
    class TestPlugin : CovalencePlugin
    {

        private PerfTimings PerfTimingsProperty
        {
            get
            {
                _perfTimings = _perfTimings ?? (PerfTimings)Manager.GetPlugin(nameof(PerfTimings));
                return _perfTimings;
            }
        }

        private PerfTimings _perfTimings = null;

        ...
```

If you would rather not use the null-coalescing operator (??), you can write it's expanded version:

```
        ...

        private PerfTimings PerfTimingsProperty
        {
            get
            {
                if (_perfTimings == null)
                {
                    _perfTimings = (PerfTimings)Manager.GetPlugin(nameof(PerfTimings));
                }
                return _perfTimings;
            }
        }

        ...
```

I just prefer the previous version since I'm used to it in C++, I believe it's unnecessary here, but in C++ it can help performance by removing a branching depending on the compiler (most modern compilers do this optimization by default).

What this property does, is only fetch the PerfTimings plugin if this property is called. So, we avoid having to fetch it on the plugin loaded event and only fetch it if needed. We do need to handle the case where the property returns a null value however, which we'll see next. 

## Adding PerfTimers

You need to manually add a performance timer in a plugin's code in order to be able to capture the timings later on.

Either create a PerfTimer by initializing one as a function member to get the timings of the local scope. Like this:
```
        ...

        protected override void LoadConfig()
        {
            PerfTimer timer = PerfTimingsProperty.CreatePerfTimer(Name, nameof(LoadConfig)))
            
            base.LoadConfig();
            // Some code here
        }

        ...
```

Or explicitly specify the scope you wish to monitor with the using directive: 
```
        ...
        protected override void LoadConfig()
        {
            base.LoadConfig();

            using (PerfTimingsProperty?.CreatePerfTimer(Name, nameof(LoadConfig)))
            {
                // Some code here
            }
                        
            // Some code here we don't wish to measure
        }

        ...
```

Note the null-conditional operator here (?), this specifies that the rest of the line is not to be executed if the current result is null. So, if our property named PerfTimingsProperty is null the code won't try to call the CreatePerfTimer method which would otherwise trigger an exception.

Note that this is for safety, as normally as long as we have the requirement define at the top of the file the required plugins will be loaded before our plugin is loaded.

## Naming Tags

`CreatePerfTimer` allows for multiple strings to be passed as arguments. Those strings will be joined using the `::` characters. So, using this here:
```
PerfTimingsProperty?.CreatePerfTimer("MyPlugin", "MyFunction")
```
will actually create a Perf Timer with the tag "MyPlugin::MyFunction"

## Advanced: Enabling/Disabling Capture

To minimize performance impact, PerfTimers are only created when we run a performance capture, otherwise they return a null object.

However, calling the CreatePerfTimer method every frame has its own performance impact, so I do not advise anyone to leave it on all the time. Especially if you are calling it in hooks that are called very frequently (like OnFrame).

In order to disable capture, you can use preprocessor directives to remove the calls entirely. Like this:

```
// Requires: PerfTimings

#define USE_PERF_TIMERS

namespace Oxide.Plugins
{
    [Info("Test Plugin", "author", "1.0.0")]
    [Description("Test Plugin")]
    class TestPlugin : CovalencePlugin
    {

#if USE_PERF_TIMERS
        private PerfTimings PerfTimingsProperty
        {
            get
            {
                _perfTimings = _perfTimings ?? (PerfTimings)Manager.GetPlugin(nameof(PerfTimings));
                return _perfTimings;
            }
        }

        private PerfTimings _perfTimings = null;
#endif

        ...

        protected override void LoadConfig()
        {
            base.LoadConfig();

#if USE_PERF_TIMERS
            using (PerfTimingsProperty?.CreatePerfTimer(Name, nameof(LoadConfig)))
#endif
            {
                // Some code here
            }
        }

        ...
    }
```

This is a conditional compilation: basically, it removes all the code in between #if and #endif if the symbol is not defined.


Then you would only have to comment out the define so the symbol doesn't exist in order not to call the PerfTimings plugin at all. You could even remove the plugin requirement if you so wished (// Requires: PerfTimings), in fact you can remove it to test you properly made your conditional directives.

```
/* 
 * Line below can be removed as mentionned, but sadly
 * cannot be excluded with conditional compilation 
 * directives.
 */
// Requires: PerfTimings

//#define USE_PERF_TIMERS

namespace Oxide.Plugins
{
    ...

```

# Starting a Capture

Once you've set up performance timers in the plugin you wish to test, you can start a performance capture by calling the command `perfcapture` with the number of milliseconds to run the capture for. e.g.

```
perfcapture 30000
```

to run a capture for 30 seconds.

The command will output something like this:
```
| Name                                                     | Min      | Max      | Avg      | Cumul    | Calls    | 
------------------------------------------------------------------------------------------------------------------- 
| AdminRadar::OnServerInitialized                          |    4.14s |    4.14s |    4.14s |    4.14s |        1 | 
| AdminRadar::OnServerInitialized::FillCache               |    4.13s |    4.13s |    4.13s |    4.13s |        1 | 
| Guardian::Init                                           | 177.64ms | 177.64ms | 177.64ms | 177.64ms |        1 | 
| Guardian::OnEntityTakeDamage                             |   0.00ns | 249.60us | 379.63ns |   5.46ms |    14382 | 
| Guardian::OnEntityDeath                                  | 100.00ns |   1.23ms |  52.15us |   2.50ms |       48 | 
| AdminRadar::OnEntitySpawned                              | 200.00ns | 645.20us |   4.19us |   2.30ms |      549 | 
| AdminRadar::OnEntityKill                                 | 200.00ns |  19.70us |   2.24us |   1.28ms |      570 | 
| AdminRadar::OnEntityDeath                                | 800.00ns | 850.50us |  21.24us |   1.02ms |       48 | 
| Guardian::OnFireBallDamage                               |   0.00ns |   7.50us |  74.87ns | 883.30us |    11797 | 
| Guardian::LoadConfig                                     | 684.70us | 684.70us | 684.70us | 684.70us |        1 | 
| Guardian::OnServerInitialized                            | 443.60us | 443.60us | 443.60us | 443.60us |        1 | 
| Guardian::OnEntityKill                                   |   0.00ns | 150.70us | 360.00ns | 205.20us |      570 | 
| AdminRadar::Init                                         | 117.30us | 117.30us | 117.30us | 117.30us |        1 | 
| Guardian::OnFireBallSpread                               | 700.00ns | 105.30us |  15.84us | 110.90us |        7 | 
| Guardian::Loaded                                         | 106.30us | 106.30us | 106.30us | 106.30us |        1 | 
| Guardian::OnItemDropped                                  |   1.00us |   4.50us |   1.54us |  12.30us |        8 | 
| Guardian::Configuration::Save                            | 800.00ns | 800.00ns | 800.00ns | 800.00ns |        1 | 
| AdminRadar::Loaded                                       | 100.00ns | 100.00ns | 100.00ns | 100.00ns |        1 | 
| Guardian::LoadDefaultMessages                            | 100.00ns | 100.00ns | 100.00ns | 100.00ns |        1 | 
------------------------------------------------------------------------------------------------------------------- 
| Total                                                    |  33.31ms | 125.00ms |  36.08ms |  300.04s |     8315 | 
```

Giving you min, max, average and cumulative run times and the number of calls for each performance timer you created, plus the overall min, max, average and cumulative run times for the frames captured and the number of frames in the capture.
