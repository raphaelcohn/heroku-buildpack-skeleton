# heroku-skeleton-buildpack

A [heroku] buildpack built with [shellfire] which can be added as a submodule to your own app repository, and so make buildpack management simple and versioned. Does a lot of sensible things. Most importantly, it tries to capture everything you need in source control for a [heroku] app, and to ensure clean separation of build time and run time dependencies (which most other buildpacks do not). Features:-

* Allows you to check in all your app config (eg names, regions, etc)
  * Just put them into `.heroku.rc.d/heroku.conf`
  * Or add them as snippets in `.heroku.rc.d/heroku.conf.d`, if there are things you need to `.gitignore` (eg secret keys)
  * Put configuration variables in `.heroku.rc.d/configuration-variables`
* A `deploy` script to create or re-deploy your app, which adjusts app settings to those in source control
* Ensures no dotfiles (eg `.buildpacks`, `Procfile`) pollute your root of your repository
  * Everything's in a nice `.heroku.rc.d` folder
* Correctly separates build and runtime dependencies, using
  * [linuxbrew] correctly with a cache
    * so that previously installed recipes are re-used
	* so that dependencies that are no longer needed are removed
	* works better than `apt`, etc as it ensures packages are correctly relocated
	* strips it, so your Dyno is smaller and has less vulnerabilities
  * A source-control managed root akin to `/opt` in `.heroku.rc.d/build.root`
    * allows you to check in [shellfire] and other scripts to source control
	* to use git submodules to manage private binary dependencies
* Buildpack is part of your repository; extend it with `.compile.sh` extensions
* Framework for installing build dependency packages on-the-fly using annotations, from [shellfire]
* A replacement for [heroku-buildpack-multi] which works by looking for git submodule buildpacks
  * And correctly separates detect, compile and release
* A place to store `.profile.d` scripts
* Makes the `BUILD_DIR` contents read-only
  * Removes owner, group and world writable from all files and folders
  * Installs a `BUILD_DIR/tmp` folder which is read-write-execute
  * Cleans up `BUILD_DIR`, removing `.heroku.rc.d`, `deploy`, `COPYRIGHT`, `LICENCE` and `README.md` files (stuff typical from GitHub hosted projects)

MIT licensed. To get going with an example, check out [heroku-buildpack-skel-example].


## Adding this to your app

From the root of your app:-

```bash
mkdir -m 0755 -p .heroku.rc.d
cd .heroku.rc.d
git submodule add https://github.com/raphaelcohn/heroku-buildpack-skeleton.git
git submodule update --init --recursive
```

That's it; everything else is optional. See below for the things you can modify and configure, or check out check out [heroku-buildpack-skel-example] for examples.


## Deploying your app

Run `.heroku.rc.d/heroku-buildpack-skeleton/deploy` from the root of your repository. Of course, that's a bit long winded, so you could just add a symlink to it:-

```bash
ln -s .heroku.rc.d/heroku-buildpack-skeleton/deploy
```

Of course, that means there's another file in your root, but at least it's a useful one that does what it says on the tin (:-) .


## Structure of `./heroku.rc.d`

```
	.heroku.rc.d/
		heroku.conf         				Optional, specify app name, region, stack, remote, domains in here
		heroku.conf.d/						Optional
			<x>.conf        				Optional files ending '.conf'; allows one to break up the config
											Like Debian run-parts (or apt sources.list.d)
											Useful if some configuration should not be in source control
											Or needs to vary by branch
				
        buildpacks/
			heroku-buildpack-skeleton/		Mandatory: git submodule of https://github.com/raphaelcohn/heroku-buildpack-skeleton.git
				deploy						A binary that can deploy your entire app with just './deploy'.
											Run into from the root of your repository (ie .heroku.rc.d's parent)
			
			A.heroku-buildpack-xxx			Optional: Other buildpacks, named so they sort in shell glob expansion order
			                                Should be git submodules ordinarily
											Executed as final step in compile as sorted
											eg submodule add https://github.com/peterkeen/heroku-buildpack-vendorbinaries.git A.heroku-buildpack-vendorbinaries
											
		buildpack-dotfiles/					Put any files in here needed by buildpacks or expected in the root by them
			Procfile						Your Procfile goes here
			.vendor_urls					Example: used when using https://github.com/peterkeen/heroku-buildpack-vendorbinaries.git
			
		build.linuxbrew						List of linuxbrew packages that are build dependencies
											(blank lines and lines starting # ignored)
											Cached
											
		run.linuxbrew						List of linuxbrew packages that are runtime dependencies
											(blank lines and lines starting # ignored)
											Cached
											
		build.root/							Files to copy to build dependencies and include in PATH during build
											See below
											
		run.root/							Files to copy to runtime dependencies and include in PATH during run
											See below
											
		configuration-variables/			Put files named by configuration variable in here
			EXAMPLE							Environment variable is 'EXAMPLE', contents are its value VALUE
											Instead of setting heroku config:set EXAMPLE VALUE on the command line, and losing control...
											Trailing new lines are stripped
											
		extensions/							Put extensions in here as folders ending '.compile.d'; run in shell glob expansion order
			example.compile.d/				Example; could be a git submodule
				example.compile.sh			All scripts ending '.compile.sh' are sourced ('. ./example.compile.sh') in shell glob expansion order
											If they want to define functions, they should do so in the namespace '_heroku_extension_SCRIPTNAME', eg
											_heroku_extension_example() { echo hello; }
											Working directory (pwd) will be the script's parent (eg example.compile.d/)
											
		run.profile.d/						Put any .profile.d scripts to deploy in here; make sure they end in '.sh'
			example.sh						Example .profile.d script
	
	deploy -> .heroku.rc.d/heroku-buildpack-skeleton/deploy		See deploy above
```

For an example structure, look at [heroku-buildpack-skel-example].


### Configuration Syntax for `heroku.conf` and `heroku.conf.d/<x>.conf`
	
	Check out <https://raw.githubusercontent.com/raphaelcohn/heroku-buildpack-skel-example/master/.heroku.rc.d/heroku.conf>.


### Structure of `.heroku.rc.d/build.root` and `.heroku.rc.d/run.root`

These two folders are organised the same way, like `/opt`:-

```
	.heroku.rc.d/
		build.root/
			bin/
				example -> ../<package>/bin/example
			info/
				example.5 -> ../<package>/man/example.5
			man/
				example.5 -> ../<package>/man/example.5
			lib/
				libexample.so -> ../<package>/lib/libexample.so
			<package>/
				... package files ..., eg
				bin/
					example
				info/
					something
				man/
					example.5
				lib/
					libexample.so
```

The folders are prepended to the relevant path variables which are then `export`ed:-

* The `bin` folder is prepended to `PATH`
* The `info` folder is prepended to `INFOPATH`
* The `man` folder is prepended to `MANPATH`
* The `lib` folder is prepended to `LD_LIBRARY_PATH`

This design is most useful for binaries written in interpreted code, eg [shellfire] ones.


### Environment Variables Available when `source`'ing compile extensions

* `LOG_TOKEN USER_LOG_FILE RUN_KEY RECEIVE_DATA SOURCE_VERSION BUILD_INFO STACK DEPLOY LOG_FILE REQUEST_ID DYNO`


## Known Issues

* When changing the installed production dependencies for [linuxbrew] old taps are left behind even when their are no known formulæ that are needed from these taps
  * This is weakness of [linuxbrew], which doesn't have (unlike `apt`) anyway of detecting automatically installed packages or taps
* When running `./deploy`, we detect if an application has been previously created by checking the git remotes. This doesn't work if a second user clones the repo.
* When running all `detect` proceses in buildpacks, the resultant YAML isn't merged together but the last run buildpack's YAML is used (or if none produced, a default)
  * Merging YAML is a hard problem that would require some bespoke non-shellscript code
* The use of app names precludes the practice of naming apps differently for different build environments
  * This is probably fixable by defining a subdomain and 'prefixes' for different environments
  * It is also fixable by having different `.heroku.rc.d/heroku/*.conf` snippets for different branches, but only if branches are long-lived
* Deployment failure isn't detected and `./deploy` exits with code `0`
  * Detect git push failure with the final line containing `error: failed to push some refs to 'https://git.heroku.com/heroku-skeleton.git'`, say?


## TODO

* Replace the use of the Heroku Toolbelt with `curl`
* xxxx: source snippets, allow use of subfolders so can be stored in source control as submodules and re-used...
* linuxbrew caching for runtime dependencies
* linuxbrew pinning of entire taps
* linuxbrew upgrade & cleanup
* linuxbrew production stripping
* Need to find a way to allow the use of additional shellfire modules for source'd snippets
* Look at integrating support for shellfire dependency extraction (essentially, get a list of packages)
  * Test it out with banias and swaddle
* To configure via source control
  * SSL endpoints (heroku certs)
  * deleting of unused environment variables
     * Include a folder of 'should be present' env variable files
  * per-environment configuration variables
* Replace configuration refspec with code that detects current git branch / commitish? That way when branching we don't need to modify code for heroku pushes...
* Explore whether we can adopt Heroku pipelines





[heroku-buildpack-skeleton]: https://github.com/raphaelcohn/heroku-buildpack-skeleton "heroku-buildpack-skeleton homepage"
[heroku-buildpack-skel-example]: https://github.com/raphaelcohn/heroku-buildpack-skel-example "heroku-buildpack-skel-example homepage"
[shellfire]: https://github.com/shellfire-dev "shellfire homepage"
[heroku]: https://heroku.com/ "heroku homepage"
[linuxbrew]: http://brew.sh/linuxbrew "linuxbrew homepage"
[heroku-buildpack-multi]: https://github.com/heroku/heroku-buildpack-multi "heroku-buildpack-multi homepage"
