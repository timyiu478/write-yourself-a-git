# write-yourself-a-git

## 3.1. The repository object

> A git repository is made of two things: a “**work tree**”, where the files meant to be in version control live, and a “**git directory**”, where Git stores its own data. In most cases, the worktree is a regular directory and the git directory is a child directory of the worktree, called `.git`.

Inside the `.git`, it contains:
- `.git/objects/` : the **object store**, which we’ll introduce [in the next section](https://wyag.thb.lt/#objects).
- `.git/refs/` the **reference store**, which we’ll discuss [a bit later](https://wyag.thb.lt/#cmd-show-ref). It contains two subdirectories, `heads` and `tags`.
- `.git/HEAD`, a reference to the current **HEAD** (more on that later!)
- `.git/config`, the repository’s **configuration** file.
- `.git/description`, holds a free-form description of this repository’s contents, for humans, and is rarely used.

## 3.3. The `rep_find` function

> Sometimes that root is the current directory, but it may also be a parent: your repository’s root may be in `~/Documents/MyProject`, but you may currently be working in `~/Documents/MyProject/src/tui/frames/mainview/`. The `repo_find()` function we’ll now create **will look for that root, starting at the current directory and recursing back to /**. To identify a path as a repository, it will **check for the presence of a .git directory**.

## 4.1. What are objects?

> Now, **what actually is a Git object?** At its core, Git is a “**content-addressed filesystem**”. That means that unlike regular filesystems, where the name of a file is arbitrary and unrelated to that file’s contents, **the names of files as stored by Git are mathematically derived from their contents**. This has a very important implication: if a single byte of, say, a text file, changes, its internal name will change, too. To put it simply: **you don’t _modify_ a file in git, you create a new file in a different location.** Objects are just that: **files in the git repository, whose paths are determined by their contents**.

Storage format:
>Before we start implementing the object storage system, we must understand their exact storage format. An object starts with a header that specifies its **type**: `blob`, `commit`, `tag` or `tree` (more on that in a second). This header is followed by an ASCII space (0x20), then the **size** of the object in bytes as an ASCII number, then null (0x00) (the null byte), then the **contents** of the object. The first 48 bytes of a commit object in Wyag’s repo look like this:

```
00000000  63 6f 6d 6d 69 74 20 31  30 38 36 00 74 72 65 65  |commit 1086.tree|
00000010  20 32 39 66 66 31 36 63  39 63 31 34 65 32 36 35  | 29ff16c9c14e265|
00000020  32 62 32 32 66 38 62 37  38 62 62 30 38 61 35 61  |2b22f8b78bb08a5a|
```

The objects (headers and contents) are stored **compressed** with `zlib`.

## 4.3. Reading Objects

> To read an object, we need to know its SHA-1 hash. We then **compute its path from this hash** (with the formula explained above: first two characters, then a directory delimiter `/`, then the remaining part) and look it up inside of the “objects” directory in the gitdir. That is, the path to `e673d1b7eaa0aa01b5bc2442d570a765bdaae751` is `.git/objects/e6/73d1b7eaa0aa01b5bc2442d570a765bdaae751`.  We then read that file as a binary file, and **decompress** it using `zlib`. From the decompressed data, we extract the two header components: the object type and its size. **From the type, we determine the actual class to use**. We convert the size to a Python integer, and check if it matches.

## 4.4. Writing Objects

> Writing an object is **reading it in reverse**: **we compute the hash, insert the header, zlib-compress everything and write the result in the correct location**. This really shouldn’t require much explanation, just notice that the hash is computed **after** the **header is added** (so it’s the hash of the object itself, uncompressed, not just its contents)

## 4.7. The hash-object command

> We will want to put our own data in our repositories, though. **hash-object is basically the opposite of cat-file**: it **reads a file, computes its hash as an object**, either storing it in the repository (if the -w flag is passed) or just printing its hash.

## 4.8. Aside: what about packfiles?

> What we’ve just implemented is called “**loose** **objects**”. Git has a second object storage mechanism called packfiles. Packfiles are much more efficient, but also much more complex, than loose objects. Simply put, a packfile is **a compilation of loose objects** (like a **tar**) but some are stored as **deltas** (as a **transformation of another object**). Packfiles are way too complex to be supported by wyag.

## 5.1. Parsing commits

The fields of the commit object:
- `tree` is a reference to a tree object, a type of object that we’ll see just next. A tree **maps blobs IDs to filesystem locations**, and describes a state of the work tree. Put simply, it is the actual content of the commit: **file contents, and where they go.**
- `parent` is a reference to the parent of this commit. It may be repeated: merge commits, for example, have multiple parents. It may also be absent: the very first commit in a repository obviously doesn’t have a parent.
- `author` and `committer` are separate, because the author of a commit is not necessarily the person who can commit it (This may not be obvious for GitHub users, but a lot of projects do Git through e-mail)
- `gpgsig` is the PGP signature of this object.

## Object Identity Rules

> 1. The first rule is that **the same name will always refer to the same object**. We’ve seen this one already, it’s just a consequence of the fact that **an object’s name is a hash of its contents.**
> 2. The second rule is subtly different: **the same object will always be referred by the same name**. This means that there shouldn’t be two equivalent objects under different names. This is why fields order matter: by modifying the order fields appear in a given commit, eg by putting the tree after the parent, we’d modify the SHA-1 hash of the commit, and we’d create two equivalent, but numerically distinct, commit objects.
>
> For example, when comparing trees, git will assume that two trees with different names are different — this is why we’ll have to make sure elements of the tree objects are properly sorted, so we don’t produce distinct but equivalent trees.

## 5.4. Anatomy of a commit

> First and foremost, we’ve been playing with commits, browsing and walking through commit objects, building a graph of commit history, without ever touching a single file in the worktree or a blob. We’ve done a lot with commits **_without considering their contents_**. This is important: work tree contents are just one part of a commit. But a commit is made of everything it holds: its contents, its authors, **also its parents**. If you remember that the ID (the SHA-1 hash) of a commit is computed from the whole commit object, you’ll understand what it means that commits are immutable: if you change the author, the parent commit or a single file, you’ve actually created a new, different object. Each and every commit is bound to its place and its relationship to the whole repository up to the very first commit. To put it otherwise, **a given commit ID not only identifies some file contents, but it also binds the commit to its whole history and to the whole repository.** 
> It’s also worth noting that from the point of view of a commit, **time somehow runs backwards**: we’re used to considering the history of a project from its humble beginnings as an evening distraction, starting with a few lines of code, some initial commits, and progressing to its present state (millions of lines of code, dozens of contributors, whatever). But **each commit is completely unaware of its future, it’s only linked to the past**. Commits have “memory”, but no premonition.

## 6.1 What is in a tree?

> Informally, a tree describes the content of the work tree, that it, it associates **blobs to paths**. It’s an array of three-element tuples made of a file mode, a path (relative to the worktree) and a SHA-1. A typical tree contents may look like this:

| Mode     | SHA-1                                      | Path         |
| -------- | ------------------------------------------ | ------------ |
| `100644` | `894a44cc066a027465cd26d634948d56d13af9af` | `.gitignore` |
| `100644` | `94a9ed024d3859793618152ea559a168bbcbb5e2` | `LICENSE`    |
| `100644` | `bab489c4f4600a38ce6dbfd652b90383a4aa3e45` | `README.md`  |
| `100644` | `6d208e47659a2a10f5f8640e0155d9276a2130a9` | `src`        |
| `040000` | `e7445b03aea61ec801b20d6ab62f076208b7d097` | `tests`      |
| `040000` | `d5ec863f17f3a2e92aa8f6b66ac18f7b09fd1b38` | `main.c`     |

> Mode is just the file’s mode, path is its location. The SHA-1 refers to either a blob or **another tree object**. If a blob, the path is a file, **if a tree, it’s directory**. To instantiate this tree in the filesystem, we would begin by loading the object associated to the first path (`.gitignore`) and check its type. Since it’s a blob, we’ll just create a file called `.gitignore` with this blob’s contents; and same for `LICENSE` and `README.md`. But the object associated with `src` is not a blob, but another tree: we’ll create the directory `src` and **repeat the same operation in that directory with the new tree**.

## The file Mode
The file mdoe is up to six bytes and is an octal representation of a file **mode**, stored in ASCII. For example, 100644 is encoded with byte values 49 (ASCII “1”), 48 (ASCII “0”), 48, 54, 52, 52. **The first two digits encode the file type (file, directory, symlink or submodule),** the last four the permissions.

## 7.1 What a ref is, and the show-ref command

Git references, or refs, are probably the most simple type of things git holds. They live in subdirectories of `.git/refs`, and are text files containing a hexadecimal representation of an object’s hash, encoded in ASCII. They’re actually as simple as this:

`6071c08bcb4757d8c89a30d9755d2466cef8c1de`

Refs can also refer to another reference, and thus only indirectly to an object, in which case they look like this:

`ref: refs/remotes/origin/master`

## 7.2 Tags as references

>　The most simple use of refs is tags. A tag is just **a user-defined name for an object**, often a commit. A very common use of tags is **identifying software releases**: You’ve just merged the last commit of, say, version 12.78.52 of your program, so your most recent commit (let’s call it `6071c08`) _is_ your version 12.78.52. To make this association explicit, all you have to do is:

```
git tag v12.78.52 6071c08
# the object hash ^here^^ is optional and defaults to HEAD.
```

## 7.5 What is a branch?

> So, what’s a branch? The answer is actually surprisingly simple, but it may also end up being simply surprising: **a branch is a reference to a commit**. You could even say that a branch is a kind of a name for a commit. In this regard, a branch is exactly the same thing as a tag. Tags are refs that live in `.git/refs/tags`, branches are refs that live in `.git/refs/heads`. There are, of course, differences between a branch and a tag: 
> 1. Branches are references to a **_commit_**, tags can refer to **any object**;
> 2. Most importantly, the branch ref is **updated at each commit**. This means that whenever you commit, Git actually does this:
    1. a new commit object is created, with the current branch’s (commit!) ID as its parent;
    2. the commit object is hashed and stored;
    3. the branch ref is updated to refer to the new commit’s hash.
That’s all.
But what about the **current** branch? It’s actually even easier. It’s a ref file outside of the `refs` hierarchy, in `.git/HEAD`, which is an **indirect** ref (that is, it is of the form `ref: path/to/other/ref`, and not a simple hash).

## What is the index file?

- This intermediate stage between **the last and the next commit** is called the **staging area**.
- It would seem natural to use a commit or tree object to represent the staging area, but Git actually and uses a completely different mechanism, in the form of what it calls the **index file**.
- You can thus consider the index file as **a three-way association list: not only paths with blobs, but also paths with actual filesystem entries**.
- Another important characteristic of the **index file** is that unlike a tree, it can represent **inconsistent states**, like a **merge conflict**, whereas a tree is always a complete, unambiguous representation.

The file changes during the commit:

1. When the repository is “clean”, the index file holds the exact same contents as the HEAD commit, plus metadata about the corresponding filesystem entries. For instance, it may contain something like:
    
    > There’s a file called `src/disp.c` whose contents are blob 797441c76e59e28794458b39b0f1eff4c85f4fa0. The real `src/disp.c` file, in the worktree, was created on 2023-07-15 15:28:29.168572151, and last modified 2023-07-15 15:28:29.1689427709. It is stored on device 65026, inode 8922881.
    
2. When you `git add` or `git rm`, the index file is modified accordingly. In the example above, if you modify `src/disp.c`, and `add` your changes, the index file will be updated with a new blob ID (**the blob itself will also be created in the process,** of course), and **the various file metadata** will be updated as well so `git status` knows when not to compare file contents.
3. When you `git commit` those changes, a new tree is produced from the index file, a new commit object is generated with that tree, branches are updated and we’re done.

The index file is **not a delta (a set of differences)!!**

## 8.5. The status command

`status` is more complex than `ls-files`, because it needs to compare the index with both **HEAD** and the **actual filesystem**. You call `git status` to know which files were added, removed or modified since the last commit, and which of these changes are actually staged, and will make it to the next commit. So `status` actually compares the `HEAD` with the staging area, and the staging area with the worktree. This is what its output looks like:

```
On branch master

# Compare with HEAD

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   write-yourself-a-git.org

# Compare with actual filesystem

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   write-yourself-a-git.org

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	org-html-themes/
	wl-copy
```

## 9. Staging area and index, part 2: staging and committing

> We have _almost_ everything we need for that, except for three last things:
> 1. We need commands to modify the index, so our commits aren’t just a copy of their parent. Those commands are `add` and `rm`.
> 2. These commands need to write the modified index back, since we commit _from the index_.
> 3. And obviously, we’ll need the `commit` function and its associated `wyag commit` command.

## 9.4 The commit command

> To do so, we first need to **convert the index into a tree object, generate and store the corresponding commit object, and update the HEAD branch to the new commit** (remember: a branch is just a ref to a commit).


# References
1. https://wyag.thb.lt/
