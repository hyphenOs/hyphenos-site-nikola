# Getting Started

To get started you need to run the following commands. These commands run on Ubuntu 22.04.

These commands will start and `activate` virtual environment inside the local directory and install `nikola` in that virtual environment.

```shell

$ python3 -m venv venv3
$ . venv3/bin/activate
$ pip install -r requirements.txt
$ nikola auto
```

Once you run these commands you may get output like - following

```shell

2024-03-19 12:01:35] INFO: auto: REBUILDING SITE
[2024-03-19 12:01:35] ERROR: auto: Rebuild failed
Traceback (most recent call last):
  File "/home/gabhijit/Work/hyphenOs/hyphenos-site-nikola/venv-nikola/lib/python3.8/site-packages/doit/dependency.py", line 155, in __init__
    self._dbm = ddbm.open(self.name, 'c')
  File "/home/gabhijit/.pyenv/versions/3.8.7/lib/python3.8/dbm/__init__.py", line 91, in open
    raise error[0]("db type is {0}, but the module is not "
dbm.error: db type is dbm.gnu, but the module is not available

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/home/gabhijit/Work/hyphenOs/hyphenos-site-nikola/venv-nikola/lib/python3.8/site-packages/doit/doit_cmd.py", line 294, in run
    return command.parse_execute(args)
  File "/home/gabhijit/Work/hyphenOs/hyphenos-site-nikola/venv-nikola/lib/python3.8/site-packages/doit/cmd_base.py", line 150, in parse_execute
    return self.execute(params, args)
  File "/home/gabhijit/Work/hyphenOs/hyphenos-site-nikola/venv-nikola/lib/python3.8/site-packages/doit/cmd_base.py", line 549, in execute
    self.dep_manager = Dependency(
  File "/home/gabhijit/Work/hyphenOs/hyphenos-site-nikola/venv-nikola/lib/python3.8/site-packages/doit/dependency.py", line 510, in __init__
    self.backend = db_class(backend_name, codec=codec_cls())
  File "/home/gabhijit/Work/hyphenOs/hyphenos-site-nikola/venv-nikola/lib/python3.8/site-packages/doit/dependency.py", line 170, in __init__
    raise DatabaseException(message)
doit.dependency.DatabaseException: db type is dbm.gnu, but the module is not available

[2024-03-19 12:01:35] INFO: auto: Serving on http://127.0.0.1:8000/ ...
```

Ignore the error above and open the link shown on the last line in your web browser. This should show the website running locally.


# What happens under the hood?

the `nikola auto` command builds the website using the configuration file `config.py` and the generated html (and other files are stored in the `output` directory). `nikola` tool then also runs a local webserver on port 8000, to which you can connect (http://127.0.0.1:8000) and check out the site locally.

To change the site behavior, one can make changes to `config.py` file. The contents of the actual site are present in the following directories

pages/  Site home page etc.
posts/  Blog posts
theme/  CSS files for themes etc.

