+++
title = "TUI in R - Rendering and key presses"
date = "2022-11-25T11:00:00+00:00"
author = "Tom Ratford"
authorTwitter = "" #do not include @
cover = ""
tags = ["R", "TUI"]
keywords = ["", ""]
description = "A foray into the world of TUIs, but written in a language where everyone uses a GUI."
showFullContent = false
readingTime = false
+++

To Preface, R isn't a language designed to be run end to end from the terminal. 
The fact that you have to run a totally different command (`rscript`) to run an R file in the terminal is sort of indicative of R's preference that you just do the bloody thing interactively.

In R's defence, this makes a lot of sense. 
R's killer app is making plots, and the terminal isn't really designed to display these.
Therefore sitting in a REPL and just rerunning the same commands or scripts (but maybe slightly tweaking them) is a much more natural programming experience.
The most common way to abstract this sort of R code is to create a Shiny app, but that's dull and overdone so I'm making a TUI.

# The Basics

Forewarning, I'm using MacOS so 90% of this probably won't fly on Windows.

The first thing to learn is that R's `system` command is your new best friend.
`system` let's you submit a string back to your systems terminal to be executed as plain text.
For example, in any R session (including an RStudio session) you can do
{{< code language="R">}}
system("echo Hello, sailor!")
# Hello, sailor!
{{< /code >}}
to feel flirted with. 
R also has a convenient function to get the standard output back as a character vector.
{{< code language="R">}}
system("echo Hello, World!", intern=TRUE)
# [1] "Hello, World!"
{{< /code >}}
We will now abuse this function to do basically everything for us.

# Pretty printing

Just printing plain ol' text would make this _really_ boring.
We can spice things up with the already existing [`cli`](https://github.com/r-lib/cli) package.
This adds a dependency **BUT** removes the need to write loads of functions that print things in a nice way, which seems like a fair trade.

We can now print a simple 'UI'.

{{< code language="R" title="tui.R">}}
cli_h1("There is a car behind one of these doors")
cli_h2("Door no.1")
cli_h2("Door no.2")
cli_h2("Doos no.3")
cli_h1("")
{{< /code >}}

However, when we run this in the terminal, we need to first clear the screen.
Those among you will know of `cat('\014')` to clear the console in RStudio.
This however will not clear the whole terminal in an `Rscript` session.
To do this we have two options:
 * Send [ANSI escape codes]() to the terminal via the `cat` command with the proper escape character (e.g. `cat("\x1bc")`) to clear the terminal
 * Use the [`tput`](https://www.man7.org/linux/man-pages/man1/tput.1.html) terminal command within a `system` command

I found the `tput` commands to be easier to work with, so I will use these throughout.
We can now clear the screen using `tput clear`.

{{< code language="R" title="tui.R">}}
system("tput clear")
cli_h1("There is a car behind one of these doors")
...
{{< /code >}}

As we will be using this a lot. Let's put this into a function

{{< code language="R" title="tui.R">}}
draw_screen <- function() {
  system("tput clear")
  cli_h1("There is a car behind one of these doors")
  for (i in 1:3) {
    cli_h2(c("Door no.",i))
  }
  cli_h1("")
}
{{< /code >}}

# A main loop

R obviously doesn't have a 'main' entrypoint like python or C, but we can easily define one to help organise our code.
Inside the function will house the loop that will be used to process input, render the display, and manage any other component we expect to change. 
Anything outside of the loop can be thought of as a sort of initialiser for the TUI.

{{< code language="R" title="tui.R">}}
main <- function() {
  repeat {
    draw_screen()
  }
}

main() # Our 'entrypoint'
{{< /code >}}

If we now run our script, you will be greeted by an infinitely rendering loop which will clog up your terminal, you're welcome (also you'll need to exit by doing CTRL-C).
This is because `tput clear` simply prints a bunch of empty lines into your terminal, to push what is currently on the screen upwards.
To stop this, inside our `draw_screen` function we can change `tput clear` to `tput reset`, which instead reinitalises a whole terminal and doesn't spam a bunch of empty lines.

{{< code language="R" title="tui.R">}}
function draw_screen() {
  system("tput reset")
  ...
}

main <- function() {
  system("tput clear")
  repeat {
    draw_screen()
  }
}
{{< /code >}}

We still keep the `tput clear` before, so what was previously run in our terminal is available if you scroll up. 

# Basics of reading input

R has some built in functions to help read lines.
However these are all unsuitable.
 * `readline` doesn't work in RScript, so that's immediately out of the question.
 * `readLines` has some limitations, most notably that it requires you to press enter/return to submit a line, this is fine for when we want to receive input (like a name), but not so good for receiving input like an arrow key.

The simplest method is to actually use our trusty friend `system`.
We can use the [`read`](https://man7.org/linux/man-pages/man2/read.2.html) command to get input.
I struggled to get `read` to print the output it received, but it's okay because we can just put it to a variable and echo it back.

{{< code language="R" title="tui.R">}}
draw_screen <-  function() {...}

read_char <- function() {
  system("read -n 1 tmp; echo $tmp", intern = TRUE)
}

main <- function() {...}
{{< /code >}}

For those wondering, the `$tmp` variable will not persist after the RScript is done.

We can now get some input to our program.

{{< code language="R" title="tui.R">}}
main <- function() {
  system("tput clear")
  repeat {
    draw_screen()
    x <- read_char()
  }
}
{{< /code >}}

Now you may notice that as you press any key the character or code will flicker for _just_ a second, let's fix that next.

# Disabling echo

It's time to introduce something else, the [`stty`](https://www.man7.org/linux/man-pages/man1/stty.1.html) command.
This command focuses entirely on the settings of your current terminal.
These setting are how the terminal knows not to echo back what you're typing when you use the `passwd` command.
If you type `stty -a` into your terminal you'll get back a bunch of funky information about what 'flags' are currently enabled in your terminal.
Before we start messing with this, we want to save our current settings.
We do this with `stty -g`.

{{< code language="R" title="tui.R">}}
main <- function() {
  current_settings <- system("stty -g", intern = TRUE)
  system("tput clear")
  ...
}
{{< /code >}}

Now we want to be a bit more defensive.
If we can't get the output from `stty -g` we abort the program so we don't start randomly changing flags and breaking everyone's terminal.
When a command fails whilst using `system` we get returned back an object with a `status` attribute.
The value of `status` doesn't really matter - we can assume that any non-null value of `status` is a bad thing, and we should probably stop.

{{< code language="R" title="tui.R">}}
main <- function() {
  stty_orig <- system("stty -g", intern = TRUE)
  if (!is.null(attr(stty_orig, "status"))) {
    cli_abort("STTY COULD NOT SAVE")
  }
  system("tput clear")
  ...
}
{{< /code >}}

Now, we want to change our loop slightly.
We want to ensure that if the loop fails at **any** point. 
We recover the original terminal settings before terminating.
R has `tryCatch`, which has the `finally=` parameter.
The expression given is executed once the code in `tryCatch` is 'complete' (that includes whether the code failed or not).
The `cli` package already depends on the `glue` package so we can use the `glue::glue` function instead of a `paste` to write the string without including another dependency.

{{< code language="R" title="tui.R">}}
main <- function() {
  stty_orig <- system("stty -g", intern = TRUE)
  if (!is.null(attr(stty_orig, "status"))) {
    cli_abort("STTY COULD NOT SAVE")
  }
  system("tput clear")
  tryCatch({
    repeat {
      draw_screen()
      x <- read_char()
    }
  },
  finally = system(glue::glue("stty {stty_orig}"))
}
{{< /code >}}

Now, we can *finally* disable the 'echo'-ing of character.
We do so with `stty -echo -icanon`.
Echo disables the echo-ing of characters, icanon changes how the input is processed.
See [here](https://man7.org/linux/man-pages/man3/termios.3.html) for more info (search for 'Canonical and noncanonical mode').
Let's also finally get to the point where R is going to print back our `read_char()`

{{< code language="R" title="tui.R">}}
main <- function() {
  stty_orig <- system("stty -g", intern = TRUE)
  if (!is.null(attr(stty_orig, "status"))) {
    cli_abort("STTY COULD NOT SAVE")
  }
  system("tput clear")
  system("stty -echo -icanon")
  x <- ""
  tryCatch({
    repeat {
      draw_screen()
      print(x)
      x <- read_char()
    }
  },
  finally = system(glue::glue("stty {stty_orig}"))
}
{{< /code >}}

# What are key presses?

_If you already know about ANSI escape codes, then skip ahead to the next section_

With our above code, we can now see the result of key presses and do a bit of exploring.
When we press a character, number or punctuation key, we are returned back the same punctuation or character.
A key like backspace or escape returns a unique code.
Now press an arrow key.
This actually sends multiple different characters.
We can have a closer look at how these key presses work by slightly modifying our code.

{{< code language="R" title="tui.R">}}
main <- function() {
  stty_orig <- system("stty -g", intern = TRUE)
  if (!is.null(attr(stty_orig, "status"))) {
    cli_abort("STTY COULD NOT SAVE")
  }
  system("tput clear")
  system("stty -echo -icanon")
  x <- ""
  tryCatch({
    repeat {
      draw_screen()
      print(x)
      x <- c(read_char(), read_char(), read_char)
    }
  },
  finally = system(glue::glue("stty {stty_orig}"))
}
{{< /code >}}

Now when we run and press the up arrow key, you will most likely see

```
[1] "\033" "["    "A"   
```

This is actually a case of the aforementioned [ANSI escape code](https://en.wikipedia.org/wiki/ANSI_escape_code).
Basically, when they made [ASCII](https://en.wikipedia.org/wiki/ASCII#Character_groups), they made it so that the first 32 characters were reserved as 'control characters'. 
These are your newlines, TAB keys, etc...
One of these is specified as the "Escape sequence".
This is number 27, which is `033` in [Octal](https://en.wikipedia.org/wiki/Octal) and `1b` in [Hexadecimal](https://en.wikipedia.org/wiki/Hexadecimal).
This is why you see `"\033"`, you might also see this escape sequence referenced online as `\x1b` or `0x1b` for the same reason.
It all means escape sequence.
The `[` means that we can expect parameters after this, in this case, we are just giving the singular 'A'.
Within the [ANSI Control Sequences](https://en.wikipedia.org/wiki/ANSI_escape_code#CSI_(Control_Sequence_Introducer)_sequences) we can see that this corresponds to the 'Cursor up' command.

> Hold on a minute! I've used these before!

You're right! 
If you've ever typed `cat("\014")` you've used ASCII control characters already!

If you look at the [control code chart](https://en.wikipedia.org/wiki/ASCII#Control_code_chart) then you can see that `014` in octal is `12` in decimal is `0C` in hex.
This corresponds to the 'form feed' control, which is also mapped to CTRL-L.
Therefore you can type `cat("\x0c")` to achieve the same result.

# How to handle arrow keys

In order to correctly handle keypresses, we need to first check if our keypress is '\033'.
If so then we need to get the two more characters back: firstly the '[' and then our actual code value.

{{< code language="R" title="tui.R" >}}
read_char <- function() {...}

read_keypress <- function() {
  char1 <- read_char()
  if (char1 == "\033") {
    square_bracket <- read_char()
    code <- read_char()
    switch(code,
           A="UP_KEY",
           B="DOWN_KEY",
           D="LEFT_KEY",
           C="RIGHT_KEY",
           default = code)
  } else {
    char1
  }
}

main <- function() {...}
{{< /code >}}

Now, we can program moving up and down in the terminal.
To do this we will have to refactor both `main()` and `draw_screen()`.
Our `main()` function will now have a persistent `cursor` variable, which it will pass to our `draw_sreen()`.

{{< code language="R" title="tui.R" >}}
main <- function() {
  ...
  tryCatch({
    cursor <- 0
    repeat {
      draw_screen(cursor)
      x <- read_keypress()
      if (x == "UP_KEY") {
        cursor <- (cursor - 1) %% 3
      } else if (x == "DOWN_KEY") {
        cursor <- (cursor + 1) %% 3
      }
    }
  },
  ...
}
{{< /code >}}

{{< code language="R" title="tui.R" >}}
draw_screen <- function(cursor) {
  system("tput reset")
  cli_h1("There is a car behind one of these doors")
  for (i in 1:3) {
    if (i-1 == cursor)
      cli_h2(bg_green(c("Door no.", i)))
    else
      cli_h2(c("Door no.",i))
  }
  cli_h1("")
}

read_char <- function() {...}
{{< /code >}}

Now when we run our program we are able to move up and down between options.
However we still can't quit.
Thankfully, we've already done 90% of the work!
when we press <kbd>CTRL+Q</kbd>, a similar escape code of `"\033" "[" "\021"` is sent to the terminal.
Hence we can simply account for it in the `if ... else` in `main()`

{{< code language="R" title="tui.R" >}}
main <- function() {
  ...
  tryCatch({
    cursor <- 0
    repeat {
      draw_screen(cursor)
      x <- read_keypress()
      if (x == "UP_KEY") {
        cursor <- (cursor - 1) %% 3
      } else if (x == "DOWN_KEY") {
        cursor <- (cursor + 1) %% 3
      } else if (x == "\021") {
        break
      }
    }
  },
  ...
}
{{< /code >}}

which simply quits our loop and ends the `tryCatch`.

{{< image src="/tui-in-r-01/final.gif" alt="Gif of keypresses in TUI" position="center" style="border-radius: 8px">}}

