# curlbomb

curlbomb is a personal HTTP(s) server for serving one-time-use shell scripts.

You know all those docs for the cool and hip software projects that start out by telling you to install their software in one line, like this?

```bash
curl http://example.com/install.sh | bash
```

I call that a curl bomb... I don't know if anyone else does.

_convenient_ as hell, but a security and trustability _nightmare_. Especially since installers usually require root access, do you trust a random file on the internet with direct access to your machine?

But I usually try to ask myself this question: is it possible to turn a _bad_ idea into a _good_ one, or at the very least a less-bad idea? Let's take a look.

curlbomb serves a single file (read from disk or stdin) via HTTP to the first client to request it, then it shuts down. A command is printed out that will construct the curl bomb the client needs to run, which includes a one-time-use passphrase (called a knock) that is required to download the resource. This command is copy/pasted (or typed) into another shell, on some other computer, which will download and run the script in one line.

curlbomb has optional (but recommended) integration with OpenSSL to secure communications. OpenSSH is supported as well, to make it easy to curlbomb from anywhere on the internet, to anywhere else, through a proxy server that you can forward the port through.

So does curlbomb measure up to making this a good idea? Decide for yourself:

Feature/Problem | Traditional curl bomb                                                                                                                                     | Using curlbomb
--------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Conveniece      | Yup, sure is.                                                                                                                                             | I think so.
Trust           | Is it even SSL? Do you know/trust the URL and it's author?                                                                                                | Self hosted server and SSL verifies connection
Security        | Even if you verify the script beforehand, [are you sure it hasn't changed?](https://www.idontplaydarts.com/2016/04/detecting-curl-pipe-bash-server-side/) | Self hosted script, you're in control of the contents.
Privacy         | Anyone who knows the URL can download/run. Cannot contain private information like passwords.                                                             | curlbomb requires a passphrase (knock) and only serves a file one time (by default.) Optionally gpg encrypt the contents of the script. Put sensitive data like SSH keys and passphrases into your script as necessary.
Repeatability   | Is the script going to stay at the same URL forever? Can you specify any parameters or at least a version number?                                         | It's your script, read whatever env vars you want. You can keep it checked into your own git repository and serve it from anywhere anytime.

curlbomb is well tested, but not intended for heavy automation work. There are better alternatives to choose from (saltstack, ansible, puppet, etc.) curlbomb can be used effectively in doing the front work for setting up these other tools, like copying SSH keys and installing packages.

For even more narrative on this topic, [you can read my blog](http://www.enigmacurry.com/2016/05/19/curlbomb-server-automation-for-the-wayfaring-hacker/).

## Install

curlbomb can be installed from the [Arch User Repository](https://aur.archlinux.org/packages/curlbomb/) (AUR):

```bash
pacaur -S curlbomb
```

Or from the [Python Package Index](https://pypi.python.org/pypi/curlbomb) (PyPI):

```bash
pip install curlbomb
```

### Dependencies

  - Python 3.5 (I haven't tested anything lower)

  - [Tornado](http://www.tornadoweb.org/)

  - [Requests](https://pypi.python.org/pypi/requests)

  - [psutil](https://pypi.python.org/pypi/psutil/)

  - OpenSSL (optional, if using --ssl)

  - OpenSSH (optional, if using --ssh)

  - GnuPG (optional, if using encrypted SSL cert or resources)

  - [python-notify2](https://pypi.python.org/pypi/notify2) (optional, for desktop notifications when using ping subcommand)

  - curl (on the client machine, preferably version >= 7.39.0, for --pinnedpubkey support)

## Example Use

Serve a script stored in a file:

```bash
curlbomb run /path/to/script
```

This outputs a curl command that you copy and paste into a shell on another computer:

```bash
KNOCK=nDnXXp8jkZKtbush bash <(curl -LSs http://192.0.2.100:48690)
```

Once pasted, the script is automatically downloaded and executed.

By default, the client must pass a KNOCK variable that is passed in the HTTP headers. This is for two reasons:

- It adds a factor of authentication. Requests without the knock are denied.
- It helps to prevent mistakes, as the knock parameter is randomly generated each time curlbomb is run and can only be used once. (See `-n 1`)

(Astute readers will notice that the KNOCK variable is being fed to the script that is being downloaded, not into the curl command. That's because it's really a curlbomb within a curlbomb. The first curl command downloads a script that includes a second curl command that _does_ require the KNOCK parameter. This nesting allows us to keep the client command as short as possible and hide some extra boilerplate. See `--unwrapped`.)

If you want just the curl, without the bomb, ie. you just want to grab the script without redirecting it to bash, use `--survey`. This is useful for testing the retrieval of scripts without running them.

You can pipe scripts directly into curlbomb:

```bash
echo "pacman --noconfirm -S openssh && systemctl start sshd" | curlbomb
```

Whenever you pipe data to curlbomb you can omit the `run` subcommand, it's assumed that you want to run a script from stdin.

This works in shell scripts too:

```bash
cat <<EOF | curlbomb
#!/bin/bash
echo "I'm a script output from another script on another computer"
EOF
```

Or type it interactively:

```bash
$ curlbomb run -
pkg instll sqlite3
echo "bad idea, I don't have spollcheck when I typ in the terminal"
```

(The single dash says to read from stdin, even when nothing is being piped. Ctrl-D ends the interactive input.)

The shebang line (#!) is interpreted and automatically changes the interpreter the client runs, the following example runs the script with python instead of the default bash:

```bash
cat <<EOF | curlbomb
#!/usr/bin/env python3
import this
print("Hello, from Python!")
EOF
```

curlbomb can also transfer files and directories with `put` and `get` subcommands:

```bash
# Recursively copy a directory
# (to whatever directory the client is run from):
curlbomb put ~/.ssh

# Recursively copy a remote directory to the server
# (to whatever directory the server is run from)
curlbomb get /var/log

# Recursively copy a directory
#  - Specifies the explicit remote destination directory.
#  - Environment vars in single quotes are evaluated on the remote end.
#  - Excludes some files you may want to keep private.
curlbomb put ~/.ssh '$HOME' --exclude='*rsa'
```

The `put` and `get` subcommands are just convenience wrappers for running tar on both ends of the curlbomb pipe. You _could_ achieve the same thing more generically:

```bash
# Copy a local directory to a client, the hard way:
tar cjh -C $HOME .ssh | curlbomb run -c "tar xjv -f"

# Copy a remote directory to the server, the hard way:
echo "tar cjh -C /var log" | curlbomb -l --client-quiet | tar xjv
```

The first example has a `run -c` parameter that tells the client that we want to interpret the data as being a tar archive rather than a script. The second example has a `-l` parameter that will output the data received to stdout, in this case piped directly into tar.

### SSH tunnel

By default, curlbomb constructs URLs with the IP address of the local machine. This usually means that clients on another network will be unable to retrieve anything from curlbomb, unless you have a port opened up through your firewall (and appropriate use of the `--domain` and `--port` arguments.) As an alternative, curlbomb can be tunneled through SSH to another host that has the proper port open. For instance:

```bash
echo "apt-get install salt-minion" | curlbomb --ssh user@example.com:8080
```

The above command connects to example.com over SSH (port 22 by default) and forwards the curlbomb server port to example.com:8080\. The URL that curlbomb prints out will now use the domain name of the ssh server, instead of the local IP address. The SSH tunnel is left open for as long as the curlbomb server remains running. Any user directly on the example.com host will be able to fetch the resource from localhost:8080\. However, by default, SSH does not open this up to the rest of the world. If you want any client to be able to connect to example.com:8080 you will need to modify the sshd_config of the server to allow GatewayPorts:

```bash
# Put this in your /etc/ssh/sshd_config and restart your ssh service:
GatewayPorts clientspecified
```

### TLS / SSL security

For extra security, you can enable TLS with `--ssl`:

```bash
echo "PASSWORD=hunter2 run_my_server" | curlbomb --ssl /path/to/cert.pem
```

The example above is passing a bit of secure information; a password. Even without TLS, curlbomb secures access with a knock parameter. For many use-cases, this is sufficient to secure it, as curlbombs are short lived and can only be retrieved one time (`-n 1`). However, the connection itself might be spied on (or even modified!) through traffic analysis at your ISP or any other router your connection flows through. Using TLS makes sure this doesn't happen.

Note that when the `--ssl` parameter is combined with the `--ssh` parameter, the SSL certificate should be generated for the host running the SSH server rather than the one running curlbomb. To prevent having to store the SSL certificate in plain text on your local machine, the file may be optionally PGP encrypted and curlbomb will decrypt it only when necessary.

You can also specify the SSL certificate path as a single `-`. In this case, a new self-signed certificate will be generated and used for this session only.

`--pin` can be used to extract the SSL certificate fingerprint and directly provide it to the client curl command (requires curl >=7.39). This avoids having to trust the client's CA root certificate store, and trusts your certificate explicitly. When generating a self-signed certificate with `--ssl`, the `--pin` option is turned on automatically. Pinning adds some extra security benefits, but makes the client command you have to paste/type much longer than it usually is, for example:

```bash
$ echo "whoami" | curlbomb --ssl -
WARNING:curlbomb.server:No SSL certificate provided, creating a new self-signed certificate for this session
Paste this command on the client:

  KNOCK=bbxfOV1ToDVhJjAl bash <(curl -LSs -k --pinnedpubkey 'sha256//RSkhZc2Qw/j8AxHMLUzipRpegEK9I0BlX7J1I5bcg0Y=' https://192.0.2.100:39817)
```

`--pin` is a different kind of trust model then using a certificate signed by a CA. When you use `--pin` you are completely bypassing the root CA certificate store of the client machine and instructing it to trust your certificate explicitly. This mitigates many man-in-the-middle type attacks that can happen with TLS, but you still need to take care that the client command is not modified or eavesdropped before being pasted into the client.

### Aliases

By now the curlbomb command might be getting quite long. Once you've encrypted and stored your SSL certificate, and setup your SSH server, create an alias for ease of use, for example:

```bash
alias cb=curlbomb --ssl ~/.curlbomb/curlbomb.pem.gpg --ssh user@example.com:22:8080
```

There's a few more examples in

[EXAMPLES.md](/EXAMPLES.md)

## Command Line Args

```bash
curlbomb [-h] [-n N] [-p PORT] [-d host[:port]] [-w] [-l] [-q] [-v]
         [--ssh SSH_FORWARD] [--ssl CERTIFICATE] [--pin] [-e]
         [--encrypt-to GPG_ID] [--passphrase] [--survey] [--unwrapped]
         [--client-logging] [--client-quiet] [--mime-type MIME_TYPE]
         [--pipe] [--disable-knock] [--knock KNOCK] [--version]
         {run,put,get,ping,ssh-copy-id, share} ...
```

curlbomb has a few subcommands:

- `run` - run a shell script
- `put` - copy local files/directories to remote system
- `get` - copy remote files/directories to local system
- `ping` - wait for a client to finish a task, with optional notification command
- `ssh-copy-id` - copy SSH public keys to the remote authorized_keys file
- `share` - share a file with the knock embedded in the URL. Serves file an unlimited number of times unless `-n` is specified.

If no subcommand is specified, and there is data being piped to stdin, then the `run` subcommand is used implicitly.

### The following arguments apply to all subcommands:

#### These arguments modify the server:

`-n N, --num-gets N` The maximum number of times the script may be fetched by clients, defaulting to 1\. Increasing this may be useful in certain circumstances, but please note that the same knock parameter is used for all requests so this is inherently less secure than the default. Setting this to 0 will allow the resource to be downloaded an unlimited number of times.

`-p PORT` The local TCP port number to use.

`--ssh SSH_FORWARD` Forwards the curlbomb server to a remote port of another computer through SSH. This is useful to serve curlbombs to clients on another network without opening up any ports to the machine running curlbomb. The syntax for SSH_FORWARD is [user@]host[:ssh_port][:http_port]. The SSH server must have the GatewayPorts setting turned on to allow remote clients to connect to this port. See sshd_config(5).

`--ssl CERTIFICATE` Run the HTTP server with TLS encryption. Provide the full path to your SSL certificate, which may be PGP encrypted. The file should contain the entire certificate chain, including the CA certificate, if any. If the SSL certificate path is specified as `-`, a temporary self-signed certificate will be generated for the current curlbomb session and `--pin` will be turned on implicitly.

`-e, --encrypt` Encrypt the resource with gpg before serving it to the client. A randomly generated symmetric passphrase will be printed below the client command on the server. This passphrase must be input on the client. You can specify the passphrase to use interactively with `--passphrase`. You can use public key encryption if you use `--encrypt-to`

`--encrypt-to GPG_ID` Encrypt the resource with the given gpg identity. Can be specified multiple times to encrypt to multiple recipients.

`--passphrase` Encrypt the resource with a passphrase interactively asked on server start.

`--mime-type MIME_TYPE` The mime-type header to send, by default "text/plain"

`--disable-knock` Don't require a X-knock HTTP header from the client. Normally, curlbombs are one-time-use and meant to be copy-pasted from terminal to terminal. If you're embedding into a script, you may not know the knock parameter ahead of time and so this disables that. This is inherently less secure than the default.

`--knock` Specify the knock string to use rather than generating a random one.

#### These arguments modify the client command:

`-d host[:port], --domain host[:port]` Specify the domain name and port that is displayed in the URL of the client command. This does not change where the resource is actually located, use --port or --ssh for that. This is useful if you are setting up your own port forwards and need to show an external URL.

`-w, --wget` Print wget syntax rather than curl syntax. Useful in the case where the client doesn't have curl installed. Not compatible with `--log--posts` or the `put` and `get` subcommands. :(

`--survey` Only print the curl (or wget) command. Don't redirect to a shell command. Useful for testing script retrieval without running them.

`--unwrapped` output the full curlbomb command, including all the boilerplate that curlbomb normally wraps inside of a nested curlbomb.

This parameter is useful when you want to source variables into your current shell:

```bash
echo "export PATH=/asdf/bin:$PATH" | curlbomb -c source --unwrapped
```

Without the --unwrapped option, the client command will not run the source command directly, but instead a bash script with a source inside it. This won't work for sourcing environment variables in your shell, so use --unwrapped when you want to use source.

`--pin` (requires curl>=7.39.0) Pin the SSL certificate fingerprint into the client curl command. This is used to bypass the root CA store of the client machine, and to tell it exactly what the server's SSL certificate looks like. This is useful for mitigating man-in-the-middle attacks, as well as when using self-signed certificates. This makes the client command quite a bit longer than usual.

`--pipe` construct the client command with pipe syntax rather than process substitution. curlbomb usually constructs client commands that look like `bash <(curl ...)`, but this doesn't work for some scripts that check for interactive input. In these cases `--pipe` will transform the client command into the more traditional `curl ... | bash`.

#### These arguments modify the CLI interaction:

`-l, --log-posts` Log the client stdout to the server stdout.

`-q, --quiet` Be more quiet. Don't print the client curlbomb command.

`-v, --verbose` Be more verbose. Turns off `--quiet`, enables `--log-posts`, and enables INFO level logging within curlbomb.

`--log LOG_FILE` Log messages to a file rather than stdout

`--client-logging` Logs all client output locally on the client to a file called curlbomb.log

`--client-quiet` Quiets the output on the client

`--version` Print the curlbomb version

### Run subcommand

```bash
curlbomb run [-c COMMAND] [--hash SHA256] [--signature FILE_OR_URL [GPG_ID ...]] [SCRIPT]
```

Runs a shell script on the remote client.

`-c COMMAND` Set the name of the command that the curlbomb is run with on the client. By default, this is autodected from the first line of the script, called the shebang (#!). If none can be detected, and one is not provided by this setting, the fallback of "bash" is used. Note that curlbomb will still wrap your script inside of bash, even with `-c` specified, so the client command will still show it as running in bash. The command you specified is put into the wrapped script. See `--unwrapped` to change this behaviour.

`--hash SHA256` Specify the expected SHA-256 hash of the script and the server will verify that it actually has that hash before the server starts. This is useful if you are pipeing a script from someplace outside of your control, like from the network. This prevents the server from serving a script other than the version you were expecting.

`--signature FILE_OR_URL [GPG_ID ...]` Specify the file or URL containing the GPG signature for the script. Optionally specify a list of GPG key identifiers that are allowed to sign the script. If no GPG_ID is specified, any valid signature from your keyring is accepted. The script will be checked for a valid signature before the server starts.

`SCRIPT` The script or other resource to serve via curlbomb. You can also leave this blank (or specify '-') and the resource will be read from stdin.

Note that the run subcommand is implied if you are pipeing data to curlbomb. For instance, this command is assumed that the run command is desired even if not explicitly used:

```bash
echo "./run_server.sh" | curlbomb
```

Which is equivalent to:

```bash
echo "./run_server.sh" | curlbomb run -
```

### Put subcommand

```bash
curlbomb put [--exclude=PATTERN] SOURCE [DEST]
```

Copies file(s) from the local SOURCE path to the remote DEST path. If a directory is specified, all child paths will be copied recursively.

If DEST path is unspecified, files/directories will be copied to the working directory of wherever the client was run.

Exclude patterns can be specified like tar(1)

### Get subcommand

```bash
curlbomb get [--exclude=PATTERN] SOURCE [DEST]
```

Copies file(s) from the remote SOURCE path to the local DEST path. If a directory is specified, all child paths will be copied recursively.

If DEST path is unspecified, files/directories will be copied to the working directory of wherever curlbomb was run.

Exclude patterns can be specified like tar(1)

### Ping subcommand

```bash
curlbomb ping [-m MESSAGE] [-r RETURN_CODE] [--return-success]
              [-c COMMAND] [-n]
```

Serves an empty body resource for the purposes of pinging the server when the client has finished some task.

`-m` sets the message the client will respond with.

`-r` sets the return code the client will respond with. This is used as the main curlbomb return code on the server as well. If `-n` > 1, the last non-zero return code received is used instead, defaulting to 0.

`--return-success` Always return 0, regardless of the return code(s) received.

`-c COMMAND` Run this command for each ping received. You can use the following placeholders to format ping data: {return_code} and {message}. {message} is replaced surrounded by quotes, so no need to do that again in your command.

### ssh-copy-id subcommand

```bash
curlbomb ssh-copy-id IDENTITY
```

Copies the given OpenSSH identity file (eg. ~/.ssh/id_rsa.pub) into the remote ~/.ssh/authorized_keys file.

Of course OpenSSH comes with it's own ssh-copy-id program, but I've never really understood the usefulness of it. The idea of using SSH keys is to not use crappy passwords, right? But the OpenSSH version of ssh-copy-id requires password authentication (at least temporarily during the setup process.) So you either have to edit your sshd_config, turn on `PasswordAuthentication`, and restart the service, or you resign yourself to run an insecure sshd all the time. `curlbomb ssh-copy-id` is easier and works in more situations.

Another difference in this version is that you must explicity specify the identity file, whereas the OpenSSH version does some automatic determination of which key to install. Especially if you maintain several ssh identities, being explicit seems the more sane thing to do than try to save some keystrokes and inevitably install the wrong key on the server.

### Share subcommand

```bash
curlbomb share FILE
```

Shares the given file via a URL which includes the knock as a URL parameter. By default, share will serve the file an unlimited number of times, but can be restricted to a single get by setting `-n 1`.

It would be nice to be able to log IP address information of the clients downloading, but this seems to not be possible when using SSL through an SSH forward (which is my normal use-case), so this has not been implemented. Perhaps some panopticlick style info could be interesting at some point.

[comment]: # "end feature table"
