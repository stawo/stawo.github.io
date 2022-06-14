
- how to manage different environments (prod, dev, test, etc.) configurations
- how to keep it as simple as possible?
  - can we do it with only one file?
  - many configs are the same for all envs, only few of them are different (e.g., pointers to different servers/workspaces)
  - do not use any additional libraries, but only default ones (such as yaml)
- assuming we have defined env specific configuration, and we want to launch our code in a specific environment, we need to have a way for the code to select the right configuration values (e.g., if we are in dev, we only want the specific dev configs)
  - keep the logic of defining the proper config values (and maybe creating a clean config file with only the right values) separate from the main code. If we for example create a script/function that creates a proper config file based on the env we are in, then we can reuse this script also in Github workflows and other places.
  We also keep the main code cleaner, as this code will assume that we pass only the necessary configurations, and it doesn't have to worry about understanding which env it is running in.
- we create also a launch script that will have to steps:
  - execute the config creation script
  - execute the main code
- configurable
  - no fixed environments