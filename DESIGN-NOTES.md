# Design Notes for a2vimode

This document covers the basic "How It Works" of a2vimode, and also a number of pitfalls encountered, on different hardware and DOS platforms, and the workarounds to them.

**Note:** throughout this document, references are made to ProDOS wrapping `KSW`, or calling the `GetLn` prompt, or modifying the AppleSoft prompt. In actuality, the core ProDOS software does not use the AppleSoft prompt, nor does it provide `KSW` or `CSW` wrappers—this functionality is provided by `BASIC.SYSTEM`, which is an optional system file provided as part of the ProDOS software package. Not all instances of ProDOS will have `BASIC.SYSTEM` installed, and so those instances will not in fact behave as described in this document. However, since ProDOS in those instances will also not be using a prompt to query the user for ProDOS commands, a ProDOS instance that does not include `BASIC.SYSTEM` will also not be using **a2vimode** (though, it could be used to run a program that does use it). Therefore, wherever you see ProDOS mentioned in this document, what we're really referring to is `BASIC.SYSTEM` running on ProDOS (probably, ProDOS 8).

## Basic Theory

### Checking For GetLn

When a2vimode is started, a routine is installed to the Standard Input hook, `KSW` (or more accurately, the DOS or ProDOS "inner" equivalent to `KSW`). When this routine is called, it looks at call-return values on the stack (the addresses to which the `PC` register is set, on successive 6502 `RTS` operations), to determine if it's being called from the firmware's line-prompt facility, known as `GetLn` or `GETLN`. Specifically, it looks for (most-recent caller):

 - An optional return to a DOS or ProDOS `KSW` wrapper routine (this return can be any word less than `$C000`)
 - A return to the `RdChar` routine (which is called by `GetLn` to check for and handle `Esc` keyrpesses, for transposing the cursor to somewhere else on the screen; the expected return value is normally `$FD37`, but on the Apple //c specifically it will instead be a sequence of 2 or 3 words in the `$Cxxx` range, so those are accepted instead.
 - Finally, it must find the expected return to the `GetLn` routine (always `$FD77`)

(Note: if you want to verify for yourself where those return addresses arrive at, please remember that `RTS` will *add one to the value on the stack* before setting the program counter, so although the routine checks for a return address of `$FD77`, the real location of the actual instruction that would be returned to, is `$FD78` (and the `JSR` instruction that (eventually) got to us, is 3 bytes earlier at `$FD75` (2 bytes less than the address on the stack). None of these, of course, are the *starts* of the routines, they are simply the point in the routine where a `JSR` operation called us (or called a routine that called us, etc).)There is no practical way to distinguish this situation from a real call from `GetLn`, but thankfully it is a very unlikely scenario.

These call-stack values are correct for every known model of Apple II running the official firmware ROMs, plus also the ROM 4X on the Apple //c. It is unknown whether these values will work on unofficial Apple II *clones*, and may not work with customized ROMs. In such cases, it would be necessary to modify **a2vimode**'s call-check routine in order for it to work on that platform.

### If We Are In GetLn

If the check succeeds, we know we are currently "in" the firmware prompt. We then *edit* the stack, replacing the call-return for `GetLn` (`$FD77`) with an address that instead "returns" into our special prompt-replacement routine—the routine that handles all the special vi-mode bindings and commands. We then immediately return with the space character (`$A0`) in the accumulator, just to let us wind our way back up the stack, through whatever DOS wrapper, and `RdChar`, until it "returns" into our prompt replacement (which just ignores the returned char, and prompts for a new one).

An early development version of this software didn't immediately return a space, but instead served as a prompt for the first character that would be received by our prompt-replacement routine. We would edit the stack so as to remove `RdChar` from the return-sequence (so that the DOS or ProDOS KSW hook would skip the return to `RdChar`, and return instead directly into our prompt replacement). However, DOS tends to be sensitive to stack size changes, since it does manipulations of its own, and did not respond well to our removing a call frame from the stack. So we leave the stack the same size, and leave `RdChar` in place, and return a dummy space character so there's no possibility of `RdChar` seeing an `Esc` from user input, and handling it specially.)

Note that, since the stack can hold arbitrary values that were pushed onto the stack, and not merely call-returns, it is entirely possible that a program could push just the right (or wrong) values onto the stack, prior to calling our Standard-Input hook routine, and trick it into believing it was called from `GetLn` via `RdChar`, when in fact it was not. This could cause a crash when returning from our replacement prompt routine, but fortunately this scenario is unlikely enough that it shouldn't happen under typical circumstances. If it should happen, the solution would be to avoid using **a2vimode** together with whatever program happens to like pushing those values on the stack just prior to invoking `KSW`. 

### Additional AppleSoft Check

After a successful check for `GetLn`, when we already know we're going to replace `GetLn` with our replacement prompt routine, we do an additional check to see if that `GetLn` was in turn called from AppleSoft's main ("direct mode") user prompt. Specifically, it checks for:

 - A call-return to `InLine` (the AppleSoft routine that calls `GetLn`, and then converts the input buffer to "real" ASCII codes, terminated by a `NUL` (`#$00`) instead of a carriage return (`#$8D`) - stack value `$D532`). This routine is in turn called from:
 - A call-return to `Restart` (`D443`)

If this secondary check succeeds, a flag is set so that the replacement prompt knows that it's being used in AppleSoft's direct mode - this will enable features like auto-increment line numbers, and summoning lines of BASIC to the prompt (which are not permitted in other prompt situations).

### If We Are *Not* In GetLn

If the check fails, then we were not called from `GetLn`, and so we just jump to the firmware's "grab a keypress" routine, to do what a Standard-Input routine is normally expected to do.

## Apple //c Quirks

As mentioned, the call signature through `RdChar` looks different on an Apple //c from on any other Apple II systems. Apple //c `RdChar` is written differently, and `JMP`s to enhanced-firmware routines in the `$Cxxx` range. All of the official Appl ROMs have the same two-call signature in that range, after `GetLn`'s call signature; the popular fan-made //c ROM "4X" has a third call in that range. The //c+ ROMs 5 and 5X were not tested.

The initial release of **a2vimode** at HackFest 2022 (part of KansasFest) did not function on an Apple //c, because it had been developed and tested via emulation on an Apple \]\[+, \]\[e, and "enhanced" //e... but not on //c until immediately prior to release. It was fixed a day or two later.

## Unenhanced Apple //e Quirks

On most Apple \]\[ machines, the built-in Standard Input routine at the `KSW` hook does not process the `Esc` key specially (that is, you should not be able to move the cursor around the screen for a simple "read-the-keyboard" call).

However, on the unenhanced Apple \]\[e, and if the 80-column firmware is active (whether or not the screen is actually in 80-column mode), the `Esc` key *is* handled specially, and the user can use it to move the cursor around the screen even though it was supposed to be an ordinary keyboard check. This is *particularly* undesirable for **a2vimode**, because it relies on the cursor position remaining stable within the prompt, so that it can update the prompt properly when new input is typed, or characters are deleted. As long as the 80-column firmware's `KSW` routine (`$C305`) is used, there is no way to prevent `Esc` from being specially interpreted, before the prompt has the chance to intercept it.

The workaround so far has been to provide the `Tab` key as an alternate "normal mode" key instead of `Esc`, and recommend its use instead. Of course, an original Apple \]\[ or \]\[+ does not have a `Tab` key, so on such systems one should still use `Esc`.

A future version of **a2vimode** will probably check if the current machine is an unenhanced Apple //e with the 80-column firmware active, and if so, use a custom replacement "keyboard check" routine instead of the standard one, and so avoid the `Esc` problem.

## ProDOS/BASIC.SYSTEM Quirks

In initial tests of the first (HackFest) version of **a2vimode** on ProDOS, running a ProDOS command such as `CAT` would execute the command, but then instead of the prompt you would just see the flashing cursor indicating that it's awaiting input. If you typed characters, they would appear, along with a few unexplained garbage characters, and if you typed `RETURN` you'd get a SYNTAX ERROR and then you'd be at a normal prompt. I assumed that there was probably a really nasty interaction with ProDOS's command processing that leads to some corrupted states, perhaps with it running an input hook that behaved differently than expected, or perhaps my `KSW` routine was incorrectly detecging it's "at the prompt" when it's not. So I decided to mark ProDOS as "not supported" in that first release, pending further investigation into what horrible interaction must be taking place.

When I did take a look at it a few days later, the explanation turned out to be pretty simple, though certainly unexpected. When the ProDOS (BASIC.SYSTEM) `KSW` hook detects that the "inner" `KSW` got a carriage return, it *immediately* checks the current state of the prompt's input buffer, and executes it if it looks like a ProDOS command. If it doesn't look like a ProDOS command, it's assumed to be BASIC, and it lets the carriage return pass through to the prompt, so that the prompt will then return and BASIC will process it. This is why non-ProDOS commands were working fine.

But if it *is* a ProDOS command, then the command is run, the input buffer flushed/emptied, the X register is set to zero, and rather than returning a carriage return, ProDOS's `KSW` wrapper routines returns a backspace instead.

Why does it do this? Well, ProDOS knows that the firmware's `GetLn` prompt (which, after all, it knows beyond all doubt is the prompt routine that it called) uses the input buffer at `$200` to store the current input, and uses the X register to track the current "cursor" position at the input buffer—which, in `GetLn`, is always also the "end" of the buffer, since the firmware prompt doesn't support inserting characters in the *middle* of the input buffer. Adding the capability to do so is, after all, much of the point of writing **a2vimode** in the first place.

ProDOS further knows that, if `GetLn` receives a backspace while the input buffer is empty, it will print a carriage return, and re-issue the prompt character (`]` when it's the main prompt).

So, to summarize: if you type a BASIC command (or anything that doesn't look like a ProDOS command), then when the user types a `RETURN`, the prompt routine receives it, inserts it into the input buffer, and returns to whoever called it. *But*, if you type a ProDOS command and type a `RETURN`, then instead of letting the prompt return, ProDOS *steals* the command from the current buffer, then tries to lie to the prompt that it actually has no input yet from the user, and emits a backspace because that will make the prompt *look the same* as if it had just started afresh. *If* the prompt is `GetLn`—which of course it is, ProDOS knows it is because that's the routine it called, directly.

But of course we had sneakily replaced `GetLn` with our own prompt, which doesn't quite behave in *precisely* the same way as `GetLn`. In particular, while, just like `GetLn`, our prompt uses the X register to track the current "cursor" position within the input buffer, we do *not* use it as an indicator of the input line's current *length*, because for our prompt these are two separate concepts. So we store the length elsewhere, in memory. As a result, when BASIC.SYSTEM tries to trick us into believing that both the current input-cursor position and the length of the input are zero, it only succeeds in the former. So if the command typed had been `CAT`, the input cursor would be at the start of input, but the input would still contain 3 characters in it after ProDOS was done processing it. And ProDOS's trick of sending us a backspace to "reissue" the prompt doesn't work on us, because unlike `GetLn` we don't do that.

So, in the example just given, the line length will still be 3, just as it was before the user typed the carriage return. *But*, the input buffer will no longer contain the word `CAT`, because ProDOS (or BASIC.SYSTEM) makes liberal use of the input buffer while it's executing some commands. When you run `CAT`, for example, the final line may contain something like `BLOCKS FREE: 190   BLOCKS USED: 90` or some such. Before printing that line to the screen, it is first constructed inside the input buffer at `$200`, beginning with a double-quote (`"`). So the contents of the (discarded/flushed) input buffer will be `"BLOCKS FREE: 190   BLOCKS USED: 90`; but **a2vimode** will "know" that the input length is 3, so the input buffer appears to contain just `"BL`. 

Now, when control returns back to the prompt again, it believed that whatever's in the input buffer is also already displayed on the screen. Contrary to ProDOS's expectations, it did not respond to a BS character by emitting a carriage return followed by the prompt character, which would make it "seem" that we're at a fresh prompt. So instead, the on-screen cursor will appear wherever ProDOS happened to leave it, and as soon as the user starts typing something, when the line's display is updated it will suddenly contain the "garbage" characters `"BL` along with whatever the user typed (which was originally part of the `"BLOCKS FREE` line ProDOS had constructed for its own purposes).

A closely-related issue with ProDOS was that if the user left the cursor anywhere but at the end of the line before hitting return, their command would be truncated and ProDOS wouldn't recognize it. For instance, if the user typed `CCAT`, then entered normal mode (via `TAB` or `ESC`), went to the beginning of the line with `0`, deleted the first character with `X`, and then typed `RETURN`, the X register would be at zero, and ProDOS would think the line was empty. So even though the line contains `CAT`, ProDOS doesn't see it, and the line gets passed down to AppleSoft, which doesn't *recognize* `CAT`, so it emits a syntax error for a perfectly valid ProDOS command.

We could work around that issue by ensuring we set the X-register (which tracks the input buffer cursor position) to the end of the input buffer, temporarily, before each prompt for a user keypress, so that if the user were to press a carriage return and ProDOS steal the command from us, at least it would see the full command. A remaining issue with that solution, though, is that if the on-screen cursor is not at the end of the input line, ProDOS will erase whatever's in the line after the cursor position when it steals the command, via a call to `ClrEol`. The actual contents of the input line remain intact, but the on-screen display of the line gets truncated. We can set the X register to the end of the input buffer, but we still have to leave the on-screen cursor position (as described by `CH` and `CV`: `$24` and `$25` on the zero page) wherever it really is within the input line, or else the flashing cursor (indicating we're ready for a user to type something) will be at the wrong spot on the screen, and the user won't be able to tell where typed characters would appear.

## DOS Quirks

Two quirks were discovered in interoperation with DOS, which have some similarities to the ProDOS quirks in that they involve failure to recognize DOS commands when the cursor's not at the end of the line, and "stealing" the input away from the prompt—but in both cases, the actual mechanisms behind them are pretty different.

Like ProDOS, DOS also "steals" the input away from us if it detects a DOS command at the input buffer. *Unlike* ProDOS, DOS doesn't do it the moment the user presses type carriage return: it waits until the prompt echos that carriage return back *out*, to the screen.

Once that happens—once we've called DOS's `CSW` wrapper with a carriage return, right after the `KSW` routine had returned one, DOS never returns control to the prompt again. It *deletes* it right off the stack, resetting the stack pointer to some saved position further up. When it's ready to prompt, it'll let AppleSoft re-issue the prompt call once again, but meanwhile if there was any cleanup we wanted to do before returning: too bad.

It turned out that, somewhere down the line in development (not in initial versions, but later), we did indeed have some cleanup to do, and so this DOS behavior caused us in turn to misbehave. Once we tracked down the problem, the answer was simple: just move any important cleanup to *before* we emit a final carriage return to the screen.

In ProDOS, ending commands with the cursor somewhere besides the end of the line weren't recognized because they were perceived as truncated. In DOS, ending commands with the cursor somewhere other than the end of the line weren't recognized, because in such a case we do some screen updates before emitting the final carriage return. It turns out (as mentioned above, but unbeknownst to us at the time), that DOS *only* recognizes commands if the prompt issues a carriage return *immediately* after receiving a keypress from the user. Since we do other screen updates, we're sending other characters out to the `CSW` wrapper, and *then* emitting the carriage return… belatedly.

The "screen updates" in question are just to move the cursor to the end of the input line before returning. In general, **a2vimode**'s design for moving the character is to get there by typing (sending characters to `COUT`/`CSW`). If we're moving the cursor forward, we retype characters from the input buffer until we're at the lcoation we want to be; if backward, we just type backspace characters until we're there. It's much less efficient than simply jumping the on-screen cursor to the right position by setting `CH` and `CV`, but it saves a lot of headaches and bug opportunities trying to calculate where that position should be. And it works just as well in 80-column mode as in 40. But in this case, it's that "retyping" that is causing DOS to decline to check whether we've just sent it a DOS command. One solution, which was the implementation we went with for a bit, is to save away the position of the end-of-the-line whenever we write it. Pretty much every change affords this opportunity: if we're not inserting or deleting characters directly at the end of the line, then we're re-adjusting the display of characters at the end of the line (and while we're there, can note where it is). We can then jump to the end of the line when we get a carriage return, instead of moving forward by printing characters, disrupting DOS.

The jump-to-the-end mechanic appeared to be working, but was removed in favor of printed characters again, when we switched to "sidelining the carriage return" as our universal solution to handle both DOS and ProDOS.

## Workarounds to DOS and ProDOS Quirks

The way we deal with these quirks from DOS and ProDOS/BASIC.SYSTEM, most of which stem from a combination of how they process the `RETURN` key, and our need to ensure the cursor is advanced to the end of the line (in both the input buffer and on the screen) before exiting, is to sideline any carriage return we receive at the prompt, hiding it from DOS and ProDOS until we have a chance to do our own cleanup and processing of it. We then call one last `KSW`, and this time we allow the (delayed) `RETURN` to be returned from our inner input hook, passing it back through the DOS or BASIC.SYSTEM `KSW` wrapper so they can process their commands as needed, and clean up any aftermath they leave.

When our prompt is ready to receive a user keypress, it sets a flag to indicate that under no circumstances should our `KSW`-installed input routine return a carriage return. It should instead set an internal flag to indicate that a carriage return was in fact received, but report via the accumulator register that it received a space instead. When the prompt sees this flag, it ignores the returned space char and instead handles the fact that a carriage return arrives. It does any necessary cleanup, and moves both the on-screen and input cursors to the end of the user's input. Then it again sets a flag and calls `KSW` (through DOS's or ProDOS's wrapper), and this time our `KSW`-installed routine will not prompt for a real keypress at all but immediately report that the "user" typed a carriage return (the one the user did in fact type, but delayed until we were ready for a DOS to process it).

If DOS sees the carriage return, and then subsequently sees us echo it back out to the screen, and if further it sees that the input buffer contains a DOS command and not a BASIC command, then it will blow our prompt away by unwinding the stack a ways, so we just need to make sure all our cleanup has been done first.

If ProDOS sees the carriage return, and further sees that the input buffer contains a ProDOS command, it will immediately consume it, reset the X register, and return a backspace character to try and trick the prompt into restarting. Since we know beyond a shadow of a doubt that our hook just told ProDOS the user typed a carriage return, then if ProDOS's `KSW` reports to us that the user typed a backspace, we know to respond by restarting the prompt.

And if the command buffer is consumed neither by DOS nor by ProDOS, then we'll reach our actual `RTS` operation, and return control to our controller (AppleSoft, if it's a direct-mode prompt that could in theory have involved either DOS), which will convert everything into `NUL`-terminated ASCII, and take it from there.