# Dqlite documentation

This repository holds the Markdown files of the Dqlite documentation.
These Markdown files are pushed to the [Discourse forum](https://discourse.dqlite.io/) and from there published to https://dqlite.io/docs.

## Documentation workflow

The forum post that rules it all is https://discourse.dqlite.io/t/dqlite-documentation/34, especially the Navigation table at the bottom.
To add a forum post to the docs, it must be added to this table.

Some information about how the transformation from Discourse to Doc output works is available here: [Creating Discourse based documentation pages](https://discourse.canonical.com/t/creating-discourse-based-documentation-pages/159)

In theory, the idea is that anybody (including community members) can just edit any documentation page.
There are several issues with this approach (see for example [this presentation](https://docs.google.com/presentation/d/1y8KlUbZxpWyJIOHiuH04P_BlTOUDzJORu2qkhiDUQ2s/edit?usp=sharing)).
Therefore, we use this GitHub repo for preparing and reviewing changes.

The general workflow for documentation updates is as follows:

1. Do your changes to the files in the Git repo (on a branch of your fork).
   * If you add pages, see [How to add pages to the documentation](#how-to-add-pages-to-the-documentation).
   * If you add images, see [How to add images](#how-to-add-images).
1. Open a pull request.
   GitHub will automatically run some automated checks.
1. When the pull request is reviewed, approved, and ready to go out (keep in mind that the update might be for a future release and should not be published yet!), publish it.
   See [How to publish documentation updates](#how-to-publish-documentation-updates).
1. After publishing, commit and push any changes that you had to do during publishing directly to the main branch of the repo.
   No need for a PR or review here.

## How to add pages to the documentation

New pages must be added to the navigation so that they show up in the documentation output.

1. Figure out where in the current navigation you want your page to show up.
1. Create the page or pages in the Git repo in the suitable place in the file structure.
   Try to keep the file names short (they are also used as URLs), but make sure to include all required information.
   Use hyphens, not underscores or spaces in the file names.
1. Add the new page or pages to the navigation table in the `index.md` file.
   * The first column in that table indicates the level.
   * The second column defines the URL under which the page will be available.
     Always use the same path and name as for the file name here, or the [publishing scripts](#how-to-publish-documentation-updates) will not work.
   * The third column links to the page.
     Specify the title that you want to use in the navigation (this title can be shorter than the page title; for example, you would usually leave out the "How to" here).
     You don't know the Discourse URL yet, so use `tbd` for now (that makes it easy to search and replace the URL during the publishing process).
1. Often, you will want to add some links to the new page or pages to other pages.
   For example, there might be links to all subpages on the parent page, or the explanation for a subject might link to all related how-to guides.
   Add all those links (using the full page title and `tbd` as URL).

## How to add images

A convenient tool for creating diagrams is https://www.diagrams.net/.

You can customise the tool by following the [README](https://github.com/anbox-cloud/discourse-docs/blob/main/images/README.md) in the Anbox Cloud doc repo.

**Important:** The brand guidelines and colour palette are about to change, so these instructions will need to be updated.

### How to publish images

Images should be published to the [Asset manager](https://manager.assets.ubuntu.com/).

Do this once:

1. Get in touch with the web team to request access to the asset manager.
   Also ask for an API key so that you can use the [upload-assets](https://github.com/canonical/canonicalwebteam.upload-assets) tool to upload images from the command line.
1. Install the [upload-assets](https://github.com/canonical/canonicalwebteam.upload-assets) tool.
1. Export the required environment variables (and put them in your `.bashrc` file):

       export UPLOAD_ASSETS_API_TOKEN=<api_token>
       export UPLOAD_ASSETS_API_DOMAIN=assets.ubuntu.com

Do this to publish your images:

1. Upload the images with the following command:

       upload-assets -t "Dqlite" images/<file_name(s)>

   Add more tags if it makes sense (as a comma-separated list).
1. Copy the URL that is returned for the image and update the Markdown file that includes the image with it.

## How to publish documentation updates

After merging a documentation pull request, the updates must be published to Discourse.

The simplest way of doing this is of course to just manually copy over the changes to the respective Discourse posts.
(Make sure to not copy from the GitHub diff though, because that includes random blank lines and messes up the Markdown.)

To publish more efficiently, use the publishing scripts (`publish.sh` for single files or `publish-pr.sh` for all changed files in a PR).

### Set up the required tools

Set up the `discedit` tool:

1. Get an API key for the Discourse instance (stgraber can create it).
1. Clone and build the [discedit](https://github.com/niemeyer/discedit) tool.
1. Add a `DISCEDIT` environment variable to your `.bashrc` file that points to the tool:

        export DISCEDIT=/home/what/ever/discedit
1. Create a `~/.discedit` file with the following content:
   ```
   forums:
     https://discourse.dqlite.io:
       username: <your_user_name>
       key: <your_API_key>
   ```

Set up a diff tool.
By default, the scripts use `meld`, but it should be possible to use a different diff tool by editing `scripts\publish.py`.
That is untested though.
If it works, we should probably introduce/use an environment variable to be able to easily configure the diff tool.

The easiest way is to just make sure you have `meld` installed (`sudo apt install meld`).

### Prepare for publishing

1. Refresh your main branch to make sure you have the latest content.
1. Check if there are any images that must be published first.
   If so, publish them (see [How to publish images](#how-to-publish-images)).
1. Check if there are any new pages that do not exist yet.
   If there are, do the following for each new page:
   1. Create a Discourse post.
      Make sure to use the correct categories.
      Enter the title and a dummy text (no need to copy and paste the content).
   1. Copy the URL of the new post.
   1. Open `index.md` and replace the `tbd` for the new page with the URL.
   1. Double-check that in the navigation table, the doc URL (2nd column) corresponds to the file path and name.
      Otherwise, the publishing will not work, so you'll need to change either the doc URL or the file name.
   1. Search through the full documentation for any other `tbd`s and replace them with the correct new URLs.

### Publish files

To publish all files in a PR, run `./publish-pr.sh <PR-number>`.

The script retrieves a list of all changed files and then opens them, one after the other, in the diff editor.
The diff shows what is currently on Discourse and what is on your local disk.
Merge the changes from your local disk into the file from Discourse and save.
When you close the diff tool, the changes are published to Discourse, and you can check them there.

Note that if a file cannot be found on the disk (for example, because the doc URL does not correspond to the file name) or if it does not exist on the forum (for example, because it is an announcement and not a doc post), the script will show an error, but this error can be hard to see between the other messages from the tool.
So make sure to always check the published result.

If you want to publish a single file (for example, to fix something you discovered after publishing the full PR), run `./publish.sh <file path>`.

### Check the output

Always check the output after publishing and make sure everything looks correct, new pages show up as expected and the links work.

If you find anything that is wrong, fix the respective file on your disk, publish again and confirm.

### Commit your changes

When the output looks good, commit all changes that you did to your local files (both for preparation and fixes) directly to the upstream main branch.

## Good to know

Having three different locations for the doc files (Git, Discourse and doc output) can be confusing.
Always keep in mind that things must look correct in the output, and we don't really care about any display issues on GitHub or Discourse.

### Links between doc pages

When the doc output is generated, links to other Discourse posts are changed to links to the corresponding doc output pages.
For example, when GitHub and Discourse have `See the [Release notes](https://discourse.ubuntu.com/t/release-notes/17842)`, the doc output has `See the <a href="/docs/release-notes">Release notes</a>`.

This means that when you link from a doc page to another doc page, you should link to its Discourse URL, not the documentation URL.
However, when you link from some other (non-doc) page, for example an announcement, you should link to the documentation URL (because we want users to read the doc output and not the Discourse pages).

If you want to use the page title as link text, you can leave the link text empty (`[](https://discourse.ubuntu.com/t/release-notes/17842)`).
However, this might cause some issues - there is a limitation for how many such links you can have in a page, they won't show up at all in the GitHub preview, and they might raise errors in the spell checker.

### Links to subheadings

Currently, no anchors are automatically generated for subheadings, so the only way to link to subheadings is to add an anchor manually.

We do this by adding a `<a name="anchor-name"></a>` in the line before the heading.
You can then link to this anchor with `[link text](#anchor-name)` (within the same page) or `[link text](https://discourse.dqlite.io/t/TOPICID#anchor-name)`.

Note that this might change in the future when we get automatic ToCs.

### Titles

Page titles are kept in a different field in Discourse than the body text.
The Git files contain only the body text, so the page title is NOT included.

Page titles are treated as L1 headings (`# Title`).
Therefore, your subheadings in the markdown files should start with L2.

You can use a different, shorter page title in the navigation.
Just specify it in the navigation table.

### Images

Images should be added both to the Git repo and to the asset manager.
See [How to publish images](#how-to-publish-images).

For the GitHub review, it can be useful to include the image from the Git repo.
That way, it is displayed correctly in the GitHub preview, and you can do changes to it by just updating it in the repo.

This won't work for publishing though, because you cannot publish just an image to Discourse.
So you must upload it to the asset manager before publishing and change the include in the file.
After that, it will still be displayed correctly in the GitHub preview, but it's more effort to update the image.

### Special markup

Any markup that is supported by Discourse or the Discourse-to-doc-output transformation but that is not Markdown will not show up correctly on GitHub.

Mainly, that is the case for notes (`[note type="information" status="Note"]`) and details (`[Details=Title]`).
But there might be other formatting - there's no real documentation on what is supported.
Some hints can be found here, though: [Snapcraft documentation guidelines](https://snapcraft.io/docs/documentation-guidelines)

### Caching

There is extensive caching going on for documentation, so your changes on Discourse might show up in the doc output - or they might not.
Or they might first show up and then disappear again.
Usually, shift-reloading a few times does the trick.
If not, append `?cache=test` to the URL.
