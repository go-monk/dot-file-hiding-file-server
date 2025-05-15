**The Scene:**
A quiet terminal room in the Temple of Code. A young apprentice, Novus, approaches the weathered yet kind mentor, Vetus. A scroll of [Go code](https://pkg.go.dev/net/http#example-FileServer-DotFileHiding) clutched tightly in hand.

---

**Novus:**
I found this scroll buried in the library. It seems to guard against dot files, but its ways are strange. Will you help me decipher it?

**Vetus (smiling):**
Let us journey through the scroll together.

**Novus (reading aloud):**

```go
package main

import (
	"io"
	"io/fs"
	"log"
	"net/http"
	"strings"
)
```

**Vetus:**
A fine start. All Go code consists of one or more packages - this scroll being a part of the executable main package. It calls upon the powers of I/O, file systems, logging, HTTP... and the strings of old. This tale is clearly one of web servers and files.

Now, what is the first function inscribed?

**Novus:**
It is thus:

```go
func containsDotFile(name string) bool {
	parts := strings.Split(name, "/")
	for _, part := range parts {
		if strings.HasPrefix(part, ".") {
			return true
		}
	}
	return false
}
```

**Vetus:**
A seeker of secrets, that one. containsDotFile walks a path of slashes, peering into each segment of a file path. If any begins with a dot — the mark of the hidden — it raises the banner of truth: “Yes, a dot file lies here!”

**Novus:**
So it recognizes hidden things merely by their names?

**Vetus:**
Indeed. In Unix lands, a dot at the front renders a file invisible to casual eyes. Wise web servers must tread carefully.

**Novus (continues reading):**
```go
type dotFileHidingFile struct {
	http.File
}
```

This part puzzles me. Why is http.File interface inside this custom type? It’s not given a name...

**Vetus (chuckling):**
You’ve discovered embedding, a quiet power in Go. In this case it's interface embedding and it resembles struct embedding. By wrapping http.File anonymously in the new type dotFileHidingFile, we say: “We are like http.File, but we carry more.”

**Novus:**
So all the methods of http.File still apply?

**Vetus (eyes twinkling):**
Yes. The new type inherits the interface of its embedded field. But watch this next spell carefully:

```go
func (f dotFileHidingFile) Readdir(n int) (fis []fs.FileInfo, err error) {
	files, err := f.File.Readdir(n)
	for _, file := range files {
		if !strings.HasPrefix(file.Name(), ".") {
			fis = append(fis, file)
		}
	}
	if err == nil && n > 0 && len(fis) == 0 {
		err = io.EOF
	}
	return
}
```

Here, we override the Readdir method — just this one — to hide files that begin with dots. A clever enchantment. It calls upon the original list of files, then prunes away the ones starting with a dot. If the request was to read, say, five files, but all were hidden... it gently returns an end-of-file signal. Polite, but firm.

**Novus:**
So even a directory full of secrets appears empty to the unaware?

**Vetus:**
Just so. What of the filesystem itself?

**Novus:**
We come to this construct:

```go
type dotFileHidingFileSystem struct {
	http.FileSystem
}
```

**Vetus:**
A filesystem that wears a mask. It borrows the shape of the true one, but speaks with its own rules. What does it do when asked to open a path?

**Novus:**

```go
func (fsys dotFileHidingFileSystem) Open(name string) (http.File, error) {
	if containsDotFile(name) {
		return nil, fs.ErrPermission
	}
	file, err := fsys.FileSystem.Open(name)
	if err != nil {
		return nil, err
	}
	return dotFileHidingFile{file}, err
}
```

**Vetus:**
Just as I thought. When a seeker asks for a file with dots in its ancestry, like `localhost:8080/.config` or even `localhost:8080/tmp/.config` the filesystem refuses, saying: “You have no permission.” A 403, in HTTP tongue.

But if the name is clean, it hands back a cloaked file — one that hides dotfiles from view.

**Novus:**
I see now. A guardian at every gate, from root to leaf.

And then... the final invocation.

```go
func main() {
	fsys := dotFileHidingFileSystem{http.Dir(".")}
	http.Handle("/", http.FileServer(fsys))
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Vetus:**
A file server, humble but vigilant. It shares the contents of the current directory through the profane port 8080. Yet not all may pass. Hidden files — passwords, config incantations, or ancient notes — are kept from prying eyes.

**Novus (bowing):**
Thank you, Vetus. I understand now. This code protects the temple's secret scrolls from idle wanderers.
