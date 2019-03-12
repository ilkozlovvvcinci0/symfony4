## Storing Private Files

Here's the tricky part: we can't just go into `UploaderHelper` and use the Flysystem
filesystem like we did before to save the uploaded file... because *that* writes
everything into the `public/uploads/` directory. If we need to check security before
letting a user download a file, then it *can't* live in the `public/` directory.

And *that* means that we need a *second* Flysystem filesystem: one that can store
things somewhere *outside* the `public/` directory. Side note: it *is* possible
to solve the "private" uploads problem with just *one* filesystem using signed URLs,
and we'll talk about it later when we move to S3.

## Creating a Private Filesystem

But for now, a great solution is to create a *private* filesystem. Open the
`config/packages/oneup_flysystem.yaml` file. Copy the `public_uploads_adapter`, paste
and call it `private_uploads_adapter`. You can store the files *anywhere*, as long
as it's not in `public/`. But, the `var/` directory is sort of meant of this type
of thing. So let's say: `var/uploads`. Oh, and I could re-use my `uploads_dir_name`
parameter here - but it won't give us any benefit - that parameter is really meant
to keep the upload directory and *public* path to assets in sync. But... these files
won't have a public path anyways - we'll make them downloadable in an entirely
different way.

Next, for filesystems, do the same thing: make a `privates_uploads_filesystem` that
will use the `private_uploads_adapter`.

Cool! Next, in `UploaderHelper`, were already passing the `$publicUploadFilesystem`
as an argument. We'll *also* need the private one. Before we add it here, go into
`services.yaml`. Remember, under `_defaults`, we're binding the
`$publicUploadFilesystem` argument to the public fileystem service. Let's do the
same for the private one. Call it `$privateUploadFilesystem` and change the service
id to point to the "private" one.

Now, copy that argument name and, in `UploaderHelper`, add a second argument:
`FilesystemInterface $privateUploadFilesystem`. Create a new property on top
called `$privateFilesystem` and set it below:
`$this->privateFilesystem = $privateUploadFilesystem`

## Re-using the Upload Logic

Ok, we're ready! Most of the logic in `uploadArticleImage()` should be reusable:
we're basically going to do the same thing... just through the *private* filesystem:
we need to figure out the filename and stream it through Flysystem. The only part
of this method that we *don't* need is the `$existingFilename`: we don't need to
delete any *existing* filename - we're not going to allow files to be "updated"
for a specific `ArticleReference`.

Let's refactor: copy all of this code down through the `fclose()` and, at the bottom.
create a new `private function` called `uploadFile()`. This will take in the
`File` object that we're uploading and we will also need to pass the directory name -
you'll see what this in a moment. And then a `bool $isPublic` flag so that this
method knows whether to store things in the public filesystem or private one. To
start, paste that exact logic and, at the bottom, `return $newFilename`.
Oh, and I should also probably add a return type.

Let's see... the first thing we need to do is handle this `$isPublic` argument. So
Let's say `$filesystem = $isPublic ?`, and if it *is* public, use `$this->filesystem`,
otherwise use  `$this->privateFilesystem`. Below, replace `$this->filesystem` with
`$filesystem`.

The other thing we need to update is the directory: it's hardcoded to `ARTICLE_IMAGE`.
Replace that with `$directory`: this is the directory inside the filesystem where
the file will be stored.

All done! Back up in `uploadArticleImage()`, re-select *all* that code we just copied,
delete it, and replace it with `$newFilename = $this->uploadFile()` passing the
`$file`, the directory - `self::ARTICLE_IMAGES` - and whether or not this file should
be public, which is `true`.

Let's do the same thing down in `uploadArticleReference`. Oh, but first, we need
to create another constant for the directory
`const ARTICLE_REFERENCE = 'article_reference`.

Back down, all we need is `return $this->uploadFile()`, with `$file`,
`self::ARTICLE_REFERENCE` and `false` so that it uses the *private* filesystem.

I think that's it! Let's test this puppy out! Move over and refresh to re-POST
the form. No error... but I have no idea if that worked - we're not rendering
anything yet. Check out the `var/` directory...
`var/uploads/article_reference/symfony-best-practices...`, we got it!

Of course, there's absolutely no way for *any* user to access this file, but we'll
handle that soon.

Next: unless we *really* trust our authors, we probably shouldn't let them upload
*any* file type. Let's tighten that up.