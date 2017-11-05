# 1. git rebase work flow

1. checkout public branch **ParentBranch**: 

	`git checkout ParentBranch`

	update ParentBranch, or add, modified on ParentBranch. 

	commit your changes.

2. checkout personal private working branch
	
	PersonalBranch which's parent branch is ParentBranch: 

	`git checkout PersonalBranch`

3. add, modified files on working branch PersonalBranch.
4. commit changes: git commit -m 'do something cool'
5. rebase working branch: git rebase -i ParentBranch PersonalBranch
6. checkout ParentBranch: git checkout ParentBranch
7. merge working branch `PersonalBranch` into public branch `ParentBranch`: git merge -v PersonalBranch

------

# 2. code example

1. step 1: pull remote public branch

```bash
git clone https://github.com/xechoz/demo.git 

git checkout -b feature-hello # create working branch
git checkout master # back to public branch

# add some files on public branch
echo "public branch file" >> public.md

git add public.md

# Note: this commit is on public branch, but not private working branch feature-hello
git commit -m 'add public file' 
```

2. step 2: checkout working branch

```bash
git checkout -b feature-hello

# or if feature-hello is exist
git checkout feature-hello
```

3. step 3 : add, modified files

```
echo "hello" >> public.md

# add more files
echo "hello file" >> hello.md
```
4. step 4: commit changes

```
git add public.md
git commit -m 'update public file'


git add hello.md
git commit -m 'hello world'
```

5. step 5: rebase working branch

```bash
# Note: this make feature-hello update-to-date with master
git rebase -i master feature-hello 
```

6. step 6: checkout back to public branch

```bash
git checkout master
```

7. step 7: merge commits on private working branch

```
# because feature-hello is update-to-date with master, so there is no redundant 'merge-commit' here
git merge -v feature-hello
```

# 3. 正确的使用习惯

## 3.1 什么时候使用 rebase， merge ？

个人分支首选 rebase， 公共分支使用 merge。

对非 upstream 的分支使用 merge 优先于 rebase， 如 `git merge -v origin/feature dev`, `git rebase -i origin/dev dev`

## 3.2 举例

1. remote origin branch: foo
2. user A branch: foo_a -> origin/foo
3. user B branch: foo_b -> origin/foo

假如 A, B 的最近共同 commit 是 commit-1;

User A's local commits is `commit-a1, commit-a2`, and pushed to remote. 

Now, remote commits become `commit-1 -> commit-a1 -> commit-a2`;


And user B commits to local : `commit-b1`. B's commit history becomes: `commit-1 -> commit-b1`

Now comes the key point, should user B run  `git merge origin/foo fooB` or `git rebase origin/foo fooB ` ?

Both of the commands can **merge(not git's merge)** A's work into B's, but with `git merge origin/foo fooB` a redundant **merge commit**  

like `Merge remote-tracking branch 'origin/foo' into fooB` or some what will be added to the commit history. 

And with `rebase`, B only get what he what, with all A's work without an additional commit.

with ``git merge origin/foo fooB``: commit-1 -> commit-b1 -> commit-a1 -> commit-a2 -> redundant merge commit

with `git merge origin/foo fooB`: commit-1 -> commit-a1 -> commit-a2 -> commit-b1 

我习惯在使用 rebase 合并远程 `origin upstream` 分支代码到个人本地分支， 比如  dev_local <--> origin/dev , 我会用 `git rebase origin/dev dev_local`合并代码， 

对于 **upstream 对应的分支**， 使用merge， 例如要合并 feature 分支到 dev_local, 使用 `git merge origin/feature dev_local` 而不是 `git rebase origin/feature dev_local`;

而对公共分支的代码合并 我用 merge， 比如 develop 分支的代码 合并到 master。
  
# 参考

1. [Merging vs. Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)