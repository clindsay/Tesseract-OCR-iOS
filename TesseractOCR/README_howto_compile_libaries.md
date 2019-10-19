
### Step 1 - Prerequisites
First you need to install these tools:

- Latest Xcode with command line tools
- M4
- Autoconf
- Automake
- Libtool
- pkg-config

### Step 2 - Build
See Notes and gotchas first... you may need some.

Run `make` in the `TesseractOCR` subfolder. This first compiles dependent libraries (png, jpeg, tiff, leptonica) and then tesseract for every architecture iOS/simulator uses (arm7 arm7s arm64 i386 x86_64), and then combines the resulting libs into one library file. It does this for both dependent libraries and tesseract, so the final results of the script are "libpng.a", "libjpeg.a", "libtiff.a", "libtesseract.a", "liblept.a", and "include" directories for both leptonica, tesseract and image libraries. Finally, the script copies these results into the "lib" and "include" directories inside `TesseractOCR` directory.

The very first total build (includinf all architectures) may take half an hour or so depending on the processing power, but the later builds will not build dependencies until any files being changed.

By default every "fat" library will contain all architectures specified above. So it can be linked with apps either for devices or simulator. If you don't need all architectures above (for example, for AppStore submittion), just specify the necessary architectures in the `ARCHS` environement variable as follows:

    export ARCHS=armv7, armv7s, arm64

### Notes and gotchas
These are things which caused me (as a newcomer) headaches and plenty of wasted time. Leaving here for future reference until they are resolved better. Hope it saves you and my future self some time.

#### Compilation for SSE, AVX enabled architectures
`tesseract-ocr/configure.ac` detects SSE, AVX support always as true causing related errors in the Tesseract 4.1+ code. Not sure why.

As for iOS we never need it, the workaround to get a working built without these errors is to change each such related detection from:

`AX_CHECK_COMPILE_FLAG([-mavx], [avx=true], [avx=false], [$WERROR])`

to (both outputs give false):

`AX_CHECK_COMPILE_FLAG([-mavx], [avx=false], [avx=false], [$WERROR])`

I don't understand the exact mechanics for the detection yet.

TODO: Investigate, possibly remove `-Qunused-arguments` and split `common_cflags` for `C` and `CXX`.

#### Proper recompilation after reconfiguration
`./configure` prepares code to individual architecture folders (e.g. `arm-apple-darwin64`) under respective `leptonica-<ver>` and `tesseract-ocr` projects, but as make is lazy, and clean doesn't seem to fully clean everything, you may need to delete those folders manually when changing build configuration.

TODO: Review and update `clean`.

#### Updating leptonica to higher version
Each version of `leptonica` brings new features, sometimes expecting additional external libraries unless you disable them explicitly.

Sadly, `leptonica` doesn't preserve branches for individual version, but only archives with them. Therefore `leptonica-<ver>` folder is downloaded automatically.

Make sure to delete the older leptonica folder if it was already downloaded / compiled and also cleanup `tesseract-ocr` folder (see above).

If you get a linker error on missing symbols after increasing `leptonica` version, you either need to link that extra library (extra steps and probably not needed in most cases).

Which exact library you need to find out based on involved file names, as for example this one was based on  `jp2kio.o`. You can find all of them in `leptonica-<ver>/configure.ac` so a safe bet may be disable all new, and only include them when needed. The full list is defined with checks like this one:

`AC_ARG_WITH([libopenjpeg], AS_HELP_STRING([--without-libopenjpeg], [do not include libopenjpeg support]))`

In this case it needed adding `--without-libopenjpeg` to the call to `./configure` for the `leptonica` target in the make file.

#### Processing in iOS is most likely single-threaded
As outlined [here](https://github.com/tesseract-ocr/tesseract/wiki/Compiling#prepare-support-for-openmp-optional), the default `tesseract-ocr` build doesn't leverage more threads when compiled on/for Mac. Not sure if this can be applied to iOS, but seems to be worth a shot.

TODO: Try compilation with multi-threaded support.
