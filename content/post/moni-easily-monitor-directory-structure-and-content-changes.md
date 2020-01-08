---
title: moni — easily monitor directory structure and content changes 
excerpt: moni — short for monitoring (thank you Captain Obvious) — is a small utility I have written in go. Its main purpose is to walk through a…
date: 2019-02-05
hero: /img/1__15Ck9QYmcclxb__ofhM__AaA.jpeg
timeToRead: 5
authors:
  - Adrian Gheorghe
---

[Published on Medium](https://medium.com/@adrian.gheorghe.dev/moni-easily-monitor-directory-structure-and-content-changes-fbfb89d19905)

[moni](https://github.com/adrian-gheorghe/moni) — short for monitoring (thank you Captain Obvious) — is a small utility I have written in go. Its main purpose is to walk through a path and detect changes to the file structure, making it easier to spot when code has been compromised.

Other use cases would be in workflows where you need to sync a file tree and only want to upload the diff (Although I am sure there are already tools that do that better).

moni is available in a Docker image on [Docker Hub](https://hub.docker.com/r/adighe/moni) or you can download it from [Github](https://github.com/adrian-gheorghe/moni) and integrate it in your workflow. In order to run, moni requires you to pass in the path to the configuration file you want to load.

```bash
./moni -—config=/app/config.yml
```
To get started you need to set the **path** configuration property to a directory of your choosing. moni can be set up to run periodically as a script in the foreground or just on demand so you can set the **periodic** property to either true or false.

On each run moni will walk through the directory specified in path (and its subfiles and subdirectories) and save the whole tree information to a json file defined in tree-store. Some of the information stored is the file path, type, modification time and mode permissions. For files, the checksum of the content is also calculated.

Each time it runs, it will compare the tree stored with the current one and will know if there are any updates in structure and in content. This output will be saved either to a log file or directed to stdout for the Docker container.

Using the **command-success** and **command-failure** properties, you can tell the script what shell command to call on each case.

There are 2 Walk algorithms implemented.

*   FlatTreeWalk is an algorithm that uses recursion to parse the file tree
*   GoDirTreeWalk uses the excellent walk implementation from [https://github.com/karrick/godirwalk](https://github.com/karrick/godirwalk)

You can easily ignore files and directories by using the **ignore** config property.

Below is a [sample configuration loaded in the Docker Image](https://github.com/adrian-gheorghe/moni/blob/master/sample.docker.config.yml) :

```yaml
general:
  # Should moni keep running and execute periodically  
  periodic: true
  # If periodic is true, what interval should moni run at? Interval value is in seconds
  interval: 3600
  # Tree is stored as a json to the following path
  tree_store: /app/output.json
  # Path to parse
  path: /var/www/html
  # Command that should run if the tree is identical to the previous one
  command_success: "echo SUCCESS"
  # 
  command_failure: "echo FAILURE"
log:
  # Log path for moni. os path or stdout accepted
  log_path: stdout
  # Memory log options are only for development use. Please keep memory_log value to false
  memory_log_path: ./memory.log
  memory_log: false
algorithm:
  # Algorithm options are:
  # - FlatTreeWalk (manual recursive treewalk)  
  # - GoDirTreeWalk - walk algorithm developed by karrick - https://github.com/karrick/godirwalk
  name: FlatTreeWalk
  processor: ObjectProcessor
  # List of directory / file names moni should ignore
  ignore:
    - ".git"
    - ".idea"
    - ".vscode"
    - ".DS_Store"
    - "node_modules"
    - "uploads"

```
Here is an example docker-compose.yml file to get moni to monitor your current directory.
```yaml
version: '3.3'  
services:  
  moni:  
    image: adighe/moni:0.4.0  
    volumes:  
      - ./:/var/www/html  
    environment:  
      CONFIG-PATH: /app/config.yml
```

The docker image contains the sample docker configuration that you can find here: [https://github.com/adrian-gheorghe/moni/blob/master/sample.docker.config.yml](https://github.com/adrian-gheorghe/moni/blob/master/sample.docker.config.yml) . The configuration can easily be overwritten by mounting your own as a volume and setting the CONFIG-PATH env variable to the new path. For example:

```yaml
version: '3.3'  
services:  
  moni:  
    image: adighe/moni:0.4.0  
    volumes:  
      - /Users/adriangheorghe/go/src:/var/www/html  
      - ./config.yml:/app/config.yml  
    environment:  
      CONFIG-PATH: /app/config.yml
```