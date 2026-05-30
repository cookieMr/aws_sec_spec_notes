# AWS Security Specialist - Study Guide

<figure>
  <img src="./src/images/aws_cc_badge.png" alt="AWS - Security Specialty" width=200>
  <figcaption><center>AWS Certified Security - Specialty<br><i>Image source: <a href="https://www.credly.com/org/amazon-web-services/badge/aws-certified-security-specialty">Credly.com</a></i></center></figcaption>
</figure>

> **Note: These are my personal study notes that I am using to prepare myself for the _AWS Certified Security - Specialty_ exam.**
>
> And I'm prompting them all out.

## Read the Book

Notes in this repository are compiled into a highly readable, searchable online book using **mdBook**.

**[Read the live study guide here](https://cookieMr.github.io/aws_sec_spec_notes/)**

### e-book

If you preffer the offline mode you can download [epub](https://github.com/cookieMr/aws_sec_spec_notes/releases/download/latest/AWS.Security.Specialty.Study.Guide.2026.epub) file.

## Building Locally

If you want to run this book locally to study offline or modify the notes, you will need the [mdBook](https://rust-lang.github.io/mdBook/) command-line tool.

1. **Install mdBook** (requires the Rust toolchain):
   ```bash
   cargo install mdbook mdbook-epub
   ```
2. **Serve the book locally**:
   ```bash
   mdbook serve --open
   ```
   This compiles the Markdown files and opens a local web server at [http://localhost:3000](http://localhost:3000) with hot-reloading enabled.

## Content Generation

The core content and technical facts within these Markdown files were initially structured and generated with the assistance of AI, then curated, reviewed, and formatted specifically for this mdBook layout.

## License

<figure>
  <img src="https://fsfe.org/graphics/gplv3-logo-red.png" alt="GNU GPL v3 Logo" width=200>
  <figcaption><center>GNU General Public License v3.0<br><i><a href="https://fsfe.org/activities/gplv3/diff-draft2-draft3.en.html">Image source: FSFE</a></i></center></figcaption>
</figure>

This project is licensed under the [GNU General Public License v3.0 (GPLv3)](https://www.gnu.org/licenses/gpl-3.0.en.html#license-text).

You are free to use, modify, and distribute this study guide, provided that any modifications or derivative works are also distributed under the same open-source GPLv3 license. See the [LICENSE](./LICENSE) file for more details.
