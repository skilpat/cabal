# Backpack "Class Struggle" .hi-boot hack

This is a fork of `cabal-install` that will additionally copy over
any `.hi-boot` files resulting from the building of any packages 
performed as part of `cabal install`.

# Steps to reproduce

These could probably be made much simpler by someone
with more GHC/Cabal knowledge than me. But for posterity,
here's the gist:

1. Clone this repo

1. Make a sandbox for building `cabal-install` that points to the
source code in `Cabal`:

    ```
    cd cabal/cabal-install
    cabal sandbox init
    cabal add-source ../Cabal
    ```

1. Install the dependencies of `cabal-install` inside the sandbox
and then build the project:

    ```
    cabal install --dependencies-only
    cabal build
    ```

1. Now you have a patched `cabal-install` executable at
`cabal/cabal-install/dist/cabal/cabal`! Let's call it `cabal-patched`.
Using that, install a package,
ideally in some totally unrelated sandbox somewhere else:

    ```
    cabal-patched install somepkg
    ```

1. At this point, the patch on the `Cabal` library, which handles the
local installation of `hi` files, has not been triggered. That's because
`cabal-install` uses your unpatched `Cabal` to compile a `Setup` executable
with which to actually install `somepkg`. It caches this executable
for all subsequent package builds. In my case, it created the following
file:

    ```
    $HOME/.cabal/setup-exe-cache/setup-Simple-Cabal-1.22.2.0-x86_64-osx-ghc-7.8.4
    ```

1. Hmm, we want to hijack that magic path by putting in our own, _patched_
`Setup` executable. First build the executable with the patched `Cabal` package
from your clone:

    ```
    cd cabal/Cabal
    ghc --make -threaded Setup.hs
    ```

1. Now simply copy the resulting `Setup` executable to exactly the path from
before, overwriting the existing one:

    ```
    cp -f Setup $HOME/.cabal/setup-exe-cache/setup-Simple-Cabal-1.22.2.0-x86_64-osx-ghc-7.8.4
    ```

1. Voila! Now you can go back to using your patched `cabal-patched` program
to install packages. (Maybe do a `cabal-patched install --reinstall somepkg`
while you're at it.) In the build log for each package installed by that
program, you should see the following messages:

    ```
    *** BACKPACK "CLASS STRUGGLE" PATCH FOR .hi-boot FILES ENABLED (2/2) ***
    *** If there are any .hi-boot files to install, it will be done! ***
    ...
    Installing dist/dist-sandbox-361d48e4/build/Foo.hi-boot to <cabal-sandbox>/lib/<ghc-info>/<somepkg>/Foo.hi-boot
    ```

1. Now you're done! Any `.hs-boot` files in the source code of (the libraries of)
packages will now result in `.hi-boot` files copied into the `libdir` of the
installed package. ... Come to think of it, you may not have needed to build
`cabal-patched` at all. Perhaps it's enough to just put your patched `Setup`
executable in the `setup-exe-cache` and let your standard `cabal-install`
program pick up the change. Ah, well.
