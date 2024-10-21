# About Me
## Academics (2008-2015)
- PhD (MathPhys, 2008-2013)
  - Statistical mechanics (quantum/classical)
  - Emergence of chaos in physical systems
  - Numerical simulations
- NSF Research Fellow (2013-2015) at Rice
  - Same as above (publishing, lecturing, wearing silly hats)

## Industry (2015+)
- Large scale (distributed) image processing
- Flight software (civilian space, unmanned, deep space exploration)
- Consumer electronics (neural networks on low power audio devices)
- Defense (2020+)
  - Numerical simulations of imaging hardware/imaging scenarios
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
...........
<Br>
- (my) first attempt was done as part of a custom middle layer for NASA's
cFS
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
#include "mine.hpp"

namespace io = pits::io
namespace mine = pits::mine;
namespace policy = pits::io::policy;
namespace common = pits::common;
namespace osal = pits::osal;

namespace glob {
// stacks for the tasks allocated somewhere, maybe here,
// statically (see createTask() calls below)
}

int main(int argc, char **argv) {
    int retval{};

    // The following is typically not done explicitly
    // as shown, but set up from a config file by a
    // runner (more on this later...)

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
        io::Channel<mine::FrameData,
                    io::size(30),
                    policy::DropStale> inp{};
        io::Channel<mine::FrameData,
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
        injector.channels("telemetry") = 
            logger.info("injector");
        injector.channels("errors") =
            logger.error("injector");
        // ... 
        watchdog.watch(injector);
        watchdog.watch(processor);
        // ...

        osal::createTask(watchdog,
                        "watchdog",
                        osal::priority(10),
                        glob::watchdogStkSize,
                        glob::watchdogStk
        );
        osal.createTask(logger,
                        "logger",
                        osal::priority(40)
                        glob::loggerStkSize,
                        glob::loggerStk
        );

        osal::createTask(injector,
                        "injector",
                        osal::priority(20),
                        glob::injectorStkSize,
                        glob::injectorStk
        );
        // ...
    }
    else
        retval = pits::dump(options.error());
    
    return retval;
}
```

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

# Anatomy of Tasks and Channels
- Tasks do not at all handle data, only process it
  - For example, what if data queue is full?
    - Don't care, not our problem
    - This is handled by channels themselves through policies
    - All policies are nonblocking
  - Data production/consumption becomes part of task's execution block (critical section?)
  - To avoid blocking channels, tasks copy data to local buffer and exit critical section
  - Tasks should use timeout locking mechanisms (if supported), but this isn't enforced
  - The time it takes to copy data figures into task's schedule
- Channels guarantee thread safety
  - Channels technically do not block, except for the time it takes for a task to consume from or produce to a channel
  - Consumption/production are generally independent (do not use the same lock)
- No critical section control necessary to handle events (events are queued on a thread-safe channel)
- No event signals raised (tasks check events on each iteration of the thread
loop, so tasks are effectively "deaf" and cannot be interrupted by an event
during main loop execution past the event check)

For example...
```cpp
namespace pits {
    class TaskInterface {
        public:
            // ... snip-snip ...
            virtual void execute() = 0;
            // ... snip-snip ...
    };
}
// ... snip-snip ...

namespace pits::mine {
    class Processor : public TaskInterface {
        public:
            // ... snip-snip ...
            virtual void execute() override;
            // ... snip-snip ...
    };
    // ... snip-snip ...

    void Processor::execute() {
        while (status_ == common::TaskStatus::ACTIVE) {
            // will take last event from the internal
            // event queue(s) and process it...
            handleEvents_();
            if (execute_) {
                // enter main task execution...
                inp_.get(currentFrame_);
                if (currentFrame_ &&
                    currentFrame.timestamp() > lastts_) {
                  // ... snip-snip ...
                }
                // ... snip-snip ...
            }
            // desaturate ... maybe sleep()?
        }
    }
}
```

- Tasks can be paused/resumed/ended, etc
- No need to keep track of a complicated state machine, everything is handled by
event queues
- Tasks are **dumb** and there is no concept of interrupts/signals/conditions
- With a bit of work, fine-grained scheduling can be added (+ affinity control,
etc), especially when porting to an RTOS.
- Not shown here: support for config files (high level config scripts - specify
task name, stack size, channel names and channel parameters, etc, in a config
file, load tasks as shared libs)
- With carefully controlled scheduling, can be essentially lockless
  - e.g., no need to copy from channel to local buffer
- Single process, multiple threads
- No explicit data bus (e.g., pub/sub) with quality of service, etc... but
channels can be implemented to sit on top of some DBUS
  - at this point you're essentially entering ROS-land