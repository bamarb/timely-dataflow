# Creating Operators

What if there isn't an operator that does what you want to do? What if what you want to do is better written as imperative code rather than a tangle of dataflow operators? Not a problem! Timely dataflow has you covered.

Timely has several "generic" dataflow operators that are pretty much ready to run, except someone (you) needs to supply their implementation. This isn't as scary as it sounds; you just need to write a closure that says "given a handle to my inputs and outputs, what do I do when timely asks me to run?".

Let's look at an example

```rust,ignore
use timely::dataflow::operators::ToStream;
use timely::dataflow::operators::generic::operator::Operator;
use timely::dataflow::channels::pact::Pipeline;

timely::example(|scope| {
    (0u64..10)
        .to_stream(scope)
        .unary(Pipeline, "increment", |capability|
            |input, output| {
                while let Some((time, data)) = input.next() {
                    let mut session = output.session(&time);
                    for datum in data.drain(..) {
                        session.give(datum + 1);
                    }
                }
            }
        );
});
```

What is going on here? The heart of the mess is the dataflow operator `unary`, which is a ready-to-assemble dataflow operator with one input and one output. The `unary` operator takes three arguments (it looks like so many more!): (i) instructions about how it should distribute its inputs, (ii) a tasteful name, and (iii) the logic it should execute whenever timely gives it a chance to do things.

Most of what is interesting lies in the closure, so let's first tidy up some loose ends before we dive in there. There are a few ways to request how input data should be distributed and `Pipeline` is the one that says "don't move anything". The string "increment" is utterly arbitrary. The `|capability|` stuff should be ignored for the moment; we'll explain in just a moment (it has to do with whether you would like the ability to send data before you receive any).

Where the really heart of the logic lies is in the closure that binds `input` and `output`. These two are handles respectively to the operator's input (from which it can read records) and the operator's output (to which it can send records).

The input handle `input` has one primary method, `next`, which may return a pair of timestamp and batch of data. Rust really likes you to demonstrate a commitment to only looking at valid data, and our `while` loop does what is called deconstruction: we acknowledge the optional structure and only execute in the case the `Option` variant is `Some`, containing data. The `next` method could also return `None`, indicating that there is no more data available at the moment. It is strongly recommended that you take the hint and drop out of your closure at that point; timely gives you the courtesy of executing whatever code you want in this closure, but if you never release control you'll break things.

The output handle `output` has one primary method, `session`, which starts up an output session at the indicated time. The resulting session can be given data in various ways: (i) element at a time with `give`, (ii) iterator at a time with `give_iterator`, and (iii) vector at a time with `give_content`. Internally it is buffering up the output and flushing automatically when the session goes out of scope, which happens above when we go around the `while` loop.

### Other shapes

The `unary` method is handy if you have one input and one output. What if you want something with two inputs? Or what about zero inputs? We've still got you covered.

There is a `binary` method which looks a lot like unary, except that it has twice as many inputs (and ways to distribute the inputs), and requires a closure accepting two inputs and one output. You still get to write arbitrary code to drive the operator around as you like.

There is also a method `operators::source` which .. has no inputs. You can't call it on a stream, for obvious reasons, but you call it on a scope instead. It looks just like the others, except you supply a closure that just takes an output as an argument and sends whatever it wants each time it gets called. This is great for reading from external sources and moving data along as you like.

### Capabilities

We skipped a discussion of the `_capability` argument, and we need to dig in to that now.

One of timely dataflow's main features is its ability to track whether an operator may or may not receive more records in the future. The way that it does this is by requiring that its operators, like the ones we have written, hold *capabilities* for sending data. A capability is an instance of the `Capability<Time>` type, which looks to the outside world like an instance of `Time`, but which `output` will demand to see before it allows you to create a session.

Remember up where we got things we called `time` and from which we created a session with `session(&time)`? That type was actually a capability.

Likewise, the `capability` argument that we basically ignored is also a capability. It is a capability for the default value of `Time`, from which one can send data at any timestamp. All operators get one of these to start out with, and until they downgrade or discard them, they retain the ability to send records at any time. The flip side of this is that the system doesn't make any progress *until* the operator downgrades or discards the capability.

The `capability` argument exists so that we can construct operators with the ability to send data before they receive any data. This is occasionally important for `unary` and `binary` operators, but it is *crucially important* for operators with no inputs. If we want to create an operator that reads from an external source and sends data, we'll need to keep hold of some capability.

Here is an example `source` implementation that produces all numbers up to some limit, each at a distinct time.

```rust,no_run
extern crate timely;

use timely::dataflow::operators::Inspect;
use timely::dataflow::operators::generic::operator::source;

fn main() {
    timely::example(|scope| {

        source(scope, "Source", |capability| {

            let mut cap = Some(capability);
            move |output| {

                let mut done = false;
                if let Some(cap) = cap.as_mut() {

                    // get some data and send it.
                    let mut time = cap.time().clone();
                    output.session(&cap)
                          .give(cap.time().inner);

                    // downgrade capability.
                    time.inner += 1;
                    *cap = cap.delayed(&time);
                    done = time.inner > 20;
                }

                if done { cap = None; }
            }
        })
        .inspect(|x| println!("number: {:?}", x));
    });
}
```

The details seem a bit tedious, but let's talk them out. The first thing we do is capture `capability` in the variable `cap`, whose type is optionally a capability. This type is important because it will allow us to eventually discard the capability.

Our next step is to define a closure, and as it is the last thing we do return said closure, that takes `output` as a parameter. The `move` keyword is part of Rust and is an important part of making sure that `cap` makes its way into the closure, rather than just evaporating from the local scope when we return.

The closure does a bit of a dance to capture the current time (not a capability, in this case), create a session with this time and send whatever the time happens to be as data, the downgrade the capability to be one timestep in the future. If it turns out that this is greater than twenty we discard the capability.

The system is smart enough to notice when you downgrade and discard capabilities, and it understand that these actions reperesent irreversible actions on your part that can now be communicated to others in the dataflow. As this closure is repeatedly executed, the timestamp of the capability will advance and the system will be able to indicate this to downstream operators.

### Stateful operators

It may seem that we have only considered stateless operators, those that are only able to read from their inputs and write to their outputs. But, you can have whatever state that you like, using the magic of Rust's closures. When we write a closure, it can capture ("close over") any state that is currently in scope, taking ownership of it. If that sounds too abstract, let's look at an example. 

Our `unary` example from way back just incremented the value and passed it along. What if we wanted to only pass values larger than any value we have seen so far? We just define a variable `max` which we check and update as we would normally. Importantly, we should define it *outside* the closure we return, so that it persists across calls, and we need to use the `move` keyword so that the closure knows it is supposed to take ownership of the variable. 

```rust,ignore
use timely::dataflow::operators::ToStream;
use timely::dataflow::operators::generic::operator::Operator;
use timely::dataflow::channels::pact::Pipeline;

timely::example(|scope| {
    (0u64..10)
        .to_stream(scope)
        .unary(Pipeline, "increment", |capability|

            let mut max = 0;    // define this here; use in the closure

            move |input, output| {
                while let Some((time, data)) = input.next() {
                    let mut session = output.session(&time);
                    for datum in data.drain(..) {
                        if datum > max {
                            session.give(datum + 1);
                            max = datum;
                        }
                    }
                }
            }
        );
});
```

This example just captures an integer, but you could just as easily define and capture ownership of a `HashMap`, or whatever complicated state you would like repeated access to.

Bear in mind that this example is probably a bit wrong, in that we update `max` without paying any attention to the times of the data that come past, and so we may report a sequence of values that doesn't seem to correspond with the sequence when sorted by time. Writing sane operators in the presence of batches of data at shuffled times requires more thought.

Specifically, for an operator to put its input back in order it needs to understand which times it might see in the future, which was the reason we were so careful about those capabilities and is the subject of the next subsection.

### Frontiered operators

Timely dataflow is constantly tracking the capabilities of operators throughout the dataflow graph, and it reports this information to operators through what are called "frontiers". Each input has an associated frontier, which is a description of the timestamps that might arrive on that input in the future.

Specifically, each input has a `frontier` method which returns a `&[Timestamp]`, indicating a list of times such that any future time must be greater or equal to some element of the list. Often this list will just have a single element, indicating the "current" time, but as we get to more complicated forms of time ("partially ordered" time, if that means anything to you yet) we may need to report multiple incomparable timestamps.

This frontier information is invaluable for operators that must be sure that their output is correct and final before they send it as output. For our `max` example, we will want to wait to apply the new maximum until we are sure that we will not see any more elements at earlier times. That isn't to say we can't do anything with data we receive "early"; in the case of the maximum, each batch at a given time can be reduced down to just its maximum value, as all received values would be applied simultaneously.

To make life easier for you, we've written a helper type called `Notificator` whose job in life is to help you keep track of times that you would like to send outputs, and to tell you when (according to your input frontiers) it is now safe to send the data. In fact, notificators do more by holding on to the *capabilities* for you, so that you can be sure that, even if you *don't* receive any more messages but just an indication that there will be none, you will still retain the ability to send your messages.

Here is a worked example where we use a binary operator that implements the behavior of `concat`, but it puts its inputs in order, buffering its inputs until their associated timestamp is complete, and then sending all data at that time. The operator defines and captures a `HashMap<Time, Vec<Data>>` named `stash` which it uses to buffer received input data that are not yet ready to send.

```rust,ignore
use std::collections::HashMap;
use timely::dataflow::operators::{Input, Inspect, FrontierNotificator};
use timely::dataflow::operators::generic::operator::Operator;
use timely::dataflow::channels::pact::Pipeline;

fn main() {
    timely::execute(timely::Configuration::Thread, |worker| {
        
        worker.dataflow(|scope| {

                let in1 = (0 .. 10).to_stream(scope);
                let in2 = (0 .. 10).to_stream(scope);

                in1.binary_frontier(&in2, Pipeline, Pipeline, "concat_buffer", |mut _builder| {
                    let mut notificator = FrontierNotificator::new();
                    let mut stash = HashMap::new();
                    move |input1, input2, output| {
                        while let Some((time, data)) = input1.next() {
                            stash.entry(time.time().clone()).or_insert(Vec::new()).push(data.take());
                            notificator.notify_at(time);
                        }
                        while let Some((time, data)) = input2.next() {
                            stash.entry(time.time().clone()).or_insert(Vec::new()).push(data.take());
                            notificator.notify_at(time);
                        }
                        for time in notificator.iter(&[input1.frontier(), input2.frontier()]) {
                            let mut session = output.session(&time);
                            if let Some(vec) = stash.remove(time.time()) {
                                for data in vec.into_iter() {
                                    session.give_content(data);
                                }
                            }
                        }
                    }
                });
        });
    });
}
```

As an exercise, this example could be improved in a few ways. How you might change it so that the data are still sent in the order they are received, but messages may be sent as soon as they are received if their time is currently in the frontier? This would avoid buffering messages that are ready to go, and would only buffer messages that are out-of-order, potentially reducing the memory footprint.

### Advanced operators

UNIMPLEMENTED!