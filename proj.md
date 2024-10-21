# About Me
## Academics (2008-2015)
- PhD (MathPhys, 2008-2013)
  - Statistical mechanics (quantum/classical)
  - Emergence of chaos in physical systems
  - Numerical simulations
- NSF Research Fellow (2013-2015) at Rice
  - Same as above

## Industry (2015+)
- Large scale (distributed) image processing
- Flight software (civilian space, unmanned, deep space exploration)
- Consumer electronics (neural networks on low power audio devices)
- Defense (2020+)
  - Numerical simulations of imaging hardware
  - Image processing (mostly traditional)
  - Real-time signal processing/control software
  - Mission planning software (read: constraint optimization)
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
# About This Presentation
- **Does NOT include any confidential/classified information**
- Generic lightweight framework for real-time async signal processing
- Severely watered down
- The "framework" is relatively simple; nontrivialities are in the algos
  - A bit more about this later...
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
# Requirements
- Modern C++ (say, 11+)
- Should execute on all "common" systems + easily portable to an RTOS
- Developers only worry about the algorithms
- Simple, explicit, intuitive
- Deliver only *the minimum functionality required*:
  - Tasks can be started/paused/stopped, asked for status
  - Tasks exist in a "vaccuum" (agnostic to the rest of the universe)
  - Nothing ever blocks indefinitely
  - Scheduling left to OS (although on an RTOS can be explicit)
- No exceptions
- No dynamic memory allocation
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
# But Why?
- Sophisticated 3rd party may be overkill
- Customers might not have approved industry solutions (e.g. ROS)
- **Single process, multiple threads**
- Customers like things done explicitly
- May need to conform to standards/certs
- Might not be (easily) portable to your target
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
# Pie (kind of) in the Sky
```cpp
#include "pits.hpp"
#include "mycommon.hpp"

namespace io = pits::io
namespace mine = pits::mine;
namespace policy = pits::io::policy;
namespace common = pits::common;
namespace osal = pits::osal;

int main(int argc, char **argv) {
    int retval{};

    auto argparser = common::ArgParser(argc, argv);
    auto options = argparser.parse();
    if (options) {
        auto logger = mine::Logger(options);
        auto injector = mine::Injector(options);
        auto processor = mine::Processor(options);
        auto display = mine::Display(options);
        auto recorder = mine::Recorder(options);
        auto ui = mine::UI(options);
        auto watchdog = mine::Watchdog(options);

        // register direct P2P channels...
        io::Channel<mine::FrameData *,
                    io::size(30),
                    policy::DropStale> inp{};
        io::Channel<mine::FrameData *,
                    io::size(30),
                    policy::DropStale> out{};
        io::Channels<mine::ImagerControlEvents,
                    io::size(2),
                    policy::BusyReject> imcontrol{};
        io::Channels<mine::UIControlEvents,
                    io::size(2),
                    policy::BusyReject> uicontrol{}
        // other channels...

        // injector will put to inp
        injector.channels("input") = inp;
        injector.channels("control") = imcontrol;
        // processor will get from inp and put to out
        processor.channels("input") = inp;
        processor.channels("output") = out;
        processor.channels("control") = uicontrol;
        // display will get from out
        display.channels("input") = out;
        display.channels("control") = uicontrol;
        // recorder will get from inp
        recorder.channels("input") = inp;
        recorder.channels("control") = uicontrol;

        // display/recorder registered with ui...
        // ui control/im control channels registered with ui...

        // register the logger...
        injector.channels("telemetry", logger.info("injector"));
        injector.channels("errors", logger.error("injector"));
        // ... 
        watchdog.watch(injector);
        watchdog.watch(processor);
        // ...

        osal::createTask(watchdog,
                        "watchdog", osal::priority(10))
        osal.createTask(logger,
                        "logger", osal::priority(40));

        osal::createTask(injector,
                        "injector", osal::priority(20));
        osal::createTask(processor,
                        "processor", osal::priority(20));
        // ...
    }
    else
        retval = pits::dump(options.error());
    
    return retval;
}
```