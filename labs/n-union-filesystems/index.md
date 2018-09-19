## Union Filesystems

Use overlayfs to mount directories and full filesystems

### Overlay two directories

Create a directory to work within. In this example, the directory is `/mysandboxdir`.

```bash
mkdir /mysandboxdir && cd /mysandboxdir
```

Create directories representing the lower, upper, working, and destination overlay directories

```bash
mkdir work layer1 layer2 overlay
```

* layer1 represents the lower branch in the overlay. This directory is not read-only, but modifications to the overlay will not affect the lower branch.
* layer2 represents the upper branch in the overlay. This directory is also not read-only. Any modifications to the overlay will be represented here.
* work represents a location where files are prepared before becoming visible in the overlay directory
* overlay represents the final result of overlaying layer1 and layer2

Create a mount of type `overlay`, providing arguments defining the desired lower, upper, and overlay directory

```bash
sudo mount -t overlay -o lowerdir=./layer1,upperdir=./layer2,work=./work my-mount ./overlay
```

Touch a file in `./layer1`

```bash
touch ./layer1/foo
```

Notice that a file named `foo` now exists in `./overlay`.

Now, touch a file in `./overlay`.

```bash
touch ./overlay/bar
```

List files in the `./layer2` directory and you will now see a file called `bar` exists.

Let's make a change to the `./overlay/foo` file.

```bash
echo "hello" > ./overlay/foo
```

Now, inspect the contents of `./layer1/foo`. Notice anything missing?  Where did the write to `foo` go?

Lastly, we'll delete a file in `./overlay` that exists in `./layer1`.

```bash
echo "hi" > ./layer1/baz
rm ./overlay/baz
```

What are the contents of `./layer1`? Does `baz` exist there?
What about `./layer2`? Is `baz` there?

You probably found something like this in `./layer2`

```bash
$ ls -lah layer2
drwxr-xr-x 2 root root 4.0K Sep 19 04:45 .
drwxr-xr-x 6 root root 4.0K Sep 19 04:30 ..
c--------- 1 root root 0, 0 Sep 19 04:45 bar
-rw-r--r-- 1 root root    0 Sep 19 04:33 baz
-rw-r--r-- 1 root root   22 Sep 19 04:34 foo
```

When a file is deleted from `./overlay` that exists in `./layer1`, a _whiteout_ character device file is generated and stored in `./layer2`.
This instructs overlayfs to ignore `./layer1/baz` and ensure the whiteout file is not visible in `./overlay`.

### Overlaying Multiple Layers

Now, let's add a `./layer0` directory and recreate our overlayfs mount with layer0 as the lowest priority

```bash
mkdir ./layer0
sudo mount -t overlay -o lowerdir=./layer1:./layer0,upperdir=./layer2,work=./work none ./overlay
```

Notice the addition of `:./layer0` to the mount command.

Create files in `./layer0` and `./layer1` to see results in `./overlay`

```bash
echo "Hello from Layer 0" > ./layer0/qux
echo "Hello from Layer 1" > ./layer1/qux
cat ./overlay/qux
```

What's in `./overlay/qux`? Re-run the mount command with the lower layers reversed and inspect `./overlay/qux` again.

```bash
sudo mount -t overlay -o lowerdir=./layer0:./layer1,upperdir=./layer2,work=./work none ./overlay
```

### Docker and Overlayfs

```bash
TBD
pull and image and inspect the layers in /var/lib/docker/overlay2
```