# Contributing to AegisLab Documentation

Thank you for your interest in contributing to the AegisLab ecosystem.

This repository contains the source for the **AegisLab documentation site**.

Documentation improvements are essential for helping users understand how to use and extend AegisLab. Contributions that improve clarity, fix errors, or add missing information are highly welcome.

Official documentation site:

https://operationspai.github.io/AegisLab-doc/

For detailed contribution guidelines for the broader AegisLab ecosystem, see:

https://operationspai.github.io/AegisLab-doc/contributing/

---

# Ways to Contribute

You can contribute to this documentation in several ways:

- Fix typos, grammar issues, or unclear explanations
- Correct outdated or incorrect technical instructions
- Fix broken links
- Improve examples or tutorials
- Add new documentation pages
- Improve explanations of architecture or workflows

If you are unsure whether a change is appropriate, feel free to open an issue first.

---

# Reporting Documentation Issues

If you find an issue in the documentation:

1. Go to the **Issues** tab of this repository
2. Click **New Issue**
3. Select **Documentation Bug Report**
4. Provide the page URL or file path
5. Describe the problem clearly

Examples of valid reports:

- Broken links
- Incorrect commands
- Missing prerequisites
- Confusing explanations
- Outdated information

---

# Requesting New Documentation

If you believe a feature or workflow should be documented but is currently missing:

1. Open a new issue
2. Select **Documentation Content Request**
3. Describe the topic you would like documented
4. Provide context or examples if possible

---

# Submitting a Documentation Fix

Follow these steps to submit a documentation improvement.

### 1. Fork the Repository

Click **Fork** on GitHub to create your own copy of the repository.

### 2. Clone Your Fork

git clone https://github.com/YOUR-USERNAME/AegisLab-doc.git cd AegisLab-doc


### 3. Create a New Branch

git checkout -b fix/documentation-improvement


### 4. Edit the Documentation

Most documentation pages are located in:

content/


Edit the relevant Markdown files to make your changes.

### 5. Commit Your Changes

git add . git commit -m "docs: improve documentation clarity"


### 6. Push the Branch

git push origin fix/documentation-improvement


### 7. Open a Pull Request

Create a pull request describing:

- What you changed
- Why the change is needed
- Which pages are affected

Maintainers will review your changes and provide feedback if necessary.

---

# Local Development Setup

This documentation site is built using **Hugo**.

### Install Hugo

Follow the official installation guide:

https://gohugo.io/installation/

The **extended version** of Hugo is recommended.

### Run the Local Development Server

Start the local development server:

hugo server


Then open the site in your browser:

http://localhost:1313


The site will automatically reload when you edit documentation files.

---

# Documentation Structure

Important directories in this repository:

content/ Documentation pages layouts/ Hugo layout templates static/ Static assets themes/ Site theme


Most contributions only require editing files in the `content/` directory.

---

# Code of Conduct

This project follows a Code of Conduct to ensure a welcoming and inclusive community.

Please read:

CODE_OF_CONDUCT.md


By participating in this project, you agree to follow these guidelines.

---

# License

By contributing to this repository, you agree that your contributions will be licensed under the same license as the project (MIT).

---

Thank you for helping improve the AegisLab documentation.
