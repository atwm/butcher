# butcher

#### Chops a command or program invocation into digestable pieces.

Similar to the `optparse-applicative` package, but less features,
more flexibility and more evil.

The main differences are:

* Provides a pure interface by default

* Exposes an evil monadic interface, which allows for much nicer binding of
  command part results to some variable name.

    In `optparse-applicative` you easily lose track of what field you are
    modifying after the 5th `<*>` (admittedly, i think -XRecordWildCards
    improves on that issue already.)

    Evil, because you are not allowed to use the monad's full power in this
    case, i.e. there is a constraint that is not statically enforced.
    See below.

* The monadic interface allows much clearer definitions of commandparses
  with (nested) subcommands. No pesky sum-types are necessary.

## Examples

The minimal example is

~~~~.hs
main = mainFromCmdParser $ addCmdImpl $ putStrLn "Hello, World!"
~~~~

But lets look at a more feature-complete example:

~~~~.hs
main = mainFromCmdParserWithHelpDesc $ \helpDesc -> do

  addCmdSynopsis "a simple butcher example program"
  addCmdHelpStr "a very long help document"

  addCmd "version" $ do
    porcelain <- addSimpleBoolFlag "" ["porcelain"]
      (flagHelpStr "print nothing but the numeric version")
    addCmdHelpStr "prints the version of this program"
    addCmdImpl $ putStrLn $ if porcelain
      then "0.0.0.999"
      else "example, version 0.0.0.999"

  addCmd "help" $ addCmdImpl $ print $ ppHelpShallow helpDesc

  short <- addSimpleBoolFlag "" ["short"]
    (flagHelpStr "make the greeting short")
  name <- addStringParam "NAME"
    (paramHelpStr "your name, so you can be greeted properly")

  addCmdImpl $ do
    if short
      then putStrLn $ "hi, " ++ name ++ "!"
      else putStrLn $ "hello, " ++ name ++ ", welcome from butcher!"
~~~~

Further:

- [Full description of the above example, including sample behaviour](example1.md)
- [Example of a pure usage of a CmdParser](example2.md)
- [Example of using a CmdParser on interactive input](example3.md)
- The [brittany](https://github.com/lspitzner/brittany) formatting tool is a
  program that uses butcher for implementing its commandline interface. See
  its [main module source](https://github.com/lspitzner/brittany/blob/master/src-brittany/Main.hs)
  or [the config flag parser](https://github.com/lspitzner/brittany/blob/master/src/Language/Haskell/Brittany/Config.hs).

## The evil monadic interface

As long as you only use Applicative or (Kleisli) Arrow, you can use the
interface freely. When you use Monad, there is one rule: Whenever you read
any command-parts like in

~~~~
f <- addFlag ...
p <- addParam ...
~~~~

you are only allowed to use bindings bound thusly in any command's
implemenation, i.e. inside the parameter to `addCmdImpl`. You are _not_
allowed to force/inspect/patternmatch on them before that. _good_ usage is:

~~~~
addCmdImpl $ do
  print x
  print y
~~~~

while _bad_ would be

~~~~
f <- addFlag
when f $ do
  p <- addParam
  -- evil: the existence of the param `p`
  -- depends on parse result for the flag `f`.
~~~~

That means that checking if a combination of flags is allowed must be done
after parsing. (But different commands and their subcommands (can) have
separate sets of flags.)

## (abstract) Package intentions

Consider a commandline invocation like "ghc -O -i src -Main.hs -o Main". This
package provides a way for the programmer to simultaneously define the
semantics of your program based on its arguments and retrieve documentation
for the user. More specifically, i had three goals in mind:

1. Straight-forward description of (sub)command and flag-specific behaviour
2. Extract understandable usage/help commandline documents/texts from that
   descriptions, think of `ghc --help` or `stack init --help`.
3. Extract necessary information to compute commandline completion results
   from any partial input. (This is not implemented to any serious degree.)

## Semantics

(Sorry, this description is severely lacking, I know.)

Basic elements of a command are flags, parameters and subcommands. These can
be composed in certain ways, i.e. flags can have a (or possibly multiple?)
parameters; parameters can be grouped into sequences, and commands can have
subcommands.

Commands are essentially `String -> Either ParseError out` where `out` can
be chosen by the user. It could for example be `IO ()`.

To allow more flexible composition, the parts of a command have the "classic"
parser's type: `String -> Maybe (p, String)` where `p` depends on the part.
Parse a prefix of the input and return something and the remaining input, or
fail with `Nothing`.

A command-parser contains a sequence of parts and then a number of subcommands
and/or some implementation. 
