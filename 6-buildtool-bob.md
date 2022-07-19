# Adding ibmi-bob as the build tool

In the tooling chapter of this book, we installed ibmi-bob with yum. We did this because ibmi-bob will be the build tool we use to build our entire application. This section is assuming source code has been moved to the IFS.

## Verify bob is installed.

While the application is called Better Object Builder (BOB), the command to invoke it is `makei`. To verify BOB is installed, we can use `makei -v` to get the version.

```
-bash-5.1$ makei -v
Bob version 2.3.9
```

## Bob implementation

BOB has two main parameters.

* `makei build` to build the entire project
* `makei compile -f mysource.rpgle` to build a single object, and items it depends on

## Getting started (`iproj.json`)

We are going to start with a simple project, [company_system](https://github.com/worksofliam/company_system), which is available as open-source on GitHub.

BOB requires a file called `iproj.json`, which describes information about your project, as well as some build configuration.

We're going to create `iproj.json` in the root of the project, like so:

```json
{
    "version": "0.0.1",
    "description": "The company system",
    "objlib": "&BINLIB",
    "preUsrlibl": ["&BINLIB", "SAMPLE"],
    "includePath": ["qrpgleref"],
    "setIBMiEnvCmd": [],
    "repository" : "https://github.com/worksofliam/company_system"
}
```

The `SAMPLE` library is included here as that's what the programs use to fetch data. The sample schema can be created by using this system-supplied SQL procedure:

```sql
call qsys.create_sql_sample('SAMPLE')
```

If you try and run `makei build`, you will see that it complains because we don't have `BINLIB` defined.

```
-bash-5.1$ makei build
BINLIB must be defined first in the environment variable.
```

#### Parameter

You can pass in environment variables as parameter.

```
-bash-5.1$ makei build -e BINLIB=LIAMA
Set variable <BINLIB> to 'LIAMA'
> make -k BUILDVARSMKPATH="/tmp/tmpdh7q_j4u" -k BOB="/QOpenSys/pkgs/lib/bob" -f "/QOpenSys/pkgs/lib/bob/mk/Makefile" all
/QOpenSys/pkgs/lib/bob/mk/Makefile:66: /home/LINUX/company_system/Rules.mk: No such file or directory
make: *** No rule to make target '/home/LINUX/company_system/Rules.mk'.
make: Failed to remake makefile '/home/LINUX/company_system/Rules.mk'.
make: Nothing to be done for 'all'.
Objects:             0 failed 0 succeed 0 total
Build Completed!
```

#### Environment variable

You can set the environment variables before the `build` is executed.

```
-bash-5.1$ export BINLIB=LIAMA
-bash-5.1$ makei build
> make -k BUILDVARSMKPATH="/tmp/tmpcu8_632a" -k BOB="/QOpenSys/pkgs/lib/bob" -f "/QOpenSys/pkgs/lib/bob/mk/Makefile" all
/QOpenSys/pkgs/lib/bob/mk/Makefile:66: /home/LINUX/company_system/Rules.mk: No such file or directory
make: *** No rule to make target '/home/LINUX/company_system/Rules.mk'.
make: Failed to remake makefile '/home/LINUX/company_system/Rules.mk'.
make: Nothing to be done for 'all'.
Objects:             0 failed 0 succeed 0 total
Build Completed!
```

### Understanding bob file types

BOB compiles objects based on what the extension is on the source.

For example:

* `.rpgle` creates an RPGLE module
* `.pgm.rpgle` creates an RPGLE program
* `.rpgleinc` doesn't create anything, but knows that it's an include file (sometimes referred to as a copybook)
* `.dspf` creates a display file object

It's important to correct all extensions before contining to build with BOB.

### Casing issues

BOB only supports extensions (compile targets) that are uppercase, and the source has to match the target. For the sake of simplicity, I changed all my source to be uppercase. Directory casing does not matter.

```
renamed:    qrpgleref/constants.rpgle -> qrpgleref/CONSTANTS.RPGLEINC
renamed:    qrpglesrc/depts.sqlrpgle -> qrpglesrc/DEPTS.PGM.SQLRPGLE
renamed:    qrpglesrc/employees.sqlrpgle -> qrpglesrc/EMPLOYEES.PGM.SQLRPGLE
renamed:    qddssrc/depts.dspf -> qddssrc/DEPTS.DSPF
renamed:    qddssrc/emps.dspf -> qddssrc/EMPLOYEES.DSPF
renamed:    qrpglesrc/mypgm.rpgle -> qrpglesrc/MYPGM.PGM.RPGLE
```

You can see the official issue for this on [GitHub](https://github.com/IBM/ibmi-bob/issues/44).

### Creating your dependency list

For BOB projects, the root of your application source code (where the `iproj.json` is) also needs a `Rules.mk` file. This file lists all the directories with sources that get built into objects.

For the case of company_system:

```makefile
SUBDIRS = qrpglesrc qddssrc
```

We need to put another `Rules.mk` file in the `qrpglesrc` directory. This file lists what each source in this file depends on being built before itself is, as well as a list of the programs to be built.

```makefile
PGMs = DEPTS.PGM EMPLOYEES.PGM

DEPTS.PGM: DEPTS.FILE $(d)/DEPTS.PGM.SQLRPGLE
EMPLOYEES.PGM: EMPS.FILE $(d)/EMPLOYEES.PGM.SQLRPGLE
```

We put one more `Rules.mk` into `qddssrc` to tell BOB how to build the display files:

```makefile
DSPFs = DEPTS.FILE EMPS.FILE

DEPTS.FILE: $(d)/DEPTS.DSPF
EMPS.FILE: $(d)/EMPS.DSPF
```

### Log files

BOB creates a handful of useful files from the build.

* `.deps` which has no purpose at the moment.
* `.evfevent` contains the compiler errors, though they aren't too human-readable.
* `.logs` has the useful files: 
   * `joblog.json` which is the joblogs from the last build/compile
   * `output.log` is all the spool files from the commands from the last build/compile

These folders belong in the `.gitignore` as they do not belong in the repository.

```
.deps
.evfevent
.logs
```

### Building your application

TODO.

## Further reading for ibmi-bob

* [BOB documentation](https://ibm.github.io/ibmi-bob/#/)
* [`iproj.json` docs](https://ibm.github.io/ibmi-bob/#/prepare-the-project/iproj-json)
* [`Rules.mk` docs](https://ibm.github.io/ibmi-bob/#/prepare-the-project/rules.mk)