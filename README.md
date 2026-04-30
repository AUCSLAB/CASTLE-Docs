# CASTLE Docs
This is the official documentation for the Alfred University CASTLE Lab.
The site is currently hosted at https://aucslab.github.io/CASTLE-Docs/

# Contributing
The docs are written in Markdown and are located in the `castledocs/docs`
directory.

Do not include any sensitive information such as passwords, private keys, names
of individuals, email addresses, etc.  If you need to refer to a person, do so
by their role rather than by name (this is better for privacy, and also means
we don't need up update things as often).  Examples of roles include:
Supervising Faculty Member, Student Lab Director, Student Maintainer.

# Build and Deploy
The HTML is built using [mkdocs](https://www.mkdocs.org/).
You can preview changes by entering the `castledocs` directory and running
`mkdocs serve`
To deploy your changes, run `mkdocs gh-deploy`. This will automatically build
the site and push it to the gh-pages branch of the repository.
Remember to also push your changes to the main branch, or your edits will be
lost next time someone deploys it!
