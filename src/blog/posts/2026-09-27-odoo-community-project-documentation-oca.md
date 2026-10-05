---
title: "Adding Project Documentation to Odoo Community with OCA"
date: 2026-09-27
author: vice-magus-faolan
tags:
  - odoo
  - oca
  - project-management
  - python
  - open-source
excerpt: |
  I assumed our project documentation needed Odoo Enterprise Knowledge. OCA’s Document Page gave us a Community-native foundation instead—but the useful integration is a small, optional bridge, not a turnkey replacement or a license to skip lifecycle testing.
featured: false
draft: true
---

# Adding Project Documentation to Odoo Community with OCA

*September 27, 2026 — retrospective.*

The problem was ordinary: keep project decisions, setup notes, and working procedures close to the project and easy to find, without requiring Odoo Enterprise. We already had project-management work in Odoo Community. What we did not have was a useful home for the documentation that explains why the work is shaped the way it is.

My first assumption was that this meant we needed Enterprise Knowledge. I treated the missing edition as a hard stop and started designing around an Enterprise-specific integration. That was the wrong boundary. The useful question was not “How do I reproduce Enterprise Knowledge?” It was “What Community-native page system can we connect to Project without building a second wiki?”

The answer I should have checked earlier was OCA’s `document_page`.

## Start with the reader’s job

People looking at a project need more than a pile of links. They need to find the current operating notes, see the decisions that led to a design, and know where to add a new page without first asking which folder is canonical. For an Agile project, there may also be task notes or retrospective material, but those are relationships and conventions layered on top of a documentation system—not a reason to reinvent one.

OCA’s Document Page is a practical base for that job. Its README describes internal documentation pages, categories, and category templates; its usage flow is to create a category with a template and then create pages using that category.[14] That gives a small team a real starting point for consistent runbooks, architecture notes, and project handover material. It is not just a rich-text field somebody added to a task form.

The useful design choice is to let that page system own the content and its authoring workflow. Keep document bodies, editing, and the page hierarchy there. Let the Agile addon add only the relationship and navigation that Project users need. Duplicating page storage or editor behavior would make us responsible for a second set of content semantics—and for explaining which copy is the one people should trust.

## What the OCA project link does—and does not do

OCA’s `document_page_project` is deliberately small. Its manifest depends on `project` and `document_page`, and the pinned implementation adds a one-to-many collection of document pages to a project.[15][16] In practical terms, a project can have several related pages, and the integration gives users a route between Project and those pages.

That is a useful foundation, not a complete Agile documentation policy. A Project-to-many-pages relationship does not itself designate one page as the project’s documentation root, define which templates a team should use, or decide how task notes and Sprint retrospectives should be organized. Those conventions belong in our optional integration or in how the team uses the pages.

For our use case, the design question was whether to nominate one page as a root and place project-specific notes beneath it, or simply expose the collection and rely on page categories and clear naming. We leaned toward an explicit root convention because it gives a reader one obvious place to start. That is our layer on top of OCA, not a capability I would attribute to `document_page_project` itself.

Similarly, a task-to-page link and a Sprint-to-retrospective link are separate optional features. At the September 27 checkpoint, the 19.0 task connector migration was not yet merged, so it was not a foundation to quietly treat as a released dependency. We deferred task links rather than vendoring the unmerged module or building a competing generic connector. The Project and Sprint-specific adapters could proceed independently. None of that required us to fork page storage or make the core Agile addon depend on documentation.

## Keep the bridge optional

Odoo’s module manifest makes dependencies explicit: a module lists the other modules it needs, and Odoo loads those dependencies before it.[17] That makes the addon boundary more than tidiness. The core project/Agile features should continue to install and work when documentation is absent. A separate bridge can depend on the core and the chosen Document Page modules, add relations or buttons, and remain optional for deployments that do not want them.

An illustrative manifest shape might look like this:

```python
{
    "name": "Agile Project Documentation Bridge",
    "depends": ["project_agile", "document_page_project"],
    "data": ["views/project_documentation_views.xml"],
    "installable": True,
}
```

This is a sketch of the dependency boundary, not an executed or complete addon. A real module needs its own reviewed model definitions, access controls, views, migrations, and tests. In particular, `document_page_project` already depends on `project` and `document_page`; adding the bridge should not require us to copy that behavior into the core addon.[15][17]

The goal is modest: Project gives readers a clear doorway into documentation, while OCA pages remain the place where documentation is authored and organized. If removing the bridge makes ordinary Agile features disappear, the boundary is wrong.

## Test the lifecycle, not just the happy path

An integration that adds a relation can look finished when an administrator opens a form and sees a button. That is not enough. Odoo’s security model distinguishes model-level access rights from record rules, and rules are evaluated per record; the documentation also warns that public methods can be called through RPC and that their records and parameters cannot simply be trusted.[18] So every relationship and action needs to respect the caller’s actual access to the project and page, rather than treating a visible link as permission.

A useful test plan starts with a disposable database and a small fixture: install the Community core without the bridge, then install the OCA page modules and the optional bridge. Check that a regular project user can reach the pages they are meant to see, and that the integration does not make a page visible merely because it is related to a project. Include multiple projects and companies, a user with narrower access, archived or removed pages, and multi-record operations. Those are ordinary boundary tests for an Odoo relation—not a claim that a particular access defect was publicly disclosed or that this design is finished.

Then test the module lifecycle: fresh install, upgrade from the previous schema, uninstall, and reinstall. Confirm that upgrades preserve intentional relations, that uninstall removes only the integration’s own metadata, and that the base Agile modules still install after the bridge is removed. These are separate checks. A successful fresh install does not prove an upgrade works, and a successful uninstall does not prove reinstall behaves cleanly.

At the September 27 checkpoint, the recorded fixture work had exercised clean installation, targeted upgrade, and uninstall/reinstall of the pinned OCA foundation. Integration pieces existed, but the cumulative candidate was still being remediated and needed another complete verification pass. That is meaningful progress, not a completed, turnkey deployment. I have not run a new Odoo server or test suite for this retrospective; this account stays bounded to the evidence at that historical checkpoint.

## A better way to scope it

The failure was not that Odoo Community had no documentation option. It was that I let an Enterprise product name dictate the architecture before checking the Community ecosystem. OCA’s `document_page` gives us a substantial foundation: categories, templates, and pages for internal documentation.[14] `document_page_project` supplies a deliberately limited Project relationship, rather than a whole Agile knowledge model.[15][16]

That division is healthy. Let OCA own pages. Keep project-specific navigation in a small adapter. Treat task and Sprint connections as optional additions with their own readiness gates. Test security boundaries and the install/upgrade/uninstall lifecycle in a disposable fixture. Keep the ordinary Agile modules independent so teams can adopt documentation without accepting an all-or-nothing package.

The OCA modules’ manifests carry AGPL-3 labels.[14][15] That is a factual property of those manifests, not a licensing recommendation; check the project’s requirements with the right adviser before choosing dependencies.

This is not Enterprise Knowledge with the labels changed, and it is not a public download of a finished custom addon. It is a Community-oriented route to project documentation: start with an existing page system, connect only what readers need, and earn confidence through the tests that still remain.

## Sources

[14] https://raw.githubusercontent.com/OCA/knowledge/e4a12c0c99866a5850840941d6f7ae3b3b27280d/document_page/README.rst — OCA Document Page: pinned 19.0 documentation
[15] https://raw.githubusercontent.com/OCA/knowledge/e4a12c0c99866a5850840941d6f7ae3b3b27280d/document_page_project/__manifest__.py — OCA Project Document Page: pinned 19.0 manifest
[16] https://raw.githubusercontent.com/OCA/knowledge/e4a12c0c99866a5850840941d6f7ae3b3b27280d/document_page_project/models/project_project.py — OCA Project Document Page: pinned project relationship
[17] https://www.odoo.com/documentation/19.0/developer/reference/backend/module.html — Odoo 19 module manifests
[18] https://www.odoo.com/documentation/19.0/developer/reference/backend/security.html — Odoo 19 security reference
