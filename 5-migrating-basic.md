# Starting the migration

This chapter will cover what migration means, what tools can be used to migrate source members, handling changes to the copy books, compiling sources from the IFS (editors and command line) and why iProjects is not used.

## What migration means

Migration actually means copying the source member contents to the IFS. Only one person has to do this step. Once all the source is migrated into git, is when normal development. If you let someone edit the source members while migrating the source to the IFS, you will have to re-copy the changed members.

For example, your library, source physical files and members might look like this:

```
DEVLIB
  - QRPGLESRC
    - PROGRAMA.RPGLE
    - PROGRAMB.RPGLE
    - PROGRAMC.RPGLE
  - QSQLSRC
    - CUSTOMERS.SQL
    - INVENTORY.SQL
  - QCLLESRC
    - STARTJOB.CLLE
  - QCMDSRC
    - STARTJOB.CMD
```

Where the resulting layout in the IFS could be very similar:

```
/home
  /barry
    /.git
    /qrpglesrc
      programa.rpgle
      programb.rpgle
      programc.rpgle
    /qsqlsrc
      customers.sql
      inventory.sql
    /qcllesrc
      startjob.cmd
    /qcmdsrc
      startjob.cmd
```

Make sure that when you migrate code, you are migrating code into a directory which is a local repository (e.g. you cloned it) so you can commit & push changes as you make them. For example, make your first commit when you've migrated the code and then make another (or multiple) after you fix up the 'copybooks'.

**Notes about migrating to the IFS**:

1. You will lose the TEXT column that source members have, which is usually used for describing what the source is. Although, you can still put that in the program as a comment.
2. The type of the source member becomes the extension when in the IFS.
3. Files and directories of sources are usually stored as all lowercase.
4. It is recommended you retain the 10 character limit on programs, commands, modules, etc - any source related to Db2 for i doesn't matter as much as Db2 for i and most ILE languages support 'long names'
5. Sources on the IFS should be stored as encoding 1208 (UTF-8) or 1252.

## Tools used for migration

Initially migrating the source code can be the hardest part of the entire process, but once it's done: it's done. There are many ways to do it, but this will only describe two.

### 1. Manually migrating

All a migration consists of is moving source members to the IFS. To our benefit, the `CPYTOSTMF` command exists, which can be used to copy a source member to a stream file. For example:

```
CPYTOSTMF FROMMBR('/QSYS.lib/DEVLIB.lib/QRPGLESRC.file/PROGRAMA.mbr') TOSTMF('/home/barry/myproject/qrpglesrc/programa.rpgle') STMFOPT(*REPLACE) STMFCCSID(1208)
```

On the basis of this command, you would have to run this command for each source member you want to migrate.

### 2. Using the migrate tool

There is an open source migrate tool, simply named 'migrate', which automates the copying of source members into a directory. It also creates the streamfiles with the correct extensions.

To use the migrate tool, you will need to clone it and build it manually.

```
git clone https://github.com/worksofliam/migrate.git
cd migrate
gmake
```

Building this solution will create the `MIGRATE` library and inside is the `MIGSRCPF` command. `MIGSRCPF` has three simple parameters.

![](./images/migsrcpf.PNG)

If we had a library with source physical files and wanted to migrate them into a new project directory, we would have to run the command once to migrate the source physical file. It will copy the source member into the IFS as a 1208 (UTF-8) streamfile. If the file or folder it tries to create already exists, it will fail. In the last chapter, we created `/home/BARRY/myproject` as a git repository.

```
MIGSRCPF LIBRARY(TESTPROJ) SOURCEPF(QRPGLESRC) OUTDIR('/home/BARRY/myproject')
MIGSRCPF LIBRARY(TESTPROJ) SOURCEPF(QRPGLEREF) OUTDIR('/home/BARRY/myproject')
MIGSRCPF LIBRARY(TESTPROJ) SOURCEPF(QCLLESRC)  OUTDIR('/home/BARRY/myproject')
```

This would create three directories in `/home/BARRY/myproject` like the following:

```
/home
  /BARRY
    /myproject
      /qrpglesrc
        /somesource.rpgle
        /somesource.rpgle
      /qrpgleref
        /whatever.rpgle
      /qcllesrc
        /pgm1.clle
        /pgm2.clle
        /pgm3.clle
```

Note that it will create all directories and stream files with lowercase names.

### Other possible ways.

You could potentially create an iProject in RDi based on a library and then have a local copy of all the source (which you can then put into a git repository later). You can also use the SPF Clone tool in ILEditor to clone a libraries source members on to your local machine (which can also be put into a git repository later).

### Don't forget to commit changes

Make sure, when you've finished migrating source code into your local developer repository. For example:

```
git add --all
git commit -m "Initial migration step"
git push
```

You will want to do this again (maybe multiple times) as you change the copybooks too.

## Handling 'copybooks' (`/COPY` & `/INCLUDE`)

Once the source has been migrated, another tedious task is changing all the copy books to point to your newly migrated stream files. We're lucky that the C, RPG and COBOL compilers for IBM i all support bringing includes in from the IFS. In this chapter, we will use RPG as it's the primary target audience for this book.

Let's say we have a program that has `/COPY` (or `/INCLUDE`) statements like the following at the top of the source:

```

      /COPY QRPGLEREF,OBJECTS
      /COPY QRPGLEREF,OBJECT
      /COPY QRPGLEREF,FORMATS
      /COPY QRPGLEREF,MEMBERS

      ** --------------------------

     D testproj        PI
     D    pLibrary                   10A   Const

      ** --------------------------
      
```

Even though this source might be in the IFS, `/COPY` (or `/INCLUDE`) can still bring in source from source members in the QSYS file system (and vice versa). What the developer should do is change the statements to use a relative path based on the root of the project to the respective streamfile on the IFS. For example `/COPY QRPGLEREF,OBJECTS` might translate to `/COPY './qrpgleref/objects.rpgle'`.

```
      /COPY `./qrplgeref/objects.rpgle`
      /COPY `./qrplgeref/object.rpgle`
      /COPY `./qrplgeref/formats.rpgle`
      /COPY `./qrplgeref/members.rpgle`
```

The reason you use a path relative to the root of the project is so we can build from the root of the project within our command line, IDE or our build system (which you will see later). It's not required that you do this to all your source at once, because you can still depend on the existing source members during a migration period - although **it is recommended you change them as soon as possible**. While it's not recommend, you can do it iteratively and change them when you work on the source. This is dangerous because the source members aren't under change control.

If you are using a 3rd party tool, like the HTTPAPI, which as it's headers in source members, then you can leave those `/COPY` (or `/INCLUDE`) statements along side your includes which point to the IFS:

```
      /COPY `./qrplgeref/objects.rpgle`
      /COPY `./qrplgeref/object.rpgle`
      /COPY `./qrplgeref/formats.rpgle`
      /COPY `./qrplgeref/members.rpgle`
      /COPY QRPGLEREF,HTTPAPI_H
```

## Compiling IFS sources

You are able to compile most sources out of the IFS using your IDE or by the command line. The following list of commands/compilers have support for compiling sources out of the IFS on IBM i with the `SRCSTMF` parameter:

* `CRTBNDRPG` / `CRTRPGMOD`
* `CRTSQLRPGI`
* `CRTSRVPGM` (for binder source) - 7.2+
* `CRTBNDCL` / `CRTCLMOD` - 7.3+
* `RUNSQLSTM`
* `CRTBNDC` / `CRTCMOD`
* `CRTBNDCBL` / `CRTCBLMOD`

When you invoke the compile commands, whether from a 5250 shell, pase shell or IDE, make sure you set the jobs current working directory to the root/directory of the project you're working with to make sure the sources `/INCLUDE` and `/COPY` statements work as intended (as shown in the chapter about copybooks).

You can change your working directory in a 5250 job using either `CD` or `CHGCURDIR`. You can also change your working directory in RDi and Code for IBM i when compiling sources. When working in pase, you can use `cd` to change working directory.

## Compiling non-IFS sources

For compilers like `CRTBNDCL` or `CRTCMD` (or any without the `SRCSTMF` parameter), in the case that a business has not updated the 7.3 yet, to compile the source the developer would have to create a temporary source member in the same library they are building the object in (which should be their own developer library) and then compile that source member. For example:

```
CRTSRCPF FILE(BARRYDEV/QCMDSRC) RCDLEN(112)
CPYFRMSTMF FROMSTMF('./qcmdsrc/mycmd.cmd') TOMBR('/QSYS.lib/BARRYDEV.lib/QCMDSRC.file/mycmd.mbr') MBROPT(*REPLACE)
CRTCMD CMD(BARRYDEV/MYCMD) PGM(BARRYDEV/MYCMD) SRCFILE(BARRYDEV/QCMDSRC)
```

This process can also be automated tool called CRTFRMSTMF, which was created by Brian Garland. It adds the `CRTFRMSTMF` command, which automated the previous three commands into one command: `CRTFRMSTMF`. [You can find that tool on GitHub](https://github.com/BrianGarland/CRTFRMSTMF).

This book will cover building things like data areas, tables and message files in another chapter.