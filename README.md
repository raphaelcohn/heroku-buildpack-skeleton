# heroku-skeleton-buildpack

Skeleton buildpack; experimental.

* Next step: Look at integrating support for shellfire dependency extraction (essentially, get a list of packages)
* To configure via source control
  * SSL endpoints (heroku certs)
  * .profile.d scripts (if appropriate)
  * snippets of detect/release/compile (source'd)
    * need to find a way to allow the use of additional shellfire modules
  * deleting of unused environment variables
     * Include a folder of 'should be present' env variable files
  * profiles, eg of different env vars; look at how pipelines work  
* Replace configuration refspec with code that detects current git branch / commitish? That way when branching we don't need to modify code for heroku pushes...
* Heroku pipelines
* Detect git push failure with final line containing `error: failed to push some refs to 'https://git.heroku.com/heroku-skeleton.git'`?
* Merge YAML from multi-buildpacks for release
