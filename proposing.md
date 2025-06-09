---
title: How to Propose a new HEP
---

Great, you are interested in submitting a new enhancement proposal! Below is the information needed to create one and start the process.

```{dropdown} Markdown as preferred HEP format

The recommended format for HEPs is MyST Markdown. It is a traditional Markdown with extra powers to enable high quality technical web publications. The MyST [website](https://mystmd.org/guide/typography) has detailed instructions of both typical and its special Markdown features. Start with the familiar and consult the MyST website when and if wanting something advanced. Or post an issue in the HEP [repo](https://github.com/HDFAlliance/heps) and we'll try to help.

Since we don't want to leave any HEP idea behind, those who for whatever reason do not want to use Markdown are free to chose any format. We will find a way how best to host such HEPs depending on author requests. For example, there is some support for writing in $\LaTeX$, more [information](https://mystmd.org/guide/writing-in-latex) at the MyST website.

Inline diagrams are possible using [MermaidJS](https://mystmd.org/guide/diagrams) syntax.
```

```{dropdown} Fork the repository

This is the required step since all HEP activities are centered around the content in this repository. GitHub hosts nice [documentaton](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks) on how to work with forked repositories.

Even if HEP authors do not want write Markdown, their proposal must be referenced in this repository. Any HEP author, if more than one, can fork this repo and create PRs with their HEP changes. We leave this up to the authors, but the expectation is that only one author at a time will make PR to update their HEP.
```

````{dropdown} Set up the new proposal

In the forked repository:

1. Create a folder with a name starting with `HEP` and then followed by three digits, e.g., `HEP967`. Choose the next available three-digit number from the `HEP`-named folders, but this is just a suggestion.
1. Copy the file [`index.md`](TEMPLATE/index.md) in the `TEMPLATE` folder into your new `HEP` folder. This file is the main HEP file.
1. Fill out the frontmatter in the `index.md` file with correct information.
1. The HEP authors are free to organize content any way they desire in the HEP's folder and its subfolders.
1. Create a PR when ready to officialy start the HEP process.
````

:::{dropdown} Working with MyST on your computer

It is important to preview the document while working with MyST Markdown that has so many advanced features. Below are instructions how to run the HEP website locally:

1. Install [NodeJS](https://mystmd.org/guide/install-node). Verify installation by running:

        $ npm --version

1. Install `myst` command-line [tool](https://mystmd.org/guide/installing).

        $ npm install -g mystmd

    Verify the tool:

        $ myst --version

Useful `myst` commands:

* Preview your HEP document

    ```shell
    $ myst start
    ```

    Point the browser to the link given in the terminal. Killing the `myst` command ({kbd}`Ctrl` + {kbd}`C`) in terminal stops the preview.

* Clean up the cache of the built files. This also removes the website template and enables its re-download on the next invocation of the above command.

    ```shell
    $ myst clean --all
    ```
:::
