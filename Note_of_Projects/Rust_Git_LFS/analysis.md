# LFS

> Git Large File Storage (LFS) replaces large files such as audio samples, videos, datasets, and graphics with **text pointers** inside Git, while storing the file contents on a remote server like GitHub.com or GitHub Enterprise.  
![LFS](./lfs_demonstration.gif)

### Features:

- Large file versioning
- More repository space
- Faster cloning and fetching
- Same Git workflow
- Same access controls and permissions

### API:

- The Git LFS client uses an HTTPS server to coordinate fetching and storing large binary objects separately from a Git server. The basic process the client goes through looks like this:
    1. Discover the LFS Server to use.
        - Git LFS will attempt to use **Git remote** to determin the LFS server. Can be cinfigured with a custom LFS server if the Git remote doesn't support one, or to use a separate one.
        - By default, append `.git/info/lfs` to the end of a Git remote url to build the **LFS server** URL it will use:  
        Git Remote: `https://git-server.com/foo/bar`  
        LFS Server: `https://git-server.com/foo/bar.git/info/lfs`
        - If Git can't guess the LFS server, or the `git-lfs-authenticate` command is not used, then specify the LFS server using `Git config`.
            - Regardless of Git remote: set `lfs.url` to set the LFS server.
                ```
                $ git config lfs.url https://lfs-server.com/foo/bar
                ```
            - For a specific remote only: set `remote.{name}.lfsurl` to set the LFS server.
                ```
                $ git config remote.dev.lfsurl http://lfs-server.dev/foo/bar
                $ git lfs env
                --------------
                Endpoint=https://git-server.coom/foo/bar.git/info/lfs (auth=none)
                Endpoint (dev)=http://lfs-server.dev/foo/bar (auth=none)
                ```
            - Git LFS will also read these settings from a `.lfsconfig` file in the root of the repository (precedence of this behaviour?). **This lets anyone commits it to the repository so that all users can use it, if that's the expected behaviour.**
    2. Apply Authentication.  
    The Git LFS API uses HTTP Basic Authentication to authorize requests. Therefore, HTTPS is strongly encouraged. The credentials can come from the following places:
        - SSH: If Git LFS detects an `SSH` remote, it will run the `git-lfs-authenticate` command. This allows supporting Git servers to give the Git LFS client alternative authentication so the user does not hav to setup a `git credential helper`(?). Which runs `$ ssh [{user}@]{server} git-lfs-authenticate {path} {operation}`. The `user`, `server`, and `path` properties are taken from the SSH remote. The `opration` can either be "download" or "upload". The SSH command can be tweaked with the `GIT_SSH` or `GIT_SSH_COMMAND` environment variables. The output for successful commands is JSON (useRON in rust?), and matches the schema as an `action` in a Batch API response. Git LFS will dump the STDERR from the `ssh` command if it returns a non-zero exit code.  
        - Git Credentials: Git provides a `credential` command for storing and retrieving credentials through a customizable credential helper. By default, it associates the credentials with a domain. Enable `credential.useHttpPath` so different repository paths have different credentials. And a basic `credential cacher` that stores passwords in memory. (Relate to the other project)
        - Specified in URL: Hardcode credentials into the git remote or LFS url properties in git config. **This is not recommended for security reasons because it relies on the credentials living in the local git config.** (Can be maliciouslly redirected?)
            ```
            $ git remote add origin https://user:password@git-server.com/foo/bar.git
            ```
    3. Make the request.
        - Batch API: used to request the ability to transfer LFS objects with the LFS server.
            - Batch URL is built by adding `/objects/batch` to the LFS server URL.  
            Git remote: https://git-server.com/foo/bar  
            LFS server: https://git-server.com/foo/bar.git/info/lfs  
            Batch API:  https://git-server.com/foo/bar.git/info/lfs/objects/batch
            - Batch API requests use the **POST verb** (?), and require the following HTTP headers. The request and response bodies are JSON (replace with RON?).
                ```
                Accept: application/vnd.git-lfs+json
                Content-Type: application/vnd.git-lfs+json
                ```
            - The client sends the following information to the Batch endpoint to transfer some objects.
                - `operation`: Should be download or upload.
                - `transfers`: An optional Array of String identifiers for transfer adapters that the client has configured. If omitted, the basic transfer adapter MUST be assumed by the server.
                - `ref`: Optional object describing the server ref that the objects belong to. Note: Added in v2.4.
                - `name`: Fully-qualified server refspec.
                - `objects`: An Array of objects to download.
                - `oid`: String OID of the LFS object.
                - `size`: Integer byte size of the LFS object. Must be at least zero.
                - `hash_algo`: The hash algorithm used to name Git LFS objects. Optional; defaults to sha256 if not specified.
        - Transfer adapter: currently use the **basic**.
        - File locking API: used to create, list, and delete locks, as well as verify that locks are respected in Git pushes.

