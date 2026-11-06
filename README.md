# desolate-lfs-scratch

A throwaway repository for trying Git LFS through desolate's push and pull
(issue #51, `docs/lfs.md` in isolated-devcontainers-suede). Its only content is
a devcontainer with the `git-lfs` feature, so the container can track and commit
LFS files itself. **Make the GitHub repository PRIVATE** -- a public one serves
LFS downloads without any credential, which would make the pull test prove
nothing.

Replace `<you>` with your GitHub user or org. Commands marked **Mac** run in a
Mac terminal; **editor** in the outer editor's terminal (or use the panel's
buttons); **container** inside the project's devcontainer.

## 0. Once per Mac

```bash
brew install git-lfs     # if you have not
git lfs install          # writes the filter into ~/.gitconfig; once per machine
```

## 1. Create the repository (Mac)

```bash
unzip desolate-lfs-scratch.zip && cd desolate-lfs-scratch
git init -b main
git add .
git commit -m "devcontainer with git-lfs"
gh repo create <you>/lfs-scratch --private --source=. --push
```

## 2. Register it with desolate and start it (Mac)

```bash
./cli.sh repo add <you>/lfs-scratch      # deploy key (write access) + clone
./cli.sh desolate <you>/lfs-scratch      # first start: checkout runs in the sandbox
```

## 3. Create LFS content from INSIDE desolate (container)

Open the project's devcontainer (its editor, or `./cli.sh exec <you>/lfs-scratch -- bash`).

```bash
git lfs version                              # the feature installed it
git config --show-origin filter.lfs.clean    # ...and the filter
git config user.name  "you"                  # if the container has none
git config user.email "you@example.com"

git lfs track '*.bin'                        # writes .gitattributes
head -c 200000 /dev/urandom > sample.bin
git add .gitattributes sample.bin
git commit -m "an LFS file, made in desolate"
git lfs ls-files -l                          # sample.bin, with its 64-hex oid
cat .git/lfs/objects/*/*/* | wc -c           # 200000: the object is here
git show HEAD:sample.bin                     # ~130 bytes: the pointer is what is committed
```

Write down the oid from `git lfs ls-files -l` (and `sha256sum sample.bin`: it is
the same value).

## 4. Review and push (editor, or the panel's Review & Push)

```bash
desolate --review <you>/lfs-scratch
#   A  sample.bin  (LFS, 195 KiB: uploaded with the push)
./cli.sh push <you>/lfs-scratch        # Mac; or `desolate --push <you>/lfs-scratch` in the editor
#   desolate: uploading 1 LFS object (195 KiB) to <you>/lfs-scratch's LFS storage, which counts against its owner's quota:
#   desolate:   sample.bin  195 KiB  <12 hex of the oid>
#   desolate: pushed main (...) to <you>/lfs-scratch
```

(It is the default branch, so it will ask for `--allow-default`: either pass it,
or do step 3 in a worktree -- `./cli.sh worktree add <you>/lfs-scratch t1`,
`./cli.sh desolate <you>/lfs-scratch --worktree t1`, then add `--worktree t1` to
the commands in this step.)

Then on github.com: open `sample.bin`. It should say **Stored with Git LFS**
and offer a download of 195 KiB, not show 130 bytes of text.

## 5. Read it back with an independent git-lfs (Mac)

```bash
cd .. && git clone git@github.com:<you>/lfs-scratch.git lfs-scratch-check && cd lfs-scratch-check
git lfs ls-files                       # "*" before the name: the content is here
sha256sum sample.bin                   # (shasum -a 256 on macOS) equals the oid from step 3
```

This is the check that desolate's upload produced a real LFS object, made by
a git-lfs that is not the one under test.

## 6. The pull direction

From the Mac clone, add a second LFS file and push with plain git-lfs:

```bash
head -c 300000 /dev/urandom > second.bin && git add second.bin && git commit -m "second, from the Mac" && git push
```

Then pull it into desolate:

```bash
./cli.sh pull <you>/lfs-scratch --fetch-only     # origin/* only: asks for no LFS credential, fetches no objects
./cli.sh pull <you>/lfs-scratch                  # fast-forwards; should end: ; fetched 1 LFS object
```

In the container, `second.bin` is a real 300000-byte file and `git status` is
clean. (If you pushed from a worktree, pull with `--worktree t1`.)

For the first-checkout case: `./cli.sh worktree add <you>/lfs-scratch t2` then
`./cli.sh desolate <you>/lfs-scratch --worktree t2`. Files whose objects are
already in the project's `.git/lfs` come out real; any others come out as
~130-byte pointers with a clean `git status`, and `./cli.sh pull ... --worktree t2`
fills them in.

## 7. A lost object must stop the push (container)

```bash
head -c 1000 /dev/urandom > lost.bin && git add lost.bin && git commit -m "lost"
oid=$(git lfs ls-files -l | awk '/lost.bin/{print $1}')
rm .git/lfs/objects/${oid:0:2}/${oid:2:2}/$oid
```

`./cli.sh push <you>/lfs-scratch` must fail, naming the object, and say the
branch was NOT pushed. `git log origin/main` on github.com does not show "lost".

## If something fails

| you see | it means |
| --- | --- |
| `names an endpoint that is not lfs.github.com/...: refusing it` | GitHub's credential `href` has a different shape than assumed (one regex, `lfsServerOf` in `transfer.ts`) |
| a git-lfs connection / TLS error in the output | the sandbox cannot reach `lfs.github.com`, `github-cloud.s3.amazonaws.com` or `github-cloud.githubusercontent.com`: the proxy's allowlist or CA. `./cli.sh proxy audit --project <you>/lfs-scratch` shows what it reached |
| `bad_permissions` on upload | the registered deploy key is read-only; `repo add` registers one with write access |
| `gave no LFS credential for ...` | the ssh `git-lfs-authenticate` call failed: GitHub's message follows |
| `files stay pointers` after a pull | the objects could not be fetched; git-lfs's own words are above that line |

Paste the whole output of the failing step back to whoever asked you to run this.

## Clean up

Delete the repository on GitHub (`gh repo delete <you>/lfs-scratch --yes`), then
`./cli.sh worktree remove` any worktrees and remove `/workspaces/<you>/lfs-scratch`.
