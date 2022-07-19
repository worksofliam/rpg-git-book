# Adding make as the build tool

In the tooling chapter of this book, we installed GNU Make with yum. We did this because GNU Make will be the build tool we use to build our entire application. This chapter will cover all the nitty gritty bits, like building data areas, message files, table, programs, service programs, binding directories, the lot.

## The `system` command in pase

The pase environment provides the `system` command which allows us to run ILE commands from a pase shell. The `system` command will run the command in a new job - this should be remembered as important when dealing with library lists and the use of QTEMP.

Some examples of the command are as follows (direct from the IBM documentation site):

1. List all of the active jobs: `system wrkactjob`
2. Create a test library: `system "CRTLIB LIB(TESTDATA) TYPE(*TEST)"`
3. Delete a library and do not write any messages: `system -q "DLTLIB LIB(TESTDATA)"`

For more, you can visit the IBM documentation on the `system` command: https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzahz/rzahzsystem.htm

## Creating a makefile for ILE projects - Simple programs

### Knowing your dependancies

Let’s start with something simple. Let’s say we have this file structure.

```
/myproject
    - src
        - programa.rpgle
        - programb.rpgle
        - programc.rpgle
        - stringlib.c
        - thecl.clle
    - headers
        - stringlib.c
        - stringlib.rpgle`
```

From your source tree, you should determine what the objects you want to build are and each dependancy is. For example:

* programa (program)
  * programa (module)
  * stringlib (module)
* programb (program)
  * programb (module)
* programc (program)
  * programc (module)
  * stringlib (module)
* thecl (program)
  * thecl (module)

From this list, you should get the impression that we build all sources into modules and then into their retrospective program objects.

### Creating your rules

This information (knowing your dependancies) will allow us to create conditions (in our makefile) based on our program dependancies. First of all, we need to setup rules for what to do for each type of source we have.

```makefile
%.rpgle:
    system "CRTRPGMOD MODULE($(BIN_LIB)/$*) SRCSTMF('./src/$*.rpgle') DBGVIEW($(DBGVIEW)) REPLACE(*YES)"

%.c:
    system "CRTCMOD MODULE($(BIN_LIB)/$*) SRCSTMF('./src/$*.c') DBGVIEW($(DBGVIEW)) REPLACE(*YES)"

%.clle:
    #Can't compile CL from IFS on all OS versions..
    -system -q "CRTSRCPF FILE($(BIN_LIB)/QSRC) RCDLEN(112)"
    system "CPYFRMSTMF FROMSTMF('./src/$*.clle') TOMBR('/QSYS.lib/$(BIN_LIB).lib/QSRC.file/$*.mbr') MBROPT(*replace)"
    system "CRTCLMOD MODULE($(BIN_LIB)/$*) SRCFILE($(BIN_LIB)/QSRC) DBGVIEW($(DBGVIEW))"

%.pgm:
    system "CRTPGM PGM($(BIN_LIB)/$*) MODULE($(patsubst %,$(BIN_LIB)/%,$(basename $^))) ENTMOD($*) REPLACE(*YES)"

all:
    @echo "Build finished!"
```

That’s all our rules setup, but let's look at the GNU Make functions and variables we are using:

* `$(BIN_LIB)` and `$(DBGVIEW)` are variables which we haven’t defined yet - that’s next.
* `$(patsubst ...` is a pattern substring function in make. We are using this to prepend the binary library for each module.
* `$(basename value)` is a function that will return the basename of a path. E.g. `myfile.rpgle` -> `myfile`.
* `$*` is used to get the current rule name that we’re working with.
* `$^` returns the values passed into the current condition.

### Dependancy list

Next, we can setup the dependancy list. **This must be placed above the rules we just created**:

```makefile
# First our variables
BIN_LIB=MYLIBRARY
DBGVIEW=*ALL

# Define what needs to get built when make is run
all: programa.pgm programb.pgm programc.pgm thecl.pgm

# %program%: depends on...

programa.pgm: programa.rpgle stringlib.c
programb.pgm: programb.rpgle
programc.pgm: programc.rpgle stringlib.c
thecl.pgm: thecl.clle

# Below are the rules.
```

### Executing GNU Make

From there, developers can call make with or without optional parameters:

* `make` - will make all (because all is the first rule)
* `make programa.pgm` will build only `programa.pgm`
* `make stringlib.c` will only build `stringlib.c`
* `make BIN_LIB=DEVLIB` will `make all`, but with the `BIN_LIB` variable as `DEVLIB`.

If you run `gmake -n` against this `makefile`, you can see the commands it would run.

```
barry$ make -n

system "CRTRPGMOD MODULE(MYLIBRARY/programa) SRCSTMF('./src/programa.rpgle') DBGVIEW(*ALL) REPLACE(*YES)"
system "CRTCMOD MODULE(MYLIBRARY/stringlib) SRCSTMF('./src/stringlib.c') DBGVIEW(*ALL) REPLACE(*YES)"
system "CRTPGM PGM(MYLIBRARY/programa) MODULE(MYLIBRARY/programa MYLIBRARY/stringlib) ENTMOD(programa) REPLACE(*YES)"
system "CRTRPGMOD MODULE(MYLIBRARY/programb) SRCSTMF('./src/programb.rpgle') DBGVIEW(*ALL) REPLACE(*YES)"
system "CRTPGM PGM(MYLIBRARY/programb) MODULE(MYLIBRARY/programb) ENTMOD(programb) REPLACE(*YES)"
system "CRTRPGMOD MODULE(MYLIBRARY/programc) SRCSTMF('./src/programc.rpgle') DBGVIEW(*ALL) REPLACE(*YES)"
system "CRTPGM PGM(MYLIBRARY/programc) MODULE(MYLIBRARY/programc MYLIBRARY/stringlib) ENTMOD(programc) REPLACE(*YES)"
# Can't compile CL from all OS versions..
system -q "CRTSRCPF FILE(MYLIBRARY/QSRC) RCDLEN(112)"
system "CPYFRMSTMF FROMSTMF('./src/thecl.clle') TOMBR('/QSYS.lib/MYLIBRARY.lib/QSRC.file/thecl.mbr') MBROPT(*replace)"
system "CRTCLMOD MODULE(MYLIBRARY/thecl) SRCFILE(MYLIBRARY/QSRC) DBGVIEW(*ALL)"
system "CRTPGM PGM(MYLIBRARY/thecl) MODULE(MYLIBRARY/thecl) ENTMOD(thecl) REPLACE(*YES)"
echo "Build finished!"
```

As a cool side note: even though we defined `stringlib.c` as a dependancy for two programs, it only built once.

## Creating a makefile for ILE projects - Service programs and binding directories

Service programs are made up of three important things:

* The module (or modules)
* The binder source
* The binding directory entry.

### Creating your rules

Let’s go ahead and create rules for creating the `.srvpgm` object, `.rpgle` module and `.bnddir` entry.

```makefile
%.rpgle:
    system "CRTRPGMOD MODULE($(BIN_LIB)/$*) SRCSTMF('./src/$*.rpgle') DBGVIEW($(DBGVIEW)) REPLACE(*YES)"

%.srvpgm:
    # We need the binder source as a member! SRCSTMF on CRTSRVPGM not available on all releases.
    -system -q "CRTSRCPF FILE($(BIN_LIB)/QSRC) RCDLEN(112)"
    system "CPYFRMSTMF FROMSTMF('./src/$*.binder') TOMBR('/QSYS.lib/$(BIN_LIB).lib/QSRC.file/$*.mbr') MBROPT(*replace)"

    system "CRTSRVPGM SRVPGM($(BIN_LIB)/$*) MODULE($(patsubst %,$(BIN_LIB)/%,$(basename $^))) SRCFILE($(BIN_LIB)/QSRC)"

%.bnddir:
    -system -q "CRTBNDDIR BNDDIR($(BIN_LIB)/$*)"
    -system -q "ADDBNDDIRE BNDDIR($(BIN_LIB)/$*) OBJ($(patsubst %.entry,(*LIBL/% *SRVPGM *IMMED),$^))"

%.entry:
    # Basically do nothing..
    @echo ""
```

### Dependency list

Next, we need to define the dependency list rules (this goes above our rules, like last time).

```makefile
BIN_LIB=MYLIBRARY
DBGVIEW=*ALL

all: apipkg.srvpgm webcalls.srvpgm tools.bnddir

apipkg.srvpgm: apipkg.rpgle
webcalls.srvpgm: webcalls.rpgle othermod.rpgle

tools.bnddir: apipkg.entry webcalls.entry
```

### Executing GNU Make

Next, if you run `gmake -n` you can see what commands make would execute:

```
barry$ make -n

system "CRTRPGMOD MODULE(MYLIBRARY/apipkg) SRCSTMF('./src/apipkg.rpgle') DBGVIEW(*ALL) REPLACE(*YES)"
system -q "CRTSRCPF FILE(MYLIBRARY/QSRC) RCDLEN(112)"
system "CPYFRMSTMF FROMSTMF('./src/apipkg.binder') TOMBR('/QSYS.lib/MYLIBRARY.lib/QSRC.file/apipkg.mbr') MBROPT(*replace)"
system "CRTSRVPGM PGM(MYLIBRARY/apipkg) MODULE(MYLIBRARY/apipkg) SRCFILE(MYLIBRARY/QSRC)"
system "CRTRPGMOD MODULE(MYLIBRARY/webcalls) SRCSTMF('./src/webcalls.rpgle') DBGVIEW(*ALL) REPLACE(*YES)"
system "CRTRPGMOD MODULE(MYLIBRARY/othermod) SRCSTMF('./src/othermod.rpgle') DBGVIEW(*ALL) REPLACE(*YES)"
system -q "CRTSRCPF FILE(MYLIBRARY/QSRC) RCDLEN(112)"
system "CPYFRMSTMF FROMSTMF('./src/webcalls.binder') TOMBR('/QSYS.lib/MYLIBRARY.lib/QSRC.file/webcalls.mbr') MBROPT(*replace)"
system "CRTSRVPGM PGM(MYLIBRARY/webcalls) MODULE(MYLIBRARY/webcalls MYLIBRARY/othermod) SRCFILE(MYLIBRARY/QSRC)"
echo ""
echo ""
system -q "CRTBNDDIR BNDDIR(MYLIBRARY/tools)"
system -q "ADDBNDDIRE BNDDIR(MYLIBRARY/tools) OBJ((*LIBL/apipkg *SRVPGM *IMMED) (*LIBL/webcalls *SRVPGM *IMMED))"
```

As you can see, it doesn’t really do anything with the .entry rule, because really we just need the values for the .bnddir rule.

### Truly only building changed source

**Todo: NEEDS REVIEW**: https://github.com/worksofliam/rpg-git-book/pull/3/files

Up until this point we haven't looked at how GNU Make handles only building changed sources. GNU Make usually looks at the time stamp of objects created on the IFS, but since we aren't creating objects on the IFS, there is no way for GNU Make to know when they were last changed.

What we need to do instead is **create temporary/empty files in the root of our application being built which would map to the QSYS objects that we create in our makefile**. We can do this with `touch`. `touch` will create or update the IFS objects timestamp with the current system time.

When make checks to see if the rule needs to be built, it will compare the name with the existing IFS object. If it doesn't exist or is out of date it will rebuild and create the object - otherwise it'll be left alone.

The following makefile indicates that:

1. We need to build 3 objects (PGMS variable)
2. `programc.srvpgm` depends on two other modules: `modulea.mod` and `moduleb.mod`
3. Any `%.pgm` will use `CRTBNDRPG`
4. Any `%.mod` will use `CRTRPGMOD`
5. `%.pgm` depends on the `src/%.rpgle` source, where `%` is the name of the object you're building (e.g. if you're building `programa.pgm`, it will check for `src/programa.rpgle`)
6. `%.mod` depends on the `src/%.rpgle` (like above)

```makefile
BIN_LIB=LIBRARY
PGMS=programa.pgm programb.pgm programc.srvpgm

all: $(PGMS)
	@echo "done"

programc.srvpgm: modulea.mod moduleb.mod
	system "CRTSRVPGM SRVPGM($(BIN_LIB)/programc) MODULE($(patsubst %,$(BIN_LIB)/%,$(basename $^))) SRCFILE($(BIN_LIB)/QSRC)"
	@touch $@

%.pgm: src/%.rpgle
	system "CRTBNDRPG PGM($(BIN_LIB)/$*) SRCSTMF('src/$@') REPLACE(*YES)"
	@touch $@

%.mod: src/%.rpgle
	system "CRTRPGMOD MODULE($(BIN_LIB)/$*) SRCSTMF('src/$@') REPLACE(*YES)"
	@touch $@

clean:
	rm -f *.pgm *.mod *.srvpgm
```

The above makefile is a good example of a makefile that will only rebuild QSYS objects when the source has changed. Effectively the only change you need to make is to add the `touch` command to all your rules, but would need testing.

## Building objects that have no source

Objects like message files and data areas don't have source. This can be a true challenge when replicating your environment on other systems. Sadly, there is nothing built into IBM i which allows you to create a data area or message file from source - but of course, you could use a CL to execute the command. That means, if you can use a CL to build them, you can use a `makefile` too!

In your `makefile`, you would hard-define the message files and data areas rules and then include them as a dependency for what is needed. For example:

```makefile
BIN_LIB=IUNIT

all: $(BIN_LIB).lib UTEMSG.msgf VERSION.dtaara PASSWORD.dtaara
	@echo "Built all!"

UTEMSG.msgf:
	system "CRTMSGF MSGF($(BIN_LIB)/UTEMSG)"
	system "ADDMSGD MSGID(UTE0001) MSGF($(BIN_LIB)/UTEMSG) MSG('&1') SECLVL('&N Cause . . . . . :   The unit test program reported the following non critical message. &N Recovery. . . . :   None needed') FMT((*CHAR 120)) CCSID(*JOB)"
	system "ADDMSGD MSGID(UTE0002) MSGF($(BIN_LIB)/UTEMSG) MSG('&1') SECLVL('&n Cause . . . . . :   The unit test program reported the following critical error. & N Recovery. . . . :   See the lower level messages for details') SEV(40) FMT((*CHAR 120)) CCSID(*JOB)"
	system "ADDMSGD MSGID(UTE0003) MSGF($(BIN_LIB)/UTEMSG) MSG('-------------- Starting Unit &1 --------------') FMT((*CHAR 32)) CCSID(*JOB)"
	system "ADDMSGD MSGID(UTE5000) MSGF($(BIN_LIB)/UTEMSG) MSG('Start of User Messages') SECLVL('All user messages are UTE5001 and higher') CCSID(*JOB)"

VERSION.dtaara:
	system "CRTDTAARA DTAARA($(BIN_LIB)/VERSION) TYPE(*CHAR) LEN(25) VALUE('0.1.0.201901070019') TEXT('0.1.0.201901070019')"
    
PASSWORD.dtaara:
	system "CRTDTAARA DTAARA($(BIN_LIB)/PASSWORD) TYPE(*CHAR) LEN(25) VALUE('BADPASSWORD') TEXT('Password dataarea')"

%.lib:
	-system -qi "CRTLIB $* TYPE(*TEST)"
	-system -qi "CRTSRCPF FILE($(BIN_LIB)/QCLLESRC) RCDLEN(240)"
	-system -qi "CRTSRCPF FILE($(BIN_LIB)/QCMDSRC) RCDLEN(240)"
```

## Handling library lists in GNU Make

### User profile job description

When using the `system` command in a `makefile`, the default library list is whatever is setup for the user profile. This may be a job description which is setup on the user profile.

To accomplish a change to the library list for each build, you may create a new job description in the build (`CRTJOBD`) with an initial library list, and then change the user profile to use that new job description (`CHGUSRPRF`).

This is not viable because it will affect any currently running build.

### Using `QSHELL` in make

In GNU Make, you can specify a shell using the `SHELL` variable. It is possible for us to define QSH/QSHELL as our shell for the commands to run in. This would allow us to run multiple commands in a single job.

QSHELL contains the `liblist` command, which allows us to alter the library list for that job. This might be required when working with programs or sources that define tables/files.

```makefile
BIN_LIB=PRODDEVLIB
LIBLIST=$(BIN_LIB) TABLES_LIB
SHELL=/QOpenSys/usr/bin/qsh

all: myprograma.rpgle myprogramb.rpgle

%.rpgle:
	liblist -a $(LIBLIST);\
	system "CRTBNDRPG PGM($(BIN_LIB)/$*) SRCSTMF('./QRPGLESRC/$*.rpgle')";
```

This `makefile` would run the `liblist` and `system` command in the same job.

* Read more about the GNU Make `SHELL` variable here: https://www.gnu.org/software/make/manual/html_node/Choosing-the-Shell.html
* Read more about the `liblist` command here: https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_71/rzahz/rzahzliblist.htm

### Statically binding (where possible)

When using `CRTPGM` and `CRTSRVPGM` you can statically bind modules, service programs and binding directories.

You can do this within a `makefile`.

```makefile
BIN_LIB=PRODDEVLIB

tests: program1.rpgle program2.rpgle
	@echo "Programs build!"

%.rpgle:
	system "CRTRPGMOD MODULE($(BIN_LIB)/$*) SRCSTMF('./qrpglesrc/$*.rpgle') DBGVIEW(*SOURCE)"
	system "CRTPGM PGM($(BIN_LIB)/$*) BNDDIR($(BIN_LIB)/A_BNDDIR)"
```

## Handling table changes

Db2 for i provides the `ON REPLACE` syntax, which allows for table columns to be updated without the loss of data and without maintaining alter statements.

Further information:

* Intro to `CREATE OR REPLACE`: https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/sqlp/rbafyreplacetable.htm
* Enhancement announcement (contains PTF information): https://www.ibm.com/developerworks/community/wikis/home?lang=en#!/wiki/IBM%20i%20Technology%20Updates/page/Create%20OR%20REPLACE%20Table

### Using `CREATE OR REPLACE`

*This example comes from the Intro to `CREATE OR REPLACE` on the IBM Knowledge Center website.*

Using `CREATE OR REPLACE TABLE` lets you consolidate the master definition of a table into one statement. You do not need to maintain the source for the original `CREATE TABLE` statement plus a complex list of `ALTER TABLE` statements needed to recreate the most current version of a table. This `CREATE TABLE` statement can be executed to deploy the current definition of the table either as a new table or to replace a prior version of the table. There are options to either keep the existing data in the table or to clear the data from the table during the replace. The default is to keep all data. If you elect to clear all the data, your new table definition does not need to be compatible with the original version.

**In all cases**, other objects that depend on the table, such as referential constraints, triggers, and views, must remain satisfied **or the replace will fail**.

Suppose your original table was this basic INVENTORY table in an initial build.

```sql
CREATE TABLE BIN_LIB/INVENTORY 
  (PARTNO   SMALLINT NOT NULL,
  DESCR    VARCHAR(24),
  QONHAND  INT,
  PRIMARY KEY(PARTNO))
```

Perhaps over time, you have updated the column names to be more descriptive, changed the `DESCR` column to be a longer Unicode column, and added a timestamp column for when the row was last updated. The following statement reflects all of these changes and can be executed against any prior version of the table, as long as the column names can be matched to the prior column names and the data types are compatible.

Also consider that if you commit this, your CI/CD pipeline might trigger a build, your build system (e.g. GNU Make) will run the statement, update the table and retain the data.

```sql
CREATE OR REPLACE TABLE BIN_LIB/INVENTORY 
  (PART_NUMBER FOR PARTNO        SMALLINT NOT NULL,
  DESCRIPTION FOR DESCR         VARGRAPHIC(500) CCSID 1200,
  QUANTITY_ON_HAND FOR QONHAND  INT,
  LAST_MODIFIED FOR MODIFIED    TIMESTAMP
        NOT NULL GENERATED ALWAYS FOR EACH ROW ON UPDATE AS ROW CHANGE TIMESTAMP,
  PRIMARY KEY(PARTNO))
```

### Execute SQL statements in GNU Make

This is an example `makefile` which could implement this. We must replace a library variable with the library we are going to build in. We do this using `sed`.

```makefile
BIN_LIB=DBTEST
all: inventory.sql

%.sql:
	sed -i.bak "s/BIN_LIB/$(BIN_LIB)/g" ./$*.sql
	system "RUNSQLSTM SRCSTMF('./$*.sql') COMMIT(*NONE)"
```

If you do not provide a library/schema when using `CREATE OR REPLACE`, it will use users current library - or QGPL if the current library is not set.
