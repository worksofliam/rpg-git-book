# Deployment

This chapter is about different methods of deployment. Every businesses deployment strategy is different, so make sure to build a deployment strategy that works for you. Below will give you some ideas of different deployment methods when using modern source control.

If you are not using any change management and you already have your own deployment system in place, then you don't have to change it. If you have a totally manual process, then it may be worth looking into new deployment methods.

## Source rebuild

It's totally normal for all objects to be rebuilt when a release is made. For example, if you were ready to deploy your app you would simply rebuild your latest changes into your production library. Or if you want to deploy onto a different system, you'd have to SSH into that system, pull down the repository of the application and rebuild that.

```
git pull
gmake BIN_LIB=PROD_LIB
```

IBM i combined with strong hardware means compiles happen fast. It's totally normal in other ecosystems (in the Linux world) to rebuild applications.

## Save file deployment

It is possible for your `makefile` to generate a save file of your master library to then deploy that with your own method. This method is also useful if you only provide savefiles to deployment/client systems.

For example:

```
release:
	@echo " -- Creating release. --"
	@echo " -- Creating save file. --"
	system "CRTSAVF FILE($(BIN_LIB)/RELEASE)"
	system "SAVLIB LIB($(BIN_LIB)) DEV(*SAVF) SAVF($(BIN_LIB)/RELEASE) OMITOBJ((RELEASE *FILE))"
	-rm -r release
	-mkdir release
	system "CPYTOSTMF FROMMBR('/QSYS.lib/$(BIN_LIB).lib/RELEASE.FILE') TOSTMF('./release/release.savf') STMFOPT(*REPLACE) STMFCCSID(1252) CVTDTA(*NONE)"
	@echo " -- Cleaning up... --"
	system "DLTOBJ OBJ($(BIN_LIB)/RELEASE) OBJTYPE(*FILE)"
	@echo " -- Release created! --"
	@echo ""
	@echo "To install the release, run:"
	@echo "  > CRTLIB $(BIN_LIB)"
	@echo "  > CPYFRMSTMF FROMSTMF('./release/release.savf') TOMBR('/QSYS.lib/$(BIN_LIB).lib/RELEASE.FILE') MBROPT(*REPLACE) CVTDTA(*NONE)"
	@echo "  > RSTLIB SAVLIB($(BIN_LIB)) DEV(*SAVF) SAVF($(BIN_LIB)/RELEASE)"
	@echo ""
```

This could be invoked with `gmake release BIN_LIB=DEV_MASTER`, which generates `./release/release.savf`.

## CI/CD

It is possible to automatically deploy your application using continuous integration and deployment tools like Jenkins. You can set up your CI/CD software to automatically build your master libraries as developers make commits and automatically deploy to your test/production environments when a tag (or GitHub Release) is made.

Jenkins is built on Java and can run on IBM i or on another platform to connect to IBM i.