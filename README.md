# if-db

Manifests, guides, case studies and docs for Impact Framework. 

A place to "check-in" manifests for real projects.


## Manifests

**WARNING!! these are still works in progress!!**

Manifest files for real projects. Each subdirectory refers to a given project. Every project *must* have a working manifest and *should* also have accompanying notes, explainers and other helpful materials. 

These are the manifests that we currently have available:

| Name                                                       | Description                                             | SCI                     | Manifest                                                       | Author(s)   | Date Added (ddmmyyyy) |
| ---------------------------------------------------------- | ------------------------------------------------------- | ----------------------- | -------------------------------------------------------------- | ----------- | --------------------- |
| [GSF docs website](manifests/if-docs-website)              | Manifest for the Green Software Foundation website      | 0.12 g CO2e/visit       | [manifest](manifests/if-docs-website/gsf-docs-website-sci.yml) | @jmcook1186 | 29/08/2024            |
| [IF docs website](manifests/if-docs-website)               | Manifest for the Impact framework documentation website | 0.12 g CO2e/visit       | [manifest](manifests/if-docs-website/if-docs-website-sci.yml)  | @jmcook1186 | 29/08/2024            |
| [Zoom conference call](manifests/zoom-calls/zoom-call.yml) | Manifest for a 30 min Zoom call wityh 5 participants    | 69 g CO2e / participant | [manifest](zoom-calls/zoom-call.yml)                           | @jmcook1186 | 13/09/2024            |
| [IF project](manifests/if-project/if-project-sci.yml)      | Manifest for overall SCI for entire IF project          | tbc                     | [manifest](manifests/if-project/if-project-sci.yml)            | @jmcook1186 | 29/08/2024            |

## How to contribute

You can contribute by adding a new manifest for a project, or by making improvements to those already existing in the `manifests` folder.

### Add a new manifest

Add a manifest to a new subfolder in the `manifests` folder. Include the manifest and a computed output file, along with a set of notes explaining how the calculations were done, what assumptions were made, and highlighting any particular areas of uncertainty.

We recommend using our [pipeline-template](./pipeline-template.md) to structure your notes.


### Improve an existing manifest

Choose a manifest from the `manifests` folder. Read the notes and check for any particular areas of uncertainty. Work on any components of the manifest that you think can be improved, and include the changes in a PR. Make sure you also update the notes and give a detailed description of the changes and their rationale.
