# Unified, customizable documentation for Renku

Created: October 11, 2022 12:00 PM

Tags: docs

---

## 🤔 Context and Problem

We currently use sphinx/readthedocs to host our documentation. In the future,
we would like to have a dedicated renku homepage, detailing the project, that
also includes user documentation. Sphinx/RTD does not easily allow branding
nor creating custom pages with a modern look and feel as would be required for
such a landing page and documentation.

## 🍴 Appetite

4 weeks. Porting over existing docs and creating initial new pages does take
some time, especially design time.

## 🎯 Solution

Find and switch to a tool that:

- Allows customization with HTML pages
- Is flexible enough to support a Renku look&feel
- Is easy to migrate existing docs and automation to
- Ideally can support including docs from all our components, where needed

One possible tool for this would be [mkdocs](https://www.mkdocs.org/) with a
customized [mkdocs-material](https://squidfunk.github.io/mkdocs-material/)
theme and [mkdocstrings](https://mkdocstrings.github.io/) for code docs.

But other tools should be evaluated first as well, to see which fits our needs
the best.

## 🐰 Rabbit holes

Porting existing docs over, especially with automation (like autodoc from Python
code in renku-python) could easily consume a lot of time, especially to get it
all to look nice, so we might be ok with an initial version that's "good enough"
instead of getting lost in details.

The customized cheatsheet solution in renku-python might be difficult to port and
is not a top priority.

## 🙅 Out of scope

This pitch excludes:

- creating new documentation, other than the custom landing page. This pitch is also not about
- setting up the hosting etc. for the new Renku homepage (i.e. on renku.ch)
