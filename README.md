
<!-- README.md is generated from README.Rmd. Please edit that file -->

## State of 03.04.2022

<!-- badges: start -->
<!-- badges: end -->

### Installation

You can install the development version of shinybreakpoint from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("gsmolinski/shinybreakpoint")
```

### Overview

`shinybreakpoint` is designed to display in the Shiny app (i.e. when the
app runs) parts of the source code used in the app and to allow to set
breakpoint in *some* places of the displayed source code. Thus it should
be clear from the beginning that this package has two-stage limitations:

-   breakpoint can’t be set on the lines which are *not* displayed
-   breakpoint won’t be set on *all* lines which are displayed

Setting breakpoint is a debugging technique - when the point (line of
code) is reached, code execution is halted, debug mode is enabled and
one can check the values of objects or perform any other operations in
the temporary environment (i.e. all changes exist only in the debug
mode). It is one of the method to find out why unexpected behavior
occurred.

`shinybreakpoint` do not provide *new* technique for Shiny apps - it is
already possible to set breakpoint using e.g. RStudio IDE, but the
current solution has its own limitation. Breakpoint can’t be set when
the code is split into multiple files (which is often the case when app
is built of modules). Although `shinybreakpoint` gives the solution for
that, it makes it in the radical way - the code has to be split into (at
least one), separated from `server` part, function.

### Usage

#### Setting Breakpoint

We will start from the snippet - minimal skeleton needed to successfully
run the Shiny app with the `shinybreakpoint` functionality:

    library(shiny)

    ui <- fluidPage(
      
    )

    appServer <- function(input, output, session) {
      # here will be the code which will be run in the
      # 'server', not in the 'server' itself
    }

    server <- function(input, output, session) {
      appServer(input, output, session)
      shinybreakpoint::shinybreakpointServer()
    }

    shinyApp(ui, server)

it is very similar to the regular Shiny snippet, however with the one
significant difference - the server part is duplicated.

This duplication is necessary, because when the breakpoint is set using
`shinybreakpoint`, the session is refreshed to enable changes in the
code (these changes are the code added to the chosen line, i.a.
`browser()`). Refreshment works only for objects nested in the functions
which are then call in the `server` part. In other words, breakpoint
won’t work for objects used directly in the `server`, but these objects
will be visible in the source code. As an example that works, we can
consider the code below.

    library(shiny)
    library(magrittr)

    ui <- fluidPage(
      numericInput("num1", "Num", 1),
      numericInput("num2", "Num", 2),
      actionButton("go", "Go")
    )

    appServer <- function(input, output, session) {
      observe({
        input$num1
      })
      
      observe({
        input$num2
      }) %>% 
        bindEvent(input$go)
    }

    server <- function(input, output, session) {
      appServer(input, output, session)
      shinybreakpoint::shinybreakpointServer()
    }

    shinyApp(ui, server)

The key concept is that all code on which we possibly would like to set
breakpoint is separated from the `server`
(`shinybreakpoint::shinybreakpointServer()` does not necessary need to
be use in the `server`, it could be used in the `appServer` as well).
When the app is built of modules, this separation is achieved *by
definition*, but when not, then `shinybreakpoint` needs this additional
step.

To run the example above, one should save the file with the code, run
the app and press `F1` (as this is key used by default, check out the
other parameters by running `?shinybreakpoint::shinybreakpointServer` in
the console) - the modal dialog will pop up. In our example there are
two numeric inputs and one button in the `UI` as well as two `observe`s
(one is *eager* - will run immediately and one will run only after the
button is pushed) in the `server`. `observe`s will be visible in the
modal dialog and breakpoint can be set on two lines:

-   `input$num1` and
-   `input$num2`

other possibilities won’t work. Generally we can say that breakpoint
won’t be set on the *edges* of the visible code blocks.

Immediately after the breakpoint is set (i.e. when the `Activate` button
in the modal dialog is pushed), session is refreshed. That means that if
the breakpoint in our example was set on the `input$num1` line, debug
mode will open up immediately, but if it was set on the `input$num2`,
then `Go` button needs to be pushed. That fact strictly depends on the
reactive programming - debug mode opens up when the code block (where
breakpoint is set) starts and the chosen line is achieved.

Before we move into debug mode, something more has to be said about
*visible code blocks*. `shinybreakpoint` tries to find and display only
objects which belong to *reactive context* (`observe`s, `reactive`s and
`render*`s), however it is not guaranteed that *all* of these objects
will be find and that *only* these objects will be find. If so, one
should remember that `shinybreakpoint` was designed to work with objects
which belong to reactive context and thus it can lead to errors if using
on other objects.

#### Debug Mode

As already said, debug mode is an (temporary) environment when is
possible to run arbitrary code and thus is very helpful to check if
e.g. variables have the expected values. Although this vignette is not
about debugging (many helpful documents are possible to find easy), some
things in the context of the `shinybreakpoint` package should be
highlighted.

Setting breakpoint is nothing more than inserting `browser()` in some
place of code, but `shinybreakpoint` inserts more than this. All of the
added code (in total four lines of code) is harmless, but will be
visible in the debug mode. It can be run (if one would like to step down
line-by-line) or ignore.

Breakpoint is constructed to be a one-time breakpoint, which means that
after pressing `c` or `f` in the debug mode, Shiny app won’t have active
breakpoint anymore (to set it again, modal dialog delivered by
`shinybreakpoint` must be use again). There is, however, case when it
could look like the breakpoint persist - during the debug mode, the app
still runs and the objects in `UI` can be use. If we take as an example
our example above, we could notice the following behavior: set the
breakpoint on `input$num1` line, activate it and during the debug mode,
change the `input$num1` in the app - then when we press `c` in the debug
mode (to continue execution of code to the end of block and then exit),
we will notice that we end up in the debug mode again, but now with the
different value of `input$num1`. If we decide to change again
`input$num1` in the app, this situation will repeat, but if not, the
debug mode will close successfully after pressing `c`. To avoid this
problem, we can generally say that *it is not recommended to use app
during the debug mode*.

And the last thing - if something is displayed on the screen in the app,
it can be impossible to see its value just be typing the variable name
in the debug mode, and it can be necessary to use `sink()` before (in
the debug mode). In our example that would be the case if we would have
`renderPrint` where we would like to display `input$num1`.

### Additional Comments

Not previously mentioned and possibly important remarks are:

-   when the `browser()` and other code is added, the `srcref` attribute
    of function is lost. However, it shouldn’t be a problem since this
    attribute is used to get information about the source of object and
    therefore is useful only for debugging
-   session is reload twice - first time after the breakpoint is set and
    the second one when debug mode is closed using `c` or `f`. This can
    be inconvenient if some heavy computations are performed when the
    session starts
-   if one uses RStudio IDE, `shinybreakpoint` recognizes the file
    opened in the source editor and if this file is used in Shiny app
    and contains reactive blocks of code, it will open it as default
    file in modal dialog. This works only in a moment when the key
    specified in `keyEvent` parameter is pressed
