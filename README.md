<!--
HarbourNova
Copyright 2026 Eric Lendvai (WA-USA)
SPDX-License-Identifier: GPL-3.0-or-later
-->

# HarbourNova

**A modern programming language for data-oriented applications.**

HarbourNova (HN) builds on the productive traditions of xBase and Harbour, with a language and runtime designed around richer native data types, structured collections, local data processing, and concurrency.

The goal is to make the pieces of a data-oriented application work naturally together: values that preserve their meaning, collections that adapt to the data they hold, and SQL that can operate directly on application-owned datasets.

This is the public HarbourNova repository. Its initial content is a set of developer articles introducing important parts of **HarbourNova 1 (HN1)**, the first release currently under development.

## Explore the HN1 design

The articles below explain the features, their design, and the practical problems they address. Each link opens the formatted document directly on GitHub.

| Topic | What you will learn |
| --- | --- |
| [The Number data type](design/HarbourNova_HN1_Number_Data_Type.md) | Exact decimal arithmetic, PostgreSQL-aligned numeric capacity, efficient internal handling of smaller values, and consistent numeric behavior across HN code and Local SQL. |
| [Tensors](design/HarbourNova_Tensors_HN1_Developer_Article.md) | Native collections with an explicit shape and rank, resizing, and storage that adapts between dense or sparse layouts and direct values or dictionary encoding. |
| [Local SQL](design/HarbourNova_Local_SQL_Developer_Overview.md) | SQL over native Cursors, typed local datasets, joins and other relational operations, and workflows that combine backend results collected through Jobs and Tasks. |

[Browse the design folder](design/).

### Why these features belong together

Consider an application that retrieves information from several backend systems, combines the results, calculates totals, and prepares a report.

- **Number** preserves exact decimal values during application calculations.
- **Tensors** provide structured collections whose storage can adapt to sparse data and repeated values.
- **Cursors and Local SQL** provide typed tables and relational processing inside the application.
- **Jobs and Tasks** allow independent retrieval work to run concurrently before the results are combined.

The articles describe how these capabilities fit into HN's language and data model, including the intended relationship between Tensor storage and future Cursor storage.

## About these documents

These articles present the HN1 design and explain its behavior from a developer's perspective. Features described as part of HN1 belong to the first-release design; each article identifies any future work separately. As development progresses, implementation details and examples may be refined.

At this initial stage, the repository publishes documentation. Compiler and runtime source distributions, installation instructions, and release announcements will identify the software available to download and use.

## Follow the project

- [HarbourNova website](https://harbournova.org/)
- [HarbourNova users community](https://groups.google.com/g/harbournova-users)
- [HarbourNova on GitHub](https://github.com/harbournova)

Questions, feedback, and real application use cases are welcome. For feedback on a document, please identify the article and section so the discussion can stay focused.

## License and copyright

Copyright 2026 Eric Lendvai (WA-USA)

Unless a file or an identified portion states otherwise, the original HarbourNova documentation and other original material in this repository are licensed under the **GNU General Public License, version 3 or, at your option, any later version** (`GPL-3.0-or-later`).

See [LICENSE.txt](LICENSE.txt) for the full license and [COPYRIGHT.md](COPYRIGHT.md) for the copyright notice, scope, and treatment of separately licensed material.
