---
title: Writing a CHIP-8 Emulator
date: 2020-06-16T20:47:56-04:00
keywords: [dev, game]
description: A nice way to enjoy old games.
draft: false
---

Being a video game lover, I have put writing a game console emulator on my to-do
list for a long time. However, I had never got into it until last year when I
saw a [blog post][wasamasa's post] on implementing an emulator in Emacs for
CHIP-8, a (virtual) machine with only 35 opcodes. That seemed to be a great
starting point and I have finally implemented it recently:

{{< figure src="/media/images/chip8-tetris-demo.png"
    caption="Playing *Tetris* with the emulator." >}}

The source code can be found [here][repo]. It turned out to be quite enjoyable
and straightforward. However, I did encounter some problems.

## CHIP-8 virtual machine

The CHIP-8 virtual machine consists of only basic components you would expect on
a working machine -- a few registers, a stack, a memory, two timers, a keypad
for user input, a screen, and a sound device. Here is the definition in my
implementation (there are some additional fields, which I will talk about
later):

```haskell
-- | Model for a CHIP-8 machine.
data Machine = Machine
  { -- | Program counter.
    _pc     :: Word16
    -- | Address register I.
  , _ir     :: Word16
    -- | Data registers.
  , _drs    :: Vector Word8
    -- | Stack pointer.
  , _sp     :: Word8
    -- | Stack.
  , _stack  :: Vector Word16
    -- | Main memory.
  , _mem    :: Vector Word8
    -- | Sound timer.
  , _st     :: Word8
    -- | Delay timer.
  , _dt     :: Word8
    -- | Keypad.
  , _keypad :: Vector Int
    -- | Frame buffer.
  , _fb     :: FrameBuffer
    -- | Random number generator.
  , _rng    :: StdGen
    -- | Whether to stay on the current instruction.
  , _paused :: Bool
  }
```

At the beginning a program is loaded to the main memory. Then at each step, the
CPU fetches an opcode (which is always two-byte long in CHIP-8) from the main
memory at the address stores in the *program counter* (PC), executes the
instruction encoded by the opcode, which could be modifying registers, writing
something to memory, or drawing something, and finally updates PC and repeats.

There are 35 instructions in total, including one that is usually unused. The
first and major work of writing this virtual machine lies in implementing those
instructions. That was actually straightforward. I just followed a reference to
implement and test each instruction and everything went smoothly -- the emulator
correctly ran a simple demo program, the maze generator, at the first run, which
was both surprising and exciting to me.

Machine instructions are only about maintaining states. Just like writing a
game, we still need to make an interface to show the screen, read user inputs,
and so on. I decided to implement a terminal user interface, and chose the
library [`brick`][brick]. `brick` employs the unidirectional model (which I
think is also called the Flux architecture, the Elm architecture, or the Redux
architecture, whatever). It maintains an event loop and processes events one by
one to update the state. Then the interface gets redrawn accordingly. The
library was easy-to-use, and the interface was done pretty quickly. Note that I
decided to "play" the sound only visually because I could not find an easy
approach for that.

Then I encountered two problems -- maintaining the keypad state and poor
emulation performance.

## Maintaining keypad state

The first problem was how to "release" a key. There are no reliable ways to
capture a key release in most text terminals, so we have to fake a key up event
after a key gets pressed. My first try was to clear the whole keypad
periodically, and as you might imagine, the result was unsatisfactory. Moreover,
because the `brick` event loop prefers terminal events to the custom events, key
presses got "eaten" quite frequently.

My second plan was to emit an event after some delay for each key press event,
and I gave up on that idea quickly. There were just too many things to consider.

Finally I decided to represent key states with non-binary values, and that is
why in the `Machine` definition the keypad is a `Vector Int`. Any value that is
positive is considered as "pressed", and when a key gets pressed its value is
set to a predefined number, like 20. Then the whole keypad gets *decayed*
periodically, that is, all positive key values are decremented.

## Emulation performance

There is no much information on how to run the emulation -- how fast the CPU
executes instructions, how often the screen gets refreshed, how they keep in
sync, etc. It seems that the standard way is to run the main loop at a screen
refresh rate and inside the loop the CPU executes multiple steps and then the
screen gets redrawn. I supposed it would be cool if we could separately control
the CPU and the screen:

```haskell
-- Set up events
chan <- newBChan 10
void . forkIO . forever $ do
  threadDelay $ 1000000 `div` 1000
  writeBChan chan Execute
void . forkIO . forever $ do
  threadDelay $ 1000000 `div` 30
  writeBChan chan Redraw
```

It basically emits an `Execute` event to ask the CPU to execute one instruction
at 1 kHz and a `Redraw` event to refresh the screen at 30 Hz. To avoid dropping
frames under a low refresh rate, I added a dirty flag to the frame buffer as
well:

```haskell
-- | Model for a frame buffer.
data FrameBuffer = FrameBuffer
  { -- | Pixels on the screen.
    _pixels :: Vector Bool
    -- | Whether frame buffer has been drawn on the screen.
  , _dirty  :: Bool
  }
```

If the frame buffer is dirty, the machine will not modify it but wait instead --
a laggy screen to me is better than a choppy screen. Then I tried to play
*Tetris*, and got a noticeable delay after pressing a key. So I started to play
around the emulation speed and screen refresh rate, and the faster the machine
ran, the slower the emulator was.

Well, that is not so accurate. Actually the performance dropped after some
point, and when the machine ran at the maximum speed, 1 MHz, the performance
seemed worse than that at 1 kHz. After some trials, I suspected the cause was
the bounded channel. Running the machine faster means the channel gets filled up
faster, and a costly event may mess everything followed.

So I increased the channel size and optimized rendering by caching the screen
area. Finally I got a smooth emulator. Speaking of caching, it seems that
`brick` does have some mechanism to prevent unnecessary redrawing. However,
maybe the diffing algorithm is not so efficient or effective. Manually caching
does improve performance apparently in my case.

Also, I noticed that waiting for a dirty frame buffer to be cleaned might not be
ideal in some cases. Because CHIP-8 uses sprites that are 8 pixels in width and
at most 15 pixels in height, programs have to combine multiple instructions to
fill a full screen. Therefore, not writing to a dirty buffer means occasionally
limiting the speed at the screen refresh rate. I did not find many resources on
how real machines handle the "dropping frame" problem so I decided to keep the
dirty flag as is.

After all those trials and errors, now I guess I understand why many choose to
aggregate multiple instructions together. In that way there will be no
constraints from the resolution and accuracy of the timer so it is possible to
achieve high emulation speed, and running a low-frequency heavy loop is easier
and more efficient than running a high-frequency light loop. My current approach
works fine for a simple machine like CHIP-8, but perhaps it will not work for
anything more complicated.

[wasamasa's post]: https://emacsninja.com/posts/smooth-video-game-emulation-in-emacs.html
[repo]: https://github.com/pengjiz/chip8hs
[brick]: https://github.com/jtdaugherty/brick
